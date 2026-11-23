-- David and Tixan Auto Collect Cash (Steal a Brainrot) | Luau, mobile friendly
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local PathfindingService = game:GetService("PathfindingService")
local Lighting = game:GetService("Lighting")
local LP = Players.LocalPlayer

local CFG = {
    PadText         = "Collect",
    PadSearchRadius = 20,    -- радиус поиска плиты вокруг надписи Collect
    PadMaxThickness = 2,     -- плита плоская, если Size.Y не больше
    PadMinSide      = 3,     -- минимальная сторона плиты
    PadMaxSide      = 14,    -- плиты больше этого размера игнорируем (Collect Zone и т.п.)
    Blacklist       = { "zone", "multi", "multiplier" }, -- такие объекты/надписи игнорируем
    EdgeMargin      = 1.0,   -- отступ от края плиты (встаём ближе к центру)
    ReachDist       = 2.5,   -- если плита не найдена: радиус "дошёл"
    PadTimeout      = 8,     -- макс. секунд на путь до одного пада
    StuckJumpAfter  = 1.0,
    Attempts        = 3,     -- попыток собрать один пад
    Dwell           = 0.8,   -- ожидание подтверждения сбора
    StepOffDist     = 7,     -- отход перед повторной попыткой
    SkipEmptyPads   = true,
    UsePathfinding  = true,
    UseFireTouch    = true,
    FallbackRadius  = 150,   -- если свой плот не найден
    RescanEvery     = 5,
    CycleDelay      = 0.4,
    Debug           = true,
}

local running = false
local home = nil
local okCount, missCount = 0, 0
local statusText = "Off"

---------------------------------------------------------------- helpers
local function getChar()
    local c = LP.Character
    if not c then return end
    local hum = c:FindFirstChildOfClass("Humanoid")
    local root = c:FindFirstChild("HumanoidRootPart")
    if hum and root and hum.Health > 0 then return c, hum, root end
end

local function flatDist(a, b)
    return Vector3.new(a.X - b.X, 0, a.Z - b.Z).Magnitude
end

local function ownerMatches(v)
    if v == nil then return false end
    if typeof(v) == "Instance" then return v == LP end
    if type(v) == "number" then return v == LP.UserId end
    if type(v) == "string" then
        return v == LP.Name or v == LP.DisplayName or v == tostring(LP.UserId)
    end
    return false
end

local ownerKeys = { "Owner", "OwnerId", "OwnerUserId", "UserId", "Player", "OwnerName" }

local function plotBelongsToMe(plot)
    for _, k in ipairs(ownerKeys) do
        if ownerMatches(plot:GetAttribute(k)) then return true end
    end
    for _, d in ipairs(plot:GetDescendants()) do
        if d:IsA("ValueBase") then
            for _, k in ipairs(ownerKeys) do
                if d.Name == k and ownerMatches(d.Value) then return true end
            end
        elseif d:IsA("TextLabel") then
            local t = d.Text
            if t ~= "" and (t:find(LP.Name, 1, true) or t:find(LP.DisplayName, 1, true)) then
                return true
            end
        end
    end
    return false
end

local function findOwnPlot()
    local folder = workspace:FindFirstChild("Plots") or workspace:FindFirstChild("Bases")
    if not folder then return nil end
    for _, plot in ipairs(folder:GetChildren()) do
        if plotBelongsToMe(plot) then return plot end
    end
end

local function guiWorldPos(gui)
    local adornee = gui:IsA("BillboardGui") and gui.Adornee or nil
    if adornee then
        if adornee:IsA("BasePart") then return adornee.Position end
        if adornee:IsA("Attachment") then return adornee.WorldPosition end
    end
    local p = gui.Parent
    if p then
        if p:IsA("BasePart") then return p.Position end
        if p:IsA("Attachment") then return p.WorldPosition end
        local part = p:FindFirstAncestorWhichIsA("BasePart")
        if part then return part.Position end
    end
end

