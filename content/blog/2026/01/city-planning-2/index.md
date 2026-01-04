---
title: "Developing a city-builder; Implementing a basic road system"
date: 2026-01-04
draft: true
tags: ["game-dev", "rust", "city-planning", "bevy"]
---

## Introduction

In the [previous post](/blog/2025/12/city-planning-1/), I outlined my frustrations with zoning in modern city builders and set a goal; create a more realistic system where plots naturally fill available space along curved roads. Before we can tackle zoning though, we need roads to zone around.

This post covers the first step, implementing a basic polyline road tool. Along the way, we'll explore how Bevy structures game logic using the Entity Component System, covering plugins, components, resources, states, observers, and input handling. By the end, you'll be able to press R, clock to place road nodes, and hit Enter to build a road.

We'll be working through the [RoadToolPlugin](https://github.com/jonas-meyer/planrs/blob/42cee71742525a0e025d23cf262ca4982beb9a33/src/road.rs) code. The [part-1](https://github.com/jonas-meyer/planrs/tree/part-1) tag can be used to follow along.

## App & Plugins

Below is our main.rs, which includes a few imports and the `main()` function which runs our Bevy application.

```rust
mod camera;
mod input;
mod road;

use bevy::prelude::*;
use bevy_enhanced_input::EnhancedInputPlugin;
use camera::CameraPlugin;
use road::RoadToolPlugin;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_plugins(CameraPlugin)
        .add_plugins(EnhancedInputPlugin)
        .add_plugins(RoadToolPlugin)
        .run();
}
```

The important concepts here, is the `App::new()` constructor, the `add_plugins(...)` method and the `run()` method.

Every Bevy application starts by creating an `App` object which implements some core engine features such as the `MainSchedulePlugin` and others depending on the features enabled. This gives us the core structure in which we can add our own game logic and resources.

To compartmentalise our different game systems, we use a concept called `Plugins`. These allow us to bundle data and logic into separate rust modules, keeping these concerns separate, reusable and easier to test in isolation. As an example, if we didn't use plugins in our project, the main function would look like below:

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_plugins(EnhancedInputPlugin)
        // Camera setup
        .init_resource::<CursorWorldPosition>()
        .add_systems(Startup, setup_camera)
        .add_systems(Update, update_cursor_world_position)
        // Road tool setup
        .init_state::<RoadToolState>()
        .init_resource::<CurrentRoad>()
        .add_input_context::<RoadBuilder>()
        .add_systems(Startup, setup_input_context)
        .add_systems(Update, (draw_road_preview, draw_finalized_road))
        .add_systems(OnEnter(RoadToolState::Building), on_enter_building)
        .add_systems(OnExit(RoadToolState::Building), on_exit_building)
        .add_observer(on_toggle_build)
        .add_observer(on_place_node)
        .add_observer(on_confirm_road)
        .add_observer(on_cancel_building)
        .run();
}
```

Imagine having to edit this instead of the clean version given at the top of this section.

The `run()` method will then run the different systems using an event loop and depending on the general setup of the application, open a platform-dependent window and render whatever you have defined.

## Components & Resources

Components and resources are the underlying data structures that can represent entities and global state.

### 1. Components: Per-Entity Data

In our road plugin, we use several components:

```rust
#[derive(Component)]
pub struct Road;

#[derive(Component)]
pub struct RoadNode;
```

These are marker components and therefore contain no data, just identity. They let us find all roads in the world (`Query<&Children, With<Road>>`) and identify which identities are road nodes (`Query<&Transform, With<RoadNode>>`).

We can then spawn these components using the following syntax:

```rust
commands.spawn((Road, Transform::default()))
```

`Transform` is used to define where in the world the component is spawned. It is also possible to spawn components in a parent-child hierarchy as follows:

```rust
commands
    .spawn((Road, Transform::default()))
    .with_children(|parent| {
        for &node in current_road.nodes.iter() {
            parent.spawn((RoadNode, Transform::from_translation(node.extend(0.0))));
        }
    });
