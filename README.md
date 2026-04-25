local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = workspace.CurrentCamera

local REQUIRED_KEY = "Essaeakey"
local keyInserted = false
local hubLoaded = false

local settings = {
    AimbotEnabled = true,
    ESPEnabled = true,
    WeaponAimbotEnabled = true,
    AutoFarmMoney = false,
    AutoFarmLevel = false,
    FlyEnabled = false,
    NoclipEnabled = false,
    TargetPart = "Head",
    TargetMode = "Auto",
    SelectedPlayer = nil,
    Smoothness = 0.25,
    WeaponSmoothness = 0.15,
    FOVRadius = 300,
    ShowFOVCircle = true,
    ESPBoxColor = Color3.fromRGB(0, 162, 255),
    ESPTextColor = Color3.fromRGB(255, 255, 255),
    FlySpeed = 50,
    AimbotKey = Enum.KeyCode.E,
    AimbotKeyName = "E",
    FlyKey = Enum.KeyCode.G,
    FlyKeyName = "G",
    NoclipKey = Enum.KeyCode.N,
    NoclipKeyName = "N",
    TeleportGunKey = Enum.KeyCode.T,
    TeleportGunKeyName = "T"
}

local bodyParts = {"Head", "UpperTorso", "LowerTorso", "HumanoidRootPart"}
local currentTarget = nil
local espObjects = {}
local fovCircle = nil
local isAiming = false
local currentWeapon = nil
local aimbotHeld = false
local flyBodyGyro = nil
local flyBodyVelocity = nil
local noclipConnections = {}
local farmConnections = {}
local gunList = {}
local collectedGuns = {}
local currentRobberyIndex = 1
local robberyRoute = {}
local farmPaused = false

local madCityChapter1Robberies = {
    {Name = "Bank", Aliases = {"Main Bank", "City Bank", "Bank"}},
    {Name = "Jewelry Store", Aliases = {"Jewelry", "Diamond Store", "Gem Store"}},
    {Name = "Museum", Aliases = {"City Museum", "Art Museum", "Museum"}},
    {Name = "Casino", Aliases = {"Casino", "Lucky Casino", "Casino Building"}},
    {Name = "Gas Station", Aliases = {"Gas Station", "Fuel Station", "Petrol Station", "Convenience Store"}},
    {Name = "Donut Shop", Aliases = {"Donut Shop", "Bakery", "Coffee Shop"}},
    {Name = "Criminal Base", Aliases = {"Criminal Base", "Villain Base", "Evil Lair", "Warehouse"}},
    {Name = "Police Station", Aliases = {"Police Station", "Police HQ", "Sheriff Office"}},
    {Name = "Prison", Aliases = {"Prison", "Jail", "Penitentiary"}},
    {Name = "Power Plant", Aliases = {"Power Plant", "Electric Plant", "Nuclear Plant", "Factory"}},
    {Name = "Hospital", Aliases = {"Hospital", "Medical Center", "Clinic"}},
    {Name = "Nightclub", Aliases = {"Nightclub", "Club", "Dance Club", "Disco"}},
    {Name = "Hero Base", Aliases = {"Hero Base", "Hero HQ", "Hero Tower", "Superhero Base"}},
    {Name = "Train Station", Aliases = {"Train Station", "Subway", "Metro Station"}},
    {Name = "Airport", Aliases = {"Airport", "Airfield", "Landing Strip"}}
}

local function getCurrentWeapon()
    local character = LocalPlayer.Character
    if not character then return nil end
    local tool = character:FindFirstChildOfClass("Tool")
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
        Scope = scope,
        HasScope = scope ~= nil
    }
end

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

local function updateWeaponAim()
    if not settings.WeaponAimbotEnabled then return end
    if not isAiming then return end
    if not settings.AimbotEnabled then return end
    if not aimbotHeld then return end
    local target = getBestTarget()
    if not target then return end
    local character = target.Character
    if not character then return end
    local targetPart = getTargetPart(character)
    if not targetPart then return end
    local weaponAimPos = getWeaponAimPosition()
    if not weaponAimPos then return end
    local currentWeaponCFrame = currentWeapon.Tool:GetPivot()
    local targetCFrame = CFrame.lookAt(weaponAimPos, targetPart.Position)
    local smoothedCFrame = currentWeaponCFrame:Lerp(targetCFrame, 1 - settings.WeaponSmoothness)
    currentWeapon.Tool:SetPrimaryPartCFrame(smoothedCFrame)
    local handle = currentWeapon.Tool:FindFirstChild("Handle")
    if handle then
        local handleTargetCFrame = CFrame.lookAt(handle.Position, targetPart.Position)
        local smoothedHandle = handle.CFrame:Lerp(handleTargetCFrame, 1 - settings.WeaponSmoothness)
        handle.CFrame = smoothedHandle
    end
end

local function buildRobberyRoute()
    robberyRoute = {}
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local rootPart = character.HumanoidRootPart
    for _, robberyData in ipairs(madCityChapter1Robberies) do
        local found = nil
        for _, alias in ipairs(robberyData.Aliases) do
            local obj = workspace:FindFirstChild(alias, true)
            if obj then
                local part = nil
                if obj:IsA("BasePart") then
                    part = obj
                elseif obj:IsA("Model") then
                    part = obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart")
                end
                if part then
                    found = {Name = robberyData.Name, Part = part, Position = part.Position}
                    break
                end
            end
        end
        if not found then
            for _, alias in ipairs(robberyData.Aliases) do
                for _, child in ipairs(workspace:GetDescendants()) do
                    if child:IsA("BasePart") and child.Name:lower():find(alias:lower()) then
                        found = {Name = robberyData.Name, Part = child, Position = child.Position}
                        break
                    end
                end
                if found then break end
            end
        end
        if not found then
            for _, alias in ipairs(robberyData.Aliases) do
                for _, child in ipairs(workspace:GetDescendants()) do
                    if child:IsA("Model") and child.Name:lower():find(alias:lower()) then
                        local part = child.PrimaryPart or child:FindFirstChildWhichIsA("BasePart")
                        if part then
                            found = {Name = robberyData.Name, Part = part, Position = part.Position}
                            break
                        end
                    end
                end
                if found then break end
            end
        end
        if found then
            table.insert(robberyRoute, found)
        end
    end
    if #robberyRoute > 1 then
        local basePos = rootPart.Position
        table.sort(robberyRoute, function(a, b)
            return (a.Position - basePos).Magnitude < (b.Position - basePos).Magnitude
        end)
        local startIdx = 1
        local minDist = math.huge
        for i, loc in ipairs(robberyRoute) do
            local dist = (loc.Position - basePos).Magnitude
            if dist < minDist then
                minDist = dist
                startIdx = i
            end
        end
        local sorted = {}
        local used = {}
        local current = startIdx
        for i = 1, #robberyRoute do
            table.insert(sorted, robberyRoute[current])
            used[current] = true
            local bestNext = nil
            local bestDist = math.huge
            for j = 1, #robberyRoute do
                if not used[j] then
                    local dist = (robberyRoute[current].Position - robberyRoute[j].Position).Magnitude
                    if dist < bestDist then
                        bestDist = dist
                        bestNext = j
                    end
                end
            end
            current = bestNext
            if not current then break end
        end
        for j = 1, #robberyRoute do
            if not used[j] then
                table.insert(sorted, robberyRoute[j])
            end
        end
        robberyRoute = sorted
    end
    currentRobberyIndex = 1
end

local function scanForLootables(robberyPosition)
    local lootables = {}
    local searchRadius = 100
    for _, descendant in ipairs(workspace:GetDescendants()) do
        if descendant:IsA("BasePart") or descendant:IsA("MeshPart") then
            local dist = (descendant.Position - robberyPosition).Magnitude
            if dist <= searchRadius then
                local name = descendant.Name:lower()
                if name:find("money") or name:find("cash") or name:find("gold") or name:find("diamond") or
                   name:find("jewel") or name:find("loot") or name:find("valuable") or name:find("artifact") or
                   name:find("painting") or name:find("sculpture") or name:find("treasure") or name:find("ring") or
                   name:find("necklace") or name:find("bracelet") or name:find("watch") or name:find("gem") or
                   name:find("safe") or name:find("chest") or name:find("briefcase") or name:find("bag") then
                    table.insert(lootables, descendant)
                end
                if descendant:IsA("Model") then
                    local modelName = descendant.Name:lower()
                    if modelName:find("money") or modelName:find("cash") or modelName:find("gold") or modelName:find("loot") then
                        table.insert(lootables, descendant)
                    end
                end
                if descendant:FindFirstChild("ClickDetector") then
                    table.insert(lootables, descendant)
                end
                if descendant.Parent and descendant.Parent:FindFirstChild("ClickDetector") then
                    table.insert(lootables, descendant.Parent)
                end
            end
        end
    end
    return lootables
end

local function robLocation(locationData)
    if not settings.AutoFarmMoney then return end
    if farmPaused then return end
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local rootPart = character.HumanoidRootPart
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end
    humanoid:MoveTo(locationData.Position)
    local arrivalTime = tick()
    local maxWait = 15
    while (rootPart.Position - locationData.Position).Magnitude > 20 do
        if not settings.AutoFarmMoney then return end
        if farmPaused then return end
        if tick() - arrivalTime > maxWait then break end
        humanoid:MoveTo(locationData.Position)
        task.wait(0.3)
    end
    if (rootPart.Position - locationData.Position).Magnitude <= 50 then
        local lootables = scanForLootables(locationData.Position)
        local collectTime = tick()
        local maxCollect = 8
        while tick() - collectTime < maxCollect do
            if not settings.AutoFarmMoney then return end
            if farmPaused then return end
            for _, loot in ipairs(lootables) do
                if loot and loot.Parent then
                    local lootPart = nil
                    if loot:IsA("BasePart") then
                        lootPart = loot
                    elseif loot:IsA("Model") then
                        lootPart = loot.PrimaryPart or loot:FindFirstChildWhichIsA("BasePart")
                    end
                    if lootPart and rootPart then
                        firetouchinterest(rootPart, lootPart, 0)
                        firetouchinterest(rootPart, lootPart, 1)
                    end
                    if loot:FindFirstChild("ClickDetector") then
                        fireclickdetector(loot.ClickDetector)
                    end
                end
            end
            local proximityItems = {}
            for _, descendant in ipairs(workspace:GetDescendants()) do
                if descendant:IsA("BasePart") or descendant:IsA("MeshPart") then
                    local dist = (descendant.Position - rootPart.Position).Magnitude
                    if dist <= 10 then
                        table.insert(proximityItems, descendant)
                    end
                    if descendant:FindFirstChild("ClickDetector") and dist <= 10 then
                        fireclickdetector(descendant.ClickDetector)
                    end
                end
            end
            for _, item in ipairs(proximityItems) do
                firetouchinterest(rootPart, item, 0)
                firetouchinterest(rootPart, item, 1)
            end
            task.wait(0.5)
        end
    end
end

local function startAutoFarmMoney()
    if settings.AutoFarmMoney then
        for _, connection in ipairs(farmConnections) do
            connection:Disconnect()
        end
        farmConnections = {}
        farmPaused = false
        currentRobberyIndex = 1
        buildRobberyRoute()
        local farmConnection
        farmConnection = RunService.Heartbeat:Connect(function()
            if not settings.AutoFarmMoney then
                farmConnection:Disconnect()
                return
            end
            if farmPaused then return end
            if #robberyRoute == 0 then
                buildRobberyRoute()
                if #robberyRoute == 0 then
                    task.wait(5)
                    buildRobberyRoute()
                    return
                end
            end
            if currentRobberyIndex > #robberyRoute then
                currentRobberyIndex = 1
                buildRobberyRoute()
            end
            local currentLocation = robberyRoute[currentRobberyIndex]
            if currentLocation then
                robLocation(currentLocation)
                currentRobberyIndex = currentRobberyIndex + 1
            end
        end)
        table.insert(farmConnections, farmConnection)
    end
end

local function stopAutoFarmMoney()
    settings.AutoFarmMoney = false
    farmPaused = false
    for _, connection in ipairs(farmConnections) do
        connection:Disconnect()
    end
    farmConnections = {}
    robberyRoute = {}
    currentRobberyIndex = 1
end

local function startAutoFarmLevel()
    if not settings.AutoFarmLevel then return end
    for _, connection in ipairs(farmConnections) do
        connection:Disconnect()
    end
    farmConnections = {}
    local levelFarmConnection
    levelFarmConnection = RunService.Heartbeat:Connect(function()
        if not settings.AutoFarmLevel then
            levelFarmConnection:Disconnect()
            return
        end
        local character = LocalPlayer.Character
        if not character or not character:FindFirstChild("HumanoidRootPart") then return end
        local rootPart = character.HumanoidRootPart
        local humanoid = character:FindFirstChild("Humanoid")
        if not humanoid or humanoid.Health <= 0 then return end
        local npcs = {}
        local ignoreNames = {"Civilian", "Citizen", "Pedestrian", "Worker", "Doctor"}
        for _, descendant in ipairs(workspace:GetDescendants()) do
            if descendant:IsA("Model") and descendant:FindFirstChild("Humanoid") and descendant:FindFirstChild("Head") then
                local hum = descendant.Humanoid
                local skip = false
                for _, ignore in ipairs(ignoreNames) do
                    if descendant.Name:lower():find(ignore:lower()) then
                        skip = true
                        break
                    end
                end
                if not skip and hum.Health > 0 and descendant ~= character then
                    local head = descendant:FindFirstChild("Head")
                    if head then
                        local dist = (rootPart.Position - head.Position).Magnitude
                        if dist <= 200 then
                            table.insert(npcs, {Model = descendant, Head = head, Distance = dist, Humanoid = hum})
                        end
                    end
                end
            end
        end
        if #npcs == 0 then return end
        table.sort(npcs, function(a, b) return a.Distance < b.Distance end)
        local target = npcs[1]
        if target and settings.AimbotEnabled then
            local camera = workspace.CurrentCamera
            local targetPosition = target.Head.Position
            local currentCFrame = camera.CFrame
            local newCFrame = CFrame.new(currentCFrame.Position, targetPosition)
            local smoothedCFrame = currentCFrame:Lerp(newCFrame, 1 - settings.Smoothness)
            camera.CFrame = smoothedCFrame
        end
        if target and target.Distance <= 100 then
            humanoid:MoveTo(target.Head.Position)
            if target.Distance <= 15 then
                local tool = character:FindFirstChildOfClass("Tool")
                if tool then
                    tool:Activate()
                    task.wait(0.2)
                    tool:Deactivate()
                end
                local mousePos = target.Head.Position
                local args = {
                    [1] = mousePos,
                    [2] = target.Model
                }
                local remoteEvents = {}
                for _, descendant in ipairs(character:GetDescendants()) do
                    if descendant:IsA("RemoteEvent") and not descendant.Name:lower():find("team") then
                        table.insert(remoteEvents, descendant)
                    end
                end
                for _, re in ipairs(remoteEvents) do
                    pcall(function()
                        re:FireServer(unpack(args))
                    end)
                end
            end
        end
    end)
    table.insert(farmConnections, levelFarmConnection)
end

local function stopAutoFarmLevel()
    settings.AutoFarmLevel = false
    for _, connection in ipairs(farmConnections) do
        connection:Disconnect()
    end
    farmConnections = {}
end

local function scanForGuns()
    gunList = {}
    collectedGuns = {}
    local workspaceItems = workspace:GetChildren()
    for _, item in ipairs(workspaceItems) do
        if item:IsA("Tool") or item:IsA("Model") then
            local isGun = false
            if item:IsA("Tool") then
                if item:FindFirstChild("Handle") then
                    isGun = true
                end
            end
            if not isGun and item:IsA("Model") then
                for _, child in ipairs(item:GetChildren()) do
                    if child.Name:lower():find("gun") or child.Name:lower():find("weapon") or child.Name:lower():find("rifle") or child.Name:lower():find("pistol") or child.Name:lower():find("shotgun") then
                        isGun = true
                        break
                    end
                end
            end
            if isGun then
                table.insert(gunList, item)
            end
        end
    end
    for _, folderName in ipairs({"Weapons", "Guns", "Items", "Loot"}) do
        local folder = workspace:FindFirstChild(folderName)
        if folder then
            for _, item in ipairs(folder:GetChildren()) do
                if item:IsA("Tool") or item:IsA("Model") then
                    table.insert(gunList, item)
                end
            end
        end
    end
end

local function teleportGunToPlayer(gunName)
    scanForGuns()
    local targetGun = nil
    if gunName and gunName ~= "" then
        for _, gun in ipairs(gunList) do
            if gun.Name:lower():find(gunName:lower()) then
                targetGun = gun
                break
            end
        end
    end
    if not targetGun and #gunList > 0 then
        targetGun = gunList[1]
    end
    if targetGun and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local rootPart = LocalPlayer.Character.HumanoidRootPart
        local gunPrimaryPart = nil
        if targetGun:IsA("Tool") then
            gunPrimaryPart = targetGun:FindFirstChild("Handle")
        end
        if not gunPrimaryPart and targetGun:IsA("Model") then
            gunPrimaryPart = targetGun.PrimaryPart or targetGun:FindFirstChild("Handle") or targetGun:FindFirstChildWhichIsA("BasePart")
        end
        if gunPrimaryPart then
            targetGun:SetPrimaryPartCFrame(rootPart.CFrame + rootPart.CFrame.LookVector * 3)
            firetouchinterest(rootPart, gunPrimaryPart, 0)
            firetouchinterest(rootPart, gunPrimaryPart, 1)
            if not collectedGuns[targetGun.Name] then
                collectedGuns[targetGun.Name] = true
            end
        end
    end
end

local function setupFly()
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid then return end
    if flyBodyGyro then flyBodyGyro:Destroy() end
    if flyBodyVelocity then flyBodyVelocity:Destroy() end
    flyBodyGyro = Instance.new("BodyGyro")
    flyBodyGyro.MaxTorque = Vector3.new(400000, 400000, 400000)
    flyBodyGyro.CFrame = character.HumanoidRootPart.CFrame
    flyBodyGyro.Parent = character.HumanoidRootPart
    flyBodyVelocity = Instance.new("BodyVelocity")
    flyBodyVelocity.MaxForce = Vector3.new(400000, 400000, 400000)
    flyBodyVelocity.Velocity = Vector3.new(0, 0, 0)
    flyBodyVelocity.Parent = character.HumanoidRootPart
    humanoid.PlatformStand = true
end

local function disableFly()
    if flyBodyGyro then flyBodyGyro:Destroy() end
    if flyBodyVelocity then flyBodyVelocity:Destroy() end
    flyBodyGyro = nil
    flyBodyVelocity = nil
    local character = LocalPlayer.Character
    if character then
        local humanoid = character:FindFirstChild("Humanoid")
        if humanoid then
            humanoid.PlatformStand = false
        end
    end
end

