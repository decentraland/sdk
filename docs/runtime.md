---
title: "Metaverse runtime"
slug: "/contributor/sdk/diagrams/metaverse-runtime"
---

# The Metaverse Runtime

## Design principles

- The whole system is data oriented, implemented using an ECS approach - https://www.dataorienteddesign.com/dodbook/
- Entities are only numeric identifiers, not classes
- Components (and therefore entities) are synchronizable
- Each instance of the runtime may have its own non-synchronized Components, i.e. some game-specific components like "CurrentTarget" or "Speed".
- The runtime is WASM + WASI, only the "update" function is exposed as an `extern "C" void update(delta: double)` 
- The runtime talks to other components via Unix sockets (file descriptors, like in Plan9 & BSD)

## Unix sockets

For every socket ended in `/entities` the _EcsProtocol_ is used to encode the packets.

- `/dcl/scene/entities` `fileDescriptor: 3` read & write socket. Used to read/write the static definition of the scene. Only tooling should be able to write to this socket, like the builder.
- `/dcl/renderer0/entities` `fileDescriptor: 4` read & write socket. Used to talk directly to the renderer and vice-versa. The renderer will send input messages through this channel.
- `/dcl/transactor0/entities` `fileDescriptor: 5` read & write socket.

## Events

Event handlers are not part of the new runtime. Historically, events were the result of hardware interrupts, for instance, the BIOS would trigger a specific electric path to the processor every time a key was pressed. That was the tool available at the moment that was also in-line with the hardware limitations (mostly ram) of the moment.

Nowadays such limitations don't apply anymore. A data-oriented approach will be considered for the runtime.

That means, in practical ways, that input events will now be components in some entity. It could be an "Input entity", or the root entity of the scene.

Every time a key is pressed, an update containing information of the `KeyPressedComponent(keyCode)` message will be queued and then read by the runtime file descriptor.

The component will be plain data attached to an entity, then the systems of the ECS can query all entities with that component at any moment to decide on different behaviors for the scenes. Systems can even remove the component if it is no-longer needed.

# EcsProtocol

_EcsProtocol_ is the protocol and serialization format used to encode synchronizable chunks of components for entities. The protocol implements the operations for a Map CRDT.

The serialization format is ProtocolBuffers. Every message implements the `MessageHeader` packet.

### Operations

### Schema & Serialization

ProtocolBuffer is the selected serialization format for the EcsProtocol, the protocol should always be small enough to ensure that an efficient (cpu and memory-wise) encoder/decoder can be created. The `data` field of the message can be serialized in other more convenient formats.

```protobuf
message WireMessage {
    repeated ComponentOperation operations = 1;
    repeated Query query = 2;
    repeated Responses query_response = 3;
}

message ComponentOperation {
    int32 message_type = 1;     // PUT_COMPONENT | DELETE_COMPONENT
    int64 entity_id = 2;        // Entity number
    int64 component_number = 3; // ClassIdentifier number for the component kind

    // We use lamport timestamps to identify components and to track in which order
    // a client created them. The key for the lamport number is key=(entity_id,component_number)
    int64 timestamp = 4;

    // @optional
    // missing data means the component was deleted
    bytes data = 5;
}

/*
# Open questions

  - What would happen if two scenes create a component for the first time, and both use the lamport number 1?
  - Should we generate random IDs for new dynamic entities?
    * we could claim buckets of random numbers every time we start a session to prevent collisions
*/


//// deprecate
message Query {
    int32 message_type = 1; // QUERY
    int64 query_type = 2; // RAYCAST | ...?
    reserved 3;

    // Serialized query
    bytes data = 5;
}

```

#### Raycast query example

```sequence
participant Scene as S
participant Renderer as R

S->S: Raycast {entity=1,dir=xyz}\nSetComponent(Entity(1), Raycast(xyz))
S-->R: PutComponent(1, Raycast(xyz))
R->R: PutComponent(Entity(1), Raycast(xyz))
R->R: RaycastSystemUpdate()\nSetComponent(Entity(1), RaycastResultComponent(xyz))
R-->S: PutComponent(1, RaycastResultComponent(xyz))
S->S: PutComponent(Entity(1), RaycastResultComponent(xyz))
S->S: RaycastSystemUpdate()\nRaycast(xyz).Resolve(RaycastResultComponent(xyz))



```


### Sources of truth, introducing the transactor

Since CRDTs don't require consensus, a constant stream of operations needs to be synchronized among peers. Including initial states and partial states. That yields an exponential amount of "merge operations" and bottlenecks identifying the deltas. Those problems are multiplied by the amount of nodes that are synchronized. To prevent those errors, a "transactor" actor is added to the system. The "transactor" is a service that listens to changes of all peers, apply changes, compress them and then broadcasts changes to all peers. The "transactor" stores the snapshots of the states of the nodes.

- https://hal.inria.fr/file/index/docid/555588/filename/techreport.pdf
- https://docs.datomic.com/on-prem/overview/transactor.html
- https://crdt.tech/resources

## CRDT

