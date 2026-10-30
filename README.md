--============================================================
--  biel's auto farm!  |  "LESS" UI EDITION
--  Compatible with: Delta, Solara, Xeno
--============================================================

local Players            = game:GetService("Players")
local RunService         = game:GetService("RunService")
local Workspace          = game:GetService("Workspace")
local TweenService       = game:GetService("TweenService")
local UserInputService   = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

-- ▸ STATE ───────────────────────────────────────────────────
local farming          = false
local loopRunning      = false
local noclip           = false
local speed            = 25
local coinsCollected   = 0
local roundLimit       = 40
local autoResetEnabled = true
local bagFull          = false
local isElite          = false

-- ▸ COLORS (Matched to Image) ───────────────────────────────
local C = {
    bg         = Color3.fromRGB(20, 20, 23),    -- Main background
    sidebar    = Color3.fromRGB(16, 16, 18),    -- Left panel
    card       = Color3.fromRGB(26, 26, 30),    -- Inner cards
    cardHover  = Color3.fromRGB(35, 35, 40),
    accent     = Color3.fromRGB(139, 92, 246),  -- Purple
    accentDark = Color3.fromRGB(109, 40, 217),
    text       = Color3.fromRGB(255, 255, 255),
    subtext    = Color3.fromRGB(160, 160, 170),
    border     = Color3.fromRGB(45, 45, 52),
    trackOff   = Color3.fromRGB(60, 60, 70),
    success    = Color3.fromRGB(34, 197, 94),
    danger     = Color3.fromRGB(239, 68, 68),
}

-- ▸ EXECUTOR SAFE PARENT ────────────────────────────────────
local function getGuiParent()
    local ok, hui = pcall(function() return gethui() end)
    if ok and hui then return hui end
    return LocalPlayer:WaitForChild("PlayerGui")
end

local function protectGui(gui)
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(gui) end end)
    pcall(function() if protectgui then protectgui(gui) end end)
end

-- ▸ UI HELPERS ──────────────────────────────────────────────
local function corner(inst, r)
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, r); c.Parent = inst
end
local function stroke(inst, col, th)
    local s = Instance.new("UIStroke")
    s.Color, s.Thickness, s.ApplyStrokeMode = col or C.border, th or 1, Enum.ApplyStrokeMode.Border
    s.Parent = inst
    return s
end

-- ▸ NOCLIP & LOGIC ──────────────────────────────────────────
local noclipConn
local function startNoclip()
    if noclipConn then return end
    noclip = true
    noclipConn = RunService.Stepped:Connect(function()
        if not noclip or not LocalPlayer.Character then return end
        for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = false end
        end
    end)
end

local function stopNoclip()
    noclip = false
    if noclipConn then noclipConn:Disconnect(); noclipConn = nil end
    if LocalPlayer.Character then
        for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = true end
        end
    end
end

local function findNearestCoin()
    local char = LocalPlayer.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local myPos = root.Position
    local nearest, nearestDist = nil, math.huge
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj:IsA("BasePart") and obj.Parent and obj.Name:lower():find("coin") then
            local d = (obj.Position - myPos).Magnitude
            if d < nearestDist then
                nearest, nearestDist = obj, d
            end
        end
    end
    return nearest
end

local function moveToCoin(coin)
    local char = LocalPlayer.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bv.Velocity = Vector3.zero
    bv.Parent = root

    local startPos = root.Position
    local targetPos = coin.Position
    local dist = (targetPos - startPos).Magnitude
    if dist < 2 then bv:Destroy(); return end

    local dur = dist / math.max(speed, 1)
    local t0 = tick()
    while farming and coin.Parent and tick() - t0 < dur do
        local t = (tick() - t0) / dur
        local target = startPos:Lerp(targetPos, t)
        local dir = target - root.Position
        if dir.Magnitude > 0.5 then bv.Velocity = dir.Unit * speed end
        RunService.RenderStepped:Wait()
    end
    bv.Velocity = Vector3.zero
    bv:Destroy()
end

