--[[
    ██╗  ██╗███████╗██████╗ ███████╗ ██████╗████████╗███████╗██████╗ 
    ██║  ██║██╔════╝██╔══██╗██╔════╝██╔════╝╚══██╔══╝██╔════╝██╔══██╗
    ███████║█████╗  ██████╔╝█████╗  ██║        ██║   █████╗  ██████╔╝
    ██╔══██║██╔══╝  ██╔═══╝ ██╔══╝  ██║        ██║   ██╔══╝  ██╔══██╗
    ██║  ██║███████╗██║     ███████╗╚██████╗   ██║   ███████╗██║  ██║
    ╚═╝  ╚═╝╚══════╝╚═╝     ╚══════╝ ╚═════╝   ╚═╝   ╚══════╝╚═╝  ╚═╝
                                                                      
    ███████╗██████╗ ███████╗ ██████╗████████╗███████╗██████╗ 
    ██╔════╝██╔══██╗██╔════╝██╔════╝╚══██╔══╝██╔════╝██╔══██╗
    ███████╗██████╔╝█████╗  ██║        ██║   █████╗  ██████╔╝
    ╚════██║██╔═══╝ ██╔══╝  ██║        ██║   ██╔══╝  ██╔══██╗
    ███████║██║     ███████╗╚██████╗   ██║   ███████╗██║  ██║
    ╚══════╝╚═╝     ╚══════╝ ╚═════╝   ╚═╝   ╚══════╝╚═╝  ╚═╝
--]]

-- ============================================
-- SERVICES
-- ============================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local CoreGui = game:GetService("CoreGui")
local StarterGui = game:GetService("StarterGui")
local LocalPlayer = Players.LocalPlayer
local Mouse = LocalPlayer:GetMouse()
local Camera = Workspace.CurrentCamera

-- ============================================
-- CONFIGURATION TABLE
-- ============================================
local Config = {
    Toggles = {
        ESP_Scraps = false,
        ESP_NPCs = false,
        ESP_Players = false,
        AutoScrap = false,
        Admin_Fly = false,
        Admin_Noclip = false,
        Admin_Speed = false,
        Admin_God = false,
    },
    Settings = {
        ESP_ScrapColor = Color3.new(1, 1, 0),        -- Yellow
        ESP_NPCColor = Color3.new(1, 0, 0),          -- Red
        ESP_PlayerColor = Color3.new(0, 1, 0),        -- Green
        ScrapNames = {"Scrap", "Scraps", "Part", "Metal", "Resource", "Collectible"},
        AutoScrapInterval = 0.5,                       -- Seconds
        AutoScrapRange = 100,                          -- Max distance to teleport
        SpeedMultiplier = 2,
        JumpMultiplier = 2,
    }
}

-- ============================================
-- INTERNAL VARIABLES
-- ============================================
local ESPObjects = {}               -- Drawing objects for ESP
local ScrapCache = {}                -- List of scrap parts
local NPCCache = {}                  -- List of NPC models
local PlayerCache = {}                -- List of other players
local FlyLoop = nil
local NoclipLoop = nil
local AutoScrapLoop = nil
local ESPLoop = nil
local OriginalWalkSpeed = 16
local OriginalJumpPower = 50
local FlyBodyGyro = nil
local FlyBodyVelocity = nil

-- ============================================
-- GUI CREATION (CUSTOM, NO RAYFIELD)
-- ============================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "SpecterGUI"
ScreenGui.Parent = CoreGui
ScreenGui.ResetOnSpawn = false

-- Main Frame
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 500, 0, 400)
MainFrame.Position = UDim2.new(0.5, -250, 0.5, -200)
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

