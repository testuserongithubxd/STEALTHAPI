# Isolated Environment: Complete Developer Reference

The Isolated Environment is a high-performance shadow layer that operates entirely outside the host engine's resource manager. This document is the exhaustive reference for every function, constant, and system behavior available to developers, heavily focused on advanced, practical examples.

---

## 1. System & Authentication

### Stealth.GetVersion()
Returns the internal build version of the environment.
```lua
-- Example: Enforcing version compatibility
local requiredVersion = 10050
if Stealth.GetVersion() < requiredVersion then
    Stealth.AddNotification("Update required for this script!", Stealth.NOTIFY_ERROR)
    return
end
```

### Stealth.GetUserID()
Retrieves a unique, hardware-linked identifier.
```lua
-- Example: Hardcoding a whitelist for specific users
local myId = Stealth.GetUserID()
local whitelist = { [123456789] = true, [987654321] = true }

if not whitelist[myId] then
    print("Unauthorized hardware ID: " .. myId)
    Stealth.CloseGame()
end
```

### Stealth.GetKey() / Stealth.getAuthKey()
Retrieves the active session license key.
```lua
-- Example: Passing the session key to a secure external API endpoint
local key = Stealth.GetKey()
Stealth.FetchContentAsync("https://api.secure.com/auth?key=" .. key, function(response)
    print("Auth response: " .. tostring(response))
end)
```

### Stealth.CloseGame()
Immediately terminates the engine process for security reasons.
```lua
-- Example: Panic button tied to a specific key combination
Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- If Left Shift + Delete are pressed simultaneously
        if Stealth.IsControlPressed(VK_LSHIFT) and Stealth.IsControlJustPressed(VK_DELETE) then
            Stealth.CloseGame()
        end
    end
end)
```

---

## 2. Native Interception (Hooks)

Native hooking globally intercepts engine functions. **Crucially**, any native called *from within* your Isolated script automatically bypasses all hooks.

### Stealth.HookNative(hash, callback)
Registers a global interceptor. 
- **Returns**: `true` on success, `false` if invalid or failed.
- **Callback**: Receives all native arguments. Return a value to block the original native and spoof the result.

**Example 1: Complete Blocking & Spoofing**
```lua
-- Hooking a native that checks entity coordinates
local ok = Stealth.HookNative(0x3FEF770D40960D5A, function(entity)
    print("Intercepted coordinate check for entity: " .. entity)
    -- The host engine will receive vector3(0,0,0) instead of the real coordinates
    return vector3(0.0, 0.0, 0.0) 
end)

if not ok then print("Failed to hook native. Invalid hash.") end
```

**Example 2: Modifying Arguments via Isolated Calls**
```lua
-- Hook a native, modify an argument, and pass it to the real engine.
-- Because Isolated scripts bypass hooks, calling the native here will NOT cause an infinite loop.
Stealth.HookNative(0xABCDEF1234567890, function(ped, boneId, x, y, z)
    local forcedBoneId = 31086 -- Force the head bone
    
    -- Call the REAL native using the modified argument
    local realResult = Citizen.InvokeNative(0xABCDEF1234567890, ped, forcedBoneId, x, y, z)
    
    -- Return the true result back to the host
    return realResult
end)
```

### Stealth.UnhookNative(hash)
Removes an active hook, restoring normal engine behavior.
```lua
-- Example: Temporary hook during a specific action
Stealth.HookNative(0x123..., function() return true end)
Wait(5000)
Stealth.UnhookNative(0x123...) -- Restore original behavior after 5 seconds
```

### Stealth.GetArg(argIdx) / Stealth.GetArgFloat(argIdx)
Alternative manual argument retrieval within a hook context.
```lua
Stealth.HookNative(0x..., function(...)
    -- Index 0 is the first argument
    local entityHandle = Stealth.GetArg(0) 
    local damageAmount = Stealth.GetArgFloat(1)
end)
```

---

## 3. Input & Controls

### Stealth.IsControlPressed(vk)
Checks if a Virtual Key is currently held down.
```lua
-- Example: Continuous action while a key is held
Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- VK_SPACE (0x20)
        if Stealth.IsControlPressed(0x20) then
            -- Apply continuous upward force
        end
    end
end)
```

### Stealth.IsControlJustPressed(vk)
Checks for a single-frame key press (triggers only once per click).
```lua
-- Example: Toggling a state
local isMenuOpen = false
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_INSERT) then
            isMenuOpen = not isMenuOpen
            Stealth.AddNotification("Menu state: " .. tostring(isMenuOpen), Stealth.NOTIFY_INFO)
        end
    end
end)
```

### Stealth.GetCurrentPressedKey() / Stealth.GetCurrentJustPressedKey()
Returns the VK code and the human-readable name of the key.
```lua
-- Example: Implementing a custom keybinder
local boundKey = nil
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if not boundKey then
            local vk, name = Stealth.GetCurrentJustPressedKey()
            if vk then
                boundKey = vk
                Stealth.AddNotification("Bound action to: " .. name, Stealth.NOTIFY_SUCCESS)
            end
        end
    end
end)
```

---

## 4. Primitive Rendering

All drawing functions use **Normalized Coordinates** (0.0 to 1.0).

### Stealth.BeginDraw() & Stealth.EndDraw()
**CRITICAL**: `BeginDraw` must be checked. If it returns false, the frame is not ready. `EndDraw` must be called to push the frame to the screen.

