local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()

local REQUIRED_KEY = "Essaeakey"
local keyInserted = false
local hubLoaded = false

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
    AimbotToggleKey = "R",
    ESPToggleKey = "E",
    UIToggleKey = "Insert",
    AimKey = "MouseButton2"
}

local keyNames = {
    ["R"] = "R",
    ["E"] = "E",
    ["Insert"] = "Insert",
    ["MouseButton2"] = "Botão Direito"
}

local keyEnumMap = {
    ["R"] = Enum.KeyCode.R,
    ["E"] = Enum.KeyCode.E,
    ["Insert"] = Enum.KeyCode.Insert,
    ["MouseButton2"] = Enum.UserInputType.MouseButton2
}

local bodyParts = {"Head", "UpperTorso", "LowerTorso", "HumanoidRootPart"}
local espObjects = {}
local fovCircle = nil
local isAiming = false
local currentWeapon = nil
local isUIHidden = false
local waitingForKey = nil
local waitingForType = nil

local function safeDestroy(obj)
    if obj and obj.Parent then
        pcall(function() obj:Destroy() end)
    end
end

local function createTween(obj, properties, duration)
    local tweenInfo = TweenInfo.new(duration, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    local tween = TweenService:Create(obj, tweenInfo, properties)
    tween:Play()
    return tween
end

local function getCurrentWeapon()
    local character = LocalPlayer.Character
    if not character then return nil end
    
    local tool = character:FindFirstChildOfClass("Tool")
    if not tool then return nil end
    
    local isWeapon = false
    
    if tool:FindFirstChild("Handle") then
        isWeapon = true
    end
    
    for _, child in ipairs(tool:GetChildren()) do
        if child:IsA("Script") or child:IsA("LocalScript") then
            local scriptName = child.Name:lower()
            if scriptName:find("weapon") or scriptName:find("gun") or scriptName:find("fire") or scriptName:find("shoot") then
                isWeapon = true
                break
            end
        end
    end
    
    local toolName = tool.Name:lower()
    local weaponKeywords = {"gun", "pistol", "rifle", "shotgun", "sniper", "ak", "m4", "deagle", "revolver"}
    for _, keyword in ipairs(weaponKeywords) do
        if toolName:find(keyword) then
            isWeapon = true
            break
        end
    end
    
    if not isWeapon then return nil end
    
    local scope = nil
    for _, child in ipairs(tool:GetDescendants()) do
        if child:IsA("Part") or child:IsA("MeshPart") then
            local name = child.Name:lower()
            if name:find("scope") or name:find("sight") or name:find("aim") or name:find("lens") or name:find("glass") then
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
        Scope = scope,
        HasScope = scope ~= nil
    }
end

local function getWeaponAimPosition()
    if not currentWeapon or not currentWeapon.Tool then return nil end
    
    local tool = currentWeapon.Tool
    local character = LocalPlayer.Character
    if not character then return nil end
    
    local rightArm = character:FindFirstChild("Right Arm") or character:FindFirstChild("RightHand") or character:FindFirstChild("RightUpperArm")
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
            aimPosition = handle.Position + lookVector * 3
        else
            local camera = workspace.CurrentCamera
            aimPosition = rightArm.Position + (camera.CFrame.LookVector * 2)
        end
    end
    
    return aimPosition
end

local function updateWeaponAim()
    if not settings.WeaponAimbotEnabled or not isAiming or not settings.AimbotEnabled then return end
    
    local target = getBestTarget()
    if not target then return end
    
    local character = target.Character
    if not character then return end
    
    local targetPart = getTargetPart(character)
    if not targetPart or not targetPart.Parent then return end
    
    local weaponAimPos = getWeaponAimPosition()
    if not weaponAimPos then return end
    
    if not currentWeapon.Tool or not currentWeapon.Tool.Parent then
        currentWeapon = nil
        return
    end
    
    local tool = currentWeapon.Tool
    local handle = tool:FindFirstChild("Handle")
    
    if handle then
        local direction = (targetPart.Position - handle.Position).Unit
        local targetCFrame = CFrame.lookAt(handle.Position, handle.Position + direction)
        local smoothedCFrame = handle.CFrame:Lerp(targetCFrame, 1 - settings.WeaponSmoothness)
        handle.CFrame = smoothedCFrame
    end
end

local function startKeyBinding(button, bindType, currentKey)
    waitingForKey = button
    waitingForType = bindType
    button.Text = "Aguardando..."
    button.BackgroundColor3 = Color3.fromRGB(255, 100, 0)
end

