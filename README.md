# Stealth Lua API Documentation

## Overview

The Stealth API provides a complete scripting environment for FiveM. All code runs in a **fully isolated runtime** — no server, no resource, and no anticheat can detect or interact with your scripts. Your code is invisible to every other resource on the server.

The API is available in the **Isolated** execution mode from the Lua Editor.

---

## Execution Modes

| Mode | Description |
|------|-------------|
| **Resource** | Injects code into an existing server resource. Visible to that resource's environment. |
| **Isolated** | Runs in a private, sandboxed resource. No resource can see it. Full Stealth API available. |

---

## Core

### `Stealth.GetVersion()`
Returns the current Stealth API version.

| | |
|---|---|
| **Parameters** | None |
| **Returns** | `number` — Version identifier (e.g. `1337`) |

```lua
local ver = Stealth.GetVersion()
print("Stealth v" .. tostring(ver))
```

---

### `Stealth.GetKey()`
Returns the license key used to authenticate the current session.

| | |
|---|---|
| **Parameters** | None |
| **Returns** | `string` — The full license key |

```lua
local key = Stealth.GetKey()
print("Key: " .. key)
```

---

### `Stealth.GetUserID()`
Returns a unique numeric identifier derived from the license key.

| | |
|---|---|
| **Parameters** | None |
| **Returns** | `number` — Unique user ID (unsigned 32-bit hash) |

```lua
local uid = Stealth.GetUserID()
print("User ID: " .. tostring(uid))
```

---

## Input

