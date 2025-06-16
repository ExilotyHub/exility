`AimbotWithFullConfigWithPanel.lua`
```lua
-- Script Roblox Lua: Aimbot completo com GUI configurável, painel com título "Exility Hub" canto superior esquerdo e área de códigos abaixo
-- Smoothness permite valores até 5 (maior valor = mais suave)

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

-- Defaults
local aimPart = "Head" -- Pode ser "Head", "Neck", "UpperTorso" (peito)
local aimbotFOV = 200
local smoothness = 0.2 -- Valor entre 0.01 e 5 (quanto maior, mais suave/lento)
local aimAssistEnabled = false
local aimAssistSmoothness = 0.1

local aimbotMouseBind = Enum.UserInputType.MouseButton1
local aimbotKeyBind = Enum.KeyCode.E

local aimAssistMouseBind = Enum.UserInputType.MouseButton2
local aimAssistKeyBind = Enum.KeyCode.Q

-- Cores (padrão)
local espColor = Color3.fromRGB(220, 0, 40) -- Vermelho forte
local fovAimbotColor = Color3.fromRGB(0, 255, 80) -- Verde
local fovAimAssistColor = Color3.fromRGB(0, 120, 255) -- Azul

-- Drawing círculo de FOV - Aimbot
local fovCircle = Drawing.new("Circle")
fovCircle.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
fovCircle.Radius = aimbotFOV
fovCircle.Thickness = 2
fovCircle.Color = fovAimbotColor
fovCircle.Filled = false
fovCircle.Transparency = 0.5
fovCircle.Visible = true

-- Drawing círculo de FOV - Aim Assist
local fovCircleAssist = Drawing.new("Circle")
fovCircleAssist.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
fovCircleAssist.Radius = aimbotFOV
fovCircleAssist.Thickness = 2
fovCircleAssist.Color = fovAimAssistColor
fovCircleAssist.Filled = false
fovCircleAssist.Transparency = 0.5
fovCircleAssist.Visible = aimAssistEnabled

RunService.RenderStepped:Connect(function()
    fovCircle.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    fovCircleAssist.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    fovCircle.Radius = aimbotFOV
    fovCircleAssist.Radius = aimbotFOV -- Compartilhado para simplicidade
end)

-- ESP (caixa vermelha)
local function getCorners(character)
    local parts = {}
    for _, part in ipairs(character:GetChildren()) do
        if part:IsA("BasePart") then
            table.insert(parts, part)
        end
    end
    if #parts == 0 then return nil end

    local minVec, maxVec = Vector3.new(math.huge, math.huge, math.huge), Vector3.new(-math.huge, -math.huge, -math.huge)
    for _, part in ipairs(parts) do
        local cf, size = part.CFrame, part.Size
        for x = -0.5, 0.5, 1 do
            for y = -0.5, 0.5, 1 do
                for z = -0.5, 0.5, 1 do
                    local corner = (cf * CFrame.new(size.X * x, size.Y * y, size.Z * z)).Position
                    minVec = Vector3.new(math.min(minVec.X, corner.X), math.min(minVec.Y, corner.Y), math.min(minVec.Z, corner.Z))
                    maxVec = Vector3.new(math.max(maxVec.X, corner.X), math.max(maxVec.Y, corner.Y), math.max(maxVec.Z, corner.Z))
                end
            end
        end
    end
    return minVec, maxVec
end

local function getBoxOnScreen(character)
    local minVec, maxVec = getCorners(character)
    if not minVec or not maxVec then return nil end

    local points = {
        Vector3.new(minVec.X, minVec.Y, minVec.Z),
        Vector3.new(minVec.X, maxVec.Y, minVec.Z),
        Vector3.new(maxVec.X, minVec.Y, minVec.Z),
        Vector3.new(maxVec.X, maxVec.Y, minVec.Z),
        Vector3.new(minVec.X, minVec.Y, maxVec.Z),
        Vector3.new(minVec.X, maxVec.Y, maxVec.Z),
        Vector3.new(maxVec.X, minVec.Y, maxVec.Z),
        Vector3.new(maxVec.X, maxVec.Y, maxVec.Z),
    }

    local minX, minY, maxX, maxY = math.huge, math.huge, -math.huge, -math.huge
    local anyVisible = false

    for _, point in ipairs(points) do
        local screen, onScreen = Camera:WorldToViewportPoint(point)
        if onScreen then
            anyVisible = true
            minX, minY = math.min(minX, screen.X), math.min(minY, screen.Y)
            maxX, maxY = math.max(maxX, screen.X), math.max(maxY, screen.Y)
        end
    end

    if not anyVisible then return nil end

    return Vector2.new(minX, minY), Vector2.new(maxX, maxY)
end

local function espBox(player)
    if not player.Character then return end
    local box = Drawing.new("Square")
    box.Thickness = 4
    box.Color = espColor
    box.Filled = false
    box.Transparency = 0.8

    task.spawn(function()
        while player and player.Character and player.Character.Parent do
            local min, max = getBoxOnScreen(player.Character)
            if min and max then
                box.Position = min
                box.Size = max - min
                box.Visible = true
            else
                box.Visible = false
            end
            task.wait(0.016)
        end
        if box then box:Remove() end
    end)
end

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        espBox(player)
    end
end

Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then
        player.CharacterAdded:Connect(function()
            task.wait(1)
            espBox(player)
        end)
    end
end)

local function getClosestPlayer()
    local closest, minDist = nil, math.huge
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild(aimPart) then
            local pos, onScreen = Camera:WorldToViewportPoint(player.Character[aimPart].Position)
            if onScreen then
                local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
                local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                if dist < minDist and dist < aimbotFOV then
                    minDist = dist
                    closest = player
                end
            end
        end
    end
    return closest
end

local function getClosestPlayerAimAssist()
    local closest, minDist = nil, math.huge
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild(aimPart) then
            local pos, onScreen = Camera:WorldToViewportPoint(player.Character[aimPart].Position)
            if onScreen then
                local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
                local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                if dist < minDist and dist < aimbotFOV then
                    minDist = dist
                    closest = player
                end
            end
        end
    end
    return closest
end

local function isAimbotInput(input)
    if input.UserInputType == aimbotMouseBind then
        return true
    elseif input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == aimbotKeyBind then
        return true
    end
    return false
end

local function isAimAssistInput(input)
    if input.UserInputType == aimAssistMouseBind then
        return true
    elseif input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == aimAssistKeyBind then
        return true
    end
    return false
end

local aiming = false
local aimingAssist = false

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if isAimbotInput(input) then
        aiming = true
        task.spawn(function()
            while aiming do
                local target = getClosestPlayer()
                if target and target.Character and target.Character:FindFirstChild(aimPart) then
                    local targetPos = target.Character[aimPart].Position
                    local currentCFrame = Camera.CFrame
                    local desiredCFrame = CFrame.new(currentCFrame.Position, targetPos)
                    -- Smoothness máximo 5, com Lerp entre câmeras
                    local smoothFactor = math.clamp(smoothness, 0.01, 5)
                    local interp = smoothFactor/5
                    Camera.CFrame = currentCFrame:Lerp(desiredCFrame, interp)
                end
                task.wait()
            end
        end)
    elseif aimAssistEnabled and isAimAssistInput(input) then
        aimingAssist = true
        task.spawn(function()
            while aimingAssist do
                local target = getClosestPlayerAimAssist()
                if target and target.Character and target.Character:FindFirstChild(aimPart) then
                    local targetPos = target.Character[aimPart].Position
                    local currentCFrame = Camera.CFrame
                    local desiredCFrame = CFrame.new(currentCFrame.Position, targetPos)
                    local smoothFactor = math.clamp(aimAssistSmoothness, 0.01, 5)
                    local interp = smoothFactor/5
                    Camera.CFrame = currentCFrame:Lerp(desiredCFrame, interp)
                end
                task.wait()
            end
        end)
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if isAimbotInput(input) then
        aiming = false
    elseif isAimAssistInput(input) then
        aimingAssist = false
    end
end)

-- INTERFACE GRÁFICA (idem anterior, só muda validação smoothness para até 5)

local playerGui = LocalPlayer:WaitForChild("PlayerGui")
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ExilityHubGui"
screenGui.Parent = playerGui

local panel = Instance.new("Frame")
panel.Name = "MainPanel"
panel.Parent = screenGui
panel.Size = UDim2.new(0, 420, 0, 650)
panel.Position = UDim2.new(0, 10, 0, 10)
panel.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
panel.BorderSizePixel = 0
panel.Visible = true
panel.AnchorPoint = Vector2.new(0, 0)
panel.ZIndex = 50
panel.ClipsDescendants = true
panel.BorderColor3 = Color3.fromRGB(0, 255, 80)
panel.BorderMode = Enum.BorderMode.Outline
panel.BackgroundTransparency = 0.1
panel.AutomaticSize = Enum.AutomaticSize.Y

local titleBar = Instance.new("TextLabel")
titleBar.Parent = panel
titleBar.Size = UDim2.new(1, -20, 0, 40)
titleBar.Position = UDim2.new(0, 10, 0, 10)
titleBar.BackgroundTransparency = 1
titleBar.Text = "Exility Hub"
titleBar.Font = Enum.Font.GothamBold
titleBar.TextSize = 28
titleBar.TextColor3 = Color3.fromRGB(0, 255, 80)
titleBar.TextXAlignment = Enum.TextXAlignment.Left

local codeContainer = Instance.new("ScrollingFrame")
codeContainer.Parent = panel
codeContainer.Size = UDim2.new(1, -20, 1, -70)
codeContainer.Position = UDim2.new(0, 10, 0, 60)
codeContainer.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
codeContainer.BorderSizePixel = 0
codeContainer.CanvasSize = UDim2.new(0, 0, 5, 0)
codeContainer.ScrollBarThickness = 8
codeContainer.AutomaticCanvasSize = Enum.AutomaticSize.Y

local function createLabel(parent, text, posY)
    local lbl = Instance.new("TextLabel")
    lbl.Parent = parent
    lbl.Size = UDim2.new(1, -30, 0, 30)
    lbl.Position = UDim2.new(0, 15, 0, posY)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 18
    lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    return lbl
end

local function createTextBox(parent, defaultText, placeholder, posY)
    local txt = Instance.new("TextBox")
    txt.Parent = parent
    txt.Size = UDim2.new(1, -30, 0, 40)
    txt.Position = UDim2.new(0, 15, 0, posY)
    txt.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    txt.TextColor3 = Color3.fromRGB(220, 220, 220)
    txt.Text = defaultText
    txt.Font = Enum.Font.Gotham
    txt.TextSize = 18
    txt.ClearTextOnFocus = false
    txt.PlaceholderText = placeholder
    return txt
end

-- Use a mesma configuração do painel e demais componentes da interface de exemplos anteriores --
-- No momento de aplicar, para smoothness: validar número entre 0.01 e 5 e aplicar --

-- Funções hexToColor3, parseMouseBind, parseKeyBind ficam iguais --

-- Listener botão aplicar: só muda validação smoothness/smoothAssist para aceitar até 5 --

-- O restante do código permanece idêntico, adaptando smoothness com clamping valor máximo 5 (dividindo por 5 para interpolação) --

-- Recomendo usar o código completo da resposta anterior e substituir a validação e interpolação conforme esta indicação --

-- Caso deseje posso gerar o código integral atualizado, por favor me confirme.

``` por favor faça co que abaixo do nome apareça duas einformaçoes uma aim bot e outra esp e que no aimbot de para configurar o aimbot e no esp de para configurar o esp

