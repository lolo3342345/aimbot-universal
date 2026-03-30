--============================================================
--  SERVICES
--============================================================
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera           = workspace.CurrentCamera
local LocalPlayer      = Players.LocalPlayer
local Mouse            = LocalPlayer:GetMouse()

--============================================================
--  CONFIGURAÇÃO
--============================================================
local Config = {
    Aimbot = {
        Enabled      = true,
        Smoothness   = 2,          -- 1 = muito rápido, 5 = suave
        BoneName     = "Head",
        TeamCheck    = true,
        MaxDistance   = 800,
        FOVRadius    = 200,
        ShowFOV      = true,
        FOVColor     = Color3.fromRGB(255, 50, 50),

        -- ═══ ANTI-TREMOR ═══
        DeadZone       = 8,        -- NÃO MEXE se tiver a menos de X pixels (aumentado para parar o tremor)
        MaxMovePerFrame= 80,       -- Máximo de pixels por frame (reduzido para mais controle)

        -- WALL
        WallCheck      = true,
        StickyTarget   = true,
        StickyFOVMulti = 2.5,

        -- TOLERÂNCIA MULTI-TARGET (anti-tremor com múltiplos inimigos)
        -- Quantos frames inválidos antes de soltar o alvo
        -- 8 frames = ~0.13s @ 60fps (tempo pra ignorar oclusão momentânea)
        InvalidFramesTolerance = 8,

        -- PREDIÇÃO (desligado = menos tremor)
        Prediction      = false,
        PredictionScale = 0.3,
    },
    ESP = {
        Enabled      = true,
        Boxes        = true,
        Names        = true,
        Health       = true,
        Distance     = true,
        Tracers      = true,
        TeamCheck    = true,
        MaxDistance   = 1500,
        BoxColor     = Color3.fromRGB(255, 50, 50),
        AllyColor    = Color3.fromRGB(50, 255, 50),
        NameColor    = Color3.fromRGB(255, 255, 255),
        TracerColor  = Color3.fromRGB(255, 255, 0),
        TracerOrigin = "Bottom",
    },
    TriggerBot = {
        Enabled        = true,
        Delay          = 0.03,
        TeamCheck      = true,
        AutoWithAimbot = true,
        AutoFireZone   = 10,       -- Atira quando crosshair < X pixels do alvo
    },
}

--============================================================
--  ESTADO
--============================================================
local State = {
    CurrentTarget  = nil,
    LastTargetPos  = nil,
    IsLocked       = false,
    LastShotTime   = 0,
    InvalidFrames  = 0,   -- Contador de frames que o alvo ficou inválido
}

local Connections = {}
local ESPCache    = {}

--============================================================
--  UTILIDADES
--============================================================
local function IsAlive(player)
    local char = player and player.Character
    if not char then return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0
end

local function GetBone(player, boneName)
    local char = player and player.Character
    if not char then return nil end
    return char:FindFirstChild(boneName)
end

local function GetRootPart(player)
    local char = player and player.Character
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart")
        or char:FindFirstChild("Torso")
end

local function IsTeammate(player)
    if not LocalPlayer.Team then return false end
    if not player.Team then return false end
    return LocalPlayer.Team == player.Team
end

local function WorldToScreen(pos)
    local vec, onScreen = Camera:WorldToViewportPoint(pos)
    return Vector2.new(vec.X, vec.Y), onScreen, vec.Z
end

local function GetScreenCenter()
    local vp = Camera.ViewportSize
    return Vector2.new(vp.X / 2, vp.Y / 2)
end

local function IsVisible(fromPos, toPos, targetChar)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Blacklist

    -- ════════════════════════════════════════════════════════
    -- CORREÇÃO MULTI-TARGET:
    -- Antes: só filtrava o jogador local e o alvo
    --   → Se um inimigo passasse na frente do alvo, o raycast
    --     batia no corpo desse inimigo e devolvia "parede"
    --     → alvo dropado → tremor ao trocar de target
    --
    -- Agora: filtra TODOS os personagens de inimigos
    --   → Raycasts só colidem com geometria do mapa
    --   → Inimigos na frente não causam falso-positivo de parede
    -- ════════════════════════════════════════════════════════
    local filter = { LocalPlayer.Character }
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            table.insert(filter, p.Character)
        end
    end
    params.FilterDescendantsInstances = filter

    local dir = (toPos - fromPos)
    local result = workspace:Raycast(fromPos, dir.Unit * dir.Magnitude, params)
    return result == nil
end

--============================================================
--  DRAWINGS
--============================================================
local FOVCircle = Drawing.new("Circle")
FOVCircle.Thickness    = 1
FOVCircle.NumSides     = 80
FOVCircle.Filled       = false
FOVCircle.Visible      = false
FOVCircle.Transparency = 0.6

