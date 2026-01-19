-- Glory Edition - ESP (Skeleton + Movable GUI) - GLORIUS GANG HUB title
-- LocalScript (StarterPlayerScripts / StarterGui)
-- Features:
--  - Skeleton ESP (R15 & R6 support, draws bone lines)
--  - Movable GUI: drag the menu by its title bar
--  - Keeps previous features: Box, Tracer, Name, Distance, HealthBar, Chams, Rainbow, TeamCheck
--  - Optimized and resilient (Drawing fallback to BillboardGui)
-- Usage: place as a LocalScript on the client.

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

-- Cached refs
local LocalPlayer = Players.LocalPlayer or Players.PlayerAdded:Wait()
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Camera = Workspace.CurrentCamera

-- Configuration
local AdvancedESP = {
    Enabled = true,
    Box = true,
    Tracer = true,
    Name = true,
    Distance = true,
    HealthBar = true,
    Skeleton = true,       -- now enabled by default (you can toggle)
    Chams = false,
    Rainbow = false,
    TeamCheck = false,
    ShowTeam = false,
    MaxDistance = 1500,

    -- visuals
    BoxColor = Color3.fromRGB(0, 255, 100),
    TracerColor = Color3.fromRGB(255, 255, 0),
    NameColor = Color3.fromRGB(255, 255, 255),
    HealthColorFull = Color3.fromRGB(0, 255, 0),
    HealthColorLow = Color3.fromRGB(255, 0, 0),
    ChamsColor = Color3.fromRGB(255, 100, 255),

    RainbowSpeed = 2,
    Thickness = 1.5,
    Transparency = 0.9,
    TracerOrigin = "Bottom"
}

-- State
local UseDrawing = pcall(function() return Drawing end)
local ESP_Containers = {} -- [player] = container
local RainbowTime = 0

-- Utilities
local function clamp(v, a, b) return math.clamp(v, a, b) end

local function safeSetVisible(obj, visible)
    if not obj then return end
    pcall(function()
        if obj.Visible ~= nil then obj.Visible = visible end
        if obj.Enabled ~= nil then obj.Enabled = visible end
    end)
end

local function safeRemove(obj)
    if not obj then return end
    pcall(function()
        local t = typeof(obj)
        if t == "userdata" or t == "Drawing" then
            if obj.Visible ~= nil then obj.Visible = false end
            if obj.Remove then pcall(obj.Remove, obj) end
            if obj.Destroy then pcall(obj.Destroy, obj) end
        else
            if obj.Destroy then obj:Destroy() end
        end
    end)
end

local function getRoot(character)
    if not character then return nil end
    return character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso")
end

