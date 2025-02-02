# Code Overview

## Table of Contents
* [Game Initialization](./code-overview.md#game-initialization)
* [Authoring Setup](./code-overview.md#authoring-setup)
* [Planets](./code-overview.md#planets)
* [Buildings](./code-overview.md#buildings)
* [Ships](./code-overview.md#ships)
* [AI](./code-overview.md#ai)
* [Entity Initialization and Death](./code-overview.md#entity-initialization-and-death)
* [Acceleration Structures](./code-overview.md#acceleration-structures)
* [VFX](./code-overview.md#vfx)

-------------------------------------------------------------------------

## Game Initialization

All game initialization is handled in `GameInitializeSystem`. It waits for an entity with a `Config` component to exist, and waits for `Config.MustInitializeGame` to be true. Once this condition is met, it initializes the game in these main steps:
* Spawn team managers and team home planets.
* Spawn neutral planets.
* Spawn moons around all planets.
* Spawn initial ships for all teams.
* Spawn initial buildings for all teams.
* Create acceleration structures: spatial database, planet navigation grid, planet network.
* Place camera.

-------------------------------------------------------------------------

## Authoring Setup

Authoring components in this project rely heavily on the `[RequireComponent()]` attribute to ensure that all components that are indirectly required by a certain authoring component are also added to the entity. For example:
* Fighter ships always need the general components common to all Ships
* Ships always need Initializable, Team, and Health components (although they're not the only type actor that may need these)
* Health always needs an Initializable component (just like Ships, but Health could be added to objects that aren't ships)
* etc...

By doing the following, we ensure that all component dependencies are added to entities whenever an authoring component is added to a GameObject:
* `FighterAuthoring` has `[RequireComponent(typeof(ShipAuthoring))]`.
* `ShipAuthoring` has `[RequireComponent(typeof(InitializableAuthoring))]`, `[RequireComponent(typeof(TeamAuthoring))]` and `[RequireComponent(typeof(HealthAuthoring))]`.
* `HealthAuthoring` has `[RequireComponent(typeof(InitializableAuthoring))]`

An alternative way of handling this would have been to make sure that `FighterAuthoring` takes care of adding Ship, Initializable, Health, and Team components directly in the baker. However, a downside of this approach is that any inspectors for Ship, Initializable, Health, and Team would have to be re-created for every type of authoring component that needs these (such as the various building authorings, the authorings for other ship types, etc...). The `[RequireComponent()]` approach doesn't have this drawback.


#### Blob data authoring

Ships and buildings all have some data that is common across all instances and never changes per-instance. In these cases, it can be beneficial to store that data in blob assets, because it reduces the memory footprint of our actors, and reduces the size of our actor archetypes in chunks (leading to improved performance in certain cases). For this, we use the `BakedScriptableObject` utility in the project. This is a simple class that `ScriptableObjects` can inherit from, and it allows them to act as authorings/bakers for blob assets. These `ScriptableObjects` can be dragged and dropped into the inspectors of authoring components, and get converted to blobs during baking.

Let's see how this works using the `FighterDataObject` as an example:
* `FighterDataObject` inherits from `BakedScriptableObject<FighterData>`, where `FighterData` is an unmanaged representation of the data that is common to all fighters and doesn't change per instance.
* `FighterDataObject` has a `FighterData` serializable field, and overrides `BakeToBlobData` (from `BakedScriptableObject`). `BakeToBlobData` is called when our `BakeToBlobData` is asked to set the blob asset unmanaged data during baking. 
* `FighterAuthoring` is our authoring behaviour, and it has a field of type `FighterDataObject`
* During baking, in `FighterAuthoring.Baker`, we call `BakeToBlob` on the authoring's `FighterDataObject`, and assign it as a `BlobAssetReference<FighterData>` in our `Fighter` component.
* With all this, we have "baked" a scriptable object to a blob asset that is referenced by all `Fighter` components.

All the various "baking" scriptable objects in the game can be found under `Assets/Settings/Game`.

-------------------------------------------------------------------------

## Planets

Most planet logic is handled in the `PlanetSystem`:
* `PlanetShipsAssessmentJob` handles building a list of ship types per team around each planet. This is used for AI.
* `PlanetConversionJob` handles updating planet conversion when workers are capturing them.
* `PlanetResourcesJob` handles updating planet resource generation.
* `PlanetClearBuildingsDataJob` handles clearing bonuses applied to the planet by the research buildings.

-------------------------------------------------------------------------

## Buildings

Most building logic is handled in the `BuildingSystem`:
* `TurretInitializeJob` handles initialization for turrets.
* `BuildingInitializeJob` handles initialization for buildings.
* `BuildingConstructionJob` handles updating the construction of buildings (insitgated by workers).
* `TurretUpdateAttackJob` handles updating target detection and attacking for turrets.
* `TurretExecuteAttack` is a single-threaded job that handles applying damage to entities attacked by turrets. It handles only the part of it that has to happen on a single thread in order to avoid race conditions. `TurretUpdateAttackJob` is a parallel job responsible for updating attack timers and setting a `ExecuteAttack` component to enabled when ready to attack. Then, `TurretExecuteAttack` updates only for entities with an enabled `ExecuteAttack` component, and handles executing the attack.
* `ResearchApplyToPlanetJob` handles making research buildings register their bonuses with their associated planet.
* `FactoryJob` handles updating ship production for factory buildings.

-------------------------------------------------------------------------

## Ships

Most ship logic is handled in `ShipSystem`:
* `ShipInitializeJob` handles initialization for ships.
* `FighterInitializeJob` handles initialization for fighters more specifically.
* `ShipNavigationJob` is a common job for handling navigation for all ship types (fly towards a destination and avoid planets).
* `FighterAIJob` handles target detection, target chasing, decision-making (which planet to go to), and attack updating for fighters.
* `WorkerAIJob` handles decision-making for workers, which means choosing either a planet to capture, or a building to construct on a planet's moon.
* `TraderAIJob` handles decision-making for traders, which means choosing planets to take resources to and from.
* `FighterExecuteAttackJob` is a single-threaded job that handles applying damage to entities attacked by fighters. It handles only the part of it that has to happen on a single thread in order to avoid race conditions. `FighterAIJob` is a parallel job responsible for updating attack timers and setting a `ExecuteAttack` component to enabled when ready to attack. Then, `FighterExecuteAttackJob` updates only for entities with an enabled `ExecuteAttack` component, and handles executing the attack.
* `WorkerExecutePlanetCaptureJob`: similarly to `FighterExecuteAttackJob`, this is the single-threaded job that handles executing worker planet capture after `WorkerAIJob` does most of the work in parallel.
* `WorkerExecuteBuildJob`: similarly to `FighterExecuteAttackJob`, this is the single-threaded job that handles making workers execute building construction after `WorkerAIJob` does most of the work in parallel.
* `TraderExecuteTradeJob`: similarly to `FighterExecuteAttackJob`, this is the single-threaded job that handles executing trader resource exchanges after `TraderAIJob` does most of the work in parallel.

-------------------------------------------------------------------------

## AI

AI in this game is implemented using a simple utility AI system.

Most AI calculations happen in `TeamAISystem`. A `TeamAIJob` will make each team build lists of possible actions that their ships and buildings can choose from, and will assign an "importance" to each of these actions. Then, ships and buildings will select an action from these lists based on a weighted random, where "importance" acts as the weight.

`TeamAIJob` computes statistics about known ships, planets, and their surroundings, and builds 4 lists of potential actions:
* `DynamicBuffer<FighterAction>`: contains all the actions that fighters can choose from. There is one action per captured planet or planet that is near a captured planet, which means a fighter's actions are essentially "which planet to go to". For each action, we remember if this is a planet to attack or to defend, and the importance assigned to that action depends on how much the team's AI parameters favors attack over defense.
* `DynamicBuffer<WorkerAction>`: contains all the actions that workers can choose from. There is one "construct a building" action per captured planet with a free moon to build something on, and one "capture planet" action per uncaptured planet neighboring our captured planets.
* `DynamicBuffer<TraderAction>`: contains all the actions that traders can choose from. There is one action per captured planet, and each action stores information about how many resources this planet has compared to the other captured planets for this team.
* `DynamicBuffer<FactoryAction>`: contains all the actions that factories can choose from. These is one action per ship type (because these actions represent "which ship should factories build"), and each ship type is given an importance score that depends on what type of ship this team lacks the most at the moment (among other factors).

Once `TeamAIJob` has finished computing all these possible actions and their importances for the team, individual actors such as ships and factories will choose from these actions, with some personal bias involved. For example, when a fighter ship chooses an action in `FighterAIJob`, it applies a "proximity bias" to the importance score of each action. This means that the fighter ship will tend to favor planets that are nearby even if they're not the best-scoring planets. Once the fighter has a final score with personal bias for each action, it will select one action with a weighted random. However, note that the random used here is always obtained via `GameUtilities.GetDeterministicRandom(entity.Index)`. This gives us a random that is unique to this entity, but does not change from frame to frame. This provides stability to AI decisions, and prevents ships from picking a different action every frame.

See Debug Views for visualizations:
* [Fighter Actions](./debug-views.md#fighter-actions)
* [Worker Actions](./debug-views.md#worker-actions)
* [Trader Actions](./debug-views.md#trader-actions)

-------------------------------------------------------------------------

## Entity Initialization and Death

Here is how entity initialization and death events are handled in this game:

Initialization:
* Ship and building entities start off with an enabled `Initialize` component.
* Jobs such as `BuildingInitializeJob`, `ShipInitializeJob`, `FighterInitializeJob` run on entities with an enabled `Initialize` component, and handle initialization logic.
* `FinishInitializeSystem` updates towards the end of the frame, and sets any `Initialize` component to disabled. Disabling this component as a separate step at the end of the frame allows multiple components on the same entity to perform their initialization step, without requiring multiple initialization enabled components.

Death:
* Ship and building entities have a `Health` component.
* Jobs such as `BuildingDeathJob` and `ShipDeathJob` detect when `health.IsDead()` is true (when health reaches 0), and perform some death-related actions.
* Finally, a `FinalizedDeathJob` runs last and handles destroying the entities with `health.IsDead()` being true.

-------------------------------------------------------------------------

## Acceleration Structures

This game uses 3 main acceleration structures:

#### Spatial Database

The spatial database allows fast querying of entities in a bounding box.

The world is divided in a uniform grid of cells around the origin, and cells store information about which entities are within their bounds. Every frame, a `ClearSpatialDatabaseSystem` clears all stored data in the spatial database. Then, `BuildSpatialDatabasesSystem` iterates over all ships and buildings, and adds them to the spatial database (calculate which cell they belong to, and add themselves to this cell). 

Once built, the spatial database can be queried using the following functions:
* `SpatialDatabase.QueryAABB`: gets the spatial database cells that would intersect the AABB, and iterates the cells in order of bottom-to-top coordinates.
* `SpatialDatabase.QueryAABBCellProximityOrder`: gets the spatial database cells that would intersect the AABB, and iterates the cells in layers around the central cell of the AABB (in other words: in rough order of proximity to center of AABB). This provides interesting optimization opportunities, because when we're interested in finding the closest result to a point in space, we can early-exit out of iterating cells if we have found a valid result in the current cell. This doesn't 100% guarantee that we have found the absolute closest result, but in most case will be a good-enough approximation.

For each spatial database entry in a cell, a `byte Team` and a `byte Type` is also stored at the moment of building the spatial database. This allows spatial queries to very quickly pick or discard results based on the team or the actor type of this entity. For example, when fighters need to query for other actors to attack in `FighterAIJob` (and subsequently in their `ShipQueryCollector`), they can efficiently discard all results that belong to their own team by comparing the result's team to their own. They do not need to do a `ComponentLookup<Team>` for each result. Similarly, when planets need to assess what types of ships of which teams are around them in `PlanetShipsAssessmentJob` (and subsequently in `PlanetAssessmentCollector`), they can very quickly understand the team and type of ship of iterated results without requiring component lookups.

See [Debug Views](./debug-views.md#spatial-database) for a visualization.


#### Planet Navigation Grid

The planet navigation grid allow fast planet avoidance for ships.

The world is divided in a uniform grid of cells around the origin. Each cell computes and stores data about the planet that is closest to that cell. During the game, ships use the Planet Navigation Grid to very efficiently process planet avoidance. With simple math, they can calculate the cell index where they currently are, and access the closest planet data at that cell index. Knowing the distance, position, and radius of that closest planet, they are able to compute planet avoidance only on the planet that matters, only when they have to, and without requiring lookups.

The navigation grid is computed once on start in `GameInitializeSystem.CreatePlanetNavigationGrid`, and is used in `ShipNavigationJob` when calling `PlanetNavigationGridUtility.GetCellDataAtPosition`.

See [Debug Views](./debug-views.md#planet-navigation-grid) for a visualization.


#### Planets Network

The planets network allows fast neighbor planet searches for AI.

In `GameInitializeSystem.ComputePlanetsNetwork`, each planet computes a `DynamicBuffer<PlanetNetwork>`, which holds the X closest planets to this planet.

See [Debug Views](./debug-views.md#planets-network) for a visualization.

-------------------------------------------------------------------------

## VFX

All VFX in this game is handled by `VFXSystem` and VFXGraphs. At the scale required by this game, spawning one VFXGraph GameObject per vfx instance would very quickly become a performance bottleneck. Instead of this, we use a mostly gameObjects-less approach where every instance of a given type of vfx is handled by one single pre-instantiated VFXGraph in the scene. Each vfx instance is a "spawn a VFX here" message sent to a single VFXGraph object through graphics buffers. This approach allows spawning a very large amount of VFX very often, at very little cost. 

`VFXSystem` holds one `VFXManager` for each type of VFX in the game: laser sparks, explosions, thrusters. `VFXManager`s hold native collections of vfx requests. During the game, various job will write to these vfx request collections in order to ask for a VFX to be spawned. For example:
* `FighterExecuteAttackJob` creates requests for laser sparks effects.
* `ShipDeathJob` creates requests for explosion effects.
* `ShipInitializeJob` creates requests to spawn a thruster VFX for this ship, and `ShipSetVFXDataJob` updates the data that this VFX uses in its update (parent transform data)

At the end of the frame, `VFXSystem` updates all `VFXManager`s, who in turn are responsible for uploading their vfx requests to their respective VFXGraphs via graphics buffers. The rest of the work is done by VFX Graphs, who sample the graphics buffers and spawn particles based on the data in these.