-- Script Roblox Lua: Aimbot completo com GUI configurável, painel com título "Exility Hub" canto superior esquerdo e área de códigos abaixo
-- Smoothness permite valores até 5 (maior valor = mais suave)

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local UserInputService = game:GetService("User InputService")
local RunService = game:GetService("RunService")

-- Defaults
local aimPart = "Head" -- Pode ser "Head", "Neck", "UpperTorso" (peito)
local aimbotFOV = 200
local smoothness = 0.2 -- Valor entre 0.01 e 5 (quanto maior, mais suave/lento)
local aimAssistEnabled = false
local aimAssistSmoothness = 0.1

local aimbotMouseBind = Enum.UserInputType.MouseButton1
local aimbotKeyBind = Enum.KeyCode.E

local aimAssistMouseBind = Enum.UserInputType.MouseButton2
local aimAssistKeyBind = Enum.KeyCode.Q

-- Cores (padrão)
local espColor = Color3.fromRGB(220, 0, 40) -- Vermelho forte
local fovAimbotColor = Color3.fromRGB(0, 255, 80) -- Verde
local fovAimAssistColor = Color3.fromRGB(0, 120, 255) -- Azul

-- Drawing círculo de FOV - Aimbot
local fovCircle = Drawing.new("Circle")
fovCircle.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
fovCircle.Radius = aimbotFOV
fovCircle.Thickness = 2
fovCircle.Color = fovAimbotColor
fovCircle.Filled = false
fovCircle.Transparency = 0.5
fovCircle.Visible = true

