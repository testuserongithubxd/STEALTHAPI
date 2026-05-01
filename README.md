# Isolated Environment: Complete API Reference

The Isolated Environment is a detached execution layer that operates independently of the host engine's resource manager. This document provides a comprehensive reference for every available function in the API.

---

## 1. System & Authentication

### Stealth.GetVersion()
Retrieves the internal build version of the Isolated Environment.
```lua
local version = Stealth.GetVersion()
print("Running version: " .. version)
```

### Stealth.GetUserID()
Retrieves a unique, hardware-linked identifier for the current user.
```lua
local id = Stealth.GetUserID()
print("Your unique hardware ID: " .. id)
```

### Stealth.GetKey()
Retrieves the license key currently used by the session.
```lua
local key = Stealth.GetKey()
print("Active Session Key: " .. key)
```

### Stealth.CloseGame()
Immediately terminates the engine process. Useful for emergency security protocols.
```lua
if some_security_breach then
    Stealth.CloseGame()
end
```

---

## 2. Native Interception (Hooking)

### Stealth.HookNative(hash, callback)
Intercepts an engine native globally. If the callback returns a value, the original call is blocked.
```lua
Stealth.HookNative(0x3FEF770D40960D5A, function(entity)
    print("Intercepted native for entity: " .. entity)
    return vector3(0.0, 0.0, 0.0) -- Block and return custom vector
end)
```

### Stealth.UnhookNative(hash)
Removes a registered native hook by its hash.
```lua
Stealth.UnhookNative(0x3FEF770D40960D5A)
```

### Stealth.GetArg(argIdx)
Retrieves an integer argument from the currently intercepted native call.
```lua
Stealth.HookNative(0xABC123..., function(...)
    local firstArg = Stealth.GetArg(0)
    print("Argument 0 is: " .. firstArg)
end)
```

### Stealth.GetArgFloat(argIdx)
Retrieves a float argument from the currently intercepted native call.
```lua
Stealth.HookNative(0xDEF456..., function(...)
    local floatArg = Stealth.GetArgFloat(1)
    print("Argument 1 is: " .. floatArg)
end)
```

---

## 3. Visuals & Viewport Rendering

### Stealth.BeginDraw()
Initializes a new rendering frame. Must be called before any drawing primitives.
```lua
if Stealth.BeginDraw() then
    -- draw calls here
    Stealth.EndDraw()
end
```

### Stealth.EndDraw()
Finalizes the current rendering frame and pushes all queued commands to the display.
```lua
Stealth.EndDraw()
```

### Stealth.DrawRect(x, y, w, h, r, g, b, a)
Draws a filled rectangle using normalized coordinates (0.0 to 1.0).
```lua
Stealth.DrawRect(0.5, 0.5, 0.1, 0.1, 255, 0, 0, 255)
```

### Stealth.DrawLine(x1, y1, x2, y2, r, g, b, a, thickness)
Draws a line between two normalized points.
```lua
Stealth.DrawLine(0.0, 0.0, 1.0, 1.0, 0, 255, 0, 255, 2.0)
```

### Stealth.DrawBox(x, y, w, h, r, g, b, a, thickness)
Draws an outlined box (wireframe rectangle) centered at X, Y.
```lua
Stealth.DrawBox(0.5, 0.5, 0.2, 0.2, 255, 255, 0, 255, 1.5)
```

### Stealth.WorldToScreen(x, y, z)
Converts 3D world coordinates to 2D normalized screen coordinates.
```lua
local sx, sy = Stealth.WorldToScreen(100.0, 200.0, 30.0)
if sx then
    Stealth.DrawRect(sx, sy, 0.01, 0.01, 255, 255, 255, 255)
end
```

### Stealth.LoadImage(url, w, h)
Asynchronously loads an image from a URL into the UI context.
```lua
local myImg = Stealth.LoadImage("https://example.com/logo.png", 100, 100)
```

### Stealth.DrawImage(img, x, y, w, h, a)
Renders a previously loaded image at the specified pixel coordinates.
```lua
Stealth.DrawImage(myImg, 10, 10, 50, 50, 255)
```

---

## 4. Input & Controls

### Stealth.IsControlPressed(vk)
Checks if a specific Virtual Key is currently held down.
```lua
if Stealth.IsControlPressed(VK_LSHIFT) then
    -- Logic for shift key
end
```

### Stealth.IsControlJustPressed(vk)
Checks if a Virtual Key was pressed in the current frame.
```lua
if Stealth.IsControlJustPressed(VK_INSERT) then
    menu_open = not menu_open
end
```

### Stealth.GetCurrentPressedKey()
Returns the VK code and name of any key currently being held.
```lua
local vk, name = Stealth.GetCurrentPressedKey()
if vk then print("Holding: " .. name) end
```

### Stealth.GetCurrentJustPressedKey()
Returns the VK code and name of the key pressed in the current frame.
```lua
local vk, name = Stealth.GetCurrentJustPressedKey()
if vk then print("Just Pressed: " .. name) end
```

---

## 5. Networking & Cross-Context

### Stealth.FetchContent(url)
Asynchronously fetches text content from a URL.
```lua
local data = Stealth.FetchContent("https://api.myapp.com/v1/config")
if data then
    local config = json.decode(data)
end
```

### Stealth.ExecuteJS(js)
Executes a string of JavaScript in the engine's underlying UI browser context.
```lua
Stealth.ExecuteJS("window.location.reload()")
```

### Stealth.InjectResource(resourceName, code)
Injects a string of Lua code into a target engine resource.
```lua
local patch = "print('Resource has been patched!')"
Stealth.InjectResource("map_manager", patch)
```

### Stealth.AddNotification(msg, type)
Pushes a notification to the internal UI.
**Types:** 0: Info, 1: Success, 2: Warning, 3: Error.
```lua
Stealth.AddNotification("Script Loaded", 1)
```

---

## 6. API Constants Reference

### Notification Types
- `Stealth.NOTIFY_INFO` (0)
- `Stealth.NOTIFY_SUCCESS` (1)
- `Stealth.NOTIFY_WARNING` (2)
- `Stealth.NOTIFY_ERROR` (3)

### Common Virtual Key Codes
- `VK_LSHIFT`, `VK_RSHIFT`
- `VK_LCONTROL`, `VK_RCONTROL`
- `VK_LMENU` (ALT)
- `VK_INSERT`, `VK_DELETE`, `VK_HOME`, `VK_END`
- `VK_F1` through `VK_F12`
- `VK_LBUTTON` (Mouse1), `VK_RBUTTON` (Mouse2)