---------------------------------------------------------------- cash parsing
local SUFFIX = { K = 1, M = 2, B = 3, T = 4, Qa = 5, Qi = 6, Sx = 7, Sp = 8, Oc = 9, No = 10, Dc = 11 }

local function parseCash(label)
    if not label or not label.Parent then return nil end
    local t = (label.Text:gsub("[%s,]", ""))
    local num, suf = t:match("^%$([%d%.]+)(%a*)$")
    local n = num and tonumber(num)
    if not n then return nil end
    if suf == "" then return n end
    local idx = SUFFIX[suf]
    if not idx then return nil end
    return n * (1000 ^ idx)
end

local function labelIsZero(label)
    if not label or not label.Parent then return false end
    return (label.Text:gsub("%s", "")) == "$0"
end

-- true = собрано, false = не собрано, nil = проверить нельзя
local function collectedNow(pad, beforeNum)
    local lbl = pad.cash
    if not lbl or not lbl.Parent then return nil end
    if labelIsZero(lbl) then return true end
    local a = parseCash(lbl)
    if a and beforeNum then return a < beforeNum * 0.5 end
    return false
end

---------------------------------------------------------------- pads
local function isGreen(c)
    return c.G > 0.6 and c.R < 0.55 and c.B < 0.55 and (c.G - math.max(c.R, c.B)) > 0.25
end

local function isBlacklisted(text)
    text = text:lower()
    for _, w in ipairs(CFG.Blacklist) do
        if text:find(w, 1, true) then return true end
    end
    return false
end

local function isCollectLabel(d)
    if not (d:IsA("TextLabel") or d:IsA("TextButton")) then return false end
    local t = d.Text:gsub("^%s+", ""):gsub("%s+$", ""):lower()
    if t ~= CFG.PadText:lower() then return false end
    if isBlacklisted(d:GetFullName()) then return false end
    return true
end

local function collectCandidates(scope)
    local list = {}
    local char = LP.Character
    for _, d in ipairs(scope:GetDescendants()) do
        if d:IsA("BasePart") and not (char and d:IsDescendantOf(char)) then
            local sz = d.Size
            local flat = sz.Y <= CFG.PadMaxThickness and sz.X >= CFG.PadMinSide and sz.Z >= CFG.PadMinSide
            local tooBig = sz.X > CFG.PadMaxSide or sz.Z > CFG.PadMaxSide
            local touch = d:FindFirstChildOfClass("TouchInterest") ~= nil
            local green = isGreen(d.Color)
            if not tooBig and not isBlacklisted(d:GetFullName()) and (touch or (green and flat)) then
                local score = (touch and 1 or 0) + (green and 1.5 or 0) + (flat and 0.5 or 0)
                table.insert(list, { part = d, score = score, touch = touch })
            end
        end
    end
    return list
end

local function pickPadPart(cands, labelPos)
    local bestA, dA = nil, math.huge -- зелёная плоская / с TouchInterest+плоская
    local bestB, dB = nil, math.huge -- всё остальное
    for _, c in ipairs(cands) do
        local p = c.part.Position
        local d = flatDist(p, labelPos)
        if d <= CFG.PadSearchRadius and math.abs(p.Y - labelPos.Y) <= 25 then
            if c.score >= 2 then
                if d < dA then bestA, dA = c, d end
            elseif d < dB then
                bestB, dB = c, d
            end
        end
    end
    return bestA or bestB
end

