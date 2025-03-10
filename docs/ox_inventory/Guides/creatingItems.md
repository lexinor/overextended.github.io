---
title: Creating items
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

## Defining item data

Before being able to see or use an item in game it **must** first be defined.

All of the items are defined in the `/data/items.lua` file with key, value pairs.  
Key is the name (not the label) of an item and the value is a table containing the
options for the item.

**Item options:**
<Tabs>
<TabItem value='shared' label='Shared'>

```lua
-- label: string
-- weight: number (optional)
-- stack: boolean (optional) -- If set to false will not allow the item to be stacked
-- degrade: number (optional) -- The amount of time in minutes an item will degrade after
-- close: boolean (optional) -- If set to false will not close the inventory on item use
-- description: string (optional)
-- consume: number (optional) -- Number of an item needed to use it, and removed after use (Default: 1)
-- allowArmed: boolean (optional) -- If set to true will allow use of item while armed with a weapon
-- client: table (optional)
-- buttons: table (optional) -- Allows you to define custom context menu functions for the item
```

</TabItem>
<TabItem value='client' label='Client'>

All values are optional.

```lua
-- event: string -- Event to trigger after item use
-- status: table -- Adjust esx_status values after use
-- anim: table -- Animation during progress bar
    -- dict: string
    -- clip: string
-- prop: table -- Attached prop during progress bar
    -- model: string or hash
    -- pos: table (x, y, z)
    -- rot: table (x, y, z)
-- disable: boolean -- Disables actions during progress bar
-- usetime: number
-- cancel: boolean -- If set to true the player can cancel item use
-- add: function(total) -- Function that triggers when recieving an item (Returns total item count as `total`)
-- remove: function(total) -- Function that triggers when removing an item (Returns total item count as `total`)
```

</TabItem>
<TabItem value='buttons' label='Buttons'>

```lua
-- label: string,
-- action: function(slot) -- Callback function when button is clicked in context menu, returns item slot
```

</TabItem>
</Tabs>

**Examples:**
<Tabs>
<TabItem value='burger' label='Burger'>

```lua
['burger'] = {
    label = 'Burger',
    weight = 220,
    stack = true,
    close = true,
    client = {
        status = { hunger = 200000 },
        anim = { dict = 'mp_player_inteat@burger', clip = 'mp_player_int_eat_burger_fp' },
        prop = {
            model = 'prop_cs_burger_01',
            pos = { x = 0.02, y = 0.02, y = -0.02},
            rot = { x = 0.0, y = 0.0, y = 0.0}
        },
        usetime = 2500,
    }
}
```

</TabItem>
<TabItem value='custom_burger' label='Custom burger'>

A modified burger item, with a description and custom crafting table.

Combined with several new functions and events you could easily create your own crafting system.

```lua
['burger'] = {
    label = 'Burger',
    description = 'Just what is the secret formula?'
    weight = 220,
    stack = true,
    close = true,
    client = {
        status = { hunger = 200000 },
        anim = { dict = 'mp_player_inteat@burger', clip = 'mp_player_int_eat_burger_fp' },
        prop = {
            model = 'prop_cs_burger_01',
            pos = { x = 0.02, y = 0.02, y = -0.02},
            rot = { x = 0.0, y = 0.0, y = 0.0}
        },
        usetime = 2500,
    }
    crafting = {
        ['bun'] = 2,
        ['ketchup'] = 1,
        ['mustard'] = 1,
        ['cheese'] = 1,
        ['pickles'] = 1,
        ['lettuce'] = 1,
        ['tomato'] = 1,
        ['onion'] = 1,
    }
}
```

</TabItem>
<TabItem value='notify_burger' label='Notify burger'>

A modified burger item, which gives you notifications on add and remove arguments.

```lua
['burger'] = {
    label = 'Burger',
    weight = 220,
    stack = true,
    consume = 0,
    client = {
        add = function(total)
            if total > 0 then
                lib.notify({description = 'Nice burger you got there!'})
            end
        end,

        remove = function(total)
            if total < 1 then
                lib.notify({description = 'You lost all of your burgers!'})
            end
        end
    }
}
```

