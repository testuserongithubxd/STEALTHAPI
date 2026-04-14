# Project Stealth — Lua API Documentation

**Version:** 98253  
**Runtime:** Isolated Resource Environment  
**Language:** Lua 5.4 (FiveM)

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Execution Modes](#execution-modes)
3. [Identity & Authentication](#identity--authentication)
4. [Native Hooking](#native-hooking)
5. [Input System](#input-system)
6. [Resource Injection](#resource-injection)
7. [Notifications](#notifications)
8. [Drawing & Images](#drawing--images)
9. [JavaScript Execution](#javascript-execution)
10. [DUI System (Dynamic UI)](#dui-system-dynamic-ui)
11. [HTTP Requests](#http-requests)
12. [Virtual Key Constants](#virtual-key-constants)
13. [Complete Examples](#complete-examples)
14. [Quick Reference](#quick-reference)

---

## Getting Started

All Stealth API functions are accessed through `Stealth.FunctionName()`. You have full access to standard FiveM Lua functions (`Citizen.CreateThread`, `Wait`, `GetPlayerPed`, `GetEntityCoords`, etc.) alongside the Stealth API.

When executing scripts, you can choose where your code runs using the dropdown in the bottom-left of the Executor tab. **Isolated** runs your code in Stealth's own private environment — this is the default and recommended mode. **Resource** mode lets you pick a specific server resource to execute inside, giving you access to that resource's variables and event handlers (the Stealth API is not available in this mode). Both modes are explained below.


**Your first script:**

```lua
print("Stealth version: " .. Stealth.GetVersion())
print("User ID: " .. Stealth.GetUserID())

Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- Your per-frame logic here
    end
end)
```

**Important:** Any code that needs to run every frame (drawing, input polling, continuous checks) **must** be inside a `Citizen.CreateThread` or function or not with a `Wait(0)` loop. Code outside threads runs once and exits.
if you prefer not to use `Citizen.CreateThread` your code will only run at the loop and will not continue if the loop is not done.

---

## Execution Modes

### Isolated Resource

Your code runs inside Stealth's own private resource. This is the default mode when you type code in the editor and press Execute. You have full access to the Stealth API and all FiveM client-side natives. Other resources on the server cannot see your code.

Use this for: persistent scripts, native hooks, drawing, keybind systems, DUI overlays, anything that needs the Stealth API.

### Resource Injection

Your code runs inside an existing server resource's Lua environment using `Stealth.InjectResource()`. This gives you access to that resource's local variables, functions, and event handlers. The Stealth API is **not** available inside injected code — only the target resource's own globals and standard FiveM functions.

Use this for: accessing resource-specific data, calling resource-local functions using debug.getupvalue (can be hooked by the anticheat)

---

## Identity & Authentication

### `Stealth.GetVersion()`

Returns the current Stealth build version as an integer.

**Returns:** `number` — version identifier

```lua
local ver = Stealth.GetVersion()
print("Running Stealth v" .. ver)
-- Output: Running Stealth v98253
```

---

### `Stealth.GetKey()`

Returns the user's license key as a string.

**Returns:** `string` — license key

**Alias:** `Stealth.getAuthKey()`

```lua
local key = Stealth.GetKey()
print("License: " .. key)
```

---

### `Stealth.GetUserID()`

Returns a unique numeric identifier derived from your license key. The same key always produces the same ID.

**Returns:** `number` — unique user ID

```lua
local uid = Stealth.GetUserID()
print("User ID: " .. uid)
```

---

## Native Hooking

The hook system intercepts GTA V native function calls made by any resource on the client. When a hooked native is called, your callback fires and you can read arguments, modify the return value, or block the call entirely.



### `Stealth.HookNative(hash, callback)`

Hooks a GTA V native by its hash. The callback fires every time any resource calls that native.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `hash` | `number` | The 64-bit native hash (e.g. `0x6D0DE6A7B5DA71F8` for `GetPlayerName`) |
| `callback` | `function` | Your hook function — see return behavior below |

**Returns:** `boolean` — `true` if the native exists and was hooked successfully, `false` if the hash is invalid, the native doesn't exist in the game

#### Callback Return Behavior

What you return from the callback controls the native's behavior:

| Return | Effect |
|--------|--------|
| *(nothing)* | **Passthrough** — native runs normally, result untouched |
| `number` | **Replace integer result** — blocks original, returns your number |
| `"string"` | **Replace string result** — blocks original, returns your string |
| `vector3(x,y,z)` | **Replace vector3 result** — blocks original, returns your vector |
| `{x, y, z}` | **Replace vector3 via table** — table with 3+ numbers |
| `x, y, z` | **Replace vector3 via multi-return** — three number values |
| `false` | **Block only** — blocks the native, returns nothing |

#### Example — Spoof player name (string return)

```lua
local ok = Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return "FakePlayerName"
end)

if ok then
    print("GetPlayerName hook active!")
else
    print("Failed to hook — invalid hash or slots full")
end

-- Verify it works
Wait(500)
print("My name is now: " .. GetPlayerName(PlayerId()))
-- Output: My name is now: FakePlayerName
```

#### Example — Spoof entity position (vector3 return)

```lua
Stealth.HookNative(0x3FEF770D40960D5A, function()
    return vector3(100.0, 200.0, 30.0)
end)
```

#### Example — Spoof health (integer return)

```lua
Stealth.HookNative(0xEEF5F7E536AB0E37, function()
    return 200
end)
```

#### Example — Log calls without modifying (passthrough)

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    local entity = Stealth.GetArg(0)
    print("GetPlayerName called for entity: " .. entity)
    -- Return nothing = native runs normally, result unchanged
end)
```

#### Example — Block a native entirely

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return false
end)
```

#### Example — Validate before continuing

```lua
local hooked = Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return "spoofed-name"
end)

if hooked then
    Stealth.AddNotification("Hook installed!", Stealth.NOTIFY_SUCCESS)
else
    Stealth.AddNotification("Hook failed! Check your hash.", Stealth.NOTIFY_ERROR)
end
```

#### Example — Invalid hash returns false

```lua
-- This hash doesn't exist — HookNative returns false
local ok = Stealth.HookNative(0xDEADBEEF12345678, function()
    return 999
end)

print(ok)  -- false
```


#### Isolated Resource Bypass

When you call a hooked native from within the Isolated resource (your own scripts), the hook **does not fire**. You always get the original, unmodified result. This means you can spoof `GetPlayerName` for every other resource on the server while still reading the real player name in your own code.

```lua
-- Hook GetPlayerName to return fake name for all other resources
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return "FakePlayer"
end)

-- But in YOUR code, you still get the real name
Citizen.CreateThread(function()
    Wait(1000)
    local realName = GetPlayerName(PlayerId())
    print("Real name: " .. realName)        -- prints the REAL name
    print("Others see: FakePlayer")          -- every other resource sees "FakePlayer"
end)
```

This applies to all hook types — string, integer, vector3, and block hooks. Your isolated code is never affected by your own hooks.

---

### `Stealth.UnhookNative(hash)`

Removes a previously installed hook.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `hash` | `number` | The native hash to unhook |

**Returns:** `boolean` — `true` if found and removed, `false` if no hook existed for that hash

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return "fake-name"
end)

Wait(5000)
Stealth.UnhookNative(0x6D0DE6A7B5DA71F8)
print("Hook removed, names are real again")
```

---

### `Stealth.GetArg(argIdx)`

Reads an integer argument from the currently executing hooked native call. **Only valid inside a hook callback.**

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `argIdx` | `number` | Zero-based argument index |

**Returns:** `number` — the argument value as an integer

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    local playerEntity = Stealth.GetArg(0)
    print("Native called with entity: " .. playerEntity)
end)
```

---

### `Stealth.GetArgFloat(argIdx)`

Reads a float argument from the currently executing hooked native call. **Only valid inside a hook callback.**

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `argIdx` | `number` | Zero-based argument index |

**Returns:** `number` — the argument's float value as raw integer bits

```lua
Stealth.HookNative(0x239528EACDC3E7DE, function()
    local x = Stealth.GetArgFloat(1)
    local y = Stealth.GetArgFloat(2)
    local z = Stealth.GetArgFloat(3)
    print(string.format("SetEntityCoords -> %.2f, %.2f, %.2f", x, y, z))
end)
```

---

## Input System

All input functions **only respond when FiveM is the active foreground window**. They will not fire when alt-tabbed or in another application.

**Important:** Input functions must be called inside a `Citizen.CreateThread` loop with `Wait(0)`.

### `Stealth.IsControlPressed(vk)`

Checks if a key is currently held down.

**Returns:** `boolean`

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlPressed(VK_SHIFT) then
            print("Shift is being held!")
        end
    end
end)
```

---

### `Stealth.IsControlJustPressed(vk)`

Checks if a key was just pressed this frame (rising edge). Returns `true` only once per key press.

**Returns:** `boolean`

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(0x46) then  -- F key
            print("F was just pressed!")
        end
    end
end)
```

---

### `Stealth.GetCurrentPressedKey()`

Returns the first key currently held down.

**Returns:** `number, string` — key code and name, or `nil` if nothing pressed

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        local vk, name = Stealth.GetCurrentPressedKey()
        if vk then
            print("Holding: " .. name .. " (0x" .. string.format("%02X", vk) .. ")")
        end
    end
end)
```

---

### `Stealth.GetCurrentJustPressedKey()`

Returns the first key that was just pressed this frame. Ideal for "press any key to bind" prompts.

**Returns:** `number, string` — key code and name, or `nil` if no key was just pressed

```lua
print("Press any key to set your toggle...")
Citizen.CreateThread(function()
    while true do
        Wait(0)
        local vk, name = Stealth.GetCurrentJustPressedKey()
        if vk then
            print("Bound to: " .. name)
            break
        end
    end
end)
```

---

## Resource Injection

### `Stealth.InjectResource(resourceName, code)`

Executes Lua code inside another resource's environment. The code runs with that resource's globals and event handlers. The Stealth API is **not** available inside injected code.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `resourceName` | `string` | Target resource name, or `"any"` for a random resource |
| `code` | `string` | Lua code to execute (max 4090 characters) |

**Returns:** `boolean` — `true` if sent

Errors in injected code are printed to the FiveM console with line numbers.

```lua
-- Print inside a specific resource
Stealth.InjectResource("hardcap", [[
    print("Hello from inside hardcap!")
    print("Resource: " .. GetCurrentResourceName())
]])
```

```lua
-- Inject into any running resource
Stealth.InjectResource("any", [[
    print("Running in: " .. GetCurrentResourceName())
    local ped = PlayerPedId()
    local coords = GetEntityCoords(ped)
    print(string.format("Pos: %.1f, %.1f, %.1f", coords.x, coords.y, coords.z))
]])
```

```lua
-- Create a persistent thread inside another resource
Stealth.InjectResource("hardcap", [[
    Citizen.CreateThread(function()
        while true do
            Wait(1000)
            print("Tick from " .. GetCurrentResourceName())
        end
    end)
]])
```

---

## Notifications

### `Stealth.AddNotification(message, type)`

Displays an on-screen notification through the Stealth overlay.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `message` | `string` | Text to display (max 500 characters) |
| `type` | `number` | Style constant (optional, defaults to `NOTIFY_INFO`) |

**Returns:** `boolean`

**Types:**

| Constant | Value | Style |
|----------|-------|-------|
| `Stealth.NOTIFY_INFO` | `0` | Neutral |
| `Stealth.NOTIFY_SUCCESS` | `1` | Green |
| `Stealth.NOTIFY_WARNING` | `2` | Yellow |
| `Stealth.NOTIFY_ERROR` | `3` | Red |

```lua
Stealth.AddNotification("Script loaded!", Stealth.NOTIFY_SUCCESS)
Stealth.AddNotification("Something went wrong", Stealth.NOTIFY_ERROR)
Stealth.AddNotification("Low health warning", Stealth.NOTIFY_WARNING)
Stealth.AddNotification("Player nearby")  -- defaults to INFO
```

---

## Drawing & Images

### `Stealth.DrawRect(x, y, w, h, r, g, b, a)`

Draws a filled rectangle. Same coordinate system as the native `DrawRect` — normalized screen space where `(0.5, 0.5)` is center.

**Must be called every frame.**

| Parameter | Type | Description |
|-----------|------|-------------|
| `x` | `number` | Center X (0.0–1.0) |
| `y` | `number` | Center Y (0.0–1.0) |
| `w` | `number` | Width (0.0–1.0) |
| `h` | `number` | Height (0.0–1.0) |
| `r, g, b` | `number` | Color (0–255 each) |
| `a` | `number` | Alpha (0–255, optional, default 255) |

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        Stealth.DrawRect(0.5, 0.5, 0.2, 0.1, 0, 0, 0, 150)
    end
end)
```

---

### `Stealth.LoadImage(url, w, h)`

Loads an image from a URL into the browser layer.

| Parameter | Type | Description |
|-----------|------|-------------|
| `url` | `string` | Direct URL to image (PNG, JPG, GIF, SVG) |
| `w` | `number` | Width in pixels (optional, default 256) |
| `h` | `number` | Height in pixels (optional, default 256) |

**Returns:** `table` — image handle (`{ id, imgId, url, w, h, visible, loaded }`), or `nil`

```lua
local logo = Stealth.LoadImage("https://example.com/logo.png", 128, 128)
```

---

### `Stealth.DrawImage(img, x, y, w, h, a)`

Positions and shows a loaded image. Uses pixel coordinates.

| Parameter | Type | Description |
|-----------|------|-------------|
| `img` | `table` | Handle from `LoadImage` |
| `x` | `number` | Left position (pixels) |
| `y` | `number` | Top position (pixels) |
| `w` | `number` | Width (pixels, optional) |
| `h` | `number` | Height (pixels, optional) |
| `a` | `number` | Alpha 0–255 (optional, default 255) |

**Returns:** `boolean`

```lua
local img = Stealth.LoadImage("https://example.com/crosshair.png", 32, 32)

Citizen.CreateThread(function()
    while true do
        Wait(0)
        if img then
            Stealth.DrawImage(img, 960 - 16, 540 - 16, 32, 32, 200)
        end
    end
end)
```

---

## JavaScript Execution

Execute code in FiveM's CEF (Chromium Embedded Framework) browser layer.

### `Stealth.ExecuteJS(script)`

Fire-and-forget JavaScript execution. No return value.

**Returns:** `boolean`

```lua
Stealth.ExecuteJS([[
    var div = document.createElement('div');
    div.style.cssText = 'position:fixed;top:10px;right:10px;background:red;color:white;padding:20px;z-index:99999;font-size:24px;border-radius:8px;';
    div.textContent = 'Stealth Active';
    document.body.appendChild(div);
    setTimeout(function() { div.remove(); }, 3000);
]])
```

---

### `Stealth.ExecuteJSWithResult(script, timeout)`

Executes JavaScript and waits for the return value. Must be called from a thread (it yields with `Wait`).

| Parameter | Type | Description |
|-----------|------|-------------|
| `script` | `string` | JS expression returning a string |
| `timeout` | `number` | Max wait in ms (optional, default 2000, max 10000) |

**Returns:** `string` or `nil` on timeout

```lua
Citizen.CreateThread(function()
    local title = Stealth.ExecuteJSWithResult("document.title")
    print("Title: " .. tostring(title))

    local ua = Stealth.ExecuteJSWithResult("navigator.userAgent", 500)
    print("UA: " .. tostring(ua))

    local exists = Stealth.ExecuteJSWithResult(
        "document.querySelector('.some-class') ? 'yes' : 'no'"
    )
    print("Element: " .. tostring(exists))
end)
```

---

## DUI System (Dynamic UI)

Create full HTML/CSS/JS overlays rendered on top of the game. Each DUI is an invisible iframe injected into the browser layer.

### `Stealth.CreateDui(url, w, h)`

Creates a new DUI overlay (starts hidden).

**Returns:** `table` — `{ id, frameId, url, w, h, visible }`, or `nil`

```lua
local ui = Stealth.CreateDui("https://example.com/overlay.html")
Stealth.ShowDui(ui)
```

### `Stealth.ShowDui(dui)` / `Stealth.HideDui(dui)`

Show or hide a DUI without destroying it.

### `Stealth.DestroyDui(dui)`

Permanently removes a DUI.

### `Stealth.SetDuiUrl(dui, url)`

Changes the URL loaded in the DUI.

### `Stealth.SetDuiPosition(dui, x, y, w, h)`

Repositions/resizes a DUI. Pixel coordinates.

```lua
Stealth.SetDuiPosition(ui, 1520, 0, 400, 300)
```

### `Stealth.SendDuiMessage(dui, data)`

Sends data to the DUI iframe via `postMessage`. Tables are auto-JSON-encoded.

```lua
Stealth.SendDuiMessage(ui, { action = "update", health = 85 })
```

HTML side:
```html
<script>
window.addEventListener('message', function(e) {
    var data = JSON.parse(e.data);
    console.log(data.action, data.health);
});
</script>
```

### `Stealth.ExecuteDuiScript(dui, script)`

Runs JavaScript directly inside the DUI iframe's context.

```lua
Stealth.ExecuteDuiScript(ui, "document.body.style.background = 'red'")
```

---

## HTTP Requests

### `Stealth.FetchContent(url)`

Synchronous HTTP GET. Returns the response body.

| Parameter | Type | Description |
|-----------|------|-------------|
| `url` | `string` | URL to fetch (max 2040 chars) |

**Returns:** `string` or `nil` on failure

**Note:** Blocks the game thread until complete. Use fast-responding URLs.

```lua
local data = Stealth.FetchContent("https://example.com/config.json")
if data then
    print("Loaded " .. #data .. " bytes")
    local config = json.decode(data)
    if config then
        print("Setting: " .. tostring(config.someSetting))
    end
else
    print("Failed to load")
end
```

```lua
-- Load and execute a remote Lua script
local code = Stealth.FetchContent("https://example.com/myscript.lua")
if code then
    local fn, err = load(code)
    if fn then
        fn()
        print("Remote script loaded!")
    else
        print("Error: " .. tostring(err))
    end
end
```

---

## Virtual Key Constants

All standard Windows virtual key codes are available as globals.

### Mouse

`VK_LBUTTON` (0x01), `VK_RBUTTON` (0x02), `VK_MBUTTON` (0x04)

### Navigation

`VK_BACK` (0x08), `VK_TAB` (0x09), `VK_RETURN` (0x0D), `VK_ESCAPE` (0x1B), `VK_SPACE` (0x20), `VK_PRIOR` (0x21), `VK_NEXT` (0x22), `VK_END` (0x23), `VK_HOME` (0x24), `VK_LEFT` (0x25), `VK_UP` (0x26), `VK_RIGHT` (0x27), `VK_DOWN` (0x28), `VK_SNAPSHOT` (0x2C), `VK_INSERT` (0x2D), `VK_DELETE` (0x2E)

### Modifiers

`VK_SHIFT` (0x10), `VK_CONTROL` (0x11), `VK_MENU` (0x12), `VK_PAUSE` (0x13), `VK_CAPITAL` (0x14), `VK_LSHIFT` (0xA0), `VK_RSHIFT` (0xA1), `VK_LCONTROL` (0xA2), `VK_RCONTROL` (0xA3), `VK_LMENU` (0xA4), `VK_RMENU` (0xA5)

### Function Keys

`VK_F1`–`VK_F12` (0x70–0x7B)

### Numpad

`VK_NUMPAD0`–`VK_NUMPAD9` (0x60–0x69), `VK_MULTIPLY` (0x6A), `VK_ADD` (0x6B), `VK_SUBTRACT` (0x6D), `VK_DECIMAL` (0x6E), `VK_DIVIDE` (0x6F), `VK_NUMLOCK` (0x90), `VK_SCROLL` (0x91)

### Letters and Numbers

Letters A–Z: `0x41`–`0x5A`. Numbers 0–9: `0x30`–`0x39`.

```lua
if Stealth.IsControlJustPressed(0x47) then  -- G
    print("G pressed!")
end
```

---

## Complete Examples

### Toggle Display with F5

```lua
local showInfo = false

Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_F5) then
            showInfo = not showInfo
            Stealth.AddNotification(
                showInfo and "Display ON" or "Display OFF",
                showInfo and Stealth.NOTIFY_SUCCESS or Stealth.NOTIFY_WARNING
            )
        end
        if showInfo then
            Stealth.DrawRect(0.5, 0.02, 0.3, 0.03, 0, 0, 0, 180)
        end
    end
end)
```

### Hook With Auto-Cleanup

```lua
local hash = 0x6D0DE6A7B5DA71F8

local ok = Stealth.HookNative(hash, function()
    return "StealthUser"
end)

if ok then
    Stealth.AddNotification("Name spoof active!", Stealth.NOTIFY_SUCCESS)
    Citizen.CreateThread(function()
        Wait(30000)
        Stealth.UnhookNative(hash)
        Stealth.AddNotification("Name spoof removed after 30s", Stealth.NOTIFY_INFO)
    end)
else
    Stealth.AddNotification("Hook failed!", Stealth.NOTIFY_ERROR)
end
```

### DUI Overlay With Live Data

```lua
local overlay = Stealth.CreateDui("https://example.com/hud.html", 400, 200)
if not overlay then return end

Stealth.SetDuiPosition(overlay, 10, 10, 400, 200)
Stealth.ShowDui(overlay)

Citizen.CreateThread(function()
    while overlay do
        Wait(1000)
        local ped = PlayerPedId()
        Stealth.SendDuiMessage(overlay, {
            health = GetEntityHealth(ped),
            armor = GetPedArmour(ped)
        })
    end
end)

Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_F6) then
            Stealth.DestroyDui(overlay)
            overlay = nil
            Stealth.AddNotification("Overlay closed", Stealth.NOTIFY_INFO)
            break
        end
    end
end)
```

### Keybind Capture System

```lua
Stealth.AddNotification("Press any key to bind...", Stealth.NOTIFY_INFO)

Citizen.CreateThread(function()
    while true do
        Wait(0)
        local vk, name = Stealth.GetCurrentJustPressedKey()
        if vk and vk ~= VK_ESCAPE then
            Stealth.AddNotification("Bound to: " .. name, Stealth.NOTIFY_SUCCESS)

            local menuOpen = false
            while true do
                Wait(0)
                if Stealth.IsControlJustPressed(vk) then
                    menuOpen = not menuOpen
                    print("Menu: " .. (menuOpen and "OPEN" or "CLOSED"))
                end
                if menuOpen then
                    Stealth.DrawRect(0.5, 0.5, 0.3, 0.4, 20, 20, 20, 200)
                end
            end
        end
    end
end)
```

### Remote Script Loader

```lua
Citizen.CreateThread(function()
    Stealth.AddNotification("Loading remote script...", Stealth.NOTIFY_INFO)

    local code = Stealth.FetchContent("https://example.com/scripts/features.lua")
    if code and #code > 0 then
        local fn, err = load(code)
        if fn then
            fn()
            Stealth.AddNotification("Loaded! (" .. #code .. " bytes)", Stealth.NOTIFY_SUCCESS)
        else
            Stealth.AddNotification("Error: " .. tostring(err), Stealth.NOTIFY_ERROR)
        end
    else
        Stealth.AddNotification("Download failed", Stealth.NOTIFY_ERROR)
    end
end)
```

---

## Quick Reference

| Function | Returns | Description |
|----------|---------|-------------|
| `Stealth.GetVersion()` | `number` | Build version |
| `Stealth.GetKey()` | `string` | License key |
| `Stealth.GetUserID()` | `number` | Unique user ID |
| `Stealth.HookNative(hash, cb)` | `boolean` | Hook native (true = success, false = invalid/full) |
| `Stealth.UnhookNative(hash)` | `boolean` | Remove hook |
| `Stealth.GetArg(idx)` | `number` | Read hook arg (int) |
| `Stealth.GetArgFloat(idx)` | `number` | Read hook arg (float bits) |
| `Stealth.IsControlPressed(vk)` | `boolean` | Key held |
| `Stealth.IsControlJustPressed(vk)` | `boolean` | Key just pressed |
| `Stealth.GetCurrentPressedKey()` | `number, string` | Any key held |
| `Stealth.GetCurrentJustPressedKey()` | `number, string` | Any key just pressed |
| `Stealth.InjectResource(name, code)` | `boolean` | Execute code in another resource |
| `Stealth.AddNotification(msg, type)` | `boolean` | Show notification |
| `Stealth.DrawRect(x,y,w,h,r,g,b,a)` | — | Draw rectangle (per frame) |
| `Stealth.LoadImage(url, w, h)` | `table` | Load image from URL |
| `Stealth.DrawImage(img, x,y,w,h,a)` | `boolean` | Position/show image |
| `Stealth.ExecuteJS(script)` | `boolean` | Run JS (fire & forget) |
| `Stealth.ExecuteJSWithResult(script, timeout)` | `string` | Run JS, get result |
| `Stealth.FetchContent(url)` | `string` | HTTP GET |
| `Stealth.CreateDui(url, w, h)` | `table` | Create DUI overlay |
| `Stealth.ShowDui(dui)` | — | Show DUI |
| `Stealth.HideDui(dui)` | — | Hide DUI |
| `Stealth.DestroyDui(dui)` | — | Remove DUI |
| `Stealth.SetDuiUrl(dui, url)` | — | Change DUI URL |
| `Stealth.SetDuiPosition(dui, x,y,w,h)` | — | Reposition DUI |
| `Stealth.SendDuiMessage(dui, data)` | — | Send data to DUI |
| `Stealth.ExecuteDuiScript(dui, script)` | — | Run JS in DUI |