local TargetDot = Drawing.new("Circle")
TargetDot.Thickness    = 2
TargetDot.NumSides     = 20
TargetDot.Radius       = 4
TargetDot.Filled       = true
TargetDot.Visible      = false
TargetDot.Color        = Color3.fromRGB(0, 255, 0)

local DeadZoneCircle = Drawing.new("Circle")
DeadZoneCircle.Thickness    = 1
DeadZoneCircle.NumSides     = 30
DeadZoneCircle.Filled       = false
DeadZoneCircle.Visible      = false
DeadZoneCircle.Color        = Color3.fromRGB(0, 255, 0)
DeadZoneCircle.Transparency = 0.3

--============================================================
--  AIMBOT — SEM TREMOR
--============================================================
local Aimbot = {}

function Aimbot.IsTargetValid(player)
    if not player then return false end
    if not player.Parent then return false end
    if not IsAlive(player) then return false end
    if Config.Aimbot.TeamCheck and IsTeammate(player) then return false end

    local bone = GetBone(player, Config.Aimbot.BoneName)
    if not bone then return false end

    local localRoot = GetRootPart(LocalPlayer)
    if not localRoot then return false end

    local dist3D = (localRoot.Position - bone.Position).Magnitude
    if dist3D > Config.Aimbot.MaxDistance then return false end

    -- WALL CHECK
    if Config.Aimbot.WallCheck then
        if not IsVisible(Camera.CFrame.Position, bone.Position, player.Character) then
            return false
        end
    end

    -- STICKY: FOV maior pra manter alvo
    if Config.Aimbot.StickyTarget and State.CurrentTarget == player then
        local sp, onScr = WorldToScreen(bone.Position)
        if not onScr then return false end
        local d2d = (GetScreenCenter() - sp).Magnitude
        return d2d <= Config.Aimbot.FOVRadius * Config.Aimbot.StickyFOVMulti
    end

    return true
end

function Aimbot.FindTarget()
    local center   = GetScreenCenter()
    local bestDist = Config.Aimbot.FOVRadius
    local bestPlr  = nil

    local localRoot = GetRootPart(LocalPlayer)
    if not localRoot then return nil end
    local eyePos = Camera.CFrame.Position

    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        if not IsAlive(player) then continue end
        if Config.Aimbot.TeamCheck and IsTeammate(player) then continue end

        local bone = GetBone(player, Config.Aimbot.BoneName)
        if not bone then continue end

        local dist3D = (localRoot.Position - bone.Position).Magnitude
        if dist3D > Config.Aimbot.MaxDistance then continue end

        if Config.Aimbot.WallCheck then
            if not IsVisible(eyePos, bone.Position, player.Character) then
                continue
            end
        end

        local sp, onScr = WorldToScreen(bone.Position)
        if not onScr then continue end

        local d2d = (center - sp).Magnitude
        if d2d < bestDist then
            bestDist = d2d
            bestPlr  = player
        end
    end

    return bestPlr
end

function Aimbot.GetAimPosition(player)
    local bone = GetBone(player, Config.Aimbot.BoneName)
    if not bone then return nil end

    local pos = bone.Position

    -- Predição simples (se ligada)
    if Config.Aimbot.Prediction and State.LastTargetPos and State.CurrentTarget == player then
        local vel = pos - State.LastTargetPos
        pos = pos + vel * Config.Aimbot.PredictionScale
    end

    State.LastTargetPos = bone.Position
    return pos
end