</TabItem>
</Tabs>

## Making the item usable

- If you are using ESX, you can continue using `ESX.RegisterUsableItem`.
- If you are using QBCore, you can continue using `QBCore.Functions.CreateUseableItem`.

Using the built-in system is more secure and provides much more functionality.

### Client callbacks

Item callbacks can be added by defining an export (recommended), or by adding it to [items/client.lua](https://github.com/overextended/ox_inventory/blob/main/modules/items/client.lua#L33).

When defining [item data](https://github.com/overextended/ox_inventory/blob/main/data/items.lua#L11), adding client.export will trigger an event on item use.  
The correct formatting is `export = resourceName.exportName`.

```lua
exports('bandage', function(data, slot)
    local playerPed = PlayerPedId()
    local maxHealth = GetEntityMaxHealth(playerPed)
    local health = GetEntityHealth(playerPed)

    -- Does the ped need to heal? We can cancel the item from being used.
    if health < maxHealth then
        -- Triggers internal-code to correctly use items.
        -- This adds security, removes the item on use, adds progressbar support, and is necessary for server callbacks.
        exports.ox_inventory:useItem(data, function(data)
            -- The server has verified the item can be used.
            if data then
                SetEntityHealth(playerPed, math.min(maxHealth, math.floor(health + maxHealth / 16)))
                lib.notify({description = 'You feel better already'})
            end
        end)
    else
        -- Don't use the item
        lib.notify({type = 'error', description = 'You don\'t need a bandage right now'})
    end
end)
```

### Server callbacks

A callback function can be defined on the server to handle several events (usingItem, usedItem, buyItem).  
This can either be an export (recommended), or added to the bottom of [items/server.lua](https://github.com/overextended/ox_inventory/blob/main/modules/items/server.lua). 
When defining [item data](https://github.com/overextended/ox_inventory/blob/main/data/items.lua#L14), adding server.export will trigger an event for the actions above.
The correct formatting is `export = resourceName.exportName`.

```lua
exports('bandage', function(event, item, inventory, slot, data)
    -- Player is attempting to use the item.
    if event == 'usingItem' then
        local playerPed = GetPlayerPed(inventory.id)
        local maxHealth = GetEntityMaxHealth(playerPed)
        local health = GetEntityHealth(playerPed)

        -- Check if the player needs to be healed.
        if health >= maxHealth then
            TriggerClientEvent('ox_lib:notify', inventory.id, {type = 'error', description = 'You don\'t need a bandage right now'})

            -- Returning 'false' will prevent the item from being used
            return false
        end

        return
    end

    -- Player has finished using the item.
    if event == 'usedItem' then
        return TriggerClientEvent('ox_lib:notify', inventory.id, {description = 'You feel better already'})
    end

    -- Player is attempting to purchase the item.
    if event == 'buying' then
        return TriggerClientEvent('ox_lib:notify', inventory.id, {type = 'success', description = 'You bought a bandage'})
    end
end)
```

## Creating container items

Like with other items the item must first be registered.

**Example:**

```lua
['paperbag'] = {
    label = 'Paper Bag',
    weight = 1,
    stack = false,
    close = false,
    consume = 0
},
```

When registered you can define the item as a container under the `Items.containers` table in `/modules/items/sever.lua`  
The key for the container is the `name` you gave it when registering the item.  
You can also define the number of slots, the maximum weight, blacklist and whitelist items.

```lua
['name'] = {
    -- size: {slots, maxWeight}
        -- slots: number
        -- maxWeight: number
    -- blacklist: table (optional)
        -- ['itemName'] = true
    -- whitelist: table (optional)
        -- ['itemName'] = true
}
```

**Example:**

```lua
['paperbag'] = {
    size = {5, 1000},
    blacklist = {
        ['testburger'] = true -- No burgers!
    }
},
```