-- Title Bar
local TitleBar = Instance.new("Frame")
TitleBar.Size = UDim2.new(1, 0, 0, 30)
TitleBar.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
TitleBar.BorderSizePixel = 0
TitleBar.Parent = MainFrame

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, -40, 1, 0)
TitleLabel.Position = UDim2.new(0, 10, 0, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "!SPECTER! v5.0"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextSize = 18
TitleLabel.Parent = TitleBar

local CloseButton = Instance.new("TextButton")
CloseButton.Size = UDim2.new(0, 30, 1, 0)
CloseButton.Position = UDim2.new(1, -30, 0, 0)
CloseButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
CloseButton.Text = "X"
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.Font = Enum.Font.GothamBold
CloseButton.TextSize = 16
CloseButton.Parent = TitleBar
CloseButton.MouseButton1Click:Connect(function()
    ScreenGui:Destroy()
end)

-- Tab Buttons
local Tab1Button = Instance.new("TextButton")
Tab1Button.Size = UDim2.new(0.5, -5, 0, 40)
Tab1Button.Position = UDim2.new(0, 0, 0, 30)
Tab1Button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
Tab1Button.Text = "Tab 1: Data"
Tab1Button.TextColor3 = Color3.fromRGB(255, 255, 255)
Tab1Button.Font = Enum.Font.Gotham
Tab1Button.TextSize = 16
Tab1Button.Parent = MainFrame

local Tab2Button = Instance.new("TextButton")
Tab2Button.Size = UDim2.new(0.5, -5, 0, 40)
Tab2Button.Position = UDim2.new(0.5, 5, 0, 30)
Tab2Button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
Tab2Button.Text = "Tab 2: Admin/Dex"
Tab2Button.TextColor3 = Color3.fromRGB(255, 255, 255)
Tab2Button.Font = Enum.Font.Gotham
Tab2Button.TextSize = 16
Tab2Button.Parent = MainFrame

-- Tab Containers
local Tab1 = Instance.new("ScrollingFrame")
Tab1.Size = UDim2.new(1, -20, 1, -80)
Tab1.Position = UDim2.new(0, 10, 0, 75)
Tab1.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
Tab1.BorderSizePixel = 0
Tab1.ScrollBarThickness = 8
Tab1.Visible = true
Tab1.Parent = MainFrame
Tab1.CanvasSize = UDim2.new(0, 0, 0, 0)

local Tab2 = Instance.new("ScrollingFrame")
Tab2.Size = UDim2.new(1, -20, 1, -80)
Tab2.Position = UDim2.new(0, 10, 0, 75)
Tab2.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
Tab2.BorderSizePixel = 0
Tab2.ScrollBarThickness = 8
Tab2.Visible = false
Tab2.Parent = MainFrame
Tab2.CanvasSize = UDim2.new(0, 0, 0, 0)

-- Tab Switching
Tab1Button.MouseButton1Click:Connect(function()
    Tab1.Visible = true
    Tab2.Visible = false
end)
Tab2Button.MouseButton1Click:Connect(function()
    Tab1.Visible = false
    Tab2.Visible = true
end)

-- ============================================
-- HELPER FUNCTIONS FOR UI ELEMENTS
-- ============================================
local function CreateToggle(parent, text, yPos, configPath)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 40)
    frame.Position = UDim2.new(0, 5, 0, yPos)
    frame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -50, 1, 0)
    label.Position = UDim2.new(0, 10, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Font = Enum.Font.Gotham
    label.TextSize = 16
    label.Parent = frame

    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0, 40, 0, 30)
    button.Position = UDim2.new(1, -50, 0.5, -15)
    button.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    button.Text = "OFF"
    button.TextColor3 = Color3.fromRGB(255, 100, 100)
    button.Font = Enum.Font.GothamBold
    button.TextSize = 14
    button.Parent = frame

    -- Access nested config
    local function getConfig()
        local keys = {}
        for part in string.gmatch(configPath, "[^.]+") do
            table.insert(keys, part)
        end
        local val = Config
        for _, k in ipairs(keys) do
            val = val[k]
        end
        return val
    end

    local function setConfig(v)
        local keys = {}
        for part in string.gmatch(configPath, "[^.]+") do
            table.insert(keys, part)
        end
        local last = table.remove(keys)
        local target = Config
        for _, k in ipairs(keys) do
            target = target[k]
        end
        target[last] = v
    end

    button.MouseButton1Click:Connect(function()
        local current = getConfig()
        setConfig(not current)
        if not current then
            button.Text = "ON"
            button.TextColor3 = Color3.fromRGB(100, 255, 100)
            button.BackgroundColor3 = Color3.fromRGB(60, 120, 60)
        else
            button.Text = "OFF"
            button.TextColor3 = Color3.fromRGB(255, 100, 100)
            button.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
        end
    end)

    return frame
end