All input functions use **Windows Virtual Key (VK) codes**. Common constants are pre-defined (see [VK Constants](#vk-constants)).

### `Stealth.IsControlPressed(vk)`
Returns whether a key is currently held down.

| | |
|---|---|
| **Parameters** | `vk` — `number` — Virtual key code |
| **Returns** | `boolean` — `true` if the key is currently held |

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlPressed(VK_SHIFT) then
            print("Shift is held")
        end
    end
end)
```

---

### `Stealth.IsControlJustPressed(vk)`
Returns whether a key was just pressed this frame (rising edge).

| | |
|---|---|
| **Parameters** | `vk` — `number` — Virtual key code |
| **Returns** | `boolean` — `true` if the key transitioned from up to down |

```lua
if Stealth.IsControlJustPressed(VK_F5) then
    print("F5 pressed!")
end
```

---

### `Stealth.GetCurrentPressedKey()`
Returns the first key currently being held down.

| | |
|---|---|
| **Parameters** | None |
| **Returns** | `number, string` — VK code and key name, or `nil` if no key is pressed |

```lua
local vk, name = Stealth.GetCurrentPressedKey()
if vk then
    print("Holding: " .. name .. " (0x" .. string.format("%02X", vk) .. ")")
end
```

---

### `Stealth.GetCurrentJustPressedKey()`
Returns the first key that was just pressed this frame.

| | |
|---|---|
| **Parameters** | None |
| **Returns** | `number, string` — VK code and key name, or `nil` if no key was just pressed |

```lua
local vk, name = Stealth.GetCurrentJustPressedKey()
if vk then
    print("Just pressed: " .. name)
end
```

---

## Native Hooking

Hook any GTA V / FiveM native function. Your callback fires every time the native is called by any resource. You can read arguments, modify return values, or block the call entirely.

**Maximum hooks:** 8 simultaneous hooks.

### `Stealth.HookNative(hash, callback)`
Hooks a native function by its 64-bit hash.

| | |
|---|---|
| **Parameters** | `hash` — `number` — 64-bit native hash |
| | `callback` — `function` — Called each time the native fires |
| **Returns** | `boolean` — `true` if the hook was registered, `false` if slots are full |

**Callback return values:**

| Return | Effect |
|--------|--------|
| *(nothing)* | Native executes normally |
| `false` | Blocks the original native from executing |
| `false, value` | Blocks the native and sets a custom return value |

```lua
-- Block all weapon damage
Stealth.HookNative(0x113, function()
    return false
end)

-- Intercept and modify a return value
Stealth.HookNative(0xA77F2060, function()
    return false, 999
end)
```

---

### `Stealth.UnhookNative(hash)`
Removes a previously registered native hook.

| | |
|---|---|
| **Parameters** | `hash` — `number` — The same hash passed to `HookNative` |
| **Returns** | `boolean` — `true` if the hook was found and removed |

```lua
Stealth.UnhookNative(0x113)
```

---

### `Stealth.GetArg(index)`
Reads an integer argument from the currently executing hooked native. Only valid inside a hook callback.

| | |
|---|---|
| **Parameters** | `index` — `number` — Argument index (0-based) |
| **Returns** | `number` — The integer value of the argument |

---

### `Stealth.GetArgFloat(index)`
Reads a float argument from the currently executing hooked native (returned as raw int bits). Only valid inside a hook callback.

| | |
|---|---|
| **Parameters** | `index` — `number` — Argument index (0-based) |
| **Returns** | `number` — The raw float bits as an integer |

---

## Notifications

### `Stealth.AddNotification(message, type)`
Displays an on-screen notification in the Stealth UI.

| | |
|---|---|
| **Parameters** | `message` — `string` — The notification text |
| | `type` — `number` *(optional, default 0)* — Notification style |
| **Returns** | `boolean` — `true` if sent successfully |

**Notification types:**

| Constant | Value | Style |
|----------|-------|-------|
| `Stealth.NOTIFY_INFO` | 0 | Blue / informational |
| `Stealth.NOTIFY_SUCCESS` | 1 | Green / success |
| `Stealth.NOTIFY_WARNING` | 2 | Yellow / warning |
| `Stealth.NOTIFY_ERROR` | 3 | Red / error |

```lua
Stealth.AddNotification("Script loaded!", Stealth.NOTIFY_SUCCESS)
Stealth.AddNotification("Something went wrong", Stealth.NOTIFY_ERROR)
```

---

## Networking

### `Stealth.FetchContent(url)`
Performs a synchronous HTTP GET request and returns the response body. Uses WinINet internally — works for any public URL.

| | |
|---|---|
| **Parameters** | `url` — `string` — The URL to fetch |
| **Returns** | `string` — The response body, or `nil` on failure |

```lua
local body = Stealth.FetchContent("https://api.ipify.org")
if body then
    print("My IP: " .. body)
end
```

```lua
local json = Stealth.FetchContent("https://jsonplaceholder.typicode.com/posts/1")
if json then
    print(json)
end
```

> **Note:** This is a blocking call. For large requests, consider wrapping in a `Citizen.CreateThread` with a `Wait`.

---

## Drawing

### `Stealth.DrawRect(x, y, w, h, r, g, b, a)`
Draws a filled rectangle on screen. Must be called every frame.

| | |
|---|---|
| **Parameters** | `x` — `number` — Center X (0.0 - 1.0) |
| | `y` — `number` — Center Y (0.0 - 1.0) |
| | `w` — `number` — Width (0.0 - 1.0) |
| | `h` — `number` — Height (0.0 - 1.0) |
| | `r, g, b` — `number` — Color (0 - 255) |
| | `a` — `number` *(optional, default 255)* — Alpha |
| **Returns** | None |

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
Loads a remote image into a runtime texture via DUI.

| | |
|---|---|
| **Parameters** | `url` — `string` — Image URL |
| | `w` — `number` *(optional, default 256)* — Texture width |
| | `h` — `number` *(optional, default 256)* — Texture height |
| **Returns** | `table` — Image handle with fields: `txd`, `name`, `dui`, `loaded` |

```lua
local logo = Stealth.LoadImage("https://i.imgur.com/example.png", 128, 128)
```

---

### `Stealth.DrawImage(image, x, y, w, h, a)`
Draws a previously loaded image on screen. Must be called every frame.

| | |
|---|---|
| **Parameters** | `image` — `table` — Handle returned by `LoadImage` |
| | `x` — `number` — Center X (0.0 - 1.0) |
| | `y` — `number` — Center Y (0.0 - 1.0) |
| | `w` — `number` — Width (0.0 - 1.0) |
| | `h` — `number` — Height (0.0 - 1.0) |
| | `a` — `number` *(optional, default 255)* — Alpha |
| **Returns** | `boolean` — `true` if drawn successfully |

```lua
local logo = Stealth.LoadImage("https://i.imgur.com/example.png", 256, 256)
Citizen.CreateThread(function()
    while true do
        Wait(0)
        Stealth.DrawImage(logo, 0.9, 0.1, 0.05, 0.05)
    end
end)
```

---

## Resource Injection

### `Stealth.InjectResource(resourceName, code)`
Injects and executes Lua code inside another resource's runtime. The target resource sees this code as its own.

| | |
|---|---|
| **Parameters** | `resourceName` — `string` — Target resource name, or `"any"` for the first available |
| | `code` — `string` — Lua code to execute |
| **Returns** | `boolean` — `true` if injection was initiated |

```lua
Stealth.InjectResource("es_extended", [[
    print("[INJECTED] Running inside es_extended!")
    local xPlayer = ESX.GetPlayerData()
    print("Job: " .. xPlayer.job.name)
]])
```

```lua
-- Inject into any available resource
Stealth.InjectResource("any", "print('Hello from injection')")
```

> **Warning:** Injected code runs inside the target resource's environment. Errors in injected code may trigger server-side logging in that resource.

---

## CEF / JavaScript Execution

FiveM uses Chromium Embedded Framework (CEF) for its NUI browser layer. The Stealth API connects to CEF's remote debugging protocol, allowing you to execute arbitrary JavaScript inside the game's browser.

### `Stealth.ExecuteJS(script)`
Executes JavaScript in the FiveM NUI browser page.

| | |
|---|---|
| **Parameters** | `script` — `string` — JavaScript code to execute |
| **Returns** | `boolean` — `true` if the command was sent |

```lua
-- Change the page title
Stealth.ExecuteJS("document.title = 'Stealth'")

-- Show a fullscreen overlay for 3 seconds
Stealth.ExecuteJS([[
    var div = document.createElement('div');
    div.style.cssText = 'position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.8);z-index:999999;display:flex;align-items:center;justify-content:center;';
    div.innerHTML = '<h1 style="color:#2d6eff;font-size:72px;font-family:Arial;">STEALTH</h1>';
    document.body.appendChild(div);
    setTimeout(function(){div.remove()}, 3000);
]])
```

> **Note:** JavaScript runs asynchronously. There is no return value from the executed script.

---

## DUI (Direct User Interface)

The DUI system lets you render custom HTML pages as overlays inside the game. Pages are injected as iframes into the NUI browser layer via CEF. They are invisible to server resources and render on top of everything.

### `Stealth.CreateDui(url, w, h)`
Creates a new DUI window from a URL. The DUI is hidden by default.

| | |
|---|---|
| **Parameters** | `url` — `string` — The page URL (can be localhost, remote, or `data:text/html,...`) |
| | `w` — `number` *(optional)* — Width in pixels (defaults to screen width) |
| | `h` — `number` *(optional)* — Height in pixels (defaults to screen height) |
| **Returns** | `table` — DUI handle with fields: `id`, `frameId`, `url`, `w`, `h`, `visible` |

```lua
local dui = Stealth.CreateDui("http://localhost:8080/menu.html", 800, 600)
```

---

### `Stealth.ShowDui(handle)`
Makes a DUI window visible.

| | |
|---|---|
| **Parameters** | `handle` — `table` — DUI handle from `CreateDui` |
| **Returns** | None |

```lua
Stealth.ShowDui(dui)
```

---

### `Stealth.HideDui(handle)`
Hides a DUI window without destroying it.

| | |
|---|---|
| **Parameters** | `handle` — `table` — DUI handle from `CreateDui` |
| **Returns** | None |

```lua
Stealth.HideDui(dui)
```

---

### `Stealth.DestroyDui(handle)`
Permanently removes a DUI window and cleans up its resources.

| | |
|---|---|
| **Parameters** | `handle` — `table` — DUI handle from `CreateDui` |
| **Returns** | None |

```lua
Stealth.DestroyDui(dui)
```

---

### `Stealth.SetDuiUrl(handle, url)`
Changes the URL of an existing DUI window.

| | |
|---|---|
| **Parameters** | `handle` — `table` — DUI handle |
| | `url` — `string` — New URL to navigate to |
| **Returns** | None |

```lua
Stealth.SetDuiUrl(dui, "https://example.com")
```

---

### `Stealth.SetDuiPosition(handle, x, y, w, h)`
Sets the position and size of a DUI window in pixels.

| | |
|---|---|
| **Parameters** | `handle` — `table` — DUI handle |
| | `x` — `number` — Left position in pixels |
| | `y` — `number` — Top position in pixels |
| | `w` — `number` — Width in pixels |
| | `h` — `number` — Height in pixels |
| **Returns** | None |

```lua
Stealth.SetDuiPosition(dui, 100, 100, 600, 400)
```

---

### `Stealth.SendDuiMessage(handle, data)`
Sends a message to the DUI page via `window.postMessage`. Catch it with `window.addEventListener("message", ...)` inside your HTML.

| | |
|---|---|
| **Parameters** | `handle` — `table` — DUI handle |
| | `data` — `string` or `table` — Data to send (tables are JSON-encoded) |
| **Returns** | None |

**Lua side:**
```lua
Stealth.SendDuiMessage(dui, { type = "update", health = 100 })
```

**HTML side:**
```html
<script>
window.addEventListener("message", function(e) {
    var data = JSON.parse(e.data);
    console.log("Received:", data.type, data.health);
});
</script>
```

---

### `Stealth.ExecuteDuiScript(handle, script)`
Executes JavaScript directly inside a DUI iframe.

| | |
|---|---|
| **Parameters** | `handle` — `table` — DUI handle |
| | `script` — `string` — JavaScript code |
| **Returns** | None |

```lua
Stealth.ExecuteDuiScript(dui, "document.body.style.background = 'red'")
```

---

## DUI Full Example

```lua
-- Create a DUI from a local server
local dui = Stealth.CreateDui("http://localhost:8080/overlay.html", 800, 500)
Wait(500)
Stealth.ShowDui(dui)

Stealth.AddNotification("DUI loaded!", Stealth.NOTIFY_SUCCESS)

-- Update loop: send player data to the HTML page every 500ms
Citizen.CreateThread(function()
    while true do
        Wait(500)
        if dui then
            local ped = PlayerPedId()
            local hp = GetEntityHealth(ped)
            local armor = GetPedArmour(ped)
            local pos = GetEntityCoords(ped)
            Stealth.SendDuiMessage(dui, {
                type = "playerUpdate",
                health = hp,
                armor = armor,
                x = pos.x,
                y = pos.y,
                z = pos.z
            })
        end
    end
end)

-- Controls
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_F5) then
            Stealth.HideDui(dui)
            Stealth.AddNotification("DUI hidden", Stealth.NOTIFY_INFO)
        end
        if Stealth.IsControlJustPressed(VK_F6) then
            Stealth.ShowDui(dui)
            Stealth.AddNotification("DUI shown", Stealth.NOTIFY_SUCCESS)
        end
        if Stealth.IsControlJustPressed(VK_F7) then
            Stealth.DestroyDui(dui)
            dui = nil
            Stealth.AddNotification("DUI destroyed", Stealth.NOTIFY_WARNING)
            break
        end
    end