local function updateKeyBind(bindType, newKey)
    if bindType == "aimbot" then
        settings.AimbotToggleKey = newKey
    elseif bindType == "esp" then
        settings.ESPToggleKey = newKey
    elseif bindType == "ui" then
        settings.UIToggleKey = newKey
    elseif bindType == "aim" then
        settings.AimKey = newKey
    end
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if waitingForKey and waitingForType then
        local keyName = nil
        local inputType = nil
        
        if input.KeyCode ~= Enum.KeyCode.Unknown then
            keyName = tostring(input.KeyCode):gsub("Enum.KeyCode.", "")
            inputType = "keycode"
        elseif input.UserInputType ~= Enum.UserInputType.None then
            keyName = tostring(input.UserInputType):gsub("Enum.UserInputType.", "")
            inputType = "userinput"
        end
        
        if keyName and keyName ~= "Unknown" then
            updateKeyBind(waitingForType, keyName)
            waitingForKey.Text = keyNames[keyName] or keyName
            waitingForKey.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
            waitingForKey = nil
            waitingForType = nil
        end
        return
    end
    
    if not hubLoaded then return end
    
    local aimbotKeyEnum = keyEnumMap[settings.AimbotToggleKey]
    local espKeyEnum = keyEnumMap[settings.ESPToggleKey]
    local uiKeyEnum = keyEnumMap[settings.UIToggleKey]
    local aimKeyEnum = keyEnumMap[settings.AimKey]
    
    if input.KeyCode == aimbotKeyEnum then
        settings.AimbotEnabled = not settings.AimbotEnabled
        updateUIElement("AimbotStatus", settings.AimbotEnabled)
    end
    
    if input.KeyCode == espKeyEnum then
        settings.ESPEnabled = not settings.ESPEnabled
        updateUIElement("ESPStatus", settings.ESPEnabled)
        if not settings.ESPEnabled then
            clearESP()
        end
    end
    
    if input.KeyCode == uiKeyEnum then
        toggleUI()
    end
    
    if input.UserInputType == aimKeyEnum then
        isAiming = true
        currentWeapon = getCurrentWeapon()
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if not hubLoaded then return end
    
    local aimKeyEnum = keyEnumMap[settings.AimKey]
    
    if input.UserInputType == aimKeyEnum then
        isAiming = false
    end
end)

local function updateUIElement(elementType, value)
    local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
    if not screenGui then return end
    
    if elementType == "AimbotStatus" then
        local statusLabel = screenGui:FindFirstChild("AimbotStatusLabel")
        if statusLabel then
            statusLabel.Text = "🎯 AIMBOT: " .. (value and "✅ ON" or "❌ OFF")
            statusLabel.TextColor3 = value and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
        end
    elseif elementType == "ESPStatus" then
        local espBtn = screenGui:FindFirstChild("ESPToggleBtn")
        if espBtn then
            espBtn.Text = value and "👁️ ESP: ON" or "👁️ ESP: OFF"
        end
    end
end

local function toggleUI()
    local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
    if not screenGui then return end
    
    local mainFrame = screenGui:FindFirstChild("MainFrame")
    if mainFrame then
        isUIHidden = not isUIHidden
        mainFrame.Visible = not isUIHidden
    end
end

local function createKeyScreen()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SnakeHubKeySystem"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = LocalPlayer.PlayerGui
    
    local background = Instance.new("Frame")
    background.Size = UDim2.new(1, 0, 1, 0)
    background.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    background.BackgroundTransparency = 0.85
    background.Parent = screenGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 450, 0, 350)
    mainFrame.Position = UDim2.new(0.5, -225, 0.5, -175)
    mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
    mainFrame.BorderSizePixel = 0
    mainFrame.ClipsDescendants = true
    mainFrame.Parent = background
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = mainFrame
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 60)
    title.Position = UDim2.new(0, 0, 0, 20)
    title.Text = "SNAKE HUB 🐍"
    title.TextColor3 = Color3.fromRGB(0, 162, 255)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.TextSize = 32
    title.Parent = mainFrame
    
    local subTitle = Instance.new("TextLabel")
    subTitle.Size = UDim2.new(1, 0, 0, 30)
    subTitle.Position = UDim2.new(0, 0, 0, 75)
    subTitle.Text = "Premium Cheat System"
    subTitle.TextColor3 = Color3.fromRGB(180, 180, 200)
    subTitle.BackgroundTransparency = 1
    subTitle.Font = Enum.Font.Gotham
    subTitle.TextSize = 14
    subTitle.Parent = mainFrame
    
    local instruction = Instance.new("TextLabel")
    instruction.Size = UDim2.new(1, -40, 0, 30)
    instruction.Position = UDim2.new(0, 20, 0, 120)
    instruction.Text = "INSIRA A KEY DE ACESSO:"
    instruction.TextColor3 = Color3.fromRGB(200, 200, 200)
    instruction.BackgroundTransparency = 1
    instruction.Font = Enum.Font.GothamBold
    instruction.TextSize = 14
    instruction.TextXAlignment = Enum.TextXAlignment.Left
    instruction.Parent = mainFrame
    
    local keyBox = Instance.new("TextBox")
    keyBox.Size = UDim2.new(1, -40, 0, 45)
    keyBox.Position = UDim2.new(0, 20, 0, 155)
    keyBox.PlaceholderText = "Digite a key aqui..."
    keyBox.Text = ""
    keyBox.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    keyBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    keyBox.Font = Enum.Font.Gotham
    keyBox.TextSize = 16
    keyBox.ClearTextOnFocus = false
    keyBox.Parent = mainFrame
    
    local keyCorner = Instance.new("UICorner")
    keyCorner.CornerRadius = UDim.new(0, 8)
    keyCorner.Parent = keyBox
    
    local confirmBtn = Instance.new("TextButton")
    confirmBtn.Size = UDim2.new(0, 180, 0, 45)
    confirmBtn.Position = UDim2.new(0.5, -90, 0, 220)
    confirmBtn.Text = "CONFIRMAR"
    confirmBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    confirmBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    confirmBtn.Font = Enum.Font.GothamBold
    confirmBtn.TextSize = 16
    confirmBtn.AutoButtonColor = false
    confirmBtn.Parent = mainFrame
    
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 8)
    btnCorner.Parent = confirmBtn
    
    local errorMsg = Instance.new("TextLabel")
    errorMsg.Size = UDim2.new(1, -40, 0, 30)
    errorMsg.Position = UDim2.new(0, 20, 0, 275)
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
            errorMsg.Text = "❌ KEY INCORRETA! Tente novamente."
            errorMsg.Visible = true
            keyBox.Text = ""
            wait(2)
            errorMsg.Visible = false
        end
    end)
    
    keyBox.FocusLost:Connect(function(enterPressed)
        if enterPressed and keyBox.Text == REQUIRED_KEY then
            keyInserted = true
            screenGui:Destroy()
            loadHub()
        elseif enterPressed then
            errorMsg.Text = "❌ KEY INCORRETA! Tente novamente."
            errorMsg.Visible = true
            keyBox.Text = ""
            wait(2)
            errorMsg.Visible = false
        end
    end)