local function isValidTarget(plr)
    if not plr or plr == LocalPlayer then return false end
    local char = plr.Character
    if not char then return false end
    local root = getRoot(char)
    if not root then return false end
    local hum = char:FindFirstChildWhichIsA("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    if AdvancedESP.TeamCheck and (not AdvancedESP.ShowTeam) and plr.Team == LocalPlayer.Team then return false end
    local cam = Camera or Workspace.CurrentCamera
    if not cam then return false end
    local distance = (root.Position - cam.CFrame.Position).Magnitude
    if distance > AdvancedESP.MaxDistance then return false end
    return true, char, root, hum, distance
end

local function hsvRainbow(dt)
    RainbowTime = RainbowTime + dt * AdvancedESP.RainbowSpeed
    return Color3.fromHSV((RainbowTime % 1), 1, 1)
end

local function getColorForHealth(hum)
    if AdvancedESP.Rainbow then
        return Color3.fromHSV((tick() % 5) / 5, 1, 1)
    end
    if not hum then return AdvancedESP.BoxColor end
    local maxH = (hum.MaxHealth > 0) and hum.MaxHealth or 100
    local pct = clamp(hum.Health / maxH, 0, 1)
    return AdvancedESP.HealthColorFull:Lerp(AdvancedESP.HealthColorLow, 1 - pct)
end

-- Drawing constructors
local function newLine()
    if not UseDrawing then return nil end
    local ok, l = pcall(function() return Drawing.new("Line") end)
    if not ok or not l then return nil end
    pcall(function()
        if l.Thickness ~= nil then l.Thickness = AdvancedESP.Thickness end
        if l.Transparency ~= nil then l.Transparency = AdvancedESP.Transparency end
        if l.Color ~= nil then l.Color = AdvancedESP.BoxColor end
        l.Visible = false
    end)
    return l
end

local function newText()
    if not UseDrawing then return nil end
    local ok, t = pcall(function() return Drawing.new("Text") end)
    if not ok or not t then return nil end
    pcall(function()
        t.Size = 14
        t.Color = AdvancedESP.NameColor
        t.Center = true
        t.Outline = true
        t.Visible = false
    end)
    return t
end

local function createBillboardForCharacter(plr)
    local char = plr.Character
    if not char then return nil end
    local head = char:FindFirstChild("Head")
    if not head then return nil end
    local bill = Instance.new("BillboardGui")
    bill.Name = "ESP_Billboard"
    bill.Adornee = head
    bill.Size = UDim2.new(0, 220, 0, 80)
    bill.StudsOffset = Vector3.new(0, 2.4, 0)
    bill.AlwaysOnTop = true
    bill.Parent = head

    local nameLabel = Instance.new("TextLabel", bill)
    nameLabel.Size = UDim2.new(1,0,0,28)
    nameLabel.Position = UDim2.new(0,0,0,0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.TextScaled = true
    nameLabel.Font = Enum.Font.SourceSansBold
    nameLabel.Text = plr.Name
    nameLabel.TextColor3 = AdvancedESP.NameColor

    local distLabel = Instance.new("TextLabel", bill)
    distLabel.Size = UDim2.new(1,0,0,20)
    distLabel.Position = UDim2.new(0,0,0,30)
    distLabel.BackgroundTransparency = 1
    distLabel.TextScaled = true
    distLabel.Font = Enum.Font.SourceSans
    distLabel.Text = ""
    distLabel.TextColor3 = Color3.fromRGB(200,200,200)

    local healthBarBG = Instance.new("Frame", bill)
    healthBarBG.Size = UDim2.new(0, 6, 0, 46)
    healthBarBG.Position = UDim2.new(0, -8, 0, 6)
    healthBarBG.BackgroundColor3 = Color3.fromRGB(50,50,50)
    healthBarBG.BorderSizePixel = 0

    local healthBar = Instance.new("Frame", healthBarBG)
    healthBar.Size = UDim2.new(1,0,0,0)
    healthBar.Position = UDim2.new(0,0,1,0)
    healthBar.AnchorPoint = Vector2.new(0,1)
    healthBar.BackgroundColor3 = AdvancedESP.HealthColorFull
    healthBar.BorderSizePixel = 0

    return {
        Billboard = bill,
        NameLabel = nameLabel,
        DistLabel = distLabel,
        HealthBarFrame = healthBar,
        HealthBarBG = healthBarBG
    }
end

-- Skeleton helper: produce bone pairs (Vector3) for a given character
local function getSkeletonPairs(char)
    if not char then return {} end
    local pairsOut = {}

    -- Helper to safe-get part Position
    local function pos(name)
        local p = char:FindFirstChild(name)
        if p and p:IsA("BasePart") then return p.Position end
        return nil
    end

    -- Try R15 mapping first
    local head = pos("Head")
    local upperTorso = pos("UpperTorso")
    local lowerTorso = pos("LowerTorso")
    local root = pos("HumanoidRootPart") or lowerTorso
    local lUpperArm = pos("LeftUpperArm")
    local lLowerArm = pos("LeftLowerArm")
    local lHand = pos("LeftHand")
    local rUpperArm = pos("RightUpperArm")
    local rLowerArm = pos("RightLowerArm")
    local rHand = pos("RightHand")
    local lUpperLeg = pos("LeftUpperLeg")
    local lLowerLeg = pos("LeftLowerLeg")
    local lFoot = pos("LeftFoot")
    local rUpperLeg = pos("RightUpperLeg")
    local rLowerLeg = pos("RightLowerLeg")
    local rFoot = pos("RightFoot")

    -- if R15 parts exist, build R15 pairs
    if head and upperTorso and lowerTorso then
        local function add(a, b) if a and b then table.insert(pairsOut, {a, b}) end end
        add(head, upperTorso)
        add(upperTorso, lowerTorso)
        add(lowerTorso, root)
        add(upperTorso, lUpperArm); add(lUpperArm, lLowerArm); add(lLowerArm, lHand)
        add(upperTorso, rUpperArm); add(rUpperArm, rLowerArm); add(rLowerArm, rHand)
        add(lowerTorso, lUpperLeg); add(lUpperLeg, lLowerLeg); add(lLowerLeg, lFoot)
        add(lowerTorso, rUpperLeg); add(rUpperLeg, rLowerLeg); add(rLowerLeg, rFoot)
        return pairsOut
    end

    -- Fallback to R6 (Torso-based)
    local torso = char:FindFirstChild("Torso")
    local headR6 = char:FindFirstChild("Head")
    local leftArm = char:FindFirstChild("Left Arm") or char:FindFirstChild("LeftArm")
    local rightArm = char:FindFirstChild("Right Arm") or char:FindFirstChild("RightArm")
    local leftLeg = char:FindFirstChild("Left Leg") or char:FindFirstChild("LeftLeg")
    local rightLeg = char:FindFirstChild("Right Leg") or char:FindFirstChild("RightLeg")
    if torso and headR6 then
        if headR6 then table.insert(pairsOut, {headR6.Position, torso.Position}) end
        if leftArm and torso then table.insert(pairsOut, {leftArm.Position, torso.Position}) end
        if rightArm and torso then table.insert(pairsOut, {rightArm.Position, torso.Position}) end
        if leftLeg and torso then table.insert(pairsOut, {leftLeg.Position, torso.Position}) end
        if rightLeg and torso then table.insert(pairsOut, {rightLeg.Position, torso.Position}) end
        return pairsOut
    end

    return pairsOut
end

-- Create ESP container for a player (includes skeleton lines)
local function createESP(plr)
    if not plr or plr == LocalPlayer then return end
    if ESP_Containers[plr] then return end

    local cont = {}
    ESP_Containers[plr] = cont

    -- Box lines
    cont.BoxLines = {}
    if AdvancedESP.Box then
        for i = 1, 8 do cont.BoxLines[i] = newLine() end
    end

    -- Tracer
    cont.Tracer = nil
    if AdvancedESP.Tracer then cont.Tracer = newLine() end

    -- Texts & health (drawing) or billboard fallback
    cont.HealthBar = nil
    cont.NameText = nil
    cont.DistText = nil
    if UseDrawing then
        if AdvancedESP.HealthBar then cont.HealthBar = newLine() end
        if AdvancedESP.Name then cont.NameText = newText() end
        if AdvancedESP.Distance then cont.DistText = newText() end
    else
        cont.Billboard = createBillboardForCharacter(plr)
    end

    -- Skeleton lines (precreate some to avoid allocation in render loop)
    cont.SkeletonLines = {}
    if AdvancedESP.Skeleton then
        for i = 1, 20 do -- up to 20 bone lines (enough for R15)
            cont.SkeletonLines[i] = newLine()
        end
    end

    -- Chams container
    cont.Chams = {}

    -- CharacterAdded: rebuild billboard & chams
    plr.CharacterAdded:Connect(function(char)
        task.wait(0.25)
        if not UseDrawing then
            if cont.Billboard and cont.Billboard.Billboard and cont.Billboard.Billboard.Parent then
                pcall(function() cont.Billboard.Billboard:Destroy() end)
            end
            cont.Billboard = createBillboardForCharacter(plr)
        end

        if AdvancedESP.Chams and char then
            for _, part in ipairs(char:GetDescendants()) do
                if part:IsA("BasePart") then
                    local ok, hl = pcall(function()
                        local h = Instance.new("Highlight")
                        h.Name = "ESP_Highlight"
                        h.Parent = part
                        h.Adornee = part
                        h.FillColor = AdvancedESP.ChamsColor
                        h.OutlineColor = Color3.fromRGB(255,255,255)
                        h.FillTransparency = 0.6
                        h.OutlineTransparency = 0.1
                        h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                        return h
                    end)
                    if ok and hl then table.insert(cont.Chams, hl) end
                end
            end
        end
    end)

    -- CharacterRemoving: remove highlights
    plr.CharacterRemoving:Connect(function()
        if cont.Chams then
            for _, hl in ipairs(cont.Chams) do
                pcall(function() if hl and hl.Destroy then hl:Destroy() end end)
            end
            cont.Chams = {}
        end
    end)

    -- Player leaving cleanup
    plr.AncestryChanged:Connect(function(_, parent)
        if not parent then
            if cont.BoxLines then for _, v in ipairs(cont.BoxLines) do safeRemove(v) end end
            safeRemove(cont.Tracer)
            safeRemove(cont.HealthBar)
            safeRemove(cont.NameText)
            safeRemove(cont.DistText)
            if cont.Billboard and cont.Billboard.Billboard then pcall(function() cont.Billboard.Billboard:Destroy() end) end
            if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeRemove(l) end end
            if cont.Chams then for _, hl in ipairs(cont.Chams) do pcall(function() if hl and hl.Destroy then hl:Destroy() end end) end end
            ESP_Containers[plr] = nil
        end
    end)
end

-- Initialize ESP containers for present players and hook for future players
for _, plr in ipairs(Players:GetPlayers()) do if plr ~= LocalPlayer then createESP(plr) end end
Players.PlayerAdded:Connect(function(plr) if plr ~= LocalPlayer then createESP(plr) end end)

-- Skeleton drawing helper (screen space)
local function drawSkeleton(cont, char)
    if not cont.SkeletonLines then return end
    local pairs3 = getSkeletonPairs(char)
    for i = 1, #cont.SkeletonLines do
        local line = cont.SkeletonLines[i]
        local pair = pairs3[i]
        if line and pair then
            local a3, b3 = pair[1], pair[2]
            -- convert to screen
            local a2 = Camera:WorldToViewportPoint(a3)
            local b2 = Camera:WorldToViewportPoint(b3)
            local aOn = a2 and a2.Z > 0
            local bOn = b2 and b2.Z > 0
            pcall(function()
                line.From = Vector2.new(a2.X, a2.Y)
                line.To = Vector2.new(b2.X, b2.Y)
                line.Color = AdvancedESP.Rainbow and hsvRainbow(0.016) or AdvancedESP.BoxColor
                line.Thickness = AdvancedESP.Thickness
                line.Transparency = AdvancedESP.Transparency
                line.Visible = AdvancedESP.Enabled and AdvancedESP.Skeleton and (aOn or bOn)
            end)
        elseif line then
            safeSetVisible(line, false)
        end
    end
end

-- Main update loop
local function UpdateESP(delta)
    Camera = Workspace.CurrentCamera or Camera
    if not Camera then return end
    if AdvancedESP.Rainbow then hsvRainbow(delta) end

    for plr, cont in pairs(ESP_Containers) do
        local valid, char, root, hum, distance = isValidTarget(plr)
        if not valid then
            if cont.BoxLines then for _, l in ipairs(cont.BoxLines) do safeSetVisible(l, false) end end
            safeSetVisible(cont.Tracer, false)
            safeSetVisible(cont.HealthBar, false)
            safeSetVisible(cont.NameText, false)
            safeSetVisible(cont.DistText, false)
            if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeSetVisible(l, false) end end
            if cont.Billboard and cont.Billboard.Billboard then pcall(function() cont.Billboard.Billboard.Enabled = false end) end
            if cont.Chams and not AdvancedESP.Chams then
                for _, hl in ipairs(cont.Chams) do pcall(function() if hl and hl.Destroy then hl:Destroy() end end) end
                cont.Chams = {}
            end
        else
            if not (char and char.Parent) then
                -- skip
            else
                local head = char:FindFirstChild("Head")
                if not head then
                    -- skip
                else
                    local head3 = head.Position + Vector3.new(0, 0.8, 0)
                    local leg3 = root.Position - Vector3.new(0, 3.5, 0)

                    local headScreen, headOnScreen = Camera:WorldToViewportPoint(head3)
                    local legScreen, legOnScreen = Camera:WorldToViewportPoint(leg3)

                    if not (headOnScreen or legOnScreen) then
                        if cont.BoxLines then for _, l in ipairs(cont.BoxLines) do safeSetVisible(l, false) end end
                        safeSetVisible(cont.Tracer, false)
                        safeSetVisible(cont.HealthBar, false)
                        safeSetVisible(cont.NameText, false)
                        safeSetVisible(cont.DistText, false)
                        if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeSetVisible(l, false) end end
                        if cont.Billboard and cont.Billboard.Billboard then pcall(function() cont.Billboard.Billboard.Enabled = false end) end
                    else
                        local height = math.abs(headScreen.Y - legScreen.Y)
                        if height < 10 then height = 10 end
                        local width = math.clamp(height * 0.55, 8, 600)

                        local topLeft = Vector2.new(headScreen.X - width/2, headScreen.Y)
                        local topRight = Vector2.new(headScreen.X + width/2, headScreen.Y)
                        local botLeft = Vector2.new(legScreen.X - width/2, legScreen.Y)
                        local botRight = Vector2.new(legScreen.X + width/2, legScreen.Y)

                        local color = AdvancedESP.Rainbow and Color3.fromHSV((tick()%1),1,1) or AdvancedESP.BoxColor
                        if hum then color = getColorForHealth(hum) end

                        -- Box
                        if AdvancedESP.Box and cont.BoxLines then
                            local segs = {
                                {From = topLeft, To = Vector2.new(topLeft.X + width/4, topLeft.Y)},
                                {From = topLeft, To = Vector2.new(topLeft.X, topLeft.Y + height/4)},
                                {From = topRight, To = Vector2.new(topRight.X - width/4, topRight.Y)},
                                {From = topRight, To = Vector2.new(topRight.X, topRight.Y + height/4)},
                                {From = botLeft, To = Vector2.new(botLeft.X + width/4, botLeft.Y)},
                                {From = botLeft, To = Vector2.new(botLeft.X, botLeft.Y - height/4)},
                                {From = botRight, To = Vector2.new(botRight.X - width/4, botRight.Y)},
                                {From = botRight, To = Vector2.new(botRight.X, botRight.Y - height/4)}
                            }
                            for i = 1, 8 do
                                local line = cont.BoxLines[i]
                                if line then
                                    pcall(function()
                                        line.From = segs[i].From
                                        line.To = segs[i].To
                                        line.Color = color
                                        line.Thickness = AdvancedESP.Thickness
                                        line.Transparency = AdvancedESP.Transparency
                                        line.Visible = AdvancedESP.Enabled
                                    end)
                                end
                            end
                        end

                        -- Tracer
                        if AdvancedESP.Tracer and cont.Tracer then
                            local fromPos
                            if AdvancedESP.TracerOrigin == "Bottom" then
                                fromPos = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y)
                            elseif AdvancedESP.TracerOrigin == "Center" then
                                local v = Camera.ViewportSize / 2
                                fromPos = Vector2.new(v.X, v.Y)
                            elseif AdvancedESP.TracerOrigin == "Mouse" then
                                fromPos = UserInputService:GetMouseLocation()
                            else
                                fromPos = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y)
                            end
                            pcall(function()
                                cont.Tracer.From = fromPos
                                cont.Tracer.To = Vector2.new((botLeft.X + botRight.X)/2, botLeft.Y)
                                cont.Tracer.Color = AdvancedESP.Rainbow and hsvRainbow(delta) or AdvancedESP.TracerColor
                                cont.Tracer.Visible = AdvancedESP.Enabled
                            end)
                        end

                        -- HealthBar (Drawing)
                        if AdvancedESP.HealthBar and cont.HealthBar then
                            local maxH = (hum and hum.MaxHealth > 0) and hum.MaxHealth or 100
                            local pct = hum and math.clamp(hum.Health / maxH, 0, 1) or 0
                            local barH = height * pct
                            pcall(function()
                                cont.HealthBar.From = Vector2.new(topLeft.X - 8, topLeft.Y + height)
                                cont.HealthBar.To = Vector2.new(topLeft.X - 8, topLeft.Y + height - barH)
                                cont.HealthBar.Color = getColorForHealth(hum)
                                cont.HealthBar.Thickness = 4
                                cont.HealthBar.Visible = AdvancedESP.Enabled
                            end)
                        end

                        -- Texts (Drawing)
                        if UseDrawing then
                            if AdvancedESP.Name and cont.NameText then
                                pcall(function()
                                    cont.NameText.Text = plr.Name
                                    cont.NameText.Position = Vector2.new(headScreen.X, headScreen.Y - 18)
                                    cont.NameText.Color = color
                                    cont.NameText.Visible = AdvancedESP.Enabled
                                end)
                            end
                            if AdvancedESP.Distance and cont.DistText then
                                pcall(function()
                                    cont.DistText.Text = math.floor(distance) .. " studs"
                                    cont.DistText.Position = Vector2.new(headScreen.X, headScreen.Y - 34)
                                    cont.DistText.Color = Color3.fromRGB(200,200,200)
                                    cont.DistText.Visible = AdvancedESP.Enabled
                                end)
                            end
                        else
                            -- Billboard fallback
                            if cont.Billboard and cont.Billboard.Billboard then
                                pcall(function()
                                    cont.Billboard.Billboard.Enabled = AdvancedESP.Enabled
                                    if cont.Billboard.NameLabel then
                                        cont.Billboard.NameLabel.Text = plr.Name
                                        cont.Billboard.NameLabel.TextColor3 = color
                                    end
                                    if cont.Billboard.DistLabel then
                                        cont.Billboard.DistLabel.Text = math.floor(distance) .. " studs"
                                    end
                                    if cont.Billboard.HealthBarFrame and hum then
                                        local maxH = hum.MaxHealth > 0 and hum.MaxHealth or 100
                                        local pct = math.clamp(hum.Health / maxH, 0, 1)
                                        cont.Billboard.HealthBarFrame.Size = UDim2.new(1,0,pct * 1,0)
                                        cont.Billboard.HealthBarFrame.BackgroundColor3 = getColorForHealth(hum)
                                    end
                                end)
                            end
                        end

                        -- Skeleton
                        if AdvancedESP.Skeleton and cont.SkeletonLines then
                            drawSkeleton(cont, char)
                        else
                            if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeSetVisible(l, false) end end
                        end

                        -- Chams
                        if AdvancedESP.Chams then
                            cont.Chams = cont.Chams or {}
                            if char then
                                for _, part in ipairs(char:GetDescendants()) do
                                    if part:IsA("BasePart") then
                                        local already = false
                                        for _, hl in ipairs(cont.Chams) do
                                            if hl and hl.Adornee == part then already = true; break end
                                        end
                                        if not already then
                                            local ok, hl = pcall(function()
                                                local h = Instance.new("Highlight")
                                                h.Name = "ESP_Highlight"
                                                h.Parent = part
                                                h.Adornee = part
                                                h.FillColor = AdvancedESP.ChamsColor
                                                h.OutlineColor = Color3.fromRGB(255,255,255)
                                                h.FillTransparency = 0.6
                                                h.OutlineTransparency = 0.1
                                                h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                                                return h
                                            end)
                                            if ok and hl then table.insert(cont.Chams, hl) end
                                        end
                                    end
                                end
                            end
                        else
                            if cont.Chams then
                                for _, hl in ipairs(cont.Chams) do pcall(function() if hl and hl.Destroy then hl:Destroy() end end) end
                                cont.Chams = {}
                            end
                        end
                    end
                end
            end
        end
    end