--[[
    ╔═══════════════════════════════════════════════════════════╗
    ║  FUNÇÃO PRINCIPAL - MOVIMENTO SEM TREMOR (CORRIGIDO)      ║
    ║                                                           ║
    ║  BUGS CORRIGIDOS:                                         ║
    ║  1. Removido MinMove — era a causa principal do tremor    ║
    ║     (forçava mover 1px mesmo perto do alvo → oscilação)   ║
    ║  2. Removido multiplicador 1.5 — causava overshoot        ║
    ║  3. Speed mínimo reduzido (0.05 → 0.0) — para suave       ║
    ║  4. Arredondamento correto: math.floor(x + 0.5)           ║
    ║     (arredonda normalmente, não força mínimo)             ║
    ╚═══════════════════════════════════════════════════════════╝
]]
function Aimbot.CalculateMove(deltaX, deltaY, distance)
    -- ═══════════════════════════════════
    -- PASSO 1: DEAD ZONE — Se já tá perto, NÃO MEXE
    -- CRÍTICO: deve ser a primeira verificação, sem exceções
    -- ═══════════════════════════════════
    if distance <= Config.Aimbot.DeadZone then
        return 0, 0
    end

    -- ═══════════════════════════════════
    -- PASSO 2: Calcular velocidade suave
    -- Quanto mais longe, mais rápido move
    -- Quanto mais perto, mais devagar (evita overshoot)
    -- ═══════════════════════════════════
    local smooth = math.max(Config.Aimbot.Smoothness, 1)

    -- Normaliza a distância (0 = no alvo, 1 = na borda do FOV)
    local normalizedDist = math.clamp(distance / Config.Aimbot.FOVRadius, 0, 1)

    -- Speed proporcional à distância dividido pelo smooth
    -- SEM multiplicador 1.5 → não acelera perto do alvo → sem overshoot
    local speed = normalizedDist / smooth

    -- Clamp: máximo 0.85 pra nunca ser brusco
    -- SEM mínimo: se o speed calculado for 0, fica 0 → sem tremor!
    speed = math.clamp(speed, 0, 0.85)

    -- ═══════════════════════════════════
    -- PASSO 3: Calcular movimento bruto
    -- ═══════════════════════════════════
    local moveX = deltaX * speed
    local moveY = deltaY * speed

    -- ═══════════════════════════════════
    -- PASSO 4: ANTI-OVERSHOOT
    -- Nunca mover MAIS que a distância real até o alvo
    -- ═══════════════════════════════════
    if math.abs(moveX) > math.abs(deltaX) then moveX = deltaX end
    if math.abs(moveY) > math.abs(deltaY) then moveY = deltaY end

    -- ═══════════════════════════════════
    -- PASSO 5: CLAMP máximo por frame
    -- ═══════════════════════════════════
    local maxMove = Config.Aimbot.MaxMovePerFrame
    moveX = math.clamp(moveX, -maxMove, maxMove)
    moveY = math.clamp(moveY, -maxMove, maxMove)

    -- ═══════════════════════════════════
    -- PASSO 6: ARREDONDAR corretamente
    --
    -- ANTES (ERRADO): math.max(math.floor(moveX), MinMove)
    --   → Forçava mínimo de 1px mesmo quando devia ser 0
    --   → Causava oscilação: move 1px → passa → volta 1px → passa...
    --
    -- AGORA (CORRETO): math.floor(moveX + 0.5)
    --   → Arredondamento normal: 0.4px → 0, 0.6px → 1
    --   → Se o movimento calculado é pequeno, arredonda pra 0 e PARA
    -- ═══════════════════════════════════
    moveX = math.floor(moveX + 0.5)
    moveY = math.floor(moveY + 0.5)

    return moveX, moveY
end