--============================================================
--  BUILD THE UI
--============================================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "BielsAutoFarm"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.DisplayOrder = 999
protectGui(screenGui)
screenGui.Parent = getGuiParent()

-- MAIN FRAME
local main = Instance.new("Frame")
main.Size = UDim2.new(0, 640, 0, 420)
main.Position = UDim2.new(0.5, -320, 0.5, -210)
main.BackgroundColor3 = C.bg
main.BorderSizePixel = 0
main.ClipsDescendants = true
main.Active = true
main.Parent = screenGui
corner(main, 10)
stroke(main, C.border, 1)

-- HEADER (Top Bar)
local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 40)
header.BackgroundColor3 = C.bg
header.BorderSizePixel = 0
header.Parent = main

local title = Instance.new("TextLabel")
title.Size = UDim2.new(0, 250, 1, 0)
title.Position = UDim2.new(0, 16, 0, 0)
title.BackgroundTransparency = 1
title.Text = "biel's auto farm!"
title.TextColor3 = C.text
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

-- Fake Keybind Dropdown
local keybindBox = Instance.new("TextButton")
keybindBox.Size = UDim2.new(0, 100, 0, 24)
keybindBox.Position = UDim2.new(0.5, -50, 0, 8)
keybindBox.BackgroundColor3 = C.sidebar
keybindBox.Text = "Keybind 0/1"
keybindBox.TextColor3 = C.text
keybindBox.Font = Enum.Font.Gotham
keybindBox.TextSize = 11
keybindBox.Parent = header
corner(keybindBox, 6)
stroke(keybindBox, C.border, 1)

-- Close Button
local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0, 30, 0, 30)
closeBtn.Position = UDim2.new(1, -40, 0, 5)
closeBtn.BackgroundColor3 = C.card
closeBtn.Text = "✕"
closeBtn.TextColor3 = C.subtext
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 14
closeBtn.Parent = header
corner(closeBtn, 6)
closeBtn.MouseButton1Click:Connect(function() screenGui:Destroy() end)

-- SIDEBAR
local sidebar = Instance.new("Frame")
sidebar.Size = UDim2.new(0, 180, 1, -40)
sidebar.Position = UDim2.new(0, 0, 0, 40)
sidebar.BackgroundColor3 = C.sidebar
sidebar.BorderSizePixel = 0
sidebar.Parent = main
stroke(sidebar, C.border, 1)

-- Profile Section
local profile = Instance.new("Frame")
profile.Size = UDim2.new(1, 0, 0, 60)
profile.BackgroundTransparency = 1
profile.Parent = sidebar

local avatar = Instance.new("ImageLabel")
avatar.Size = UDim2.new(0, 32, 0, 32)
avatar.Position = UDim2.new(0, 16, 0, 14)
avatar.BackgroundColor3 = C.card
avatar.Image = "rbxthumb://type=AvatarHeadShot&id="..LocalPlayer.UserId.."&w=150&h=150"
avatar.Parent = profile
corner(avatar, 16)

local pName = Instance.new("TextLabel")
pName.Size = UDim2.new(1, -60, 0, 16)
pName.Position = UDim2.new(0, 56, 0, 14)
pName.BackgroundTransparency = 1
pName.Text = LocalPlayer.DisplayName
pName.TextColor3 = C.text
pName.Font = Enum.Font.GothamBold
pName.TextSize = 13
pName.TextXAlignment = Enum.TextXAlignment.Left
pName.Parent = profile

local pGame = Instance.new("TextLabel")
pGame.Size = UDim2.new(1, -60, 0, 12)
pGame.Position = UDim2.new(0, 56, 0, 30)
pGame.BackgroundTransparency = 1
pGame.Text = "Murder Mystery 2"
pGame.TextColor3 = C.subtext
pGame.Font = Enum.Font.Gotham
pGame.TextSize = 10
pGame.TextXAlignment = Enum.TextXAlignment.Left
pGame.Parent = profile

-- Search Bar
local search = Instance.new("Frame")
search.Size = UDim2.new(1, -32, 0, 28)
search.Position = UDim2.new(0, 16, 0, 62)
search.BackgroundColor3 = C.bg
search.Parent = sidebar
corner(search, 6)
stroke(search, C.border, 1)