end

-- Connect update
if RunService and RunService.RenderStepped then
    RunService.RenderStepped:Connect(UpdateESP)
elseif RunService and RunService.Heartbeat then
    RunService.Heartbeat:Connect(function(dt) UpdateESP(dt) end)
else
    warn("No RunService stepping available; ESP will not update automatically.")
end

-- GUI: larger, two-column, animated buttons, draggable by title
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "GloryESPMenu"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.Parent = PlayerGui

local targetSize = UDim2.new(0, 560, 0, 600)
local closedSize = UDim2.new(0, 0, 0, 0)

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = closedSize
mainFrame.Position = UDim2.new(0, 20, 0, 80)
mainFrame.BackgroundColor3 = Color3.fromRGB(12,12,12)
mainFrame.BorderSizePixel = 0
mainFrame.ClipsDescendants = true
mainFrame.Active = true
mainFrame.Parent = screenGui

-- TitleBar (use as drag handle)
local titleBar = Instance.new("Frame")
titleBar.Name = "TitleBar"
titleBar.Size = UDim2.new(1, 0, 0, 56)
titleBar.Position = UDim2.new(0, 0, 0, 0)
titleBar.BackgroundTransparency = 1
titleBar.Parent = mainFrame

local title = Instance.new("TextLabel")
title.Name = "Title"
title.Size = UDim2.new(1, -28, 0, 28)
title.Position = UDim2.new(0, 14, 0, 12)
title.BackgroundTransparency = 1
title.Text = "GLORIUS GANG HUB" -- <- Updated title as requested
title.TextColor3 = Color3.new(1,1,1)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 20
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = titleBar