end

local function updatePlayerList(playerListFrame)
    for _, child in ipairs(playerListFrame:GetChildren()) do
        if child:IsA("TextButton") then
            child:Destroy()
        end
    end
    
    local yOffset = 0
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local playerBtn = Instance.new("TextButton")
            playerBtn.Size = UDim2.new(1, -10, 0, 28)
            playerBtn.Position = UDim2.new(0, 5, 0, yOffset)
            playerBtn.Text = player.Name
            playerBtn.BackgroundColor3 = (settings.SelectedPlayer == player.Name) and Color3.fromRGB(0, 100, 200) or Color3.fromRGB(45, 45, 55)
            playerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            playerBtn.Font = Enum.Font.Gotham
            playerBtn.TextSize = 12
            playerBtn.AutoButtonColor = false
            playerBtn.Parent = playerListFrame
            
            local btnCorner = Instance.new("UICorner")
            btnCorner.CornerRadius = UDim.new(0, 4)
            btnCorner.Parent = playerBtn
            
            playerBtn.MouseButton1Click:Connect(function()
                if settings.TargetMode == "Manual" then
                    settings.SelectedPlayer = player.Name
                    updatePlayerList(playerListFrame)
                end
            end)
            
            yOffset = yOffset + 32
        end
    end
    playerListFrame.CanvasSize = UDim2.new(0, 0, 0, yOffset + 5)
end