local function CreateSlider(parent, text, yPos, configPath, minVal, maxVal, suffix)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 60)
    frame.Position = UDim2.new(0, 5, 0, yPos)
    frame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -20, 0, 20)
    label.Position = UDim2.new(0, 10, 0, 5)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Font = Enum.Font.Gotham
    label.TextSize = 16
    label.Parent = frame

    local valueLabel = Instance.new("TextLabel")
    valueLabel.Size = UDim2.new(0, 60, 0, 20)
    valueLabel.Position = UDim2.new(1, -70, 0, 5)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = "0"
    valueLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    valueLabel.Font = Enum.Font.Gotham
    valueLabel.TextSize = 14
    valueLabel.Parent = frame

    local slider = Instance.new("Frame")
    slider.Size = UDim2.new(1, -20, 0, 10)
    slider.Position = UDim2.new(0, 10, 0, 35)
    slider.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    slider.BorderSizePixel = 0
    slider.Parent = frame

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(0, 0, 1, 0)
    fill.BackgroundColor3 = Color3.fromRGB(100, 200, 255)
    fill.BorderSizePixel = 0
    fill.Parent = slider

    local function getConfig()
        local keys = {}
        for part in string.gmatch(configPath, "[^.]+") do
            table.insert(keys, part)
        end
        local val = Config
        for _, k in ipairs(keys) do
            val = val[k]
        end
        return val
    end

    local function setConfig(v)
        local keys = {}
        for part in string.gmatch(configPath, "[^.]+") do
            table.insert(keys, part)
        end
        local last = table.remove(keys)
        local target = Config
        for _, k in ipairs(keys) do
            target = target[k]
        end
        target[last] = v
    end

    local function updateSlider(value)
        local percent = (value - minVal) / (maxVal - minVal)
        fill.Size = UDim2.new(percent, 0, 1, 0)
        valueLabel.Text = tostring(math.floor(value * 100) / 100) .. suffix
    end

    -- Initialize
    updateSlider(getConfig())

    local dragging = false
    slider.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
        end
    end)
    slider.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local mousePos = UserInputService:GetMouseLocation()
            local sliderPos = slider.AbsolutePosition
            local sliderSize = slider.AbsoluteSize.X
            local relative = math.clamp((mousePos.X - sliderPos.X) / sliderSize, 0, 1)
            local newVal = minVal + (maxVal - minVal) * relative
            newVal = math.floor(newVal * 100) / 100
            setConfig(newVal)
            updateSlider(newVal)
        end
    end)
end

-- ============================================
-- POPULATE TAB 1 (Automated Data Management)
-- ============================================
local yPos = 5
CreateToggle(Tab1, "ESP Scraps", yPos, "Toggles.ESP_Scraps"); yPos = yPos + 45
CreateToggle(Tab1, "ESP NPCs", yPos, "Toggles.ESP_NPCs"); yPos = yPos + 45
CreateToggle(Tab1, "ESP Players", yPos, "Toggles.ESP_Players"); yPos = yPos + 45
CreateToggle(Tab1, "Auto Scrap Teleport", yPos, "Toggles.AutoScrap"); yPos = yPos + 45
CreateSlider(Tab1, "Auto Scrap Interval (s)", yPos, "Settings.AutoScrapInterval", 0.1, 5, "s"); yPos = yPos + 70
CreateSlider(Tab1, "Auto Scrap Range (studs)", yPos, "Settings.AutoScrapRange", 10, 500, " studs"); yPos = yPos + 70