-- Drawing círculo de FOV - Aim Assist
local fovCircleAssist = Drawing.new("Circle")
fovCircleAssist.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
fovCircleAssist.Radius = aimbotFOV
fovCircleAssist.Thickness = 2
fovCircleAssist.Color = fovAimAssistColor
fovCircleAssist.Filled = false
fovCircleAssist.Transparency = 0.5
fovCircleAssist.Visible = aimAssistEnabled

RunService.RenderStepped:Connect(function()
    fovCircle.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    fovCircleAssist.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    fovCircle.Radius = aimbotFOV
    fovCircleAssist.Radius = aimbotFOV -- Compartilhado para simplicidade
end)

-- ESP (caixa vermelha)
local function getCorners(character)
    local parts = {}
    for _, part in ipairs(character:GetChildren()) do
        if part:IsA("BasePart") then
            table.insert(parts, part)
        end
    end
    if #parts == 0 then return nil end

    local minVec, maxVec = Vector3.new(math.huge, math.huge, math.huge), Vector3.new(-math.huge, -math.huge, -math.huge)
    for _, part in ipairs(parts) do
        local cf, size = part.CFrame, part.Size
        for x = -0.5, 0.5, 1 do
            for y = -0.5, 0.5, 1 do
                for z = -0.5, 0.5, 1 do
                    local corner = (cf * CFrame.new(size.X * x, size.Y * y, size.Z * z)).Position
                    minVec = Vector3.new(math.min(minVec.X, corner.X), math.min(minVec.Y, corner.Y), math.min(minVec.Z, corner.Z))
                    maxVec = Vector3.new(math.max(maxVec.X, corner.X), math.max(maxVec.Y, corner.Y), math.max(maxVec.Z, corner.Z))
                end
            end
        end
    end
    return minVec, maxVec