local function updateFly()
    if not settings.FlyEnabled then return end
    if not flyBodyGyro or not flyBodyVelocity then return end
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local camera = workspace.CurrentCamera
    local rootPart = character.HumanoidRootPart
    flyBodyGyro.CFrame = camera.CFrame
    local velocity = Vector3.new(0, 0, 0)
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then
        velocity = velocity + camera.CFrame.LookVector * settings.FlySpeed
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then
        velocity = velocity - camera.CFrame.LookVector * settings.FlySpeed
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then
        velocity = velocity - camera.CFrame.RightVector * settings.FlySpeed
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then
        velocity = velocity + camera.CFrame.RightVector * settings.FlySpeed
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
        velocity = velocity + Vector3.new(0, settings.FlySpeed, 0)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
        velocity = velocity - Vector3.new(0, settings.FlySpeed, 0)
    end
    flyBodyVelocity.Velocity = velocity
end

local function setupNoclip()
    for _, connection in ipairs(noclipConnections) do
        connection:Disconnect()
    end
    noclipConnections = {}
    local connection = RunService.Stepped:Connect(function()
        if not settings.NoclipEnabled then return end
        local character = LocalPlayer.Character
        if not character then return end
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") and part.CanCollide then
                part.CanCollide = false
            end
        end
    end)
    table.insert(noclipConnections, connection)
end

local function disableNoclip()
    for _, connection in ipairs(noclipConnections) do
        connection:Disconnect()
    end
    noclipConnections = {}
    local character = LocalPlayer.Character
    if character then
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = true
            end
        end
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
    background.BackgroundTransparency = 0.7
    background.Parent = screenGui

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

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 50)
    title.Position = UDim2.new(0, 0, 0, 10)
    title.Text = "SNAKE HUB v2.0"
    title.TextColor3 = Color3.fromRGB(0, 162, 255)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.TextSize = 28
    title.Parent = mainFrame

    local subTitle = Instance.new("TextLabel")
    subTitle.Size = UDim2.new(1, 0, 0, 30)
    subTitle.Position = UDim2.new(0, 0, 0, 55)
    subTitle.Text = "Mad City Chapter 1 Exclusive"
    subTitle.TextColor3 = Color3.fromRGB(180, 180, 200)
    subTitle.BackgroundTransparency = 1
    subTitle.Font = Enum.Font.Gotham
    subTitle.TextSize = 14
    subTitle.Parent = mainFrame

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
            task.wait(2)
            errorMsg.Visible = false
        end
    end)

    keyBox.FocusLost:Connect(function(enterPressed)
        if enterPressed and keyBox.Text == REQUIRED_KEY then
            keyInserted = true
            screenGui:Destroy()
            loadHub()
        elseif enterPressed then
            errorMsg.Text = "KEY INCORRETA! Tente novamente."
            errorMsg.Visible = true
            keyBox.Text = ""
            task.wait(2)
            errorMsg.Visible = false
        end
    end)
end

local function createUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SnakeHubUI"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = LocalPlayer.PlayerGui

    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 340, 0, 580)
    mainFrame.Position = UDim2.new(0, 20, 0.5, -290)
    mainFrame.BackgroundColor3 = Color3.fromRGB(18, 20, 30)
    mainFrame.BackgroundTransparency = 0.03
    mainFrame.BorderSizePixel = 0
    mainFrame.Parent = screenGui

    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 12)
    mainCorner.Parent = mainFrame

    local gradient = Instance.new("UIGradient")
    gradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 120, 255)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(100, 0, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 200, 255))
    })
    gradient.Rotation = 45
    gradient.Parent = mainFrame

    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 55)
    titleBar.Position = UDim2.new(0, 0, 0, 0)
    titleBar.BackgroundColor3 = Color3.fromRGB(12, 14, 22)
    titleBar.BorderSizePixel = 0
    titleBar.BackgroundTransparency = 0.5
    titleBar.Parent = mainFrame

    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 12)
    titleCorner.Parent = titleBar

    local titleGradient = Instance.new("UIGradient")
    titleGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 162, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 220, 255))
    })
    titleGradient.Parent = titleBar

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 1, 0)
    title.Text = "SNAKE HUB v2.0"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBold
    title.TextSize = 22
    title.Parent = titleBar

    local version = Instance.new("TextLabel")
    version.Size = UDim2.new(0, 80, 0, 20)
    version.Position = UDim2.new(1, -85, 0, 30)
    version.Text = "Mad City Ch.1"
    version.TextColor3 = Color3.fromRGB(150, 200, 255)
    version.BackgroundTransparency = 1
    version.Font = Enum.Font.Gotham
    version.TextSize = 10
    version.Parent = titleBar

    local tabFrame = Instance.new("Frame")
    tabFrame.Size = UDim2.new(1, 0, 0, 40)
    tabFrame.Position = UDim2.new(0, 0, 0, 58)
    tabFrame.BackgroundColor3 = Color3.fromRGB(22, 24, 36)
    tabFrame.BorderSizePixel = 0
    tabFrame.Parent = mainFrame

    local tabs = {}
    local tabNames = {"AIMBOT", "FARM", "MOVEMENT", "VISUALS", "WEAPONS", "KEYS"}
    local activeTab = "AIMBOT"
    local tabContents = {}

    for i, tabName in ipairs(tabNames) do
        local tab = Instance.new("TextButton")
        tab.Size = UDim2.new(1/#tabNames, -4, 0, 35)
        tab.Position = UDim2.new((i-1)/#tabNames, 2, 0, 2)
        tab.Text = tabName
        tab.BackgroundColor3 = Color3.fromRGB(30, 32, 45)
        tab.TextColor3 = Color3.fromRGB(180, 180, 200)
        tab.Font = Enum.Font.GothamBold
        tab.TextSize = 11
        tab.BorderSizePixel = 0
        tab.Parent = tabFrame

        local tabCorner = Instance.new("UICorner")
        tabCorner.CornerRadius = UDim.new(0, 6)
        tabCorner.Parent = tab

        tab.MouseButton1Click:Connect(function()
            for _, t in ipairs(tabs) do
                t.BackgroundColor3 = Color3.fromRGB(30, 32, 45)
                t.TextColor3 = Color3.fromRGB(180, 180, 200)
            end
            tab.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
            tab.TextColor3 = Color3.fromRGB(255, 255, 255)
            activeTab = tabName
            for name, contentFrame in pairs(tabContents) do
                contentFrame.Visible = (name == tabName)
            end
        end)

        table.insert(tabs, tab)
    end
    tabs[1].BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    tabs[1].TextColor3 = Color3.fromRGB(255, 255, 255)

    local contentArea = Instance.new("Frame")
    contentArea.Size = UDim2.new(1, -20, 1, -110)
    contentArea.Position = UDim2.new(0, 10, 0, 100)
    contentArea.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
    contentArea.BorderSizePixel = 0
    contentArea.BackgroundTransparency = 0.5
    contentArea.Parent = mainFrame

    local contentCorner = Instance.new("UICorner")
    contentCorner.CornerRadius = UDim.new(0, 8)
    contentCorner.Parent = contentArea

    local aimbotFrame = Instance.new("Frame")
    aimbotFrame.Name = "AIMBOT"
    aimbotFrame.Size = UDim2.new(1, 0, 1, 0)
    aimbotFrame.BackgroundTransparency = 1
    aimbotFrame.Parent = contentArea
    tabContents["AIMBOT"] = aimbotFrame

    local farmFrame = Instance.new("Frame")
    farmFrame.Name = "FARM"
    farmFrame.Size = UDim2.new(1, 0, 1, 0)
    farmFrame.BackgroundTransparency = 1
    farmFrame.Visible = false
    farmFrame.Parent = contentArea
    tabContents["FARM"] = farmFrame

    local movementFrame = Instance.new("Frame")
    movementFrame.Name = "MOVEMENT"
    movementFrame.Size = UDim2.new(1, 0, 1, 0)
    movementFrame.BackgroundTransparency = 1
    movementFrame.Visible = false
    movementFrame.Parent = contentArea
    tabContents["MOVEMENT"] = movementFrame

    local visualsFrame = Instance.new("Frame")
    visualsFrame.Name = "VISUALS"
    visualsFrame.Size = UDim2.new(1, 0, 1, 0)
    visualsFrame.BackgroundTransparency = 1
    visualsFrame.Visible = false
    visualsFrame.Parent = contentArea
    tabContents["VISUALS"] = visualsFrame

    local weaponsFrame = Instance.new("Frame")
    weaponsFrame.Name = "WEAPONS"
    weaponsFrame.Size = UDim2.new(1, 0, 1, 0)
    weaponsFrame.BackgroundTransparency = 1
    weaponsFrame.Visible = false
    weaponsFrame.Parent = contentArea
    tabContents["WEAPONS"] = weaponsFrame

    local keysFrame = Instance.new("Frame")
    keysFrame.Name = "KEYS"
    keysFrame.Size = UDim2.new(1, 0, 1, 0)
    keysFrame.BackgroundTransparency = 1
    keysFrame.Visible = false
    keysFrame.Parent = contentArea
    tabContents["KEYS"] = keysFrame

    local function createToggle(parent, y, text, default, callback)
        local toggleFrame = Instance.new("Frame")
        toggleFrame.Size = UDim2.new(1, -20, 0, 40)
        toggleFrame.Position = UDim2.new(0, 10, 0, y)
        toggleFrame.BackgroundColor3 = Color3.fromRGB(35, 38, 50)
        toggleFrame.BackgroundTransparency = 0.3
        toggleFrame.BorderSizePixel = 0
        toggleFrame.Parent = parent

        local toggleCorner = Instance.new("UICorner")
        toggleCorner.CornerRadius = UDim.new(0, 8)
        toggleCorner.Parent = toggleFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0.6, 0, 1, 0)
        label.Position = UDim2.new(0, 10, 0, 0)
        label.Text = text
        label.TextColor3 = Color3.fromRGB(220, 220, 230)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 13
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = toggleFrame

        local toggleBtn = Instance.new("TextButton")
        toggleBtn.Size = UDim2.new(0, 50, 0, 24)
        toggleBtn.Position = UDim2.new(1, -60, 0.5, -12)
        toggleBtn.Text = ""
        toggleBtn.BackgroundColor3 = default and Color3.fromRGB(0, 200, 100) or Color3.fromRGB(80, 80, 90)
        toggleBtn.BorderSizePixel = 0
        toggleBtn.Parent = toggleFrame

        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 12)
        btnCorner.Parent = toggleBtn

        local circle = Instance.new("Frame")
        circle.Size = UDim2.new(0, 18, 0, 18)
        circle.Position = default and UDim2.new(1, -21, 0.5, -9) or UDim2.new(0, 3, 0.5, -9)
        circle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        circle.BorderSizePixel = 0
        circle.Parent = toggleBtn

        local circleCorner = Instance.new("UICorner")
        circleCorner.CornerRadius = UDim.new(0, 9)
        circleCorner.Parent = circle

        local toggled = default
        toggleBtn.MouseButton1Click:Connect(function()
            toggled = not toggled
            if toggled then
                toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 100)
                circle:TweenPosition(UDim2.new(1, -21, 0.5, -9), "Out", "Quad", 0.15, true)
            else
                toggleBtn.BackgroundColor3 = Color3.fromRGB(80, 80, 90)
                circle:TweenPosition(UDim2.new(0, 3, 0.5, -9), "Out", "Quad", 0.15, true)
            end
            callback(toggled)
        end)

        return toggleFrame
    end

    local function createDropdown(parent, y, text, options, default, callback)
        local dropdownFrame = Instance.new("Frame")
        dropdownFrame.Size = UDim2.new(1, -20, 0, 40)
        dropdownFrame.Position = UDim2.new(0, 10, 0, y)
        dropdownFrame.BackgroundColor3 = Color3.fromRGB(35, 38, 50)
        dropdownFrame.BackgroundTransparency = 0.3
        dropdownFrame.BorderSizePixel = 0
        dropdownFrame.Parent = parent

        local dropdownCorner = Instance.new("UICorner")
        dropdownCorner.CornerRadius = UDim.new(0, 8)
        dropdownCorner.Parent = dropdownFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0.5, 0, 1, 0)
        label.Position = UDim2.new(0, 10, 0, 0)
        label.Text = text
        label.TextColor3 = Color3.fromRGB(220, 220, 230)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 13
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = dropdownFrame

        local selectedText = Instance.new("TextButton")
        selectedText.Size = UDim2.new(0, 100, 0, 28)
        selectedText.Position = UDim2.new(1, -110, 0.5, -14)
        selectedText.Text = default
        selectedText.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
        selectedText.TextColor3 = Color3.fromRGB(255, 255, 255)
        selectedText.Font = Enum.Font.Gotham
        selectedText.TextSize = 11
        selectedText.BorderSizePixel = 0
        selectedText.Parent = dropdownFrame

        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 6)
        btnCorner.Parent = selectedText

        local dropdownOpen = false
        local dropdownList = nil

        selectedText.MouseButton1Click:Connect(function()
            if dropdownOpen then
                if dropdownList then dropdownList:Destroy() end
                dropdownOpen = false
                return
            end
            dropdownList = Instance.new("Frame")
            dropdownList.Size = UDim2.new(0, 100, 0, #options * 30)
            dropdownList.Position = UDim2.new(1, -110, 0, 40)
            dropdownList.BackgroundColor3 = Color3.fromRGB(40, 42, 55)
            dropdownList.BorderSizePixel = 0
            dropdownList.ZIndex = 10
            dropdownList.Parent = parent

            local listCorner = Instance.new("UICorner")
            listCorner.CornerRadius = UDim.new(0, 6)
            listCorner.Parent = dropdownList

            for i, option in ipairs(options) do
                local btn = Instance.new("TextButton")
                btn.Size = UDim2.new(1, 0, 0, 30)
                btn.Position = UDim2.new(0, 0, 0, (i-1)*30)
                btn.Text = option
                btn.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
                btn.TextColor3 = Color3.fromRGB(255, 255, 255)
                btn.Font = Enum.Font.Gotham
                btn.TextSize = 11
                btn.BorderSizePixel = 0
                btn.ZIndex = 10
                btn.Parent = dropdownList

                btn.MouseButton1Click:Connect(function()
                    selectedText.Text = option
                    callback(option)
                    dropdownList:Destroy()
                    dropdownOpen = false
                end)
            end
            dropdownOpen = true
        end)

        return dropdownFrame
    end

    local function createSlider(parent, y, text, min, max, default, callback)
        local sliderFrame = Instance.new("Frame")
        sliderFrame.Size = UDim2.new(1, -20, 0, 50)
        sliderFrame.Position = UDim2.new(0, 10, 0, y)
        sliderFrame.BackgroundColor3 = Color3.fromRGB(35, 38, 50)
        sliderFrame.BackgroundTransparency = 0.3
        sliderFrame.BorderSizePixel = 0
        sliderFrame.Parent = parent

        local sliderCorner = Instance.new("UICorner")
        sliderCorner.CornerRadius = UDim.new(0, 8)
        sliderCorner.Parent = sliderFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(1, -20, 0, 18)
        label.Position = UDim2.new(0, 10, 0, 4)
        label.Text = text .. ": " .. default
        label.TextColor3 = Color3.fromRGB(200, 200, 210)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 11
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = sliderFrame

        local sliderBox = Instance.new("TextBox")
        sliderBox.Size = UDim2.new(0, 60, 0, 22)
        sliderBox.Position = UDim2.new(1, -70, 0, 26)
        sliderBox.Text = tostring(default)
        sliderBox.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
        sliderBox.TextColor3 = Color3.fromRGB(255, 255, 255)
        sliderBox.Font = Enum.Font.Gotham
        sliderBox.TextSize = 11
        sliderBox.BorderSizePixel = 0
        sliderBox.Parent = sliderFrame

        local boxCorner = Instance.new("UICorner")
        boxCorner.CornerRadius = UDim.new(0, 4)
        boxCorner.Parent = sliderBox

        sliderBox.FocusLost:Connect(function(enterPressed)
            if enterPressed then
                local num = tonumber(sliderBox.Text)
                if num and num >= min and num <= max then
                    label.Text = text .. ": " .. num
                    callback(num)
                else
                    sliderBox.Text = tostring(default)
                end
            end
        end)

        return sliderFrame
    end

    local function createKeybind(parent, y, text, defaultKey, callback)
        local keyFrame = Instance.new("Frame")
        keyFrame.Size = UDim2.new(1, -20, 0, 40)
        keyFrame.Position = UDim2.new(0, 10, 0, y)
        keyFrame.BackgroundColor3 = Color3.fromRGB(35, 38, 50)
        keyFrame.BackgroundTransparency = 0.3
        keyFrame.BorderSizePixel = 0
        keyFrame.Parent = parent

        local keyCorner = Instance.new("UICorner")
        keyCorner.CornerRadius = UDim.new(0, 8)
        keyCorner.Parent = keyFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0.5, 0, 1, 0)
        label.Position = UDim2.new(0, 10, 0, 0)
        label.Text = text
        label.TextColor3 = Color3.fromRGB(220, 220, 230)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 13
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = keyFrame

        local keyBtn = Instance.new("TextButton")
        keyBtn.Size = UDim2.new(0, 80, 0, 28)
        keyBtn.Position = UDim2.new(1, -90, 0.5, -14)
        keyBtn.Text = defaultKey
        keyBtn.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
        keyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        keyBtn.Font = Enum.Font.GothamBold
        keyBtn.TextSize = 12
        keyBtn.BorderSizePixel = 0
        keyBtn.Parent = keyFrame

        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 6)
        btnCorner.Parent = keyBtn

        local listening = false
        keyBtn.MouseButton1Click:Connect(function()
            listening = true
            keyBtn.Text = "..."
            keyBtn.BackgroundColor3 = Color3.fromRGB(200, 160, 0)
        end)

        local inputConn
        inputConn = UserInputService.InputBegan:Connect(function(input, gameProcessed)
            if listening and input.KeyCode ~= Enum.KeyCode.Unknown then
                local keyName = input.KeyCode.Name
                keyBtn.Text = keyName
                keyBtn.BackgroundColor3 = Color3.fromRGB(45, 48, 60)
                listening = false
                callback(input.KeyCode, keyName)
            end
        end)
    end

    createToggle(aimbotFrame, 10, "Enable Aimbot", settings.AimbotEnabled, function(val)
        settings.AimbotEnabled = val
    end)

    createToggle(aimbotFrame, 55, "Weapon Aim", settings.WeaponAimbotEnabled, function(val)
        settings.WeaponAimbotEnabled = val
    end)

    createDropdown(aimbotFrame, 100, "Target Part", bodyParts, settings.TargetPart, function(val)
        settings.TargetPart = val
    end)

    createDropdown(aimbotFrame, 145, "Target Mode", {"Auto", "Manual"}, settings.TargetMode, function(val)
        settings.TargetMode = val
        if val == "Manual" then
            settings.SelectedPlayer = nil
        end
    end)

    createSlider(aimbotFrame, 195, "Camera Smoothness", 0.05, 0.8, settings.Smoothness, function(val)
        settings.Smoothness = val
    end)

    createSlider(aimbotFrame, 250, "Weapon Smoothness", 0.05, 0.8, settings.WeaponSmoothness, function(val)
        settings.WeaponSmoothness = val
    end)

    createSlider(aimbotFrame, 305, "FOV Radius", 0, 800, settings.FOVRadius, function(val)
        settings.FOVRadius = val
        updateFOVCircle()
    end)

    createToggle(aimbotFrame, 360, "Show FOV Circle", settings.ShowFOVCircle, function(val)
        settings.ShowFOVCircle = val
        updateFOVCircle()
    end)

    local playerListLabel = Instance.new("TextLabel")
    playerListLabel.Size = UDim2.new(1, -20, 0, 25)
    playerListLabel.Position = UDim2.new(0, 10, 0, 410)
    playerListLabel.Text = "PLAYERS:"
    playerListLabel.TextColor3 = Color3.fromRGB(200, 200, 210)
    playerListLabel.BackgroundTransparency = 1
    playerListLabel.Font = Enum.Font.GothamBold
    playerListLabel.TextSize = 13
    playerListLabel.TextXAlignment = Enum.TextXAlignment.Left
    playerListLabel.Parent = aimbotFrame

    local playerListFrame = Instance.new("ScrollingFrame")
    playerListFrame.Size = UDim2.new(1, -20, 0, 100)
    playerListFrame.Position = UDim2.new(0, 10, 0, 435)
    playerListFrame.BackgroundColor3 = Color3.fromRGB(30, 33, 45)
    playerListFrame.BorderSizePixel = 0
    playerListFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    playerListFrame.ScrollBarThickness = 6
    playerListFrame.ScrollBarImageColor3 = Color3.fromRGB(0, 162, 255)
    playerListFrame.Parent = aimbotFrame

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
                playerBtn.Size = UDim2.new(1, -10, 0, 28)
                playerBtn.Position = UDim2.new(0, 5, 0, yOffset)
                playerBtn.Text = player.Name
                playerBtn.BackgroundColor3 = (settings.SelectedPlayer == player.Name) and Color3.fromRGB(0, 140, 255) or Color3.fromRGB(40, 43, 55)
                playerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
                playerBtn.Font = Enum.Font.Gotham
                playerBtn.TextSize = 11
                playerBtn.BorderSizePixel = 0
                playerBtn.Parent = playerListFrame

                local btnCorner = Instance.new("UICorner")
                btnCorner.CornerRadius = UDim.new(0, 4)
                btnCorner.Parent = playerBtn

                playerBtn.MouseButton1Click:Connect(function()
                    if settings.TargetMode == "Manual" then
                        settings.SelectedPlayer = player.Name
                        updatePlayerList()
                    end
                end)
                yOffset = yOffset + 32
            end
        end
        playerListFrame.CanvasSize = UDim2.new(0, 0, 0, yOffset)
    end
    updatePlayerList()

    createToggle(farmFrame, 10, "Auto Farm Money", false, function(val)
        settings.AutoFarmMoney = val
        if val then
            startAutoFarmMoney()
        else
            stopAutoFarmMoney()
        end
    end)

    createToggle(farmFrame, 55, "Auto Farm Level", false, function(val)
        settings.AutoFarmLevel = val
        if val then
            startAutoFarmLevel()
        else
            stopAutoFarmLevel()
        end
    end)

    local farmInfo = Instance.new("TextLabel")
    farmInfo.Size = UDim2.new(1, -20, 0, 100)
    farmInfo.Position = UDim2.new(0, 10, 0, 110)
    farmInfo.Text = "Farm de Dinheiro:\nRouba todos os locais do mapa automaticamente incluindo:\n- Banco\n- Joalheria\n- Museu\n- Cassino\n- Posto de Gasolina\n- Loja de Donuts\n- Base Criminal\n- Delegacia\n- Prisao\n- Usina Eletrica\n- Hospital\n- Nightclub\n- Base de Herois\n- Estacao de Trem\n- Aeroporto"
    farmInfo.TextColor3 = Color3.fromRGB(160, 180, 200)
    farmInfo.BackgroundTransparency = 1
    farmInfo.Font = Enum.Font.Gotham
    farmInfo.TextSize = 11
    farmInfo.TextXAlignment = Enum.TextXAlignment.Left
    farmInfo.TextYAlignment = Enum.TextYAlignment.Top
    farmInfo.Parent = farmFrame

    createToggle(movementFrame, 10, "Fly", false, function(val)
        settings.FlyEnabled = val
        if val then
            setupFly()
        else
            disableFly()
        end
    end)

    createSlider(movementFrame, 55, "Fly Speed", 10, 200, settings.FlySpeed, function(val)
        settings.FlySpeed = val
    end)

    createToggle(movementFrame, 110, "Noclip", false, function(val)
        settings.NoclipEnabled = val
        if val then
            setupNoclip()
        else
            disableNoclip()
        end
    end)

    local movInfo = Instance.new("TextLabel")
    movInfo.Size = UDim2.new(1, -20, 0, 60)
    movInfo.Position = UDim2.new(0, 10, 0, 165)
    movInfo.Text = "Fly: W/A/S/D = Movimento, Espaco = Subir, Ctrl = Descer\nNoclip: Atravessa paredes e objetos"
    movInfo.TextColor3 = Color3.fromRGB(160, 180, 200)
    movInfo.BackgroundTransparency = 1
    movInfo.Font = Enum.Font.Gotham
    movInfo.TextSize = 11
    movInfo.TextXAlignment = Enum.TextXAlignment.Left
    movInfo.TextYAlignment = Enum.TextYAlignment.Top
    movInfo.Parent = movementFrame

    createToggle(visualsFrame, 10, "ESP", settings.ESPEnabled, function(val)
        settings.ESPEnabled = val
        if not val then clearESP() end
    end)

    createDropdown(visualsFrame, 55, "ESP Box Color", {"Blue", "Red", "Green", "Yellow", "Purple", "White"}, "Blue", function(val)
        local colors = {Blue = Color3.fromRGB(0, 162, 255), Red = Color3.fromRGB(255, 50, 50), Green = Color3.fromRGB(50, 255, 50), Yellow = Color3.fromRGB(255, 255, 50), Purple = Color3.fromRGB(180, 50, 255), White = Color3.fromRGB(255, 255, 255)}
        settings.ESPBoxColor = colors[val] or Color3.fromRGB(0, 162, 255)
    end)

    createDropdown(visualsFrame, 100, "ESP Text Color", {"White", "Blue", "Red", "Green", "Yellow", "Purple"}, "White", function(val)
        local colors = {White = Color3.fromRGB(255, 255, 255), Blue = Color3.fromRGB(0, 162, 255), Red = Color3.fromRGB(255, 50, 50), Green = Color3.fromRGB(50, 255, 50), Yellow = Color3.fromRGB(255, 255, 50), Purple = Color3.fromRGB(180, 50, 255)}
        settings.ESPTextColor = colors[val] or Color3.fromRGB(255, 255, 255)
    end)

    createToggle(weaponsFrame, 10, "Teleport Gun", false, function(val)
        if val then
            teleportGunToPlayer("")
        end
    end)

    local gunBox = Instance.new("TextBox")
    gunBox.Size = UDim2.new(1, -20, 0, 35)
    gunBox.Position = UDim2.new(0, 10, 0, 55)
    gunBox.PlaceholderText = "Nome da arma (ex: Rifle, Pistol, Shotgun)..."
    gunBox.Text = ""
    gunBox.BackgroundColor3 = Color3.fromRGB(35, 38, 50)
    gunBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    gunBox.Font = Enum.Font.Gotham
    gunBox.TextSize = 12
    gunBox.BorderSizePixel = 0
    gunBox.Parent = weaponsFrame

    local gunCorner = Instance.new("UICorner")
    gunCorner.CornerRadius = UDim.new(0, 6)
    gunCorner.Parent = gunBox

    local teleportBtn = Instance.new("TextButton")
    teleportBtn.Size = UDim2.new(1, -20, 0, 35)
    teleportBtn.Position = UDim2.new(0, 10, 0, 95)
    teleportBtn.Text = "TELEPORT ARMA"
    teleportBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    teleportBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    teleportBtn.Font = Enum.Font.GothamBold
    teleportBtn.TextSize = 14
    teleportBtn.BorderSizePixel = 0
    teleportBtn.Parent = weaponsFrame

    local teleportCorner = Instance.new("UICorner")
    teleportCorner.CornerRadius = UDim.new(0, 6)
    teleportCorner.Parent = teleportBtn

    teleportBtn.MouseButton1Click:Connect(function()
        teleportGunToPlayer(gunBox.Text)
    end)

    local refreshBtn = Instance.new("TextButton")
    refreshBtn.Size = UDim2.new(1, -20, 0, 35)
    refreshBtn.Position = UDim2.new(0, 10, 0, 140)
    refreshBtn.Text = "SCAN ARMAS (Refresh)"
    refreshBtn.BackgroundColor3 = Color3.fromRGB(35, 38, 50)
    refreshBtn.TextColor3 = Color3.fromRGB(220, 220, 230)
    refreshBtn.Font = Enum.Font.Gotham
    refreshBtn.TextSize = 13
    refreshBtn.BorderSizePixel = 0
    refreshBtn.Parent = weaponsFrame

    local refreshCorner = Instance.new("UICorner")
    refreshCorner.CornerRadius = UDim.new(0, 6)
    refreshCorner.Parent = refreshBtn

    refreshBtn.MouseButton1Click:Connect(function()
        scanForGuns()
    end)

    createKeybind(keysFrame, 10, "Aimbot Key", settings.AimbotKeyName, function(keyCode, keyName)
        settings.AimbotKey = keyCode
        settings.AimbotKeyName = keyName
    end)

    createKeybind(keysFrame, 55, "Fly Key", settings.FlyKeyName, function(keyCode, keyName)
        settings.FlyKey = keyCode
        settings.FlyKeyName = keyName
    end)

    createKeybind(keysFrame, 100, "Noclip Key", settings.NoclipKeyName, function(keyCode, keyName)
        settings.NoclipKey = keyCode
        settings.NoclipKeyName = keyName
    end)

    createKeybind(keysFrame, 145, "Teleport Gun Key", settings.TeleportGunKeyName, function(keyCode, keyName)
        settings.TeleportGunKey = keyCode
        settings.TeleportGunKeyName = keyName
    end)

    local keyInfo = Instance.new("TextLabel")
    keyInfo.Size = UDim2.new(1, -20, 0, 80)
    keyInfo.Position = UDim2.new(0, 10, 0, 200)
    keyInfo.Text = "Clique no botao da keybind e pressione a tecla desejada para configurar.\n\nAs keybinds sao salvas durante a sessao."
    keyInfo.TextColor3 = Color3.fromRGB(160, 180, 200)
    keyInfo.BackgroundTransparency = 1
    keyInfo.Font = Enum.Font.Gotham
    keyInfo.TextSize = 11
    keyInfo.TextXAlignment = Enum.TextXAlignment.Left
    keyInfo.TextYAlignment = Enum.TextYAlignment.Top
    keyInfo.Parent = keysFrame

    local bottomBar = Instance.new("Frame")
    bottomBar.Size = UDim2.new(1, 0, 0, 30)
    bottomBar.Position = UDim2.new(0, 0, 1, -30)
    bottomBar.BackgroundColor3 = Color3.fromRGB(12, 14, 22)
    bottomBar.BackgroundTransparency = 0.3
    bottomBar.BorderSizePixel = 0
    bottomBar.Parent = mainFrame

    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 80, 0, 24)
    closeBtn.Position = UDim2.new(1, -90, 0.5, -12)
    closeBtn.Text = "HIDE UI"
    closeBtn.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.TextSize = 11
    closeBtn.BorderSizePixel = 0
    closeBtn.Parent = bottomBar

    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 4)
    closeCorner.Parent = closeBtn

    closeBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = false
    end)

    local openBtn = Instance.new("TextButton")
    openBtn.Size = UDim2.new(0, 45, 0, 45)
    openBtn.Position = UDim2.new(0, 20, 0.5, -22)
    openBtn.Text = "S"
    openBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    openBtn.Font = Enum.Font.GothamBold
    openBtn.TextSize = 22
    openBtn.BorderSizePixel = 0
    openBtn.Parent = screenGui

    local openCorner = Instance.new("UICorner")
    openCorner.CornerRadius = UDim.new(0, 22)
    openCorner.Parent = openBtn

    local openStroke = Instance.new("UIStroke")
    openStroke.Thickness = 2
    openStroke.Color = Color3.fromRGB(0, 220, 255)
    openStroke.Parent = openBtn

    openBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = true
        updatePlayerList()
    end)

    local dragToggle = false
    local dragStart = nil
    local frameStart = nil

    titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragToggle = true
            dragStart = input.Position
            frameStart = mainFrame.Position
        end
    end)

    titleBar.InputEnded:Connect(function(input)
        dragToggle = false
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragToggle and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = input.Position - dragStart
            mainFrame.Position = UDim2.new(
                frameStart.X.Scale,
                frameStart.X.Offset + delta.X,
                frameStart.Y.Scale,
                frameStart.Y.Offset + delta.Y
            )
        end
    end)

    return screenGui
