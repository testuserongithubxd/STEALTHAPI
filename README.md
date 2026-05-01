# Isolated Environment: Complete Developer Reference

The Isolated Environment is a high-performance shadow layer that operates entirely outside the host engine's resource manager. This document is the exhaustive reference for every function, constant, and system behavior available to developers.

---

## 1. System & Authentication

### Stealth.GetVersion()
Returns the internal build version of the environment.
```lua
local version = Stealth.GetVersion()
print("Build Version: " .. version)
```

### Stealth.GetUserID()
Retrieves a unique hardware-linked identifier.
```lua
local userId = Stealth.GetUserID()
print("User ID: " .. userId)
```

### Stealth.GetKey() / Stealth.getAuthKey()
Retrieves the active session license key.
```lua
local key = Stealth.GetKey()
print("License: " .. key)
```

### Stealth.CloseGame()
Immediately terminates the engine process for security reasons.
```lua
if security_breach then
    Stealth.CloseGame()
end
```

---

## 2. Native Interception (Hooks)

### Stealth.HookNative(hash, callback)
Registers a global interceptor. 
- **Returns**: `true` on success, `false` if invalid or failed.
- **Callback**: Receives all native arguments. Return a value to block the original native.

```lua
local ok = Stealth.HookNative(0x3FEF770D40960D5A, function(entity)
    print("Intercepted! Entity: " .. entity)
    return vector3(0,0,0) -- Blocks and spoofs return
end)
```

### Stealth.UnhookNative(hash)
Removes an active hook. Returns `true` if found and removed.
```lua
Stealth.UnhookNative(0x3FEF770D40960D5A)
```

### Stealth.GetArg(argIdx) / Stealth.GetArgFloat(argIdx)
Manual argument retrieval within a hook context.
```lua
Stealth.HookNative(0x..., function(...)
    local ent = Stealth.GetArg(0)
    local val = Stealth.GetArgFloat(1)
end)
```

---

## 3. Input & Controls

### Stealth.IsControlPressed(vk)
Checks if a Virtual Key is held.
```lua
if Stealth.IsControlPressed(VK_LSHIFT) then
    -- logic
end
```

### Stealth.IsControlJustPressed(vk)
Checks for a single-frame key press.
```lua
if Stealth.IsControlJustPressed(VK_INSERT) then
    -- menu toggle
end
```

### Stealth.GetCurrentPressedKey() / Stealth.GetCurrentJustPressedKey()
Returns the VK code and the human-readable name of the key.
```lua
local vk, name = Stealth.GetCurrentJustPressedKey()
if vk then print("Key: " .. name) end
```

---

## 4. Primitive Rendering

### Stealth.BeginDraw()
Starts a rendering frame. **Returns false if context is busy or unready.**
```lua
if Stealth.BeginDraw() then
    -- draw
    Stealth.EndDraw()
end
```

### Stealth.EndDraw()
Pushes frame commands to the display. Required for visibility.

### Stealth.WorldToScreen(x, y, z)
Converts 3D coordinates to normalized 2D (0.0 - 1.0).
```lua
local sx, sy = Stealth.WorldToScreen(0.0, 0.0, 0.0)
```

### Primitives
- `Stealth.DrawRect(x, y, w, h, r, g, b, a)`
- `Stealth.DrawLine(x1, y1, x2, y2, r, g, b, a, thickness)`
- `Stealth.DrawBox(x, y, w, h, r, g, b, a, thickness)`

---

## 5. Image & Browser Rendering (DUI)

### Stealth.LoadImage(url, w, h)
Loads an image from the web. Returns an image object.
```lua
local img = Stealth.LoadImage("https://i.imgur.com/xyz.png", 200, 200)
```

### Stealth.DrawImage(img, x, y, w, h, a)
Renders a loaded image.
```lua
Stealth.DrawImage(img, 0.1, 0.1, 0.05, 0.05, 255)
```

### DUI System (Advanced Browser Frames)
- `Stealth.CreateDui(url, w, h)`: Creates a hidden browser instance.
- `Stealth.ShowDui(dui)`: Makes the browser visible.
- `Stealth.HideDui(dui)`: Hides the browser.
- `Stealth.DestroyDui(dui)`: Deletes the browser instance.
- `Stealth.SetDuiUrl(dui, url)`: Changes the browser source.
- `Stealth.SetDuiPosition(dui, x, y, w, h)`: Moves/resizes the browser.
- `Stealth.SendDuiMessage(dui, data)`: Sends JSON to the browser via `postMessage`.
- `Stealth.ExecuteDuiScript(dui, script)`: Runs JS inside the specific DUI frame.