local function readPads(scope)
    local cands = collectCandidates(scope)
    local labels = {}
    for _, d in ipairs(scope:GetDescendants()) do
        if isCollectLabel(d) then
            local gui = d:FindFirstAncestorWhichIsA("BillboardGui") or d:FindFirstAncestorWhichIsA("SurfaceGui")
            local labelPos = gui and guiWorldPos(gui)
            if labelPos then
                local cashLabel
                for _, l in ipairs(gui:GetDescendants()) do
                    if l:IsA("TextLabel") and l.Text:match("^%s*%$") then cashLabel = l break end
                end
                table.insert(labels, { gui = gui, pos = labelPos, cash = cashLabel })
            end
        end
    end

    -- пары метка<->плита, каждая плита и метка используются один раз (ближайшие первыми)
    local pairsList = {}
    for li, lb in ipairs(labels) do
        for _, c in ipairs(cands) do
            local p = c.part.Position
            local d = flatDist(p, lb.pos)
            if d <= CFG.PadSearchRadius and math.abs(p.Y - lb.pos.Y) <= 25 then
                table.insert(pairsList, { li = li, c = c, d = d - (c.score >= 2 and 5 or 0) })
            end
        end
    end
    table.sort(pairsList, function(a, b) return a.d < b.d end)

    local usedLabel, usedPart, pads = {}, {}, {}
    for _, pr in ipairs(pairsList) do
        if not usedLabel[pr.li] and not usedPart[pr.c.part] then
            usedLabel[pr.li], usedPart[pr.c.part] = true, true
            local lb = labels[pr.li]
            table.insert(pads, {
                part = pr.c.part,
                pos = pr.c.part.Position,
                touchPart = pr.c.touch and pr.c.part or nil,
                cash = lb.cash,
                grade = pr.c.score >= 2 and "GREEN" or "weak",
                name = lb.gui:GetFullName(),
            })
        end
    end
    -- метки без плиты: идём к точке под надписью
    for li, lb in ipairs(labels) do
        if not usedLabel[li] then
            local pos = lb.pos
            local _, _, root = getChar()
            local params = RaycastParams.new()
            params.FilterDescendantsInstances = { LP.Character }
            params.FilterType = Enum.RaycastFilterType.Exclude
            local hit = workspace:Raycast(pos, Vector3.new(0, -40, 0), params)
            if hit then pos = hit.Position end
            table.insert(pads, { part = nil, pos = pos, cash = lb.cash, grade = "FALLBACK", name = lb.gui:GetFullName() })
        end
    end
    return pads
end

local function isVisible(obj)
    local cur = obj
    while cur and cur ~= game do
        if cur:IsA("GuiObject") and not cur.Visible then return false end
        if cur:IsA("LayerCollector") and not cur.Enabled then return false end
        cur = cur.Parent
    end
    return true
end

-- пад считается рабочим, только если у него есть видимая надпись с суммой > 0
local function padHasCash(pad)
    if not CFG.SkipEmptyPads then return true end
    local lbl = pad.cash
    if not lbl or not lbl.Parent then return false end
    if not isVisible(lbl) then return false end
    local t = (lbl.Text:gsub("[%s,]", ""))
    if t == "" or t == "$0" or t == "$0.0" then return false end
    local n = parseCash(lbl)
    if n ~= nil and n <= 0 then return false end
    return true
end

local function onPad(pad, rootPos)
    local part = pad.part
    if part and part.Parent then
        local lp = part.CFrame:PointToObjectSpace(rootPos)
        local hx = math.max(part.Size.X / 2 - CFG.EdgeMargin, 0.5)
        local hz = math.max(part.Size.Z / 2 - CFG.EdgeMargin, 0.5)
        return math.abs(lp.X) <= hx and math.abs(lp.Z) <= hz
    end
    return flatDist(rootPos, pad.pos) <= CFG.ReachDist
end

local function touchPad(pad)
    if not (CFG.UseFireTouch and firetouchinterest and pad.touchPart and pad.touchPart.Parent) then return end
    local _, _, root = getChar()
    if not root then return end
    pcall(function()
        firetouchinterest(root, pad.touchPart, 0)
        task.wait(0.05)
        firetouchinterest(root, pad.touchPart, 1)
    end)
end

