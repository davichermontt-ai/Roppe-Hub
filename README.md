-- Roppe Hub - Hub fixo, botão movível e suave (PC/celular), funcionalidades completas

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local LocalPlayer = Players.LocalPlayer

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "Roppe Hub"
ScreenGui.IgnoreGuiInset = true
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = game:GetService("CoreGui")

local function lerp(a, b, t) return a + (b - a) * t end
local function lerpUDim2(a, b, t)
    return UDim2.new(
        lerp(a.X.Scale, b.X.Scale, t),
        lerp(a.X.Offset, b.X.Offset, t),
        lerp(a.Y.Scale, b.Y.Scale, t),
        lerp(a.Y.Offset, b.Y.Offset, t)
    )
end
local function getViewportSize()
    return Workspace.CurrentCamera and Workspace.CurrentCamera.ViewportSize or Vector2.new(800,600)
end

-- Centralização dinâmica do Hub
local HUB_WIDTH, HUB_HEIGHT = 260, 240
local function centerHub()
    local vp = getViewportSize()
    return UDim2.new(0, math.floor((vp.X - HUB_WIDTH)/2), 0, math.floor((vp.Y - HUB_HEIGHT)/2))
end

-- HUB FRAME (fixo)
local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0, HUB_WIDTH, 0, HUB_HEIGHT)
Frame.Position = centerHub()
Frame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Visible = true
Frame.Parent = ScreenGui
Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 16)
local UIStroke = Instance.new("UIStroke", Frame)
UIStroke.Color = Color3.fromRGB(255,255,0)
UIStroke.Thickness = 2

-- Título do Hub
local TitleBar = Instance.new("Frame")
TitleBar.Size = UDim2.new(1, 0, 0, 32)
TitleBar.BackgroundTransparency = 1
TitleBar.Parent = Frame
TitleBar.Name = "TitleBar"
TitleBar.Active = false

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 1, 0)
Title.BackgroundTransparency = 1
Title.Text = "Roppe Hub"
Title.Font = Enum.Font.GothamBold
Title.TextColor3 = Color3.fromRGB(255,255,0)
Title.TextSize = 18
Title.Parent = TitleBar

-- BOTÃO FORA DO HUB (movível e suave)
local ToggleBtn = Instance.new("ImageButton")
ToggleBtn.Name = "RoppeHubToggleBtn"
ToggleBtn.Size = UDim2.new(0, 42, 0, 42)
ToggleBtn.Position = UDim2.new(1, -54, 0, 12)
ToggleBtn.AnchorPoint = Vector2.new(0, 0)
ToggleBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
ToggleBtn.BackgroundTransparency = 0
ToggleBtn.Parent = ScreenGui
ToggleBtn.Image = "rbxassetid://104526885174787"
ToggleBtn.AutoButtonColor = true
Instance.new("UICorner", ToggleBtn).CornerRadius = UDim.new(1,0)
local BtnUIStroke = Instance.new("UIStroke", ToggleBtn)
BtnUIStroke.Color = Color3.fromRGB(255,255,0)
BtnUIStroke.Thickness = 2

-- Drag do Botão (PC e CELULAR) - drag contínuo e suave
local btnDragging, btnDragInput, btnDragStart, btnStartPos
local btnTargetPos = ToggleBtn.Position

local function getInputDelta(input)
    local vp = getViewportSize()
    local delta = input.Position - btnDragStart
    local x = math.clamp(btnStartPos.X.Offset + delta.X, 0, vp.X - ToggleBtn.Size.X.Offset)
    local y = math.clamp(btnStartPos.Y.Offset + delta.Y, 0, vp.Y - ToggleBtn.Size.Y.Offset)
    return UDim2.new(0, x, 0, y)
end

ToggleBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        btnDragging = true
        btnDragInput = input
        btnDragStart = input.Position
        btnStartPos = ToggleBtn.Position

        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                btnDragging = false
                btnDragInput = nil
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if btnDragging and input == btnDragInput then
        btnTargetPos = getInputDelta(input)
    end
end)

RunService.RenderStepped:Connect(function()
    if ToggleBtn.Position ~= btnTargetPos then
        ToggleBtn.Position = lerpUDim2(ToggleBtn.Position, btnTargetPos, 0.24)
    end
end)

