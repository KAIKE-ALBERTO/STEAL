local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

-- estado global
local noclipAtivo = false
local walkSpeedDesejado = 16
local anchorCFrame = nil
local autoReturn = true
local janelaGrab = 0

-- ============================================================
-- NOCLIP — hook no __index do metatable do game
-- ============================================================
local mt = getrawmetatable(game)
local oldIndex = mt.__index
setreadonly(mt, false)

mt.__index = newcclosure(function(self, key)
    if noclipAtivo and key == "CanCollide" and self:IsA("BasePart") then
        local char = LocalPlayer.Character
        if char and self:IsDescendantOf(char) then
            return false
        end
    end
    return oldIndex(self, key)
end)

setreadonly(mt, true)

-- ============================================================
-- SPEED
-- ============================================================
local function aplicarSpeed()
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then hum.WalkSpeed = walkSpeedDesejado end
end

-- ============================================================
-- ANCHOR / TP
-- ============================================================
local function getHRP()
    local char = LocalPlayer.Character
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart")
end

local function salvarAnchor()
    local hrp = getHRP()
    if hrp then
        anchorCFrame = hrp.CFrame
        return true
    end
    return false
end

local function voltarParaAnchor()
    local hrp = getHRP()
    if not (hrp and anchorCFrame) then return end
    hrp.CFrame = anchorCFrame
    hrp.AssemblyLinearVelocity = Vector3.zero
    hrp.AssemblyAngularVelocity = Vector3.zero
    for _, v in ipairs(hrp:GetChildren()) do
        if v:IsA("BodyVelocity") or v:IsA("LinearVelocity") then
            v:Destroy()
        end
    end
end

-- ============================================================
-- AUTO-RETURN — monitora Tool entrando no Backpack/Character
-- ============================================================
local function hookContainer(container)
    container.ChildAdded:Connect(function(child)
        if not autoReturn or not anchorCFrame then return end
        if not child:IsA("Tool") then return end
        local agora = tick()
        if agora - janelaGrab < 0.15 then return end
        janelaGrab = agora
        task.wait(0.06)
        voltarParaAnchor()
    end)
end

local function ligarMonitoramento()
    if LocalPlayer:FindFirstChild("Backpack") then
        hookContainer(LocalPlayer.Backpack)
    end
    LocalPlayer.ChildAdded:Connect(function(child)
        if child.Name == "Backpack" then hookContainer(child) end
    end)
    if LocalPlayer.Character then
        hookContainer(LocalPlayer.Character)
    end
    LocalPlayer.CharacterAdded:Connect(function(char)
        task.wait(0.5)
        aplicarSpeed()
        hookContainer(char)
    end)
end
ligarMonitoramento()

-- ============================================================
-- GUI
-- ============================================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "XenoAllInOne"
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local main = Instance.new("Frame")
main.Size = UDim2.new(0, 280, 0, 300)
main.Position = UDim2.new(0.5, -140, 0.5, -150)
main.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
main.BorderSizePixel = 0
main.Active = true
main.Draggable = true
main.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 8)
corner.Parent = main

local titulo = Instance.new("TextLabel")
titulo.Size = UDim2.new(1, 0, 0, 32)
titulo.BackgroundColor3 = Color3.fromRGB(30, 30, 38)
titulo.BorderSizePixel = 0
titulo.Text = "Anchor + Noclip + Speed"
titulo.TextColor3 = Color3.fromRGB(255, 255, 255)
titulo.TextSize = 16
titulo.Font = Enum.Font.GothamBold
titulo.Parent = main

local tituloCorner = Instance.new("UICorner")
tituloCorner.CornerRadius = UDim.new(0, 8)
tituloCorner.Parent = titulo

local function criarBotao(texto, y, corFundo, corTexto)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.85, 0, 0, 34)
    btn.Position = UDim2.new(0.075, 0, 0, y)
    btn.BackgroundColor3 = corFundo
    btn.Text = texto
    btn.TextColor3 = corTexto
    btn.TextSize = 13
    btn.Font = Enum.Font.Gotham
    btn.BorderSizePixel = 0
    btn.Parent = main
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 6)
    c.Parent = btn
    return btn
end

local noclipBtn = criarBotao("Noclip: OFF", 42, Color3.fromRGB(45, 45, 55), Color3.fromRGB(200, 200, 200))
noclipBtn.MouseButton1Click:Connect(function()
    noclipAtivo = not noclipAtivo
    noclipBtn.Text = noclipAtivo and "Noclip: ON" or "Noclip: OFF"
    noclipBtn.BackgroundColor3 = noclipAtivo and Color3.fromRGB(0, 140, 80) or Color3.fromRGB(45, 45, 55)
end)

