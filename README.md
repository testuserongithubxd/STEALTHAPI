# Isolated Environment: Complete Developer Reference

The Isolated Environment is a high-performance shadow layer that operates entirely outside the host engine's resource manager. This document is the exhaustive reference for every function, constant, and system behavior available to developers, focused purely on realistic, practical script development.

---

## 1. Native Interception (Hooks)

Native hooking globally intercepts engine functions. **Crucially**, any native called *from within* your Isolated script automatically bypasses all hooks.

### Stealth.HookNative(hash, callback)
Registers a global interceptor. 
- **Returns**: `true` on success, `false` if invalid or failed.
- **Callback**: Receives all native arguments. Return a value to block the original native and spoof the result.

**Realistic Example: Identity Spoofing (`GET_PLAYER_NAME`)**
Spoof your name to all host scripts while secretly logging who is trying to read it.
```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function(player_id)
    -- This executes isolated, meaning we can call the original native safely
    -- without triggering an infinite loop to find our REAL name.
    local realName = Citizen.InvokeNative(0x6D0DE6A7B5DA71F8, player_id)
    
    print(GetCurrentResourceName() .. " tried to get our real name: " .. tostring(realName))
    
    -- Return our spoofed name to the host engine.
    -- Returning a string blocks the original call and injects this data.
    return "stealth-user"
end)

-- Test that the spoof worked by injecting into a standard resource:
Stealth.InjectResource("ANY", [[
    print("The host engine thinks our name is: " .. GetPlayerName(PlayerId()))
]])
```

**Realistic Example: Blocking Anti-Cheat Teleports (`SET_ENTITY_COORDS`)**
Prevent server scripts from teleporting you.
```lua
Stealth.HookNative(0x06843DA7060A026B, function(entity, x, y, z, xAxis, yAxis, zAxis, clearArea)
    -- If the entity being teleported is us...
    if entity == PlayerPedId() then
        print("Blocked host attempt to teleport us to: " .. x .. ", " .. y)
        -- Returning anything (even just 'true') blocks the native call
        return true 
    end
    -- If we return nothing, the native continues normally for other entities
end)
```

### Stealth.UnhookNative(hash)
Removes an active hook, restoring normal engine behavior.
```lua
-- Example: Restore your real name
Stealth.UnhookNative(0x6D0DE6A7B5DA71F8) 
```

### Stealth.GetArg(argIdx) / Stealth.GetArgFloat(argIdx)
Manual argument retrieval within a hook context (useful if dealing with unknown argument structures).

---

## 2. Cross-Context & Interop

### Stealth.InjectResource(resourceName, code)
Executes Lua inside a standard engine resource. This allows you to monitor, manipulate, or steal data from **any** running resource without modifying files on disk.

**Realistic Example: Stealing Framework Data**
Inject into a core framework resource (like ESX or QBCore) and dump their secure global tables back to your isolated environment.
```lua
local stealCode = [[
    -- Search the global scope of this resource for the core framework object
    if _G.ESX then
        -- Trigger an event or send data back to our shadow layer (via NUI or decorators)
        print("Successfully hijacked ESX object from resource: " .. GetCurrentResourceName())
        
        -- Example manipulation: Give ourselves admin internally
        if _G.ESX.GetPlayerData then
            local pData = _G.ESX.GetPlayerData()
            pData.group = 'superadmin'
        end
    end
]]
-- Inject into the core framework resource
Stealth.InjectResource("es_extended", stealCode)
```

### Stealth.ExecuteJS(script)
Runs JavaScript in the global UI context.
```lua
-- Realistic Example: Hiding the host's Watermark/Logo element
local hideJs = [[
    var logo = document.getElementById('server_watermark');
    if (logo) logo.style.display = 'none';
]]
Stealth.ExecuteJS(hideJs)
```

### GetCurrentResourceName() (Global Override)
Always returns `"cfx_internal"`. Any script or hook checking your origin will see this secure alias.

---

## 3. Input & Controls

### Stealth.IsControlPressed(vk)
Checks if a Virtual Key is currently held down.
```lua
-- Realistic Example: Aimbot Lock-On (Right Mouse Button)
Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- VK_RBUTTON (0x02)
        if Stealth.IsControlPressed(0x02) then
            -- Execute memory aim logic here
            -- print("Aimbot active")
        end
    end
end)
```

### Stealth.IsControlJustPressed(vk)
Checks for a single-frame key press (triggers only once per click).
```lua
-- Realistic Example: Menu Toggle (Insert Key)
local menuOpen = false
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_INSERT) then
            menuOpen = not menuOpen
            -- Toggle your UI rendering logic based on 'menuOpen'
        end
    end
end)
```

---

## 4. Primitive Rendering

All drawing functions use **Normalized Coordinates** (0.0 to 1.0).

### Stealth.BeginDraw() & Stealth.EndDraw()
**CRITICAL**: `BeginDraw` must be checked. If it returns false, the frame is not ready. `EndDraw` must be called to push the frame to the screen.