-- Centraliza Hub ao abrir e ao mudar viewport
local function centralizeIfVisible()
    if Frame.Visible then
        local pos = centerHub()
        Frame.Position = pos
    end
end
centralizeIfVisible()
Workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(centralizeIfVisible)
UserInputService:GetPropertyChangedSignal("TouchEnabled"):Connect(centralizeIfVisible)

-- Botão abre/fecha o Hub
ToggleBtn.MouseButton1Click:Connect(function()
    Frame.Visible = not Frame.Visible
    if Frame.Visible then
        centralizeIfVisible()
    end
end)

---------------------------------------------------------------
-- Rope Hub funcionalidades originais (Highlight, Speed etc) --
---------------------------------------------------------------

-- Highlight humanoids (F)
local highlightsEnabled = false
local highlightColor = Color3.fromRGB(255, 255, 0)
local highlightTable = {}
local highlightUpdateConn = nil
local function getHumanoidModels()
    local list = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player.Character and player.Character:FindFirstChildWhichIsA("Humanoid") then
            table.insert(list, player.Character)
        end
    end
    for _, model in ipairs(Workspace:GetDescendants()) do
        if model:IsA("Model") and model:FindFirstChildWhichIsA("Humanoid") then
            if not Players:GetPlayerFromCharacter(model) then
                table.insert(list, model)
            end
        end
    end
    return list
end
local function removeHighlights()
    for _, hl in pairs(highlightTable) do
        if hl and hl.Parent then hl:Destroy() end
    end
    highlightTable = {}
end
local function updateHighlights()
    for model, hl in pairs(highlightTable) do
        if not model.Parent then
            if hl and hl.Parent then hl:Destroy() end
            highlightTable[model] = nil
        end
    end
    for _, model in ipairs(getHumanoidModels()) do
        if not highlightTable[model] then
            local highlight = Instance.new("Highlight")
            highlight.Name = "_RoppeHubHighlight"
            highlight.Adornee = model
            highlight.FillColor = highlightColor
            highlight.OutlineColor = Color3.new(0,0,0)
            highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            highlight.Parent = model
            highlightTable[model] = highlight
        else
            highlightTable[model].FillColor = highlightColor
        end
        highlightTable[model].Enabled = true
    end
    for model, hl in pairs(highlightTable) do
        if not model:FindFirstChildWhichIsA("Humanoid") then
            if hl and hl.Parent then hl:Destroy() end
            highlightTable[model] = nil
        end
    end
end
local function startHighlightLoop()
    if highlightUpdateConn then highlightUpdateConn:Disconnect() end
    highlightUpdateConn = RunService.RenderStepped:Connect(function()
        if highlightsEnabled then updateHighlights() else removeHighlights() end
    end)
end

local ToggleHighlight = Instance.new("TextButton")
ToggleHighlight.Size = UDim2.new(0, 220, 0, 28)
ToggleHighlight.Position = UDim2.new(0, 20, 0, 44)
ToggleHighlight.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
ToggleHighlight.Text = "Enable Highlights [F]"
ToggleHighlight.Font = Enum.Font.Gotham
ToggleHighlight.TextColor3 = Color3.fromRGB(255,255,255)
ToggleHighlight.TextSize = 15
ToggleHighlight.Parent = Frame

local ColorBox = Instance.new("TextBox")
ColorBox.Size = UDim2.new(0, 220, 0, 22)
ColorBox.Position = UDim2.new(0, 20, 0, 76)
ColorBox.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
ColorBox.Text = "Color (R,G,B) ex: 255,111,40"
ColorBox.Font = Enum.Font.Gotham
ColorBox.TextColor3 = Color3.fromRGB(200,200,200)
ColorBox.TextSize = 13
ColorBox.ClearTextOnFocus = true
ColorBox.Parent = Frame

ToggleHighlight.MouseButton1Click:Connect(function()
    highlightsEnabled = not highlightsEnabled
    ToggleHighlight.Text = (highlightsEnabled and "Disable Highlights [F]") or "Enable Highlights [F]"
    if highlightsEnabled then
        startHighlightLoop()
    else
        removeHighlights()
        if highlightUpdateConn then highlightUpdateConn:Disconnect() end
    end
end)
ColorBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then
        local r,g,b = string.match(ColorBox.Text, "(%d+)%s*,%s*(%d+)%s*,%s*(%d+)")
        if r and g and b then
            highlightColor = Color3.fromRGB(tonumber(r), tonumber(g), tonumber(b))
            for _, hl in pairs(highlightTable) do
                if hl then hl.FillColor = highlightColor end
            end
        else
            ColorBox.Text = "Invalid! Use R,G,B"
        end
    end
