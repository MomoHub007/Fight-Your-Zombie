local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
local SaveManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua"))()
local InterfaceManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua"))()

local Window = Fluent:CreateWindow({
    Title = "Gemini Ultimate v4.1",
    SubTitle = "English Interface - Minimalist",
    TabWidth = 160,
    Size = UDim2.fromOffset(550, 420),
    Acrylic = true, 
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.LeftControl
})

local Tabs = {
    Main = Window:AddTab({ Title = "Dashboard", Icon = "layout" }),
    Settings = Window:AddTab({ Title = "Settings", Icon = "settings" })
}

local Options = Fluent.Options
local LP = game.Players.LocalPlayer
local SavedSpawnPosition = nil
local IsSettingUp = false
local IsWalkingFarmActive = false
local IsLuckyBlockActive = false

-- // [Helper Functions] //
local function getLuckyBlock()
    local debris = workspace:FindFirstChild("Debris")
    local normal = debris and debris:FindFirstChild("Normal")
    local main = normal and normal:FindFirstChild("Main")
    if main and main:IsA("BasePart") then
        local prompt = main:FindFirstChildWhichIsA("ProximityPrompt", true)
        return main, prompt
    end
    return nil, nil
end

local function getAxeList()
    local list = {}
    local seenNames = {}
    local axesFolder = LP:FindFirstChild("Data") and LP.Data:FindFirstChild("Axes")
    if axesFolder then
        for _, axeFolder in pairs(axesFolder:GetChildren()) do
            local axeNameValue = axeFolder:FindFirstChild("AxeName")
            if axeNameValue then 
                local name = axeNameValue.Value
                if not seenNames[name] then
                    table.insert(list, name .. " (" .. axeFolder.Name .. ")")
                    seenNames[name] = true
                end
            end
        end
    end
    return #list > 0 and list or {"No Axe Found"}
end

local function getClosestWalkingZombie()
    local closest = nil
    local shortestDist = math.huge
    local char = LP.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return nil end
    local zombiesFolder = workspace:FindFirstChild("WalkingZombies")
    if zombiesFolder then
        for _, zombieObj in pairs(zombiesFolder:GetChildren()) do
            local root = zombieObj:FindFirstChild("RootPart")
            if root then
                local dist = (char.HumanoidRootPart.Position - root.Position).Magnitude
                if dist < shortestDist then shortestDist = dist; closest = zombieObj end
            end
        end
    end
    return closest
end

local function getClosestTreeSpawnGlobally()
    local closestPos = nil
    local shortestDistance = math.huge
    local myPos = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") and LP.Character.HumanoidRootPart.Position
    if not myPos then return nil end
    for _, plot in pairs(workspace.Plots:GetChildren()) do
        pcall(function()
            local contents = plot:FindFirstChild("PlotContents")
            local treeSpawn = contents and contents:FindFirstChild("TreeSpawn")
            if treeSpawn then
                local pos = (treeSpawn:IsA("BasePart") and treeSpawn.Position) or (treeSpawn:IsA("Model") and treeSpawn:GetModelCFrame().Position)
                if pos then
                    local d = (myPos - pos).Magnitude
                    if d < shortestDistance then shortestDistance = d; closestPos = pos end
                end
            end
        end)
    end
    return closestPos
end

-- // [DASHBOARD TAB - ENGLISH UI] //
do
    local EqSection = Tabs.Main:AddSection("Equipment & Combat")
    
    local AxeDropdown = EqSection:AddDropdown("SelectedAxe", {
        Title = "Weapon Selector",
        Values = getAxeList(),
        Default = 1
    })

    EqSection:AddButton({
        Title = "Refresh Inventory",
        Callback = function() 
            AxeDropdown:SetValues(getAxeList())
            AxeDropdown:SetValue(nil)
            Fluent:Notify({Title = "System", Content = "Weapon list has been updated.", Duration = 2})
        end
    })

    EqSection:AddToggle("KillAura", {Title = "Auto Kill Aura", Default = false })
    EqSection:AddToggle("AutoCollectCoin", {Title = "Auto Collect Coins", Default = false })

    local FarmSection = Tabs.Main:AddSection("Farming System (Priority)")

    local LuckyToggle = FarmSection:AddToggle("AutoLuckyBlock", {Title = "[P1] Lucky Block Farm", Default = false })
    local WalkingToggle = FarmSection:AddToggle("AutoFarmWalking", {Title = "[P2] Walking Zombie Farm", Default = false })
    local PlotToggle = FarmSection:AddToggle("FarmZombie", {Title = "[P3] Plot Zombie Farm", Default = false })

    PlotToggle:OnChanged(function(Value)
        if Value and SavedSpawnPosition == nil then
            IsSettingUp = true
            Fluent:Notify({
                Title = "System Setup", 
                Content = "Recording spawn position... Please wait.", 
                Duration = 5
            })
            if LP.Character and LP.Character:FindFirstChild("Humanoid") then LP.Character.Humanoid.Health = 0 end
            LP.CharacterAdded:Wait(); task.wait(1.5)
            local root = LP.Character:WaitForChild("HumanoidRootPart", 10)
            if root then SavedSpawnPosition = root.Position end
            IsSettingUp = false
            Fluent:Notify({
                Title = "Success", 
                Content = "Spawn position saved. Farming started.", 
                Duration = 3
            })
        end
    end)