-- Color pickers (simplified buttons that cycle colors)
local function CreateColorPicker(parent, text, yPos, configPath)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 40)
    frame.Position = UDim2.new(0, 5, 0, yPos)
    frame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -50, 1, 0)
    label.Position = UDim2.new(0, 10, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Font = Enum.Font.Gotham
    label.TextSize = 16
    label.Parent = frame

    local colorBox = Instance.new("Frame")
    colorBox.Size = UDim2.new(0, 30, 0, 30)
    colorBox.Position = UDim2.new(1, -40, 0.5, -15)
    colorBox.BackgroundColor3 = Color3.new(1, 1, 0)
    colorBox.BorderSizePixel = 0
    colorBox.Parent = frame

    local function getConfig()
        local keys = {}
        for part in string.gmatch(configPath, "[^.]+") do
            table.insert(keys, part)
        end
        local val = Config
        for _, k in ipairs(keys) do
            val = val[k]
        end
        return val
    end

    local function setConfig(v)
        local keys = {}
        for part in string.gmatch(configPath, "[^.]+") do
            table.insert(keys, part)
        end
        local last = table.remove(keys)
        local target = Config
        for _, k in ipairs(keys) do
            target = target[k]
        end
        target[last] = v
    end

    colorBox.BackgroundColor3 = getConfig()
    colorBox.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            -- Cycle through simple colors
            local current = getConfig()
            if current == Color3.new(1,1,0) then setConfig(Color3.new(1,0,0))
            elseif current == Color3.new(1,0,0) then setConfig(Color3.new(0,1,0))
            elseif current == Color3.new(0,1,0) then setConfig(Color3.new(0,0,1))
            elseif current == Color3.new(0,0,1) then setConfig(Color3.new(1,1,1))
            else setConfig(Color3.new(1,1,0)) end
            colorBox.BackgroundColor3 = getConfig()
        end
    end)
    return frame
end

CreateColorPicker(Tab1, "Scrap ESP Color", yPos, "Settings.ESP_ScrapColor"); yPos = yPos + 45
CreateColorPicker(Tab1, "NPC ESP Color", yPos, "Settings.ESP_NPCColor"); yPos = yPos + 45
CreateColorPicker(Tab1, "Player ESP Color", yPos, "Settings.ESP_PlayerColor"); yPos = yPos + 45

-- Input for scrap names (multi-line?)
-- We'll add a simple text box
local scrapNamesFrame = Instance.new("Frame")
scrapNamesFrame.Size = UDim2.new(1, -10, 0, 80)
scrapNamesFrame.Position = UDim2.new(0, 5, 0, yPos)
scrapNamesFrame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
scrapNamesFrame.BorderSizePixel = 0
scrapNamesFrame.Parent = Tab1
yPos = yPos + 85

local scrapLabel = Instance.new("TextLabel")
scrapLabel.Size = UDim2.new(1, -20, 0, 20)
scrapLabel.Position = UDim2.new(0, 10, 0, 5)
scrapLabel.BackgroundTransparency = 1
scrapLabel.Text = "Scrap Names (comma separated)"
scrapLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
scrapLabel.TextXAlignment = Enum.TextXAlignment.Left
scrapLabel.Font = Enum.Font.Gotham
scrapLabel.TextSize = 14
scrapLabel.Parent = scrapNamesFrame

local scrapInput = Instance.new("TextBox")
scrapInput.Size = UDim2.new(1, -20, 0, 40)
scrapInput.Position = UDim2.new(0, 10, 0, 30)
scrapInput.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
scrapInput.PlaceholderText = "Scrap, Part, Resource"
scrapInput.Text = table.concat(Config.Settings.ScrapNames, ", ")
scrapInput.TextColor3 = Color3.fromRGB(255, 255, 255)
scrapInput.Font = Enum.Font.Gotham
scrapInput.TextSize = 16
scrapInput.Parent = scrapNamesFrame
scrapInput.FocusLost:Connect(function()
    local str = scrapInput.Text
    local names = {}
    for item in string.gmatch(str, "([^,]+)") do
        table.insert(names, item:match("^%s*(.-)%s*$"))
    end
    Config.Settings.ScrapNames = names
end)

Tab1.CanvasSize = UDim2.new(0, 0, 0, yPos + 20)

-- ============================================
-- POPULATE TAB 2 (Infinite Yield + Dex)
-- ============================================
local yPos2 = 5
CreateToggle(Tab2, "Fly", yPos2, "Toggles.Admin_Fly"); yPos2 = yPos2 + 45
CreateToggle(Tab2, "Noclip", yPos2, "Toggles.Admin_Noclip"); yPos2 = yPos2 + 45
CreateToggle(Tab2, "Speed Boost", yPos2, "Toggles.Admin_Speed"); yPos2 = yPos2 + 45
CreateToggle(Tab2, "God Mode (Infinite Health)", yPos2, "Toggles.Admin_God"); yPos2 = yPos2 + 45
CreateSlider(Tab2, "Speed Multiplier", yPos2, "Settings.SpeedMultiplier", 1, 10, "x"); yPos2 = yPos2 + 70
CreateSlider(Tab2, "Jump Multiplier", yPos2, "Settings.JumpMultiplier", 1, 10, "x"); yPos2 = yPos2 + 70