end)

-- Speed
local ToggleSpeed = Instance.new("TextButton")
ToggleSpeed.Size = UDim2.new(0, 220, 0, 28)
ToggleSpeed.Position = UDim2.new(0, 20, 0, 106)
ToggleSpeed.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
ToggleSpeed.Text = "Enable Speed [G]"
ToggleSpeed.Font = Enum.Font.Gotham
ToggleSpeed.TextColor3 = Color3.fromRGB(255,255,255)
ToggleSpeed.TextSize = 15
ToggleSpeed.Parent = Frame

local SpeedLabel = Instance.new("TextLabel")
SpeedLabel.Size = UDim2.new(0, 80, 0, 24)
SpeedLabel.Position = UDim2.new(0, 20, 0, 138)
SpeedLabel.BackgroundTransparency = 1
SpeedLabel.Text = "Speed:"
SpeedLabel.Font = Enum.Font.Gotham
SpeedLabel.TextColor3 = Color3.fromRGB(200,200,0)
SpeedLabel.TextSize = 13
SpeedLabel.TextXAlignment = Enum.TextXAlignment.Left
SpeedLabel.Parent = Frame

local SpeedBox = Instance.new("TextBox")
SpeedBox.Size = UDim2.new(0, 50, 0, 22)
SpeedBox.Position = UDim2.new(0, 65, 0, 139)
SpeedBox.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
SpeedBox.Text = "28"
SpeedBox.Font = Enum.Font.Gotham
SpeedBox.TextColor3 = Color3.fromRGB(200,200,200)
SpeedBox.TextSize = 13
SpeedBox.ClearTextOnFocus = true
SpeedBox.Parent = Frame

local DefaultSpeedLabel = Instance.new("TextLabel")
DefaultSpeedLabel.Size = UDim2.new(0, 120, 0, 24)
DefaultSpeedLabel.Position = UDim2.new(0, 120, 0, 138)
DefaultSpeedLabel.BackgroundTransparency = 1
DefaultSpeedLabel.Text = "(Default: 16 - Max seguro: 30~35)"
DefaultSpeedLabel.Font = Enum.Font.Gotham
DefaultSpeedLabel.TextColor3 = Color3.fromRGB(200,200,0)
DefaultSpeedLabel.TextSize = 12
DefaultSpeedLabel.TextXAlignment = Enum.TextXAlignment.Left
DefaultSpeedLabel.Parent = Frame

local speedEnabled = false
local desiredSpeed = 28
local defaultSpeed = 16
local speedLoopConn = nil
local function getHumanoid()
    local char = LocalPlayer.Character
    if char then return char:FindFirstChildWhichIsA("Humanoid") end
    return nil
end
local function setSpeed(speed)
    local hum = getHumanoid()
    if hum then hum.WalkSpeed = speed end
end
local function maintainSpeed()
    if speedLoopConn then speedLoopConn:Disconnect() end
    speedLoopConn = RunService.RenderStepped:Connect(function()
        if not speedEnabled then
            setSpeed(defaultSpeed)
            if speedLoopConn then speedLoopConn:Disconnect() speedLoopConn = nil end
            return
        end
        setSpeed(desiredSpeed)
    end)
end
local function applySpeed()
    if speedEnabled then maintainSpeed() else setSpeed(defaultSpeed) if speedLoopConn then speedLoopConn:Disconnect() speedLoopConn = nil end end
end
ToggleSpeed.MouseButton1Click:Connect(function()
    speedEnabled = not speedEnabled
    ToggleSpeed.Text = (speedEnabled and "Disable Speed [G]") or "Enable Speed [G]"
    applySpeed()
end)
SpeedBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then
        local num = tonumber(SpeedBox.Text)
        if num and num > 0 then
            desiredSpeed = num
            if speedEnabled then setSpeed(desiredSpeed) end
        else
            SpeedBox.Text = tostring(desiredSpeed)
        end
    end
end)