end)
```

---

## VK Constants

All standard Windows virtual key codes are pre-defined as global constants.

### Mouse
| Constant | Value | Key |
|----------|-------|-----|
| `VK_LBUTTON` | `0x01` | Left mouse |
| `VK_RBUTTON` | `0x02` | Right mouse |
| `VK_MBUTTON` | `0x04` | Middle mouse |

### Navigation
| Constant | Value | Key |
|----------|-------|-----|
| `VK_BACK` | `0x08` | Backspace |
| `VK_TAB` | `0x09` | Tab |
| `VK_RETURN` | `0x0D` | Enter |
| `VK_ESCAPE` | `0x1B` | Escape |
| `VK_SPACE` | `0x20` | Space |
| `VK_PRIOR` | `0x21` | Page Up |
| `VK_NEXT` | `0x22` | Page Down |
| `VK_END` | `0x23` | End |
| `VK_HOME` | `0x24` | Home |
| `VK_LEFT` | `0x25` | Left Arrow |
| `VK_UP` | `0x26` | Up Arrow |
| `VK_RIGHT` | `0x27` | Right Arrow |
| `VK_DOWN` | `0x28` | Down Arrow |
| `VK_INSERT` | `0x2D` | Insert |
| `VK_DELETE` | `0x2E` | Delete |
| `VK_SNAPSHOT` | `0x2C` | Print Screen |

### Modifiers
| Constant | Value | Key |
|----------|-------|-----|
| `VK_SHIFT` | `0x10` | Shift (either) |
| `VK_CONTROL` | `0x11` | Ctrl (either) |
| `VK_MENU` | `0x12` | Alt (either) |
| `VK_LSHIFT` | `0xA0` | Left Shift |
| `VK_RSHIFT` | `0xA1` | Right Shift |
| `VK_LCONTROL` | `0xA2` | Left Ctrl |
| `VK_RCONTROL` | `0xA3` | Right Ctrl |
| `VK_LMENU` | `0xA4` | Left Alt |
| `VK_RMENU` | `0xA5` | Right Alt |
| `VK_CAPITAL` | `0x14` | Caps Lock |
| `VK_NUMLOCK` | `0x90` | Num Lock |
| `VK_SCROLL` | `0x91` | Scroll Lock |

### Function Keys
| Constant | Value | Key |
|----------|-------|-----|
| `VK_F1` — `VK_F12` | `0x70` — `0x7B` | F1 through F12 |

### Numpad
| Constant | Value | Key |
|----------|-------|-----|
| `VK_NUMPAD0` — `VK_NUMPAD9` | `0x60` — `0x69` | Numpad 0 - 9 |
| `VK_MULTIPLY` | `0x6A` | Numpad * |
| `VK_ADD` | `0x6B` | Numpad + |
| `VK_SUBTRACT` | `0x6D` | Numpad - |
| `VK_DECIMAL` | `0x6E` | Numpad . |
| `VK_DIVIDE` | `0x6F` | Numpad / |

### Letters & Numbers
Letter keys `A`-`Z` use their ASCII values: `0x41` — `0x5A`.
Number keys `0`-`9` use their ASCII values: `0x30` — `0x39`.
No constants are defined — use the values directly:
```lua
if Stealth.IsControlJustPressed(0x41) then -- A key
    print("A pressed")