---------------------------------------------------------------- walking
-- прямая ходьба с прыжком при застревании
local function goDirect(target, arrived, timeout)
    local _, hum, root = getChar()
    if not hum then return false end
    local start = os.clock()
    local lastPos, lastMove, lastIssue = root.Position, os.clock(), 0
    while running and os.clock() - start < timeout do
        local _, h, r = getChar()
        if not h then return false end
        if arrived(r.Position) then return true end
        if os.clock() - lastIssue > 0.5 then
            h:MoveTo(target)
            lastIssue = os.clock()
        end
        if (r.Position - lastPos).Magnitude > 0.5 then
            lastPos, lastMove = r.Position, os.clock()
        elseif os.clock() - lastMove > CFG.StuckJumpAfter then
            h.Jump = true
            lastMove = os.clock()
        end
        RunService.Heartbeat:Wait()
    end
    return false
end

-- ходьба по PathfindingService, при неудаче прямая
local function walkToPoint(target, arrived)
    local _, hum, root = getChar()
    if not hum then return false end
    if arrived(root.Position) then return true end

    if CFG.UsePathfinding then
        local path = PathfindingService:CreatePath({
            AgentRadius = 2, AgentHeight = 5, AgentCanJump = true,
        })
        local ok = pcall(function() path:ComputeAsync(root.Position, target) end)
        if ok and path.Status == Enum.PathStatus.Success then
            for _, wp in ipairs(path:GetWaypoints()) do
                if not running then return false end
                local _, h, r = getChar()
                if not h then return false end
                if wp.Action == Enum.PathWaypointAction.Jump then h.Jump = true end
                goDirect(wp.Position, function(p)
                    return flatDist(p, wp.Position) <= 2.5 or arrived(p)
                end, 2.5)
                local _, _, r2 = getChar()
                if not r2 then return false end
                if arrived(r2.Position) then return true end
            end
        end
    end

    return goDirect(target, arrived, CFG.PadTimeout)
end

local function walkToPad(pad)
    return walkToPoint(pad.pos, function(p) return onPad(pad, p) end)
end

local function stepOff(pad)
    local _, _, root = getChar()
    if not root then return end
    local dir = Vector3.new(root.Position.X - pad.pos.X, 0, root.Position.Z - pad.pos.Z)
    dir = dir.Magnitude > 0.5 and dir.Unit or Vector3.new(0, 0, 1)
    local away = pad.pos + dir * (math.max(pad.part and pad.part.Size.X or 6, 6) / 2 + CFG.StepOffDist)
    goDirect(away, function(p) return flatDist(p, away) <= 2 end, 3)
end

-- true (собрано), false (не подтверждено), nil (нельзя проверить)
local function collectPad(pad)
    for attempt = 1, CFG.Attempts do
        if not running then return false end
        local beforeNum = parseCash(pad.cash)
        if walkToPad(pad) then
            touchPad(pad)
            local t0 = os.clock()
            while running and os.clock() - t0 < CFG.Dwell do
                local c = collectedNow(pad, beforeNum)
                if c == true then return true end
                if c == nil then
                    task.wait(CFG.Dwell)
                    return nil
                end
                task.wait(0.1)
            end
        elseif not running then
            return false
        end
        if attempt < CFG.Attempts and running then stepOff(pad) end
    end
    return false
end

local function nearestFirst(pads, from)
    local ordered, pool, cur = {}, table.clone(pads), from
    while #pool > 0 do
        local bi, bd = 1, math.huge
        for i, p in ipairs(pool) do
            local d = flatDist(cur, p.pos)
            if d < bd then bi, bd = i, d end
        end
        local p = table.remove(pool, bi)
        table.insert(ordered, p)
        cur = p.pos
    end
    return ordered
end