**Realistic Example: Drawing a Custom Crosshair**
```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- 1. Initialize Frame
        if not Stealth.BeginDraw() then return end

        local cx, cy = 0.5, 0.5
        local size = 0.01

        -- 2. Draw commands
        Stealth.DrawLine(cx - size, cy, cx + size, cy, 255, 0, 0, 255, 2.0)
        Stealth.DrawLine(cx, cy - (size * 1.77), cx, cy + (size * 1.77), 255, 0, 0, 255, 2.0)

        -- 3. Push to screen
        Stealth.EndDraw()
    end
end)
```

### Stealth.WorldToScreen(x, y, z)
Converts 3D coordinates to normalized 2D. (See the ESP Example at the bottom for full implementation).

---

## 5. System & Authentication

### Stealth.GetVersion() / Stealth.GetUserID() / Stealth.GetKey()
```lua
-- Realistic Example: Validating your script against a backend
local hwid = Stealth.GetUserID()
local key = Stealth.GetKey()

Stealth.FetchContentAsync("https://my-auth-api.com/verify?key=" .. key .. "&hwid=" .. hwid, function(res)
    if res ~= "OK" then
        Stealth.AddNotification("Unauthorized license key.", Stealth.NOTIFY_ERROR)
        Stealth.CloseGame()
    end
end)
```

### Stealth.CloseGame()
Immediately terminates the engine process.

---

## 6. Networking & Async

### Stealth.FetchContent(url) (Synchronous)
```lua
-- Realistic Example: Loading a JSON config before script start
local cfg = Stealth.FetchContent("https://api.mycheat.com/offsets.json")
local offsets = json.decode(cfg)
```

### Stealth.FetchContentAsync(url, callback) (Asynchronous)
```lua
-- Realistic Example: Webhook logging without stuttering the game
local data = '{"content": "Injected successfully"}'
-- Assuming a custom endpoint that forwards to Discord
Stealth.FetchContentAsync("https://myapi.com/log?msg=" .. data, function() end)
```

---

## 7. Image & Browser Rendering (DUI)

### Stealth.LoadImage(url, w, h) & Stealth.DrawImage(img, x, y, w, h, a)
```lua
-- Realistic Example: Drawing a custom watermark overlay
local logo = Stealth.LoadImage("https://i.imgur.com/example.png", 64, 64)

Citizen.CreateThread(function()
    while true do
        Wait(0)
        -- Draw at top left with slight transparency
        Stealth.DrawImage(logo, 10, 10, 64, 64, 200)
    end
end)
```

### DUI System (Advanced Browser Frames)
```lua
-- Realistic Example: Creating a React/Vue based interface
local menuUI = Stealth.CreateDui("https://my-hosted-menu.com", 1920, 1080)

-- When 'Insert' is pressed, show the menu
Citizen.CreateThread(function()
    local visible = false
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_INSERT) then
            visible = not visible
            if visible then Stealth.ShowDui(menuUI) else Stealth.HideDui(menuUI) end
            
            -- Pass state to the web application
            Stealth.SendDuiMessage(menuUI, { action = "toggle_visibility", state = visible })
        end
    end
end)
```

---

## 8. Complete Master Script: Self-ESP
A fully functional ESP integrating `BeginDraw`, `EndDraw`, `WorldToScreen`, bounding boxes, health bars, and skeleton mapping.

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if not Stealth.BeginDraw() then return end

        repeat
            local myPed = PlayerPedId()
            if not DoesEntityExist(myPed) or IsEntityDead(myPed) then break end
            
            -- GET_FOLLOW_PED_CAM_VIEW_MODE
            if Citizen.InvokeNative(0x8D4D46230B2C353A) == 4 then break end

            -- GET_PED_BONE_COORDS
            local headPos = Citizen.InvokeNative(0x17C07FC640E86B4E, myPed, 31086, 0.0, 0.0, 0.0)
            local lFoot = Citizen.InvokeNative(0x17C07FC640E86B4E, myPed, 14201, 0.0, 0.0, 0.0)
            local rFoot = Citizen.InvokeNative(0x17C07FC640E86B4E, myPed, 52301, 0.0, 0.0, 0.0)
            local footY = math.min(lFoot.z, rFoot.z)

            local hx, hy = Stealth.WorldToScreen(headPos.x, headPos.y, headPos.z + 0.2)
            local fx, fy = Stealth.WorldToScreen(headPos.x, headPos.y, footY - 0.1)

            if hx and fx then
                local height = math.abs(fy - hy)
                local width = height * 0.35
                local bx = (hx + fx) * 0.5
                local by = (hy + fy) * 0.5
                Stealth.DrawBox(bx, by, width, height, 51, 115, 230, 200, 1.0)

                -- GET_ENTITY_HEALTH
                local hp = Citizen.InvokeNative(0xEEF059FAD016D209, myPed) - 100
                -- GET_ENTITY_MAX_HEALTH
                local maxHp = Citizen.InvokeNative(0x15D757609D01783D, myPed) - 100
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
                local p1 = Citizen.InvokeNative(0x17C07FC640E86B4E, myPed, pair[1], 0.0, 0.0, 0.0)
                local p2 = Citizen.InvokeNative(0x17C07FC640E86B4E, myPed, pair[2], 0.0, 0.0, 0.0)
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