**Example: Drawing a Custom Crosshair**
```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- 1. Initialize Frame
        if not Stealth.BeginDraw() then return end

        -- Center coordinates
        local cx, cy = 0.5, 0.5
        local size = 0.01
        local thick = 2.0

        -- 2. Draw commands
        -- Horizontal line
        Stealth.DrawLine(cx - size, cy, cx + size, cy, 255, 0, 0, 255, thick)
        -- Vertical line
        Stealth.DrawLine(cx, cy - (size * 1.77), cx, cy + (size * 1.77), 255, 0, 0, 255, thick)

        -- 3. Push to screen
        Stealth.EndDraw()
    end
end)
```

### Stealth.WorldToScreen(x, y, z)
Converts 3D coordinates to normalized 2D.
```lua
-- Example: Drawing a marker on a specific coordinate
local targetPos = vector3(100.0, 200.0, 30.0)
local sx, sy = Stealth.WorldToScreen(targetPos.x, targetPos.y, targetPos.z)

if sx and sy then
    -- Point is on screen, draw a box over it
    Stealth.DrawBox(sx, sy, 0.05, 0.05, 0, 255, 0, 255, 1.5)
end
```

---

## 5. Cross-Context & Interop

### Stealth.InjectResource(resourceName, code)
Executes Lua inside a standard engine resource. This allows you to monitor, manipulate, or steal data from **any** running resource without modifying files on disk.

**Example 1: Deep Monitoring / Function Hooking in a Target Resource**
```lua
-- Inject into a hypothetical "target_system" resource
local monitorCode = [[
    -- Store the original function globally within the target resource
    local originalFunction = _G.CalculateDamage
    
    if originalFunction then
        -- Override the global function
        _G.CalculateDamage = function(...)
            local args = {...}
            -- Monitor the arguments being passed inside the resource
            print("[Monitor] CalculateDamage called with args: " .. json.encode(args))
            
            -- Call the original function so the resource doesn't break
            return originalFunction(...)
        end
    end
]]
Stealth.InjectResource("target_system", monitorCode)
```

**Example 2: Global Event Sniffer**
```lua
-- Inject into ANY resource to sniff internal event triggers
local sniffCode = [[
    local originalTrigger = TriggerEvent
    TriggerEvent = function(eventName, ...)
        -- Print every event triggered by this resource
        print("[Sniffer] Caught Event: " .. tostring(eventName))
        originalTrigger(eventName, ...)
    end
]]
Stealth.InjectResource("any_resource_name", sniffCode)
```

### Stealth.ExecuteJS(script)
Runs JavaScript in the global UI context.
```lua
-- Example: Creating a completely custom, unblockable UI overlay
local jsCode = [[
    (function() {
        var overlay = document.createElement('div');
        overlay.id = 'stealth_overlay';
        overlay.innerHTML = '<b>SHADOW LAYER ACTIVE</b>';
        overlay.style.cssText = 'position:fixed;top:20px;right:20px;color:#00FF00;font-family:monospace;font-size:24px;z-index:999999;background:rgba(0,0,0,0.5);padding:10px;border:1px solid #00FF00;pointer-events:none;';
        document.body.appendChild(overlay);
    })();
]]
Stealth.ExecuteJS(jsCode)
```

### Stealth.ExecuteJSWithResult(script, timeout)
Runs JS and waits for a return value.
```lua
-- Example: Extracting data from the host's DOM
local data = Stealth.ExecuteJSWithResult("return document.title;", 1000)
print("The host UI title is: " .. tostring(data))
```

### GetCurrentResourceName() (Global Override)
Always returns `"cfx_internal"`.
```lua
-- Any script attempting to check your resource name will receive this spoofed string
print(GetCurrentResourceName()) -- Outputs: cfx_internal
```

---

## 6. Image & Browser Rendering (DUI)

### Stealth.LoadImage(url, w, h) & Stealth.DrawImage(img, x, y, w, h, a)
```lua
-- Example: Rendering a custom watermark
local logo = Stealth.LoadImage("https://example.com/logo.png", 128, 128)

Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- Draw the image at the top left with 200 alpha
        Stealth.DrawImage(logo, 10, 10, 128, 128, 200)
    end
end)
```

### DUI System (Advanced Browser Frames)
Creates hidden browser instances for complex UI rendering.
```lua
-- Example: Creating a dynamic web-based menu
local myMenu = Stealth.CreateDui("https://my-custom-menu-url.com", 1920, 1080)

-- Show the menu
Stealth.ShowDui(myMenu)

-- Send JSON data to the web page
Stealth.SendDuiMessage(myMenu, { type = "init", config = { color = "red" } })

-- Execute JS directly inside the frame
Stealth.ExecuteDuiScript(myMenu, "document.body.style.backgroundColor = 'rgba(0,0,0,0.8)';")

-- Cleanup when done
-- Stealth.DestroyDui(myMenu)
```

---

## 7. Networking & Async

### Stealth.FetchContent(url)
Blocking GET request (pauses the thread until completion).
```lua
-- Example: Synchronous config loading
local configData = Stealth.FetchContent("https://api.com/config.json")
```

### Stealth.FetchContentAsync(url, callback)
Non-blocking GET request (safe for use in fast loops).
```lua
-- Example: Polling an API without freezing the game
Stealth.FetchContentAsync("https://api.com/status", function(response)
    if response then
        print("Server status: " .. response)
    end
end)
```

---

## 8. Complete Implementation Example: Self-ESP
This master example demonstrates `BeginDraw`, `EndDraw`, `WorldToScreen`, bounding boxes, health bars, and skeleton line rendering in a single, cohesive script.

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