local sIcon = Instance.new("TextLabel")
sIcon.Size = UDim2.new(0, 24, 1, 0)
sIcon.Text = "🔍"
sIcon.BackgroundTransparency = 1
sIcon.TextColor3 = C.subtext
sIcon.TextSize = 12
sIcon.Parent = search

local sTxt = Instance.new("TextLabel")
sTxt.Size = UDim2.new(1, -24, 1, 0)
sTxt.Position = UDim2.new(0, 24, 0, 0)
sTxt.BackgroundTransparency = 1
sTxt.Text = "Search"
sTxt.TextColor3 = C.subtext
sTxt.Font = Enum.Font.Gotham
sTxt.TextSize = 11
sTxt.TextXAlignment = Enum.TextXAlignment.Left
sTxt.Parent = search

-- Navigation Tabs
local navList = Instance.new("Frame")
navList.Size = UDim2.new(1, 0, 1, -100)
navList.Position = UDim2.new(0, 0, 0, 100)
navList.BackgroundTransparency = 1
navList.Parent = sidebar

local navLayout = Instance.new("UIListLayout")
navLayout.Padding = UDim.new(0, 2)
navLayout.Parent = navList

local function createNavBtn(text, icon, active)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -16, 0, 32)
    btn.BackgroundColor3 = active and C.cardHover or C.sidebar
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.Parent = navList
    corner(btn, 6)
    if active then stroke(btn, C.border, 1) end

    local ico = Instance.new("TextLabel")
    ico.Size = UDim2.new(0, 32, 1, 0)
    ico.Position = UDim2.new(0, 8, 0, 0)
    ico.BackgroundTransparency = 1
    ico.Text = icon
    ico.TextColor3 = active and C.accent or C.subtext
    ico.TextSize = 14
    ico.Parent = btn

    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(1, -40, 1, 0)
    txt.Position = UDim2.new(0, 40, 0, 0)
    txt.BackgroundTransparency = 1
    txt.Text = text
    txt.TextColor3 = active and C.text or C.subtext
    txt.Font = active and Enum.Font.GothamBold or Enum.Font.Gotham
    txt.TextSize = 12
    txt.TextXAlignment = Enum.TextXAlignment.Left
    txt.Parent = btn

    btn.MouseEnter:Connect(function()
        if not active then TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = C.card}):Play() end
    end)
    btn.MouseLeave:Connect(function()
        if not active then TweenService:Create(btn, TweenInfo.new(0.2), {BackgroundColor3 = C.sidebar}):Play() end
    end)
    return btn
end

createNavBtn("Farm", "⚔️", true)
createNavBtn("Settings", "⚙️", false)
createNavBtn("ESP", "👁️", false)
createNavBtn("Teleport", "📍", false)
createNavBtn("Troll Fun", "😈", false)

-- CONTENT AREA
local content = Instance.new("ScrollingFrame")
content.Size = UDim2.new(1, -180, 1, -40)
content.Position = UDim2.new(0, 180, 0, 40)
content.BackgroundColor3 = C.bg
content.BorderSizePixel = 0
content.CanvasSize = UDim2.new(0, 0, 0, 0)
content.AutomaticCanvasSize = Enum.AutomaticSize.Y
content.ScrollBarThickness = 2
content.ScrollBarImageColor3 = C.accent
content.Parent = main

local cPad = Instance.new("UIPadding")
cPad.PaddingTop = UDim.new(0, 16)
cPad.PaddingLeft = UDim.new(0, 16)
cPad.PaddingRight = UDim.new(0, 16)
cPad.PaddingBottom = UDim.new(0, 16)
cPad.Parent = content

local cLayout = Instance.new("UIListLayout")
cLayout.Padding = UDim.new(0, 12)
cLayout.SortOrder = Enum.SortOrder.LayoutOrder
cLayout.Parent = content

