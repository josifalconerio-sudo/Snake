--[[
    SNAKE HUB 🐍
    ROBLOX AIMBOT COMPLETO (Camera + Arma ADS)
    - Keybind PRINCIPAL: R (liga/desliga aimbot)
    - Sistema de KEY: "Essaeakey" (necessário para ativar)
    - Mira da câmera (suave)
    - Mira da arma quando apertar botão direito (ADS)
    - Identifica automaticamente armas e miras
    - UI completa com seleção de alvo
    - ESP com nome, vida e distância
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

-- ========== SISTEMA DE KEY ==========
local REQUIRED_KEY = "Essaeakey" -- Key necessária para ativar o hub
local keyInserted = false
local keyInputFrame = nil

-- ========== CONFIGURAÇÕES ==========
local settings = {
    AimbotEnabled = true,
    ESPEnabled = true,
    WeaponAimbotEnabled = true,
    TargetPart = "Head",
    TargetMode = "Auto",
    SelectedPlayer = nil,
    Smoothness = 0.25,
    WeaponSmoothness = 0.15,
    FOVRadius = 300,
    ShowFOVCircle = true,
    ESPBoxColor = Color3.fromRGB(0, 162, 255),
    ESPTextColor = Color3.fromRGB(255, 255, 255),
    ADSSensitivity = 0.8,
}

-- Lista de partes do corpo
local bodyParts = {"Head", "UpperTorso", "LowerTorso", "HumanoidRootPart"}

-- ========== VARIÁVEIS ==========
local currentTarget = nil
local espObjects = {}
local fovCircle = nil
local isAiming = false
local currentWeapon = nil
local weaponScope = nil
local originalMouseBehavior = nil
local hubLoaded = false

-- ========== TELA DE INSERIR KEY ==========
local function createKeyScreen()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SnakeHubKeySystem"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = LocalPlayer.PlayerGui
    
    -- Fundo escuro
    local background = Instance.new("Frame")
    background.Size = UDim2.new(1, 0, 1, 0)
    background.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    background.BackgroundTransparency = 0.7
    background.Parent = screenGui
    
    -- Frame principal
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 400, 0, 250)
    mainFrame.Position = UDim2.new(0.5, -200, 0.5, -125)
    mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
    mainFrame.BorderSizePixel = 2
    mainFrame.BorderColor3 = Color3.fromRGB(0, 162, 255)
    mainFrame.Parent = background
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = mainFrame
    
    -- Título Snake Hub
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 50)
    title.Position = UDim2.new(0, 0, 0, 10)
    title.Text = "SNAKE HUB 🐍"
    title.TextColor3 = Color3.fromRGB(0, 162, 255)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.TextSize = 28
    title.Parent = mainFrame
    
    -- By: Snake
    local subTitle = Instance.new("TextLabel")
    subTitle.Size = UDim2.new(1, 0, 0, 30)
    subTitle.Position = UDim2.new(0, 0, 0, 55)
    subTitle.Text = "By: Snake"
    subTitle.TextColor3 = Color3.fromRGB(180, 180, 200)
    subTitle.BackgroundTransparency = 1
    subTitle.Font = Enum.Font.Gotham
    subTitle.TextSize = 16
    subTitle.Parent = mainFrame
    
    -- Texto instrução
    local instruction = Instance.new("TextLabel")
    instruction.Size = UDim2.new(1, -40, 0, 30)
    instruction.Position = UDim2.new(0, 20, 0, 100)
    instruction.Text = "INSIRA A KEY PARA ATIVAR:"
    instruction.TextColor3 = Color3.fromRGB(200, 200, 200)
    instruction.BackgroundTransparency = 1
    instruction.Font = Enum.Font.Gotham
    instruction.TextSize = 14
    instruction.TextXAlignment = Enum.TextXAlignment.Left
    instruction.Parent = mainFrame
    
    -- Campo de texto para key
    local keyBox = Instance.new("TextBox")
    keyBox.Size = UDim2.new(1, -40, 0, 40)
    keyBox.Position = UDim2.new(0, 20, 0, 135)
    keyBox.PlaceholderText = "Digite a key aqui..."
    keyBox.Text = ""
    keyBox.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    keyBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    keyBox.Font = Enum.Font.Gotham
    keyBox.TextSize = 14
    keyBox.Parent = mainFrame
    
    local keyCorner = Instance.new("UICorner")
    keyCorner.CornerRadius = UDim.new(0, 5)
    keyCorner.Parent = keyBox
    
    -- Botão confirmar
    local confirmBtn = Instance.new("TextButton")
    confirmBtn.Size = UDim2.new(0, 150, 0, 40)
    confirmBtn.Position = UDim2.new(0.5, -75, 0, 190)
    confirmBtn.Text = "CONFIRMAR"
    confirmBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    confirmBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    confirmBtn.Font = Enum.Font.GothamBold
    confirmBtn.TextSize = 16
    confirmBtn.Parent = mainFrame
    
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 5)
    btnCorner.Parent = confirmBtn
    
    -- Mensagem de erro
    local errorMsg = Instance.new("TextLabel")
    errorMsg.Size = UDim2.new(1, -40, 0, 25)
    errorMsg.Position = UDim2.new(0, 20, 0, 235)
    errorMsg.Text = ""
    errorMsg.TextColor3 = Color3.fromRGB(255, 70, 70)
    errorMsg.BackgroundTransparency = 1
    errorMsg.Font = Enum.Font.Gotham
    errorMsg.TextSize = 12
    errorMsg.Visible = false
    errorMsg.Parent = mainFrame
    
    confirmBtn.MouseButton1Click:Connect(function()
        if keyBox.Text == REQUIRED_KEY then
            keyInserted = true
            screenGui:Destroy()
            loadHub()
        else
            errorMsg.Text = "KEY INCORRETA! Tente novamente."
            errorMsg.Visible = true
            keyBox.Text = ""
            wait(2)
            errorMsg.Visible = false
        end
    end)
    
    -- Permitir pressionar Enter
    keyBox.FocusLost:Connect(function(enterPressed)
        if enterPressed and keyBox.Text == REQUIRED_KEY then
            keyInserted = true
            screenGui:Destroy()
            loadHub()
        elseif enterPressed then
            errorMsg.Text = "KEY INCORRETA! Tente novamente."
            errorMsg.Visible = true
            keyBox.Text = ""
            wait(2)
            errorMsg.Visible = false
        end
    end)