end

-- // [CORE LOOPS] //

-- Loop 1: Lucky Block (Priority 1)
task.spawn(function()
    while true do
        if not IsSettingUp and Options.AutoLuckyBlock.Value then
            local block, prompt = getLuckyBlock()
            local charRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
            if block and charRoot then
                IsLuckyBlockActive = true
                charRoot.CFrame = block.CFrame * CFrame.new(0, 0, 3)
                repeat
                    task.wait(0.1)
                    if prompt then fireproximityprompt(prompt) end
                    if charRoot and block.Parent then charRoot.CFrame = block.CFrame * CFrame.new(0, 0, 3) end
                until not block.Parent or not Options.AutoLuckyBlock.Value
                if SavedSpawnPosition and charRoot then charRoot.CFrame = CFrame.new(SavedSpawnPosition + Vector3.new(0, 3, 0)); task.wait(0.3) end
                IsLuckyBlockActive = false
            else IsLuckyBlockActive = false end
        end
        task.wait(0.1)
        if Fluent.Unloaded then break end
    end
end)

-- Loop 2: Walking Zombie (Priority 2)
task.spawn(function()
    while true do
        if not IsSettingUp and Options.AutoFarmWalking.Value and not IsLuckyBlockActive then
            local zombie = getClosestWalkingZombie()
            local charRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
            if zombie and zombie:FindFirstChild("RootPart") and charRoot then
                IsWalkingFarmActive = true
                charRoot.CFrame = zombie.RootPart.CFrame * CFrame.new(0, 0, 3)
            else
                if IsWalkingFarmActive and SavedSpawnPosition and charRoot then
                    charRoot.CFrame = CFrame.new(SavedSpawnPosition + Vector3.new(0, 3, 0))
                    task.wait(0.3); IsWalkingFarmActive = false
                end
            end
        end
        task.wait(0.01)
        if Fluent.Unloaded then break end
    end
end)

-- Loop 3: Plot Farm (Priority 3)
task.spawn(function()
    while true do
        if not IsSettingUp and Options.FarmZombie.Value then
            if not IsLuckyBlockActive and not IsWalkingFarmActive then
                local root = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
                local target = getClosestTreeSpawnGlobally()
                if root and target then root.CFrame = CFrame.new(target + Vector3.new(0, 3, 0)) end
            end
        end
        task.wait(0.1)
        if Fluent.Unloaded then break end
    end
end)

-- Loop 4: Kill Aura & Combat
task.spawn(function()
    while true do
        if not IsSettingUp and not Fluent.Unloaded then
            if Options.KillAura.Value then
                pcall(function()
                    local axeString = Options.SelectedAxe.Value
                    local axeIndex = axeString:match("%((%d+)%)")
                    if axeIndex then
                        game:GetService("ReplicatedStorage").Remotes.EquipAxe:FireServer(axeIndex)
                        game:GetService("ReplicatedStorage").Remotes.AxeSwing:FireServer()
                    end
                end)
            end
            if Options.AutoCollectCoin.Value then
                pcall(function()
                    for _, c in pairs(workspace.Orbs:GetChildren()) do
                        if c.Name == "Coin" and c:FindFirstChild("HitBox") then
                            c.HitBox.Size = Vector3.new(50,50,50); c.HitBox.CanCollide = false
                        end
                    end
                end)
            end
        end
        task.wait(0.1)
        if Fluent.Unloaded then break end
    end
end)

-- // [SETTINGS CONFIG] //
SaveManager:SetLibrary(Fluent)
InterfaceManager:SetLibrary(Fluent)
InterfaceManager:SetFolder("GeminiAxeConfig")
SaveManager:SetFolder("GeminiAxeConfig/Main")
InterfaceManager:BuildInterfaceSection(Tabs.Settings)
SaveManager:BuildConfigSection(Tabs.Settings)
Window:SelectTab(1)
SaveManager:LoadAutoloadConfig()