end

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
    healthLabel.Text = "Health: " .. math.floor(humanoid.Health)
    healthLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
    healthLabel.BackgroundTransparency = 1
    healthLabel.Font = Enum.Font.Gotham
    healthLabel.TextSize = 12
    healthLabel.Parent = frame
    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Size = UDim2.new(1, 0, 0, 16)
    distanceLabel.Position = UDim2.new(0, 0, 0, 36)
    distanceLabel.Text = "Dist: --"
    distanceLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.Font = Enum.Font.Gotham
    distanceLabel.TextSize = 11
    distanceLabel.Parent = frame
    local highlight = Instance.new("Highlight")
    highlight.Name = "ESP_Highlight_" .. player.Name
    highlight.FillColor = settings.ESPBoxColor
    highlight.FillTransparency = 0.7
    highlight.OutlineColor = settings.ESPBoxColor
    highlight.OutlineTransparency = 0.2
    highlight.OutlineWidth = 2
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
            if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
                if not espObjects[player] then
                    createESP(player)
                else
                    local humanoid = character:FindFirstChild("Humanoid")
                    local rootPart = character:FindFirstChild("HumanoidRootPart")
                    if espObjects[player].HealthLabel and humanoid then
                        espObjects[player].HealthLabel.Text = "Health: " .. math.floor(humanoid.Health)
                    end
                    if espObjects[player].DistanceLabel and rootPart and localPos then
                        local dist = (rootPart.Position - localPos).Magnitude
                        espObjects[player].DistanceLabel.Text = "Dist: " .. math.floor(dist) .. " studs"
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

local function updateCameraAimbot()
    if not settings.AimbotEnabled then return end
    if not aimbotHeld then return end
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

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if not hubLoaded then return end
    if input.KeyCode == settings.AimbotKey then
        aimbotHeld = true
    end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = true
        currentWeapon = getCurrentWeapon()
    end
    if input.KeyCode == settings.FlyKey then
        settings.FlyEnabled = not settings.FlyEnabled
        if settings.FlyEnabled then
            setupFly()
        else
            disableFly()
        end
    end
    if input.KeyCode == settings.NoclipKey then
        settings.NoclipEnabled = not settings.NoclipEnabled
        if settings.NoclipEnabled then
            setupNoclip()
        else
            disableNoclip()
        end
    end
    if input.KeyCode == settings.TeleportGunKey then
        teleportGunToPlayer("")
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if input.KeyCode == settings.AimbotKey then
        aimbotHeld = false
    end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = false
    end
end)

function loadHub()
    createUI()
    updateFOVCircle()
    hubLoaded = true
    RunService.RenderStepped:Connect(function()
        if not hubLoaded then return end
        updateCameraAimbot()
        updateESP()
        updateFly()
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
    Players.PlayerAdded:Connect(function()
        if hubLoaded then
            local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
            if screenGui then
                local mainFrame = screenGui:FindFirstChild("MainFrame")
                if mainFrame then
                    local contentArea = mainFrame:FindFirstChildOfClass("Frame")
                    if contentArea then
                        for _, child in ipairs(contentArea:GetDescendants()) do
                            if child.Name == "AIMBOT" then
                                local playerListFrame = child:FindFirstChildOfClass("ScrollingFrame")
                                if playerListFrame then
                                    for _, btn in ipairs(playerListFrame:GetChildren()) do
                                        if btn:IsA("TextButton") then
                                            btn:Destroy()
                                        end
                                    end
                                    local yOffset = 0
                                    for _, player in ipairs(Players:GetPlayers()) do
                                        if player ~= LocalPlayer then
                                            local playerBtn = Instance.new("TextButton")
                                            playerBtn.Size = UDim2.new(1, -10, 0, 28)
                                            playerBtn.Position = UDim2.new(0, 5, 0, yOffset)
                                            playerBtn.Text = player.Name
                                            playerBtn.BackgroundColor3 = (settings.SelectedPlayer == player.Name) and Color3.fromRGB(0, 140, 255) or Color3.fromRGB(40, 43, 55)
                                            playerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
                                            playerBtn.Font = Enum.Font.Gotham
                                            playerBtn.TextSize = 11
                                            playerBtn.BorderSizePixel = 0
                                            playerBtn.Parent = playerListFrame
                                            playerBtn.MouseButton1Click:Connect(function()
                                                if settings.TargetMode == "Manual" then
                                                    settings.SelectedPlayer = player.Name
                                                end
                                            end)
                                            yOffset = yOffset + 32
                                        end
                                    end
                                    playerListFrame.CanvasSize = UDim2.new(0, 0, 0, yOffset)
                                end
                            end
                        end
                    end
                end
            end
        end
    end)
    Players.PlayerRemoving:Connect(function(player)
        if espObjects[player] then
            pcall(function()
                espObjects[player].Billboard:Destroy()
                if espObjects[player].Highlight then espObjects[player].Highlight:Destroy() end
            end)
            espObjects[player] = nil
        end
    end)
end

createKeyScreen()
    TeleportGunKeyName = "T",
    TeleportAllGuns = false,
    InfiniteAmmo = false,
    NoRecoil = false,
    RapidFire = false,
    TriggerBot = false,
    SpinBot = false,
    SpinSpeed = 100,
    AntiAfk = true,
    AutoRespawn = true,
    ChatSpyEnabled = false,
    RainbowESP = false,
    RainbowSpeed = 1,
    ViewDistance = 500,
    TeamCheck = false,
    CrosshairEnabled = false,
    CrosshairSize = 20,
    CrosshairColor = Color3.fromRGB(0, 255, 0),
    CrosshairThickness = 2,
    AutoRobEnabled = false,
    AutoArrest = false,
    KillAura = false,
    KillAuraRange = 15,
    TeleportToPlayer = false,
    GodMode = false,
    InfiniteJump = false,
    WalkSpeed = 16,
    JumpPower = 50,
    FOVChanger = 70,
    ThirdPerson = false,
    ThirdPersonDistance = 10,
    NoClipSpeed = 100,
    FarmLoopDelay = 3,
    AutoCollectLoot = false,
    AutoSellItems = false
}