local function createUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SnakeHubUI"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = LocalPlayer.PlayerGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 380, 0, 650)
    mainFrame.Position = UDim2.new(0, 10, 0, 10)
    mainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
    mainFrame.BackgroundTransparency = 0.05
    mainFrame.BorderSizePixel = 0
    mainFrame.ClipsDescendants = true
    mainFrame.Visible = true
    mainFrame.Parent = screenGui
    
    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 12)
    mainCorner.Parent = mainFrame
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 50)
    title.Position = UDim2.new(0, 0, 0, 0)
    title.Text = "SNAKE HUB 🐍"
    title.TextColor3 = Color3.fromRGB(0, 162, 255)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.TextSize = 22
    title.Parent = mainFrame
    
    local subTitle = Instance.new("TextLabel")
    subTitle.Size = UDim2.new(1, 0, 0, 20)
    subTitle.Position = UDim2.new(0, 0, 0, 42)
    subTitle.Text = "By: Snake | Premium Edition"
    subTitle.TextColor3 = Color3.fromRGB(150, 150, 170)
    subTitle.BackgroundTransparency = 1
    subTitle.Font = Enum.Font.Gotham
    subTitle.TextSize = 11
    subTitle.Parent = mainFrame
    
    local aimbotStatus = Instance.new("TextLabel")
    aimbotStatus.Name = "AimbotStatusLabel"
    aimbotStatus.Size = UDim2.new(1, -20, 0, 35)
    aimbotStatus.Position = UDim2.new(0, 10, 0, 70)
    aimbotStatus.Text = "🎯 AIMBOT: " .. (settings.AimbotEnabled and "✅ ON" or "❌ OFF")
    aimbotStatus.TextColor3 = settings.AimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    aimbotStatus.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    aimbotStatus.BackgroundTransparency = 0.5
    aimbotStatus.Font = Enum.Font.GothamBold
    aimbotStatus.TextSize = 14
    aimbotStatus.TextXAlignment = Enum.TextXAlignment.Left
    aimbotStatus.Parent = mainFrame
    
    local aimbotStatusCorner = Instance.new("UICorner")
    aimbotStatusCorner.CornerRadius = UDim.new(0, 6)
    aimbotStatusCorner.Parent = aimbotStatus
    
    local aimbotBindBtn = Instance.new("TextButton")
    aimbotBindBtn.Size = UDim2.new(0, 100, 0, 25)
    aimbotBindBtn.Position = UDim2.new(1, -110, 0, 75)
    aimbotBindBtn.Text = "Bind: " .. (keyNames[settings.AimbotToggleKey] or settings.AimbotToggleKey)
    aimbotBindBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    aimbotBindBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    aimbotBindBtn.Font = Enum.Font.Gotham
    aimbotBindBtn.TextSize = 10
    aimbotBindBtn.Parent = mainFrame
    
    local aimbotBindCorner = Instance.new("UICorner")
    aimbotBindCorner.CornerRadius = UDim.new(0, 4)
    aimbotBindCorner.Parent = aimbotBindBtn
    
    aimbotBindBtn.MouseButton1Click:Connect(function()
        startKeyBinding(aimbotBindBtn, "aimbot", settings.AimbotToggleKey)
    end)
    
    local weaponStatus = Instance.new("TextLabel")
    weaponStatus.Size = UDim2.new(1, -20, 0, 30)
    weaponStatus.Position = UDim2.new(0, 10, 0, 115)
    weaponStatus.Text = "🔫 WEAPON AIM: " .. (settings.WeaponAimbotEnabled and "✅ ON" or "❌ OFF")
    weaponStatus.TextColor3 = settings.WeaponAimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    weaponStatus.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    weaponStatus.BackgroundTransparency = 0.5
    weaponStatus.Font = Enum.Font.Gotham
    weaponStatus.TextSize = 12
    weaponStatus.TextXAlignment = Enum.TextXAlignment.Left
    weaponStatus.Parent = mainFrame
    
    local weaponStatusCorner = Instance.new("UICorner")
    weaponStatusCorner.CornerRadius = UDim.new(0, 6)
    weaponStatusCorner.Parent = weaponStatus
    
    local weaponToggle = Instance.new("TextButton")
    weaponToggle.Size = UDim2.new(0, 80, 0, 25)
    weaponToggle.Position = UDim2.new(1, -90, 0, 117)
    weaponToggle.Text = "TOGGLE"
    weaponToggle.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    weaponToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
    weaponToggle.Font = Enum.Font.Gotham
    weaponToggle.TextSize = 11
    weaponToggle.Parent = mainFrame
    
    local weaponToggleCorner = Instance.new("UICorner")
    weaponToggleCorner.CornerRadius = UDim.new(0, 4)
    weaponToggleCorner.Parent = weaponToggle
    
    weaponToggle.MouseButton1Click:Connect(function()
        settings.WeaponAimbotEnabled = not settings.WeaponAimbotEnabled
        weaponStatus.Text = "🔫 WEAPON AIM: " .. (settings.WeaponAimbotEnabled and "✅ ON" or "❌ OFF")
        weaponStatus.TextColor3 = settings.WeaponAimbotEnabled and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    end)
    
    local aimKeyLabel = Instance.new("TextLabel")
    aimKeyLabel.Size = UDim2.new(0, 120, 0, 25)
    aimKeyLabel.Position = UDim2.new(0, 10, 0, 155)
    aimKeyLabel.Text = "🔫 Tecla de Mira:"
    aimKeyLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    aimKeyLabel.BackgroundTransparency = 1
    aimKeyLabel.Font = Enum.Font.Gotham
    aimKeyLabel.TextSize = 11
    aimKeyLabel.TextXAlignment = Enum.TextXAlignment.Left
    aimKeyLabel.Parent = mainFrame
    
    local aimBindBtn = Instance.new("TextButton")
    aimBindBtn.Size = UDim2.new(0, 100, 0, 25)
    aimBindBtn.Position = UDim2.new(1, -110, 0, 153)
    aimBindBtn.Text = (keyNames[settings.AimKey] or settings.AimKey)
    aimBindBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    aimBindBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    aimBindBtn.Font = Enum.Font.Gotham
    aimBindBtn.TextSize = 10
    aimBindBtn.Parent = mainFrame
    
    local aimBindCorner = Instance.new("UICorner")
    aimBindCorner.CornerRadius = UDim.new(0, 4)
    aimBindCorner.Parent = aimBindBtn
    
    aimBindBtn.MouseButton1Click:Connect(function()
        startKeyBinding(aimBindBtn, "aim", settings.AimKey)
    end)
    
    local sep = Instance.new("Frame")
    sep.Size = UDim2.new(1, -20, 0, 2)
    sep.Position = UDim2.new(0, 10, 0, 188)
    sep.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
    sep.BackgroundTransparency = 0.5
    sep.Parent = mainFrame
    
    local partLabel = Instance.new("TextLabel")
    partLabel.Size = UDim2.new(1, -20, 0, 25)
    partLabel.Position = UDim2.new(0, 10, 0, 198)
    partLabel.Text = "🎯 MIRAR EM:"
    partLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    partLabel.BackgroundTransparency = 1
    partLabel.Font = Enum.Font.GothamBold
    partLabel.TextSize = 12
    partLabel.TextXAlignment = Enum.TextXAlignment.Left
    partLabel.Parent = mainFrame
    
    local partValue = Instance.new("TextLabel")
    partValue.Size = UDim2.new(0, 100, 0, 25)
    partValue.Position = UDim2.new(1, -110, 0, 198)
    partValue.Text = settings.TargetPart
    partValue.TextColor3 = Color3.fromRGB(0, 162, 255)
    partValue.BackgroundTransparency = 1
    partValue.Font = Enum.Font.GothamBold
    partValue.TextSize = 12
    partValue.TextXAlignment = Enum.TextXAlignment.Right
    partValue.Parent = mainFrame
    
    local partDropdown = Instance.new("TextButton")
    partDropdown.Size = UDim2.new(0, 30, 0, 25)
    partDropdown.Position = UDim2.new(1, -30, 0, 198)
    partDropdown.Text = "▼"
    partDropdown.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    partDropdown.TextColor3 = Color3.fromRGB(255, 255, 255)
    partDropdown.Font = Enum.Font.Gotham
    partDropdown.TextSize = 11
    partDropdown.Parent = mainFrame
    
    local dropdownCorner = Instance.new("UICorner")
    dropdownCorner.CornerRadius = UDim.new(0, 4)
    dropdownCorner.Parent = partDropdown
    
    local dropdownOpen = false
    local dropdownList = nil
    
    partDropdown.MouseButton1Click:Connect(function()
        if dropdownOpen then
            if dropdownList then dropdownList:Destroy() end
            dropdownOpen = false
            return
        end
        
        dropdownList = Instance.new("Frame")
        dropdownList.Size = UDim2.new(0, 120, 0, 100)
        dropdownList.Position = UDim2.new(1, -130, 0, 226)
        dropdownList.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
        dropdownList.BorderSizePixel = 1
        dropdownList.BorderColor3 = Color3.fromRGB(0, 162, 255)
        dropdownList.ClipsDescendants = true
        dropdownList.Parent = mainFrame
        
        local listCorner = Instance.new("UICorner")
        listCorner.CornerRadius = UDim.new(0, 6)
        listCorner.Parent = dropdownList
        
        for i, part in ipairs(bodyParts) do
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(1, 0, 0, 25)
            btn.Position = UDim2.new(0, 0, 0, (i-1)*25)
            btn.Text = part
            btn.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
            btn.Font = Enum.Font.Gotham
            btn.TextSize = 11
            btn.Parent = dropdownList
            
            btn.MouseButton1Click:Connect(function()
                settings.TargetPart = part
                partValue.Text = part
                dropdownList:Destroy()
                dropdownOpen = false
            end)
        end
        dropdownOpen = true
    end)
    
    local modeLabel = Instance.new("TextLabel")
    modeLabel.Size = UDim2.new(1, -20, 0, 25)
    modeLabel.Position = UDim2.new(0, 10, 0, 231)
    modeLabel.Text = "🎮 MODO:"
    modeLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    modeLabel.BackgroundTransparency = 1
    modeLabel.Font = Enum.Font.GothamBold
    modeLabel.TextSize = 12
    modeLabel.TextXAlignment = Enum.TextXAlignment.Left
    modeLabel.Parent = mainFrame
    
    local modeValue = Instance.new("TextLabel")
    modeValue.Size = UDim2.new(0, 80, 0, 25)
    modeValue.Position = UDim2.new(1, -90, 0, 231)
    modeValue.Text = settings.TargetMode
    modeValue.TextColor3 = Color3.fromRGB(0, 162, 255)
    modeValue.BackgroundTransparency = 1
    modeValue.Font = Enum.Font.GothamBold
    modeValue.TextSize = 12
    modeValue.TextXAlignment = Enum.TextXAlignment.Right
    modeValue.Parent = mainFrame
    
    local modeBtn = Instance.new("TextButton")
    modeBtn.Size = UDim2.new(0, 30, 0, 25)
    modeBtn.Position = UDim2.new(1, -30, 0, 231)
    modeBtn.Text = "▼"
    modeBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    modeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    modeBtn.Font = Enum.Font.Gotham
    modeBtn.TextSize = 11
    modeBtn.Parent = mainFrame
    
    modeBtn.MouseButton1Click:Connect(function()
        settings.TargetMode = (settings.TargetMode == "Auto" and "Manual" or "Auto")
        modeValue.Text = settings.TargetMode
        if settings.TargetMode == "Manual" then
            settings.SelectedPlayer = nil
        end
        updatePlayerList(playerListFrame)
    end)
    
    local playerListFrame = Instance.new("ScrollingFrame")
    playerListFrame.Size = UDim2.new(1, -20, 0, 100)
    playerListFrame.Position = UDim2.new(0, 10, 0, 266)
    playerListFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    playerListFrame.BackgroundTransparency = 0.5
    playerListFrame.BorderSizePixel = 1
    playerListFrame.BorderColor3 = Color3.fromRGB(60, 60, 70)
    playerListFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    playerListFrame.ScrollBarThickness = 6
    playerListFrame.Parent = mainFrame
    
    local listCorner = Instance.new("UICorner")
    listCorner.CornerRadius = UDim.new(0, 6)
    listCorner.Parent = playerListFrame
    
    updatePlayerList(playerListFrame)
    
    local smoothLabel = Instance.new("TextLabel")
    smoothLabel.Size = UDim2.new(1, -20, 0, 20)
    smoothLabel.Position = UDim2.new(0, 10, 0, 376)
    smoothLabel.Text = "⚙️ CAMERA SMOOTH:"
    smoothLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    smoothLabel.BackgroundTransparency = 1
    smoothLabel.Font = Enum.Font.Gotham
    smoothLabel.TextSize = 11
    smoothLabel.TextXAlignment = Enum.TextXAlignment.Left
    smoothLabel.Parent = mainFrame
    
    local smoothValue = Instance.new("TextLabel")
    smoothValue.Size = UDim2.new(0, 60, 0, 20)
    smoothValue.Position = UDim2.new(1, -70, 0, 376)
    smoothValue.Text = string.format("%.2f", settings.Smoothness)
    smoothValue.TextColor3 = Color3.fromRGB(0, 162, 255)
    smoothValue.BackgroundTransparency = 1
    smoothValue.Font = Enum.Font.GothamBold
    smoothValue.TextSize = 11
    smoothValue.TextXAlignment = Enum.TextXAlignment.Right
    smoothValue.Parent = mainFrame
    
    local smoothSlider = Instance.new("TextBox")
    smoothSlider.Size = UDim2.new(0, 150, 0, 20)
    smoothSlider.Position = UDim2.new(1, -160, 0, 374)
    smoothSlider.Text = tostring(settings.Smoothness)
    smoothSlider.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    smoothSlider.TextColor3 = Color3.fromRGB(255, 255, 255)
    smoothSlider.Font = Enum.Font.Gotham
    smoothSlider.TextSize = 10
    smoothSlider.Parent = mainFrame
    
    smoothSlider.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            local num = tonumber(smoothSlider.Text)
            if num and num >= 0.05 and num <= 0.8 then
                settings.Smoothness = num
                smoothValue.Text = string.format("%.2f", settings.Smoothness)
            end
            smoothSlider.Text = tostring(settings.Smoothness)
        end
    end)
    
    local weaponSmoothLabel = Instance.new("TextLabel")
    weaponSmoothLabel.Size = UDim2.new(1, -20, 0, 20)
    weaponSmoothLabel.Position = UDim2.new(0, 10, 0, 404)
    weaponSmoothLabel.Text = "🔫 WEAPON SMOOTH:"
    weaponSmoothLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    weaponSmoothLabel.BackgroundTransparency = 1
    weaponSmoothLabel.Font = Enum.Font.Gotham
    weaponSmoothLabel.TextSize = 11
    weaponSmoothLabel.TextXAlignment = Enum.TextXAlignment.Left
    weaponSmoothLabel.Parent = mainFrame
    
    local weaponSmoothValue = Instance.new("TextLabel")
    weaponSmoothValue.Size = UDim2.new(0, 60, 0, 20)
    weaponSmoothValue.Position = UDim2.new(1, -70, 0, 404)
    weaponSmoothValue.Text = string.format("%.2f", settings.WeaponSmoothness)
    weaponSmoothValue.TextColor3 = Color3.fromRGB(0, 162, 255)
    weaponSmoothValue.BackgroundTransparency = 1
    weaponSmoothValue.Font = Enum.Font.GothamBold
    weaponSmoothValue.TextSize = 11
    weaponSmoothValue.TextXAlignment = Enum.TextXAlignment.Right
    weaponSmoothValue.Parent = mainFrame
    
    local weaponSmoothSlider = Instance.new("TextBox")
    weaponSmoothSlider.Size = UDim2.new(0, 150, 0, 20)
    weaponSmoothSlider.Position = UDim2.new(1, -160, 0, 402)
    weaponSmoothSlider.Text = tostring(settings.WeaponSmoothness)
    weaponSmoothSlider.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    weaponSmoothSlider.TextColor3 = Color3.fromRGB(255, 255, 255)
    weaponSmoothSlider.Font = Enum.Font.Gotham
    weaponSmoothSlider.TextSize = 10
    weaponSmoothSlider.Parent = mainFrame
    
    weaponSmoothSlider.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            local num = tonumber(weaponSmoothSlider.Text)
            if num and num >= 0.05 and num <= 0.8 then
                settings.WeaponSmoothness = num
                weaponSmoothValue.Text = string.format("%.2f", settings.WeaponSmoothness)
            end
            weaponSmoothSlider.Text = tostring(settings.WeaponSmoothness)
        end
    end)
    
    local fovLabel = Instance.new("TextLabel")
    fovLabel.Size = UDim2.new(1, -20, 0, 20)
    fovLabel.Position = UDim2.new(0, 10, 0, 432)
    fovLabel.Text = "👁️ FOV RAIO:"
    fovLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    fovLabel.BackgroundTransparency = 1
    fovLabel.Font = Enum.Font.Gotham
    fovLabel.TextSize = 11
    fovLabel.TextXAlignment = Enum.TextXAlignment.Left
    fovLabel.Parent = mainFrame
    
    local fovValue = Instance.new("TextLabel")
    fovValue.Size = UDim2.new(0, 60, 0, 20)
    fovValue.Position = UDim2.new(1, -70, 0, 432)
    fovValue.Text = tostring(settings.FOVRadius)
    fovValue.TextColor3 = Color3.fromRGB(0, 162, 255)
    fovValue.BackgroundTransparency = 1
    fovValue.Font = Enum.Font.GothamBold
    fovValue.TextSize = 11
    fovValue.TextXAlignment = Enum.TextXAlignment.Right
    fovValue.Parent = mainFrame
    
    local fovSlider = Instance.new("TextBox")
    fovSlider.Size = UDim2.new(0, 150, 0, 20)
    fovSlider.Position = UDim2.new(1, -160, 0, 430)
    fovSlider.Text = tostring(settings.FOVRadius)
    fovSlider.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    fovSlider.TextColor3 = Color3.fromRGB(255, 255, 255)
    fovSlider.Font = Enum.Font.Gotham
    fovSlider.TextSize = 10
    fovSlider.Parent = mainFrame
    
    fovSlider.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            local num = tonumber(fovSlider.Text)
            if num and num >= 0 and num <= 800 then
                settings.FOVRadius = num
                fovValue.Text = tostring(settings.FOVRadius)
                updateFOVCircle()
            end
            fovSlider.Text = tostring(settings.FOVRadius)
        end
    end)
    
    local fovCircleBtn = Instance.new("TextButton")
    fovCircleBtn.Name = "FOVCircleBtn"
    fovCircleBtn.Size = UDim2.new(0, 100, 0, 28)
    fovCircleBtn.Position = UDim2.new(0, 10, 0, 460)
    fovCircleBtn.Text = settings.ShowFOVCircle and "🔘 FOV: ON" or "⚪ FOV: OFF"
    fovCircleBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    fovCircleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    fovCircleBtn.Font = Enum.Font.Gotham
    fovCircleBtn.TextSize = 11
    fovCircleBtn.Parent = mainFrame
    
    local fovBtnCorner = Instance.new("UICorner")
    fovBtnCorner.CornerRadius = UDim.new(0, 6)
    fovBtnCorner.Parent = fovCircleBtn
    
    fovCircleBtn.MouseButton1Click:Connect(function()
        settings.ShowFOVCircle = not settings.ShowFOVCircle
        fovCircleBtn.Text = settings.ShowFOVCircle and "🔘 FOV: ON" or "⚪ FOV: OFF"
        updateFOVCircle()
    end)
    
    local espBtn = Instance.new("TextButton")
    espBtn.Name = "ESPToggleBtn"
    espBtn.Size = UDim2.new(0, 100, 0, 28)
    espBtn.Position = UDim2.new(1, -110, 0, 460)
    espBtn.Text = settings.ESPEnabled and "👁️ ESP: ON" or "👁️ ESP: OFF"
    espBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    espBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    espBtn.Font = Enum.Font.Gotham
    espBtn.TextSize = 11
    espBtn.Parent = mainFrame
    
    local espBtnCorner = Instance.new("UICorner")
    espBtnCorner.CornerRadius = UDim.new(0, 6)
    espBtnCorner.Parent = espBtn
    
    espBtn.MouseButton1Click:Connect(function()
        settings.ESPEnabled = not settings.ESPEnabled
        espBtn.Text = settings.ESPEnabled and "👁️ ESP: ON" or "👁️ ESP: OFF"
        if not settings.ESPEnabled then
            clearESP()
        end
    end)
    
    local espBindBtn = Instance.new("TextButton")
    espBindBtn.Size = UDim2.new(0, 80, 0, 20)
    espBindBtn.Position = UDim2.new(1, -110, 0, 492)
    espBindBtn.Text = "Bind: " .. (keyNames[settings.ESPToggleKey] or settings.ESPToggleKey)
    espBindBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    espBindBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    espBindBtn.Font = Enum.Font.Gotham
    espBindBtn.TextSize = 9
    espBindBtn.Parent = mainFrame
    
    local espBindCorner = Instance.new("UICorner")
    espBindCorner.CornerRadius = UDim.new(0, 4)
    espBindCorner.Parent = espBindBtn
    
    espBindBtn.MouseButton1Click:Connect(function()
        startKeyBinding(espBindBtn, "esp", settings.ESPToggleKey)
    end)
    
    local uiBindBtn = Instance.new("TextButton")
    uiBindBtn.Size = UDim2.new(0, 80, 0, 20)
    uiBindBtn.Position = UDim2.new(1, -110, 0, 516)
    uiBindBtn.Text = "UI: " .. (keyNames[settings.UIToggleKey] or settings.UIToggleKey)
    uiBindBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    uiBindBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    uiBindBtn.Font = Enum.Font.Gotham
    uiBindBtn.TextSize = 9
    uiBindBtn.Parent = mainFrame
    
    local uiBindCorner = Instance.new("UICorner")
    uiBindCorner.CornerRadius = UDim.new(0, 4)
    uiBindCorner.Parent = uiBindBtn
    
    uiBindBtn.MouseButton1Click:Connect(function()
        startKeyBinding(uiBindBtn, "ui", settings.UIToggleKey)
    end)
    
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 80, 0, 30)
    closeBtn.Position = UDim2.new(0.5, -40, 1, -40)
    closeBtn.Text = "FECHAR"
    closeBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.TextSize = 12
    closeBtn.Parent = mainFrame
    
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 6)
    closeCorner.Parent = closeBtn
    
    closeBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = false
    end)
    
    local openBtn = Instance.new("TextButton")
    openBtn.Size = UDim2.new(0, 50, 0, 50)
    openBtn.Position = UDim2.new(0, 10, 1, -60)
    openBtn.Text = "🐍"
    openBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    openBtn.Font = Enum.Font.GothamBold
    openBtn.TextSize = 24
    openBtn.Parent = screenGui
    
    local openCorner = Instance.new("UICorner")
    openCorner.CornerRadius = UDim.new(1, 0)
    openCorner.Parent = openBtn
    
    openBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = true
    end)
    
    return screenGui