-- Helper to create Cards
local function createCard(titleText, order)
    local card = Instance.new("Frame")
    card.Size = UDim2.new(1, 0, 0, 0)
    card.AutomaticSize = Enum.AutomaticSize.Y
    card.BackgroundColor3 = C.card
    card.Parent = content
    corner(card, 8)
    stroke(card, C.border, 1)
    card.LayoutOrder = order

    local cLayoutInner = Instance.new("UIListLayout")
    cLayoutInner.Padding = UDim.new(0, 8)
    cLayoutInner.SortOrder = Enum.SortOrder.LayoutOrder
    cLayoutInner.Parent = card

    local cPadInner = Instance.new("UIPadding")
    cPadInner.PaddingTop = UDim.new(0, 12)
    cPadInner.PaddingBottom = UDim.new(0, 12)
    cPadInner.PaddingLeft = UDim.new(0, 12)
    cPadInner.PaddingRight = UDim.new(0, 12)
    cPadInner.Parent = card

    local titleBox = Instance.new("Frame")
    titleBox.Size = UDim2.new(1, 0, 0, 20)
    titleBox.BackgroundTransparency = 1
    titleBox.LayoutOrder = 1
    titleBox.Parent = card

    local tText = Instance.new("TextLabel")
    tText.Size = UDim2.new(1, -20, 1, 0)
    tText.BackgroundTransparency = 1
    tText.Text = titleText
    tText.TextColor3 = C.text
    tText.Font = Enum.Font.GothamBold
    tText.TextSize = 13
    tText.TextXAlignment = Enum.TextXAlignment.Left
    tText.Parent = titleBox

    local arrow = Instance.new("TextLabel")
    arrow.Size = UDim2.new(0, 20, 1, 0)
    arrow.Position = UDim2.new(1, -20, 0, 0)
    arrow.BackgroundTransparency = 1
    arrow.Text = "⌄"
    arrow.TextColor3 = C.subtext
    arrow.TextSize = 14
    arrow.Parent = titleBox

    return card
end

-- Helper: Toggle Switch
local function createToggle(parent, text, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 24)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -50, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = C.subtext
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local sw = Instance.new("TextButton")
    sw.Size = UDim2.new(0, 36, 0, 20)
    sw.Position = UDim2.new(1, -36, 0.5, -10)
    sw.BackgroundColor3 = default and C.accent or C.trackOff
    sw.Text = ""
    sw.AutoButtonColor = false
    sw.Parent = row
    corner(sw, 10)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 16, 0, 16)
    knob.Position = default and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
    knob.BackgroundColor3 = C.text
    knob.Parent = sw
    corner(knob, 8)

    local state = default
    sw.MouseButton1Click:Connect(function()
        state = not state
        TweenService:Create(sw, TweenInfo.new(0.2), {BackgroundColor3 = state and C.accent or C.trackOff}):Play()
        TweenService:Create(knob, TweenInfo.new(0.2), {Position = state and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)}):Play()
        if callback then callback(state) end
    end)
    return row
end

