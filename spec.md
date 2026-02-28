# Cross-Mod Item Selection Interop Spec v1

## Purpose

Allow Factorio 2.0 mods to broadcast item/fluid selection events so that other mods can track recent selections and offer quick-access buttons.

## Remote Interface

Each participating mod exposes the following functions via `remote.add_interface`:

### Required

#### `get_on_item_selected() -> uint`

Returns the custom event ID for the `on_item_selected` event. The event ID should be lazily generated on first call via `script.generate_event_name()`.

#### `version() -> uint`

Returns the interop spec version (currently `1`).

### Event Payload: `on_item_selected`

Raised via `script.raise_event` when a player selects an item or fluid in the mod's GUI.

| Field | Type | Description |
|-------|------|-------------|
| `player_index` | `uint` | The player who selected the item |
| `item_name` | `string` | The prototype name of the selected item or fluid |

## Auto-Discovery

At `on_init`, `on_load`, and `on_configuration_changed`, each mod iterates `remote.interfaces` and subscribes to any other mod that exposes `get_on_item_selected`:

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

## Re-Entrancy Guard

To prevent echo loops (Mod A selects item -> broadcasts -> Mod B receives -> broadcasts -> Mod A receives...), each mod should use a `handling_external` flag:

```lua
local handling_external = false

local function raise_item_selected(player_index, item_name)
  if handling_external then return end
  script.raise_event(on_item_selected, {
    player_index = player_index,
    item_name = item_name,
  })
end
```

Set `handling_external = true` before processing incoming external events if your mod would re-broadcast as a side effect.

## Recent Items Panel

Each mod maintains a per-player list of the last 5 externally-selected items:

```lua
storage.recent_external_items[player_index] = {
  { item_name = "iron-plate", source = "bottleneck-analyzer" },
  { item_name = "copper-cable", source = "cybersyn" },
  -- ...
}
```

### Storage Rules

- **Deduplication**: If an item already exists in the list, remove the old entry before inserting at the front.
- **Cap**: Maximum 5 entries per player.
- **Persistence**: Stored in `storage` so it survives save/load.
- **Migration**: Initialize `storage.recent_external_items = {}` in `on_init` and ensure it exists in `on_configuration_changed`.

### GUI Rendering

Recent items are displayed as `sprite-button` elements (style `frame_action_button`) embedded in the mod's titlebar. Clicking a recent item selects it in the current mod and broadcasts the selection to other mods.

### Behavior

- External events should **not** automatically change the mod's active filter/selection. They should only add to the recent items list.
- The recent panel updates immediately if the mod's GUI is open when an external event arrives.
- The recent panel is refreshed when the GUI is opened.
- Clicking a recent item navigates to that item within the mod and broadcasts the selection.

## Prototype Validation

Before processing an incoming item name, verify it exists:

```lua
if item_name and (prototypes.item[item_name] or prototypes.fluid[item_name]) then
  -- safe to use
end
```

## Scope

This spec covers **items and fluids only**. Recipes, entities, technologies, and other prototype types are not broadcast. Mods that work with entities (e.g., Recipe Book opening an entity page) should resolve to the corresponding item if possible before broadcasting.