CRDTs (conflict-free replicated data types) are data types on which the same 
set of operations yields the same outcome, regardless of order of execution 
and duplication of operations. This allows data convergence without the need 
for consensus between replicas. In turn, this allows for easier implementation 
(no consensus protocol implementation) as well as lower latency (no wait-time 
for consensus).

Operations on CRDTs need to adhere [to the following rules][mixu]:

- Associativity (a+(b+c)=(a+b)+c), so that grouping doesn't matter.
- Commutativity (a+b=b+a), so that order of application doesn't matter.
- Idempotence (a+a=a), so that duplication doesn't matter.

Data types as well as operations have to be specifically crafted to meet these
rules. CRDTs have known implementations for counters, registers, sets, graphs,
and others. Roshi implements a set data type, specifically the Last Writer
Wins element set (LWW-element-set).

This is an intuitive description of the LWW-element-set:

- An element is in the set, if its most-recent operation was an add.
- An element is not in the set, if its most-recent operation was a remove.

A more formal description of a LWW-element-set, as informed by
[Shapiro][shapiro], is as follows: a set S is represented by two internal
sets, the add set A and the remove set R. To add an element e to the set S,
add a tuple t with the element and the current timestamp t=(e, now()) to A. To
remove an element from the set S, add a tuple t with the element and the
current timestamp t=(e, now()) to R. To check if an element e is in the set S,
check if it is in the add set A and not in the remove set R with a higher
timestamp.

Roshi implements the above definition, but extends it by applying a sort of
instant garbage collection.  When inserting an element E to the logical set S,
check if E is already in the add set A or the remove set R. If so, check the
existing timestamp. If the existing timestamp is **lower** than the incoming
timestamp, the write succeeds: remove the existing (element, timestamp) tuple
from whichever set it was found in, and add the incoming (element, timestamp)
tuple to the add set A. If the existing timestamp is higher than the incoming
timestamp, the write is a no-op.

Below are all possible combinations of add and remove operations.
A(elements...) is the state of the add set. R(elements...) is the state of
the remove set. An element is a tuple with (value, timestamp). add(element)
and remove(element) are the operations.

Original state | Operation   | Resulting state
---------------|-------------|-----------------
A(a,1) R()     | add(a,0)    | A(a,1) R()
A(a,1) R()     | add(a,1)    | A(a,1) R()
A(a,1) R()     | add(a,2)    | A(a,2) R()
A(a,1) R()     | remove(a,0) | A(a,1) R()
A(a,1) R()     | remove(a,1) | A(a,1) R()
A(a,1) R()     | remove(a,2) | A() R(a,2)
A() R(a,1)     | add(a,0)    | A() R(a,1)
A() R(a,1)     | add(a,1)    | A() R(a,1)
A() R(a,1)     | add(a,2)    | A(a,2) R()
A() R(a,1)     | remove(a,0) | A() R(a,1)
A() R(a,1)     | remove(a,1) | A() R(a,1)
A() R(a,1)     | remove(a,2) | A() R(a,2)

For LWW-element-set, an element will always be in either the add or
the remove set exclusively, but never in both and never more than once. This
means that the logical set S is the same as the add set A.

Every key represents a set. Each set is its own LWW-element-set.

For more information on CRDTs, the following resources might be helpful:

- [The chapter on CRDTs][mixu] in "Distributed Systems for Fun and Profit" by Mixu
- "[A comprehensive study of Convergent and Commutative Replicated Data Types][shapiro]" by Mark Shapiro et al. 2011

[mixu]: http://book.mixu.net/distsys/eventual.html
[shapiro]: http://hal.inria.fr/docs/00/55/55/88/PDF/techreport.pdf

## Synchronization rules

```fsharp
type EntityComponentValue = {
    bytes  data      // serialization of the component value
    number timestamp // lamport timestamp
}

fun sendUpdate(entity, component_id, value) = ???

fun processUpdate(entity_id, component_id, newValue) =
    let currentValue = entities[entity_id][component_id]
    if (currentValue.timestamp > newValue.timestamp) {
        // discardMessage() and send newer state to the sender
        // keep our current value
        sendUpdates(entity, component_id, currentValue)
    } else if (currentValue.data > newValue.data) {
        // if somehow 🌈 the currentValue is greater than the new value,
        // send newer state to the sender. keep our current value
        sendUpdates(entity, component_id, currentValue)
    } else {
        entities[entity_id][component_id] = newValue
    }
```

## Open questions

- Without limitations, any actor in a system can create entities. How can infinite storage, resources or messages be avoided?
  Can synchronized entities be created in a single place?  
- Will every entity be synchronized?
- Will every entity be stored?
- If entities are synchronized, what is the mechanism to clear the storage in new scene deployments? -> Scene deployments may have different CRDT roots


## References

- https://www.serverless.com/blog/crdt-explained-supercharge-serverless-at-edge/
- https://www.researchgate.net/publication/310212186_Near_Real-Time_Peer-to-Peer_Shared_Editing_on_Extensible_Data_Types
- https://josephg.com/blog/crdts-are-the-future/
- https://github.com/automerge/automerge


https://diagrams.menduz.com/#/notebook/2l3t8FEx6Yc4GyDvkdDe4EQKf2L2/-MiSDLMvharXiZu6Vfjl