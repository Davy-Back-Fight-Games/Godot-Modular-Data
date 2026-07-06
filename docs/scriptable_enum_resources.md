# Scriptable Enum Resources Plan

This plan adapts Ryan Hipple's ScriptableObject enum idea to Godot `Resource` assets.

Hipple's broader ScriptableObject architecture uses assets as stable, inspectable project-level dependencies instead of hard-coded scene references, singleton lookups, or primitive constants. In the Unite 2017 talk, the relevant enum idea is "No More Enums": replace code-defined enum values with one asset per option. Code compares asset identity, while designers can add new options by creating resources instead of editing scripts.

This fits the addon because `addons/modular_data/runtime/` implements the same family of patterns for Variables, References, and RuntimeSets.

## Goal

Add a resource-backed enum system for categories, tags, states, types, and other named game concepts that should be data-driven.

Examples:

- Damage types: Fire, Ice, Physical.
- Teams/factions: Player, Enemy, Neutral.
- Skill schools: Holy, Arcane, Martial.
- Item rarity: Common, Rare, Legendary.
- Gameplay states where values are content-facing rather than control-flow-only.

## Non-Goals

- Do not replace local implementation enums such as `SkillDefinition.CastMode` or `FilteredRuntimeSetTargeting.SortMode` yet. Those drive fixed code branches and still fit normal GDScript enums.
- Do not build a full editor database in the first pass.
- Do not serialize enum values as strings unless external save data requires it.

## Core Model

Use `res://addons/modular_data/runtime/scriptable_enum.gd`:

```gdscript
@tool
class_name ScriptableEnum
extends Resource

@export var display_name: String = ""
@export_multiline var description: String = ""
@export var debug_log: bool = false

func get_label() -> String:
	if display_name != "":
		return display_name
	if resource_path != "":
		return resource_path.get_file().get_basename().capitalize()
	return "<unnamed enum>"
```

This gives every enum option a stable asset identity and the same metadata shape as `FloatVariable` and `RuntimeSet`.

For stronger Inspector filtering, add small subclasses for each enum family only when useful:

```gdscript
@tool
class_name TeamId
extends ScriptableEnum
```

```gdscript
@tool
class_name DamageType
extends ScriptableEnum
```

Then exported fields can be typed directly:

```gdscript
@export var team: TeamId
@export var damage_type: DamageType
@export var allowed_damage_types: Array[DamageType] = []
```

## Comparison Rules

Use resource identity for normal comparisons:

```gdscript
if incoming_damage.damage_type == fire_damage_type:
	# fire-specific behavior
```

For collection membership:

```gdscript
if allowed_damage_types.has(incoming_damage.damage_type):
	# accepted
```

Avoid comparing by `display_name` or file name. Those are editable labels, not identity.

## Folder Layout

Core scripts:

- `res://addons/modular_data/runtime/scriptable_enum.gd`

Game-specific enum-family scripts:

- `res://data/enums/team_id.gd`
- Feature-specific enum scripts can live beside the feature that owns them.

Shared enum assets:

- `res://data/enums/teams/enemy.tres`
- `res://data/enums/damage_types/fire.tres`
- `res://data/enums/damage_types/ice.tres`

This keeps enum assets grouped clearly while leaving addon scripts under `res://addons/modular_data/runtime/`.

## First Implementation Slice

1. Add `ScriptableEnum` base resource.
2. Add one concrete enum family that solves a real project need, such as `TeamId`.
3. Create project assets for the enum values currently needed by gameplay.
4. Update consumers from strings or primitive constants to exported scriptable enum resources.
5. Add project-specific docs that explain where enum families and assets live.

## Migration Pattern

String-based code often looks like:

```gdscript
@export var team: StringName = &"enemy"
```

Resource-backed code should become:

```gdscript
@export var team: TeamId
```

Filters should become:

```gdscript
@export var allowed_teams: Array[ScriptableEnum] = []
@export var team_property: StringName = &"team"

func accepts_target(_caster: Node, _skill: Resource, target: Node) -> bool:
	if allowed_teams.is_empty():
		return true
	var target_team = target.get(team_property)
	return target_team is ScriptableEnum and allowed_teams.has(target_team)
```

Only add backward compatibility for string identifiers if saved scenes, assets, or external data already rely on them.

## Optional `EnumReference`

For parity with Variables/References, we can add a reference wrapper later:

```gdscript
@tool
class_name ScriptableEnumReference
extends Resource

@export var value: ScriptableEnum

func get_value() -> ScriptableEnum:
	return value
```

Do not add this in the first pass unless a real use case appears. Unlike numeric variables, enum options do not need constant-vs-variable behavior because the asset itself is already the constant.

## Editor Tooling Later

The first pass should rely on Godot's native resource picker and typed exported fields.

Later tooling ideas:

- A dock listing enum families and their assets.
- A creator button for new enum assets under the correct folder.
- A usage scanner similar to the event registry for `.tscn` and `.tres` references.
- Validation that enum-family arrays do not mix base `ScriptableEnum` assets from unrelated families.

## Pitfalls

- Resource identity is path/reference based; duplicate `.tres` files are distinct enum values even if they share labels.
- Use typed subclasses where possible so the Inspector prevents assigning the wrong enum family.
- Do not use scriptable enum resources for fixed algorithm branches unless designers are expected to add new values without code changes.
- For save games or backend data, store a stable external id separately if needed; do not rely on translated display names.

## Acceptance Criteria

- Designers can create a new enum option by creating a `.tres` resource, not editing code.
- Scripts can export typed enum resources and arrays of typed enum resources.
- Equality and membership checks use resource identity.
- At least one existing string-based category is migrated in the consuming project to demonstrate the pattern.
- Documentation explains when to use resource enums vs normal GDScript enums.
