-- =========================================================
--  全新UI - 离瑜二 
-- =========================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")

-- 等待玩家加载
local LocalPlayer = Players.LocalPlayer
if not LocalPlayer.Character then
    LocalPlayer.CharacterAdded:Wait()
end

-- 创建主GUI
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "离瑜二UI_" .. HttpService:GenerateGUID(false)
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = CoreGui

-- 功能状态变量
local ENABLED = false
local ESP_ENABLED = false
local NOCLIP_ENABLED = false
local BULLET_TRACKING_ENABLED = false
local isFlying = false
local flySpeed = 30
local teleportingAll = false
local teleportingSelected = false
local selectedPlayers = {}
local attackWhitelist = {}
local savedPosition = nil

-- 空心圆相关
local CAM = Workspace.CurrentCamera
local RADIUS = 60
local SEGMENTS = 360
local dots = {}
local colorIdx = 1

-- 7 色表
local COLORS = {
    Color3.fromRGB(255, 0, 0),
    Color3.fromRGB(255, 127, 0),
    Color3.fromRGB(255, 255, 0),
    Color3.fromRGB(0, 255, 0),
    Color3.fromRGB(0, 255, 255),
    Color3.fromRGB(0, 0, 255),
    Color3.fromRGB(127, 0, 255)
}

-- 创建空心圆
for i = 1, SEGMENTS do
    local d = Instance.new("Frame")
    d.Size = UDim2.new(0, 1, 0, 1)
    d.BorderSizePixel = 0
    d.Visible = false
    d.Parent = ScreenGui
    dots[i] = d
end

-- 0.5 秒换色
task.spawn(function()
    while true do
        if ENABLED then
            local col = COLORS[colorIdx]
            for _, d in ipairs(dots) do 
                d.BackgroundColor3 = col 
            end
            colorIdx = colorIdx % #COLORS + 1
        end
        task.wait(0.5)
    end
end)

local function updateCircle()
    local c = CAM.ViewportSize / 2
    for i = 1, SEGMENTS do
        local a = (i - 1) / SEGMENTS * 2 * math.pi
        dots[i].Position = UDim2.new(0, c.X + math.cos(a) * RADIUS, 0, c.Y + math.sin(a) * RADIUS)
    end
end

updateCircle()
CAM:GetPropertyChangedSignal("ViewportSize"):Connect(updateCircle)
RunService.RenderStepped:Connect(updateCircle)

-- === 射线拦截（子弹追踪核心）===
local function setupBulletTracking()
    local success, meta = pcall(getrawmetatable, game)
    if not success then return end
    
    local old = meta.__namecall
    setreadonly(meta, false)
    
    meta.__namecall = newcclosure(function(self, ...)
        local method = getnamecallmethod()
        local args = {...}
        
        if (method == "Raycast" or method == "FindPartOnRay") and self == Workspace then
            if not BULLET_TRACKING_ENABLED then return old(self, ...) end
            
            local origin, direction
            if method == "Raycast" then
                origin, direction = args[1], args[2]
            else
                local ray = args[1]
                if typeof(ray) == "Ray" then
                    origin, direction = ray.Origin, ray.Direction
                end
            end
            
            if origin and direction then
                local closestHead
                local closestDistance = math.huge
                local cameraPos = CAM.CFrame.Position
                local lookVec = CAM.CFrame.LookVector
                
                for _, plr in ipairs(Players:GetPlayers()) do
                    -- 检查白名单
                    if attackWhitelist[plr] then
                        continue
                    end
                    
                    if plr ~= LocalPlayer and plr.Character then
                        local char = plr.Character
                        local head = char:FindFirstChild("Head")
                        local hum = char:FindFirstChildOfClass("Humanoid")
                        
                        if head and hum and hum.Health > 0 and not char:FindFirstChild("ForceField") then
                            local dir = (head.Position - cameraPos).Unit
                            local angle = math.deg(math.acos(lookVec:Dot(dir)))
                            
                            if angle <= RADIUS then
                                local dist = (head.Position - cameraPos).Magnitude
                                if dist < closestDistance then
                                    closestHead = head
                                    closestDistance = dist
                                end
                            end
                        end
                    end
                end
                
                if closestHead then
                    return {
                        Instance = closestHead,
                        Position = closestHead.Position,
                        Normal = (closestHead.Position - origin).Unit,
                        Material = Enum.Material.Plastic
                    }
                end
            end
        end
        return old(self, ...)
    end)
    
    setreadonly(meta, true)
    return function()
        if meta and old then
            setreadonly(meta, false)
            meta.__namecall = old
            setreadonly(meta, true)
        end
    end
end

local cleanupMeta = setupBulletTracking()

-- 透视功能
local espObjects = {}

local function createESP(character)
    if not character or not character:IsA("Model") or espObjects[character] then return end
    
    local highlight = Instance.new("Highlight")
    highlight.Name = "ESP_Highlight"
    highlight.Adornee = character
    highlight.FillColor = Color3.fromRGB(255, 0, 0)
    highlight.FillTransparency = 0.5
    highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
    highlight.OutlineTransparency = 0
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = character
    
    espObjects[character] = highlight
    
    character.AncestryChanged:Connect(function(_, parent)
        if not parent then
            if highlight then highlight:Destroy() end
            espObjects[character] = nil
        end
    end)
end

