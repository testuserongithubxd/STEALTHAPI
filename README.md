# Isolated Environment

## 1. Hooks

### Stealth.HookNative(hash, callback)
```lua
Stealth.HookNative(0xF25DF915FA38C5F3, function(ped)
    print("RemoveAllPedWeapons: " .. tostring(ped))
    Stealth.SetArg(ped, 0) 
end)
```

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function()
    return "FakePlayerName"
end)
```

### Stealth.SetArg(target, value)
`target` is the 0-based index or the current value.

```lua
Stealth.HookNative(0xF25DF915FA38C5F3, function(ped)
    Stealth.SetArg(ped, 0) 
end)
```

```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function(player)
    Stealth.SetArg(0, 0) -- 0 is the first argument if you prefer this somehow
end)
```

### Stealth.UnhookNative(hash)
```lua
Stealth.UnhookNative(0x6D0DE6A7B5DA71F8) 
```

### Stealth.GetArg(argIdx) / Stealth.GetArgFloat(argIdx)

## 2. Resources

### Stealth.InjectResource(resourceName, code)
```lua
Stealth.HookNative(0x6D0DE6A7B5DA71F8, function(player_id)
    return "FakePlayerName"
end)

Stealth.InjectResource("any", [[
    print(GetPlayerName(PlayerId()))
]])
```

### Stealth.ExecuteJS(script)
```lua
Stealth.ExecuteJS("console.log('Hello World')") -- check NUI
```

## 3. Input

### Stealth.IsControlPressed(vk)
```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlPressed(0x02) then
            -- Right Mouse
        end
    end
end)
```

### Stealth.IsControlJustPressed(vk)
```lua
Citizen.CreateThread(function()
    while true do
        Wait(0)
        if Stealth.IsControlJustPressed(VK_INSERT) then
            -- Insert
        end
    end
end)
```

### Stealth.GetCurrentPressedKey() / Stealth.GetCurrentJustPressedKey()

## 4. Rendering

### Stealth.BeginDraw() / Stealth.EndDraw()
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

## 5. System

### Stealth.GetVersion() / Stealth.GetUserID() / Stealth.GetKey()

### Stealth.CloseGame()

## 6. Networking

### Stealth.FetchContent(url)

### Stealth.FetchContentAsync(url, callback)

## 7. DUI

### Stealth.LoadImage(url, w, h) / Stealth.DrawImage(img, x, y, w, h, a)
```lua
local logo = Stealth.LoadImage("https://i.imgur.com/example.png", 64, 64)

Citizen.CreateThread(function()
    while true do
        Wait(0)
        Stealth.DrawImage(logo, 10, 10, 64, 64, 255)
    end
end)
```

### Stealth.CreateDui(url, w, h)
```lua
local menuUI = Stealth.CreateDui("https://example.com", 1920, 1080)
```

## 8. ESP Script

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