end

-- ========== FUNÇÃO PARA IDENTIFICAR ARMA ATUAL ==========
local function getCurrentWeapon()
    local character = LocalPlayer.Character
    if not character then return nil end
    
    local tool = LocalPlayer.Character:FindFirstChildOfClass("Tool")
    if not tool then return nil end
    
    local isWeapon = false
    
    if tool:FindFirstChild("Handle") or tool:FindFirstChild("Grip") then
        isWeapon = true
    end
    
    for _, child in ipairs(tool:GetChildren()) do
        if child:IsA("Script") or child:IsA("LocalScript") then
            local scriptName = child.Name:lower()
            if scriptName:find("weapon") or scriptName:find("gun") or scriptName:find("fire") then
                isWeapon = true
                break
            end
        end
    end
    
    local weaponParts = {"Barrel", "Scope", "Sight", "Magazine", "Trigger"}
    for _, partName in ipairs(weaponParts) do
        if tool:FindFirstChild(partName) then
            isWeapon = true
            break
        end
    end
    
    if not isWeapon then return nil end
    
    local scope = nil
    for _, child in ipairs(tool:GetDescendants()) do
        if child:IsA("Part") or child:IsA("MeshPart") then
            local name = child.Name:lower()
            if name:find("scope") or name:find("sight") or name:find("aim") or name:find("lens") then
                scope = child
                break
            end
        end
        
        if child:IsA("Attachment") and (child.Name:lower():find("aim") or child.Name:lower():find("scope")) then
            scope = child
            break
        end
    end
    
    return {
        Tool = tool,
        WeaponType = nil,
        Scope = scope,
        HasScope = scope ~= nil
    }
end

-- ========== FUNÇÃO PARA OBTER A POSIÇÃO DE MIRA DA ARMA ==========
local function getWeaponAimPosition()
    if not currentWeapon or not currentWeapon.Tool then return nil end
    
    local tool = currentWeapon.Tool
    local character = LocalPlayer.Character
    if not character then return nil end
    
    local rightArm = character:FindFirstChild("Right Arm") or character:FindFirstChild("RightHand")
    if not rightArm then return nil end
    
    local aimPosition = nil
    
    if currentWeapon.Scope and currentWeapon.Scope:IsA("Attachment") then
        aimPosition = currentWeapon.Scope.WorldPosition
    elseif currentWeapon.Scope and (currentWeapon.Scope:IsA("Part") or currentWeapon.Scope:IsA("MeshPart")) then
        aimPosition = currentWeapon.Scope.Position
    else
        local handle = tool:FindFirstChild("Handle")
        if handle then
            local lookVector = handle.CFrame.LookVector
            aimPosition = handle.Position + lookVector * 2
        else
            aimPosition = rightArm.Position + Vector3.new(0, -0.5, -1.5)
        end
    end
    
    return aimPosition
end

-- ========== FUNÇÃO PARA ATUALIZAR MIRA DA ARMA ==========
local function updateWeaponAim()
    if not settings.WeaponAimbotEnabled then return end
    if not isAiming then return end
    if not settings.AimbotEnabled then return end
    
    local target = getBestTarget()
    if not target then return end
    
    local character = target.Character
    if not character then return end
    
    local targetPart = getTargetPart(character)
    if not targetPart then return end
    
    local weaponAimPos = getWeaponAimPosition()
    if not weaponAimPos then return end
    
    local currentAimPos = weaponAimPos
    
    local currentWeaponCFrame = currentWeapon.Tool:GetPivot()
    local targetCFrame = CFrame.lookAt(currentAimPos, targetPart.Position)
    
    local smoothedCFrame = currentWeaponCFrame:Lerp(targetCFrame, 1 - settings.WeaponSmoothness)
    currentWeapon.Tool:SetPrimaryPartCFrame(smoothedCFrame)
    
    local handle = currentWeapon.Tool:FindFirstChild("Handle")
    if handle then
        local handleTargetCFrame = CFrame.lookAt(handle.Position, targetPart.Position)
        local smoothedHandle = handle.CFrame:Lerp(handleTargetCFrame, 1 - settings.WeaponSmoothness)
        handle.CFrame = smoothedHandle
    end