local function toggleESP(state)
    ESP_ENABLED = state
    
    if ESP_ENABLED then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer and player.Character then
                createESP(player.Character)
            end
        end
        
        -- 监听新玩家
        Players.PlayerAdded:Connect(function(player)
            player.CharacterAdded:Connect(function(character)
                if ESP_ENABLED then
                    createESP(character)
                end
            end)
        end)
    else
        for character, highlight in pairs(espObjects) do
            if highlight then highlight:Destroy() end
        end
        espObjects = {}
    end
end

-- 穿墙功能
local noclipConnection

local function enableNoclip()
    if not LocalPlayer.Character then return end
    
    noclipConnection = RunService.Stepped:Connect(function()
        if LocalPlayer.Character then
            for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = false
                end
            end
        end
    end)
    
    NOCLIP_ENABLED = true
end

local function disableNoclip()
    if noclipConnection then
        noclipConnection:Disconnect()
        noclipConnection = nil
    end
    
    NOCLIP_ENABLED = false
end

local function toggleNoclip(state)
    if state then 
        enableNoclip() 
    else 
        disableNoclip() 
    end
end

-- ===== 移动端飞行系统 =====
local moveButtons = {} -- 存储移动按钮
local moveConnections = {} -- 存储移动按钮的事件连接
local bodyV = nil
local flySpeed = 30
local isFlying = false

-- 移动控制状态
local moveState = {
    forward = false,
    backward = false,
    up = false,
    down = false
}

-- 创建移动控制按钮函数
local function createMobileControls()
    -- 前进按钮
    local forwardBtn = Instance.new("TextButton")
    forwardBtn.Name = "MobileForward"
    forwardBtn.Size = UDim2.new(0, 50, 0, 50)
    forwardBtn.Position = UDim2.new(0.5, -25, 0.9, -130)
    forwardBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 200)
    forwardBtn.BackgroundTransparency = 0.5
    forwardBtn.BorderSizePixel = 0
    forwardBtn.Text = "前进"
    forwardBtn.TextColor3 = Color3.new(1, 1, 1)
    forwardBtn.TextSize = 14
    forwardBtn.Visible = false
    forwardBtn.ZIndex = 20
    forwardBtn.Parent = ScreenGui
    
    local forwardCorner = Instance.new("UICorner")
    forwardCorner.CornerRadius = UDim.new(0, 8)
    forwardCorner.Parent = forwardBtn
    
    -- 后退按钮
    local backwardBtn = Instance.new("TextButton")
    backwardBtn.Name = "MobileBackward"
    backwardBtn.Size = UDim2.new(0, 50, 0, 50)
    backwardBtn.Position = UDim2.new(0.5, -25, 0.9, -70)
    backwardBtn.BackgroundColor3 = Color3.fromRGB(200, 70, 70)
    backwardBtn.BackgroundTransparency = 0.5
    backwardBtn.BorderSizePixel = 0
    backwardBtn.Text = "后退"
    backwardBtn.TextColor3 = Color3.new(1, 1, 1)
    backwardBtn.TextSize = 14
    backwardBtn.Visible = false
    backwardBtn.ZIndex = 20
    backwardBtn.Parent = ScreenGui
    
    local backwardCorner = Instance.new("UICorner")
    backwardCorner.CornerRadius = UDim.new(0, 8)
    backwardCorner.Parent = backwardBtn
    
    -- 上升按钮
    local upBtn = Instance.new("TextButton")
    upBtn.Name = "MobileUp"
    upBtn.Size = UDim2.new(0, 50, 0, 50)
    upBtn.Position = UDim2.new(0.8, -25, 0.9, -130)
    upBtn.BackgroundColor3 = Color3.fromRGB(70, 200, 70)
    upBtn.BackgroundTransparency = 0.5
    upBtn.BorderSizePixel = 0
    upBtn.Text = "上升"
    upBtn.TextColor3 = Color3.new(1, 1, 1)
    upBtn.TextSize = 14
    upBtn.Visible = false
    upBtn.ZIndex = 20
    upBtn.Parent = ScreenGui
    
    local upCorner = Instance.new("UICorner")
    upCorner.CornerRadius = UDim.new(0, 8)
    upCorner.Parent = upBtn
    
    -- 下降按钮
    local downBtn = Instance.new("TextButton")
    downBtn.Name = "MobileDown"
    downBtn.Size = UDim2.new(0, 50, 0, 50)
    downBtn.Position = UDim2.new(0.8, -25, 0.9, -70)
    downBtn.BackgroundColor3 = Color3.fromRGB(200, 200, 70)
    downBtn.BackgroundTransparency = 0.5
    downBtn.BorderSizePixel = 0
    downBtn.Text = "下降"
    downBtn.TextColor3 = Color3.new(1, 1, 1)
    downBtn.TextSize = 14
    downBtn.Visible = false
    downBtn.ZIndex = 20
    downBtn.Parent = ScreenGui
    
    local downCorner = Instance.new("UICorner")
    downCorner.CornerRadius = UDim.new(0, 8)
    downCorner.Parent = downBtn
    
    -- 存储按钮
    moveButtons = {
        forward = forwardBtn,
        backward = backwardBtn,
        up = upBtn,
        down = downBtn
    }
    
    -- 设置按钮事件
    local function setupButtonEvents()
        -- 清理旧连接
        for _, conn in pairs(moveConnections) do
            conn:Disconnect()
        end
        moveConnections = {}
        
        -- 前进按钮
        table.insert(moveConnections, forwardBtn.MouseButton1Down:Connect(function()
            moveState.forward = true
        end))
        
        table.insert(moveConnections, forwardBtn.MouseButton1Up:Connect(function()
            moveState.forward = false
        end))
        
        table.insert(moveConnections, forwardBtn.MouseButton2Down:Connect(function()
            moveState.forward = false
        end))
        
        -- 后退按钮
        table.insert(moveConnections, backwardBtn.MouseButton1Down:Connect(function()
            moveState.backward = true
        end))
        
        table.insert(moveConnections, backwardBtn.MouseButton1Up:Connect(function()
            moveState.backward = false
        end))
        
        table.insert(moveConnections, backwardBtn.MouseButton2Down:Connect(function()
            moveState.backward = false
        end))
        
        -- 上升按钮
        table.insert(moveConnections, upBtn.MouseButton1Down:Connect(function()
            moveState.up = true
        end))
        
        table.insert(moveConnections, upBtn.MouseButton1Up:Connect(function()
            moveState.up = false
        end))
        
        table.insert(moveConnections, upBtn.MouseButton2Down:Connect(function()
            moveState.up = false
        end))
        
        -- 下降按钮
        table.insert(moveConnections, downBtn.MouseButton1Down:Connect(function()
            moveState.down = true
        end))
        
        table.insert(moveConnections, downBtn.MouseButton1Up:Connect(function()
            moveState.down = false
        end))
        
        table.insert(moveConnections, downBtn.MouseButton2Down:Connect(function()
            moveState.down = false
        end))
    end
    
    setupButtonEvents()
    return moveButtons
