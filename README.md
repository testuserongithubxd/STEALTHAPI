# Isolated Environment: Complete Developer Reference

The Isolated Environment is a high-performance shadow layer that operates entirely outside the host engine's resource manager. This document is the exhaustive reference for every function, constant, and system behavior available to developers.

---

## 1. Native Interception (Hooks)

Native hooking globally intercepts engine functions. Any native called *from within* your Isolated script automatically bypasses all hooks, meaning you can safely call standard native wrappers like `GetPlayerName()` inside your hook without causing infinite loops.

### Stealth.HookNative(hash, callback)
Registers a global interceptor. 
- **Returns**: `true` on success, `false` if invalid or failed.
- **Callback**: Receives all native arguments. Return a value to block the original native and spoof the result.

```lua
Stealth.HookNative(0x3FEF770D40960D5A, function(entity)
    print(entity)
    print("0x3FEF770D40960D5A called")
    return vector3(0,0,0)
end)
```

```lua
local ok = Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return "FakePlayerName"
end)

if ok then
    print("GetPlayerName hook active!")
else
    print("Failed to hook — invalid hash or slots full")
end
```

### Stealth.UnhookNative(hash)
Removes an active hook, restoring normal engine behavior.
```lua
Stealth.UnhookNative(0x6D0DE6A7B5DA71F8) 
```

### Stealth.GetArg(argIdx) / Stealth.GetArgFloat(argIdx)
Manual argument retrieval within a hook context.

---

## 2. Cross-Context & Interop

### Stealth.InjectResource(resourceName, code)
Executes Lua inside a standard engine resource. Using `"any"` will target any available resource.

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function(player_id)
    print(GetCurrentResourceName() .. " tried to get our real name: " .. GetPlayerName(player_id))
    return "macho-man"
end)

Stealth.InjectResource("any", [[
    print(GetPlayerName(PlayerId()))
]])
```

### Stealth.ExecuteJS(script)
Runs JavaScript in the global UI context.
```lua
Stealth.ExecuteJS("document.body.style.backgroundColor = 'red'")
```

### GetCurrentResourceName() (Global Override)
Always returns `"cfx_internal"`. Any script or hook checking your origin will see this secure alias.

---

## 3. Input & Controls

### Stealth.IsControlPressed(vk)
Checks if a Virtual Key is currently held down.
```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlPressed(0x02) then
            -- Action while holding Right Mouse Button
        end
    end
end)
```

### Stealth.IsControlJustPressed(vk)
Checks for a single-frame key press (triggers only once per click).
```lua
local menuOpen = false
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_INSERT) then
            menuOpen = not menuOpen
        end
    end
end)
```

### Stealth.GetCurrentPressedKey() / Stealth.GetCurrentJustPressedKey()
Returns the VK code and the human-readable name of the key.
```lua
local vk, name = Stealth.GetCurrentJustPressedKey()
if vk then print("Key Pressed: " .. name) end
```

---

## 4. Primitive Rendering

All drawing functions use **Normalized Coordinates** (0.0 to 1.0).

### Stealth.BeginDraw() & Stealth.EndDraw()
**CRITICAL**: `BeginDraw` must be checked. If it returns false, the frame is not ready. `EndDraw` must be called to push the frame to the screen.

```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if not Stealth.BeginDraw() then return end

        Stealth.DrawLine(0.49, 0.5, 0.51, 0.5, 255, 0, 0, 255, 2.0)
        Stealth.DrawLine(0.5, 0.48, 0.5, 0.52, 255, 0, 0, 255, 2.0)

        Stealth.EndDraw()
    end
end)
```

### Stealth.WorldToScreen(x, y, z)
Converts 3D coordinates to normalized 2D. 
```lua
local sx, sy = Stealth.WorldToScreen(100.0, 200.0, 30.0)
if sx and sy then
    Stealth.DrawBox(sx, sy, 0.05, 0.05, 0, 255, 0, 255, 1.5)
end
```

---

## 5. System & Authentication

### Stealth.GetVersion() / Stealth.GetUserID() / Stealth.GetKey()
```lua
local hwid = Stealth.GetUserID()
local key = Stealth.GetKey()
local ver = Stealth.GetVersion()
```

### Stealth.CloseGame()
Immediately terminates the engine process.

---

## 6. Networking & Async

### Stealth.FetchContent(url) (Synchronous)
```lua
local cfg = Stealth.FetchContent("https://api.example.com/data.json")
```

### Stealth.FetchContentAsync(url, callback) (Asynchronous)
```lua
Stealth.FetchContentAsync("https://api.example.com/status", function(response)
    print("Response: " .. tostring(response))
end)
```

---

## 7. Image & Browser Rendering (DUI)

### Stealth.LoadImage(url, w, h) & Stealth.DrawImage(img, x, y, w, h, a)
```lua
local logo = Stealth.LoadImage("https://i.imgur.com/example.png", 64, 64)

Citizen.CreateThread(function()
    while true do
        Wait(0)
        Stealth.DrawImage(logo, 10, 10, 64, 64, 255)
    end
end)
```

### DUI System (Advanced Browser Frames)
```lua
local menuUI = Stealth.CreateDui("https://my-hosted-menu.com", 1920, 1080)

Citizen.CreateThread(function()
    local visible = false
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_INSERT) then
            visible = not visible
            if visible then Stealth.ShowDui(menuUI) else Stealth.HideDui(menuUI) end
            Stealth.SendDuiMessage(menuUI, { action = "toggle", state = visible })
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
