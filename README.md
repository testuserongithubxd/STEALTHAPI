# Stealth Lua API Documentation

> **Version:** 1337  
> **Runtime:** Isolated Lua Environment  
> **Execution:** All code runs in a fully isolated environment with its own globals and state. Your scripts are invisible to other resources and the server.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Authentication](#authentication)
3. [Native Hooking](#native-hooking)
4. [Resource Injection](#resource-injection)
5. [Input System](#input-system)
6. [Notifications](#notifications)
7. [Drawing](#drawing)
8. [Image System](#image-system)
9. [HTTP Requests](#http-requests)
10. [Virtual Key Constants](#virtual-key-constants)
11. [Full Examples](#full-examples)

---

## Getting Started

All Stealth API functions are accessed through the global `Stealth` table. The API is automatically available when you execute code in Isolated mode. You do not need to require or import anything.

```lua
print(Stealth.GetVersion())
```

**Output:**
```
1337
```

If you need a persistent loop (for drawing, input checking, or timed logic), wrap your code in `Citizen.CreateThread`:

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- your per-frame code here
    end
end)
```

**Important:** `Wait(0)` yields every frame. `Wait(1000)` yields for one second. Never run an infinite loop without `Wait` or you will freeze the game.

Use the **Reset** button in the editor to stop all running loops and recreate the environment cleanly.

---

## Authentication

### `Stealth.GetVersion()`

Returns the current Stealth API version.

**Returns:** `integer`

```lua
local version = Stealth.GetVersion()
print("Running Stealth v" .. tostring(version))
```

---

### `Stealth.GetKey()`

Returns the user's unique license key as a string.

**Returns:** `string`

```lua
local key = Stealth.GetKey()
print("License: " .. key)
```

---

### `Stealth.getAuthKey()`

Alias for `Stealth.GetKey()`.

```lua
local key = Stealth.getAuthKey()
```

---

### `Stealth.GetUserID()`

Returns a unique numeric ID derived from the license key. Always the same for the same key. Useful for identifying users without exposing the full key.

**Returns:** `integer`

```lua
local uid = Stealth.GetUserID()
print("User ID: " .. tostring(uid))
```

**Example: Key-based authentication against a remote whitelist**

```lua
local userKey = Stealth.getAuthKey()
local keyList = Stealth.FetchContent("https://pastebin.com/raw/XXXXXXXX")

if keyList and string.find(keyList, userKey) then
    print("Key is authenticated [" .. userKey .. "]")
else
    print("Key is not in the list [" .. userKey .. "]")
end
```

---

## Native Hooking

The hooking system lets you intercept any FiveM native function. When a hooked native is called by any other resource on the server, your callback fires. You can read the arguments, modify the return value, or block the original call entirely.

**Your own code is never affected by your hooks.** When you call a hooked native from inside the isolated environment, you always get the real unmodified result.

### `Stealth.HookNative(hash, callback)`

Registers a hook on a native function.

**Parameters:**
- `hash` — `integer` — the 64-bit native hash (e.g. `0x6D0DE6A7B5DA71F8` for `GetPlayerName`)
- `callback` — `function` — called every time the native fires from another resource

**Returns:** `boolean` — `true` if the hook was registered, `false` if max hooks reached (limit: 8)

**Callback return values:**
- Return nothing or `true` — the original result is unchanged
- Return `false` — skip the original result
- Return `false, value` — override the result with `value`
  - If `value` is a `string`, the native returns that string
  - If `value` is a `number`, the native returns that integer

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return false, "not-real-name"
end)
```

After this, every resource that calls `GetPlayerName(PlayerId())` receives `"not-real-name"` instead of the real name. Your own code still sees the real name.

---

### `Stealth.UnhookNative(hash)`

Removes a previously registered hook.

**Parameters:**
- `hash` — `integer` — the native hash to unhook

**Returns:** `boolean` — `true` if found and removed

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return false, "not-real-name"
end)

Stealth.UnhookNative(0x6D0DE6A7B5DA71F8)
```

---

### `Stealth.GetArg(index)`

Reads an integer argument from the currently executing hooked native. Only valid inside a hook callback.

**Parameters:**
- `index` — `integer` — the argument index (0-based)

**Returns:** `integer`

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    local playerId = Stealth.GetArg(0)
    print("GetPlayerName called for player ID: " .. tostring(playerId))
    return false, "not-real-name"
end)
```

---

### `Stealth.GetArgFloat(index)`

Reads a float argument from the currently executing hooked native. Only valid inside a hook callback.

**Parameters:**
- `index` — `integer` — the argument index (0-based)

**Returns:** `number`

```lua
Stealth.HookNative(0xSOME_NATIVE, function()
    local x = Stealth.GetArgFloat(0)
    local y = Stealth.GetArgFloat(1)
    local z = Stealth.GetArgFloat(2)
    print("Position: " .. x .. ", " .. y .. ", " .. z)
    return true
end)
```

---

### Common Native Hashes

| Native | Hash | Description |
|--------|------|-------------|
| `GetPlayerName` | `0x6D0DE6A7B5DA71F8` | Returns player display name |
| `GetEntityCoords` | `0x3FEF770D40960D5A` | Returns entity position |
| `GetEntityHealth` | `0xEEF059FAD016D209` | Returns entity health |
| `GetVehicleNumberPlateText` | `0x7CE1CCB9B293020E` | Returns plate text |
| `GetPlayerServerId` | `0x4D97BCC7` | Returns server ID |

Full native reference: [FiveM Native Reference](https://docs.fivem.net/natives/)

---

## Resource Injection

### `Stealth.InjectResource(resourceName, code)`

Executes Lua code inside another resource's environment. The code runs as if it was part of that resource, with access to that resource's globals, events, and state.

**Parameters:**
- `resourceName` — `string` — the target resource name, or `"any"` for a random safe resource
- `code` — `string` — the Lua code to execute

**Returns:** `boolean`

```lua
Stealth.InjectResource("any", [[
    print("Hello from " .. GetCurrentResourceName())
]])
```

```lua
Stealth.InjectResource("monitor", [[
    print("Injected into monitor!")
    print("Player name: " .. GetPlayerName(PlayerId()))
]])
```

**Notes:**
- Injected code does **not** have access to the `Stealth` table — it runs in another resource's environment
- Injected code **is** affected by your native hooks (unlike your own code)
- Use `"any"` to auto-select a safe resource (avoids known anticheat resources)
- Maximum code length is approximately 4000 characters per injection

---

## Input System

The input system detects keyboard and mouse input independently of FiveM's control system. Keys are detected even when the game normally blocks input.

### `Stealth.IsControlPressed(vk)`

Returns `true` every frame the key is held down.

**Parameters:**
- `vk` — `integer` — a Virtual Key code (see [Virtual Key Constants](#virtual-key-constants))

**Returns:** `boolean`

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlPressed(VK_SHIFT) then
            print("SHIFT is being held!")
        end
    end
end)
```

---

### `Stealth.IsControlJustPressed(vk)`

Returns `true` only once when the key transitions from released to pressed. Does not repeat while held.

**Parameters:**
- `vk` — `integer` — a Virtual Key code

**Returns:** `boolean`

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_F5) then
            print("F5 pressed once!")
        end
    end
end)
```

---

### `Stealth.GetCurrentPressedKey()`

Returns the first key currently being held down, with its name.

**Returns:** `integer, string` — VK code and key name, or `nil` if no key pressed

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        local vk, name = Stealth.GetCurrentPressedKey()
        if vk then
            print("Holding: " .. name)
        end
    end
end)
```

---

### `Stealth.GetCurrentJustPressedKey()`

Returns the first key just pressed this frame (single-fire). Does not repeat while held.

**Returns:** `integer, string` — VK code and key name, or `nil` if no new press

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        local vk, name = Stealth.GetCurrentJustPressedKey()
        if vk then
            print("Just pressed: " .. name)
        end
    end
end)
```

---

### Input Comparison

| Function | Fires when | Use case |
|----------|-----------|----------|
| `IsControlPressed` | Every frame the key is held | Movement, continuous actions |
| `IsControlJustPressed` | Once on key-down | Toggle menu, single actions |
| `GetCurrentPressedKey` | Every frame any key held | Display held key |
| `GetCurrentJustPressedKey` | Once per new press | Keybind selection |

---

## Notifications

### `Stealth.AddNotification(message, type)`

Displays a notification popup.

**Parameters:**
- `message` — `string` — the notification text
- `type` — `integer` — the notification style (optional, defaults to `0`)

**Notification types:**
- `Stealth.NOTIFY_INFO` (0) — neutral
- `Stealth.NOTIFY_SUCCESS` (1) — green
- `Stealth.NOTIFY_WARNING` (2) — yellow
- `Stealth.NOTIFY_ERROR` (3) — red

**Returns:** `boolean`

```lua
Stealth.AddNotification("Script loaded", Stealth.NOTIFY_SUCCESS)
Stealth.AddNotification("Low health!", Stealth.NOTIFY_WARNING)
Stealth.AddNotification("Connection failed", Stealth.NOTIFY_ERROR)
Stealth.AddNotification("Scanning...", Stealth.NOTIFY_INFO)
```

---

## Drawing

Drawing functions render directly on screen. All coordinates use normalized screen space: `0.0` is left/top, `1.0` is right/bottom. Center of screen is `0.5, 0.5`.

**Drawing must happen every frame inside a `Citizen.CreateThread` loop with `Wait(0)`.**

### `Stealth.DrawRect(x, y, width, height, r, g, b, a)`

Draws a filled rectangle.

**Parameters:**
- `x` — `number` — center X (0.0 to 1.0)
- `y` — `number` — center Y (0.0 to 1.0)
- `width` — `number` — width (0.0 to 1.0)
- `height` — `number` — height (0.0 to 1.0)
- `r, g, b` — `integer` — color (0-255)
- `a` — `integer` — alpha (0-255, optional, defaults to 255)

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        Stealth.DrawRect(0.5, 0.5, 0.3, 0.1, 0, 0, 0, 180)
    end
end)
```

---

## Image System

### `Stealth.LoadImage(url, width, height)`

Loads an image from a URL. Returns an image handle for use with `DrawImage`.

**Parameters:**
- `url` — `string` — direct URL to the image file
- `width` — `integer` — texture width in pixels (optional, defaults to 256)
- `height` — `integer` — texture height in pixels (optional, defaults to 256)

**Returns:** `table` — image handle with `loaded` field indicating success

```lua
local banner = Stealth.LoadImage("https://i.imgur.com/example.png", 512, 128)
```

**Note:** Images load asynchronously. Wait at least 500ms-1000ms after loading before drawing.

---

### `Stealth.DrawImage(image, x, y, width, height, alpha)`

Draws a previously loaded image.

**Parameters:**
- `image` — `table` — image handle from `LoadImage`
- `x` — `number` — center X (0.0 to 1.0)
- `y` — `number` — center Y (0.0 to 1.0)
- `width` — `number` — display width (0.0 to 1.0)
- `height` — `number` — display height (0.0 to 1.0)
- `alpha` — `integer` — opacity (0-255, optional, defaults to 255)

**Returns:** `boolean` — `true` if drawn

```lua
local banner = Stealth.LoadImage("https://i.imgur.com/example.png", 512, 128)

Citizen.CreateThread(function()
    Wait(1000)
    while true do
        Wait(0)
        Stealth.DrawRect(0.5, 0.15, 0.3, 0.07, 10, 10, 10, 200)
        Stealth.DrawImage(banner, 0.5, 0.15, 0.28, 0.06, 255)
    end
end)
```

---

## HTTP Requests

### `Stealth.FetchContent(url)`

Performs an HTTP GET request and returns the response body.

**Parameters:**
- `url` — `string` — the full URL to fetch

**Returns:** `string` — the response body, or `nil` on failure

**Note:** This blocks while the request completes. Use for small requests only (config files, key lists, short API responses).

```lua
local data = Stealth.FetchContent("https://pastebin.com/raw/XXXXXXXX")
if data then
    print("Got " .. #data .. " bytes")
    print(data)
else
    print("Request failed")
end
```

**Example: Load and execute remote script**

```lua
local script = Stealth.FetchContent("https://pastebin.com/raw/YYYYYYYY")
if script then
    load(script)()
end
```

---

## Virtual Key Constants

All standard Windows Virtual Key codes are available as global constants.

### Common Keys

| Constant | Value | Key |
|----------|-------|-----|
| `VK_BACK` | `0x08` | Backspace |
| `VK_TAB` | `0x09` | Tab |
| `VK_RETURN` | `0x0D` | Enter |
| `VK_SHIFT` | `0x10` | Shift (either) |
| `VK_CONTROL` | `0x11` | Ctrl (either) |
| `VK_MENU` | `0x12` | Alt (either) |
| `VK_PAUSE` | `0x13` | Pause |
| `VK_CAPITAL` | `0x14` | Caps Lock |
| `VK_ESCAPE` | `0x1B` | Escape |
| `VK_SPACE` | `0x20` | Spacebar |

### Navigation Keys

| Constant | Value | Key |
|----------|-------|-----|
| `VK_PRIOR` | `0x21` | Page Up |
| `VK_NEXT` | `0x22` | Page Down |
| `VK_END` | `0x23` | End |
| `VK_HOME` | `0x24` | Home |
| `VK_LEFT` | `0x25` | Left Arrow |
| `VK_UP` | `0x26` | Up Arrow |
| `VK_RIGHT` | `0x27` | Right Arrow |
| `VK_DOWN` | `0x28` | Down Arrow |
| `VK_SNAPSHOT` | `0x2C` | Print Screen |
| `VK_INSERT` | `0x2D` | Insert |
| `VK_DELETE` | `0x2E` | Delete |

### Function Keys

| Constant | Value | Key |
|----------|-------|-----|
| `VK_F1` — `VK_F12` | `0x70` — `0x7B` | F1 through F12 |

### Numpad Keys

| Constant | Value | Key |
|----------|-------|-----|
| `VK_NUMPAD0` — `VK_NUMPAD9` | `0x60` — `0x69` | Numpad 0-9 |
| `VK_MULTIPLY` | `0x6A` | Numpad * |
| `VK_ADD` | `0x6B` | Numpad + |
| `VK_SUBTRACT` | `0x6D` | Numpad - |
| `VK_DECIMAL` | `0x6E` | Numpad . |
| `VK_DIVIDE` | `0x6F` | Numpad / |
| `VK_NUMLOCK` | `0x90` | Num Lock |

### Modifier Keys

| Constant | Value | Key |
|----------|-------|-----|
| `VK_LSHIFT` | `0xA0` | Left Shift |
| `VK_RSHIFT` | `0xA1` | Right Shift |
| `VK_LCONTROL` | `0xA2` | Left Ctrl |
| `VK_RCONTROL` | `0xA3` | Right Ctrl |
| `VK_LMENU` | `0xA4` | Left Alt |
| `VK_RMENU` | `0xA5` | Right Alt |

### Mouse Buttons

| Constant | Value | Key |
|----------|-------|-----|
| `VK_LBUTTON` | `0x01` | Left Mouse |
| `VK_RBUTTON` | `0x02` | Right Mouse |
| `VK_MBUTTON` | `0x04` | Middle Mouse |

### Letter and Number Keys

Letters A-Z use their ASCII values: `0x41` (A) through `0x5A` (Z).  
Numbers 0-9 use: `0x30` (0) through `0x39` (9).

```lua
if Stealth.IsControlJustPressed(0x42) then
    print("B key pressed!")
end
```

---

## Full Examples

### Example 1: Name Spoofer

Replaces your name for all other resources on the server.

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    local pid = Stealth.GetArg(0)
    if pid == PlayerId() then
        return false, "not-real-name"
    end
    return true
end)

Stealth.AddNotification("Name spoofer active", Stealth.NOTIFY_SUCCESS)
```

---

### Example 2: Toggle System with Keybind

```lua
local godMode = false

Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_F5) then
            godMode = not godMode
            Stealth.AddNotification("God Mode: " .. (godMode and "ON" or "OFF"),
                godMode and Stealth.NOTIFY_SUCCESS or Stealth.NOTIFY_INFO)
        end
        if godMode then
            local ped = PlayerPedId()
            SetEntityHealth(ped, 200)
        end
    end
end)
```

---

### Example 3: Remote Key Authentication

```lua
local userKey = Stealth.getAuthKey()
local userId = Stealth.GetUserID()

local keyList = Stealth.FetchContent("https://pastebin.com/raw/XXXXXXXX")

if keyList and string.find(keyList, userKey) then
    print("Key is authenticated [" .. userKey .. "]")
    Stealth.AddNotification("Welcome! ID: " .. tostring(userId), Stealth.NOTIFY_SUCCESS)
else
    print("Key is not in the list [" .. userKey .. "]")
    Stealth.AddNotification("Access denied", Stealth.NOTIFY_ERROR)
end
```

---

### Example 4: Resource Injection

```lua
Stealth.InjectResource("hardcap", [[
    print("Hello from hardcap!")
    print("Name: " .. GetPlayerName(PlayerId()))
]])

Stealth.InjectResource("any", [[
    local ped = PlayerPedId()
    local hp = GetEntityHealth(ped)
    print("Health: " .. tostring(hp))
]])
```

---

### Example 5: Hook with Argument Logging

```lua
local callCount = 0

Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    callCount = callCount + 1
    local pid = Stealth.GetArg(0)
    print("GetPlayerName #" .. callCount .. " for player " .. tostring(pid))
    return false, "Player_" .. tostring(pid)
end)
```

---

### Example 6: Image Banner

```lua
local banner = Stealth.LoadImage("https://i.imgur.com/XXXXXXX.png", 512, 128)

Citizen.CreateThread(function()
    Wait(1500)
    while true do
        Wait(0)
        Stealth.DrawRect(0.5, 0.12, 0.3, 0.07, 10, 12, 25, 210)
        if banner and banner.loaded then
            Stealth.DrawImage(banner, 0.5, 0.12, 0.28, 0.06, 255)
        end
        if Stealth.IsControlJustPressed(VK_ESCAPE) then
            break
        end
    end
end)
```

---

### Example 7: Keybind Listener

```lua
Citizen.CreateThread(function()
    print("Press any key...")
    while true do
        Wait(0)
        local vk, name = Stealth.GetCurrentJustPressedKey()
        if vk then
            print("Key: " .. name .. " | VK: 0x" .. string.format("%02X", vk))
            Stealth.AddNotification("Pressed: " .. name, Stealth.NOTIFY_INFO)
        end
    end
end)
```

---

### Example 8: Coordinate HUD

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        local ped = PlayerPedId()
        local coords = GetEntityCoords(ped)
        local label = string.format("X: %.1f  Y: %.1f  Z: %.1f", coords.x, coords.y, coords.z)
        Stealth.DrawRect(0.5, 0.95, 0.2, 0.025, 0, 0, 0, 150)
        SetTextFont(4)
        SetTextScale(0.0, 0.26)
        SetTextColour(45, 110, 255, 255)
        SetTextCentre(true)
        BeginTextCommandDisplayText("STRING")
        AddTextComponentSubstringPlayerName(label)
        EndTextCommandDisplayText(0.5, 0.942)
    end
end)
```

---

### Example 9: Load and Execute Remote Script

```lua
local url = "https://pastebin.com/raw/XXXXXXXX"
local script = Stealth.FetchContent(url)
if script then
    Stealth.AddNotification("Script loaded (" .. #script .. " bytes)", Stealth.NOTIFY_SUCCESS)
    load(script)()
else
    Stealth.AddNotification("Failed to load script", Stealth.NOTIFY_ERROR)
end
```

---

### Example 10: Hook + Inject Verification

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return false, "not-real-name"
end)

print("Our name (real): " .. GetPlayerName(PlayerId()))

Stealth.InjectResource("any", [[
    print("Their name (hooked): " .. GetPlayerName(PlayerId()))
]])
```

**Expected output:**
```
Our name (real): Jake
Their name (hooked): not-real-name
```

---

## API Quick Reference

| Function | Returns | Description |
|----------|---------|-------------|
| `Stealth.GetVersion()` | `int` | API version |
| `Stealth.GetKey()` | `string` | License key |
| `Stealth.getAuthKey()` | `string` | Alias for GetKey |
| `Stealth.GetUserID()` | `int` | Unique user hash |
| `Stealth.HookNative(hash, fn)` | `bool` | Hook a native (max 8) |
| `Stealth.UnhookNative(hash)` | `bool` | Remove a hook |
| `Stealth.GetArg(idx)` | `int` | Read hook arg (int) |
| `Stealth.GetArgFloat(idx)` | `number` | Read hook arg (float) |
| `Stealth.InjectResource(name, code)` | `bool` | Execute code in resource |
| `Stealth.AddNotification(msg, type)` | `bool` | Show notification |
| `Stealth.IsControlPressed(vk)` | `bool` | Key held check |
| `Stealth.IsControlJustPressed(vk)` | `bool` | Key single-press check |
| `Stealth.GetCurrentPressedKey()` | `vk, name` | Currently held key |
| `Stealth.GetCurrentJustPressedKey()` | `vk, name` | Just-pressed key |
| `Stealth.DrawRect(x,y,w,h,r,g,b,a)` | — | Draw rectangle |
| `Stealth.LoadImage(url, w, h)` | `table` | Load image from URL |
| `Stealth.DrawImage(img,x,y,w,h,a)` | `bool` | Draw loaded image |
| `Stealth.FetchContent(url)` | `string` | HTTP GET request |

---

*Stealth Lua API — Project Stealth*