end

-- 飞行控制
local function updateFlight()
    if not isFlying or not bodyV then return end
    
    local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not root then return end
    
    local move = Vector3.new()
    
    -- 移动端按钮控制
    if moveState.forward then
        move = move + CAM.CFrame.LookVector
    end
    if moveState.backward then
        move = move - CAM.CFrame.LookVector
    end
    if moveState.up then
        move = move + Vector3.new(0, 1, 0)
    end
    if moveState.down then
        move = move - Vector3.new(0, 1, 0)
    end
    
    -- 应用移动
    if move.Magnitude > 0 then
        bodyV.Velocity = move.Unit * flySpeed
    else
        bodyV.Velocity = Vector3.new(0, 0, 0)
    end
end

-- 飞行主函数
local function toggleFly(state)
    if state == isFlying then return end
    
    isFlying = state
    
    if isFlying then
        local char = LocalPlayer.Character
        if not char then return end
        
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then
            char:WaitForChild("HumanoidRootPart", 2)
            root = char:FindFirstChild("HumanoidRootPart")
            if not root then return end
        end
        
        -- 创建飞行物理
        bodyV = Instance.new("BodyVelocity")
        bodyV.Name = "FlightBodyVelocity"
        bodyV.MaxForce = Vector3.new(1e6, 1e6, 1e6)
        bodyV.Velocity = Vector3.new()
        bodyV.Parent = root
        
        -- 创建移动控制按钮（如果不存在）
        if not moveButtons.forward then
            createMobileControls()
        end
        
        -- 显示移动控制按钮
        for _, btn in pairs(moveButtons) do
            if btn then btn.Visible = true end
        end
        
        -- 开始飞行更新
        RunService.RenderStepped:Connect(updateFlight)
        
    else
        -- 清理飞行物理
        if bodyV then
            bodyV:Destroy()
            bodyV = nil
        end
        
        -- 隐藏移动控制按钮
        for _, btn in pairs(moveButtons) do
            if btn then btn.Visible = false end
        end
        
        -- 重置移动状态
        for key in pairs(moveState) do
            moveState[key] = false
        end
    end
end

-- 传送功能
local function teleportToPlayer(player)
    if not player or not player.Character then return false end
    
    local humanoidRootPart = player.Character:FindFirstChild("HumanoidRootPart")
    local head = player.Character:FindFirstChild("Head")
    local localChar = LocalPlayer.Character
    local localHumanoidRootPart = localChar and localChar:FindFirstChild("HumanoidRootPart")
    
    if not humanoidRootPart or not head or not localHumanoidRootPart then return false end
    
    localHumanoidRootPart.CFrame = CFrame.new(head.Position + Vector3.new(0, 5, 0))
    return true
end

local function teleportToAllPlayersLoop()
    if teleportingAll then return end
    teleportingAll = true
    
    task.spawn(function()
        while teleportingAll do
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= LocalPlayer and player.Character and teleportingAll then
                    teleportToPlayer(player)
                    task.wait(1)
                end
            end
            task.wait(0.5)
        end
    end)
end

local function teleportToSelectedPlayerLoop()
    if not next(selectedPlayers) then return end
    if teleportingSelected then return end
    teleportingSelected = true
    
    task.spawn(function()
        while teleportingSelected do
            for player, _ in pairs(selectedPlayers) do
                if not teleportingSelected then break end
                if player and player.Character then
                    teleportToPlayer(player)
                    task.wait(1)
                end
            end
            task.wait(0.5)
        end
    end)
end

-- 彩色循环系统
local hue = 0
local activeStrokes = {}

local function AddActiveStroke(stroke)
    if stroke and stroke.Parent then 
        activeStrokes[stroke] = true 
    end
end

