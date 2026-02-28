# Cross-Mod Item Selection Interop Spec v1

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Purpose

Allow Factorio 2.0 mods to broadcast item/fluid selection events so that other mods can track recent selections and offer quick-access buttons.

## Remote Interface

Mods that publish item selections MUST expose the following functions via `remote.add_interface`:

### `get_on_item_selected() -> uint`

Returns the custom event ID for the `on_item_selected` event.

### `interop_version() -> uint`

Returns the interop spec version. Implementations of this version of the spec MUST return `1`.

## `on_item_selected` Event Payload

Raised via `script.raise_event` when a player selects an item or fluid in the mod's GUI.

| Field | Type | Description |
|-------|------|-------------|
| `player_index` | `uint` | REQUIRED. The player who selected the item. |
| `item_name` | `string` | REQUIRED. The prototype name of the selected item or fluid. |

## Auto-Discovery

Mods that wish to receive external item selections MUST subscribe to other mods' events at `on_init`, `on_load`, and `on_configuration_changed`. Mods MUST iterate `remote.interfaces` and subscribe to any mod that exposes `get_on_item_selected`, skipping their own interface:

```lua
for iface, functions in pairs(remote.interfaces) do
  if iface == "my-mod" then goto continue end
  if functions["get_on_item_selected"] then
    local event_id = remote.call(iface, "get_on_item_selected")
    if event_id then
      script.on_event(event_id, function(e)
        -- handle e.player_index, e.item_name
      end)
    end
  end
  ::continue::
end
```

## Handling External Events

External events MUST NOT automatically change the mod's active filter or selection, and MUST NOT trigger a re-broadcast of `on_item_selected`. How a mod uses incoming events (e.g., displaying a recent items panel) is an implementation detail.

## Prototype Validation

Mods MUST validate incoming item names before processing them:

```lua
if item_name and (prototypes.item[item_name] or prototypes.fluid[item_name]) then
  -- safe to use
end
```

## Scope

This spec covers items and fluids only. Recipes, entities, technologies, and other prototype types MUST NOT be broadcast. Mods that work with entities (e.g., Recipe Book opening an entity page) SHOULD resolve to the corresponding item if possible before broadcasting.

## Versioning

The spec version (`1`) MUST only be incremented for breaking changes: new required fields in the event payload, new required remote functions, or changes to existing behavior.

Additive changes (new optional events like `on_time_slice_changed`) do not require a version bump. Mods SHOULD check for the presence of specific functions (e.g., `functions["get_on_time_slice_changed"]`) rather than relying on the version number for optional features.