local bodyParts = {"Head", "UpperTorso", "LowerTorso", "HumanoidRootPart", "Left Arm", "Right Arm", "Left Leg", "Right Leg"}
local currentTarget = nil
local espObjects = {}
local fovCircle = nil
local isAiming = false
local currentWeapon = nil
local aimbotHeld = false
local flyBodyGyro = nil
local flyBodyVelocity = nil
local noclipConnections = {}
local farmConnections = {}
local spinConnections = {}
local killAuraConnections = {}
local gunList = {}
local collectedGuns = {}
local currentRobberyIndex = 1
local robberyRoute = {}
local farmPaused = false
local crosshairGui = nil
local chatLog = {}
local originalWalkspeed = 16
local originalJumpPower = 50
local originalFOV = 70
local godModeConnection = nil
local infiniteJumpConnection = nil
local rainbowConnection = nil
local chatSpyConnection = nil
local triggerBotConnection = nil
local rapidFireConnection = nil
local noRecoilConnection = nil
local infiniteAmmoConnection = nil
local antiAfkConnection = nil
local autoRespawnConnection = nil
local thirdPersonConnection = nil

local madCityChapter1Robberies = {
    {Name = "Bank", Aliases = {"Main Bank", "City Bank", "Bank"}},
    {Name = "Jewelry Store", Aliases = {"Jewelry", "Diamond Store", "Gem Store"}},
    {Name = "Museum", Aliases = {"City Museum", "Art Museum", "Museum"}},
    {Name = "Casino", Aliases = {"Casino", "Lucky Casino", "Casino Building"}},
    {Name = "Gas Station", Aliases = {"Gas Station", "Fuel Station", "Petrol Station", "Convenience Store"}},
    {Name = "Donut Shop", Aliases = {"Donut Shop", "Bakery", "Coffee Shop"}},
    {Name = "Criminal Base", Aliases = {"Criminal Base", "Villain Base", "Evil Lair", "Warehouse"}},
    {Name = "Police Station", Aliases = {"Police Station", "Police HQ", "Sheriff Office"}},
    {Name = "Prison", Aliases = {"Prison", "Jail", "Penitentiary"}},
    {Name = "Power Plant", Aliases = {"Power Plant", "Electric Plant", "Nuclear Plant", "Factory"}},
    {Name = "Hospital", Aliases = {"Hospital", "Medical Center", "Clinic"}},
    {Name = "Nightclub", Aliases = {"Nightclub", "Club", "Dance Club", "Disco"}},
    {Name = "Hero Base", Aliases = {"Hero Base", "Hero HQ", "Hero Tower", "Superhero Base"}},
    {Name = "Train Station", Aliases = {"Train Station", "Subway", "Metro Station"}},
    {Name = "Airport", Aliases = {"Airport", "Airfield", "Landing Strip"}}
}

local function getCurrentWeapon()
    local character = LocalPlayer.Character
    if not character then return nil end
    local tool = character:FindFirstChildOfClass("Tool")
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
        Scope = scope,
        HasScope = scope ~= nil
    }
end

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

local function updateWeaponAim()
    if not settings.WeaponAimbotEnabled then return end
    if not isAiming then return end
    if not settings.AimbotEnabled then return end
    if not aimbotHeld then return end
    local target = getBestTarget()
    if not target then return end
    local character = target.Character
    if not character then return end
    local targetPart = getTargetPart(character)
    if not targetPart then return end
    local weaponAimPos = getWeaponAimPosition()
    if not weaponAimPos then return end
    local currentWeaponCFrame = currentWeapon.Tool:GetPivot()
    local targetCFrame = CFrame.lookAt(weaponAimPos, targetPart.Position)
    local smoothedCFrame = currentWeaponCFrame:Lerp(targetCFrame, 1 - settings.WeaponSmoothness)
    currentWeapon.Tool:SetPrimaryPartCFrame(smoothedCFrame)
    local handle = currentWeapon.Tool:FindFirstChild("Handle")
    if handle then
        local handleTargetCFrame = CFrame.lookAt(handle.Position, targetPart.Position)
        local smoothedHandle = handle.CFrame:Lerp(handleTargetCFrame, 1 - settings.WeaponSmoothness)
        handle.CFrame = smoothedHandle
    end
end

local function buildRobberyRoute()
    robberyRoute = {}
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local rootPart = character.HumanoidRootPart
    for _, robberyData in ipairs(madCityChapter1Robberies) do
        local found = nil
        for _, alias in ipairs(robberyData.Aliases) do
            local obj = workspace:FindFirstChild(alias, true)
            if obj then
                local part = nil
                if obj:IsA("BasePart") then
                    part = obj
                elseif obj:IsA("Model") then
                    part = obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart")
                end
                if part then
                    found = {Name = robberyData.Name, Part = part, Position = part.Position}
                    break
                end
            end
        end
        if not found then
            for _, alias in ipairs(robberyData.Aliases) do
                for _, child in ipairs(workspace:GetDescendants()) do
                    if child:IsA("BasePart") and child.Name:lower():find(alias:lower()) then
                        found = {Name = robberyData.Name, Part = child, Position = child.Position}
                        break
                    end
                end
                if found then break end
            end
        end
        if not found then
            for _, alias in ipairs(robberyData.Aliases) do
                for _, child in ipairs(workspace:GetDescendants()) do
                    if child:IsA("Model") and child.Name:lower():find(alias:lower()) then
                        local part = child.PrimaryPart or child:FindFirstChildWhichIsA("BasePart")
                        if part then
                            found = {Name = robberyData.Name, Part = part, Position = part.Position}
                            break
                        end
                    end
                end
                if found then break end
            end
        end
        if found then
            table.insert(robberyRoute, found)
        end
    end
    if #robberyRoute > 1 then
        local basePos = rootPart.Position
        table.sort(robberyRoute, function(a, b)
            return (a.Position - basePos).Magnitude < (b.Position - basePos).Magnitude
        end)
        local startIdx = 1
        local minDist = math.huge
        for i, loc in ipairs(robberyRoute) do
            local dist = (loc.Position - basePos).Magnitude
            if dist < minDist then
                minDist = dist
                startIdx = i
            end
        end
        local sorted = {}
        local used = {}
        local current = startIdx
        for i = 1, #robberyRoute do
            table.insert(sorted, robberyRoute[current])
            used[current] = true
            local bestNext = nil
            local bestDist = math.huge
            for j = 1, #robberyRoute do
                if not used[j] then
                    local dist = (robberyRoute[current].Position - robberyRoute[j].Position).Magnitude
                    if dist < bestDist then
                        bestDist = dist
                        bestNext = j
                    end
                end
            end
            current = bestNext
            if not current then break end
        end
        for j = 1, #robberyRoute do
            if not used[j] then
                table.insert(sorted, robberyRoute[j])
            end
        end
        robberyRoute = sorted
    end
    currentRobberyIndex = 1
end

local function scanForLootables(robberyPosition)
    local lootables = {}
    local searchRadius = 100
    for _, descendant in ipairs(workspace:GetDescendants()) do
        if descendant:IsA("BasePart") or descendant:IsA("MeshPart") then
            local dist = (descendant.Position - robberyPosition).Magnitude
            if dist <= searchRadius then
                local name = descendant.Name:lower()
                if name:find("money") or name:find("cash") or name:find("gold") or name:find("diamond") or
                   name:find("jewel") or name:find("loot") or name:find("valuable") or name:find("artifact") or
                   name:find("painting") or name:find("sculpture") or name:find("treasure") or name:find("ring") or
                   name:find("necklace") or name:find("bracelet") or name:find("watch") or name:find("gem") or
                   name:find("safe") or name:find("chest") or name:find("briefcase") or name:find("bag") then
                    table.insert(lootables, descendant)
                end
                if descendant:IsA("Model") then
                    local modelName = descendant.Name:lower()
                    if modelName:find("money") or modelName:find("cash") or modelName:find("gold") or modelName:find("loot") then
                        table.insert(lootables, descendant)
                    end
                end
                if descendant:FindFirstChild("ClickDetector") then
                    table.insert(lootables, descendant)
                end
                if descendant.Parent and descendant.Parent:FindFirstChild("ClickDetector") then
                    table.insert(lootables, descendant.Parent)
                end
            end
        end
    end
    return lootables
end

local function robLocation(locationData)
    if not settings.AutoFarmMoney then return end
    if farmPaused then return end
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local rootPart = character.HumanoidRootPart
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end
    humanoid:MoveTo(locationData.Position)
    local arrivalTime = tick()
    local maxWait = 15
    while (rootPart.Position - locationData.Position).Magnitude > 20 do
        if not settings.AutoFarmMoney then return end
        if farmPaused then return end
        if tick() - arrivalTime > maxWait then break end
        humanoid:MoveTo(locationData.Position)
        task.wait(0.3)
    end
    if (rootPart.Position - locationData.Position).Magnitude <= 50 then
        local lootables = scanForLootables(locationData.Position)
        local collectTime = tick()
        local maxCollect = 8
        while tick() - collectTime < maxCollect do
            if not settings.AutoFarmMoney then return end
            if farmPaused then return end
            for _, loot in ipairs(lootables) do
                if loot and loot.Parent then
                    local lootPart = nil
                    if loot:IsA("BasePart") then
                        lootPart = loot
                    elseif loot:IsA("Model") then
                        lootPart = loot.PrimaryPart or loot:FindFirstChildWhichIsA("BasePart")
                    end
                    if lootPart and rootPart then
                        firetouchinterest(rootPart, lootPart, 0)
                        firetouchinterest(rootPart, lootPart, 1)
                    end
                    if loot:FindFirstChild("ClickDetector") then
                        fireclickdetector(loot.ClickDetector)
                    end
                end
            end
            local proximityItems = {}
            for _, descendant in ipairs(workspace:GetDescendants()) do
                if descendant:IsA("BasePart") or descendant:IsA("MeshPart") then
                    local dist = (descendant.Position - rootPart.Position).Magnitude
                    if dist <= 10 then
                        table.insert(proximityItems, descendant)
                    end
                    if descendant:FindFirstChild("ClickDetector") and dist <= 10 then
                        fireclickdetector(descendant.ClickDetector)
                    end
                end
            end
            for _, item in ipairs(proximityItems) do
                firetouchinterest(rootPart, item, 0)
                firetouchinterest(rootPart, item, 1)
            end
            task.wait(0.5)
        end
    end
end

local function startAutoFarmMoney()
    if settings.AutoFarmMoney then
        for _, connection in ipairs(farmConnections) do
            connection:Disconnect()
        end
        farmConnections = {}
        farmPaused = false
        currentRobberyIndex = 1
        buildRobberyRoute()
        local farmConnection
        farmConnection = RunService.Heartbeat:Connect(function()
            if not settings.AutoFarmMoney then
                farmConnection:Disconnect()
                return
            end
            if farmPaused then return end
            if #robberyRoute == 0 then
                buildRobberyRoute()
                if #robberyRoute == 0 then
                    task.wait(5)
                    buildRobberyRoute()
                    return
                end
            end
            if currentRobberyIndex > #robberyRoute then
                currentRobberyIndex = 1
                buildRobberyRoute()
                task.wait(settings.FarmLoopDelay)
            end
            local currentLocation = robberyRoute[currentRobberyIndex]
            if currentLocation then
                robLocation(currentLocation)
                currentRobberyIndex = currentRobberyIndex + 1
            end
        end)
        table.insert(farmConnections, farmConnection)
    end
end

local function stopAutoFarmMoney()
    settings.AutoFarmMoney = false
    farmPaused = false
    for _, connection in ipairs(farmConnections) do
        connection:Disconnect()
    end
    farmConnections = {}
    robberyRoute = {}
    currentRobberyIndex = 1
end

local function startAutoFarmLevel()
    if not settings.AutoFarmLevel then return end
    for _, connection in ipairs(farmConnections) do
        connection:Disconnect()
    end
    farmConnections = {}
    local levelFarmConnection
    levelFarmConnection = RunService.Heartbeat:Connect(function()
        if not settings.AutoFarmLevel then
            levelFarmConnection:Disconnect()
            return
        end
        local character = LocalPlayer.Character
        if not character or not character:FindFirstChild("HumanoidRootPart") then return end
        local rootPart = character.HumanoidRootPart
        local humanoid = character:FindFirstChild("Humanoid")
        if not humanoid or humanoid.Health <= 0 then return end
        local npcs = {}
        local ignoreNames = {"Civilian", "Citizen", "Pedestrian", "Worker", "Doctor"}
        for _, descendant in ipairs(workspace:GetDescendants()) do
            if descendant:IsA("Model") and descendant:FindFirstChild("Humanoid") and descendant:FindFirstChild("Head") then
                local hum = descendant.Humanoid
                local skip = false
                for _, ignore in ipairs(ignoreNames) do
                    if descendant.Name:lower():find(ignore:lower()) then
                        skip = true
                        break
                    end
                end
                if not skip and hum.Health > 0 and descendant ~= character then
                    local head = descendant:FindFirstChild("Head")
                    if head then
                        local dist = (rootPart.Position - head.Position).Magnitude
                        if dist <= 200 then
                            table.insert(npcs, {Model = descendant, Head = head, Distance = dist, Humanoid = hum})
                        end
                    end
                end
            end
        end
        if #npcs == 0 then return end
        table.sort(npcs, function(a, b) return a.Distance < b.Distance end)
        local target = npcs[1]
        if target and target.Distance <= 100 then
            humanoid:MoveTo(target.Head.Position)
            if target.Distance <= 15 then
                local tool = character:FindFirstChildOfClass("Tool")
                if tool then
                    tool:Activate()
                    task.wait(0.1)
                    tool:Deactivate()
                end
            end
        end
    end)
    table.insert(farmConnections, levelFarmConnection)
end

local function stopAutoFarmLevel()
    settings.AutoFarmLevel = false
    for _, connection in ipairs(farmConnections) do
        connection:Disconnect()
    end
    farmConnections = {}
end

local function setupKillAura()
    if settings.KillAura then
        for _, connection in ipairs(killAuraConnections) do
            connection:Disconnect()
        end
        killAuraConnections = {}
        local killConnection
        killConnection = RunService.Heartbeat:Connect(function()
            if not settings.KillAura then
                killConnection:Disconnect()
                return
            end
            local character = LocalPlayer.Character
            if not character or not character:FindFirstChild("HumanoidRootPart") then return end
            local rootPart = character.HumanoidRootPart
            local humanoid = character:FindFirstChild("Humanoid")
            if not humanoid or humanoid.Health <= 0 then return end
            local tool = character:FindFirstChildOfClass("Tool")
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= LocalPlayer then
                    local targetChar = player.Character
                    if targetChar and targetChar:FindFirstChild("Humanoid") and targetChar.Humanoid.Health > 0 then
                        local targetRoot = targetChar:FindFirstChild("HumanoidRootPart") or targetChar:FindFirstChild("Head")
                        if targetRoot then
                            local dist = (rootPart.Position - targetRoot.Position).Magnitude
                            if dist <= settings.KillAuraRange then
                                if settings.AimbotEnabled then
                                    local camera = workspace.CurrentCamera
                                    camera.CFrame = CFrame.new(camera.CFrame.Position, targetRoot.Position)
                                end
                                if tool then
                                    tool:Activate()
                                    task.wait(0.05)
                                    tool:Deactivate()
                                end
                            end
                        end
                    end
                end
            end
        end)
        table.insert(killAuraConnections, killConnection)
    end
end

local function setupSpinBot()
    if settings.SpinBot then
        for _, connection in ipairs(spinConnections) do
            connection:Disconnect()
        end
        spinConnections = {}
        local spinConnection
        spinConnection = RunService.RenderStepped:Connect(function()
            if not settings.SpinBot then
                spinConnection:Disconnect()
                return
            end
            local character = LocalPlayer.Character
            if not character or not character:FindFirstChild("HumanoidRootPart") then return end
            local rootPart = character.HumanoidRootPart
            rootPart.CFrame = rootPart.CFrame * CFrame.Angles(0, math.rad(settings.SpinSpeed * 0.1), 0)
        end)
        table.insert(spinConnections, spinConnection)
    end
end

local function setupTriggerBot()
    if settings.TriggerBot then
        if triggerBotConnection then triggerBotConnection:Disconnect() end
        triggerBotConnection = RunService.RenderStepped:Connect(function()
            if not settings.TriggerBot then
                triggerBotConnection:Disconnect()
                triggerBotConnection = nil
                return
            end
            local mousePos = UserInputService:GetMouseLocation()
            local camera = workspace.CurrentCamera
            local mouseRay = camera:ViewportPointToRay(mousePos.X, mousePos.Y)
            local raycastParams = RaycastParams.new()
            raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
            raycastParams.FilterDescendantsInstances = {LocalPlayer.Character or workspace}
            local rayResult = workspace:Raycast(mouseRay.Origin, mouseRay.Direction * 1000, raycastParams)
            if rayResult then
                local hitChar = rayResult.Instance:FindFirstAncestorOfClass("Model")
                if hitChar then
                    local hitPlayer = Players:GetPlayerFromCharacter(hitChar)
                    if hitPlayer and hitPlayer ~= LocalPlayer then
                        local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Tool")
                        if tool then
                            tool:Activate()
                            task.wait(0.05)
                            tool:Deactivate()
                        end
                    end
                end
            end
        end)
    end
end

local function setupRapidFire()
    if settings.RapidFire then
        if rapidFireConnection then rapidFireConnection:Disconnect() end
        rapidFireConnection = RunService.RenderStepped:Connect(function()
            if not settings.RapidFire then
                rapidFireConnection:Disconnect()
                rapidFireConnection = nil
                return
            end
            if UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1) then
                local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Tool")
                if tool then
                    tool:Activate()
                    task.wait()
                    tool:Deactivate()
                end
            end
        end)
    end
end

local function setupNoRecoil()
    if settings.NoRecoil then
        if noRecoilConnection then noRecoilConnection:Disconnect() end
        noRecoilConnection = RunService.RenderStepped:Connect(function()
            if not settings.NoRecoil then
                noRecoilConnection:Disconnect()
                noRecoilConnection = nil
                return
            end
            local character = LocalPlayer.Character
            if not character then return end
            local tool = character:FindFirstChildOfClass("Tool")
            if tool then
                local handle = tool:FindFirstChild("Handle")
                if handle then
                    handle.Velocity = Vector3.new(0, 0, 0)
                    handle.RotVelocity = Vector3.new(0, 0, 0)
                end
            end
        end)
    end
end

local function setupInfiniteAmmo()
    if settings.InfiniteAmmo then
        if infiniteAmmoConnection then infiniteAmmoConnection:Disconnect() end
        infiniteAmmoConnection = RunService.RenderStepped:Connect(function()
            if not settings.InfiniteAmmo then
                infiniteAmmoConnection:Disconnect()
                infiniteAmmoConnection = nil
                return
            end
            local tool = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Tool")
            if tool then
                for _, child in ipairs(tool:GetDescendants()) do
                    if child:IsA("IntValue") and child.Name:lower():find("ammo") then
                        child.Value = 999
                    end
                    if child:IsA("NumberValue") and child.Name:lower():find("ammo") then
                        child.Value = 999
                    end
                    if child.Name:lower():find("ammo") or child.Name:lower():find("clip") then
                        if child:IsA("IntValue") or child:IsA("NumberValue") then
                            child.Value = 999
                        end
                    end
                end
            end
        end)
    end