RunService.Heartbeat:Connect(function(deltaTime)
    hue = (hue + deltaTime * 360) % 360
    local color = Color3.fromHSV(hue / 360, 0.9, 1)
    
    for stroke in pairs(activeStrokes) do
        if stroke and stroke.Parent then 
            stroke.Color = color 
        end
    end
end)

-- 首次启动提示
task.spawn(function()
    task.wait(1)
    
    local notification = Instance.new("TextLabel")
    notification.Size = UDim2.new(0, 200, 0, 60)
    notification.Position = UDim2.new(1, -220, 1, -80)
    notification.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    notification.BackgroundTransparency = 0.2
    notification.BorderSizePixel = 0
    notification.Text = "祝你使用愉快"
    notification.TextColor3 = Color3.new(1, 1, 1)
    notification.TextSize = 14
    notification.TextWrapped = true
    notification.ZIndex = 100
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = notification
    
    notification.Parent = ScreenGui
    
    task.wait(3)
    notification:Destroy()
end)

-- === QQ音乐风格UI ===
local FloatingButton = Instance.new("TextButton")
FloatingButton.Name = "FloatingButton"
FloatingButton.Size = UDim2.new(0, 60, 0, 60)
FloatingButton.Position = UDim2.new(0, 20, 0.5, -30)
FloatingButton.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
FloatingButton.BackgroundTransparency = 0.3
FloatingButton.BorderSizePixel = 0
FloatingButton.Text = ""
FloatingButton.AutoButtonColor = false
FloatingButton.ZIndex = 10
FloatingButton.Active = true
FloatingButton.Draggable = true

local UICorner1 = Instance.new("UICorner")
UICorner1.CornerRadius = UDim.new(1, 0)
UICorner1.Parent = FloatingButton

local FloatingStroke = Instance.new("UIStroke")
FloatingStroke.Thickness = 2
FloatingStroke.Color = Color3.fromRGB(80, 80, 80)
FloatingStroke.Parent = FloatingButton

local FloatingText = Instance.new("TextLabel")
FloatingText.Size = UDim2.new(1, 0, 1, 0)
FloatingText.BackgroundTransparency = 1
FloatingText.Text = "离瑜二"
FloatingText.TextColor3 = Color3.new(1, 1, 1)
FloatingText.TextSize = 12
FloatingText.TextWrapped = true
FloatingText.ZIndex = 11
FloatingText.Parent = FloatingButton

FloatingButton.Parent = ScreenGui

-- 主面板
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 380, 0, 300)
MainFrame.Position = UDim2.new(0, 100, 0.5, -150)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
MainFrame.BackgroundTransparency = 0.2
MainFrame.BorderSizePixel = 0
MainFrame.ClipsDescendants = true
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Visible = false
MainFrame.ZIndex = 5

local UICorner2 = Instance.new("UICorner")
UICorner2.CornerRadius = UDim.new(0, 8)
UICorner2.Parent = MainFrame

local MainFrameStroke = Instance.new("UIStroke")
MainFrameStroke.Thickness = 2
MainFrameStroke.Color = Color3.fromRGB(80, 80, 80)
MainFrameStroke.Parent = MainFrame

-- 右上角关闭按钮
local CloseButton = Instance.new("TextButton")
CloseButton.Size = UDim2.new(0, 25, 0, 25)
CloseButton.Position = UDim2.new(1, -30, 0, 5)
CloseButton.BackgroundColor3 = Color3.fromRGB(220, 50, 50)
CloseButton.BackgroundTransparency = 0.3
CloseButton.BorderSizePixel = 0
CloseButton.Text = "X"
CloseButton.TextColor3 = Color3.new(1, 1, 1)
CloseButton.TextSize = 14
CloseButton.ZIndex = 20
CloseButton.Parent = MainFrame

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 4)
CloseCorner.Parent = CloseButton

-- 侧边栏
local Sidebar = Instance.new("Frame")
Sidebar.Size = UDim2.new(0, 80, 1, 0)
Sidebar.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
Sidebar.BackgroundTransparency = 0.3
Sidebar.BorderSizePixel = 0
Sidebar.ZIndex = 6

local UICorner4 = Instance.new("UICorner")
UICorner4.CornerRadius = UDim.new(0, 8)
UICorner4.Parent = Sidebar

-- 侧边栏按钮容器
local SidebarButtons = Instance.new("Frame")
SidebarButtons.Size = UDim2.new(1, 0, 1, -40)
SidebarButtons.Position = UDim2.new(0, 0, 0, 40)
SidebarButtons.BackgroundTransparency = 1
SidebarButtons.ZIndex = 7
SidebarButtons.Parent = Sidebar

local SidebarListLayout = Instance.new("UIListLayout")
SidebarListLayout.Padding = UDim.new(0, 5)
SidebarListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
SidebarListLayout.Parent = SidebarButtons

-- 侧边栏标题
local SidebarTitle = Instance.new("TextLabel")
SidebarTitle.Size = UDim2.new(1, 0, 0, 40)
SidebarTitle.BackgroundTransparency = 1
SidebarTitle.Text = "离瑜二"
SidebarTitle.TextColor3 = Color3.new(1, 1, 1)
SidebarTitle.TextSize = 16
SidebarTitle.Font = Enum.Font.GothamBold
SidebarTitle.ZIndex = 7
SidebarTitle.Parent = Sidebar

Sidebar.Parent = MainFrame