local subtitle = Instance.new("TextLabel")
subtitle.Name = "Subtitle"
subtitle.Size = UDim2.new(1, -28, 0, 18)
subtitle.Position = UDim2.new(0, 14, 0, 36)
subtitle.BackgroundTransparency = 1
subtitle.Text = "Clique para alternar. Arraste para mover."
subtitle.TextColor3 = Color3.fromRGB(200,200,200)
subtitle.Font = Enum.Font.SourceSans
subtitle.TextSize = 12
subtitle.TextXAlignment = Enum.TextXAlignment.Left
subtitle.Parent = titleBar

-- Button container
local buttonContainer = Instance.new("Frame")
buttonContainer.Name = "ButtonContainer"
buttonContainer.Size = UDim2.new(1, -28, 1, -100)
buttonContainer.Position = UDim2.new(0, 14, 0, 76)
buttonContainer.BackgroundTransparency = 1
buttonContainer.Parent = mainFrame

local grid = Instance.new("UIGridLayout")
grid.Parent = buttonContainer
grid.SortOrder = Enum.SortOrder.LayoutOrder
grid.CellPadding = UDim2.new(0, 10, 0, 10)
grid.CellSize = UDim2.new(0.48, 0, 0, 50)
grid.FillDirection = Enum.FillDirection.Horizontal