```lua
local myDui = Stealth.CreateDui("https://myapp.com", 1920, 1080)
Stealth.ShowDui(myDui)
Stealth.SendDuiMessage(myDui, { action = "open_menu" })
```

---

## 6. Networking & Async

### Stealth.FetchContent(url)
Blocking GET request (waits for result).
```lua
local data = Stealth.FetchContent("https://api.com/data")
```

### Stealth.FetchContentAsync(url, callback)
Non-blocking GET request.
```lua
Stealth.FetchContentAsync("https://api.com/data", function(res)
    print("Data received: " .. #res)
end)
```

### Stealth.FetchReady()
Checks if a pending request is complete.
```lua
if Stealth.FetchReady() then -- data is in buffer end
```

---

## 7. Cross-Context & Interop

### Stealth.InjectResource(resourceName, code)
Executes Lua inside a standard engine resource.
```lua
Stealth.InjectResource("chat", "print('Shadow Link Established')")
```

### Stealth.ExecuteJS(script)
Runs JavaScript in the global UI context.
```lua
Stealth.ExecuteJS("alert('Global JS Injection')")
```

### Stealth.ExecuteJSWithResult(script, timeout)
Runs JS and waits for a return value.
```lua
local cookies = Stealth.ExecuteJSWithResult("document.cookie", 1000)
```

### GetCurrentResourceName() (Global Override)
Always returns `"cfx_internal"` to spoof standard engine checks.

---

## 8. Full Implementation Example: Self-ESP
```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if not Stealth.BeginDraw() then return end

        repeat
            local myPed = PlayerPedId()
            if not DoesEntityExist(myPed) or IsEntityDead(myPed) then break end
            if GetFollowPedCamViewMode() == 4 then break end

            local headPos = GetPedBoneCoords(myPed, 31086, 0.0, 0.0, 0.0)
            local lFoot = GetPedBoneCoords(myPed, 14201, 0.0, 0.0, 0.0)
            local rFoot = GetPedBoneCoords(myPed, 52301, 0.0, 0.0, 0.0)
            local footY = math.min(lFoot.z, rFoot.z)

            local hx, hy = Stealth.WorldToScreen(headPos.x, headPos.y, headPos.z + 0.2)
            local fx, fy = Stealth.WorldToScreen(headPos.x, headPos.y, footY - 0.1)

            if hx and fx then
                local height = math.abs(fy - hy)
                local width = height * 0.35
                local bx = (hx + fx) * 0.5
                local by = (hy + fy) * 0.5
                Stealth.DrawBox(bx, by, width, height, 51, 115, 230, 200, 1.0)

                local hp = GetEntityHealth(myPed) - 100
                local maxHp = GetEntityMaxHealth(myPed) - 100
                if maxHp > 0 then
                    local hpPct = hp / maxHp
                    local barX = bx - width * 0.5 - 0.004
                    local barH = height * hpPct
                    local barY = fy - barH
                    Stealth.DrawRect(barX, barY + barH * 0.5, 0.003, barH, 50, 205, 50, 200)
                end
            end

            local bones = {
                {31086, 39317}, {39317, 45509}, {45509, 61163}, {61163, 18905},
                {39317, 40269}, {40269, 28252}, {28252, 57005}, {39317, 24818},
                {24818, 24817}, {24817, 24816}, {24816, 23553}, {23553, 11816},
                {11816, 58271}, {58271, 63931}, {63931, 14201}, {11816, 51826}, 
                {51826, 36864}, {36864, 52301},
            }
            for _, pair in ipairs(bones) do
                local p1 = GetPedBoneCoords(myPed, pair[1], 0.0, 0.0, 0.0)
                local p2 = GetPedBoneCoords(myPed, pair[2], 0.0, 0.0, 0.0)
                local lx1, ly1 = Stealth.WorldToScreen(p1.x, p1.y, p1.z)
                local lx2, ly2 = Stealth.WorldToScreen(p2.x, p2.y, p2.z)
                if lx1 and lx2 then
                    Stealth.DrawLine(lx1, ly1, lx2, ly2, 255, 255, 255, 180, 1.0)
                end
            end
        until true

        Stealth.EndDraw()
    end
end)
```