-- 内容区域
local ContentFrame = Instance.new("Frame")
ContentFrame.Size = UDim2.new(1, -80, 1, 0)
ContentFrame.Position = UDim2.new(0, 80, 0, 0)
ContentFrame.BackgroundTransparency = 1
ContentFrame.ClipsDescendants = true
ContentFrame.ZIndex = 6
ContentFrame.Parent = MainFrame

-- 滚动框架
local ScrollFrame = Instance.new("ScrollingFrame")
ScrollFrame.Size = UDim2.new(1, -10, 1, -10)
ScrollFrame.Position = UDim2.new(0, 5, 0, 5)
ScrollFrame.BackgroundTransparency = 1
ScrollFrame.ScrollBarImageColor3 = Color3.fromRGB(100, 100, 100)
ScrollFrame.ScrollBarThickness = 4
ScrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
ScrollFrame.BorderSizePixel = 0
ScrollFrame.ScrollingDirection = Enum.ScrollingDirection.Y
ScrollFrame.VerticalScrollBarInset = Enum.ScrollBarInset.Always
ScrollFrame.ZIndex = 7
ScrollFrame.Parent = ContentFrame

local ContentLayout = Instance.new("UIListLayout")
ContentLayout.Padding = UDim.new(0, 8)
ContentLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
ContentLayout.Parent = ScrollFrame

-- 页面容器
local Pages = {
    General = Instance.new("Frame"),
    Teleport = Instance.new("Frame"),
    Info = Instance.new("Frame")
}

for pageName, pageFrame in pairs(Pages) do
    pageFrame.Size = UDim2.new(1, -20, 0, 0)
    pageFrame.BackgroundTransparency = 1
    pageFrame.Visible = false
    pageFrame.ZIndex = 8
    pageFrame.Parent = ScrollFrame
    
    local pageLayout = Instance.new("UIListLayout")
    pageLayout.Padding = UDim.new(0, 8)
    pageLayout.Parent = pageFrame
    
    pageLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        pageFrame.Size = UDim2.new(1, -20, 0, pageLayout.AbsoluteContentSize.Y)
    end)
end

-- 更新滚动区域大小
ContentLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    ScrollFrame.CanvasSize = UDim2.new(0, 0, 0, ContentLayout.AbsoluteContentSize.Y + 20)
end)

-- 创建侧边栏按钮函数
local function CreateSidebarButton(text, pageName)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0, 60, 0, 60)
    button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    button.BackgroundTransparency = 0.3
    button.BorderSizePixel = 0
    button.Text = text
    button.TextColor3 = Color3.new(1, 1, 1)
    button.TextSize = 12
    button.TextWrapped = true
    button.ZIndex = 8
    button.AutoButtonColor = false
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = button
    
    button.MouseEnter:Connect(function() 
        button.BackgroundColor3 = Color3.fromRGB(60, 60, 60) 
    end)
    
    button.MouseLeave:Connect(function() 
        button.BackgroundColor3 = Color3.fromRGB(50, 50, 50) 
    end)
    
    button.MouseButton1Down:Connect(function() 
        button.BackgroundColor3 = Color3.fromRGB(70, 70, 70) 
    end)
    
    button.MouseButton1Up:Connect(function() 
        button.BackgroundColor3 = Color3.fromRGB(60, 60, 60) 
    end)
    
    button.MouseButton1Click:Connect(function()
        for _, page in pairs(Pages) do 
            page.Visible = false 
        end
        Pages[pageName].Visible = true
    end)
    
    button.Parent = SidebarButtons
    return button
end

-- 创建功能按钮函数
local function CreateFunctionButton(text, parent, toggleCallback)
    local buttonFrame = Instance.new("Frame")
    buttonFrame.Size = UDim2.new(1, 0, 0, 35)
    buttonFrame.BackgroundTransparency = 1
    buttonFrame.ZIndex = 9
    buttonFrame.Parent = parent
    
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(1, -50, 1, 0)
    button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    button.BackgroundTransparency = 0.3
    button.BorderSizePixel = 0
    button.Text = "  " .. text
    button.TextColor3 = Color3.new(1, 1, 1)
    button.TextSize = 12
    button.TextXAlignment = Enum.TextXAlignment.Left
    button.ZIndex = 10
    button.AutoButtonColor = false
    
    local padding = Instance.new("UIPadding")
    padding.PaddingLeft = UDim.new(0, 10)
    padding.Parent = button
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = button
    
    local toggleButton = Instance.new("TextButton")
    toggleButton.Size = UDim2.new(0, 40, 1, 0)
    toggleButton.Position = UDim2.new(1, -45, 0, 0)
    toggleButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    toggleButton.BorderSizePixel = 0
    toggleButton.Text = "关"
    toggleButton.TextColor3 = Color3.new(1, 1, 1)
    toggleButton.TextSize = 12
    toggleButton.ZIndex = 10
    toggleButton.AutoButtonColor = false
    
    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 6)
    toggleCorner.Parent = toggleButton
    
    local isEnabled = false
    
    local function UpdateButtonColors()
        local baseColor = isEnabled and Color3.fromRGB(70, 70, 70) or Color3.fromRGB(50, 50, 50)
        button.BackgroundColor3 = baseColor
        toggleButton.BackgroundColor3 = isEnabled and Color3.fromRGB(100, 100, 100) or Color3.fromRGB(80, 80, 80)
    end
    
    button.MouseEnter:Connect(function() 
        if not isEnabled then 
            button.BackgroundColor3 = Color3.fromRGB(60, 60, 60) 
        end 
    end)
    
    button.MouseLeave:Connect(UpdateButtonColors)
    
    button.MouseButton1Down:Connect(function() 
        if not isEnabled then 
            button.BackgroundColor3 = Color3.fromRGB(70, 70, 70) 
        end 
    end)
    
    button.MouseButton1Up:Connect(UpdateButtonColors)
    
    toggleButton.MouseButton1Click:Connect(function()
        isEnabled = not isEnabled
        toggleButton.Text = isEnabled and "开" or "关"
        UpdateButtonColors()
        if toggleCallback then 
            toggleCallback(isEnabled) 
        end
    end)
    
    button.Parent = buttonFrame
    toggleButton.Parent = buttonFrame
    
    return buttonFrame, toggleButton