-- Tween presets and helper
local tweenFast = TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local tweenMed = TweenInfo.new(0.28, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local function tweenObject(obj, props, info)
    info = info or tweenFast
    local ok, tween = pcall(function() return TweenService:Create(obj, info, props) end)
    if ok and tween then pcall(function() tween:Play() end) end
end

-- Animated toggle creation
local function createAnimatedToggle(labelText, key)
    local btn = Instance.new("TextButton")
    btn.Name = "Btn_" .. key
    btn.Size = UDim2.new(1, 0, 0, 50)
    btn.BackgroundColor3 = Color3.fromRGB(28,28,28)
    btn.BorderSizePixel = 0
    btn.TextColor3 = Color3.new(1,1,1)
    btn.Font = Enum.Font.SourceSans
    btn.TextSize = 16
    btn.AutoButtonColor = false
    btn.Text = labelText

    local indicator = Instance.new("Frame", btn)
    indicator.Name = "Indicator"
    indicator.Size = UDim2.new(0, 44, 0, 26)
    indicator.Position = UDim2.new(1, -52, 0.5, -13)
    indicator.BackgroundColor3 = Color3.fromRGB(80,80,80)
    indicator.BorderSizePixel = 0
    local indCorner = Instance.new("UICorner", indicator); indCorner.CornerRadius = UDim.new(0.5, 0)

    local dot = Instance.new("Frame", indicator)
    dot.Name = "Dot"
    dot.Size = UDim2.new(0, 18, 0, 18)
    dot.Position = UDim2.new(0, 6, 0.5, -9)
    dot.BackgroundColor3 = Color3.fromRGB(230,230,230)
    dot.BorderSizePixel = 0
    local dotCorner = Instance.new("UICorner", dot); dotCorner.CornerRadius = UDim.new(1,0)

    local function updateVisual(state, animate)
        local onBg = Color3.fromRGB(230,230,230)
        local offBg = Color3.fromRGB(28,28,28)
        local onText = Color3.fromRGB(8,8,8)
        local offText = Color3.new(1,1,1)
        local indOn = Color3.fromRGB(0,170,70)
        local indOff = Color3.fromRGB(80,80,80)
        if animate then
            tweenObject(btn, {BackgroundColor3 = state and onBg or offBg}, tweenMed)
            tweenObject(btn, {TextColor3 = state and onText or offText}, tweenMed)
            tweenObject(indicator, {BackgroundColor3 = state and indOn or indOff}, tweenMed)
            local dotPos = state and UDim2.new(1, -24, 0.5, -9) or UDim2.new(0, 6, 0.5, -9)
            tweenObject(dot, {Position = dotPos}, tweenMed)
            -- pulse
            tweenObject(btn, {Size = UDim2.new(1, 8, 0, 54)}, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out))
            delay(0.12, function()
                tweenObject(btn, {Size = UDim2.new(1, 0, 0, 50)}, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out))
            end)
        else
            btn.BackgroundColor3 = state and onBg or offBg
            btn.TextColor3 = state and onText or offText
            indicator.BackgroundColor3 = state and indOn or indOff
            dot.Position = state and UDim2.new(1, -24, 0.5, -9) or UDim2.new(0, 6, 0.5, -9)
        end
    end

    updateVisual(AdvancedESP[key], false)

    btn.MouseButton1Click:Connect(function()
        AdvancedESP[key] = not AdvancedESP[key]
        updateVisual(AdvancedESP[key], true)
        if key == "Enabled" and not AdvancedESP.Enabled then
            for _, cont in pairs(ESP_Containers) do
                if cont.BoxLines then for _, l in ipairs(cont.BoxLines) do safeSetVisible(l, false) end end
                safeSetVisible(cont.Tracer, false)
                safeSetVisible(cont.HealthBar, false)
                safeSetVisible(cont.NameText, false)
                safeSetVisible(cont.DistText, false)
                if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeSetVisible(l, false) end end
                if cont.Billboard and cont.Billboard.Billboard then pcall(function() cont.Billboard.Billboard.Enabled = false end) end
            end
        end
    end)

    return btn