---------------------------------------------------------------- main loop
local function loop()
    local plot, lastScan, pads = nil, 0, {}
    while running do
        local _, _, root = getChar()
        if not root then
            statusText = "Waiting for character"
            task.wait(0.5)
        else
            home = home or root.Position
            if os.clock() - lastScan > CFG.RescanEvery or #pads == 0 then
                if not (plot and plot.Parent) then plot = findOwnPlot() end
                if plot then
                    pads = readPads(plot)
                    statusText = ("Plot found, pads: %d"):format(#pads)
                end
                if #pads == 0 then
                    local all = readPads(workspace)
                    pads = {}
                    for _, p in ipairs(all) do
                        if flatDist(p.pos, home) <= CFG.FallbackRadius then table.insert(pads, p) end
                    end
                    statusText = ("Radius mode, pads: %d"):format(#pads)
                end
                lastScan = os.clock()

                if CFG.Debug then
                    for i, p in ipairs(pads) do
                        print(("[AutoCollect] pad %d [%s]: label=%s | part=%s | cash=%s"):format(
                            i, p.grade, p.name,
                            p.part and p.part:GetFullName() or "none",
                            p.cash and p.cash.Text or "?"))
                    end
                end
            end

            for _, pad in ipairs(nearestFirst(pads, root.Position)) do
                if not running then break end
                if padHasCash(pad) then
                    local r = collectPad(pad)
                    if r == true then
                        okCount += 1
                    elseif r == false and running then
                        missCount += 1
                        if CFG.Debug then print("[AutoCollect] not confirmed: " .. pad.name) end
                    end
                end
            end
            task.wait(CFG.CycleDelay)
        end
    end
    local _, hum, root = getChar()
    if hum and root then hum:MoveTo(root.Position) end
    statusText = "Off"
end

---------------------------------------------------------------- anti-lag + no animation (одна функция)
local antiLagOn = false

local function enableAntiLag()
    if antiLagOn then return end
    antiLagOn = true

    local animators = {}

    local function isPlayerModel(inst)
        local m = inst:FindFirstAncestorOfClass("Model")
        while m do
            if Players:GetPlayerFromCharacter(m) then return true end
            m = m.Parent and m.Parent:FindFirstAncestorOfClass("Model") or nil
        end
        return false
    end

    -- обработка одного объекта: убрать текстуры/эффекты и остановить анимации
    local function process(d)
        pcall(function()
            if d:IsA("Terrain") then return end
            if d:IsA("BasePart") then
                if LP.Character and d:IsDescendantOf(LP.Character) then return end
                d.Material = Enum.Material.SmoothPlastic
                d.CastShadow = false
                d.Reflectance = 0
                if d:IsA("MeshPart") then d.TextureID = "" end
            elseif d:IsA("Decal") or d:IsA("Texture") then
                d.Transparency = 1
            elseif d:IsA("SpecialMesh") then
                d.TextureId = ""
            elseif d:IsA("ParticleEmitter") or d:IsA("Trail") or d:IsA("Beam")
                or d:IsA("Smoke") or d:IsA("Fire") or d:IsA("Sparkles") then
                d.Enabled = false
            elseif d:IsA("PostEffect") then
                d.Enabled = false
            elseif d:IsA("Explosion") then
                d.Visible = false
            elseif d:IsA("Animator") and not animators[d] and not isPlayerModel(d) then
                animators[d] = true
                d.AnimationPlayed:Connect(function(track)
                    pcall(function() track:Stop(0) end)
                end)
                d.Destroying:Connect(function() animators[d] = nil end)
            end
        end)
    end

    -- освещение, вода, качество графики
    pcall(function()
        Lighting.GlobalShadows = false
        Lighting.FogEnd = 9e9
        Lighting.Brightness = 2
        for _, e in ipairs(Lighting:GetDescendants()) do process(e) end
        local atm = Lighting:FindFirstChildOfClass("Atmosphere")
        if atm then atm.Density = 0 atm.Haze = 0 end
    end)
    pcall(function()
        local t = workspace.Terrain
        t.WaterWaveSize, t.WaterWaveSpeed = 0, 0
        t.WaterReflectance, t.WaterTransparency = 0, 1
    end)
    pcall(function() sethiddenproperty(workspace.Terrain, "Decoration", false) end)
    pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)

    -- весь существующий мир (пачками, чтобы не фризить)
    task.spawn(function()
        local n = 0
        for _, d in ipairs(workspace:GetDescendants()) do
            process(d)
            n += 1
            if n % 400 == 0 then task.wait() end
        end
    end)

    -- новые объекты
    workspace.DescendantAdded:Connect(process)

    -- повторная остановка анимаций, если игра запускает их снова
    task.spawn(function()
        while antiLagOn do
            for a in pairs(animators) do
                pcall(function()
                    for _, t in ipairs(a:GetPlayingAnimationTracks()) do t:Stop(0) end
                end)
            end
            task.wait(0.3)
        end
    end)
end

---------------------------------------------------------------- GUI
local parent = (gethui and gethui()) or game:GetService("CoreGui")
pcall(function()
    local old = parent:FindFirstChild("AutoCollectCashGui")
    if old then old:Destroy() end
end)

local gui = Instance.new("ScreenGui")
gui.Name = "AutoCollectCashGui"
gui.ResetOnSpawn = false
gui.Parent = parent

local frame = Instance.new("Frame")
frame.Size = UDim2.fromOffset(230, 140)
frame.Position = UDim2.new(0, 20, 0.4, 0)
frame.BackgroundColor3 = Color3.fromRGB(20, 20, 24)
frame.BorderSizePixel = 0
frame.Active = true
frame.Parent = gui
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 24)
title.BackgroundTransparency = 1
title.Text = "David and Tixan Auto Collect Cash"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 12
title.TextScaled = false
title.TextTruncate = Enum.TextTruncate.AtEnd
title.Parent = frame