end

-- 创建下拉框函数
local function CreateDropdown(text, parent, selectionCallback)
    local dropdownFrame = Instance.new("Frame")
    dropdownFrame.Size = UDim2.new(1, 0, 0, 35)
    dropdownFrame.BackgroundTransparency = 1
    dropdownFrame.ZIndex = 100
    dropdownFrame.Parent = parent

    local dropdownButton = Instance.new("TextButton")
    dropdownButton.Size = UDim2.new(1, 0, 1, 0)
    dropdownButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    dropdownButton.BackgroundTransparency = 0.3
    dropdownButton.BorderSizePixel = 0
    dropdownButton.Text = text
    dropdownButton.TextColor3 = Color3.new(1, 1, 1)
    dropdownButton.TextSize = 12
    dropdownButton.ZIndex = 101
    dropdownButton.AutoButtonColor = false

    local dropdownCorner = Instance.new("UICorner")
    dropdownCorner.CornerRadius = UDim.new(0, 6)
    dropdownCorner.Parent = dropdownButton

    -- 玩家列表
    local playerListFrame = Instance.new("Frame")
    playerListFrame.Size = UDim2.new(1, 0, 0, 180)
    playerListFrame.Position = UDim2.new(0, 0, 1, 5)
    playerListFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    playerListFrame.BackgroundTransparency = 0.1
    playerListFrame.BorderSizePixel = 0
    playerListFrame.Visible = false
    playerListFrame.ZIndex = 200
    playerListFrame.Parent = dropdownFrame

    local playerListCorner = Instance.new("UICorner")
    playerListCorner.CornerRadius = UDim.new(0, 6)
    playerListCorner.Parent = playerListFrame

    local playerScroll = Instance.new("ScrollingFrame")
    playerScroll.Size = UDim2.new(1, -5, 1, -5)
    playerScroll.Position = UDim2.new(0, 5, 0, 5)
    playerScroll.BackgroundTransparency = 1
    playerScroll.ScrollBarThickness = 4
    playerScroll.ZIndex = 201
    playerScroll.Parent = playerListFrame

    local playerListLayout = Instance.new("UIListLayout")
    playerListLayout.Padding = UDim.new(0, 2)
    playerListLayout.Parent = playerScroll

    local function updatePlayerList(selectedList)
        for _, child in pairs(playerScroll:GetChildren()) do
            if child:IsA("TextButton") then 
                child:Destroy() 
            end
        end
        
        for _, player in pairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local playerBtn = Instance.new("TextButton")
                playerBtn.Size = UDim2.new(1, -5, 0, 25)
                playerBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
                playerBtn.BackgroundTransparency = 0.3
                playerBtn.BorderSizePixel = 0
                playerBtn.Text = selectedList[player] and "✓ " .. player.Name or player.Name
                playerBtn.TextColor3 = selectedList[player] and Color3.fromRGB(0, 255, 0) or Color3.new(1, 1, 1)
                playerBtn.TextSize = 12
                playerBtn.ZIndex = 202
                playerBtn.AutoButtonColor = false
                
                playerBtn.MouseButton1Click:Connect(function()
                    if selectionCallback then
                        selectionCallback(player, not selectedList[player])
                        updatePlayerList(selectedList)
                        
                        -- 更新下拉框显示
                        local selectedCount = 0
                        for _ in pairs(selectedList) do 
                            selectedCount = selectedCount + 1 
                        end
                        
                        if selectedCount > 0 then
                            dropdownButton.Text = "已选择 " .. selectedCount .. " 名玩家"
                        else
                            dropdownButton.Text = text
                        end
                    end
                end)
                
                playerBtn.Parent = playerScroll
            end
        end
    end

    dropdownButton.MouseButton1Click:Connect(function()
        playerListFrame.Visible = not playerListFrame.Visible
        if playerListFrame.Visible and selectionCallback then 
            updatePlayerList(selectionCallback(nil, nil, true)) 
        end
    end)

    playerListFrame.Parent = dropdownFrame
    dropdownButton.Parent = dropdownFrame
    
    return dropdownFrame
end

-- 创建侧边栏按钮
CreateSidebarButton("通用功能", "General")
CreateSidebarButton("传送功能", "Teleport")
CreateSidebarButton("脚本信息", "Info")

-- === 通用页面功能 ===
-- 追踪白名单下拉框
local trackingWhitelistDropdown = CreateDropdown("追踪白名单", Pages.General, function(player, state, getOnly)
    if getOnly then return attackWhitelist end
    if player then
        if state then
            attackWhitelist[player] = true
        else
            attackWhitelist[player] = nil
        end
    end
    return attackWhitelist
end)