```

Here we spawn each node as a child entity to the Road component with the actual node positions. This allowed us to do transformations on the whole road without having to select each node in the road individually (e.g deleting the parent road).

Components can also hold data. For example, a road node could store additional properties in the future:

```rust
#[derive(Component)]
pub struct RoadNode {
    pub width: f32,
    pub speed_limit: u32,
}
```

### 2. Resources: Global Singleton Data

Resources on the other hand are global unique, there's only one instance of each resource type in your application. Use resources when nly one instance should exist (cursor position, current score, settings) and/or multiple systems need to read/write the same shared state.

In our road plugin:

```rust
#[derive(Resource, Default)]
pub struct CurrentRoad {
    pub nodes: Vec<Vec2>,
}
```

`CurrentRoad` is a resource because only one road can be edited at a time.

## Systems & Schedules

In Bevy, systems are just rust functions with special parameters that are passed into the app initialisation. Below is a system that draws roads onto the window:

```rust
fn draw_finalized_road(
    mut gizmos: Gizmos,
    roads: Query<&Children, With<Road>>,
    nodes: Query<&Transform, With<RoadNode>>,
) {
    for children in &roads {
        let nodes: Vec<Vec2> = nodes
            .iter_many(children)
            .map(|t| t.translation.truncate())
            .collect();

        if nodes.len() >= 2 {
            gizmos.linestrip_2d(nodes.iter().copied(), Color::srgb(0.3, 0.5, 0.9));
        }

        for &node in &nodes {
            gizmos.circle_2d(node, 5.0, Color::srgb(0.4, 0.6, 1.0));
        }
    }
}
```

This function is passed a mutable Gizmos object (we will explain this later), and two queries. A query is the idiomatic Bevy way of getting entities from the world. In this case, we query for all entities that have the `Road` marker and specifically get all the children of this component. The children query returns only the entity ids, so we also need to query for the actual node components as well using `Query<&Transform, With<RoadNode>>`. This returns a list of nodes with their location. Using this information we can then draw these on a window.

Systems can be run on different schedules. For the `draw_finalized_road` system, we run this on an `Update` schedule as defined here:

```rust
pub struct RoadToolPlugin;

impl Plugin for RoadToolPlugin {
    fn build(&self, app: &mut App) {
        app.init_state::<RoadToolState>()
            ...
            .add_systems(Update, (draw_road_preview, draw_finalized_road))
            ...
    }
}
```

The update schedule runs every frame and is the most used schedule. Other schedules are also available, such as `Startup` which only runs once at app start, or `OnEnter` / `OnExit` for changes in State which we'll explain next. One can also create their own run conditions which are just functions which return a `bool`.

## State

States are finite state machines that are used to model the structure of your program. They are app-wide and come in three flavours:

1. Standard States
2. SubStates that are children of other states
3. Computed States that are derived from other states

They are mainly used to run systems conditionally, either during a state transition or in a specific state. One can also use them to scope entities to a specific state, allowing for flexible despawning and spawning on entities. For our `RoadPlugin`, we use it as follows:

```rust
#[derive(States, Debug, Clone, Copy, PartialEq, Eq, Hash, Default)]
pub enum RoadToolState {
    #[default]
    Idle,
    Building,
}

impl Plugin for RoadToolPlugin {
    fn build(&self, app: &mut App) {
        app.init_state::<RoadToolState>()
            ...
            .add_systems(OnEnter(RoadToolState::Building), on_enter_building)
            .add_systems(OnExit(RoadToolState::Building), on_exit_building)
            ...
    }
}