-- Plataforma
local TogglePlatform = Instance.new("TextButton")
TogglePlatform.Size = UDim2.new(0, 220, 0, 28)
TogglePlatform.Position = UDim2.new(0, 20, 0, 171)
TogglePlatform.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
TogglePlatform.Text = "Enable Rising Platform [H]"
TogglePlatform.Font = Enum.Font.Gotham
TogglePlatform.TextColor3 = Color3.fromRGB(255,255,255)
TogglePlatform.TextSize = 15
TogglePlatform.Parent = Frame

local PlatformLabel = Instance.new("TextLabel")
PlatformLabel.Size = UDim2.new(1, -40, 0, 20)
PlatformLabel.Position = UDim2.new(0, 20, 0, 201)
PlatformLabel.BackgroundTransparency = 1
PlatformLabel.Text = "Atalho: H. Plataforma sobe se andar para frente."
PlatformLabel.Font = Enum.Font.Gotham
PlatformLabel.TextColor3 = Color3.fromRGB(255,255,0)
PlatformLabel.TextSize = 12
PlatformLabel.TextXAlignment = Enum.TextXAlignment.Left
PlatformLabel.Parent = Frame

local platformEnabled = false
local platformPart = nil
local platformConnection = nil
local platformHeightOffset = 0.1
local platformSize = Vector3.new(6, 1, 6)
local platformColor = Color3.fromRGB(170, 90, 255)
local risingSpeed = 2
local function removePlatform()
    if platformConnection then platformConnection:Disconnect() platformConnection = nil end
    if platformPart then platformPart:Destroy() platformPart = nil end
end
local function createPlatform()
    removePlatform()
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    platformPart = Instance.new("Part")
    platformPart.Size = platformSize
    platformPart.Anchored = true
    platformPart.CanCollide = true
    platformPart.Color = platformColor
    platformPart.Transparency = 0.1
    platformPart.Material = Enum.Material.Neon
    platformPart.Name = "__RoppeHubPlatform"
    platformPart.Parent = workspace
    platformConnection = RunService.RenderStepped:Connect(function(dt)
        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        local humanoid = char and char:FindFirstChildWhichIsA("Humanoid")
        if not hrp or not humanoid then return end
        local baseY = hrp.Position.Y - (hrp.Size.Y/2) - humanoid.HipHeight - platformHeightOffset - (platformPart.Size.Y/2)
        platformPart.Position = Vector3.new(hrp.Position.X, baseY, hrp.Position.Z)
        if humanoid.MoveDirection.Z < -0.1 then
            platformPart.Position = platformPart.Position + Vector3.new(0, risingSpeed * dt, 0)
        end
        if platformPart and (platformPart.Parent ~= workspace) then
            platformPart.Parent = workspace
        end
    end)
end
local function enablePlatform() createPlatform() end
local function disablePlatform() removePlatform() end
TogglePlatform.MouseButton1Click:Connect(function()
    platformEnabled = not platformEnabled
    TogglePlatform.Text = (platformEnabled and "Disable Rising Platform [H]") or "Enable Rising Platform [H]"
    if platformEnabled then enablePlatform() else disablePlatform() end
end)

-- Atalhos teclado
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.F then
        highlightsEnabled = not highlightsEnabled
        ToggleHighlight.Text = (highlightsEnabled and "Disable Highlights [F]") or "Enable Highlights [F]"
        if highlightsEnabled then startHighlightLoop() else removeHighlights() if highlightUpdateConn then highlightUpdateConn:Disconnect() end end
    elseif input.KeyCode == Enum.KeyCode.G then
        speedEnabled = not speedEnabled
        ToggleSpeed.Text = (speedEnabled and "Disable Speed [G]") or "Enable Speed [G]"
        applySpeed()
    elseif input.KeyCode == Enum.KeyCode.H then
        platformEnabled = not platformEnabled
        TogglePlatform.Text = (platformEnabled and "Disable Rising Platform [H]") or "Enable Rising Platform [H]"
        if platformEnabled then enablePlatform() else disablePlatform() end
    end
end)

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1)
    if highlightsEnabled then startHighlightLoop() end
    applySpeed()
    if platformEnabled then enablePlatform() end
end)

if LocalPlayer.Character then
    if highlightsEnabled then startHighlightLoop() end
    applySpeed()
    if platformEnabled then enablePlatform() end
end