end

-- ========== DETECÇÃO DE MIRA (BOTÃO DIREITO) ==========
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if not hubLoaded then return end
    
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = true
        currentWeapon = getCurrentWeapon()
        
        if settings.WeaponAimbotEnabled then
            local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
            if screenGui then
                local feedback = Instance.new("TextLabel")
                feedback.Size = UDim2.new(0, 200, 0, 30)
                feedback.Position = UDim2.new(0.5, -100, 0.85, 0)
                feedback.Text = "🔫 MIRA ATIVA" .. (currentWeapon and currentWeapon.HasScope and " (SCOPE)" or "")
                feedback.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
                feedback.TextColor3 = Color3.fromRGB(255, 255, 255)
                feedback.Font = Enum.Font.GothamBold
                feedback.TextSize = 14
                feedback.Parent = screenGui
                game:GetService("Debris"):AddItem(feedback, 1)
            end
        end
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = false
    end
end)

-- ========== FUNÇÕES UI ==========
local function createUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SnakeHubUI"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = LocalPlayer.PlayerGui
    
    -- Frame principal
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 320, 0, 480)
    mainFrame.Position = UDim2.new(0, 10, 0, 10)
    mainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
    mainFrame.BackgroundTransparency = 0.05
    mainFrame.BorderSizePixel = 1
    mainFrame.BorderColor3 = Color3.fromRGB(0, 162, 255)
    mainFrame.Parent = screenGui
    
    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 8)
    mainCorner.Parent = mainFrame
    
    -- Título Snake Hub
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 45)
    title.Position = UDim2.new(0, 0, 0, 0)
    title.Text = "SNAKE HUB 🐍"
    title.TextColor3 = Color3.fromRGB(0, 162, 255)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.TextSize = 20
    title.Parent = mainFrame
    
    -- By: Snake
    local subTitle = Instance.new("TextLabel")
    subTitle.Size = UDim2.new(1, 0, 0, 20)
    subTitle.Position = UDim2.new(0, 0, 0, 38)
    subTitle.Text = "By: Snake"
    subTitle.TextColor3 = Color3.fromRGB(150, 150, 170)
    subTitle.BackgroundTransparency = 1
    subTitle.Font = Enum.Font.Gotham
    subTitle.TextSize = 12
    subTitle.Parent = mainFrame
    
    -- Status Aimbot
    local aimbotStatus = Instance.new("TextLabel")
    aimbotStatus.Size = UDim2.new(1, -20, 0, 30)
    aimbotStatus.Position = UDim2.new(0, 10, 0, 65)
    aimbotStatus.Text = "🎯 AIMBOT: " .. (settings.AimbotEnabled and "✅ ON" or "❌ OFF")
    aimbotStatus.TextColor3 = settings.AimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    aimbotStatus.BackgroundTransparency = 1
    aimbotStatus.Font = Enum.Font.Gotham
    aimbotStatus.TextSize = 13
    aimbotStatus.TextXAlignment = Enum.TextXAlignment.Left
    aimbotStatus.Parent = mainFrame
    
    -- Toggle Aimbot (R)
    local toggleBtn = Instance.new("TextButton")
    toggleBtn.Size = UDim2.new(0, 80, 0, 30)
    toggleBtn.Position = UDim2.new(1, -90, 0, 65)
    toggleBtn.Text = "R - TOGGLE"
    toggleBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    toggleBtn.Font = Enum.Font.Gotham
    toggleBtn.TextSize = 11
    toggleBtn.Parent = mainFrame
    
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 4)
    btnCorner.Parent = toggleBtn
    
    toggleBtn.MouseButton1Click:Connect(function()
        settings.AimbotEnabled = not settings.AimbotEnabled
        aimbotStatus.Text = "🎯 AIMBOT: " .. (settings.AimbotEnabled and "✅ ON" or "❌ OFF")
        aimbotStatus.TextColor3 = settings.AimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    end)
    
    -- Status Weapon Aimbot
    local weaponStatus = Instance.new("TextLabel")
    weaponStatus.Size = UDim2.new(1, -20, 0, 25)
    weaponStatus.Position = UDim2.new(0, 10, 0, 100)
    weaponStatus.Text = "🔫 WEAPON AIM: " .. (settings.WeaponAimbotEnabled and "✅ ON" or "❌ OFF")
    weaponStatus.TextColor3 = settings.WeaponAimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    weaponStatus.BackgroundTransparency = 1
    weaponStatus.Font = Enum.Font.Gotham
    weaponStatus.TextSize = 12
    weaponStatus.TextXAlignment = Enum.TextXAlignment.Left
    weaponStatus.Parent = mainFrame
    
    local weaponToggle = Instance.new("TextButton")
    weaponToggle.Size = UDim2.new(0, 80, 0, 25)
    weaponToggle.Position = UDim2.new(1, -90, 0, 100)
    weaponToggle.Text = "TOGGLE"
    weaponToggle.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    weaponToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
    weaponToggle.Font = Enum.Font.Gotham
    weaponToggle.TextSize = 11
    weaponToggle.Parent = mainFrame
    
    weaponToggle.MouseButton1Click:Connect(function()
        settings.WeaponAimbotEnabled = not settings.WeaponAimbotEnabled
        weaponStatus.Text = "🔫 WEAPON AIM: " .. (settings.WeaponAimbotEnabled and "✅ ON" or "❌ OFF")
        weaponStatus.TextColor3 = settings.WeaponAimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    end)
    
    -- Separador
    local sep = Instance.new("Frame")
    sep.Size = UDim2.new(1, -20, 0, 1)
    sep.Position = UDim2.new(0, 10, 0, 132)
    sep.BackgroundColor3 = Color3.fromRGB(80, 80, 90)
    sep.Parent = mainFrame
    
    -- Dropdown parte do corpo
    local partLabel = Instance.new("TextLabel")
    partLabel.Size = UDim2.new(1, -20, 0, 25)
    partLabel.Position = UDim2.new(0, 10, 0, 142)
    partLabel.Text = "🎯 MIRAR EM: " .. settings.TargetPart
    partLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    partLabel.BackgroundTransparency = 1
    partLabel.Font = Enum.Font.Gotham
    partLabel.TextSize = 12
    partLabel.TextXAlignment = Enum.TextXAlignment.Left
    partLabel.Parent = mainFrame
    
    local partDropdown = Instance.new("TextButton")
    partDropdown.Size = UDim2.new(0, 100, 0, 25)
    partDropdown.Position = UDim2.new(1, -110, 0, 142)
    partDropdown.Text = settings.TargetPart
    partDropdown.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    partDropdown.TextColor3 = Color3.fromRGB(255, 255, 255)
    partDropdown.Font = Enum.Font.Gotham
    partDropdown.TextSize = 11
    partDropdown.Parent = mainFrame
    
    local dropdownOpen = false
    local dropdownList = nil
    
    partDropdown.MouseButton1Click:Connect(function()
        if dropdownOpen then
            if dropdownList then dropdownList:Destroy() end
            dropdownOpen = false
            return
        end
        
        dropdownList = Instance.new("Frame")
        dropdownList.Size = UDim2.new(0, 100, 0, 100)
        dropdownList.Position = UDim2.new(1, -110, 0, 167)
        dropdownList.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
        dropdownList.BorderSizePixel = 1
        dropdownList.BorderColor3 = Color3.fromRGB(0, 162, 255)
        dropdownList.Parent = mainFrame
        
        for i, part in ipairs(bodyParts) do
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(1, 0, 0, 25)
            btn.Position = UDim2.new(0, 0, 0, (i-1)*25)
            btn.Text = part
            btn.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
            btn.Font = Enum.Font.Gotham
            btn.TextSize = 11
            btn.Parent = dropdownList
            
            btn.MouseButton1Click:Connect(function()
                settings.TargetPart = part
                partDropdown.Text = part
                partLabel.Text = "🎯 MIRAR EM: " .. part
                dropdownList:Destroy()
                dropdownOpen = false
            end)
        end
        dropdownOpen = true
    end)
    
    -- Modo de alvo
    local modeLabel = Instance.new("TextLabel")
    modeLabel.Size = UDim2.new(1, -20, 0, 25)
    modeLabel.Position = UDim2.new(0, 10, 0, 177)
    modeLabel.Text = "🎮 MODO: " .. settings.TargetMode
    modeLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    modeLabel.BackgroundTransparency = 1
    modeLabel.Font = Enum.Font.Gotham
    modeLabel.TextSize = 12
    modeLabel.TextXAlignment = Enum.TextXAlignment.Left
    modeLabel.Parent = mainFrame
    
    local modeBtn = Instance.new("TextButton")
    modeBtn.Size = UDim2.new(0, 80, 0, 25)
    modeBtn.Position = UDim2.new(1, -90, 0, 177)
    modeBtn.Text = settings.TargetMode
    modeBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    modeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    modeBtn.Font = Enum.Font.Gotham
    modeBtn.TextSize = 11
    modeBtn.Parent = mainFrame
    
    modeBtn.MouseButton1Click:Connect(function()
        settings.TargetMode = (settings.TargetMode == "Auto" and "Manual" or "Auto")
        modeBtn.Text = settings.TargetMode
        modeLabel.Text = "🎮 MODO: " .. settings.TargetMode
        if settings.TargetMode == "Manual" then
            settings.SelectedPlayer = nil
        end
    end)
    
    -- Lista de jogadores
    local playerListFrame = Instance.new("ScrollingFrame")
    playerListFrame.Size = UDim2.new(1, -20, 0, 100)
    playerListFrame.Position = UDim2.new(0, 10, 0, 212)
    playerListFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    playerListFrame.BorderSizePixel = 1
    playerListFrame.BorderColor3 = Color3.fromRGB(60, 60, 70)
    playerListFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    playerListFrame.ScrollBarThickness = 8
    playerListFrame.Parent = mainFrame
    
    local function updatePlayerList()
        for _, child in ipairs(playerListFrame:GetChildren()) do
            if child:IsA("TextButton") then
                child:Destroy()
            end
        end
        
        local yOffset = 0
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local playerBtn = Instance.new("TextButton")
                playerBtn.Size = UDim2.new(1, -10, 0, 25)
                playerBtn.Position = UDim2.new(0, 5, 0, yOffset)
                playerBtn.Text = player.Name
                playerBtn.BackgroundColor3 = (settings.SelectedPlayer == player.Name) and Color3.fromRGB(0, 100, 200) or Color3.fromRGB(50, 50, 60)
                playerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
                playerBtn.Font = Enum.Font.Gotham
                playerBtn.TextSize = 12
                playerBtn.Parent = playerListFrame
                
                playerBtn.MouseButton1Click:Connect(function()
                    if settings.TargetMode == "Manual" then
                        settings.SelectedPlayer = player.Name
                        updatePlayerList()
                    end
                end)
                
                yOffset = yOffset + 28
            end
        end
        playerListFrame.CanvasSize = UDim2.new(0, 0, 0, yOffset)
    end
    
    updatePlayerList()
    Players.PlayerAdded:Connect(updatePlayerList)
    Players.PlayerRemoving:Connect(updatePlayerList)
    
    -- Sliders
    local smoothLabel = Instance.new("TextLabel")
    smoothLabel.Size = UDim2.new(1, -20, 0, 20)
    smoothLabel.Position = UDim2.new(0, 10, 0, 322)
    smoothLabel.Text = "⚙️ CAMERA SMOOTH: " .. string.format("%.2f", settings.Smoothness)
    smoothLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    smoothLabel.BackgroundTransparency = 1
    smoothLabel.Font = Enum.Font.Gotham
    smoothLabel.TextSize = 11
    smoothLabel.TextXAlignment = Enum.TextXAlignment.Left
    smoothLabel.Parent = mainFrame
    
    local smoothSlider = Instance.new("TextBox")
    smoothSlider.Size = UDim2.new(0, 80, 0, 20)
    smoothSlider.Position = UDim2.new(1, -90, 0, 320)
    smoothSlider.Text = tostring(settings.Smoothness)
    smoothSlider.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    smoothSlider.TextColor3 = Color3.fromRGB(255, 255, 255)
    smoothSlider.Font = Enum.Font.Gotham
    smoothSlider.TextSize = 11
    smoothSlider.Parent = mainFrame
    
    smoothSlider.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            local num = tonumber(smoothSlider.Text)
            if num and num >= 0.05 and num <= 0.8 then
                settings.Smoothness = num
                smoothLabel.Text = "⚙️ CAMERA SMOOTH: " .. string.format("%.2f", settings.Smoothness)
            else
                smoothSlider.Text = tostring(settings.Smoothness)
            end
        end
    end)
    
    local weaponSmoothLabel = Instance.new("TextLabel")
    weaponSmoothLabel.Size = UDim2.new(1, -20, 0, 20)
    weaponSmoothLabel.Position = UDim2.new(0, 10, 0, 350)
    weaponSmoothLabel.Text = "🔫 WEAPON SMOOTH: " .. string.format("%.2f", settings.WeaponSmoothness)
    weaponSmoothLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    weaponSmoothLabel.BackgroundTransparency = 1
    weaponSmoothLabel.Font = Enum.Font.Gotham
    weaponSmoothLabel.TextSize = 11
    weaponSmoothLabel.TextXAlignment = Enum.TextXAlignment.Left
    weaponSmoothLabel.Parent = mainFrame
    
    local weaponSmoothSlider = Instance.new("TextBox")
    weaponSmoothSlider.Size = UDim2.new(0, 80, 0, 20)
    weaponSmoothSlider.Position = UDim2.new(1, -90, 0, 348)
    weaponSmoothSlider.Text = tostring(settings.WeaponSmoothness)
    weaponSmoothSlider.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    weaponSmoothSlider.TextColor3 = Color3.fromRGB(255, 255, 255)
    weaponSmoothSlider.Font = Enum.Font.Gotham
    weaponSmoothSlider.TextSize = 11
    weaponSmoothSlider.Parent = mainFrame
    
    weaponSmoothSlider.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            local num = tonumber(weaponSmoothSlider.Text)
            if num and num >= 0.05 and num <= 0.8 then
                settings.WeaponSmoothness = num
                weaponSmoothLabel.Text = "🔫 WEAPON SMOOTH: " .. string.format("%.2f", settings.WeaponSmoothness)
            else
                weaponSmoothSlider.Text = tostring(settings.WeaponSmoothness)
            end
        end
    end)
    
    -- FOV slider
    local fovLabel = Instance.new("TextLabel")
    fovLabel.Size = UDim2.new(1, -20, 0, 20)
    fovLabel.Position = UDim2.new(0, 10, 0, 378)
    fovLabel.Text = "👁️ FOV RAIO: " .. settings.FOVRadius
    fovLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    fovLabel.BackgroundTransparency = 1
    fovLabel.Font = Enum.Font.Gotham
    fovLabel.TextSize = 11
    fovLabel.TextXAlignment = Enum.TextXAlignment.Left
    fovLabel.Parent = mainFrame
    
    local fovSlider = Instance.new("TextBox")
    fovSlider.Size = UDim2.new(0, 80, 0, 20)
    fovSlider.Position = UDim2.new(1, -90, 0, 376)
    fovSlider.Text = tostring(settings.FOVRadius)
    fovSlider.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    fovSlider.TextColor3 = Color3.fromRGB(255, 255, 255)
    fovSlider.Font = Enum.Font.Gotham
    fovSlider.TextSize = 11
    fovSlider.Parent = mainFrame
    
    fovSlider.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            local num = tonumber(fovSlider.Text)
            if num and num >= 0 and num <= 800 then
                settings.FOVRadius = num
                fovLabel.Text = "👁️ FOV RAIO: " .. settings.FOVRadius
                updateFOVCircle()
            else
                fovSlider.Text = tostring(settings.FOVRadius)
            end
        end
    end)
    
    -- Toggle FOV
    local fovCircleBtn = Instance.new("TextButton")
    fovCircleBtn.Size = UDim2.new(0, 100, 0, 25)
    fovCircleBtn.Position = UDim2.new(0, 10, 0, 408)
    fovCircleBtn.Text = settings.ShowFOVCircle and "🔘 FOV: ON" or "⚪ FOV: OFF"
    fovCircleBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    fovCircleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    fovCircleBtn.Font = Enum.Font.Gotham
    fovCircleBtn.TextSize = 11
    fovCircleBtn.Parent = mainFrame
    
    fovCircleBtn.MouseButton1Click:Connect(function()
        settings.ShowFOVCircle = not settings.ShowFOVCircle
        fovCircleBtn.Text = settings.ShowFOVCircle and "🔘 FOV: ON" or "⚪ FOV: OFF"
        updateFOVCircle()
    end)
    
    -- Toggle ESP
    local espBtn = Instance.new("TextButton")
    espBtn.Size = UDim2.new(0, 100, 0, 25)
    espBtn.Position = UDim2.new(1, -110, 0, 408)
    espBtn.Text = settings.ESPEnabled and "👁️ ESP: ON" or "👁️ ESP: OFF"
    espBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    espBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    espBtn.Font = Enum.Font.Gotham
    espBtn.TextSize = 11
    espBtn.Parent = mainFrame
    
    espBtn.MouseButton1Click:Connect(function()
        settings.ESPEnabled = not settings.ESPEnabled
        espBtn.Text = settings.ESPEnabled and "👁️ ESP: ON" or "👁️ ESP: OFF"
        if not settings.ESPEnabled then
            clearESP()
        end
    end)
    
    -- Botão fechar
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 60, 0, 25)
    closeBtn.Position = UDim2.new(1, -70, 1, -30)
    closeBtn.Text = "FECHAR"
    closeBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.Font = Enum.Font.Gotham
    closeBtn.TextSize = 11
    closeBtn.Parent = mainFrame
    
    closeBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = false
    end)
    
    -- Botão abrir
    local openBtn = Instance.new("TextButton")
    openBtn.Size = UDim2.new(0, 40, 0, 40)
    openBtn.Position = UDim2.new(0, 10, 1, -50)
    openBtn.Text = "🐍"
    openBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    openBtn.Font = Enum.Font.GothamBold
    openBtn.TextSize = 20
    openBtn.Parent = screenGui
    
    local openCorner = Instance.new("UICorner")
    openCorner.CornerRadius = UDim.new(0, 20)
    openCorner.Parent = openBtn
    
    openBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = true
    end)
    
    return screenGui