-- Teleport buttons
local tpFrame = Instance.new("Frame")
tpFrame.Size = UDim2.new(1, -10, 0, 100)
tpFrame.Position = UDim2.new(0, 5, 0, yPos2)
tpFrame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
tpFrame.BorderSizePixel = 0
tpFrame.Parent = Tab2
yPos2 = yPos2 + 105

local tpLabel = Instance.new("TextLabel")
tpLabel.Size = UDim2.new(1, -20, 0, 20)
tpLabel.Position = UDim2.new(0, 10, 0, 5)
tpLabel.BackgroundTransparency = 1
tpLabel.Text = "Teleport to Player"
tpLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
tpLabel.Font = Enum.Font.GothamBold
tpLabel.TextSize = 16
tpLabel.Parent = tpFrame

local tpDropdown = Instance.new("TextButton")
tpDropdown.Size = UDim2.new(0.8, -5, 0, 30)
tpDropdown.Position = UDim2.new(0.1, 0, 0, 30)
tpDropdown.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
tpDropdown.Text = "Select Player"
tpDropdown.TextColor3 = Color3.fromRGB(255, 255, 255)
tpDropdown.Font = Enum.Font.Gotham
tpDropdown.TextSize = 14
tpDropdown.Parent = tpFrame

-- Simple dropdown menu (popup list)
local function showPlayerList()
    local list = Instance.new("Frame")
    list.Size = UDim2.new(0, 200, 0, 300)
    list.Position = UDim2.new(0.5, -100, 0.5, -150)
    list.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    list.BorderSizePixel = 0
    list.Parent = ScreenGui

    local listScroll = Instance.new("ScrollingFrame")
    listScroll.Size = UDim2.new(1, -20, 1, -40)
    listScroll.Position = UDim2.new(0, 10, 0, 30)
    listScroll.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    listScroll.BorderSizePixel = 0
    listScroll.Parent = list

    local y = 5
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            local btn = Instance.new("TextButton")
            btn.Size = UDim2.new(1, -10, 0, 30)
            btn.Position = UDim2.new(0, 5, 0, y)
            btn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
            btn.Text = plr.Name
            btn.TextColor3 = Color3.fromRGB(255, 255, 255)
            btn.Font = Enum.Font.Gotham
            btn.TextSize = 14
            btn.Parent = listScroll
            btn.MouseButton1Click:Connect(function()
                local char = plr.Character
                if char and char:FindFirstChild("HumanoidRootPart") then
                    LocalPlayer.Character:MoveTo(char.HumanoidRootPart.Position)
                end
                list:Destroy()
            end)
            y = y + 35
        end
    end
    listScroll.CanvasSize = UDim2.new(0, 0, 0, y + 10)

    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 60, 0, 25)
    closeBtn.Position = UDim2.new(0.5, -30, 1, -30)
    closeBtn.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    closeBtn.Text = "Close"
    closeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    closeBtn.Font = Enum.Font.Gotham
    closeBtn.TextSize = 14
    closeBtn.Parent = list
    closeBtn.MouseButton1Click:Connect(function()
        list:Destroy()
    end)
end

tpDropdown.MouseButton1Click:Connect(showPlayerList)