end

local function setupGodMode()
    if settings.GodMode then
        if godModeConnection then godModeConnection:Disconnect() end
        godModeConnection = RunService.RenderStepped:Connect(function()
            if not settings.GodMode then
                godModeConnection:Disconnect()
                godModeConnection = nil
                return
            end
            local character = LocalPlayer.Character
            if character then
                local humanoid = character:FindFirstChild("Humanoid")
                if humanoid then
                    humanoid.Health = humanoid.MaxHealth
                    humanoid.HealthDisplayDistance = math.huge
                    humanoid.NameDisplayDistance = math.huge
                end
                for _, part in ipairs(character:GetDescendants()) do
                    if part:IsA("BasePart") and part.Name ~= "HumanoidRootPart" then
                        part.CanCollide = false
                        part.Massless = true
                    end
                end
            end
        end)
    end
end

local function setupInfiniteJump()
    if settings.InfiniteJump then
        if infiniteJumpConnection then infiniteJumpConnection:Disconnect() end
        infiniteJumpConnection = UserInputService.JumpRequest:Connect(function()
            if not settings.InfiniteJump then return end
            local character = LocalPlayer.Character
            if character then
                local humanoid = character:FindFirstChild("Humanoid")
                if humanoid then
                    humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
                end
            end
        end)
        local character = LocalPlayer.Character
        if character then
            local humanoid = character:FindFirstChild("Humanoid")
            if humanoid then
                humanoid.JumpPower = settings.JumpPower
                humanoid.UseJumpPower = true
                humanoid.AutoRotate = true
            end
        end
    end
end

local function setupWalkSpeed()
    local character = LocalPlayer.Character
    if character then
        local humanoid = character:FindFirstChild("Humanoid")
        if humanoid then
            humanoid.WalkSpeed = settings.WalkSpeed
            humanoid.JumpPower = settings.JumpPower
        end
    end
end

local function setupFOV()
    workspace.CurrentCamera.FieldOfView = settings.FOVChanger
end

local function setupThirdPerson()
    if settings.ThirdPerson then
        if thirdPersonConnection then thirdPersonConnection:Disconnect() end
        LocalPlayer.CameraMode = Enum.CameraMode.Classic
        local character = LocalPlayer.Character
        if character then
            local humanoid = character:FindFirstChild("Humanoid")
            if humanoid then
                humanoid.CameraOffset = Vector3.new(0, 0, settings.ThirdPersonDistance)
            end
        end
        thirdPersonConnection = RunService.RenderStepped:Connect(function()
            if not settings.ThirdPerson then
                thirdPersonConnection:Disconnect()
                thirdPersonConnection = nil
                return
            end
            local character = LocalPlayer.Character
            if character then
                local humanoid = character:FindFirstChild("Humanoid")
                if humanoid then
                    humanoid.CameraOffset = Vector3.new(0, 0, settings.ThirdPersonDistance)
                end
            end
        end)
    else
        LocalPlayer.CameraMode = Enum.CameraMode.LockFirstPerson
    end
end

local function setupAntiAfk()
    if settings.AntiAfk then
        if antiAfkConnection then antiAfkConnection:Disconnect() end
        antiAfkConnection = RunService.RenderStepped:Connect(function()
            if not settings.AntiAfk then
                antiAfkConnection:Disconnect()
                antiAfkConnection = nil
                return
            end
            local character = LocalPlayer.Character
            if character then
                local humanoid = character:FindFirstChild("Humanoid")
                if humanoid then
                    humanoid.MoveToFinished:Fire()
                    local rootPart = character:FindFirstChild("HumanoidRootPart")
                    if rootPart then
                        rootPart.Position = rootPart.Position + Vector3.new(math.random(-1, 1) * 0.01, 0, 0)
                    end
                end
            end
            task.wait(30)
        end)
    end
end

local function setupAutoRespawn()
    if settings.AutoRespawn then
        if autoRespawnConnection then autoRespawnConnection:Disconnect() end
        autoRespawnConnection = LocalPlayer.CharacterAdded:Connect(function()
            task.wait(0.5)
            setupWalkSpeed()
            setupFOV()
            if settings.InfiniteJump then setupInfiniteJump() end
        end)
        LocalPlayer.CharacterRemoving:Connect(function()
            task.wait(2)
            if settings.AutoRespawn and not LocalPlayer.Character then
                local respawnButton = LocalPlayer.PlayerGui:FindFirstChild("Respawn", true)
                if respawnButton and respawnButton:IsA("TextButton") then
                    firesignal(respawnButton.MouseButton1Click)
                end
            end
        end)
    end
end

local function setupChatSpy()
    if settings.ChatSpyEnabled then
        if chatSpyConnection then chatSpyConnection:Disconnect() end
        chatSpyConnection = Players.PlayerChatted:Connect(function(type, player, message)
            if player ~= LocalPlayer and settings.ChatSpyEnabled then
                local displayMessage = "[CHAT SPY] " .. player.Name .. ": " .. message
                table.insert(chatLog, displayMessage)
                if #chatLog > 100 then
                    table.remove(chatLog, 1)
                end
            end
        end)
    end
end

local function setupRainbowESP()
    if settings.RainbowESP then
        if rainbowConnection then rainbowConnection:Disconnect() end
        rainbowConnection = RunService.RenderStepped:Connect(function()
            if not settings.RainbowESP then
                rainbowConnection:Disconnect()
                rainbowConnection = nil
                return
            end
            local hue = (tick() * settings.RainbowSpeed) % 1
            settings.ESPBoxColor = Color3.fromHSV(hue, 1, 1)
            settings.ESPTextColor = Color3.fromHSV((hue + 0.5) % 1, 1, 1)
            for _, objects in pairs(espObjects) do
                if objects.Highlight then
                    objects.Highlight.FillColor = settings.ESPBoxColor
                    objects.Highlight.OutlineColor = settings.ESPBoxColor
                end
                if objects.NameLabel then
                    objects.NameLabel.TextColor3 = settings.ESPTextColor
                end
            end
        end)
    end
end

local function setupCrosshair()
    if crosshairGui then crosshairGui:Destroy() end
    if not settings.CrosshairEnabled then return end
    crosshairGui = Instance.new("ScreenGui")
    crosshairGui.Name = "CrosshairGui"
    crosshairGui.Parent = LocalPlayer.PlayerGui
    crosshairGui.ResetOnSpawn = false
    local horizontal = Instance.new("Frame")
    horizontal.Size = UDim2.new(0, settings.CrosshairSize, 0, settings.CrosshairThickness)
    horizontal.Position = UDim2.new(0.5, -settings.CrosshairSize/2, 0.5, -settings.CrosshairThickness/2)
    horizontal.BackgroundColor3 = settings.CrosshairColor
    horizontal.BorderSizePixel = 0
    horizontal.Parent = crosshairGui
    local vertical = Instance.new("Frame")
    vertical.Size = UDim2.new(0, settings.CrosshairThickness, 0, settings.CrosshairSize)
    vertical.Position = UDim2.new(0.5, -settings.CrosshairThickness/2, 0.5, -settings.CrosshairSize/2)
    vertical.BackgroundColor3 = settings.CrosshairColor
    vertical.BorderSizePixel = 0
    vertical.Parent = crosshairGui
end

local function scanForGuns()
    gunList = {}
    collectedGuns = {}
    local workspaceItems = workspace:GetChildren()
    for _, item in ipairs(workspaceItems) do
        if item:IsA("Tool") or item:IsA("Model") then
            local isGun = false
            if item:IsA("Tool") then
                if item:FindFirstChild("Handle") then
                    isGun = true
                end
            end
            if not isGun and item:IsA("Model") then
                for _, child in ipairs(item:GetChildren()) do
                    if child.Name:lower():find("gun") or child.Name:lower():find("weapon") or child.Name:lower():find("rifle") or child.Name:lower():find("pistol") or child.Name:lower():find("shotgun") then
                        isGun = true
                        break
                    end
                end
            end
            if isGun then
                table.insert(gunList, item)
            end
        end
    end
    for _, folderName in ipairs({"Weapons", "Guns", "Items", "Loot"}) do
        local folder = workspace:FindFirstChild(folderName)
        if folder then
            for _, item in ipairs(folder:GetChildren()) do
                if item:IsA("Tool") or item:IsA("Model") then
                    table.insert(gunList, item)
                end
            end
        end
    end
end

local function teleportGunToPlayer(gunName)
    scanForGuns()
    local targetGun = nil
    if gunName and gunName ~= "" then
        for _, gun in ipairs(gunList) do
            if gun.Name:lower():find(gunName:lower()) then
                targetGun = gun
                break
            end
        end
    end
    if not targetGun and #gunList > 0 then
        targetGun = gunList[1]
    end
    if targetGun and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local rootPart = LocalPlayer.Character.HumanoidRootPart
        local gunPrimaryPart = nil
        if targetGun:IsA("Tool") then
            gunPrimaryPart = targetGun:FindFirstChild("Handle")
        end
        if not gunPrimaryPart and targetGun:IsA("Model") then
            gunPrimaryPart = targetGun.PrimaryPart or targetGun:FindFirstChild("Handle") or targetGun:FindFirstChildWhichIsA("BasePart")
        end
        if gunPrimaryPart then
            targetGun:SetPrimaryPartCFrame(rootPart.CFrame + rootPart.CFrame.LookVector * 3)
            firetouchinterest(rootPart, gunPrimaryPart, 0)
            firetouchinterest(rootPart, gunPrimaryPart, 1)
            if not collectedGuns[targetGun.Name] then
                collectedGuns[targetGun.Name] = true
            end
        end
    end
end

local function teleportAllGunsToPlayer()
    scanForGuns()
    for _, gun in ipairs(gunList) do
        if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            local rootPart = LocalPlayer.Character.HumanoidRootPart
            local gunPrimaryPart = nil
            if gun:IsA("Tool") then
                gunPrimaryPart = gun:FindFirstChild("Handle")
            end
            if not gunPrimaryPart and gun:IsA("Model") then
                gunPrimaryPart = gun.PrimaryPart or gun:FindFirstChild("Handle") or gun:FindFirstChildWhichIsA("BasePart")
            end
            if gunPrimaryPart then
                gun:SetPrimaryPartCFrame(rootPart.CFrame + rootPart.CFrame.LookVector * 3 + Vector3.new(math.random(-5, 5), 0, math.random(-5, 5)))
                firetouchinterest(rootPart, gunPrimaryPart, 0)
                firetouchinterest(rootPart, gunPrimaryPart, 1)
                if not collectedGuns[gun.Name] then
                    collectedGuns[gun.Name] = true
                end
            end
        end
    end
end

local function setupFly()
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid then return end
    if flyBodyGyro then flyBodyGyro:Destroy() end
    if flyBodyVelocity then flyBodyVelocity:Destroy() end
    flyBodyGyro = Instance.new("BodyGyro")
    flyBodyGyro.MaxTorque = Vector3.new(400000, 400000, 400000)
    flyBodyGyro.CFrame = character.HumanoidRootPart.CFrame
    flyBodyGyro.Parent = character.HumanoidRootPart
    flyBodyVelocity = Instance.new("BodyVelocity")
    flyBodyVelocity.MaxForce = Vector3.new(400000, 400000, 400000)
    flyBodyVelocity.Velocity = Vector3.new(0, 0, 0)
    flyBodyVelocity.Parent = character.HumanoidRootPart
    humanoid.PlatformStand = true
end

local function disableFly()
    if flyBodyGyro then flyBodyGyro:Destroy() end
    if flyBodyVelocity then flyBodyVelocity:Destroy() end
    flyBodyGyro = nil
    flyBodyVelocity = nil
    local character = LocalPlayer.Character
    if character then
        local humanoid = character:FindFirstChild("Humanoid")
        if humanoid then
            humanoid.PlatformStand = false
        end
    end
end

local function updateFly()
    if not settings.FlyEnabled then return end
    if not flyBodyGyro or not flyBodyVelocity then return end
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local camera = workspace.CurrentCamera
    local rootPart = character.HumanoidRootPart
    flyBodyGyro.CFrame = camera.CFrame
    local velocity = Vector3.new(0, 0, 0)
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then
        velocity = velocity + camera.CFrame.LookVector * settings.FlySpeed
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then
        velocity = velocity - camera.CFrame.LookVector * settings.FlySpeed
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then
        velocity = velocity - camera.CFrame.RightVector * settings.FlySpeed
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then
        velocity = velocity + camera.CFrame.RightVector * settings.FlySpeed
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
        velocity = velocity + Vector3.new(0, settings.FlySpeed, 0)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
        velocity = velocity - Vector3.new(0, settings.FlySpeed, 0)
    end
    flyBodyVelocity.Velocity = velocity
end

local function setupNoclip()
    for _, connection in ipairs(noclipConnections) do
        connection:Disconnect()
    end
    noclipConnections = {}
    local connection = RunService.Stepped:Connect(function()
        if not settings.NoclipEnabled then return end
        local character = LocalPlayer.Character
        if not character then return end
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") and part.CanCollide then
                part.CanCollide = false
            end
        end
    end)
    table.insert(noclipConnections, connection)
end

local function disableNoclip()
    for _, connection in ipairs(noclipConnections) do
        connection:Disconnect()
    end
    noclipConnections = {}
    local character = LocalPlayer.Character
    if character then
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = true
            end
        end
    end
end

local function getTargetPart(character)
    if not character then return nil end
    local part = character:FindFirstChild(settings.TargetPart)
    if not part then
        for _, bodyPart in ipairs(bodyParts) do
            part = character:FindFirstChild(bodyPart)
            if part then break end
        end
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
            if settings.TeamCheck and player.Team == LocalPlayer.Team then
                continue
            end
            local character = player.Character
            if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
                local targetRoot = character:FindFirstChild("HumanoidRootPart")
                local localRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                if targetRoot and localRoot then
                    if (targetRoot.Position - localRoot.Position).Magnitude > settings.ViewDistance then
                        continue
                    end
                end
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

local function updateCameraAimbot()
    if not settings.AimbotEnabled then return end
    if not aimbotHeld then return end
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
    nameLabel.Size = UDim2.new(1, 0, 0, 20)
    nameLabel.Position = UDim2.new(0, 0, 0, 0)
    nameLabel.Text = player.Name
    nameLabel.TextColor3 = settings.ESPTextColor
    nameLabel.BackgroundTransparency = 1
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = 13
    nameLabel.TextStrokeTransparency = 0.3
    nameLabel.Parent = frame
    local healthLabel = Instance.new("TextLabel")
    healthLabel.Size = UDim2.new(1, 0, 0, 16)
    healthLabel.Position = UDim2.new(0, 0, 0, 22)
    healthLabel.Text = "HP: " .. math.floor(humanoid.Health)
    healthLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
    healthLabel.BackgroundTransparency = 1
    healthLabel.Font = Enum.Font.Gotham
    healthLabel.TextSize = 11
    healthLabel.Parent = frame
    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Size = UDim2.new(1, 0, 0, 16)
    distanceLabel.Position = UDim2.new(0, 0, 0, 38)
    distanceLabel.Text = "Dist: --"
    distanceLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.Font = Enum.Font.Gotham
    distanceLabel.TextSize = 11
    distanceLabel.Parent = frame
    local weaponLabel = Instance.new("TextLabel")
    weaponLabel.Size = UDim2.new(1, 0, 0, 16)
    weaponLabel.Position = UDim2.new(0, 0, 0, 54)
    weaponLabel.Text = ""
    weaponLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    weaponLabel.BackgroundTransparency = 1
    weaponLabel.Font = Enum.Font.Gotham
    weaponLabel.TextSize = 10
    weaponLabel.Parent = frame
    local highlight = Instance.new("Highlight")
    highlight.Name = "ESP_Highlight_" .. player.Name
    highlight.FillColor = settings.ESPBoxColor
    highlight.FillTransparency = 0.7
    highlight.OutlineColor = settings.ESPBoxColor
    highlight.OutlineTransparency = 0.2
    highlight.OutlineWidth = 2
    highlight.Parent = character
    local tracer = Drawing.new("Line")
    tracer.Visible = false
    tracer.Color = settings.ESPBoxColor
    tracer.Thickness = 1
    tracer.Transparency = 0.5
    espObjects[player] = {
        Billboard = billboard,
        Highlight = highlight,
        NameLabel = nameLabel,
        HealthLabel = healthLabel,
        DistanceLabel = distanceLabel,
        WeaponLabel = weaponLabel,
        Tracer = tracer
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
            if settings.TeamCheck and player.Team == LocalPlayer.Team then
                if espObjects[player] then
                    espObjects[player].Billboard:Destroy()
                    if espObjects[player].Highlight then espObjects[player].Highlight:Destroy() end
                    espObjects[player] = nil
                end
                continue
            end
            local character = player.Character
            if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
                local rootPart = character:FindFirstChild("HumanoidRootPart")
                if rootPart and localPos then
                    if (rootPart.Position - localPos).Magnitude > settings.ViewDistance then
                        if espObjects[player] then
                            espObjects[player].Billboard:Destroy()
                            if espObjects[player].Highlight then espObjects[player].Highlight:Destroy() end
                            espObjects[player] = nil
                        end
                        continue
                    end
                end
                if not espObjects[player] then
                    createESP(player)
                else
                    local humanoid = character:FindFirstChild("Humanoid")
                    local tool = character:FindFirstChildOfClass("Tool")
                    if espObjects[player].HealthLabel and humanoid then
                        espObjects[player].HealthLabel.Text = "HP: " .. math.floor(humanoid.Health)
                    end
                    if espObjects[player].DistanceLabel and rootPart and localPos then
                        local dist = (rootPart.Position - localPos).Magnitude
                        espObjects[player].DistanceLabel.Text = "Dist: " .. math.floor(dist) .. " studs"
                    end
                    if espObjects[player].WeaponLabel then
                        espObjects[player].WeaponLabel.Text = tool and tool.Name or ""
                    end
                    if espObjects[player].Tracer and camera then
                        local head = character:FindFirstChild("Head")
                        local localHead = localChar and localChar:FindFirstChild("Head")
                        if head and localHead then
                            local screenPos1 = camera:WorldToViewportPoint(localHead.Position)
                            local screenPos2 = camera:WorldToViewportPoint(head.Position)
                            espObjects[player].Tracer.From = Vector2.new(screenPos1.X, screenPos1.Y)
                            espObjects[player].Tracer.To = Vector2.new(screenPos2.X, screenPos2.Y)
                            espObjects[player].Tracer.Visible = true
                        end
                    end
                end
            elseif espObjects[player] then
                espObjects[player].Billboard:Destroy()
                if espObjects[player].Highlight then espObjects[player].Highlight:Destroy() end
                if espObjects[player].Tracer then
                    espObjects[player].Tracer:Remove()
                end
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
            if objects.Tracer then objects.Tracer:Remove() end
        end)
    end
    espObjects = {}