end

-- ========== FOV CIRCLE ==========
local function updateFOVCircle()
    if fovCircle then fovCircle:Destroy() end
    if not settings.ShowFOVCircle or settings.FOVRadius <= 0 then return end
    
    local circle = Instance.new("Frame")
    circle.Size = UDim2.new(0, settings.FOVRadius * 2, 0, settings.FOVRadius * 2)
    circle.Position = UDim2.new(0.5, -settings.FOVRadius, 0.5, -settings.FOVRadius)
    circle.BackgroundTransparency = 1
    circle.BorderSizePixel = 2
    circle.BorderColor3 = Color3.fromRGB(255, 255, 255)
    circle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    circle.Parent = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI") or LocalPlayer.PlayerGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = circle
    
    fovCircle = circle
end

-- ========== ESP ==========
local function createESP(player)
    if espObjects[player] then return end
    
    local character = player.Character
    if not character then return end
    
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end
    
    local head = character:FindFirstChild("Head")
    if not head then return end
    
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "ESP_" .. player.Name
    billboard.Size = UDim2.new(0, 200, 0, 60)
    billboard.StudsOffset = Vector3.new(0, 2.5, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = head
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 1, 0)
    frame.BackgroundTransparency = 1
    frame.Parent = billboard
    
    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, 0, 0, 20)
    nameLabel.Position = UDim2.new(0, 0, 0, 0)
    nameLabel.Text = player.Name
    nameLabel.TextColor3 = settings.ESPTextColor
    nameLabel.BackgroundTransparency = 1
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = 14
    nameLabel.TextStrokeTransparency = 0.3
    nameLabel.Parent = frame
    
    local healthLabel = Instance.new("TextLabel")
    healthLabel.Size = UDim2.new(1, 0, 0, 16)
    healthLabel.Position = UDim2.new(0, 0, 0, 20)
    healthLabel.Text = "❤️ " .. math.floor(humanoid.Health)
    healthLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
    healthLabel.BackgroundTransparency = 1
    healthLabel.Font = Enum.Font.Gotham
    healthLabel.TextSize = 12
    healthLabel.Parent = frame
    
    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Size = UDim2.new(1, 0, 0, 16)
    distanceLabel.Position = UDim2.new(0, 0, 0, 36)
    distanceLabel.Text = "📏 --"
    distanceLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.Font = Enum.Font.Gotham
    distanceLabel.TextSize = 11
    distanceLabel.Parent = frame
    
    local highlight = Instance.new("Highlight")
    highlight.Name = "ESP_Highlight_" .. player.Name
    highlight.FillColor = settings.ESPBoxColor
    highlight.FillTransparency = 0.8
    highlight.OutlineColor = settings.ESPBoxColor
    highlight.OutlineTransparency = 0.3
    highlight.Parent = character
    
    espObjects[player] = {
        Billboard = billboard,
        Highlight = highlight,
        NameLabel = nameLabel,
        HealthLabel = healthLabel,
        DistanceLabel = distanceLabel
    }