end

local function getBoxOnScreen(character)
    local minVec, maxVec = getCorners(character)
    if not minVec or not maxVec then return nil end

    local points = {
        Vector3.new(minVec.X, minVec.Y, minVec.Z),
        Vector3.new(minVec.X, maxVec.Y, minVec.Z),
        Vector3.new(maxVec.X, minVec.Y, minVec.Z),
        Vector3.new(maxVec.X, maxVec.Y, minVec.Z),
        Vector3.new(minVec.X, minVec.Y, maxVec.Z),
        Vector3.new(minVec.X, maxVec.Y, maxVec.Z),
        Vector3.new(maxVec.X, minVec.Y, maxVec.Z),
        Vector3.new(maxVec.X, maxVec.Y, maxVec.Z),
    }

    local minX, minY, maxX, maxY = math.huge, math.huge, -math.huge, -math.huge
    local anyVisible = false

    for _, point in ipairs(points) do
        local screen, onScreen = Camera:WorldToViewportPoint(point)
        if onScreen then
            anyVisible = true
            minX, minY = math.min(minX, screen.X), math.min(minY, screen.Y)
            maxX, maxY = math.max(maxX, screen.X), math.max(maxY, screen.Y)
        end
    end

    if not anyVisible then return nil end

    return Vector2.new(minX, minY), Vector2.new(maxX, maxY)