end

local function updateFOVCircle()
    if fovCircle then fovCircle:Destroy() end
    if not settings.ShowFOVCircle or settings.FOVRadius <= 0 then return end
    local circle = Drawing.new("Circle")
    circle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    circle.Radius = settings.FOVRadius
    circle.Color = Color3.fromRGB(255, 255, 255)
    circle.Thickness = 1
    circle.Transparency = 0.5
    circle.Filled = false
    circle.Visible = true
    fovCircle = circle
    local connection
    connection = RunService.RenderStepped:Connect(function()
        if not fovCircle or not settings.ShowFOVCircle then
            if connection then connection:Disconnect() end
            return
        end
        fovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
        fovCircle.Radius = settings.FOVRadius
    end)
end

local function createKeyScreen()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SnakeHubKeySystem"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = LocalPlayer.PlayerGui

    local background = Instance.new("Frame")
    background.Size = UDim2.new(1, 0, 1, 0)
    background.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    background.BackgroundTransparency = 0.7
    background.Parent = screenGui

    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 450, 0, 320)
    mainFrame.Position = UDim2.new(0.5, -225, 0.5, -160)
    mainFrame.BackgroundColor3 = Color3.fromRGB(18, 20, 30)
    mainFrame.BorderSizePixel = 0
    mainFrame.Parent = background

    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 15)
    mainCorner.Parent = mainFrame

    local mainStroke = Instance.new("UIStroke")
    mainStroke.Color = Color3.fromRGB(0, 162, 255)
    mainStroke.Thickness = 2
    mainStroke.Parent = mainFrame

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 60)
    title.Position = UDim2.new(0, 0, 0, 20)
    title.Text = "SNAKE HUB v2.0"
    title.TextColor3 = Color3.fromRGB(0, 200, 255)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBlack
    title.TextSize = 32
    title.Parent = mainFrame

    local subTitle = Instance.new("TextLabel")
    subTitle.Size = UDim2.new(1, 0, 0, 25)
    subTitle.Position = UDim2.new(0, 0, 0, 80)
    subTitle.Text = "MAD CITY CHAPTER 1"
    subTitle.TextColor3 = Color3.fromRGB(150, 180, 220)
    subTitle.BackgroundTransparency = 1
    subTitle.Font = Enum.Font.GothamBold
    subTitle.TextSize = 16
    subTitle.Parent = mainFrame

    local instruction = Instance.new("TextLabel")
    instruction.Size = UDim2.new(1, -40, 0, 30)
    instruction.Position = UDim2.new(0, 20, 0, 130)
    instruction.Text = "INSIRA A KEY DE ATIVACAO:"
    instruction.TextColor3 = Color3.fromRGB(200, 200, 210)
    instruction.BackgroundTransparency = 1
    instruction.Font = Enum.Font.Gotham
    instruction.TextSize = 14
    instruction.TextXAlignment = Enum.TextXAlignment.Left
    instruction.Parent = mainFrame

    local keyBox = Instance.new("TextBox")
    keyBox.Size = UDim2.new(1, -40, 0, 45)
    keyBox.Position = UDim2.new(0, 20, 0, 165)
    keyBox.PlaceholderText = "Digite a key..."
    keyBox.Text = ""
    keyBox.BackgroundColor3 = Color3.fromRGB(30, 33, 45)
    keyBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    keyBox.Font = Enum.Font.Gotham
    keyBox.TextSize = 16
    keyBox.BorderSizePixel = 0
    keyBox.Parent = mainFrame

    local keyCorner = Instance.new("UICorner")
    keyCorner.CornerRadius = UDim.new(0, 8)
    keyCorner.Parent = keyBox

    local keyStroke = Instance.new("UIStroke")
    keyStroke.Color = Color3.fromRGB(0, 100, 200)
    keyStroke.Thickness = 1
    keyStroke.Parent = keyBox

    local confirmBtn = Instance.new("TextButton")
    confirmBtn.Size = UDim2.new(0, 180, 0, 45)
    confirmBtn.Position = UDim2.new(0.5, -90, 0, 230)
    confirmBtn.Text = "ATIVAR HUB"
    confirmBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    confirmBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    confirmBtn.Font = Enum.Font.GothamBlack
    confirmBtn.TextSize = 18
    confirmBtn.BorderSizePixel = 0
    confirmBtn.Parent = mainFrame

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 8)
    btnCorner.Parent = confirmBtn

    local errorMsg = Instance.new("TextLabel")
    errorMsg.Size = UDim2.new(1, -40, 0, 30)
    errorMsg.Position = UDim2.new(0, 20, 0, 285)
    errorMsg.Text = ""
    errorMsg.TextColor3 = Color3.fromRGB(255, 60, 60)
    errorMsg.BackgroundTransparency = 1
    errorMsg.Font = Enum.Font.GothamBold
    errorMsg.TextSize = 13
    errorMsg.Visible = false
    errorMsg.Parent = mainFrame

    confirmBtn.MouseButton1Click:Connect(function()
        if keyBox.Text == REQUIRED_KEY then
            keyInserted = true
            screenGui:Destroy()
            loadHub()
        else
            errorMsg.Text = "KEY INCORRETA!"
            errorMsg.Visible = true
            keyBox.Text = ""
            task.wait(2)
            errorMsg.Visible = false
        end
    end)

    keyBox.FocusLost:Connect(function(enterPressed)
        if enterPressed and keyBox.Text == REQUIRED_KEY then
            keyInserted = true
            screenGui:Destroy()
            loadHub()
        elseif enterPressed then
            errorMsg.Text = "KEY INCORRETA!"
            errorMsg.Visible = true
            keyBox.Text = ""
            task.wait(2)
            errorMsg.Visible = false
        end
    end)
end