end

local function updateESP()
    if not settings.ESPEnabled then
        clearESP()
        return
    end
    
    local camera = workspace.CurrentCamera
    local localChar = LocalPlayer.Character
    local localPos = localChar and localChar:FindFirstChild("HumanoidRootPart") and localChar.HumanoidRootPart.Position or Vector3.zero
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local character = player.Character
            if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
                if not espObjects[player] then
                    createESP(player)
                else
                    local humanoid = character:FindFirstChild("Humanoid")
                    local rootPart = character:FindFirstChild("HumanoidRootPart")
                    
                    if espObjects[player].HealthLabel and humanoid then
                        espObjects[player].HealthLabel.Text = "❤️ " .. math.floor(humanoid.Health)
                    end
                    
                    if espObjects[player].DistanceLabel and rootPart and localPos then
                        local dist = (rootPart.Position - localPos).Magnitude
                        espObjects[player].DistanceLabel.Text = "📏 " .. math.floor(dist) .. " studs"
                    end
                end
            elseif espObjects[player] then
                espObjects[player].Billboard:Destroy()
                if espObjects[player].Highlight then espObjects[player].Highlight:Destroy() end
                espObjects[player] = nil
            end
        end
    end
end

local function clearESP()
    for player, objects in pairs(espObjects) do
        pcall(function()
            objects.Billboard:Destroy()
            if objects.Highlight then objects.Highlight:Destroy() end
        end)
    end
    espObjects = {}