end

-- Toggle list
local toggles = {
    {label = "Master", key = "Enabled"},
    {label = "Box", key = "Box"},
    {label = "Tracer", key = "Tracer"},
    {label = "Name", key = "Name"},
    {label = "Distance", key = "Distance"},
    {label = "HealthBar", key = "HealthBar"},
    {label = "Chams", key = "Chams"},
    {label = "Rainbow", key = "Rainbow"},
    {label = "TeamCheck", key = "TeamCheck"},
    {label = "Skeleton", key = "Skeleton"}
}

local buttons = {}
for _, t in ipairs(toggles) do
    buttons[t.key] = createAnimatedToggle(t.label, t.key)
    buttons[t.key].Parent = buttonContainer
end

-- MaxDistance controls
local distFrame = Instance.new("Frame")
distFrame.Size = UDim2.new(1, 0, 0, 64)
distFrame.BackgroundTransparency = 1
distFrame.LayoutOrder = 999
distFrame.Parent = buttonContainer

local distLabel = Instance.new("TextLabel", distFrame)
distLabel.Size = UDim2.new(1, -140, 0, 28)
distLabel.Position = UDim2.new(0, 10, 0, 8)
distLabel.BackgroundTransparency = 1
distLabel.Text = "MaxDistance: " .. tostring(AdvancedESP.MaxDistance)
distLabel.Font = Enum.Font.SourceSans
distLabel.TextSize = 16
distLabel.TextColor3 = Color3.new(1,1,1)
distLabel.TextXAlignment = Enum.TextXAlignment.Left