local function createUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "SnakeHubUI"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = LocalPlayer.PlayerGui

    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 360, 0, 620)
    mainFrame.Position = UDim2.new(0, 30, 0.5, -310)
    mainFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 25)
    mainFrame.BorderSizePixel = 0
    mainFrame.BackgroundTransparency = 0.02
    mainFrame.Parent = screenGui

    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 14)
    mainCorner.Parent = mainFrame

    local mainStroke = Instance.new("UIStroke")
    mainStroke.Color = Color3.fromRGB(0, 162, 255)
    mainStroke.Thickness = 1.5
    mainStroke.Transparency = 0.3
    mainStroke.Parent = mainFrame

    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 60)
    titleBar.Position = UDim2.new(0, 0, 0, 0)
    titleBar.BackgroundColor3 = Color3.fromRGB(10, 12, 20)
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame

    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 14)
    titleCorner.Parent = titleBar

    local titleGradient = Instance.new("UIGradient")
    titleGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 162, 255)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(0, 100, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 220, 255))
    })
    titleGradient.Parent = titleBar

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 1, 0)
    title.Text = "SNAKE HUB v2.0"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBlack
    title.TextSize = 24
    title.Parent = titleBar

    local version = Instance.new("TextLabel")
    version.Size = UDim2.new(0, 100, 0, 20)
    version.Position = UDim2.new(1, -105, 0, 35)
    version.Text = "MAD CITY CH.1"
    version.TextColor3 = Color3.fromRGB(130, 200, 255)
    version.BackgroundTransparency = 1
    version.Font = Enum.Font.GothamBold
    version.TextSize = 9
    version.Parent = titleBar

    local categoryFrame = Instance.new("ScrollingFrame")
    categoryFrame.Size = UDim2.new(0, 90, 1, -80)
    categoryFrame.Position = UDim2.new(0, 5, 0, 65)
    categoryFrame.BackgroundTransparency = 1
    categoryFrame.BorderSizePixel = 0
    categoryFrame.ScrollBarThickness = 0
    categoryFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    categoryFrame.Parent = mainFrame

    local contentScroll = Instance.new("ScrollingFrame")
    contentScroll.Size = UDim2.new(1, -100, 1, -110)
    contentScroll.Position = UDim2.new(0, 95, 0, 65)
    contentScroll.BackgroundTransparency = 1
    contentScroll.BorderSizePixel = 0
    contentScroll.ScrollBarThickness = 4
    contentScroll.ScrollBarImageColor3 = Color3.fromRGB(0, 162, 255)
    contentScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
    contentScroll.Parent = mainFrame

    local categories = {
        {Name = "AIMBOT", Icon = "🎯"},
        {Name = "FARM", Icon = "💰"},
        {Name = "MOVEMENT", Icon = "🏃"},
        {Name = "VISUALS", Icon = "👁️"},
        {Name = "COMBAT", Icon = "⚔️"},
        {Name = "WEAPONS", Icon = "🔫"},
        {Name = "PLAYER", Icon = "👤"},
        {Name = "MISC", Icon = "⚙️"},
        {Name = "KEYS", Icon = "⌨️"}
    }

    local categoryBtns = {}
    local activeCategory = "AIMBOT"
    local categoryContents = {}
    local totalCanvasHeight = 0

    for i, cat in ipairs(categories) do
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(1, -4, 0, 38)
        btn.Position = UDim2.new(0, 2, 0, (i-1) * 42)
        btn.Text = cat.Icon .. " " .. cat.Name
        btn.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
        btn.TextColor3 = Color3.fromRGB(180, 180, 200)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 10
        btn.BorderSizePixel = 0
        btn.TextXAlignment = Enum.TextXAlignment.Left
        btn.TextTruncate = Enum.TextTruncate.AtEnd
        btn.Parent = categoryFrame

        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 6)
        btnCorner.Parent = btn

        btn.MouseButton1Click:Connect(function()
            for _, b in ipairs(categoryBtns) do
                b.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
                b.TextColor3 = Color3.fromRGB(180, 180, 200)
            end
            btn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
            activeCategory = cat.Name
            for name, frame in pairs(categoryContents) do
                frame.Visible = (name == cat.Name)
            end
            contentScroll.CanvasSize = UDim2.new(0, 0, 0, totalCanvasHeight)
        end)

        table.insert(categoryBtns, btn)
    end

    categoryBtns[1].BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    categoryBtns[1].TextColor3 = Color3.fromRGB(255, 255, 255)

    local function addContent(categoryName, yPosition, height)
        if not categoryContents[categoryName] then
            local frame = Instance.new("Frame")
            frame.Size = UDim2.new(1, -10, 0, height)
            frame.Position = UDim2.new(0, 5, 0, yPosition)
            frame.BackgroundTransparency = 1
            frame.BorderSizePixel = 0
            frame.Name = categoryName
            frame.Visible = (categoryName == activeCategory)
            frame.Parent = contentScroll
            categoryContents[categoryName] = frame
            return frame
        end
        return categoryContents[categoryName]
    end

    local function createToggle(parent, y, text, default, callback)
        local toggleFrame = Instance.new("Frame")
        toggleFrame.Size = UDim2.new(1, -10, 0, 38)
        toggleFrame.Position = UDim2.new(0, 5, 0, y)
        toggleFrame.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
        toggleFrame.BorderSizePixel = 0
        toggleFrame.Parent = parent

        local toggleCorner = Instance.new("UICorner")
        toggleCorner.CornerRadius = UDim.new(0, 6)
        toggleCorner.Parent = toggleFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0.6, 0, 1, 0)
        label.Position = UDim2.new(0, 8, 0, 0)
        label.Text = text
        label.TextColor3 = Color3.fromRGB(210, 210, 220)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 12
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = toggleFrame

        local toggleBtn = Instance.new("TextButton")
        toggleBtn.Size = UDim2.new(0, 48, 0, 22)
        toggleBtn.Position = UDim2.new(1, -56, 0.5, -11)
        toggleBtn.Text = ""
        toggleBtn.BackgroundColor3 = default and Color3.fromRGB(0, 200, 100) or Color3.fromRGB(60, 62, 75)
        toggleBtn.BorderSizePixel = 0
        toggleBtn.Parent = toggleFrame

        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 11)
        btnCorner.Parent = toggleBtn

        local circle = Instance.new("Frame")
        circle.Size = UDim2.new(0, 16, 0, 16)
        circle.Position = default and UDim2.new(1, -19, 0.5, -8) or UDim2.new(0, 3, 0.5, -8)
        circle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        circle.BorderSizePixel = 0
        circle.Parent = toggleBtn

        local circleCorner = Instance.new("UICorner")
        circleCorner.CornerRadius = UDim.new(0, 8)
        circleCorner.Parent = circle

        local toggled = default
        toggleBtn.MouseButton1Click:Connect(function()
            toggled = not toggled
            if toggled then
                toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 100)
                circle:TweenPosition(UDim2.new(1, -19, 0.5, -8), "Out", "Quad", 0.12, true)
            else
                toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 62, 75)
                circle:TweenPosition(UDim2.new(0, 3, 0.5, -8), "Out", "Quad", 0.12, true)
            end
            callback(toggled)
        end)
    end

    local function createDropdown(parent, y, text, options, default, callback)
        local dropdownFrame = Instance.new("Frame")
        dropdownFrame.Size = UDim2.new(1, -10, 0, 38)
        dropdownFrame.Position = UDim2.new(0, 5, 0, y)
        dropdownFrame.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
        dropdownFrame.BorderSizePixel = 0
        dropdownFrame.Parent = parent

        local dropdownCorner = Instance.new("UICorner")
        dropdownCorner.CornerRadius = UDim.new(0, 6)
        dropdownCorner.Parent = dropdownFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0.5, 0, 1, 0)
        label.Position = UDim2.new(0, 8, 0, 0)
        label.Text = text
        label.TextColor3 = Color3.fromRGB(210, 210, 220)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 12
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = dropdownFrame

        local selectedBtn = Instance.new("TextButton")
        selectedBtn.Size = UDim2.new(0, 90, 0, 26)
        selectedBtn.Position = UDim2.new(1, -98, 0.5, -13)
        selectedBtn.Text = default
        selectedBtn.BackgroundColor3 = Color3.fromRGB(35, 38, 52)
        selectedBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        selectedBtn.Font = Enum.Font.Gotham
        selectedBtn.TextSize = 10
        selectedBtn.BorderSizePixel = 0
        selectedBtn.Parent = dropdownFrame

        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 5)
        btnCorner.Parent = selectedBtn

        local isOpen = false
        local dropList = nil

        selectedBtn.MouseButton1Click:Connect(function()
            if isOpen then
                if dropList then dropList:Destroy() end
                isOpen = false
                return
            end
            dropList = Instance.new("Frame")
            dropList.Size = UDim2.new(0, 90, 0, #options * 28)
            dropList.Position = UDim2.new(1, -98, 0, 38)
            dropList.BackgroundColor3 = Color3.fromRGB(35, 38, 52)
            dropList.BorderSizePixel = 0
            dropList.ZIndex = 15
            dropList.Parent = parent

            local listCorner = Instance.new("UICorner")
            listCorner.CornerRadius = UDim.new(0, 5)
            listCorner.Parent = dropList

            for i, option in ipairs(options) do
                local optionBtn = Instance.new("TextButton")
                optionBtn.Size = UDim2.new(1, 0, 0, 28)
                optionBtn.Position = UDim2.new(0, 0, 0, (i-1)*28)
                optionBtn.Text = option
                optionBtn.BackgroundColor3 = Color3.fromRGB(40, 43, 58)
                optionBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
                optionBtn.Font = Enum.Font.Gotham
                optionBtn.TextSize = 10
                optionBtn.BorderSizePixel = 0
                optionBtn.ZIndex = 16
                optionBtn.Parent = dropList

                optionBtn.MouseButton1Click:Connect(function()
                    selectedBtn.Text = option
                    callback(option)
                    dropList:Destroy()
                    isOpen = false
                end)
            end
            isOpen = true
        end)
    end

    local function createSlider(parent, y, text, min, max, default, callback)
        local sliderFrame = Instance.new("Frame")
        sliderFrame.Size = UDim2.new(1, -10, 0, 50)
        sliderFrame.Position = UDim2.new(0, 5, 0, y)
        sliderFrame.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
        sliderFrame.BorderSizePixel = 0
        sliderFrame.Parent = parent

        local sliderCorner = Instance.new("UICorner")
        sliderCorner.CornerRadius = UDim.new(0, 6)
        sliderCorner.Parent = sliderFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(1, -20, 0, 20)
        label.Position = UDim2.new(0, 8, 0, 3)
        label.Text = text .. ": " .. tostring(default)
        label.TextColor3 = Color3.fromRGB(200, 200, 210)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 11
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = sliderFrame

        local inputBox = Instance.new("TextBox")
        inputBox.Size = UDim2.new(0, 55, 0, 22)
        inputBox.Position = UDim2.new(1, -63, 0, 25)
        inputBox.Text = tostring(default)
        inputBox.BackgroundColor3 = Color3.fromRGB(35, 38, 52)
        inputBox.TextColor3 = Color3.fromRGB(255, 255, 255)
        inputBox.Font = Enum.Font.Gotham
        inputBox.TextSize = 10
        inputBox.BorderSizePixel = 0
        inputBox.Parent = sliderFrame

        local boxCorner = Instance.new("UICorner")
        boxCorner.CornerRadius = UDim.new(0, 4)
        boxCorner.Parent = inputBox

        inputBox.FocusLost:Connect(function(enterPressed)
            if enterPressed then
                local num = tonumber(inputBox.Text)
                if num and num >= min and num <= max then
                    label.Text = text .. ": " .. tostring(num)
                    callback(num)
                else
                    inputBox.Text = tostring(default)
                end
            end
        end)
    end

    local function createKeybind(parent, y, text, defaultKey, callback)
        local keyFrame = Instance.new("Frame")
        keyFrame.Size = UDim2.new(1, -10, 0, 38)
        keyFrame.Position = UDim2.new(0, 5, 0, y)
        keyFrame.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
        keyFrame.BorderSizePixel = 0
        keyFrame.Parent = parent

        local keyCorner = Instance.new("UICorner")
        keyCorner.CornerRadius = UDim.new(0, 6)
        keyCorner.Parent = keyFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0.5, 0, 1, 0)
        label.Position = UDim2.new(0, 8, 0, 0)
        label.Text = text
        label.TextColor3 = Color3.fromRGB(210, 210, 220)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.Gotham
        label.TextSize = 12
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = keyFrame

        local keyBtn = Instance.new("TextButton")
        keyBtn.Size = UDim2.new(0, 75, 0, 26)
        keyBtn.Position = UDim2.new(1, -83, 0.5, -13)
        keyBtn.Text = defaultKey
        keyBtn.BackgroundColor3 = Color3.fromRGB(35, 38, 52)
        keyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        keyBtn.Font = Enum.Font.GothamBold
        keyBtn.TextSize = 11
        keyBtn.BorderSizePixel = 0
        keyBtn.Parent = keyFrame

        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 5)
        btnCorner.Parent = keyBtn

        local listening = false
        keyBtn.MouseButton1Click:Connect(function()
            listening = true
            keyBtn.Text = "..."
            keyBtn.BackgroundColor3 = Color3.fromRGB(200, 150, 0)
        end)

        UserInputService.InputBegan:Connect(function(input)
            if listening and input.KeyCode ~= Enum.KeyCode.Unknown then
                local keyName = input.KeyCode.Name
                keyBtn.Text = keyName
                keyBtn.BackgroundColor3 = Color3.fromRGB(35, 38, 52)
                listening = false
                callback(input.KeyCode, keyName)
            end
        end)
    end

    local aimbotFrame = addContent("AIMBOT", 5, 520)
    createToggle(aimbotFrame, 5, "Enable Aimbot", settings.AimbotEnabled, function(val) settings.AimbotEnabled = val end)
    createToggle(aimbotFrame, 48, "Weapon Aim", settings.WeaponAimbotEnabled, function(val) settings.WeaponAimbotEnabled = val end)
    createDropdown(aimbotFrame, 91, "Target Part", bodyParts, settings.TargetPart, function(val) settings.TargetPart = val end)
    createDropdown(aimbotFrame, 134, "Target Mode", {"Auto", "Manual"}, settings.TargetMode, function(val) settings.TargetMode = val end)
    createSlider(aimbotFrame, 177, "Camera Smooth", 0.05, 0.8, settings.Smoothness, function(val) settings.Smoothness = val end)
    createSlider(aimbotFrame, 232, "Weapon Smooth", 0.05, 0.8, settings.WeaponSmoothness, function(val) settings.WeaponSmoothness = val end)
    createSlider(aimbotFrame, 287, "FOV Radius", 0, 800, settings.FOVRadius, function(val) settings.FOVRadius = val; updateFOVCircle() end)
    createToggle(aimbotFrame, 342, "Show FOV Circle", settings.ShowFOVCircle, function(val) settings.ShowFOVCircle = val; updateFOVCircle() end)
    createToggle(aimbotFrame, 385, "Team Check", settings.TeamCheck, function(val) settings.TeamCheck = val end)
    createSlider(aimbotFrame, 428, "View Distance", 100, 2000, settings.ViewDistance, function(val) settings.ViewDistance = val end)

    local farmFrame = addContent("FARM", 5, 360)
    createToggle(farmFrame, 5, "Auto Farm Money", false, function(val)
        settings.AutoFarmMoney = val
        if val then startAutoFarmMoney() else stopAutoFarmMoney() end
    end)
    createToggle(farmFrame, 48, "Auto Farm Level", false, function(val)
        settings.AutoFarmLevel = val
        if val then startAutoFarmLevel() else stopAutoFarmLevel() end
    end)
    createToggle(farmFrame, 91, "Auto Collect Loot", false, function(val) settings.AutoCollectLoot = val end)
    createToggle(farmFrame, 134, "Auto Sell Items", false, function(val) settings.AutoSellItems = val end)
    createSlider(farmFrame, 177, "Farm Loop Delay", 1, 30, settings.FarmLoopDelay, function(val) settings.FarmLoopDelay = val end)
    createToggle(farmFrame, 232, "Auto Rob", false, function(val) settings.AutoRobEnabled = val end)
    createToggle(farmFrame, 275, "Auto Arrest", false, function(val) settings.AutoArrest = val end)

    local movementFrame = addContent("MOVEMENT", 5, 330)
    createToggle(movementFrame, 5, "Fly", false, function(val)
        settings.FlyEnabled = val
        if val then setupFly() else disableFly() end
    end)
    createSlider(movementFrame, 48, "Fly Speed", 10, 200, settings.FlySpeed, function(val) settings.FlySpeed = val end)
    createToggle(movementFrame, 103, "Noclip", false, function(val)
        settings.NoclipEnabled = val
        if val then setupNoclip() else disableNoclip() end
    end)
    createSlider(movementFrame, 146, "Noclip Speed", 16, 200, settings.NoClipSpeed, function(val) settings.NoClipSpeed = val end)
    createToggle(movementFrame, 201, "Spin Bot", false, function(val)
        settings.SpinBot = val
        setupSpinBot()
    end)
    createSlider(movementFrame, 244, "Spin Speed", 10, 500, settings.SpinSpeed, function(val) settings.SpinSpeed = val end)
    createToggle(movementFrame, 299, "Teleport to Player", false, function(val) settings.TeleportToPlayer = val end)

    local visualsFrame = addContent("VISUALS", 5, 430)
    createToggle(visualsFrame, 5, "ESP", settings.ESPEnabled, function(val) settings.ESPEnabled = val; if not val then clearESP() end end)
    createToggle(visualsFrame, 48, "Rainbow ESP", settings.RainbowESP, function(val)
        settings.RainbowESP = val
        if val then setupRainbowESP() end
    end)
    createSlider(visualsFrame, 91, "Rainbow Speed", 0.1, 5, settings.RainbowSpeed, function(val) settings.RainbowSpeed = val end)
    createDropdown(visualsFrame, 146, "ESP Box Color", {"Blue", "Red", "Green", "Yellow", "Purple", "White", "Orange", "Cyan"}, "Blue", function(val)
        local colors = {
            Blue = Color3.fromRGB(0, 162, 255), Red = Color3.fromRGB(255, 50, 50),
            Green = Color3.fromRGB(50, 255, 50), Yellow = Color3.fromRGB(255, 255, 50),
            Purple = Color3.fromRGB(180, 50, 255), White = Color3.fromRGB(255, 255, 255),
            Orange = Color3.fromRGB(255, 150, 0), Cyan = Color3.fromRGB(0, 255, 200)
        }
        settings.ESPBoxColor = colors[val] or Color3.fromRGB(0, 162, 255)
    end)
    createDropdown(visualsFrame, 189, "ESP Text Color", {"White", "Blue", "Red", "Green", "Yellow", "Purple"}, "White", function(val)
        local colors = {
            White = Color3.fromRGB(255, 255, 255), Blue = Color3.fromRGB(0, 162, 255),
            Red = Color3.fromRGB(255, 50, 50), Green = Color3.fromRGB(50, 255, 50),
            Yellow = Color3.fromRGB(255, 255, 50), Purple = Color3.fromRGB(180, 50, 255)
        }
        settings.ESPTextColor = colors[val] or Color3.fromRGB(255, 255, 255)
    end)
    createToggle(visualsFrame, 232, "Crosshair", settings.CrosshairEnabled, function(val)
        settings.CrosshairEnabled = val
        setupCrosshair()
    end)
    createSlider(visualsFrame, 275, "Crosshair Size", 5, 50, settings.CrosshairSize, function(val) settings.CrosshairSize = val; setupCrosshair() end)
    createSlider(visualsFrame, 330, "Crosshair Thickness", 1, 10, settings.CrosshairThickness, function(val) settings.CrosshairThickness = val; setupCrosshair() end)
    createToggle(visualsFrame, 385, "Third Person", settings.ThirdPerson, function(val)
        settings.ThirdPerson = val
        setupThirdPerson()
    end)
    createSlider(visualsFrame, 420, "Third Person Distance", 2, 20, settings.ThirdPersonDistance, function(val) settings.ThirdPersonDistance = val end)

    local combatFrame = addContent("COMBAT", 5, 300)
    createToggle(combatFrame, 5, "Kill Aura", false, function(val)
        settings.KillAura = val
        setupKillAura()
    end)
    createSlider(combatFrame, 48, "Kill Aura Range", 5, 50, settings.KillAuraRange, function(val) settings.KillAuraRange = val end)
    createToggle(combatFrame, 103, "Trigger Bot", false, function(val)
        settings.TriggerBot = val
        setupTriggerBot()
    end)
    createToggle(combatFrame, 146, "Rapid Fire", false, function(val)
        settings.RapidFire = val
        setupRapidFire()
    end)
    createToggle(combatFrame, 189, "No Recoil", false, function(val)
        settings.NoRecoil = val
        setupNoRecoil()
    end)
    createToggle(combatFrame, 232, "Infinite Ammo", false, function(val)
        settings.InfiniteAmmo = val
        setupInfiniteAmmo()
    end)

    local weaponsFrame = addContent("WEAPONS", 5, 250)
    createToggle(weaponsFrame, 5, "Teleport Gun", false, function(val)
        if val then teleportGunToPlayer("") end
    end)
    createToggle(weaponsFrame, 48, "Teleport All Guns", false, function(val)
        settings.TeleportAllGuns = val
        if val then teleportAllGunsToPlayer() end
    end)

    local gunBox = Instance.new("TextBox")
    gunBox.Size = UDim2.new(1, -10, 0, 35)
    gunBox.Position = UDim2.new(0, 5, 0, 95)
    gunBox.PlaceholderText = "Nome da arma..."
    gunBox.Text = ""
    gunBox.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
    gunBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    gunBox.Font = Enum.Font.Gotham
    gunBox.TextSize = 12
    gunBox.BorderSizePixel = 0
    gunBox.Parent = weaponsFrame

    local gunCorner = Instance.new("UICorner")
    gunCorner.CornerRadius = UDim.new(0, 6)
    gunCorner.Parent = gunBox

    local teleportBtn = Instance.new("TextButton")
    teleportBtn.Size = UDim2.new(1, -10, 0, 35)
    teleportBtn.Position = UDim2.new(0, 5, 0, 135)
    teleportBtn.Text = "TELEPORT ARMA ESPECIFICA"
    teleportBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    teleportBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    teleportBtn.Font = Enum.Font.GothamBold
    teleportBtn.TextSize = 12
    teleportBtn.BorderSizePixel = 0
    teleportBtn.Parent = weaponsFrame

    local teleportCorner = Instance.new("UICorner")
    teleportCorner.CornerRadius = UDim.new(0, 6)
    teleportCorner.Parent = teleportBtn

    teleportBtn.MouseButton1Click:Connect(function()
        teleportGunToPlayer(gunBox.Text)
    end)

    local scanBtn = Instance.new("TextButton")
    scanBtn.Size = UDim2.new(1, -10, 0, 35)
    scanBtn.Position = UDim2.new(0, 5, 0, 175)
    scanBtn.Text = "ESCANEAR ARMAS"
    scanBtn.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
    scanBtn.TextColor3 = Color3.fromRGB(210, 210, 220)
    scanBtn.Font = Enum.Font.Gotham
    scanBtn.TextSize = 12
    scanBtn.BorderSizePixel = 0
    scanBtn.Parent = weaponsFrame

    local scanCorner = Instance.new("UICorner")
    scanCorner.CornerRadius = UDim.new(0, 6)
    scanCorner.Parent = scanBtn

    scanBtn.MouseButton1Click:Connect(function() scanForGuns() end)

    local playerFrame = addContent("PLAYER", 5, 250)
    createToggle(playerFrame, 5, "God Mode", false, function(val)
        settings.GodMode = val
        setupGodMode()
    end)
    createToggle(playerFrame, 48, "Infinite Jump", false, function(val)
        settings.InfiniteJump = val
        setupInfiniteJump()
    end)
    createSlider(playerFrame, 91, "Walk Speed", 16, 200, settings.WalkSpeed, function(val) settings.WalkSpeed = val; setupWalkSpeed() end)
    createSlider(playerFrame, 146, "Jump Power", 50, 300, settings.JumpPower, function(val) settings.JumpPower = val; setupWalkSpeed() end)
    createSlider(playerFrame, 201, "FOV Changer", 30, 120, settings.FOVChanger, function(val) settings.FOVChanger = val; setupFOV() end)

    local miscFrame = addContent("MISC", 5, 250)
    createToggle(miscFrame, 5, "Anti AFK", settings.AntiAfk, function(val)
        settings.AntiAfk = val
        if val then setupAntiAfk() end
    end)
    createToggle(miscFrame, 48, "Auto Respawn", settings.AutoRespawn, function(val)
        settings.AutoRespawn = val
        if val then setupAutoRespawn() end
    end)
    createToggle(miscFrame, 91, "Chat Spy", settings.ChatSpyEnabled, function(val)
        settings.ChatSpyEnabled = val
        if val then setupChatSpy() end
    end)

    local keysFrame = addContent("KEYS", 5, 220)
    createKeybind(keysFrame, 5, "Aimbot Key", settings.AimbotKeyName, function(keyCode, keyName)
        settings.AimbotKey = keyCode
        settings.AimbotKeyName = keyName
    end)
    createKeybind(keysFrame, 48, "Fly Key", settings.FlyKeyName, function(keyCode, keyName)
        settings.FlyKey = keyCode
        settings.FlyKeyName = keyName
    end)
    createKeybind(keysFrame, 91, "Noclip Key", settings.NoclipKeyName, function(keyCode, keyName)
        settings.NoclipKey = keyCode
        settings.NoclipKeyName = keyName
    end)
    createKeybind(keysFrame, 134, "Teleport Gun Key", settings.TeleportGunKeyName, function(keyCode, keyName)
        settings.TeleportGunKey = keyCode
        settings.TeleportGunKeyName = keyName
    end)

    totalCanvasHeight = 520

    local bottomBar = Instance.new("Frame")
    bottomBar.Size = UDim2.new(1, 0, 0, 32)
    bottomBar.Position = UDim2.new(0, 0, 1, -32)
    bottomBar.BackgroundColor3 = Color3.fromRGB(10, 12, 20)
    bottomBar.BorderSizePixel = 0
    bottomBar.Parent = mainFrame

    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 80, 0, 24)
    closeBtn.Position = UDim2.new(1, -88, 0.5, -12)
    closeBtn.Text = "HIDE UI"
    closeBtn.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.TextSize = 10
    closeBtn.BorderSizePixel = 0
    closeBtn.Parent = bottomBar

    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 4)
    closeCorner.Parent = closeBtn

    closeBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = false
    end)

    local openBtn = Instance.new("TextButton")
    openBtn.Size = UDim2.new(0, 45, 0, 45)
    openBtn.Position = UDim2.new(0, 34, 0.5, -22)
    openBtn.Text = "S"
    openBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    openBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    openBtn.Font = Enum.Font.GothamBlack
    openBtn.TextSize = 24
    openBtn.BorderSizePixel = 0
    openBtn.Parent = screenGui

    local openCorner = Instance.new("UICorner")
    openCorner.CornerRadius = UDim.new(0, 22)
    openCorner.Parent = openBtn

    local openStroke = Instance.new("UIStroke")
    openStroke.Thickness = 2
    openStroke.Color = Color3.fromRGB(0, 220, 255)
    openStroke.Parent = openBtn

    openBtn.MouseButton1Click:Connect(function()
        mainFrame.Visible = true
    end)

    local dragging = false
    local dragStart = nil
    local frameStart = nil

    titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            frameStart = mainFrame.Position
        end
    end)

    titleBar.InputEnded:Connect(function(input)
        dragging = false
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = input.Position - dragStart
            mainFrame.Position = UDim2.new(frameStart.X.Scale, frameStart.X.Offset + delta.X, frameStart.Y.Scale, frameStart.Y.Offset + delta.Y)
        end
    end)

    return screenGui
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if not hubLoaded then return end
    if input.KeyCode == settings.AimbotKey then
        aimbotHeld = true
    end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = true
        currentWeapon = getCurrentWeapon()
    end
    if input.KeyCode == settings.FlyKey then
        settings.FlyEnabled = not settings.FlyEnabled
        if settings.FlyEnabled then setupFly() else disableFly() end
    end
    if input.KeyCode == settings.NoclipKey then
        settings.NoclipEnabled = not settings.NoclipEnabled
        if settings.NoclipEnabled then setupNoclip() else disableNoclip() end
    end
    if input.KeyCode == settings.TeleportGunKey then
        teleportGunToPlayer("")
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if input.KeyCode == settings.AimbotKey then
        aimbotHeld = false
    end
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        isAiming = false
    end
end)

function loadHub()
    createUI()
    updateFOVCircle()
    setupCrosshair()
    hubLoaded = true
    setupWalkSpeed()
    setupFOV()
    setupThirdPerson()
    setupAntiAfk()
    setupAutoRespawn()

    RunService.RenderStepped:Connect(function()
        if not hubLoaded then return end
        updateCameraAimbot()
        updateESP()
        updateFly()
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

    Players.PlayerAdded:Connect(function()
        if hubLoaded then
            setupWalkSpeed()
            setupFOV()
        end
    end)

    Players.PlayerRemoving:Connect(function(player)
        if espObjects[player] then
            pcall(function()
                espObjects[player].Billboard:Destroy()
                if espObjects[player].Highlight then espObjects[player].Highlight:Destroy() end
            end)
            espObjects[player] = nil
        end
    end)

    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(0.3)
        setupWalkSpeed()
        setupFOV()
        setupThirdPerson()
        if settings.InfiniteJump then setupInfiniteJump() end
        if settings.GodMode then setupGodMode() end
        if settings.FlyEnabled then setupFly() end
        if settings.NoclipEnabled then setupNoclip() end
    end)
end

createKeyScreen()
local function createNotification(text, duration)
    local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
    if not screenGui then return end
    
    local notification = Instance.new("Frame")
    notification.Size = UDim2.new(0, 300, 0, 40)
    notification.Position = UDim2.new(0.5, -150, 1, -50)
    notification.BackgroundColor3 = Color3.fromRGB(15, 17, 25)
    notification.BorderSizePixel = 0
    notification.Parent = screenGui
    
    local notifCorner = Instance.new("UICorner")
    notifCorner.CornerRadius = UDim.new(0, 8)
    notifCorner.Parent = notification
    
    local notifStroke = Instance.new("UIStroke")
    notifStroke.Color = Color3.fromRGB(0, 255, 100)
    notifStroke.Thickness = 1
    notifStroke.Parent = notification
    
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -10, 1, 0)
    label.Position = UDim2.new(0, 5, 0, 0)
    label.Text = text
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Gotham
    label.TextSize = 13
    label.Parent = notification
    
    notification:TweenPosition(UDim2.new(0.5, -150, 0.85, 0), "Out", "Back", 0.3, true)
    
    task.spawn(function()
        task.wait(duration or 3)
        notification:TweenPosition(UDim2.new(0.5, -150, 1, 50), "In", "Back", 0.3, true)
        task.wait(0.3)
        notification:Destroy()
    end)
end

local function teleportToPlayer(targetPlayer)
    if not targetPlayer then return end
    local character = LocalPlayer.Character
    local targetChar = targetPlayer.Character
    if not character or not targetChar then return end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    local targetRoot = targetChar:FindFirstChild("HumanoidRootPart")
    if not rootPart or not targetRoot then return end
    rootPart.CFrame = targetRoot.CFrame + Vector3.new(0, 3, 0)
    createNotification("Teleportado para " .. targetPlayer.Name, 2)
end