end

-- ========== FUNÇÕES PRINCIPAIS ==========
local function getTargetPart(character)
    if not character then return nil end
    local part = character:FindFirstChild(settings.TargetPart)
    if not part then
        part = character:FindFirstChild("Head") or character:FindFirstChild("UpperTorso") or character:FindFirstChild("HumanoidRootPart")
    end
    return part
end

local function getBestTarget()
    if not settings.AimbotEnabled then return nil end
    
    local camera = workspace.CurrentCamera
    local viewportCenter = camera.ViewportSize / 2
    local bestPlayer = nil
    local bestDistance = settings.FOVRadius > 0 and settings.FOVRadius or math.huge
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local character = player.Character
            if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
                
                if settings.TargetMode == "Manual" and settings.SelectedPlayer then
                    if player.Name == settings.SelectedPlayer then
                        return player
                    else
                        continue
                    end
                end
                
                local targetPart = getTargetPart(character)
                if targetPart then
                    local screenPos, onScreen = camera:WorldToViewportPoint(targetPart.Position)
                    if onScreen then
                        local distToCenter = (Vector2.new(screenPos.X, screenPos.Y) - viewportCenter).Magnitude
                        if distToCenter < bestDistance then
                            bestDistance = distToCenter
                            bestPlayer = player
                        end
                    end
                end
            end
        end
    end
    
    return bestPlayer