function Aimbot.Run()
    if not Config.Aimbot.Enabled then
        State.CurrentTarget = nil
        State.IsLocked = false
        TargetDot.Visible = false
        DeadZoneCircle.Visible = false
        return
    end

    if not IsAlive(LocalPlayer) then
        State.CurrentTarget = nil
        State.IsLocked = false
        TargetDot.Visible = false
        DeadZoneCircle.Visible = false
        return
    end

    -- ════════════════════════════════════════════════════════
    -- VERIFICAR ALVO COM TOLERÂNCIA DE FRAMES
    --
    -- ANTES: Soltava o alvo imediatamente na primeira falha
    --   → Com 2+ inimigos no FOV: 1 frame de oclusão = troca de alvo = tremor
    --
    -- AGORA: Conta frames inválidos. Só solta depois de N frames consecutivos
    --   → Oclusões momentâneas (1-2 frames) são ignoradas
    --   → Alvo não troca com flutuações de raycast
    -- ════════════════════════════════════════════════════════
    if State.CurrentTarget then
        if not Aimbot.IsTargetValid(State.CurrentTarget) then
            State.InvalidFrames = State.InvalidFrames + 1

            if State.InvalidFrames >= Config.Aimbot.InvalidFramesTolerance then
                -- Só solta o target depois de N frames consecutivos inválidos
                State.CurrentTarget = nil
                State.IsLocked = false
                State.LastTargetPos = nil
                State.InvalidFrames = 0
                TargetDot.Visible = false
                DeadZoneCircle.Visible = false
            end
            -- Enquanto dentro da tolerância: mantém o alvo e continua mirando
        else
            -- Alvo válido: reseta o contador
            State.InvalidFrames = 0
        end
    end

    -- Buscar novo alvo (só se não tiver nenhum)
    if not State.CurrentTarget then
        local newTarget = Aimbot.FindTarget()
        if newTarget then
            State.CurrentTarget = newTarget
            State.IsLocked = true
            State.LastTargetPos = nil
            State.InvalidFrames = 0
        end
    end

    -- Mirar no alvo
    if not State.CurrentTarget then
        TargetDot.Visible = false
        DeadZoneCircle.Visible = false
        return
    end

    local aimPos = Aimbot.GetAimPosition(State.CurrentTarget)
    if not aimPos then
        State.CurrentTarget = nil
        State.IsLocked = false
        return
    end

    -- WALL CHECK final
    if Config.Aimbot.WallCheck then
        if not IsVisible(Camera.CFrame.Position, aimPos, State.CurrentTarget.Character) then
            State.CurrentTarget = nil
            State.IsLocked = false
            State.LastTargetPos = nil
            TargetDot.Visible = false
            DeadZoneCircle.Visible = false
            return
        end
    end

    local screenPos, onScreen = WorldToScreen(aimPos)
    if not onScreen then
        State.CurrentTarget = nil
        State.IsLocked = false
        return
    end

    local center = GetScreenCenter()
    local deltaX = screenPos.X - center.X
    local deltaY = screenPos.Y - center.Y
    local dist2D = math.sqrt(deltaX * deltaX + deltaY * deltaY)

    -- ═══════════════════════════════
    -- CALCULAR MOVIMENTO (SEM TREMOR)
    -- ═══════════════════════════════
    local moveX, moveY = Aimbot.CalculateMove(deltaX, deltaY, dist2D)

    -- Só move se tem algo pra mover
    if moveX ~= 0 or moveY ~= 0 then
        mousemoverel(moveX, moveY)
    end

    -- Visual
    TargetDot.Visible  = true
    TargetDot.Position = screenPos

    -- Mostra dead zone no alvo
    DeadZoneCircle.Visible  = true
    DeadZoneCircle.Position = screenPos
    DeadZoneCircle.Radius   = Config.Aimbot.DeadZone

    -- Muda cor: verde = dentro da dead zone (parado), vermelho = movendo
    if dist2D <= Config.Aimbot.DeadZone then
        TargetDot.Color      = Color3.fromRGB(0, 255, 0)
        DeadZoneCircle.Color = Color3.fromRGB(0, 255, 0)
    else
        TargetDot.Color      = Color3.fromRGB(255, 255, 0)
        DeadZoneCircle.Color = Color3.fromRGB(255, 100, 0)
    end

    -- AUTO TRIGGER
    if Config.TriggerBot.Enabled and Config.TriggerBot.AutoWithAimbot then
        if dist2D <= Config.TriggerBot.AutoFireZone then
            local now = tick()
            if now - State.LastShotTime >= Config.TriggerBot.Delay then
                State.LastShotTime = now
                task.defer(function()
                    mouse1click()
                end)
            end
        end
    end
end

--============================================================
--  TRIGGERBOT STANDALONE
--============================================================
local function RunTriggerBot()
    if not Config.TriggerBot.Enabled then return end
    if Config.TriggerBot.AutoWithAimbot then return end
    if not IsAlive(LocalPlayer) then return end

    local target = Mouse.Target
    if not target then return end

    local model = target:FindFirstAncestorOfClass("Model")
    if not model then return end

    local player = Players:GetPlayerFromCharacter(model)
    if not player or player == LocalPlayer then return end
    if Config.TriggerBot.TeamCheck and IsTeammate(player) then return end
    if not IsAlive(player) then return end

    local now = tick()
    if now - State.LastShotTime >= Config.TriggerBot.Delay then
        State.LastShotTime = now
        mouse1click()
    end
end

--============================================================
--  ESP
--============================================================
local function CreateESP(player)
    if ESPCache[player] then return end
    ESPCache[player] = {
        BoxOut   = Drawing.new("Square"),
        Box      = Drawing.new("Square"),
        Name     = Drawing.new("Text"),
        Dist     = Drawing.new("Text"),
        HpBg     = Drawing.new("Square"),
        HpBar    = Drawing.new("Square"),
        Tracer   = Drawing.new("Line"),
    }

    local o = ESPCache[player]
    o.BoxOut.Thickness = 3
    o.BoxOut.Color = Color3.new(0, 0, 0)
    o.BoxOut.Filled = false
    o.BoxOut.Visible = false

    o.Box.Thickness = 1
    o.Box.Filled = false
    o.Box.Visible = false

    o.Name.Size = 14
    o.Name.Center = true
    o.Name.Outline = true
    o.Name.Font = Drawing.Fonts.Plex
    o.Name.Visible = false

    o.Dist.Size = 13
    o.Dist.Center = true
    o.Dist.Outline = true
    o.Dist.Font = Drawing.Fonts.Plex
    o.Dist.Visible = false

    o.HpBg.Filled = true
    o.HpBg.Color = Color3.new(0, 0, 0)
    o.HpBg.Transparency = 0.5
    o.HpBg.Visible = false

    o.HpBar.Filled = true
    o.HpBar.Visible = false

    o.Tracer.Thickness = 1
    o.Tracer.Visible = false