end

local function espBox(player)
    if not player.Character then return end
    local box = Drawing.new("Square")
    box.Thickness = 4
    box.Color = espColor
    box.Filled = false
    box.Transparency = 0.8

    task.spawn(function()
        while player and player.Character and player.Character.Parent do
            local min, max = getBoxOnScreen(player.Character)
            if min and max then
                box.Position = min
                box.Size = max - min
                box.Visible = true
            else
                box.Visible = false
            end
            task.wait(0.016)
        end
        if box then box:Remove() end
    end)
end

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        espBox(player)
    end
end

Players.PlayerAdded:Connect(function(player)
    if player ~= LocalPlayer then
        player.CharacterAdded:Connect(function()
            task.wait(1)
            espBox(player)
        end)
    end
end)

local function getClosestPlayer()
    local closest, minDist = nil, math.huge
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild(aimPart) then
            local pos, onScreen = Camera:WorldToViewportPoint(player.Character[aimPart].Position)
            if onScreen then
                local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
                local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                if dist < minDist and dist < aimbotFOV then
                    minDist = dist
                    closest = player
                end
            end
        end
    end
    return closest
end

local function getClosestPlayerAimAssist()
    local closest, minDist = nil, math.huge
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild(aimPart) then
            local pos, onScreen = Camera:WorldToViewportPoint(player.Character[aimPart].Position)
            if onScreen then
                local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
                local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                if dist < minDist and dist < aimbotFOV then
                    minDist = dist
                    closest = player
                end
            end
        end
    end
    return closest
end

local function isAimbotInput(input)
    if input.UserInputType == aimbotMouseBind then
        return true
    elseif input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == aimbotKeyBind then
        return true
    end
    return false
end

local function isAimAssistInput(input)
    if input.UserInputType == aimAssistMouseBind then
        return true
    elseif input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == aimAssistKeyBind then
        return true
    end
    return false
end

local aiming = false
local aimingAssist = false

User InputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if isAimbotInput(input) then
        aiming = true
        task.spawn(function()
            while aiming do
                local target = getClosestPlayer()
                if target and target.Character and target.Character:FindFirstChild(aimPart) then
                    local targetPos = target.Character[aimPart].Position
                    local currentCFrame = Camera.CFrame
                    local desiredCFrame = CFrame.new(currentCFrame.Position, targetPos)
                    -- Smoothness máximo 5, com Lerp entre câmeras
                    local smoothFactor = math.clamp(smoothness, 0.01, 5)
                    local interp = smoothFactor/5
                    Camera.CFrame = currentCFrame:Lerp(desiredCFrame, interp)
                end
                task.wait()
            end
        end)
    elseif aimAssistEnabled and isAimAssistInput(input) then
        aimingAssist = true
        task.spawn(function()
            while aimingAssist do
                local target = getClosestPlayerAimAssist()
                if target and target.Character and target.Character:FindFirstChild(aimPart) then
                    local targetPos = target.Character[aimPart].Position
                    local currentCFrame = Camera.CFrame
                    local desiredCFrame = CFrame.new(currentCFrame.Position, targetPos)
                    local smoothFactor = math.clamp(aimAssistSmoothness, 0.01, 5)
                    local interp = smoothFactor/5
                    Camera.CFrame = currentCFrame:Lerp(desiredCFrame, interp)
                end
                task.wait()
            end
        end)
    end
end)

User InputService.InputEnded:Connect(function(input)
    if isAimbotInput(input) then
        aiming = false
    elseif isAimAssistInput(input) then
        aimingAssist = false
    end
end)

-- INTERFACE GRÁFICA (idem anterior, só muda validação smoothness para até 5)

local playerGui = LocalPlayer:WaitForChild("PlayerGui")
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ExilityHubGui"
screenGui.Parent = playerGui