end

-- Aimbot da câmera
local function updateCameraAimbot()
    if not settings.AimbotEnabled then return end
    
    local target = getBestTarget()
    if not target then return end
    
    local character = target.Character
    if not character then return end
    
    local targetPart = getTargetPart(character)
    if not targetPart then return end
    
    local camera = workspace.CurrentCamera
    local targetPosition = targetPart.Position
    local currentCFrame = camera.CFrame
    
    local newCFrame = CFrame.new(currentCFrame.Position, targetPosition)
    local smoothedCFrame = currentCFrame:Lerp(newCFrame, 1 - settings.Smoothness)
    camera.CFrame = smoothedCFrame
end

-- ========== KEYBIND R (PRINCIPAL) ==========
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if not hubLoaded then return end
    
    -- Tecla R para ligar/desligar aimbot
    if input.KeyCode == Enum.KeyCode.R then
        settings.AimbotEnabled = not settings.AimbotEnabled
        
        local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
        if screenGui then
            local mainFrame = screenGui:FindFirstChildOfClass("Frame")
            if mainFrame then
                for _, child in ipairs(mainFrame:GetChildren()) do
                    if child:IsA("TextLabel") and child.Text:find("AIMBOT:") then
                        child.Text = "🎯 AIMBOT: " .. (settings.AimbotEnabled and "✅ ON" or "❌ OFF")
                        child.TextColor3 = settings.AimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
                    end
                end
            end
        end
        
        -- Feedback visual
        local feedback = Instance.new("TextLabel")
        feedback.Size = UDim2.new(0, 200, 0, 30)
        feedback.Position = UDim2.new(0.5, -100, 0.7, 0)
        feedback.Text = settings.AimbotEnabled and "⚡ AIMBOT ATIVADO" or "❌ AIMBOT DESATIVADO"
        feedback.BackgroundColor3 = settings.AimbotEnabled and Color3.fromRGB(0, 162, 255) or Color3.fromRGB(200, 50, 50)
        feedback.TextColor3 = Color3.fromRGB(255, 255, 255)
        feedback.Font = Enum.Font.GothamBold
        feedback.TextSize = 14
        feedback.Parent = screenGui or LocalPlayer.PlayerGui
        game:GetService("Debris"):AddItem(feedback, 1.5)
    end