fn on_enter_building(mut current_road: ResMut<CurrentRoad>) {
    current_road.clear();
    info!("Entered road building mode. Click to place nodes, Enter to confirm, Escape to cancel.");
}
```

We first create a state which defines our `RoadToolState`, either we're in `Idle`, which is the default state, or we're in `Building` mode. We can then schedule the `on_enter_ building` or `on_exit_building` systems to only run when the state transitions from `Idle` to `Building` or vice-versa. The system itself clears the vector of nodes in the `current_road` resource to prevent allocating nodes to already existing roads.

## Observers & Events

In Bevy, events are a way to announce that something happened, while observers are systems that react to those events. Together they enable loose coupling between different parts of your application.

In a more complicated `RoadToolPlugin`, without events and observers, we would have to directly call or know about other systems. For example, the following `RoadToolPlugin` system:

```rust
fn on_place_node(
    input: Res<ButtonInput<MouseButton>>,
    current_state: Res<State<RoadToolState>>,
    cursor_pos: Res<CursorWorldPosition>,
    mut current_road: ResMut<CurrentRoad>,
    // These systems need to know about ALL consumers of "node placed" events:
    mut audio: ResMut<AudioPlayer>,
    mut undo_stack: ResMut<UndoStack>,
    mut analytics: ResMut<Analytics>,
    mut ui_state: ResMut<UiState>,
) {
    if !input.just_pressed(MouseButton::Left) {
        return;
    }
    if *current_state.get() != RoadToolState::Building {
        return;
    }
    let node_pos = cursor_pos.0;
    current_road.nodes.push(node_pos);
    // Road plugin now has to know about audio, undo, analytics, UI...
    audio.play_sound("node_placed.wav");
    undo_stack.push(UndoAction::PlaceNode(node_pos));
    analytics.track("node_placed");
    ui_state.set_dirty();
}
```

The `RoadToolPlugin` must depend on the `AudioPlugin`, `UndoPlugin`, `AnalyticsPlugin`, `UiPlugin`. We can't add a new feature without having to modify `on_place_node` and we can't reuse the `RoadToolPlugin` without bringing all these dependencies. Further, we would need to mock all dependencies to do any kind of testing.

With observers, all the road tool needs to do is announce what happened.

```rust
#[derive(Event)]
struct NodePlaced {
    position: Vec2,
}
fn on_place_node(
    _trigger: On<Start<PlaceNode>>,
    current_state: Res<State<RoadToolState>>,
    cursor_pos: Res<CursorWorldPosition>,
    mut current_road: ResMut<CurrentRoad>,
    mut commands: Commands,
) {
    if *current_state.get() != RoadToolState::Building {
        return;
    }
    let node_pos = cursor_pos.0;
    current_road.nodes.push(node_pos);
    // Just announce what happened - don't care who's listening
    commands.trigger(NodePlaced { position: node_pos });
}
```

Now other plugins can independently react to this event:

```rust
// In audio.rs - audio plugin doesn't need to know road internals
fn on_node_placed_audio(_trigger: On<NodePlaced>) {
    // play sound
}
```

Each plugin registers its own observer:

```rust
impl Plugin for AudioPlugin {
    fn build(&self, app: &mut App) {
        app.add_observer(on_node_placed_audio);
    }
}
```

Something to keep in mind is that observers run in an undefined order, so if ordering is important, use something like chain events, or have multiple actions in a single observer.

## Input with `bevy_enhanced_input`

Bevy's built-in input handling couples your game logic directly to specific keys or buttons. If you want to jump with `Space`, your jump system checks for `KeyCode::Space`. This becomes problematic when you want to support multiple input methods (keyboard, gamepad, touch), allow rebindable controls or change bindings without recompiling.

The `bevy_enhanced_input` crate solves this by separating actions from bindings. An action is what the player wants to do (e.g., `PlaceNode`, `ToggleBuild`), while a binding is how they do it (e.g., left mouse button, gamepad A button). Your game logic only cares about actions, it observers `On<Start<PlaceNode>>` and doesn't know or care whether that came from a mouse click or a touchscreen tap. Bindings on the other hand, are defined separately and can even be loaded from a configuration file at runtime (we'll get to this), enabling user-customizable controls without touching your game code.

Below are some examples used in the `RoadToolPlugin`:

```rust
// Define WHAT actions exist (game logic cares about these)
#[derive(InputAction)]
#[input_action(output = bool)]
struct PlaceNode;
#[derive(InputAction)]
#[input_action(output = bool)]
struct ToggleBuild;
```

First, we define some actions which can have different output (bool, f32, Vec2) depending on button, single and multi axis actions.

```rust
// Define HOW actions are triggered (configuration, not logic)
let bindings = actions!(RoadBuilder[
    (Action::<ToggleBuild>::new(), bindings![KeyCode::KeyR]),
    (Action::<PlaceNode>::new(), bindings![MouseButton::Left]),
]);
```

Then, we define bindings for those actions. These bindings can be changed at runtime and can come from various sources.

```rust
// Game logic only knows about actions, not bindings
fn on_place_node(
    _trigger: On<Start<PlaceNode>>,  // Don't care if it was mouse, keyboard, or gamepad
    mut current_road: ResMut<CurrentRoad>,
    cursor_pos: Res<CursorWorldPosition>,
) {
    current_road.nodes.push(cursor_pos.0);
}
```

Finally, we use the event that the action triggers, since the crate uses the observer event pattern. For the above example, we run the system whenever the start of a `PlaceNode` occurs, and push the current cursor position to the current_road node vector structure.

## Configuration with RON

In order to allow for loading configuration, such as input keys, we use a data serialization format called RON, which stands for `Rusty Object Notation`. It looks similar to rust notation and supports all of Serde's data model. Have a look at the following:

```rust
(
    toggle_build: Key(KeyR),
    place_node: Mouse(Left),
    confirm_road: Key(Enter),
    cancel_building: Key(Escape),
)
```

This is defined in a file called settings.ron and can be read in using the following helper function:

```rust
pub fn load_config<T: DeserializeOwned + Default>(path: &str) -> T {
    fs::read_to_string(path)
        .ok()
        .and_then(|s| ron::from_str(&s).ok())
        .unwrap_or_default()
}
```

This generic function is used to load the config into the following `RoadInputConfig` struct:

```rust
#[derive(Deserialize)]
struct RoadInputConfig {
    toggle_build: InputBinding,
    place_node: InputBinding,
    confirm_road: InputBinding,
    cancel_building: InputBinding,
}
```

As you can see, it's pretty flexible and self describing. No need to implement specific serializing + deserializing logic for fields, it's all taken care of by RON.

## Rendering with Gizmos

For this example we're using [Gizmos](https://docs.rs/bevy/latest/bevy/prelude/struct.Gizmos.html), which use Bevy's immediate mode for rendering (re-rendered each frame) and are easy to implement for demo / debug purposes. We use the `linestrip_2d` and `circle_2d` method to draw the polylines onto the window each frame. See here:

```rust
if nodes.len() >= 2 {
    gizmos.linestrip_2d(nodes.iter().copied(), Color::srgb(0.3, 0.5, 0.9));
}

