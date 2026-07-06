# Modular Data Core

The modular data core is a Godot `Resource`-based shared data system inspired by Ryan Hipple's ScriptableObject modular data pattern from [Game Architecture with Scriptable Objects](https://youtu.be/raQ3iHhE_Kk?si=UyB4W0lxhlAKDVjF&t=925). It adapts that idea from Unity ScriptableObjects to Godot Resources, letting scene scripts depend on exported data assets instead of hard-coded singleton lookups or scene-specific config nodes.

Core scripts live in `res://addons/modular_data/runtime/`.

## Variables

Variable resources are shared value assets:

- `FloatVariable`
- `IntVariable`
- `BoolVariable`
- `StringVariable`
- `Vector2Variable`

Use a Variable when multiple nodes should read, write, or observe the same value.

- `initial_value` is the intended starting value.
- `value` is the current mutable value.
- `value_changed(value)` emits when `value` changes.
- `reset_to_initial_value()` copies `initial_value` into `value`.
- `set_value(new_value)` writes the current value and emits if changed.
- `set_value_from(variable)` copies another variable's current `value`.
- `apply_change(amount)` and `apply_change_from(variable)` exist only on `FloatVariable`, `IntVariable`, and `Vector2Variable`.

## References

Reference resources wrap either a local constant or a shared Variable:

- `FloatReference`
- `IntReference`
- `BoolReference`
- `StringReference`
- `Vector2Reference`

Reference fields:

- `use_constant`: when `true`, use `constant_value`.
- `constant_value`: local value stored on the Reference resource.
- `variable`: shared Variable asset used when `use_constant` is `false`.

Use a Reference for tunable inputs that may be either a constant or a shared data asset. Use a Variable directly when the script owns or mutates shared state and needs signals or reset behavior.

The addon currently relies on Godot's native Inspector UI for resource fields.

## RuntimeSet

`RuntimeSet` is a resource-owned, transient list of live scene `Node`s. It is useful for groups like enemies, interactables, targets, or active objectives without a singleton manager.

`RuntimeSetMember` is a `Node` helper. Add it to the node that should appear in the set and assign a `RuntimeSet`; it registers itself on `_enter_tree()` and unregisters on `_exit_tree()` when `auto_register` is enabled.

RuntimeSet API:

- `item_added(item)` emits when a node is added.
- `item_removed(item)` emits when a node is removed.
- `changed` emits after add, remove, clear, or invalid-item pruning.
- `get_items()` returns a duplicate `Array[Node]`.
- `size()` returns the current count.
- `clear()` removes all current items.
- `has_item(item)` checks current membership.

## ScriptableEnum

`ScriptableEnum` is a base resource for data-driven enum values. Each option is a `.tres` asset, so code can compare resource identity instead of string names or primitive constants.

Use a small typed subclass when Inspector filtering matters:

```gdscript
@tool
class_name TeamId
extends ScriptableEnum
```

Export typed enum resources directly:

```gdscript
@export var team: TeamId
@export var allowed_teams: Array[ScriptableEnum] = []

func is_allowed(target_team: TeamId) -> bool:
	return allowed_teams.has(target_team)
```

Example `TeamId` resources can live under a project folder such as `res://data/enums/teams/`.

## Common Usage

Reading a Reference value:

```gdscript
extends Node

@export var move_speed: FloatReference

func _physics_process(delta: float) -> void:
	position.x += move_speed.get_value() * delta
```

Registering with a `RuntimeSetMember` in a scene:

```text
Enemy
script: RuntimeSetMember
runtime_set: enemies_runtime_set.tres
```

Leave `auto_register` enabled for automatic enter/exit tree registration. If `RuntimeSetMember` is added as a child helper node instead, the child helper node is what gets stored in the set.

Iterating a RuntimeSet:

```gdscript
@export var enemies: RuntimeSet

func print_enemies() -> void:
	for enemy in enemies.get_items():
		print(enemy.name)
```

## Asset Organization

Place shared data resources under a consistent project folder such as `res://data/` unless a feature needs a more specific folder. Group shared enum assets by family, for example `res://data/enums/teams/enemy.tres`.

Reset runtime values on scene or session start when deterministic state matters:

```gdscript
@export var player_hp: FloatVariable

func _ready() -> void:
	player_hp.reset_to_initial_value()
```

## Notes And Pitfalls

- Resource assets are shared by every node that references them.
- Compare ScriptableEnum values by resource identity, not by `display_name` or file name.
- RuntimeSet membership is transient and is not exported or persisted.
- A `.tres` value may retain editor or runtime changes depending on how it is edited; use `initial_value` and `reset_to_initial_value()` explicitly for deterministic starts.
- Do not use Resource variables for per-instance state unless sharing is intended.