end)

-- ========== CARREGAR HUB ==========
function loadHub()
    createUI()
    updateFOVCircle()
    hubLoaded = true
    
    -- Loop principal
    RunService.RenderStepped:Connect(function()
        if not hubLoaded then return end
        updateCameraAimbot()
        updateESP()
        
        local newWeapon = getCurrentWeapon()
        if newWeapon and (not currentWeapon or currentWeapon.Tool ~= newWeapon.Tool) then
            currentWeapon = newWeapon
        elseif not newWeapon and currentWeapon then
            currentWeapon = nil
        end
        
        if isAiming and currentWeapon then
            updateWeaponAim()
        end
    end)
    
    -- Atualiza lista de jogadores
    local function updateFullPlayerList()
        if not hubLoaded then return end
        local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
        if screenGui then
            local mainFrame = screenGui:FindFirstChildOfClass("Frame")
            if mainFrame then
                local playerListFrame = mainFrame:FindFirstChildOfClass("ScrollingFrame")
                if playerListFrame then
                    for _, child in ipairs(playerListFrame:GetChildren()) do
                        if child:IsA("TextButton") then
                            child:Destroy()
                        end
                    end
                    
                    local yOffset = 0
                    for _, player in ipairs(Players:GetPlayers()) do
                        if player ~= LocalPlayer then
                            local playerBtn = Instance.new("TextButton")
                            playerBtn.Size = UDim2.new(1, -10, 0, 25)
                            playerBtn.Position = UDim2.new(0, 5, 0, yOffset)
                            playerBtn.Text = player.Name
                            playerBtn.BackgroundColor3 = (settings.SelectedPlayer == player.Name) and Color3.fromRGB(0, 100, 200) or Color3.fromRGB(50, 50, 60)
                            playerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
                            playerBtn.Font = Enum.Font.Gotham
                            playerBtn.TextSize = 12
                            playerBtn.Parent = playerListFrame
                            
                            playerBtn.MouseButton1Click:Connect(function()
                                if settings.TargetMode == "Manual" then
                                    settings.SelectedPlayer = player.Name
                                    updateFullPlayerList()
                                end
                            end)
                            
                            yOffset = yOffset + 28
                        end
                    end
                    playerListFrame.CanvasSize = UDim2.new(0, 0, 0, yOffset)
                end
            end
        end
    end
    
    Players.PlayerAdded:Connect(updateFullPlayerList)
    Players.PlayerRemoving:Connect(function(player)
        updateFullPlayerList()
        if espObjects[player] then
            pcall(function()
                espObjects[player].Billboard:Destroy()
                if espObjects[player].Highlight then espObjects[player].Highlight:Destroy() end
            end)
            espObjects[player] = nil
        end
    end)
    
    print("✅ Snake Hub 🐍 Carregado com sucesso!")
    print("📌 KEY: Essaeakey - Verificada com sucesso!")
    print("📌 Pressione R para ligar/desligar o aimbot")
    print("📌 Clique direito para mirar com a arma")
    print("📌 Modo Manual: clique em um jogador na lista da UI")
end

-- ========== INICIAR SISTEMA DE KEY ==========
createKeyScreen()
