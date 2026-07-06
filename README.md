# Godot Modular Data

Godot Modular Data is a small addon that provides Resource-based shared data patterns for Godot projects:

- Variable resources for shared mutable values.
- Reference resources for constant-or-variable exported fields.
- Runtime sets for transient lists of live scene nodes.
- Scriptable enum resources for data-driven categories and identities.

## Install

Copy `addons/modular_data` into a Godot 4 project, then enable **Project > Project Settings > Plugins > Modular Data**.

The runtime scripts live at `res://addons/modular_data/runtime/` and expose `class_name` types, so they can be used directly in exported fields without preloads.

## Documentation

- `docs/modular_data.md`
- `docs/scriptable_enum_resources.md`
- `examples/README.md`

## License

MIT. See `LICENSE`.