end

local function updateFOVCircle()
    safeDestroy(fovCircle)
    if not settings.ShowFOVCircle or settings.FOVRadius <= 0 then return end
    
    local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
    if not screenGui then return end
    
    local circle = Instance.new("Frame")
    circle.Size = UDim2.new(0, settings.FOVRadius * 2, 0, settings.FOVRadius * 2)
    circle.Position = UDim2.new(0.5, -settings.FOVRadius, 0.5, -settings.FOVRadius)
    circle.BackgroundTransparency = 1
    circle.BorderSizePixel = 2
    circle.BorderColor3 = Color3.fromRGB(255, 255, 255)
    circle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    circle.BackgroundTransparency = 0.9
    circle.Parent = screenGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = circle
    
    fovCircle = circle
end

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
    billboard.Size = UDim2.new(0, 200, 0, 80)
    billboard.StudsOffset = Vector3.new(0, 2.5, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = head
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 1, 0)
    frame.BackgroundTransparency = 1
    frame.Parent = billboard
    
    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, 0, 0, 22)
    nameLabel.Position = UDim2.new(0, 0, 0, 0)
    nameLabel.Text = player.Name
    nameLabel.TextColor3 = settings.ESPTextColor
    nameLabel.BackgroundTransparency = 1
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = 14
    nameLabel.TextStrokeTransparency = 0.3
    nameLabel.Parent = frame
    
    local healthLabel = Instance.new("TextLabel")
    healthLabel.Size = UDim2.new(1, 0, 0, 18)
    healthLabel.Position = UDim2.new(0, 0, 0, 22)
    healthLabel.Text = "❤️ " .. math.floor(humanoid.Health)
    healthLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
    healthLabel.BackgroundTransparency = 1
    healthLabel.Font = Enum.Font.Gotham
    healthLabel.TextSize = 12
    healthLabel.Parent = frame
    
    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Size = UDim2.new(1, 0, 0, 18)
    distanceLabel.Position = UDim2.new(0, 0, 0, 40)
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
    
    local localChar = LocalPlayer.Character
    local localPos = localChar and localChar:FindFirstChild("HumanoidRootPart") and localChar.HumanoidRootPart.Position or Vector3.zero
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local character = player.Character
            local humanoid = character and character:FindFirstChild("Humanoid")
            
            if character and humanoid and humanoid.Health > 0 then
                if not espObjects[player] then
                    createESP(player)
                else
                    local rootPart = character:FindFirstChild("HumanoidRootPart")
                    
                    if espObjects[player].HealthLabel and humanoid then
                        espObjects[player].HealthLabel.Text = "❤️ " .. math.floor(humanoid.Health)
                    end
                    
                    if espObjects[player].DistanceLabel and rootPart and localPos then
                        local dist = (rootPart.Position - localPos).Magnitude
                        espObjects[player].DistanceLabel.Text = "📏 " .. math.floor(dist) .. "m"
                    end
                end
            elseif espObjects[player] then
                safeDestroy(espObjects[player].Billboard)
                safeDestroy(espObjects[player].Highlight)
                espObjects[player] = nil
            end
        end
    end