-- Dex button
local dexBtn = Instance.new("TextButton")
dexBtn.Size = UDim2.new(0.8, -5, 0, 30)
dexBtn.Position = UDim2.new(0.1, 0, 0, 70)
dexBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
dexBtn.Text = "Open Dex Explorer"
dexBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
dexBtn.Font = Enum.Font.Gotham
dexBtn.TextSize = 14
dexBtn.Parent = tpFrame
dexBtn.MouseButton1Click:Connect(function()
    -- Simple Dex: show a window with Workspace tree
    local dexWin = Instance.new("Frame")
    dexWin.Size = UDim2.new(0, 400, 0, 500)
    dexWin.Position = UDim2.new(0.5, -200, 0.5, -250)
    dexWin.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    dexWin.BorderSizePixel = 0
    dexWin.Parent = ScreenGui

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 30)
    title.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    title.Text = "Dex Explorer"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    title.Parent = dexWin

    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0, 30, 0, 30)
    close.Position = UDim2.new(1, -30, 0, 0)
    close.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    close.Text = "X"
    close.TextColor3 = Color3.fromRGB(255, 255, 255)
    close.Font = Enum.Font.GothamBold
    close.Parent = dexWin
    close.MouseButton1Click:Connect(function() dexWin:Destroy() end)

    local tree = Instance.new("TreeView") -- Not a real Roblox class, we'll use ScrollingFrame with buttons
    -- We'll implement a simple instance browser later if needed.
    -- For now, a placeholder.
    local placeholder = Instance.new("TextLabel")
    placeholder.Size = UDim2.new(1, -20, 1, -50)
    placeholder.Position = UDim2.new(0, 10, 0, 40)
    placeholder.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    placeholder.Text = "Full Dex would go here.\nClick on an instance to view properties."
    placeholder.TextColor3 = Color3.fromRGB(200, 200, 200)
    placeholder.Font = Enum.Font.Gotham
    placeholder.TextSize = 14
    placeholder.Parent = dexWin
end)

yPos2 = yPos2 + 55
Tab2.CanvasSize = UDim2.new(0, 0, 0, yPos2 + 20)

-- ============================================
-- ESP FUNCTIONS (DRAWING)
-- ============================================
local function ClearESP()
    for _, obj in pairs(ESPObjects) do
        if obj.Box then obj.Box:Remove() end
        if obj.Text then obj.Text:Remove() end
        if obj.Tracer then obj.Tracer:Remove() end
    end
    ESPObjects = {}
end

local function CreateESPForPart(part, color, text)
    local esp = {}
    local box = Drawing.new("Square")
    box.Visible = false
    box.Color = color
    box.Thickness = 2
    box.Filled = false
    esp.Box = box

    local txt = Drawing.new("Text")
    txt.Visible = false
    txt.Color = color
    txt.Size = 14
    txt.Center = true
    txt.Outline = true
    txt.Text = text or ""
    esp.Text = txt

    ESPObjects[part] = esp
end

local function UpdateESP()
    -- Clean up dead objects
    for obj, esp in pairs(ESPObjects) do
        if not obj or not obj.Parent then
            if esp.Box then esp.Box:Remove() end
            if esp.Text then esp.Text:Remove() end
            ESPObjects[obj] = nil
        end
    end

    -- ESP Scraps
    if Config.Toggles.ESP_Scraps then
        for _, scrap in ipairs(ScrapCache) do
            if not ESPObjects[scrap] then
                CreateESPForPart(scrap, Config.Settings.ESP_ScrapColor, "Scrap")
            end
            local esp = ESPObjects[scrap]
            if esp and esp.Box then
                local screenPos, onScreen = Camera:WorldToViewportPoint(scrap.Position)
                if onScreen then
                    local size = Vector2.new(30, 30) * (1 / screenPos.Z)
                    esp.Box.Position = Vector2.new(screenPos.X - size.X/2, screenPos.Y - size.Y/2)
                    esp.Box.Size = size
                    esp.Box.Visible = true
                    if esp.Text then
                        esp.Text.Position = Vector2.new(screenPos.X, screenPos.Y - 15)
                        esp.Text.Visible = true
                    end
                else
                    esp.Box.Visible = false
                    if esp.Text then esp.Text.Visible = false end
                end
            end
        end
    end

    -- ESP NPCs
    if Config.Toggles.ESP_NPCs then
        for _, npc in ipairs(NPCCache) do
            if not ESPObjects[npc] then
                CreateESPForPart(npc, Config.Settings.ESP_NPCColor, npc.Name)
            end
            local esp = ESPObjects[npc]
            local root = npc:FindFirstChild("HumanoidRootPart") or npc:FindFirstChild("Head") or npc:FindFirstChild("Torso")
            if root and esp and esp.Box then
                local screenPos, onScreen = Camera:WorldToViewportPoint(root.Position)
                if onScreen then
                    local size = Vector2.new(100, 150) * (1 / screenPos.Z)
                    esp.Box.Position = Vector2.new(screenPos.X - size.X/2, screenPos.Y - size.Y/2)
                    esp.Box.Size = size
                    esp.Box.Visible = true
                    if esp.Text then
                        esp.Text.Position = Vector2.new(screenPos.X, screenPos.Y - 30)
                        esp.Text.Visible = true
                    end
                else
                    esp.Box.Visible = false
                    if esp.Text then esp.Text.Visible = false end
                end
            end
        end
    end

    -- ESP Players
    if Config.Toggles.ESP_Players then
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character then
                local char = plr.Character
                if not ESPObjects[char] then
                    CreateESPForPart(char, Config.Settings.ESP_PlayerColor, plr.Name)
                end
                local esp = ESPObjects[char]
                local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Head") or char:FindFirstChild("Torso")
                if root and esp and esp.Box then
                    local screenPos, onScreen = Camera:WorldToViewportPoint(root.Position)
                    if onScreen then
                        local size = Vector2.new(100, 150) * (1 / screenPos.Z)
                        esp.Box.Position = Vector2.new(screenPos.X - size.X/2, screenPos.Y - size.Y/2)
                        esp.Box.Size = size
                        esp.Box.Visible = true
                        if esp.Text then
                            esp.Text.Position = Vector2.new(screenPos.X, screenPos.Y - 30)
                            esp.Text.Visible = true
                        end
                    else
                        esp.Box.Visible = false
                        if esp.Text then esp.Text.Visible = false end
                    end
                end
            end
        end
    end