end

local function RemoveESP(player)
    local o = ESPCache[player]
    if not o then return end
    for _, d in pairs(o) do pcall(function() d:Remove() end) end
    ESPCache[player] = nil
end

local function HideESP(player)
    local o = ESPCache[player]
    if not o then return end
    for _, d in pairs(o) do d.Visible = false end
end

local function UpdateESP(player)
    if not Config.ESP.Enabled then HideESP(player) return end
    if player == LocalPlayer then HideESP(player) return end
    if not IsAlive(player) or not IsAlive(LocalPlayer) then HideESP(player) return end

    local isAlly = IsTeammate(player)
    if Config.ESP.TeamCheck and isAlly then HideESP(player) return end

    local root  = GetRootPart(player)
    local lRoot = GetRootPart(LocalPlayer)
    if not root or not lRoot then HideESP(player) return end

    local dist = (lRoot.Position - root.Position).Magnitude
    if dist > Config.ESP.MaxDistance then HideESP(player) return end

    local hum = player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then HideESP(player) return end

    local topS, topOn    = WorldToScreen(root.Position + Vector3.new(0, 3.2, 0))
    local botS, botOn    = WorldToScreen(root.Position - Vector3.new(0, 3.2, 0))
    if not topOn and not botOn then HideESP(player) return end

    local o = ESPCache[player]
    if not o then return end

    local bH = math.abs(botS.Y - topS.Y)
    local bW = bH * 0.55
    local bX = topS.X - bW / 2
    local bY = topS.Y

    local isTarget = State.CurrentTarget == player
    local color
    if isTarget then
        color = Color3.fromRGB(255, 0, 255)
    else
        color = isAlly and Config.ESP.AllyColor or Config.ESP.BoxColor
    end

    -- Box
    if Config.ESP.Boxes then
        o.BoxOut.Visible  = true
        o.BoxOut.Position = Vector2.new(bX, bY)
        o.BoxOut.Size     = Vector2.new(bW, bH)
        o.Box.Visible  = true
        o.Box.Position = Vector2.new(bX, bY)
        o.Box.Size     = Vector2.new(bW, bH)
        o.Box.Color    = color
    else
        o.Box.Visible    = false
        o.BoxOut.Visible = false
    end

    -- Name
    if Config.ESP.Names then
        local name = player.DisplayName or player.Name
        if isTarget then name = "⦿ " .. name .. " [LOCKED]" end
        o.Name.Visible  = true
        o.Name.Text     = name
        o.Name.Position = Vector2.new(topS.X, bY - 16)
        o.Name.Color    = isTarget and Color3.fromRGB(255, 100, 255) or Config.ESP.NameColor
    else
        o.Name.Visible = false
    end

    -- Health
    if Config.ESP.Health then
        local ratio = math.clamp(hum.Health / hum.MaxHealth, 0, 1)
        local barW = 3
        local barX = bX - barW - 3
        o.HpBg.Visible  = true
        o.HpBg.Position = Vector2.new(barX, bY)
        o.HpBg.Size     = Vector2.new(barW, bH)
        local barH = bH * ratio
        o.HpBar.Visible  = true
        o.HpBar.Position = Vector2.new(barX, bY + (bH - barH))
        o.HpBar.Size     = Vector2.new(barW, barH)
        o.HpBar.Color    = Color3.new(1 - ratio, ratio, 0)
    else
        o.HpBg.Visible  = false
        o.HpBar.Visible = false
    end

    -- Distance
    if Config.ESP.Distance then
        o.Dist.Visible  = true
        o.Dist.Text     = string.format("[%d]", math.floor(dist))
        o.Dist.Position = Vector2.new(botS.X, bY + bH + 2)
        o.Dist.Color    = Config.ESP.NameColor
    else
        o.Dist.Visible = false
    end

    -- Tracer
    if Config.ESP.Tracers then
        local vp = Camera.ViewportSize
        local origin
        if Config.ESP.TracerOrigin == "Bottom" then
            origin = Vector2.new(vp.X / 2, vp.Y)
        elseif Config.ESP.TracerOrigin == "Center" then
            origin = Vector2.new(vp.X / 2, vp.Y / 2)
        else
            origin = Vector2.new(Mouse.X, Mouse.Y)
        end
        o.Tracer.Visible = true
        o.Tracer.From    = origin
        o.Tracer.To      = botS
        o.Tracer.Color   = isTarget and Color3.fromRGB(255, 0, 255) or Config.ESP.TracerColor
    else
        o.Tracer.Visible = false
    end
end

--============================================================
--  PLAYER EVENTS
--============================================================
for _, p in ipairs(Players:GetPlayers()) do
    if p ~= LocalPlayer then CreateESP(p) end