local function sellAllItems()
    local character = LocalPlayer.Character
    if not character then return end
    local sellPoints = {}
    for _, descendant in ipairs(workspace:GetDescendants()) do
        if descendant:IsA("BasePart") or descendant:IsA("Model") then
            local name = descendant.Name:lower()
            if name:find("sell") or name:find("shop") or name:find("dealer") or name:find("merchant") or name:find("store") then
                table.insert(sellPoints, descendant)
            end
        end
        if descendant:FindFirstChild("ClickDetector") then
            local parentName = descendant.Parent and descendant.Parent.Name:lower() or ""
            if parentName:find("sell") or parentName:find("shop") or parentName:find("dealer") then
                table.insert(sellPoints, descendant)
            end
        end
    end
    for _, point in ipairs(sellPoints) do
        if point:FindFirstChild("ClickDetector") then
            fireclickdetector(point.ClickDetector)
        end
    end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    for _, descendant in ipairs(workspace:GetDescendants()) do
        if descendant:IsA("BasePart") then
            local name = descendant.Name:lower()
            if name:find("sell") or name:find("shop") then
                local dist = (descendant.Position - rootPart.Position).Magnitude
                if dist <= 20 then
                    firetouchinterest(rootPart, descendant, 0)
                    firetouchinterest(rootPart, descendant, 1)
                end
            end
        end
    end
    createNotification("Itens vendidos!", 2)
end

local function autoRobNearest()
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local rootPart = character.HumanoidRootPart
    local nearestRob = nil
    local nearestDist = math.huge
    for _, descendant in ipairs(workspace:GetDescendants()) do
        if descendant:IsA("ProximityPrompt") then
            local parent = descendant.Parent
            if parent and parent:IsA("BasePart") then
                local dist = (parent.Position - rootPart.Position).Magnitude
                if dist < nearestDist then
                    nearestDist = dist
                    nearestRob = descendant
                end
            end
        end
    end
    if nearestRob and nearestDist <= 15 then
        fireproximityprompt(nearestRob)
        createNotification("Roubando local...", 1.5)
    end
end

local function autoArrestNearest()
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local rootPart = character.HumanoidRootPart
    local nearestCriminal = nil
    local nearestDist = math.huge
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local targetChar = player.Character
            if targetChar then
                local targetRoot = targetChar:FindFirstChild("HumanoidRootPart")
                if targetRoot then
                    local dist = (targetRoot.Position - rootPart.Position).Magnitude
                    if dist < nearestDist and dist <= 15 then
                        nearestDist = dist
                        nearestCriminal = player
                    end
                end
            end
        end
    end
    if nearestCriminal then
        local tool = character:FindFirstChildOfClass("Tool")
        if tool and tool.Name:lower():find("handcuff") or tool.Name:lower():find("arrest") then
            tool:Activate()
            task.wait(0.3)
            tool:Deactivate()
            createNotification("Prendeu " .. nearestCriminal.Name, 2)
        end
    end
end

local function startAutoRob()
    if settings.AutoRobEnabled then
        local autoRobConnection
        autoRobConnection = RunService.Heartbeat:Connect(function()
            if not settings.AutoRobEnabled then
                autoRobConnection:Disconnect()
                return
            end
            autoRobNearest()
            task.wait(2)
        end)
        table.insert(farmConnections, autoRobConnection)
    end
end

local function startAutoArrest()
    if settings.AutoArrest then
        local autoArrestConnection
        autoArrestConnection = RunService.Heartbeat:Connect(function()
            if not settings.AutoArrest then
                autoArrestConnection:Disconnect()
                return
            end
            autoArrestNearest()
            task.wait(3)
        end)
        table.insert(farmConnections, autoArrestConnection)
    end
end

local function startTeleportToPlayer()
    if settings.TeleportToPlayer then
        local teleportConnection
        teleportConnection = RunService.Heartbeat:Connect(function()
            if not settings.TeleportToPlayer then
                teleportConnection:Disconnect()
                return
            end
            if settings.SelectedPlayer then
                local target = Players:FindFirstChild(settings.SelectedPlayer)
                if target then
                    teleportToPlayer(target)
                    task.wait(5)
                end
            end
        end)
        table.insert(farmConnections, teleportConnection)
    end
end

local function startAutoCollectLoot()
    if settings.AutoCollectLoot then
        local collectConnection
        collectConnection = RunService.Heartbeat:Connect(function()
            if not settings.AutoCollectLoot then
                collectConnection:Disconnect()
                return
            end
            local character = LocalPlayer.Character
            if not character or not character:FindFirstChild("HumanoidRootPart") then return end
            local rootPart = character.HumanoidRootPart
            for _, descendant in ipairs(workspace:GetDescendants()) do
                if descendant:IsA("BasePart") or descendant:IsA("MeshPart") then
                    local dist = (descendant.Position - rootPart.Position).Magnitude
                    if dist <= 15 then
                        local name = descendant.Name:lower()
                        if name:find("money") or name:find("cash") or name:find("gold") or name:find("loot") or name:find("item") then
                            firetouchinterest(rootPart, descendant, 0)
                            firetouchinterest(rootPart, descendant, 1)
                        end
                        if descendant:FindFirstChild("ClickDetector") then
                            fireclickdetector(descendant.ClickDetector)
                        end
                    end
                end
            end
            task.wait(1)
        end)
        table.insert(farmConnections, collectConnection)
    end
end

local function startAutoSellItems()
    if settings.AutoSellItems then
        local sellConnection
        sellConnection = RunService.Heartbeat:Connect(function()
            if not settings.AutoSellItems then
                sellConnection:Disconnect()
                return
            end
            sellAllItems()
            task.wait(30)
        end)
        table.insert(farmConnections, sellConnection)
    end
end

local function createPlayerList()
    local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
    if not screenGui then return end
    
    local playerListFrame = Instance.new("Frame")
    playerListFrame.Name = "PlayerListFrame"
    playerListFrame.Size = UDim2.new(0, 200, 0, 300)
    playerListFrame.Position = UDim2.new(1, -220, 0, 100)
    playerListFrame.BackgroundColor3 = Color3.fromRGB(15, 17, 25)
    playerListFrame.BackgroundTransparency = 0.05
    playerListFrame.BorderSizePixel = 0
    playerListFrame.Visible = false
    playerListFrame.Parent = screenGui
    
    local listCorner = Instance.new("UICorner")
    listCorner.CornerRadius = UDim.new(0, 10)
    listCorner.Parent = playerListFrame
    
    local listStroke = Instance.new("UIStroke")
    listStroke.Color = Color3.fromRGB(0, 162, 255)
    listStroke.Thickness = 1.5
    listStroke.Parent = playerListFrame
    
    local listTitle = Instance.new("TextLabel")
    listTitle.Size = UDim2.new(1, 0, 0, 30)
    listTitle.Position = UDim2.new(0, 0, 0, 5)
    listTitle.Text = "PLAYERS ONLINE"
    listTitle.TextColor3 = Color3.fromRGB(0, 200, 255)
    listTitle.BackgroundTransparency = 1
    listTitle.Font = Enum.Font.GothamBold
    listTitle.TextSize = 14
    listTitle.Parent = playerListFrame
    
    local scrollingFrame = Instance.new("ScrollingFrame")
    scrollingFrame.Size = UDim2.new(1, -10, 1, -45)
    scrollingFrame.Position = UDim2.new(0, 5, 0, 40)
    scrollingFrame.BackgroundTransparency = 1
    scrollingFrame.BorderSizePixel = 0
    scrollingFrame.ScrollBarThickness = 5
    scrollingFrame.ScrollBarImageColor3 = Color3.fromRGB(0, 162, 255)
    scrollingFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    scrollingFrame.Parent = playerListFrame
    
    local function updateList()
        for _, child in ipairs(scrollingFrame:GetChildren()) do
            if child:IsA("TextButton") then
                child:Destroy()
            end
        end
        local yOffset = 0
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local playerBtn = Instance.new("TextButton")
                playerBtn.Size = UDim2.new(1, 0, 0, 30)
                playerBtn.Position = UDim2.new(0, 0, 0, yOffset)
                playerBtn.Text = player.Name
                playerBtn.BackgroundColor3 = Color3.fromRGB(25, 28, 40)
                playerBtn.TextColor3 = Color3.fromRGB(220, 220, 230)
                playerBtn.Font = Enum.Font.Gotham
                playerBtn.TextSize = 11
                playerBtn.BorderSizePixel = 0
                playerBtn.Parent = scrollingFrame
                
                local btnCorner = Instance.new("UICorner")
                btnCorner.CornerRadius = UDim.new(0, 4)
                btnCorner.Parent = playerBtn
                
                playerBtn.MouseButton1Click:Connect(function()
                    if settings.TargetMode == "Manual" then
                        settings.SelectedPlayer = player.Name
                        updateList()
                    end
                end)
                
                playerBtn.MouseButton2Click:Connect(function()
                    teleportToPlayer(player)
                end)
                
                yOffset = yOffset + 35
            end
        end
        scrollingFrame.CanvasSize = UDim2.new(0, 0, 0, yOffset)
    end
    
    updateList()
    Players.PlayerAdded:Connect(updateList)
    Players.PlayerRemoving:Connect(updateList)
    
    local toggleBtn = Instance.new("TextButton")
    toggleBtn.Size = UDim2.new(0, 45, 0, 45)
    toggleBtn.Position = UDim2.new(0, 90, 0.5, -22)
    toggleBtn.Text = "P"
    toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    toggleBtn.Font = Enum.Font.GothamBlack
    toggleBtn.TextSize = 22
    toggleBtn.BorderSizePixel = 0
    toggleBtn.Parent = screenGui
    
    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 22)
    toggleCorner.Parent = toggleBtn
    
    toggleBtn.MouseButton1Click:Connect(function()
        playerListFrame.Visible = not playerListFrame.Visible
        updateList()
    end)
    
    return playerListFrame
end

local function saveSettings()
    local saveData = {
        AimbotEnabled = settings.AimbotEnabled,
        ESPEnabled = settings.ESPEnabled,
        WeaponAimbotEnabled = settings.WeaponAimbotEnabled,
        TargetPart = settings.TargetPart,
        Smoothness = settings.Smoothness,
        WeaponSmoothness = settings.WeaponSmoothness,
        FOVRadius = settings.FOVRadius,
        ShowFOVCircle = settings.ShowFOVCircle,
        FlySpeed = settings.FlySpeed,
        FOVChanger = settings.FOVChanger,
        WalkSpeed = settings.WalkSpeed,
        JumpPower = settings.JumpPower,
        ThirdPersonDistance = settings.ThirdPersonDistance,
        CrosshairSize = settings.CrosshairSize,
        CrosshairThickness = settings.CrosshairThickness,
        KillAuraRange = settings.KillAuraRange,
        SpinSpeed = settings.SpinSpeed,
        RainbowSpeed = settings.RainbowSpeed,
        FarmLoopDelay = settings.FarmLoopDelay,
        ViewDistance = settings.ViewDistance
    }
    local json = HttpService:JSONEncode(saveData)
    if writefile then
        pcall(function()
            writefile("SnakeHub_MadCity_Settings.json", json)
        end)
    end
end

local function loadSettings()
    if readfile and isfile and isfile("SnakeHub_MadCity_Settings.json") then
        pcall(function()
            local json = readfile("SnakeHub_MadCity_Settings.json")
            local saveData = HttpService:JSONDecode(json)
            for key, value in pairs(saveData) do
                if settings[key] ~= nil then
                    settings[key] = value
                end
            end
        end)
    end
end

local function createWatermark()
    local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
    if not screenGui then return end
    
    local watermark = Instance.new("Frame")
    watermark.Name = "Watermark"
    watermark.Size = UDim2.new(0, 200, 0, 30)
    watermark.Position = UDim2.new(1, -210, 0, 10)
    watermark.BackgroundColor3 = Color3.fromRGB(15, 17, 25)
    watermark.BackgroundTransparency = 0.2
    watermark.BorderSizePixel = 0
    watermark.Parent = screenGui
    
    local waterCorner = Instance.new("UICorner")
    waterCorner.CornerRadius = UDim.new(0, 6)
    waterCorner.Parent = watermark
    
    local waterStroke = Instance.new("UIStroke")
    waterStroke.Color = Color3.fromRGB(0, 162, 255)
    waterStroke.Thickness = 1
    waterStroke.Transparency = 0.5
    waterStroke.Parent = watermark
    
    local waterLabel = Instance.new("TextLabel")
    waterLabel.Size = UDim2.new(1, -10, 1, 0)
    waterLabel.Position = UDim2.new(0, 5, 0, 0)
    waterLabel.Text = "SNAKE HUB | " .. LocalPlayer.Name
    waterLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    waterLabel.BackgroundTransparency = 1
    waterLabel.Font = Enum.Font.GothamBold
    waterLabel.TextSize = 12
    waterLabel.TextXAlignment = Enum.TextXAlignment.Left
    waterLabel.Parent = watermark
    
    local fpsLabel = Instance.new("TextLabel")
    fpsLabel.Size = UDim2.new(0, 50, 1, 0)
    fpsLabel.Position = UDim2.new(1, -55, 0, 0)
    fpsLabel.Text = "60 FPS"
    fpsLabel.TextColor3 = Color3.fromRGB(0, 255, 100)
    fpsLabel.BackgroundTransparency = 1
    fpsLabel.Font = Enum.Font.Gotham
    fpsLabel.TextSize = 10
    fpsLabel.Parent = watermark
    
    RunService.RenderStepped:Connect(function()
        local fps = math.floor(1 / RunService.RenderStepped:Wait())
        fpsLabel.Text = fps .. " FPS"
        if fps >= 60 then
            fpsLabel.TextColor3 = Color3.fromRGB(0, 255, 100)
        elseif fps >= 30 then
            fpsLabel.TextColor3 = Color3.fromRGB(255, 255, 0)
        else
            fpsLabel.TextColor3 = Color3.fromRGB(255, 50, 50)
        end
    end)
end

local function antiCheatDetection()
    spawn(function()
        while hubLoaded do
            task.wait(30)
            local character = LocalPlayer.Character
            if character then
                local humanoid = character:FindFirstChild("Humanoid")
                if humanoid then
                    if humanoid.WalkSpeed > 200 then
                        humanoid.WalkSpeed = settings.WalkSpeed
                    end
                    if humanoid.JumpPower > 500 then
                        humanoid.JumpPower = settings.JumpPower
                    end
                    if humanoid.Health > humanoid.MaxHealth * 2 then
                        humanoid.Health = humanoid.MaxHealth
                    end
                end
            end
        end
    end)
end

local function autoReconnect()
    local lastPlace = game.PlaceId
    Players.PlayerRemoving:Connect(function()
        if hubLoaded then
            saveSettings()
        end
    end)
    game:BindToClose(function()
        saveSettings()
    end)
end

local function setupAll()
    loadSettings()
    createPlayerList()
    createWatermark()
    startAutoRob()
    startAutoArrest()
    startAutoCollectLoot()
    startAutoSellItems()
    startTeleportToPlayer()
    antiCheatDetection()
    autoReconnect()
    createNotification("SNAKE HUB v2.0 CARREGADO!", 3)
    createNotification("Pressione o botao S para abrir o menu", 4)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if not hubLoaded then return end
    if input.KeyCode == Enum.KeyCode.F3 then
        saveSettings()
        createNotification("Configuracoes salvas!", 2)
    end
    if input.KeyCode == Enum.KeyCode.F4 then
        loadSettings()
        createNotification("Configuracoes carregadas!", 2)
    end
    if input.KeyCode == Enum.KeyCode.Delete then
        hubLoaded = false
        disableFly()
        disableNoclip()
        for _, connection in ipairs(farmConnections) do
            connection:Disconnect()
        end
        for _, connection in ipairs(killAuraConnections) do
            connection:Disconnect()
        end
        for _, connection in ipairs(spinConnections) do
            connection:Disconnect()
        end
        for _, connection in ipairs(noclipConnections) do
            connection:Disconnect()
        end
        if godModeConnection then godModeConnection:Disconnect() end
        if infiniteJumpConnection then infiniteJumpConnection:Disconnect() end
        if rainbowConnection then rainbowConnection:Disconnect() end
        if chatSpyConnection then chatSpyConnection:Disconnect() end
        if triggerBotConnection then triggerBotConnection:Disconnect() end
        if rapidFireConnection then rapidFireConnection:Disconnect() end
        if noRecoilConnection then noRecoilConnection:Disconnect() end
        if infiniteAmmoConnection then infiniteAmmoConnection:Disconnect() end
        if antiAfkConnection then antiAfkConnection:Disconnect() end
        if autoRespawnConnection then autoRespawnConnection:Disconnect() end
        if thirdPersonConnection then thirdPersonConnection:Disconnect() end
        clearESP()
        local screenGui = LocalPlayer.PlayerGui:FindFirstChild("SnakeHubUI")
        if screenGui then screenGui:Destroy() end
        createNotification("Hub descarregado!", 2)
    end
end)

function loadHub()
    createUI()
    updateFOVCircle()
    setupCrosshair()
    hubLoaded = true
    setupWalkSpeed()
    setupFOV()
    setupThirdPerson()
    setupAntiAfk()
    setupAutoRespawn()
    setupAll()

    RunService.RenderStepped:Connect(function()
        if not hubLoaded then return end
        updateCameraAimbot()
        updateESP()
        updateFly()
        local newWeapon = getCurrentWeapon()
        if newWeapon and (not currentWeapon or currentWeapon.Tool ~= newWeapon.Tool) then
            currentWeapon = newWeapon
        elseif not newWeapon and currentWeapon then
            currentWeapon = nil
        end
        if isAiming and currentWeapon then
            updateWeaponAim()
        end
        if settings.NoclipEnabled then
            local character = LocalPlayer.Character
            if character then
                local humanoid = character:FindFirstChild("Humanoid")
                if humanoid then
                    humanoid.WalkSpeed = settings.NoClipSpeed
                end
            end
        end
        if settings.SpinBot and settings.AimbotEnabled and aimbotHeld then
            local character = LocalPlayer.Character
            if character then
                local rootPart = character:FindFirstChild("HumanoidRootPart")
                if rootPart then
                    rootPart.CFrame = rootPart.CFrame * CFrame.Angles(0, math.rad(settings.SpinSpeed * 0.2), 0)
                end
            end
        end
    end)

    Players.PlayerAdded:Connect(function()
        if hubLoaded then
            setupWalkSpeed()
            setupFOV()
            createNotification("Novo jogador entrou!", 1.5)
        end
    end)

    Players.PlayerRemoving:Connect(function(player)
        if espObjects[player] then
            pcall(function()
                espObjects[player].Billboard:Destroy()
                if espObjects[player].Highlight then espObjects[player].Highlight:Destroy() end
                if espObjects[player].Tracer then espObjects[player].Tracer:Remove() end
            end)
            espObjects[player] = nil
        end
    end)

    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(0.3)
        setupWalkSpeed()
        setupFOV()
        setupThirdPerson()
        if settings.InfiniteJump then setupInfiniteJump() end
        if settings.GodMode then setupGodMode() end
        if settings.FlyEnabled then setupFly() end
        if settings.NoclipEnabled then setupNoclip() end
    end)

    LocalPlayer.Idled:Connect(function()
        if settings.AntiAfk then
            local virtualInput = game:GetService("VirtualUser")
            virtualInput:CaptureController()
            virtualInput:ClickButton2(Vector2.new())
        end
    end)
end

createKeyScreen()