local minusBtn = Instance.new("TextButton", distFrame)
minusBtn.Size = UDim2.new(0, 64, 0, 28)
minusBtn.Position = UDim2.new(1, -120, 0, 8)
minusBtn.Text = "-"
minusBtn.Font = Enum.Font.SourceSansBold
minusBtn.TextSize = 20
minusBtn.BackgroundColor3 = Color3.fromRGB(28,28,28)
minusBtn.TextColor3 = Color3.new(1,1,1)
minusBtn.BorderSizePixel = 0

local plusBtn = Instance.new("TextButton", distFrame)
plusBtn.Size = UDim2.new(0, 64, 0, 28)
plusBtn.Position = UDim2.new(1, -48, 0, 8)
plusBtn.Text = "+"
plusBtn.Font = Enum.Font.SourceSansBold
plusBtn.TextSize = 20
plusBtn.BackgroundColor3 = Color3.fromRGB(28,28,28)
plusBtn.TextColor3 = Color3.new(1,1,1)
plusBtn.BorderSizePixel = 0

minusBtn.MouseButton1Click:Connect(function()
    AdvancedESP.MaxDistance = math.max(100, AdvancedESP.MaxDistance - 100)
    distLabel.Text = "MaxDistance: " .. tostring(AdvancedESP.MaxDistance)
    tweenObject(minusBtn, {BackgroundColor3 = Color3.fromRGB(200,200,200)}, TweenInfo.new(0.12))
    delay(0.12, function() tweenObject(minusBtn, {BackgroundColor3 = Color3.fromRGB(28,28,28)}, TweenInfo.new(0.12)) end)
end)
plusBtn.MouseButton1Click:Connect(function()
    AdvancedESP.MaxDistance = math.min(10000, AdvancedESP.MaxDistance + 100)
    distLabel.Text = "MaxDistance: " .. tostring(AdvancedESP.MaxDistance)
    tweenObject(plusBtn, {BackgroundColor3 = Color3.fromRGB(200,200,200)}, TweenInfo.new(0.12))
    delay(0.12, function() tweenObject(plusBtn, {BackgroundColor3 = Color3.fromRGB(28,28,28)}, TweenInfo.new(0.12)) end)
end)

