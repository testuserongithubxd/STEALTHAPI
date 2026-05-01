# Isolated Environment: Definitive Technical Guide

The Isolated Environment is a specialized, shadow-layer execution context that operates detached from the host engine's resource manager. This document provides an exhaustive, detailed reference for every API function, accompanied by practical examples and architectural explanations.

---

## 1. Core Architecture & Philosophy

The environment operates as a **Shadow Layer**. It is not a standard resource and cannot be detected by enumeration scripts or management tools. 

- **Detached Lifecycle**: Variables, threads, and handlers are isolated.
- **Hook Bypass**: Native calls within this environment bypass all host-side hooks, ensuring you see the "True Engine State."
- **Invisibility**: The environment is invisible to the host's task lists and resource manifests.

---

## 2. Native Interception (Hooking)

Native hooking allows you to globally intercept engine functions. This is the primary method for modifying engine behavior without patching memory.

### Stealth.HookNative(hash, callback)
Registers a global interceptor for a native hash. 

- **Returns**: `true` on success, `false` if the native hash is invalid or the hook failed.
- **Argument Handling**: The `callback` receives all arguments passed to the native by the host engine. You can name these arguments or use `...` to capture them.

**Example: Detailed Interception**
```lua
-- Hooking a property check native
local ok = Stealth.HookNative(0x3FEF770D40960D5A, function(entity)
    print("Native 0x3FEF770D40960D5A was called!")
    print("Entity Handle: " .. tostring(entity))
    
    -- Returning a value blocks the original engine call and returns this to the host
    return vector3(0.0, 0.0, 0.0)
end)

if not ok then
    print("Failed to hook native: Hash might be invalid or protected.")
end
```

### Stealth.UnhookNative(hash)
Removes an active hook and restores the original engine behavior.
```lua
Stealth.UnhookNative(0x3FEF770D40960D5A)
```

### Stealth.GetArg(argIdx) / Stealth.GetArgFloat(argIdx)
Alternative method to retrieve arguments inside a hook if you prefer manual indexing.
```lua
Stealth.HookNative(0xABC123..., function(...)
    local firstArg = Stealth.GetArg(0)
    local secondArg = Stealth.GetArgFloat(1)
end)
```

---

## 3. Visuals & Viewport Rendering

The rendering system provides direct viewport access. It uses a **Normalized Coordinate System** (0.0 to 1.0) and a **Frame-Based Pipeline**.

### Stealth.BeginDraw()
Starts a new rendering frame. This is **CRITICAL**:
- Always check the return value. If it returns `false`, the rendering context is not ready (e.g., during engine transitions).
- It resets the internal command buffer for the new frame.

### Stealth.EndDraw()
Finalizes the current frame and pushes all queued commands to the engine's compositing buffer.
- Without calling this, **nothing will appear on screen**.

### Rendering Primitives
- `Stealth.DrawRect(x, y, w, h, r, g, b, a)`: Filled rectangle.
- `Stealth.DrawLine(x1, y1, x2, y2, r, g, b, a, thickness)`: Vector line.
- `Stealth.DrawBox(x, y, w, h, r, g, b, a, thickness)`: Outlined box centered at (X, Y).
- `Stealth.WorldToScreen(x, y, z)`: Returns `sx, sy` (normalized) if the point is on screen, or `nil` if off-screen.

---

## 4. Advanced "Real-World" Examples

### Full Script: Self-ESP (Skeleton & Health)
This example demonstrates a complete rendering loop, including world-to-screen projection, skeleton bone mapping, and health bar rendering.

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        
        -- 1. Initialize the frame. Always check return!
        if not Stealth.BeginDraw() then return end

        repeat
            local myPed = PlayerPedId()
            if not DoesEntityExist(myPed) or IsEntityDead(myPed) then break end
            
            -- Don't draw in first person
            if GetFollowPedCamViewMode() == 4 then break end

            -- Calculate Screen Coordinates for Box
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
                
                -- Draw the main ESP box
                Stealth.DrawBox(bx, by, width, height, 51, 115, 230, 200, 1.0)

                -- Draw Health Bar
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

            -- Skeleton Rendering
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

        -- 2. Push commands to viewport. Stops everything if omitted!
        Stealth.EndDraw()
    end
end)
```

---

## 5. Additional System Functions

### Stealth.FetchContent(url)
Asynchronous web requests. 
```lua
local data = Stealth.FetchContent("https://api.example.com")
```

### Stealth.InjectResource(resourceName, code)
Executes code in a standard engine resource context.
```lua
Stealth.InjectResource("chat", "print('Hello from Shadow Layer!')")
```

### Stealth.AddNotification(msg, type)
Pushes a system notification (0: Info, 1: Success, 2: Warning, 3: Error).
```lua
Stealth.AddNotification("Script Initialized", 1)
```

### Stealth.ExecuteJS(js)
Direct JavaScript injection into the engine's browser context.
```lua
Stealth.ExecuteJS("document.body.style.backgroundColor = 'red'")
```

---

## 6. Constants Reference

- `Stealth.NOTIFY_INFO` (0), `Stealth.NOTIFY_SUCCESS` (1), `Stealth.NOTIFY_WARNING` (2), `Stealth.NOTIFY_ERROR` (3)
- Common VK Codes: `VK_LSHIFT`, `VK_CONTROL`, `VK_INSERT`, `VK_DELETE`, `VK_F1`-`VK_F12`.