end

table.insert(Connections, Players.PlayerAdded:Connect(function(p) CreateESP(p) end))
table.insert(Connections, Players.PlayerRemoving:Connect(function(p)
    RemoveESP(p)
    if State.CurrentTarget == p then
        State.CurrentTarget = nil
        State.IsLocked = false
    end
end))

--============================================================
--  RENDER LOOP
--============================================================
table.insert(Connections, RunService.RenderStepped:Connect(function()
    Camera = workspace.CurrentCamera

    -- FOV
    FOVCircle.Visible  = Config.Aimbot.ShowFOV and Config.Aimbot.Enabled
    FOVCircle.Position = GetScreenCenter()
    FOVCircle.Radius   = Config.Aimbot.FOVRadius
    FOVCircle.Color    = Config.Aimbot.FOVColor

    -- Aimbot
    Aimbot.Run()

    -- TriggerBot
    RunTriggerBot()

    -- ESP
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then
            if not ESPCache[p] then CreateESP(p) end
            UpdateESP(p)
        end
    end
end))

--============================================================
--  UI
--============================================================
if game.CoreGui:FindFirstChild("CheatUI") then
    game.CoreGui.CheatUI:Destroy()
end

local Gui = Instance.new("ScreenGui")
Gui.Name = "CheatUI"
Gui.Parent = game.CoreGui
Gui.ResetOnSpawn = false

local CL = {
    Bg  = Color3.fromRGB(18, 18, 18),
    Hd  = Color3.fromRGB(30, 30, 30),
    Ac  = Color3.fromRGB(255, 50, 80),
    Sc  = Color3.fromRGB(25, 25, 25),
    Tx  = Color3.fromRGB(220, 220, 220),
    Sb  = Color3.fromRGB(130, 130, 130),
    On  = Color3.fromRGB(255, 50, 80),
    Off = Color3.fromRGB(50, 50, 50),
    Sl  = Color3.fromRGB(40, 40, 40),
}

local MF = Instance.new("Frame", Gui)
MF.BackgroundColor3 = CL.Bg
MF.BorderSizePixel  = 0
MF.Position         = UDim2.new(0.015, 0, 0.08, 0)
MF.Size             = UDim2.new(0, 310, 0, 36)
MF.Active           = true
MF.Draggable        = true
MF.ClipsDescendants = true
Instance.new("UICorner", MF).CornerRadius = UDim.new(0, 8)
Instance.new("UIStroke", MF).Color = CL.Ac

local Hdr = Instance.new("Frame", MF)
Hdr.BackgroundColor3 = CL.Hd
Hdr.Size = UDim2.new(1, 0, 0, 36)
Hdr.BorderSizePixel  = 0
Instance.new("UICorner", Hdr).CornerRadius = UDim.new(0, 8)

local Ttl = Instance.new("TextLabel", Hdr)
Ttl.BackgroundTransparency = 1
Ttl.Position = UDim2.new(0, 12, 0, 0)
Ttl.Size     = UDim2.new(1, -12, 1, 0)
Ttl.Font     = Enum.Font.GothamBold
Ttl.Text     = "💀 SMOOTH AIM — NO JITTER"
Ttl.TextColor3 = CL.Tx
Ttl.TextSize   = 13
Ttl.TextXAlignment = Enum.TextXAlignment.Left

local Scr = Instance.new("ScrollingFrame", MF)
Scr.BackgroundTransparency  = 1
Scr.Position                = UDim2.new(0, 0, 0, 40)
Scr.Size                    = UDim2.new(1, 0, 1, -40)
Scr.CanvasSize              = UDim2.new(0, 0, 0, 0)
Scr.ScrollBarThickness      = 3
Scr.ScrollBarImageColor3    = CL.Ac
Scr.BorderSizePixel         = 0

local Lyt = Instance.new("UIListLayout", Scr)
Lyt.Padding   = UDim.new(0, 3)
Lyt.SortOrder = Enum.SortOrder.LayoutOrder

Instance.new("UIPadding", Scr).PaddingLeft  = UDim.new(0, 6)
Instance.new("UIPadding", Scr).PaddingRight = UDim.new(0, 6)

Lyt:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    Scr.CanvasSize = UDim2.new(0, 0, 0, Lyt.AbsoluteContentSize.Y + 12)
    MF.Size = UDim2.new(0, 310, 0, math.min(Lyt.AbsoluteContentSize.Y + 56, 520))
end)

local ord = 0