-- Menu open/close
local menuOpen = false
local function setMenuOpen(open)
    if open == menuOpen then return end
    menuOpen = open
    if open then
        mainFrame.Visible = true
        tweenObject(mainFrame, {Size = targetSize}, TweenInfo.new(0.28, Enum.EasingStyle.Quad, Enum.EasingDirection.Out))
        tweenObject(title, {TextTransparency = 0}, tweenMed)
        tweenObject(subtitle, {TextTransparency = 0}, tweenMed)
    else
        local t = TweenService:Create(mainFrame, TweenInfo.new(0.22, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Size = closedSize})
        t:Play()
        t.Completed:Connect(function() mainFrame.Visible = false end)
    end
end

-- Keybinds
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.P then
            setMenuOpen(not menuOpen)
        elseif input.KeyCode == Enum.KeyCode.Escape then
            setMenuOpen(false)
        end
    end
end)

-- Keep button visuals synced
spawn(function()
    while true do
        for _, t in ipairs(toggles) do
            local b = buttons[t.key]
            if b then
                local state = AdvancedESP[t.key]
                local indicator = b:FindFirstChild("Indicator")
                local dot = indicator and indicator:FindFirstChild("Dot")
                if state then
                    b.BackgroundColor3 = Color3.fromRGB(230,230,230)
                    b.TextColor3 = Color3.fromRGB(8,8,8)
                    if indicator then indicator.BackgroundColor3 = Color3.fromRGB(0,170,70) end
                    if dot then dot.Position = UDim2.new(1, -24, 0.5, -9) end
                else
                    b.BackgroundColor3 = Color3.fromRGB(28,28,28)
                    b.TextColor3 = Color3.new(1,1,1)
                    if indicator then indicator.BackgroundColor3 = Color3.fromRGB(80,80,80) end
                    if dot then dot.Position = UDim2.new(0, 6, 0.5, -9) end
                end
            end
        end
        distLabel.Text = "MaxDistance: " .. tostring(AdvancedESP.MaxDistance)
        task.wait(0.4)
    end
end)

-- Make GUI draggable by titleBar
do
    local dragging = false
    local dragInput, dragStart, startPos

    local function update(input)
        local delta = input.Position - dragStart
        mainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end

    titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = mainFrame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    titleBar.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and input == dragInput then
            update(input)
        end
    end)
end

print("GLORIUS GANG HUB - ESP (Skeleton + Movable GUI) loaded. Press 'P' to open the menu. Drawing available:", UseDrawing)