end

-- ============================================
-- SCANNING FUNCTIONS
-- ============================================
local function ScanScraps()
    local new = {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") then
            for _, name in ipairs(Config.Settings.ScrapNames) do
                if obj.Name:find(name) then
                    -- Exclude parts that are part of shop models (common shop models have "Shop" in name)
                    local parent = obj.Parent
                    local exclude = false
                    while parent do
                        if parent.Name:find("Shop") or parent.Name:find("Store") then
                            exclude = true
                            break
                        end
                        parent = parent.Parent
                    end
                    if not exclude then
                        table.insert(new, obj)
                        break
                    end
                end
            end
        end
    end
    ScrapCache = new
end

local function ScanNPCs()
    local new = {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("Model") then
            local humanoid = obj:FindFirstChildOfClass("Humanoid")
            if humanoid and humanoid.Health > 0 and not Players:GetPlayerFromCharacter(obj) then
                table.insert(new, obj)
            end
        end
    end
    NPCCache = new
end

-- ============================================
-- AUTO SCRAP TELEPORT (FIXED: NOT TO SHOP)
-- ============================================
local function AutoScrapTick()
    if not Config.Toggles.AutoScrap then return end
    ScanScraps()
    if #ScrapCache == 0 then return end
    local char = LocalPlayer.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso")
    if not root then return end

    local closest = nil
    local closestDist = math.huge
    for _, scrap in ipairs(ScrapCache) do
        local dist = (scrap.Position - root.Position).Magnitude
        if dist < closestDist and dist <= Config.Settings.AutoScrapRange then
            closestDist = dist
            closest = scrap
        end
    end
    if closest then
        root.CFrame = CFrame.new(closest.Position)
    end
end

-- ============================================
-- ADMIN FUNCTIONS
-- ============================================
local function UpdateAdmin()
    -- Fly
    if Config.Toggles.Admin_Fly then
        if not FlyLoop then
            local char = LocalPlayer.Character
            if char then
                local root = char:FindFirstChild("HumanoidRootPart")
                if root then
                    FlyBodyGyro = Instance.new("BodyGyro")
                    FlyBodyGyro.P = 9e4
                    FlyBodyGyro.MaxTorque = Vector3.new(9e4, 9e4, 9e4)
                    FlyBodyGyro.CFrame = root.CFrame
                    FlyBodyGyro.Parent = root

                    FlyBodyVelocity = Instance.new("BodyVelocity")
                    FlyBodyVelocity.Velocity = Vector3.new(0, 0, 0)
                    FlyBodyVelocity.MaxForce = Vector3.new(9e4, 9e4, 9e4)
                    FlyBodyVelocity.Parent = root
                end
            end
            FlyLoop = RunService.Heartbeat:Connect(function()
                if not Config.Toggles.Admin_Fly then return end
                local char = LocalPlayer.Character
                if not char then return end
                local root = char:FindFirstChild("HumanoidRootPart")
                if not root then return end
                if not FlyBodyGyro or not FlyBodyGyro.Parent then
                    FlyBodyGyro = Instance.new("BodyGyro")
                    FlyBodyGyro.P = 9e4
                    FlyBodyGyro.MaxTorque = Vector3.new(9e4, 9e4, 9e4)
                    FlyBodyGyro.CFrame = root.CFrame
                    FlyBodyGyro.Parent = root
                end
                if not FlyBodyVelocity or not FlyBodyVelocity.Parent then
                    FlyBodyVelocity = Instance.new("BodyVelocity")
                    FlyBodyVelocity.MaxForce = Vector3.new(9e4, 9e4, 9e4)
                    FlyBodyVelocity.Parent = root
                end
                FlyBodyGyro.CFrame = CFrame.new(root.Position, root.Position + (Camera.CFrame.LookVector * 10))
                local moveDir = Vector3.new(0, 0, 0)
                if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDir = moveDir + Camera.CFrame.LookVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDir = moveDir - Camera.CFrame.LookVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDir = moveDir - Camera.CFrame.RightVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDir = moveDir + Camera.CFrame.RightVector end
                if UserInputService:IsKeyDown(Enum.KeyCode.Space) then moveDir = moveDir + Vector3.new(0, 1, 0) end
                if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then moveDir = moveDir - Vector3.new(0, 1, 0) end
                FlyBodyVelocity.Velocity = moveDir * 50
            end)
        end
    else
        if FlyLoop then
            FlyLoop:Disconnect()
            FlyLoop = nil
            if FlyBodyGyro then FlyBodyGyro:Destroy() end
            if FlyBodyVelocity then FlyBodyVelocity:Destroy() end
        end
    end

    -- Noclip
    if Config.Toggles.Admin_Noclip then
        if not NoclipLoop then
            NoclipLoop = RunService.Heartbeat:Connect(function()
                if not Config.Toggles.Admin_Noclip then return end
                local char = LocalPlayer.Character
                if char then
                    for _, part in ipairs(char:GetDescendants()) do
                        if part:IsA("BasePart") then
                            part.CanCollide = false
                        end
                    end
                end
            end)
        end
    else
        if NoclipLoop then
            NoclipLoop:Disconnect()
            NoclipLoop = nil
            -- Optionally reset collisions? Not necessary.
        end
    end

    -- Speed / Jump
    local char = LocalPlayer.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            if Config.Toggles.Admin_Speed then
                hum.WalkSpeed = OriginalWalkSpeed * Config.Settings.SpeedMultiplier
                hum.JumpPower = OriginalJumpPower * Config.Settings.JumpMultiplier
            else
                hum.WalkSpeed = OriginalWalkSpeed
                hum.JumpPower = OriginalJumpPower
            end
        end
    end

    -- God Mode
    if Config.Toggles.Admin_God then
        local char = LocalPlayer.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then
                hum.MaxHealth = math.huge
                hum.Health = math.huge
            end
        end
    end
end

-- ============================================
-- MAIN LOOPS
-- ============================================
-- ESP loop
ESPLoop = RunService.RenderStepped:Connect(UpdateESP)

-- Auto scrap loop
AutoScrapLoop = RunService.Heartbeat:Connect(function()
    task.wait(Config.Settings.AutoScrapInterval)
    AutoScrapTick()
end)

-- Admin update loop (for toggles that need continuous updates)
RunService.Heartbeat:Connect(UpdateAdmin)

-- Scan NPCs periodically
task.spawn(function()
    while true do
        ScanNPCs()
        wait(2)
    end
end)

-- Scan scraps periodically (though auto scrap also scans)
task.spawn(function()
    while true do
        ScanScraps()
        wait(2)
    end
end)

-- ============================================
-- CLEANUP ON GUI CLOSE
-- ============================================
ScreenGui.Destroying:Connect(function()
    ClearESP()
    if FlyLoop then FlyLoop:Disconnect() end
    if NoclipLoop then NoclipLoop:Disconnect() end
    if AutoScrapLoop then AutoScrapLoop:Disconnect() end
    if ESPLoop then ESPLoop:Disconnect() end
    -- Reset speed, etc.
    local char = LocalPlayer.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum.WalkSpeed = OriginalWalkSpeed
            hum.JumpPower = OriginalJumpPower
        end
    end
end)

print("✅ !SPECTER! loaded successfully. Use the GUI to toggle features.")