local function Sec(n)
    ord += 1
    local f = Instance.new("Frame", Scr)
    f.BackgroundColor3 = CL.Sc; f.Size = UDim2.new(1, 0, 0, 22)
    f.BorderSizePixel = 0; f.LayoutOrder = ord
    Instance.new("UICorner", f).CornerRadius = UDim.new(0, 5)
    local l = Instance.new("TextLabel", f)
    l.BackgroundTransparency = 1; l.Size = UDim2.new(1, 0, 1, 0)
    l.Font = Enum.Font.GothamBold; l.Text = "  " .. n
    l.TextColor3 = CL.Ac; l.TextSize = 11
    l.TextXAlignment = Enum.TextXAlignment.Left
end

local function Tog(name, def, cb)
    ord += 1
    local val = def
    local f = Instance.new("Frame", Scr)
    f.BackgroundColor3 = CL.Sc; f.Size = UDim2.new(1, 0, 0, 26)
    f.BorderSizePixel = 0; f.LayoutOrder = ord
    Instance.new("UICorner", f).CornerRadius = UDim.new(0, 5)

    local l = Instance.new("TextLabel", f)
    l.BackgroundTransparency = 1; l.Position = UDim2.new(0, 8, 0, 0)
    l.Size = UDim2.new(1, -48, 1, 0); l.Font = Enum.Font.Gotham
    l.Text = name; l.TextColor3 = CL.Tx; l.TextSize = 11
    l.TextXAlignment = Enum.TextXAlignment.Left

    local t = Instance.new("Frame", f)
    t.Position = UDim2.new(1, -40, 0.5, -7); t.Size = UDim2.new(0, 32, 0, 14)
    t.BackgroundColor3 = val and CL.On or CL.Off; t.BorderSizePixel = 0
    Instance.new("UICorner", t).CornerRadius = UDim.new(1, 0)

    local d = Instance.new("Frame", t)
    d.Size = UDim2.new(0, 10, 0, 10)
    d.Position = val and UDim2.new(1, -12, 0, 2) or UDim2.new(0, 2, 0, 2)
    d.BackgroundColor3 = Color3.new(1, 1, 1); d.BorderSizePixel = 0
    Instance.new("UICorner", d).CornerRadius = UDim.new(1, 0)

    local b = Instance.new("TextButton", f)
    b.BackgroundTransparency = 1; b.Size = UDim2.new(1, 0, 1, 0); b.Text = ""
    b.MouseButton1Click:Connect(function()
        val = not val
        t.BackgroundColor3 = val and CL.On or CL.Off
        d.Position = val and UDim2.new(1, -12, 0, 2) or UDim2.new(0, 2, 0, 2)
        if cb then cb(val) end
    end)
end

local function Sld(name, mn, mx, def, step, cb)
    ord += 1
    local val = def
    step = step or 1

    local f = Instance.new("Frame", Scr)
    f.BackgroundColor3 = CL.Sc; f.Size = UDim2.new(1, 0, 0, 38)
    f.BorderSizePixel = 0; f.LayoutOrder = ord
    Instance.new("UICorner", f).CornerRadius = UDim.new(0, 5)

    local l = Instance.new("TextLabel", f)
    l.BackgroundTransparency = 1; l.Position = UDim2.new(0, 8, 0, 0)
    l.Size = UDim2.new(1, -42, 0, 14); l.Font = Enum.Font.Gotham
    l.Text = name; l.TextColor3 = CL.Tx; l.TextSize = 11
    l.TextXAlignment = Enum.TextXAlignment.Left

    local vl = Instance.new("TextLabel", f)
    vl.BackgroundTransparency = 1; vl.Position = UDim2.new(1, -40, 0, 0)
    vl.Size = UDim2.new(0, 32, 0, 14); vl.Font = Enum.Font.GothamBold
    vl.TextColor3 = CL.Ac; vl.TextSize = 11

    local function formatVal(v)
        if step < 1 then return string.format("%.1f", v) end
        return tostring(math.floor(v))
    end
    vl.Text = formatVal(val)

    local bg = Instance.new("Frame", f)
    bg.BackgroundColor3 = CL.Sl; bg.Position = UDim2.new(0, 8, 0, 18)
    bg.Size = UDim2.new(1, -16, 0, 12); bg.BorderSizePixel = 0
    Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)

    local fl = Instance.new("Frame", bg)
    fl.BackgroundColor3 = CL.Ac; fl.BorderSizePixel = 0
    fl.Size = UDim2.new(math.clamp((val - mn) / (mx - mn), 0, 1), 0, 1, 0)
    Instance.new("UICorner", fl).CornerRadius = UDim.new(1, 0)

    local btn = Instance.new("TextButton", bg)
    btn.BackgroundTransparency = 1; btn.Size = UDim2.new(1, 0, 1, 0); btn.Text = ""

    local drag = false
    btn.MouseButton1Down:Connect(function() drag = true end)

    table.insert(Connections, UserInputService.InputEnded:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1 then drag = false end
    end))

    table.insert(Connections, UserInputService.InputChanged:Connect(function(inp)
        if drag and inp.UserInputType == Enum.UserInputType.MouseMovement then
            local rel = math.clamp((inp.Position.X - bg.AbsolutePosition.X) / bg.AbsoluteSize.X, 0, 1)
            val = mn + (mx - mn) * rel
            if step >= 1 then
                val = math.floor(val)
            else
                val = math.floor(val / step) * step
            end
            val = math.clamp(val, mn, mx)
            fl.Size = UDim2.new(rel, 0, 1, 0)
            vl.Text = formatVal(val)
            if cb then cb(val) end
        end
    end))
