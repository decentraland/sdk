# Static entities

Static (or reserved) entities are a special reserved number of entities to be
used by the renderer itself and the scenes. Those entities hold no special logic compared
to other entities, but the systems around them use the static IDs of those entities
to share information about the renderer, the player, camera, etc.

There are 512 reserved static entities numbers, starting at 0. The "0" entity is the root
entity of the scenes, that entity is the parent of all the entities by default. 

## List of static entities per scene

- `RootEntity = 0` it is the root of the scene
  - The root entity has the scene metadata (scene.json) included as a `SceneInformation`
  - The `ForeginEntitiesFrom { names: string[] }` component is set from the scene itself and it is used to signal the renderer about Foregin entity replication policies
  - The `ForeginEntitiesReservation { name: string, from: Entity, to: Entity }` component is used to signal the renderer (or other actors) that this scene CAN share its entities as ForeginEntities.
  - 
- `PlayerEntity = 1` represents the current player avatar
  - The `Transform` component is READ/WRITE from the scene
  - The `PlayerPortableExperiences` component (READ ONLY) contains the information about the current portable experiences run by the user
  - The `PlayerIdentity` component (READ ONLY) contains information about the ethereum address, guest mode and names of the player
  - The (internal) `AvatarShape` component (READ ONLY) contains information about the wearables, hair, eyes and skin colors, and the equiped wearables

- `CameraEntity = 2`
  - The Transform component is READ ONLY from the scene
  - There is a special `CameraMode { mode = ThirdPerson/FirstPerson }` component used to get the current camera mode

## Foregin entities in scenes

There are some cases in which we can connect scenes and get the entities and component information from other scenes, this
can be used for a variety of scenarios, but for a practical approach we are focusing now in the GlobalAvatarScene.

The GlobalAvatarScene is a scene like any other that runs connected to comms, it is in charge of interpreting the comms messages rendering
the avatars in-world. In other terms, we can see other players in-world because a portable experience (GlobalAvatarScene) renders them as entities.

This is so, to leverage the performance of the scenes and to not add extra moving parts to the renderer. Keeping its only responsiblity within the boundaries
of rendering scenes and UIs only.

To extract the position or shape information of other avatars from any other scene, we should have a mechanism in place to either: 1) query other scene entities
or 2) having references/foregin entities replicated to all scenes.

We decided to do the 2nd to leverage all the CRDT messaging.

To prevent Entity IDs collisions, the scene that will own the foregin entities must reserve a slice of IDs at the moment of initialization (via RootEntity component).
