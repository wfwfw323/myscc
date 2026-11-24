------------------------------------------------------------
-- FLAME LOCK v2 — AUTO OFFSET X + SMART PRED + FLAME HP
------------------------------------------------------------

local Services = {
    Players             = game:GetService("Players"),
    RunService          = game:GetService("RunService"),
    UserInputService    = game:GetService("UserInputService"),
    Workspace           = game:GetService("Workspace"),
    VirtualInputManager = game:GetService("VirtualInputManager"),
}

local LocalPlayer  = Services.Players.LocalPlayer
local Camera       = workspace.CurrentCamera
local scriptActive = true

local Connections = {}
local function AddConnection(conn)
    table.insert(Connections, conn)
    return conn
end

------------------------------------------------------------
-- VELOCITY HISTORY
------------------------------------------------------------
local VelocityHistory  = {}
local VEL_HISTORY_SIZE = 6

local function pushVelocity(plr, vel)
    if not VelocityHistory[plr] then VelocityHistory[plr] = {} end
    local h = VelocityHistory[plr]
    h[#h + 1] = vel
    while #h > VEL_HISTORY_SIZE do table.remove(h, 1) end
end

local function getSmoothedVelocity(plr)
    local h = VelocityHistory[plr]
    if not h or #h == 0 then return Vector3.new() end
    local sum = Vector3.new()
    for _, v in ipairs(h) do sum += v end
    return sum / #h
end

------------------------------------------------------------
-- CONFIG
------------------------------------------------------------
local Config = {
    FlameLock = {
        Enabled   = true,
        Target    = nil,
        Desync    = true,

        Prediction       = true,
        PredictionAmount = 0.100,

        AutoSwitch       = true,
        WallCheck        = true,
        KOCheck          = true,
        TeamCheck        = true,
        WhitelistCheck   = true,
        Whitelist        = {},
        HoldM1ToTarget   = true,
        IgnoreWhenEating = true,

        Smoothness = 1.0,

        VerticalLock          = false,
        VerticalLockThreshold = 15,

        AutoOffset = {
            Enabled   = true,
            CloseMax  = 15,
            MediumMax = 40,
            CloseX    = 2.35,
            MediumX   = 1.85,
            FarX      = 3.20,
        },

        SwitchHysteresis = 70,

        FOV = {
            Visible     = false,
            Radius      = 300,
            IdleColor   = Color3.fromRGB(255, 255, 255),
            LockedColor = Color3.fromRGB(255, 70,  70),
            Thickness   = 1.5,
            ShowDot     = false,
            DotRadius   = 3,
        },
    },

    -- FLAME HP DRAIN
    FlameHP = {
        Enabled       = false,
        DamagePerHit  = 5,       -- HP снимается за каждый Heartbeat пока струя на тебе
        CheckRadius   = 60,      -- studs — радиус поиска flame-партов
        RayLength     = 25,      -- длина луча от flame-парта
        Cooldown      = 0.08,    -- секунды между тиками урона
    },

    Reset = {
        Enabled = true,
        Keybind = Enum.KeyCode.Z,
    },
}

local DaHoodFoods = {
    "hamburger","taco","pizza","cranberry","donut",
    "hotdog","chicken","water","lettuce","cabbage",
    "starblox","bloxy","icecream","popcorn","protein",
    "gatorade","lemonade","cola","chocblox","steak"
}

------------------------------------------------------------
-- FOV DRAWING
------------------------------------------------------------
local FOVOuter        = Drawing.new("Circle")
FOVOuter.Color        = Color3.fromRGB(0, 0, 0)
FOVOuter.Thickness    = 3
FOVOuter.Filled       = false
FOVOuter.Transparency = 0.55
FOVOuter.Visible      = Config.FlameLock.FOV.Visible
FOVOuter.Radius       = Config.FlameLock.FOV.Radius

local FOVCircle        = Drawing.new("Circle")
FOVCircle.Color        = Config.FlameLock.FOV.IdleColor
FOVCircle.Thickness    = Config.FlameLock.FOV.Thickness
FOVCircle.Filled       = false
FOVCircle.Transparency = 1
FOVCircle.Visible      = Config.FlameLock.FOV.Visible
FOVCircle.Radius       = Config.FlameLock.FOV.Radius

local FOVDot           = Drawing.new("Circle")
FOVDot.Color           = Config.FlameLock.FOV.LockedColor
FOVDot.Thickness       = 1
FOVDot.Filled          = true
FOVDot.Transparency    = 1
FOVDot.Radius          = Config.FlameLock.FOV.DotRadius
FOVDot.Visible         = false

------------------------------------------------------------
-- UTILS & CHECKS
------------------------------------------------------------
local function isValidVector(vec)
    return vec and (vec.X == vec.X) and (vec.Y == vec.Y) and (vec.Z == vec.Z)
end

local function hasWallBetween(startPos, targetPart)
    if not (startPos and targetPart) then return true end
    local ignoreList = { LocalPlayer.Character, targetPart.Parent }
    local ray = Ray.new(startPos, (targetPart.Position - startPos).Unit * (targetPart.Position - startPos).Magnitude)
    local hit  = Services.Workspace:FindPartOnRayWithIgnoreList(ray, ignoreList)
    if hit and hit.CanCollide and hit.Transparency < 1 then return true end
    return false
end

local function isPlayerDead(plr)
    if not plr or not plr.Character then return true end
    local hum = plr.Character:FindFirstChild("Humanoid")
    return not hum or hum.Health <= 0
end

local function isKnockedRaw(plr)
    if not plr or not plr.Character then return false end
    local be = plr.Character:FindFirstChild("BodyEffects")
    local ko = be and be:FindFirstChild("K.O")
    return ko and ko.Value or false
end

local function isSameCrew(plr)
    if not plr or not Config.FlameLock.TeamCheck then return false end
    if plr == LocalPlayer then return true end
    if LocalPlayer.Team and plr.Team and LocalPlayer.Team == plr.Team then return true end
    local myData     = LocalPlayer:FindFirstChild("DataFolder")
    local targetData = plr:FindFirstChild("DataFolder")
    if myData and targetData then
        local myInfo = myData.Information
        local tInfo  = targetData.Information
        local myCrew = myInfo and myInfo:FindFirstChild("Crew")
        local tCrew  = tInfo  and tInfo:FindFirstChild("Crew")
        if myCrew and tCrew and myCrew.Value ~= "" and myCrew.Value == tCrew.Value then
            return true
        end
    end
    return false
end

local function isWhitelisted(plr)
    if not plr or not Config.FlameLock.WhitelistCheck then return false end
    local n = string.lower(plr.Name)
    local d = string.lower(plr.DisplayName)
    for _, name in ipairs(Config.FlameLock.Whitelist) do
        local ln = string.lower(name)
        if n == ln or d == ln then return true end
    end
    return false
end

local function isHoldingFood(char)
    if not char then return false end
    for _, child in ipairs(char:GetChildren()) do
        if child:IsA("Tool") then
            local tn = string.lower(child.Name):gsub("[%s%[%]]", "")
            for _, food in ipairs(DaHoodFoods) do
                if string.find(tn, food) then return true end
            end
        end
    end
    return false
end

local function isBehindCharacter(targetPart)
    if not targetPart or not LocalPlayer.Character then return false end
    local root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not root then return false end
    local toTarget = (targetPart.Position - root.Position).Unit
    return root.CFrame.LookVector:Dot(toTarget) < -0.1
end

------------------------------------------------------------
-- PING
------------------------------------------------------------
local function getPing()
    local ok, ping = pcall(function() return Services.Players:GetNetworkPing() end)
    if ok and ping and ping > 0 then return math.floor(ping * 1000) end
    local ok2, stats = pcall(function() return game:GetService("Stats") end)
    if ok2 and stats then
        local net = stats:FindFirstChild("Network")
        if net then
            local sst = net:FindFirstChild("ServerStepTime")
            if sst and type(sst.Value) == "number" and sst.Value > 0 then
                return math.clamp(math.floor(sst.Value * 2 * 1000), 10, 800)
            end
        end
    end
    return 80
end

local function getTargetPart(plr)
    if not plr or not plr.Character then return nil end
    return plr.Character:FindFirstChild("HumanoidRootPart")
end

------------------------------------------------------------
-- SMART PREDICTION — dist-based, real-time
------------------------------------------------------------
-- Возвращает (predMs, bonusMs, zoneName)
local function calcSmartPred(dist)
    local ping    = getPing()
    local base    = math.clamp(ping + 10, 80, 200)
    local bonus   = 0
    local zone    = "far"
    if dist < 15 then
        bonus = 100
        zone  = "close"
    elseif dist < 40 then
        bonus = 10
        zone  = "medium"
    end
    local total = math.clamp(base + bonus, 80, 400)
    return total, base, bonus, zone
end

local function getAutoPredictionTime(dist)
    local total, _, _, _ = calcSmartPred(dist)
    return total / 1000
end

local function getPredictedPosition(part, dist, plr)
    if not part or not isValidVector(part.Position) then
        return part and part.Position or Vector3.new()
    end
    local velocity = part.AssemblyLinearVelocity or Vector3.new()
    if not isValidVector(velocity) then velocity = Vector3.new() end
    if plr then
        pushVelocity(plr, velocity)
        velocity = getSmoothedVelocity(plr)
    end
    return part.Position + (velocity * getAutoPredictionTime(dist))
end

------------------------------------------------------------
-- TARGET SPEED & AUTO OFFSET X
------------------------------------------------------------
local function getTargetSpeed(targetPart)
    if not targetPart then return 0 end
    local vel = targetPart.AssemblyLinearVelocity
    if not isValidVector(vel) then return 0 end
    -- Horizontal movement speed (X/Z) in studs/sec
    return Vector3.new(vel.X, 0, vel.Z).Magnitude
end

local function getOffsetZone(dist)
    local ao = Config.FlameLock.AutoOffset
    if dist <= ao.CloseMax  then return "close",  ao.CloseX  end
    if dist <= ao.MediumMax then return "medium", ao.MediumX end
    return "far", ao.FarX
end

local function applySpeedOffsetRule(baseOffset, speed)
    if speed <= 10 then
        return baseOffset - 0.12
    elseif speed > 16 then
        return baseOffset + 0.12
    end
    return baseOffset
end

local function getAutoOffsetX(dist, targetPart)
    if not Config.FlameLock.AutoOffset.Enabled then return 0 end
    local _, x = getOffsetZone(dist)
    local speed = getTargetSpeed(targetPart)
    return applySpeedOffsetRule(x, speed)
end

------------------------------------------------------------
-- FLAME HP DRAIN — реальный урон через raycast
------------------------------------------------------------
local FLAME_PART_NAMES = {
    "flamestream", "flame", "fire", "flamethrower",
    "flameball", "burnpart", "firepart",
}

local function isFlameRelated(name)
    local n = string.lower(name)
    for _, fn in ipairs(FLAME_PART_NAMES) do
        if string.find(n, fn) then return true end
    end
    return false
end

local lastFlameDmgTime = 0

local function checkFlameHP()
    if not Config.FlameHP.Enabled then return end

    local char = LocalPlayer.Character
    if not char then return end
    local myRoot = char:FindFirstChild("HumanoidRootPart")
    local hum    = char:FindFirstChild("Humanoid")
    if not myRoot or not hum or hum.Health <= 0 then return end

    local now = tick()
    if (now - lastFlameDmgTime) < Config.FlameHP.Cooldown then return end

    local myPos       = myRoot.Position
    local checkRadius = Config.FlameHP.CheckRadius
    local rayLength   = Config.FlameHP.RayLength

    local function scanDescendants(parent)
        for _, obj in ipairs(parent:GetDescendants()) do
            if not obj:IsA("BasePart") then continue end
            if not isFlameRelated(obj.Name) then continue end

            if obj:IsDescendantOf(char) then continue end

            local partPos = obj.Position
            if (partPos - myPos).Magnitude > checkRadius then continue end

            local dir = nil
            local vel = obj.AssemblyLinearVelocity
            if vel and vel.Magnitude > 2 then
                dir = vel.Unit
            else
                local model = obj.Parent
                if model and model:IsA("Model") then
                    local root = model:FindFirstChild("HumanoidRootPart")
                    if root then
                        dir = root.CFrame.LookVector
                    end
                end
            end

            if not dir then
                local toMe = (myPos - partPos)
                if toMe.Magnitude < 0.1 then continue end
                dir = toMe.Unit
            end

            local rayOrigin = partPos
            local rayDir    = dir * rayLength

            local raycastParams = RaycastParams.new()
            raycastParams.FilterType = Enum.RaycastFilterType.Whitelist
            raycastParams.FilterDescendantsInstances = { char }

            local result = workspace:Raycast(rayOrigin, rayDir, raycastParams)

            if result and result.Instance and result.Instance:IsDescendantOf(char) then
                hum.Health = math.max(0, hum.Health - Config.FlameHP.DamagePerHit)
                lastFlameDmgTime = now
                return
            end
        end
    end

    for _, plr in ipairs(Services.Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        if not plr.Character then continue end
        if (plr.Character:GetPivot().Position - myPos).Magnitude > checkRadius + 20 then continue end
        scanDescendants(plr.Character)
    end

    scanDescendants(workspace)
end

------------------------------------------------------------
-- RESET CHARACTER
------------------------------------------------------------
local VIM           = Services.VirtualInputManager
local resetDebounce = false

local function doResetCharacter()
    if resetDebounce then return end
    resetDebounce = true
    VIM:SendKeyEvent(true,  Enum.KeyCode.R,      false, game)
    VIM:SendKeyEvent(false, Enum.KeyCode.R,      false, game)
    task.wait(0.09)
    VIM:SendKeyEvent(true,  Enum.KeyCode.Escape, false, game)
    VIM:SendKeyEvent(false, Enum.KeyCode.Escape, false, game)
    task.wait(0.09)
    VIM:SendKeyEvent(true,  Enum.KeyCode.R,      false, game)
    VIM:SendKeyEvent(false, Enum.KeyCode.R,      false, game)
    task.wait(0.09)
    VIM:SendKeyEvent(true,  Enum.KeyCode.Return, false, game)
    VIM:SendKeyEvent(false, Enum.KeyCode.Return, false, game)
    task.wait(0.6)
    resetDebounce = false
end

------------------------------------------------------------
-- MAIN LOOP — RENDER
------------------------------------------------------------
local fakeCFrame = nil

AddConnection(Services.RunService.RenderStepped:Connect(function()
    if not scriptActive then
        FOVCircle.Visible = false
        FOVOuter.Visible  = false
        FOVDot.Visible    = false
        return
    end

    local mousePos  = Services.UserInputService:GetMouseLocation()
    local radius    = Config.FlameLock.FOV.Radius
    local hasTarget = Config.FlameLock.Target ~= nil
    local fovVis    = Config.FlameLock.FOV.Visible

    FOVCircle.Position  = mousePos
    FOVCircle.Radius    = radius
    FOVCircle.Thickness = Config.FlameLock.FOV.Thickness
    FOVCircle.Color     = hasTarget
        and Config.FlameLock.FOV.LockedColor
        or  Config.FlameLock.FOV.IdleColor
    FOVCircle.Visible   = fovVis

    FOVOuter.Position   = mousePos
    FOVOuter.Radius     = radius + 1
    FOVOuter.Visible    = fovVis

    FOVDot.Position     = mousePos
    FOVDot.Radius       = Config.FlameLock.FOV.DotRadius
    FOVDot.Visible      = Config.FlameLock.FOV.ShowDot and hasTarget and fovVis

    if fakeCFrame then
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root then root.CFrame = fakeCFrame end
        fakeCFrame = nil
    end

    if Config.FlameLock.Enabled and Config.FlameLock.VerticalLock and Config.FlameLock.Target then
        local char  = LocalPlayer.Character
        local root  = char and char:FindFirstChild("HumanoidRootPart")
        local tPart = getTargetPart(Config.FlameLock.Target)

        if root and tPart and not isPlayerDead(LocalPlayer) then
            local deltaY = tPart.Position.Y - root.Position.Y
            if math.abs(deltaY) <= Config.FlameLock.VerticalLockThreshold then
                local camCF = Camera.CFrame
                local look  = camCF.LookVector
                local flat  = Vector3.new(look.X, 0, look.Z)
                if flat.Magnitude > 0.001 and isValidVector(flat) then
                    Camera.CFrame = CFrame.lookAt(camCF.Position, camCF.Position + flat.Unit)
                end
            end
        end
    end
end))

------------------------------------------------------------
-- PLAYERS IN FOV
------------------------------------------------------------
local function getPlayersInFOV()
    local mouseLocation = Services.UserInputService:GetMouseLocation()
    local validPlayers  = {}

    for _, plr in ipairs(Services.Players:GetPlayers()) do
        if plr == LocalPlayer or not plr.Character then continue end
        local targetLimb = getTargetPart(plr)
        if not targetLimb then continue end
        if isPlayerDead(plr)       then continue end
        if isBehindCharacter(targetLimb) then continue end
        if Config.FlameLock.KOCheck        and isKnockedRaw(plr)   then continue end
        if Config.FlameLock.TeamCheck      and isSameCrew(plr)      then continue end
        if Config.FlameLock.WhitelistCheck and isWhitelisted(plr)   then continue end

        local screenPos, onScreen = Camera:WorldToViewportPoint(targetLimb.Position)
        if not onScreen then continue end

        local dist2D = (Vector2.new(screenPos.X, screenPos.Y) - mouseLocation).Magnitude
        if dist2D > Config.FlameLock.FOV.Radius then continue end

        if Config.FlameLock.WallCheck and hasWallBetween(Camera.CFrame.Position, targetLimb) then continue end

        validPlayers[#validPlayers + 1] = { Player = plr, Dist = dist2D, Part = targetLimb }
    end

    table.sort(validPlayers, function(a, b) return a.Dist < b.Dist end)
    return validPlayers
end

------------------------------------------------------------
-- MAIN LOOP — HEARTBEAT
------------------------------------------------------------
AddConnection(Services.RunService.Heartbeat:Connect(function()
    if not scriptActive then return end

    -- FLAME HP CHECK — каждый тик
    checkFlameHP()

    if Config.FlameLock.IgnoreWhenEating and isHoldingFood(LocalPlayer.Character) then
        Config.FlameLock.Target = nil
        return
    end

    if not Config.FlameLock.Enabled then return end

    local isM1Held = false
    if Config.FlameLock.HoldM1ToTarget then
        isM1Held = Services.UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
        if not isM1Held then Config.FlameLock.Target = nil end
    end

    local inFovPlayers  = getPlayersInFOV()
    local canAutoSwitch = Config.FlameLock.AutoSwitch and
        (not Config.FlameLock.HoldM1ToTarget or isM1Held)

    if canAutoSwitch then
        local stillInFOV        = false
        local currentTargetDist = math.huge

        if Config.FlameLock.Target then
            for _, data in ipairs(inFovPlayers) do
                if data.Player == Config.FlameLock.Target then
                    stillInFOV        = true
                    currentTargetDist = data.Dist
                    break
                end
            end
            if not stillInFOV then Config.FlameLock.Target = nil end
        end

        if Config.FlameLock.Target and stillInFOV then
            if #inFovPlayers > 1 then
                local best = inFovPlayers[1]
                if best.Player ~= Config.FlameLock.Target
                    and best.Dist < (currentTargetDist - Config.FlameLock.SwitchHysteresis) then
                    Config.FlameLock.Target = best.Player
                end
            end
        elseif not Config.FlameLock.Target then
            if #inFovPlayers > 0 then
                Config.FlameLock.Target = inFovPlayers[1].Player
            end
        end

    elseif not Config.FlameLock.Target and Config.FlameLock.HoldM1ToTarget and isM1Held then
        if #inFovPlayers > 0 then
            Config.FlameLock.Target = inFovPlayers[1].Player
        end
    end

    if Config.FlameLock.Target and Config.FlameLock.Desync then
        local targetLimb = getTargetPart(Config.FlameLock.Target)
        local char       = LocalPlayer.Character
        local root       = char and char:FindFirstChild("HumanoidRootPart")

        if root and targetLimb and not isPlayerDead(LocalPlayer) then
            local dist3D = (root.Position - targetLimb.Position).Magnitude

            if dist3D < 1500 then
                fakeCFrame = root.CFrame

                local pred = targetLimb.Position
                if Config.FlameLock.Prediction then
                    pred = getPredictedPosition(targetLimb, dist3D, Config.FlameLock.Target)
                end

                local offsetX   = getAutoOffsetX(dist3D, targetLimb)
                local camRight  = Camera.CFrame.RightVector
                local offsetVec = camRight * offsetX
                pred = pred + offsetVec

                local currentPos = root.Position
                local lookAt     = Vector3.new(pred.X, currentPos.Y, pred.Z)

                if isValidVector(lookAt) and (lookAt - currentPos).Magnitude > 0.5 then
                    local targetCF = CFrame.lookAt(currentPos, lookAt)
                    if Config.FlameLock.Smoothness < 1 then
                        root.CFrame = root.CFrame:Lerp(targetCF, Config.FlameLock.Smoothness)
                    else
                        root.CFrame = targetCF
                    end
                end
            end
        end
    end
end))

------------------------------------------------------------
-- GUI (LINORIALIB)
------------------------------------------------------------
local Library
pcall(function()
    Library = loadstring(game:HttpGet('https://raw.githubusercontent.com/violin-suzutsuki/LinoriaLib/main/Library.lua'))()
end)

if Library then
    local Window = Library:CreateWindow({
        Title        = 'Flame Lock v2',
        Center       = true,
        AutoShow     = true,
        TabPadding   = 8,
        MenuFadeTime = 0.2,
    })

    local MainTab = Window:AddTab('Flame Combat')

    local UI = {
        Target         = nil,
        Dist           = nil,
        DeltaY         = nil,
        Ping           = nil,
        Pred           = nil,
        OffsetLive     = nil,
        FlameHPStatus  = nil,
    }

    ----------------------------------------------------------------
    -- LEFT: Flame Lock Settings
    ----------------------------------------------------------------
    local FLMainBox = MainTab:AddLeftGroupbox('Flame Lock Settings')
    do
        FLMainBox:AddToggle('FLEnabled', {
            Text     = 'Enable Flame Lock',
            Default  = Config.FlameLock.Enabled,
            Callback = function(v)
                Config.FlameLock.Enabled = v
                if not v then Config.FlameLock.Target = nil end
            end
        })
        FLMainBox:AddToggle('FLIgnoreFood', {
            Text     = 'Disable When Eating',
            Default  = Config.FlameLock.IgnoreWhenEating,
            Callback = function(v) Config.FlameLock.IgnoreWhenEating = v end
        })
        FLMainBox:AddToggle('FLDesync', {
            Text     = 'Use Desync Aimbot',
            Default  = Config.FlameLock.Desync,
            Callback = function(v) Config.FlameLock.Desync = v end
        })
        FLMainBox:AddToggle('FLHoldM1', {
            Text     = 'Hold M1 to Target',
            Default  = Config.FlameLock.HoldM1ToTarget,
            Callback = function(v)
                Config.FlameLock.HoldM1ToTarget = v
                if not v then Config.FlameLock.Target = nil end
            end
        })
        FLMainBox:AddToggle('FLAutoSwitch', {
            Text     = 'Auto Switch Targets',
            Default  = Config.FlameLock.AutoSwitch,
            Callback = function(v) Config.FlameLock.AutoSwitch = v end
        })
        FLMainBox:AddDivider()
        FLMainBox:AddToggle('FLKOCheck', {
            Text     = 'KO Check',
            Default  = Config.FlameLock.KOCheck,
            Callback = function(v) Config.FlameLock.KOCheck = v end
        })
        FLMainBox:AddToggle('FLWallCheck', {
            Text     = 'Wall Check',
            Default  = Config.FlameLock.WallCheck,
            Callback = function(v) Config.FlameLock.WallCheck = v end
        })
        FLMainBox:AddToggle('FLTeamCheck', {
            Text     = 'Team Check',
            Default  = Config.FlameLock.TeamCheck,
            Callback = function(v) Config.FlameLock.TeamCheck = v end
        })
    end

    ----------------------------------------------------------------
    -- LEFT: Live Info
    ----------------------------------------------------------------
    local FLInfoBox = MainTab:AddLeftGroupbox('Live Info')
    do
        UI.Target     = FLInfoBox:AddLabel('Target: None')
        UI.Dist       = FLInfoBox:AddLabel('3D Distance: —')
        UI.DeltaY     = FLInfoBox:AddLabel('Height dY: —  (VLock: —)')
        UI.Ping       = FLInfoBox:AddLabel('Ping: — ms')
        UI.Pred       = FLInfoBox:AddLabel('Pred: — ms  [—]')
        UI.OffsetLive = FLInfoBox:AddLabel('Offset X: —  [—]')
        UI.FlameHPStatus = FLInfoBox:AddLabel('Flame HP: OFF')
    end

    ----------------------------------------------------------------
    -- LEFT: Smart Prediction
    ----------------------------------------------------------------
    local FLPredBox = MainTab:AddLeftGroupbox('Smart Prediction')
    do
        FLPredBox:AddToggle('FLPred', {
            Text     = 'Enable Prediction',
            Default  = Config.FlameLock.Prediction,
            Callback = function(v) Config.FlameLock.Prediction = v end
        })
        FLPredBox:AddDivider()
        FLPredBox:AddLabel('far  (>= 40 studs): base only')
        FLPredBox:AddLabel('med  (< 40 studs):  base + 10 ms')
        FLPredBox:AddLabel('close(< 15 studs):  base + 100 ms')
        FLPredBox:AddLabel('base = clamp(ping+10, 80, 200)')
        FLPredBox:AddLabel('Velocity averaged over last 6 frames')
    end

    ----------------------------------------------------------------
    -- LEFT: Flame HP Drain
    ----------------------------------------------------------------
    local FlameHPBox = MainTab:AddLeftGroupbox('Flame HP Drain')
    do
        FlameHPBox:AddToggle('FlameHPEnabled', {
            Text     = 'Enable Flame HP Drain',
            Default  = Config.FlameHP.Enabled,
            Callback = function(v) Config.FlameHP.Enabled = v end
        })
        FlameHPBox:AddDivider()
        FlameHPBox:AddSlider('FlameHPDmg', {
            Text     = 'Damage Per Tick',
            Default  = Config.FlameHP.DamagePerHit,
            Min      = 1, Max = 50, Rounding = 0,
            Callback = function(v) Config.FlameHP.DamagePerHit = v end
        })
        FlameHPBox:AddSlider('FlameHPCooldown', {
            Text     = 'Tick Cooldown (ms)',
            Default  = Config.FlameHP.Cooldown * 1000,
            Min      = 20, Max = 500, Rounding = 0,
            Callback = function(v) Config.FlameHP.Cooldown = v / 1000 end
        })
        FlameHPBox:AddSlider('FlameHPRadius', {
            Text     = 'Detection Radius (studs)',
            Default  = Config.FlameHP.CheckRadius,
            Min      = 10, Max = 150, Rounding = 0,
            Callback = function(v) Config.FlameHP.CheckRadius = v end
        })
        FlameHPBox:AddSlider('FlameHPRayLen', {
            Text     = 'Ray Length (studs)',
            Default  = Config.FlameHP.RayLength,
            Min      = 5, Max = 60, Rounding = 0,
            Callback = function(v) Config.FlameHP.RayLength = v end
        })
        FlameHPBox:AddDivider()
        FlameHPBox:AddLabel('Raycast: flame part direction → hits you?')
        FlameHPBox:AddLabel('No hit on you = no damage.')
    end

    ----------------------------------------------------------------
    -- RIGHT: Auto Offset X (Zone Snap)
    ----------------------------------------------------------------
    local FLAutoOffBox = MainTab:AddRightGroupbox('Auto Offset X (Zone Snap)')
    do
        FLAutoOffBox:AddToggle('FLAutoOffset', {
            Text     = 'Enable Auto Offset X',
            Default  = Config.FlameLock.AutoOffset.Enabled,
            Callback = function(v) Config.FlameLock.AutoOffset.Enabled = v end
        })
        FLAutoOffBox:AddDivider()
        FLAutoOffBox:AddSlider('FLCloseX', {
            Text     = 'Close X Offset',
            Default  = Config.FlameLock.AutoOffset.CloseX,
            Min      = -20, Max = 20, Rounding = 2,
            Callback = function(v) Config.FlameLock.AutoOffset.CloseX = v end
        })
        FLAutoOffBox:AddSlider('FLMedX', {
            Text     = 'Medium X Offset',
            Default  = Config.FlameLock.AutoOffset.MediumX,
            Min      = -20, Max = 20, Rounding = 2,
            Callback = function(v) Config.FlameLock.AutoOffset.MediumX = v end
        })
        FLAutoOffBox:AddSlider('FLFarX', {
            Text     = 'Far X Offset',
            Default  = Config.FlameLock.AutoOffset.FarX,
            Min      = -20, Max = 20, Rounding = 2,
            Callback = function(v) Config.FlameLock.AutoOffset.FarX = v end
        })
        FLAutoOffBox:AddDivider()
        UI.OffsetSliderLive = FLAutoOffBox:AddLabel('Active: X: —  [—]')
    end

    ----------------------------------------------------------------
    -- RIGHT: Smoothness & Hysteresis
    ----------------------------------------------------------------
    local FLOffsetBox = MainTab:AddRightGroupbox('Smoothness & Hysteresis')
    do
        FLOffsetBox:AddSlider('FLSmoothness', {
            Text     = 'Smoothness',
            Default  = Config.FlameLock.Smoothness,
            Min      = 0.01, Max = 1.0, Rounding = 2,
            Callback = function(v) Config.FlameLock.Smoothness = v end
        })
        FLOffsetBox:AddSlider('FLSwitchHyst', {
            Text     = 'Switch Hysteresis (px)',
            Default  = Config.FlameLock.SwitchHysteresis,
            Min      = 0, Max = 200, Rounding = 0,
            Callback = function(v) Config.FlameLock.SwitchHysteresis = v end
        })
    end

    ----------------------------------------------------------------
    -- RIGHT: FOV Settings
    ----------------------------------------------------------------
    local FOVBox = MainTab:AddRightGroupbox('FOV Settings')
    do
        FOVBox:AddToggle('FLFOVVisible', {
            Text     = 'Show FOV Circle',
            Default  = Config.FlameLock.FOV.Visible,
            Callback = function(v) Config.FlameLock.FOV.Visible = v end
        })
        FOVBox:AddToggle('FLFOVDot', {
            Text     = 'Show Lock Dot (center)',
            Default  = Config.FlameLock.FOV.ShowDot,
            Callback = function(v) Config.FlameLock.FOV.ShowDot = v end
        })
        FOVBox:AddSlider('FLFOVRadius', {
            Text     = 'FOV Radius',
            Default  = Config.FlameLock.FOV.Radius,
            Min      = 50,  Max = 600, Rounding = 0,
            Callback = function(v) Config.FlameLock.FOV.Radius = v end
        })
        FOVBox:AddSlider('FLFOVThickness', {
            Text     = 'FOV Thickness',
            Default  = Config.FlameLock.FOV.Thickness,
            Min      = 0.5, Max = 4.0, Rounding = 1,
            Callback = function(v) Config.FlameLock.FOV.Thickness = v end
        })
        FOVBox:AddSlider('FLDotRadius', {
            Text     = 'Dot Radius',
            Default  = Config.FlameLock.FOV.DotRadius,
            Min      = 1, Max = 10, Rounding = 0,
            Callback = function(v) Config.FlameLock.FOV.DotRadius = v end
        })
    end

    ----------------------------------------------------------------
    -- RIGHT: Vertical Lock
    ----------------------------------------------------------------
    local FLVertBox = MainTab:AddRightGroupbox('Vertical Lock (Anti Flame Up/Down)')
    do
        FLVertBox:AddToggle('FLVerticalLock', {
            Text     = 'Enable Vertical Lock',
            Default  = Config.FlameLock.VerticalLock,
            Callback = function(v) Config.FlameLock.VerticalLock = v end
        })
        FLVertBox:AddSlider('FLVerticalThreshold', {
            Text     = 'Height Threshold (studs)',
            Default  = Config.FlameLock.VerticalLockThreshold,
            Min      = 1, Max = 50, Rounding = 0,
            Callback = function(v) Config.FlameLock.VerticalLockThreshold = v end
        })
        FLVertBox:AddDivider()
        FLVertBox:AddLabel('|dY| <= threshold → flame stays flat')
        FLVertBox:AddLabel('|dY| >  threshold → normal vertical aim')
    end

    ----------------------------------------------------------------
    -- RIGHT: Reset
    ----------------------------------------------------------------
    local ResetBox = MainTab:AddRightGroupbox('Reset Character')
    do
        ResetBox:AddToggle('ResetEnabled', {
            Text     = 'Enable Reset Keybind',
            Default  = Config.Reset.Enabled,
            Callback = function(v) Config.Reset.Enabled = v end
        })
        ResetBox:AddButton('Reset Now', function() doResetCharacter() end)
        ResetBox:AddLabel("Reset Keybind"):AddKeyPicker("ResetKeybind", {
            Default         = "Q",
            SyncToggleState = false,
            Mode            = "Toggle",
            Text            = "Reset Key"
        })
        Options.ResetKeybind:OnClick(function()
            if Config.Reset.Enabled then doResetCharacter() end
        end)
    end

    ----------------------------------------------------------------
    -- RIGHT: UI Settings
    ----------------------------------------------------------------
    local UIBox = MainTab:AddRightGroupbox('UI Settings')
    do
        UIBox:AddButton('Unload Script', function()
            scriptActive = false
            FOVCircle:Remove()
            FOVOuter:Remove()
            FOVDot:Remove()
            for _, conn in ipairs(Connections) do
                if conn then conn:Disconnect() end
            end
            Connections = {}
            Library:Unload()
        end)
        UIBox:AddLabel("Menu Toggle"):AddKeyPicker("MenuToggleKey", {
            Default = "RightShift",
            Mode    = "Toggle",
            Text    = "Toggle Menu"
        })
        Library.ToggleKeybind = Options.MenuToggleKey
    end

    ----------------------------------------------------------------
    -- Live Info — обновление каждый кадр
    ----------------------------------------------------------------
    AddConnection(Services.RunService.RenderStepped:Connect(function()
        if not scriptActive then return end

        local ping   = getPing()
        local target = Config.FlameLock.Target
        local dist3D = nil

        if UI.Ping then UI.Ping:SetText(string.format('Ping: %d ms', ping)) end

        if UI.Target then
            UI.Target:SetText(target and target.Character
                and string.format('Target: %s', target.Name)
                or 'Target: None')
        end

        if UI.FlameHPStatus then
            UI.FlameHPStatus:SetText(Config.FlameHP.Enabled
                and string.format('Flame HP: ON  (-%d dmg / %.0fms)', Config.FlameHP.DamagePerHit, Config.FlameHP.Cooldown * 1000)
                or  'Flame HP: OFF')
        end

        if target and target.Character then
            local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            local tRoot  = target.Character:FindFirstChild("HumanoidRootPart")
            if myRoot and tRoot then
                dist3D = (myRoot.Position - tRoot.Position).Magnitude
                local speed = getTargetSpeed(tRoot)

                if UI.Dist then
                    UI.Dist:SetText(string.format('3D Distance: %.1f studs', dist3D))
                end

                if UI.DeltaY then
                    local dY     = tRoot.Position.Y - myRoot.Position.Y
                    local locked = Config.FlameLock.VerticalLock
                        and math.abs(dY) <= Config.FlameLock.VerticalLockThreshold
                    UI.DeltaY:SetText(string.format('Height dY: %.1f  (VLock: %s)',
                        dY, locked and "FLAT" or "OFF"))
                end

                if UI.OffsetLive then
                    if Config.FlameLock.AutoOffset.Enabled then
                        local zone, baseOx = getOffsetZone(dist3D)
                        local finalOx = applySpeedOffsetRule(baseOx, speed)
                        UI.OffsetLive:SetText(string.format('Offset X: %.2f  [%s | %.1f s/s]', finalOx, zone, speed))
                    else
                        UI.OffsetLive:SetText('Offset X: OFF (Auto disabled)')
                    end
                end

                if UI.OffsetSliderLive then
                    if Config.FlameLock.AutoOffset.Enabled then
                        local zone, baseOx = getOffsetZone(dist3D)
                        local finalOx = applySpeedOffsetRule(baseOx, speed)
                        UI.OffsetSliderLive:SetText(string.format('Active: X: %.2f  [%s]', finalOx, zone))
                    else
                        UI.OffsetSliderLive:SetText('Active: X: —  (disabled)')
                    end
                end

                if UI.Pred then
                    if Config.FlameLock.Prediction then
                        local total, base, bonus, zone = calcSmartPred(dist3D)
                        UI.Pred:SetText(string.format(
                            'Pred: %d ms  [%s]  b%d+d%d',
                            total, zone, base, bonus
                        ))
                    else
                        UI.Pred:SetText('Pred: OFF')
                    end
                end

            else
                if UI.Dist           then UI.Dist:SetText('3D Distance: —') end
                if UI.DeltaY         then UI.DeltaY:SetText('Height dY: —  (VLock: —)') end
                if UI.OffsetLive     then UI.OffsetLive:SetText('Offset X: —  [—]') end
                if UI.OffsetSliderLive then UI.OffsetSliderLive:SetText('Active: X: —  [—]') end
                if UI.Pred           then UI.Pred:SetText('Pred: — ms  [—]') end
            end
        else
            if UI.Dist           then UI.Dist:SetText('3D Distance: —') end
            if UI.DeltaY         then UI.DeltaY:SetText('Height dY: —  (VLock: —)') end
            if UI.OffsetLive     then UI.OffsetLive:SetText('Offset X: —  [—]') end
            if UI.OffsetSliderLive then UI.OffsetSliderLive:SetText('Active: X: —  [—]') end
            if UI.Pred then
                local base = math.clamp(getPing() + 10, 80, 200)
                UI.Pred:SetText(string.format('Pred: %d ms  [no target]', base))
            end
        end
    end))
end