local salvarBtn = criarBotao("Definir Posição (P)", 82, Color3.fromRGB(45, 55, 80), Color3.fromRGB(180, 210, 255))
local anchorLabel = Instance.new("TextLabel")
anchorLabel.Size = UDim2.new(0.85, 0, 0, 18)
anchorLabel.Position = UDim2.new(0.075, 0, 0, 118)
anchorLabel.BackgroundTransparency = 1
anchorLabel.Text = "Âncora: nenhuma"
anchorLabel.TextColor3 = Color3.fromRGB(150, 150, 160)
anchorLabel.TextSize = 11
anchorLabel.Font = Enum.Font.Gotham
anchorLabel.TextXAlignment = Enum.TextXAlignment.Left
anchorLabel.Parent = main

local function atualizarLabelAnchor()
    anchorLabel.Text = string.format("Âncora: %.0f, %.0f, %.0f",
        anchorCFrame.Position.X, anchorCFrame.Position.Y, anchorCFrame.Position.Z)
    anchorLabel.TextColor3 = Color3.fromRGB(0, 200, 120)
end

salvarBtn.MouseButton1Click:Connect(function()
    if salvarAnchor() then atualizarLabelAnchor() end
end)

local tpBtn = criarBotao("TP p/ Âncora (T)", 140, Color3.fromRGB(60, 40, 60), Color3.fromRGB(255, 200, 255))
tpBtn.MouseButton1Click:Connect(voltarParaAnchor)

local autoBtn = criarBotao("Auto-Return: ON", 180, Color3.fromRGB(0, 140, 80), Color3.fromRGB(255, 255, 255))
autoBtn.MouseButton1Click:Connect(function()
    autoReturn = not autoReturn
    autoBtn.Text = autoReturn and "Auto-Return: ON" or "Auto-Return: OFF"
    autoBtn.BackgroundColor3 = autoReturn and Color3.fromRGB(0, 140, 80) or Color3.fromRGB(45, 45, 55)
end)

local speedLabel = Instance.new("TextLabel")
speedLabel.Size = UDim2.new(0.85, 0, 0, 20)
speedLabel.Position = UDim2.new(0.075, 0, 0, 222)
speedLabel.BackgroundTransparency = 1
speedLabel.Text = "WalkSpeed: 16"
speedLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
speedLabel.TextSize = 13
speedLabel.Font = Enum.Font.Gotham
speedLabel.TextXAlignment = Enum.TextXAlignment.Left
speedLabel.Parent = main

local sliderTrack = Instance.new("Frame")
sliderTrack.Size = UDim2.new(0.85, 0, 0, 8)
sliderTrack.Position = UDim2.new(0.075, 0, 0, 248)
sliderTrack.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
sliderTrack.BorderSizePixel = 0
sliderTrack.Parent = main

local sliderCorner = Instance.new("UICorner")
sliderCorner.CornerRadius = UDim.new(0, 4)
sliderCorner.Parent = sliderTrack

local sliderFill = Instance.new("Frame")
sliderFill.Size = UDim2.new(0, 0, 1, 0)
sliderFill.BackgroundColor3 = Color3.fromRGB(0, 140, 80)
sliderFill.BorderSizePixel = 0
sliderFill.Parent = sliderTrack

local fillCorner = Instance.new("UICorner")
fillCorner.CornerRadius = UDim.new(0, 4)
fillCorner.Parent = sliderFill

local sliderHandle = Instance.new("TextButton")
sliderHandle.Size = UDim2.new(0, 16, 0, 16)
sliderHandle.Position = UDim2.new(0, -8, 0.5, -8)
sliderHandle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
sliderHandle.Text = ""
sliderHandle.BorderSizePixel = 0
sliderHandle.Parent = sliderTrack

local handleCorner = Instance.new("UICorner")
handleCorner.CornerRadius = UDim.new(1, 0)
handleCorner.Parent = sliderHandle

local MIN_SPEED, MAX_SPEED = 16, 200
local arrastando = false

local function setSpeedFromX(x)
    local relative = math.clamp((x - sliderTrack.AbsolutePosition.X) / sliderTrack.AbsoluteSize.X, 0, 1)
    walkSpeedDesejado = math.floor(MIN_SPEED + (MAX_SPEED - MIN_SPEED) * relative)
    sliderFill.Size = UDim2.new(relative, 0, 1, 0)
    sliderHandle.Position = UDim2.new(relative, -8, 0.5, -8)
    speedLabel.Text = "WalkSpeed: " .. walkSpeedDesejado
    aplicarSpeed()
end

sliderHandle.MouseButton1Down:Connect(function() arrastando = true end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then arrastando = false end
end)
UserInputService.InputChanged:Connect(function(input)
    if arrastando and input.UserInputType == Enum.UserInputType.MouseMovement then
        setSpeedFromX(input.Position.X)
    end
end)
sliderTrack.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        setSpeedFromX(input.Position.X)
    end
end)

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.P then
        if salvarAnchor() then atualizarLabelAnchor() end
    elseif input.KeyCode == Enum.KeyCode.T then
        voltarParaAnchor()
    elseif input.KeyCode == Enum.KeyCode.RightShift then
        main.Visible = not main.Visible
    end
end)

aplicarSpeed()
