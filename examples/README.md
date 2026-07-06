# Examples

This repository keeps examples intentionally lightweight. Create `.tres` resources from the addon classes and assign them through exported fields.

## FloatReference

```gdscript
extends Node2D

@export var move_speed: FloatReference

func _physics_process(delta: float) -> void:
	position.x += move_speed.get_value() * delta
```

## RuntimeSet

Add `RuntimeSetMember` to a node, assign a `RuntimeSet` resource, and leave `auto_register` enabled. The node registers on `_enter_tree()` and unregisters on `_exit_tree()`.