end
```

---

## FiveM Natives

All standard FiveM / GTA V natives are available directly in the isolated runtime. You can call them as you would in any normal resource:

```lua
local ped = PlayerPedId()
local pos = GetEntityCoords(ped)
local hp = GetEntityHealth(ped)
local veh = GetVehiclePedIsIn(ped, false)
```

The full native reference is available at: https://docs.fivem.net/natives/

---

## Reset Behavior

Pressing **Reset** in the Lua Editor performs a full cleanup:

- All native hooks are removed
- All DUI iframes are destroyed
- The Lua runtime is destroyed and recreated
- All state (buffers, key states, fetch results) is cleared
- A fresh isolated environment is created on next Execute

---

## Quick Reference

| Function | Returns | Description |
|----------|---------|-------------|
| `Stealth.GetVersion()` | `number` | API version |
| `Stealth.GetKey()` | `string` | License key |
| `Stealth.GetUserID()` | `number` | Unique user ID |
| `Stealth.IsControlPressed(vk)` | `boolean` | Key held down |
| `Stealth.IsControlJustPressed(vk)` | `boolean` | Key just pressed |
| `Stealth.GetCurrentPressedKey()` | `number, string` | First held key |
| `Stealth.GetCurrentJustPressedKey()` | `number, string` | First just-pressed key |
| `Stealth.HookNative(hash, cb)` | `boolean` | Hook a native |
| `Stealth.UnhookNative(hash)` | `boolean` | Unhook a native |
| `Stealth.GetArg(idx)` | `number` | Read hook arg (int) |
| `Stealth.GetArgFloat(idx)` | `number` | Read hook arg (float bits) |
| `Stealth.AddNotification(msg, type)` | `boolean` | Show notification |
| `Stealth.FetchContent(url)` | `string` | HTTP GET |
| `Stealth.DrawRect(x,y,w,h,r,g,b,a)` | — | Draw rectangle |
| `Stealth.LoadImage(url, w, h)` | `table` | Load remote image |
| `Stealth.DrawImage(img,x,y,w,h,a)` | `boolean` | Draw loaded image |
| `Stealth.InjectResource(name, code)` | `boolean` | Inject into resource |
| `Stealth.ExecuteJS(script)` | `boolean` | Execute JS in NUI |
| `Stealth.CreateDui(url, w, h)` | `table` | Create DUI overlay |
| `Stealth.ShowDui(handle)` | — | Show DUI |
| `Stealth.HideDui(handle)` | — | Hide DUI |
| `Stealth.DestroyDui(handle)` | — | Destroy DUI |
| `Stealth.SetDuiUrl(handle, url)` | — | Change DUI URL |
| `Stealth.SetDuiPosition(handle, x,y,w,h)` | — | Position DUI |
| `Stealth.SendDuiMessage(handle, data)` | — | Send message to DUI |
| `Stealth.ExecuteDuiScript(handle, js)` | — | Run JS inside DUI |