-- Helper: Slider
local function createSlider(parent, text, min, max, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 40)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -50, 0, 16)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = C.subtext
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(0, 50, 0, 16)
    valLbl.Position = UDim2.new(1, -50, 0, 0)
    valLbl.BackgroundTransparency = 1
    valLbl.Text = tostring(default)
    valLbl.TextColor3 = C.text
    valLbl.Font = Enum.Font.GothamBold
    valLbl.TextSize = 12
    valLbl.TextXAlignment = Enum.TextXAlignment.Right
    valLbl.Parent = row

    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, 0, 0, 4)
    track.Position = UDim2.new(0, 0, 0, 28)
    track.BackgroundColor3 = C.trackOff
    track.Parent = row
    corner(track, 2)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((default-min)/(max-min), 0, 1, 0)
    fill.BackgroundColor3 = C.accent
    fill.Parent = track
    corner(fill, 2)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 12, 0, 12)
    knob.Position = UDim2.new((default-min)/(max-min), -6, 0.5, -6)
    knob.BackgroundColor3 = C.text
    knob.Parent = track
    corner(knob, 6)

    local dragging = false
    local function update(input)
        local pos = math.clamp((input.Position.X - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
        local val = math.floor(min + (max - min) * pos)
        fill.Size = UDim2.new(pos, 0, 1, 0)
        knob.Position = UDim2.new(pos, -6, 0.5, -6)
        valLbl.Text = tostring(val)
        if callback then callback(val) end
    end

    track.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            update(input)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            update(input)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

--============================================================
--  CARD 1: AUTO FARM
--============================================================
local cardFarm = createCard("Auto Farm", 1)

local startToggle = createToggle(cardFarm, "Coin Autofarm", false, function(state)
    farming = state
    if farming then
        startNoclip()
        if not loopRunning then task.spawn(farmLoop) end
    else
        stopNoclip()
    end
end)

createSlider(cardFarm, "Farm Speed", 10, 80, speed, function(val)
    speed = val
end)

--============================================================
--  CARD 2: COIN BAG (Custom status card)
--============================================================
local cardBag = createCard("Coin Bag Status", 2)

local bagRow = Instance.new("Frame")
bagRow.Size = UDim2.new(1, 0, 0, 24)
bagRow.BackgroundTransparency = 1
bagRow.Parent = cardBag

local bagLbl = Instance.new("TextLabel")
bagLbl.Size = UDim2.new(1, -120, 1, 0)
bagLbl.BackgroundTransparency = 1
bagLbl.Text = "Bag Status"
bagLbl.TextColor3 = C.subtext
bagLbl.Font = Enum.Font.Gotham
bagLbl.TextSize = 12
bagLbl.TextXAlignment = Enum.TextXAlignment.Left
bagLbl.Parent = bagRow

local bagVal = Instance.new("TextLabel")
bagVal.Size = UDim2.new(0, 100, 1, 0)
bagVal.Position = UDim2.new(1, -100, 0, 0)
bagVal.BackgroundTransparency = 1
bagVal.Text = "0 / 40"
bagVal.TextColor3 = C.text
bagVal.Font = Enum.Font.GothamBold
bagVal.TextSize = 13
bagVal.TextXAlignment = Enum.TextXAlignment.Right
bagVal.Parent = bagRow

local function updateBag()
    bagVal.Text = string.format("%d / %d", coinsCollected, roundLimit)
    if coinsCollected >= roundLimit then
        bagFull = true
        bagVal.TextColor3 = C.success
        bagVal.Text = "FULL"
    else
        bagFull = false
        bagVal.TextColor3 = C.text
    end
end

--============================================================
--  CARD 3: SETTINGS
--============================================================
local cardSettings = createCard("Settings", 3)

createToggle(cardSettings, "Auto-Reset When Bag Full", autoResetEnabled, function(state)
    autoResetEnabled = state
end)

createToggle(cardSettings, "Anti-AFK (Visual)", true, function() end)
createToggle(cardSettings, "Performance Mode (Visual)", false, function() end)

--============================================================
--  CORE LOOP
--============================================================
local resetting = false
local function doReset()
    if resetting then return end
    resetting = true
    stopNoclip()
    if LocalPlayer.Character then LocalPlayer.Character:BreakJoints() end
    coinsCollected = 0
    updateBag()
    LocalPlayer.CharacterAdded:Wait()
    task.wait(1.5)
    if farming then startNoclip() end
    resetting = false
end

local function farmLoop()
    loopRunning = true
    while farming do
        local coin = findNearestCoin()
        if coin then
            moveToCoin(coin)
            task.wait(0.1)
            if coin.Parent == nil then
                coinsCollected = coinsCollected + 1
                updateBag()
            end
        else
            task.wait(0.4)
        end

        updateBag()

        if bagFull and autoResetEnabled and farming then
            doReset()
        end
        task.wait(0.15)
    end
    loopRunning = false
end

--============================================================
--  DRAG & INIT
--============================================================
local dragging, dragStart, startPos, dragInput
header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = main.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)
header.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - dragStart
        main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

updateBag()

print("[MM2] biel's auto farm! Loaded.")