local trackButton, trackToggle = CreateFunctionButton("子弹追踪", Pages.General, function(enabled)
    BULLET_TRACKING_ENABLED = enabled
    ENABLED = enabled
    for _, d in ipairs(dots) do 
        d.Visible = enabled 
    end
end)

local espButton, espToggle = CreateFunctionButton("透视", Pages.General, function(enabled)
    toggleESP(enabled)
end)

local flyButton, flyToggle = CreateFunctionButton("飞行", Pages.General, function(enabled)
    toggleFly(enabled)
end)

local noclipButton, noclipToggle = CreateFunctionButton("穿墙", Pages.General, function(enabled)
    toggleNoclip(enabled)
end)

-- 速度按钮
local speedFrame = Instance.new("Frame")
speedFrame.Size = UDim2.new(1, 0, 0, 35)
speedFrame.BackgroundTransparency = 1
speedFrame.ZIndex = 9
speedFrame.Parent = Pages.General

local speedButton = Instance.new("TextButton")
speedButton.Size = UDim2.new(1, 0, 1, 0)
speedButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
speedButton.BackgroundTransparency = 0.3
speedButton.BorderSizePixel = 0
speedButton.Text = "  速度 " .. flySpeed
speedButton.TextColor3 = Color3.new(1, 1, 1)
speedButton.TextSize = 12
speedButton.TextXAlignment = Enum.TextXAlignment.Left
speedButton.ZIndex = 10
speedButton.AutoButtonColor = false

local speedCorner = Instance.new("UICorner")
speedCorner.CornerRadius = UDim.new(0, 6)
speedCorner.Parent = speedButton

speedButton.MouseButton1Click:Connect(function()
    if isFlying then
        flySpeed = flySpeed + 10
        if flySpeed > 150 then 
            flySpeed = 10 
        end
        speedButton.Text = "  速度 " .. flySpeed
    end
end)

speedButton.Parent = speedFrame

-- === 传送页面功能 ===
-- 玩家选择下拉框
local teleportDropdown = CreateDropdown("选择玩家 (多选)", Pages.Teleport, function(player, state, getOnly)
    if getOnly then return selectedPlayers end
    if player then
        if state then
            selectedPlayers[player] = true
        else
            selectedPlayers[player] = nil
        end
    end
    return selectedPlayers
end)

-- 传送功能按钮
local teleportAllButton, teleportAllToggle = CreateFunctionButton("循环传送所有人", Pages.Teleport, function(enabled)
    if enabled then
        teleportToAllPlayersLoop()
        teleportAllToggle.Text = "开"
    else
        teleportingAll = false
        teleportAllToggle.Text = "关"
    end
end)

local teleportSelectedButton, teleportSelectedToggle = CreateFunctionButton("循环传送选定玩家", Pages.Teleport, function(enabled)
    if enabled then
        if next(selectedPlayers) then
            teleportToSelectedPlayerLoop()
            teleportSelectedToggle.Text = "开"
        else
            teleportSelectedToggle.Text = "关"
        end
    else
        teleportingSelected = false
        teleportSelectedToggle.Text = "关"
    end
end)

-- 坐标记录功能
local savePosFrame = Instance.new("Frame")
savePosFrame.Size = UDim2.new(1, 0, 0, 35)
savePosFrame.BackgroundTransparency = 1
savePosFrame.ZIndex = 9
savePosFrame.Parent = Pages.Teleport

local savePosButton = Instance.new("TextButton")
savePosButton.Size = UDim2.new(1, 0, 1, 0)
savePosButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
savePosButton.BackgroundTransparency = 0.3
savePosButton.BorderSizePixel = 0
savePosButton.Text = "  记录位置"
savePosButton.TextColor3 = Color3.new(1, 1, 1)
savePosButton.TextSize = 12
savePosButton.TextXAlignment = Enum.TextXAlignment.Left
savePosButton.ZIndex = 10
savePosButton.AutoButtonColor = false

local savePosCorner = Instance.new("UICorner")
savePosCorner.CornerRadius = UDim.new(0, 6)
savePosCorner.Parent = savePosButton

savePosButton.MouseButton1Click:Connect(function()
    if LocalPlayer.Character then
        local root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if root then
            savedPosition = root.CFrame
            savePosButton.Text = "  ✓ 位置已记录"
            savePosButton.BackgroundColor3 = Color3.fromRGB(0, 180, 0)
            
            task.wait(3)
            savePosButton.Text = "  记录位置"
            savePosButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        end
    end
end)

savePosButton.Parent = savePosFrame

local returnPosFrame = Instance.new("Frame")
returnPosFrame.Size = UDim2.new(1, 0, 0, 35)
returnPosFrame.BackgroundTransparency = 1
returnPosFrame.ZIndex = 9
returnPosFrame.Parent = Pages.Teleport

local returnPosButton = Instance.new("TextButton")
returnPosButton.Size = UDim2.new(1, 0, 1, 0)
returnPosButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
returnPosButton.BackgroundTransparency = 0.3
returnPosButton.BorderSizePixel = 0
returnPosButton.Text = "  返回位置"
returnPosButton.TextColor3 = Color3.new(1, 1, 1)
returnPosButton.TextSize = 12
returnPosButton.TextXAlignment = Enum.TextXAlignment.Left
returnPosButton.ZIndex = 10
returnPosButton.AutoButtonColor = false