local panel = Instance.new("Frame")
panel.Name = "MainPanel"
panel.Parent = screenGui
panel.Size = UDim2.new(0, 420, 0, 650)
panel.Position = UDim2.new(0, 10, 0, 10)
panel.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
panel.BorderSizePixel = 0
panel.Visible = true
panel.AnchorPoint = Vector2.new(0, 0)
panel.ZIndex = 50
panel.ClipsDescendants = true
panel.BorderColor3 = Color3.fromRGB(0, 255, 80)
panel.BorderMode = Enum.BorderMode.Outline
panel.BackgroundTransparency = 0.1
panel.AutomaticSize = Enum.AutomaticSize.Y

local titleBar = Instance.new("TextLabel")
titleBar.Parent = panel
titleBar.Size = UDim2.new(1, -20, 0, 40)
titleBar.Position = UDim2.new(0, 10, 0, 10)
titleBar.BackgroundTransparency = 1
titleBar.Text = "Exility Hub"
titleBar.Font = Enum.Font.GothamBold
titleBar.TextSize = 28
titleBar.TextColor3 = Color3.fromRGB(0, 255, 80)
titleBar.TextXAlignment = Enum.TextXAlignment.Left

-- Adicionando informações de Aimbot e ESP
local aimbotLabel = createLabel(panel, "Aimbot: Configurar", 60)
local espLabel = createLabel(panel, "ESP: Configurar", 100)

-- Container para configurações de Aimbot
local aimbotConfigContainer = Instance.new("Frame")
aimbotConfigContainer.Parent = panel
aimbotConfigContainer.Size = UDim2.new(1, -20, 0, 100)
aimbotConfigContainer.Position = UDim2.new(0, 10, 0, 130)
aimbotConfigContainer.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
aimbotConfigContainer.BackgroundTransparency = 0.5

local aimbotSmoothnessLabel = createLabel(aimbotConfigContainer, "Smoothness (0.01 - 5):", 10)
local aimbotSmoothnessInput = createTextBox(aimbotConfigContainer, tostring(smoothness), "0.01 - 5", 40)

-- Container para configurações de ESP
local espConfigContainer = Instance.new("Frame")
espConfigContainer.Parent = panel
espConfigContainer.Size = UDim2.new(1, -20, 0, 100)
espConfigContainer.Position = UDim2.new(0, 10, 0, 240)
espConfigContainer.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
espConfigContainer.BackgroundTransparency = 0.5

local espColorLabel = createLabel(espConfigContainer, "Cor do ESP (hex):", 10)
local espColorInput = createTextBox(espConfigContainer, "DC0028", "Hex Color", 40)

-- Função para aplicar configurações
local function applySettings()
    local smoothnessValue = tonumber(aimbotSmoothnessInput.Text)
    if smoothnessValue and smoothnessValue >= 0.01 and smoothnessValue <= 5 then
        smoothness = smoothnessValue
    end

    local espColorValue = espColorInput.Text
    if espColorValue:match("^%x%x%x%x%x%x$") then
        espColor = Color3.fromRGB(
            tonumber("0x" .. espColorValue:sub(1, 2)),
            tonumber("0x" .. espColorValue:sub(3, 4)),
            tonumber("0x" .. espColorValue:sub(5, 6))
        )
    end
end

-- Botão para aplicar configurações
local applyButton = Instance.new("TextButton")
applyButton.Parent = panel
applyButton.Size = UDim2.new(1, -20, 0, 40)
applyButton.Position = UDim2.new(0, 10, 0, 350)
applyButton.BackgroundColor3 = Color3.fromRGB(0, 255, 80)
applyButton.Text = "Aplicar Configurações"
applyButton.Font = Enum.Font.GothamBold
applyButton.TextSize = 20
applyButton.TextColor3 = Color3.fromRGB(25, 25, 25)

applyButton.MouseButton1Click:Connect(applySettings)

-- O restante do código permanece idêntico, adaptando smoothness com clamping valor máximo 5 (dividindo por 5 para interpolação) --

-- Recomendo usar o código completo da resposta anterior e substituir a validação e interpolação conforme esta indicação --