end

local function clearESP()
    for player, objects in pairs(espObjects) do
        pcall(function()
            safeDestroy(objects.Billboard)
            safeDestroy(objects.Highlight)
        end)
    end
    espObjects = {}
end

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
            local humanoid = character and character:FindFirstChild("Humanoid")
            
            if character and humanoid and humanoid.Health > 0 then
                
                if settings.TargetMode == "Manual" and settings.SelectedPlayer then
                    if player.Name == settings.SelectedPlayer then
                        return player
                    else
                        goto continue
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
        
        ::continue::
    end
    
    return bestPlayer
end

local function updateCameraAimbot()
    if not settings.AimbotEnabled then return end
    
    local target = getBestTarget()
    if not target then return end
    
    local character = target.Character
    if not character then return end
    
    local targetPart = getTargetPart(character)
    if not targetPart or not targetPart.Parent then return end
    
    local camera = workspace.CurrentCamera
    local targetPosition = targetPart.Position
    local currentCFrame = camera.CFrame
    
    local newCFrame = CFrame.new(currentCFrame.Position, targetPosition)
    local smoothedCFrame = currentCFrame:Lerp(newCFrame, 1 - settings.Smoothness)
    camera.CFrame = smoothedCFrame
end

function loadHub()
    createUI()
    updateFOVCircle()
    hubLoaded = true
    
    RunService.RenderStepped:Connect(function()
        if not hubLoaded then return end
        
        pcall(function()
            updateCameraAimbot()
            updateESP()
            
            local newWeapon = getCurrentWeapon()
            if newWeapon and (not currentWeapon or currentWeapon.Tool ~= newWeapon.Tool) then
                currentWeapon = newWeapon
            elseif not newWeapon and currentWeapon then
                currentWeapon = nil
            end
            
            if isAiming and currentWeapon and settings.WeaponAimbotEnabled then
                updateWeaponAim()
            end
        end)
    end)
    
    Players.PlayerAdded:Connect(function()
        local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
        if screenGui then
            local mainFrame = screenGui:FindFirstChild("MainFrame")
            if mainFrame then
                local playerListFrame = mainFrame:FindFirstChildOfClass("ScrollingFrame")
                if playerListFrame then
                    updatePlayerList(playerListFrame)
                end
            end
        end
    end)
    
    Players.PlayerRemoving:Connect(function(player)
        local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
        if screenGui then
            local mainFrame = screenGui:FindFirstChild("MainFrame")
            if mainFrame then
                local playerListFrame = mainFrame:FindFirstChildOfClass("ScrollingFrame")
                if playerListFrame then
                    updatePlayerList(playerListFrame)
                end
            end
        end
        
        if espObjects[player] then
            pcall(function()
                safeDestroy(espObjects[player].Billboard)
                safeDestroy(espObjects[player].Highlight)
            end)
            espObjects[player] = nil
        end
    end)
end

createKeyScreen()