for &node in &nodes {
    gizmos.circle_2d(node, 5.0, Color::srgb(0.4, 0.6, 1.0));
}
```

In the future, we'll be using actual meshes and rendering which will allow for more intricate graphics and better performance as we won't need to re-render each frame.

## Putting it all together

Now that we've covered the individual concepts, let's see how they connect in a complete user interaction.

### The Flow

| Input | Event | Observer | Result |
|-------|-------|----------|--------|
| Press `R` | `Start<ToggleBuild>` | `on_toggle_build` | State → `Building`, clears `CurrentRoad` |
| Left Click | `Start<PlaceNode>` | `on_place_node` | Pushes cursor position to `CurrentRoad.nodes` |
| Press `Enter` | `Start<ConfirmRoad>` | `on_confirm_road` | Spawns `Road` + `RoadNode` children |
| Press `Escape` | `Start<CancelBuilding>` | `on_cancel_building` | State → `Idle` |

Each frame, `draw_road_preview` renders the in-progress road from `CurrentRoad`, while `draw_finalized_road` queries all `Road` entities and renders them.

### The Plugin Structure

Here's how these pieces are wired together in the plugin:

```rust
impl Plugin for RoadToolPlugin {
    fn build(&self, app: &mut App) {
        app
            // Data: state machine + in-progress road storage
            .init_state::<RoadToolState>()
            .init_resource::<CurrentRoad>()

            // Input: action-to-binding mappings from config
            .add_input_context::<RoadBuilder>()
            .add_systems(Startup, setup_input_context)

            // Rendering: runs every frame
            .add_systems(Update, (draw_road_preview, draw_finalized_road))

            // State hooks: cleanup on enter/exit
            .add_systems(OnEnter(RoadToolState::Building), on_enter_building)
            .add_systems(OnExit(RoadToolState::Building), on_exit_building)

            // Input observers: react to user actions
            .add_observer(on_toggle_build)
            .add_observer(on_place_node)
            .add_observer(on_confirm_road)
            .add_observer(on_cancel_building);
    }
}
```

The `RoadToolState` gates which actions are valid. The `CurrentRoad` resource holds in-progress nodes until confirmed. On confirmation, nodes become `RoadNode` entities parented to a `Road` entity. The gizmo systems query these components each frame to render both preview and finalized roads.

### What We Built

With these building blocks, we have a functional polyline road tool: press R to toggle build mode, click to place nodes, Enter to confirm, Escape to cancel. The loose coupling between input, state, and rendering means each piece can be extended independently as we add more features.

## Next Steps

For our next steps, we want to continue building out the road tool, by allowing to edit existing roads, implementing simple mesh rendering and gating specific systems behind `bevy_enhanced_input` Contexts. With some luck we might even implement some curved roads!