local returnPosCorner = Instance.new("UICorner")
returnPosCorner.CornerRadius = UDim.new(0, 6)
returnPosCorner.Parent = returnPosButton

returnPosButton.MouseButton1Click:Connect(function()
    if savedPosition and LocalPlayer.Character then
        local root = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if root then
            root.CFrame = savedPosition
        end
    end
end)

returnPosButton.Parent = returnPosFrame

-- === 信息页面 ===
local infoFrame = Instance.new("Frame")
infoFrame.Size = UDim2.new(1, 0, 0, 60)
infoFrame.BackgroundTransparency = 1
infoFrame.ZIndex = 9
infoFrame.Parent = Pages.Info

local authorLabel = Instance.new("TextLabel")
authorLabel.Size = UDim2.new(1, 0, 0, 25)
authorLabel.BackgroundTransparency = 1
authorLabel.Text = "作者：离瑜二"
authorLabel.TextColor3 = Color3.new(1, 1, 1)
authorLabel.TextSize = 14
authorLabel.ZIndex = 10
authorLabel.Parent = infoFrame

local deputyLabel = Instance.new("TextLabel")
deputyLabel.Size = UDim2.new(1, 0, 0, 25)
deputyLabel.Position = UDim2.new(0, 0, 0, 30)
deputyLabel.BackgroundTransparency = 1
deputyLabel.Text = "副官：嫩小雨"
deputyLabel.TextColor3 = Color3.new(1, 1, 1)
deputyLabel.TextSize = 14
deputyLabel.ZIndex = 10
deputyLabel.Parent = infoFrame

-- 默认显示通用页面
Pages.General.Visible = true

-- === 悬浮窗事件 ===
FloatingButton.MouseButton1Click:Connect(function()
    MainFrame.Visible = not MainFrame.Visible
end)

FloatingButton.MouseEnter:Connect(function()
    FloatingButton.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
end)

FloatingButton.MouseLeave:Connect(function()
    FloatingButton.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
end)

-- 右上角关闭按钮事件
CloseButton.MouseButton1Click:Connect(function()
    local closeNotification = Instance.new("Frame")
    closeNotification.Size = UDim2.new(0, 220, 0, 100)
    closeNotification.Position = UDim2.new(0.5, -110, 0.5, -50)
    closeNotification.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    closeNotification.BackgroundTransparency = 0.2
    closeNotification.BorderSizePixel = 0
    closeNotification.ZIndex = 100
    
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 8)
    closeCorner.Parent = closeNotification
    
    local messageLabel = Instance.new("TextLabel")
    messageLabel.Size = UDim2.new(1, 0, 0.5, 0)
    messageLabel.BackgroundTransparency = 1
    messageLabel.Text = "期待下一次相遇"
    messageLabel.TextColor3 = Color3.new(1, 1, 1)
    messageLabel.TextSize = 16
    messageLabel.ZIndex = 101
    messageLabel.Parent = closeNotification
    
    local cancelButton = Instance.new("TextButton")
    cancelButton.Size = UDim2.new(0.6, 0, 0.3, 0)
    cancelButton.Position = UDim2.new(0.2, 0, 0.6, 0)
    cancelButton.BackgroundColor3 = Color3.fromRGB(70, 70, 200)
    cancelButton.BackgroundTransparency = 0.3
    cancelButton.BorderSizePixel = 0
    cancelButton.Text = "取消"
    cancelButton.TextColor3 = Color3.new(1, 1, 1)
    cancelButton.TextSize = 14
    cancelButton.ZIndex = 101
    cancelButton.Parent = closeNotification
    
    local cancelCorner = Instance.new("UICorner")
    cancelCorner.CornerRadius = UDim.new(0, 6)
    cancelCorner.Parent = cancelButton
    
    closeNotification.Parent = ScreenGui
    
    cancelButton.MouseButton1Click:Connect(function()
        closeNotification:Destroy()
    end)
    
    task.spawn(function()
        task.wait(5)
        if closeNotification and closeNotification.Parent then
            closeNotification:Destroy()
            
            -- 清理所有功能
            toggleESP(false)
            toggleNoclip(false)
            toggleFly(false)
            BULLET_TRACKING_ENABLED = false
            teleportingAll = false
            teleportingSelected = false
            
            for _, d in ipairs(dots) do 
                d:Destroy() 
            end
            
            if ScreenGui then
                ScreenGui:Destroy()
            end
            
            -- 恢复元表
            if cleanupMeta then
                cleanupMeta()
            end
        end
    end)
end)

MainFrame.Parent = ScreenGui

-- 立即添加悬浮窗和主面板的彩色循环
AddActiveStroke(FloatingStroke)
AddActiveStroke(MainFrameStroke)

-- 重生时重新获取root
LocalPlayer.CharacterAdded:Connect(function(newChar)
    task.wait(0.5) -- 等待角色完全加载
    
    -- 重新启用穿墙如果之前是开启状态
    if NOCLIP_ENABLED then
        toggleNoclip(false) -- 先禁用
        task.wait(0.2)
        toggleNoclip(true) -- 重新启用
    end
    
    -- 如果飞行开启，重新初始化飞行
    if isFlying then
        toggleFly(false) -- 先关闭
        task.wait(0.3)
        toggleFly(true) -- 重新开启
    end
end)

-- 预创建移动控制按钮（但不显示）
createMobileControls()

print("离瑜二UI - 全新脚本加载完成！")