local btn = Instance.new("TextButton")
btn.Size = UDim2.new(1, -20, 0, 32)
btn.Position = UDim2.fromOffset(10, 28)
btn.BackgroundColor3 = Color3.fromRGB(170, 50, 50)
btn.Text = "OFF"
btn.TextColor3 = Color3.new(1, 1, 1)
btn.Font = Enum.Font.GothamBold
btn.TextSize = 14
btn.Parent = frame
Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)

local lagBtn = Instance.new("TextButton")
lagBtn.Size = UDim2.new(1, -20, 0, 26)
lagBtn.Position = UDim2.fromOffset(10, 64)
lagBtn.BackgroundColor3 = Color3.fromRGB(50, 80, 140)
lagBtn.Text = "Anti-Lag + No Anim"
lagBtn.TextColor3 = Color3.new(1, 1, 1)
lagBtn.Font = Enum.Font.GothamBold
lagBtn.TextSize = 12
lagBtn.Parent = frame
Instance.new("UICorner", lagBtn).CornerRadius = UDim.new(0, 6)

local info = Instance.new("TextLabel")
info.Size = UDim2.new(1, -20, 0, 42)
info.Position = UDim2.fromOffset(10, 94)
info.BackgroundTransparency = 1
info.TextColor3 = Color3.fromRGB(200, 200, 200)
info.Font = Enum.Font.Gotham
info.TextSize = 11
info.TextWrapped = true
info.TextXAlignment = Enum.TextXAlignment.Left
info.TextYAlignment = Enum.TextYAlignment.Top
info.Text = "Off"
info.Parent = frame

btn.Activated:Connect(function()
    running = not running
    if running then
        home = nil
        okCount, missCount = 0, 0
        btn.Text = "ON"
        btn.BackgroundColor3 = Color3.fromRGB(50, 150, 70)
        task.spawn(loop)
    else
        btn.Text = "OFF"
        btn.BackgroundColor3 = Color3.fromRGB(170, 50, 50)
    end
end)

lagBtn.Activated:Connect(function()
    enableAntiLag()
    lagBtn.Text = "Anti-Lag + No Anim: ON"
    lagBtn.BackgroundColor3 = Color3.fromRGB(50, 150, 70)
end)

task.spawn(function()
    while gui.Parent do
        info.Text = ("%s\nok: %d | miss: %d"):format(statusText, okCount, missCount)
        task.wait(0.3)
    end
end)

-- перетаскивание окна
local dragging, dragStart, startPos
frame.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
        dragging, dragStart, startPos = true, i.Position, frame.Position
        i.Changed:Connect(function()
            if i.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(i)
    if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
        local d = i.Position - dragStart
        frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
    end
end)