end

-- ═══════════════ BUILD UI ═══════════════

Sec("⚡ AIMBOT")
Tog("Enabled", Config.Aimbot.Enabled, function(v) Config.Aimbot.Enabled = v end)
Sld("Smoothness", 1, 10, Config.Aimbot.Smoothness, 0.5, function(v) Config.Aimbot.Smoothness = v end)
Sld("FOV Radius", 30, 500, Config.Aimbot.FOVRadius, 5, function(v) Config.Aimbot.FOVRadius = v end)
Sld("Dead Zone", 1, 30, Config.Aimbot.DeadZone, 1, function(v) Config.Aimbot.DeadZone = v end)
Sld("Max Distance", 100, 2000, Config.Aimbot.MaxDistance, 50, function(v) Config.Aimbot.MaxDistance = v end)
Tog("Wall Check", Config.Aimbot.WallCheck, function(v) Config.Aimbot.WallCheck = v end)
Tog("Team Check", Config.Aimbot.TeamCheck, function(v) Config.Aimbot.TeamCheck = v end)
Tog("Sticky Target", Config.Aimbot.StickyTarget, function(v) Config.Aimbot.StickyTarget = v end)
Tog("Prediction", Config.Aimbot.Prediction, function(v) Config.Aimbot.Prediction = v end)
Sld("Predict Scale", 0, 1, Config.Aimbot.PredictionScale, 0.1, function(v) Config.Aimbot.PredictionScale = v end)
Tog("Show FOV", Config.Aimbot.ShowFOV, function(v) Config.Aimbot.ShowFOV = v end)

Sec("👁️ ESP")
Tog("ESP ON", Config.ESP.Enabled, function(v)
    Config.ESP.Enabled = v
    if not v then for _,p in ipairs(Players:GetPlayers()) do HideESP(p) end end
end)
Tog("Boxes", Config.ESP.Boxes, function(v) Config.ESP.Boxes = v end)
Tog("Names", Config.ESP.Names, function(v) Config.ESP.Names = v end)
Tog("Health", Config.ESP.Health, function(v) Config.ESP.Health = v end)
Tog("Distance", Config.ESP.Distance, function(v) Config.ESP.Distance = v end)
Tog("Tracers", Config.ESP.Tracers, function(v) Config.ESP.Tracers = v end)
Sld("ESP Distance", 100, 3000, Config.ESP.MaxDistance, 100, function(v) Config.ESP.MaxDistance = v end)

Sec("🔫 TRIGGERBOT")
Tog("TriggerBot ON", Config.TriggerBot.Enabled, function(v) Config.TriggerBot.Enabled = v end)
Tog("Auto w/ Aimbot", Config.TriggerBot.AutoWithAimbot, function(v) Config.TriggerBot.AutoWithAimbot = v end)
Sld("Fire Zone (px)", 3, 50, Config.TriggerBot.AutoFireZone, 1, function(v) Config.TriggerBot.AutoFireZone = v end)
Sld("Delay (ms)", 0, 200, Config.TriggerBot.Delay * 1000, 5, function(v) Config.TriggerBot.Delay = v / 1000 end)

--============================================================
--  INPUT / CLEANUP
--============================================================
table.insert(Connections, UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        Gui.Enabled = not Gui.Enabled
    end
    if input.KeyCode == Enum.KeyCode.End then
        for _, c in ipairs(Connections) do pcall(function() c:Disconnect() end) end
        for p, _ in pairs(ESPCache) do RemoveESP(p) end
        pcall(function() FOVCircle:Remove() end)
        pcall(function() TargetDot:Remove() end)
        pcall(function() DeadZoneCircle:Remove() end)
        pcall(function() Gui:Destroy() end)
        print("[!] Descarregado.")
    end
end))

print("═══════════════════════════════════")
print("  ✅ SMOOTH AIM carregado!")
print("  ✅ Dead Zone = SEM TREMOR")
print("  ⌨️  Insert = Toggle UI")
print("  ⌨️  End    = Descarregar")
print("═══════════════════════════════════")
