local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local SCREEN_GUI_NAME = "KaijuAlphaHUD"
local COPY_NAME = "OverheadLevelCopy"
local CREDIT_TEXT = "made with love by Studly <3"

local THEMES = {
	Red = Color3.fromRGB(255, 55, 55),
	Green = Color3.fromRGB(65, 255, 95),
	Blue = Color3.fromRGB(70, 150, 255),
	RGB = nil,
}

local themeName = "Red"
local skullEnabled = false
local liveEnabled = false
local skullConn
local liveConn
local rgbConn
local alive = true

local screenGui
local panel
local titleLabel
local creditLabel
local themeLabel
local statusLabel
local mainTab
local settingsTab
local mainPage
local settingsPage
local skullButton
local liveButton
local redTheme
local greenTheme
local blueTheme
local rgbTheme
local currentTab = "Main"

local function addCorner(parent, radius)
	local corner = parent:FindFirstChildOfClass("UICorner")
	if not corner then
		corner = Instance.new("UICorner")
		corner.Parent = parent
	end
	corner.CornerRadius = radius
end

local function addStroke(parent, color, transparency, thickness)
	local stroke = parent:FindFirstChildOfClass("UIStroke")
	if not stroke then
		stroke = Instance.new("UIStroke")
		stroke.Parent = parent
	end
	stroke.Color = color
	stroke.Transparency = transparency
	stroke.Thickness = thickness
	return stroke
end

local function getAccent()
	if themeName ~= "RGB" then
		return THEMES[themeName]
	end

	local t = os.clock()
	local r = math.floor((math.sin(t * 2) * 0.5 + 0.5) * 255)
	local g = math.floor((math.sin(t * 2 + 2) * 0.5 + 0.5) * 255)
	local b = math.floor((math.sin(t * 2 + 4) * 0.5 + 0.5) * 255)
	return Color3.fromRGB(r, g, b)
end

local function colorToRgbString(color)
	return string.format("%d,%d,%d", math.floor(color.R * 255), math.floor(color.G * 255), math.floor(color.B * 255))
end

local function updateSkullAppearance(skullModel)
	local cube = skullModel:FindFirstChild("Cube", true)
	if not cube then
		return
	end

	local adornee = cube
	if not cube:IsA("BasePart") then
		adornee = cube:FindFirstChildWhichIsA("BasePart", true)
	end
	if not adornee then
		return
	end

	local billboard = adornee:FindFirstChild("MZSkullBillboard")
	if not billboard then
		billboard = Instance.new("BillboardGui")
		billboard.Name = "MZSkullBillboard"
		billboard.Adornee = adornee
		billboard.AlwaysOnTop = true
		billboard.LightInfluence = 0
		billboard.MaxDistance = math.huge
		billboard.Size = UDim2.fromOffset(240, 56)
		billboard.StudsOffset = Vector3.new(0, 8, 0)
		billboard.Parent = adornee
	end

	local card = billboard:FindFirstChild("Card")
	if not card then
		card = Instance.new("Frame")
		card.Name = "Card"
		card.Parent = billboard
	end
	card.Size = UDim2.fromScale(1, 1)
	card.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
	card.BackgroundTransparency = 0.08
	card.BorderSizePixel = 0
	addCorner(card, UDim.new(0, 10))
	addStroke(card, getAccent(), 0.02, 1.8)

	local label = card:FindFirstChild("MZSkullLabel")
	if not label then
		label = Instance.new("TextLabel")
		label.Name = "MZSkullLabel"
		label.BackgroundTransparency = 1
		label.Size = UDim2.fromScale(1, 1)
		label.Font = Enum.Font.GothamBold
		label.TextScaled = false
		label.TextSize = 24
		label.TextStrokeTransparency = 0
		label.TextStrokeColor3 = Color3.new(0, 0, 0)
		label.TextXAlignment = Enum.TextXAlignment.Center
		label.Parent = card
	end

	label.Text = "MZ Skull Marker"
	label.TextColor3 = getAccent()
end

local function setupSkull(skullModel)
	if not alive or not skullModel or not skullModel:IsA("Model") or skullModel.Name ~= "Skull" then
		return
	end

	local cube = skullModel:FindFirstChild("Cube", true)
	if not cube then
		return
	end

	local prompt = cube:FindFirstChildOfClass("ProximityPrompt") or cube:FindFirstChild("ProximityPrompt", true)
	if not prompt then
		prompt = Instance.new("ProximityPrompt")
		prompt.Name = "ProximityPrompt"
		prompt.ActionText = "Interact"
		prompt.ObjectText = "Skull"
		prompt.MaxActivationDistance = 10
		prompt.RequiresLineOfSight = false
		prompt:SetAttribute("KaijuAlphaCreated", true)
		prompt.Parent = cube
	end

	updateSkullAppearance(skullModel)
end

local function refreshAllSkulls()
	local mapFolder = workspace:FindFirstChild("Map")
	if not mapFolder then
		return
	end
	local debrisFolder = mapFolder:FindFirstChild("Debris")
	if not debrisFolder then
		return
	end
	for _, descendant in ipairs(debrisFolder:GetDescendants()) do
		if descendant:IsA("Model") and descendant.Name == "Skull" then
			updateSkullAppearance(descendant)
		end
	end
end

local function removeSkullEffects(skullModel)
	for _, descendant in ipairs(skullModel:GetDescendants()) do
		if descendant:IsA("BillboardGui") and descendant.Name == "MZSkullBillboard" then
			descendant:Destroy()
		elseif descendant:IsA("ProximityPrompt") and descendant:GetAttribute("KaijuAlphaCreated") then
			descendant:Destroy()
		end
	end
end

local function setupLiveCopy(model)
	if not alive or not model or not model:IsA("Model") then
		return
	end

	local overhead = model:FindFirstChild("Overhead")
	if not overhead or not overhead:IsA("BillboardGui") then
		return
	end

	local originalFrame = overhead:FindFirstChild("Frame")
	local originalLevel = originalFrame and originalFrame:FindFirstChild("Level")
	local originalLabel = originalLevel and originalLevel:FindFirstChildWhichIsA("TextLabel")
	if not originalLabel then
		return
	end

	local copyGui = model:FindFirstChild(COPY_NAME)
	if not copyGui then
		copyGui = Instance.new("BillboardGui")
		copyGui.Name = COPY_NAME
		copyGui.Parent = model
	end

	copyGui.Adornee = overhead.Adornee
	copyGui.AlwaysOnTop = true
	copyGui.Size = UDim2.new(0, 180, 0, 28)
	copyGui.StudsOffset = overhead.StudsOffset + Vector3.new(3, 0, 0)

	local copyFrame = copyGui:FindFirstChild("Frame")
	if not copyFrame then
		copyFrame = Instance.new("Frame")
		copyFrame.Name = "Frame"
		copyFrame.Parent = copyGui
	end
	copyFrame.BackgroundTransparency = 1
	copyFrame.BorderSizePixel = 0
	copyFrame.Size = UDim2.fromScale(1, 1)

	local copyLevel = copyFrame:FindFirstChild("Level")
	if not copyLevel then
		copyLevel = Instance.new("Frame")
		copyLevel.Name = "Level"
		copyLevel.Parent = copyFrame
	end
	copyLevel.BackgroundTransparency = 1
	copyLevel.Size = UDim2.fromScale(1, 1)

	local copyLabel = copyLevel:FindFirstChild("TextLabel")
	if not copyLabel then
		copyLabel = Instance.new("TextLabel")
		copyLabel.Name = "TextLabel"
		copyLabel.Parent = copyLevel
	end

	copyLabel.Size = UDim2.fromScale(1, 1)
	copyLabel.BackgroundTransparency = 1
	copyLabel.Text = originalLabel.Text
	copyLabel.TextScaled = false
	copyLabel.TextSize = 22
	copyLabel.Font = Enum.Font.GothamBold
	copyLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
	copyLabel.TextStrokeTransparency = 0
	copyLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
	copyLabel.TextWrapped = false
	copyLabel.TextXAlignment = Enum.TextXAlignment.Center
	copyLabel.TextYAlignment = Enum.TextYAlignment.Center

	if not copyGui:FindFirstChild("TextSyncBound") then
		local marker = Instance.new("BoolValue")
		marker.Name = "TextSyncBound"
		marker.Parent = copyGui
		originalLabel:GetPropertyChangedSignal("Text"):Connect(function()
			if copyLabel and copyLabel.Parent then
				copyLabel.Text = originalLabel.Text
			end
		end)
	end
end

local function removeLiveCopy(model)
	local copyGui = model:FindFirstChild(COPY_NAME)
	if copyGui then
		copyGui:Destroy()
	end
end

local function setSkullEnabled(enabled)
	skullEnabled = enabled
	local mapFolder = workspace:FindFirstChild("Map")
	local debrisFolder = mapFolder and mapFolder:FindFirstChild("Debris")

	if enabled then
		if debrisFolder and not skullConn then
			skullConn = debrisFolder.DescendantAdded:Connect(function(descendant)
				if skullEnabled and descendant:IsA("Model") and descendant.Name == "Skull" then
					setupSkull(descendant)
				end
			end)
		end
		if debrisFolder then
			for _, descendant in ipairs(debrisFolder:GetDescendants()) do
				if descendant:IsA("Model") and descendant.Name == "Skull" then
					setupSkull(descendant)
				end
			end
		end
	else
		if skullConn then
			skullConn:Disconnect()
			skullConn = nil
		end
		if debrisFolder then
			for _, descendant in ipairs(debrisFolder:GetDescendants()) do
				if descendant:IsA("Model") and descendant.Name == "Skull" then
					removeSkullEffects(descendant)
				end
			end
		end
	end
end

local function setLiveEnabled(enabled)
	liveEnabled = enabled
	local liveFolder = workspace:FindFirstChild("Live")
	if not liveFolder then
		return
	end

	if enabled then
		if not liveConn then
			liveConn = liveFolder.ChildAdded:Connect(function(child)
				if liveEnabled then
					setupLiveCopy(child)
				end
			end)
		end
		for _, child in ipairs(liveFolder:GetChildren()) do
			setupLiveCopy(child)
		end
	else
		if liveConn then
			liveConn:Disconnect()
			liveConn = nil
		end
		for _, child in ipairs(liveFolder:GetChildren()) do
			removeLiveCopy(child)
		end
	end
end

local function setTab(name)
	currentTab = name
	mainPage.Visible = name == "Main"
	settingsPage.Visible = name == "Settings"
	local accent = getAccent()
	mainTab.TextColor3 = name == "Main" and accent or Color3.fromRGB(255, 255, 255)
	settingsTab.TextColor3 = name == "Settings" and accent or Color3.fromRGB(255, 255, 255)
end

local function styleAccent()
	local accent = getAccent()
	local accentText = ('<font color="rgb(%s)">Kaiju</font> <font color="rgb(255,255,255)">Alpha</font>'):format(colorToRgbString(accent))
	titleLabel.Text = accentText
	titleLabel.RichText = true
	themeLabel.Text = "Theme: " .. themeName
	themeLabel.TextColor3 = accent
	creditLabel.Text = CREDIT_TEXT
	panel:FindFirstChildOfClass("UIStroke").Color = accent
	mainTab:FindFirstChildOfClass("UIStroke").Color = accent
	settingsTab:FindFirstChildOfClass("UIStroke").Color = accent
	skullButton:FindFirstChildOfClass("UIStroke").Color = accent
	liveButton:FindFirstChildOfClass("UIStroke").Color = accent
	redTheme:FindFirstChildOfClass("UIStroke").Color = accent
	greenTheme:FindFirstChildOfClass("UIStroke").Color = accent
	blueTheme:FindFirstChildOfClass("UIStroke").Color = accent
	rgbTheme:FindFirstChildOfClass("UIStroke").Color = accent

	mainTab.TextColor3 = currentTab == "Main" and accent or Color3.fromRGB(255, 255, 255)
	settingsTab.TextColor3 = currentTab == "Settings" and accent or Color3.fromRGB(255, 255, 255)

	if skullEnabled then
		refreshAllSkulls()
	end

	if skullButton then
		skullButton.TextColor3 = skullEnabled and accent or Color3.fromRGB(255, 255, 255)
	end
	if liveButton then
		liveButton.TextColor3 = liveEnabled and accent or Color3.fromRGB(255, 255, 255)
	end
end

local function setTheme(name)
	themeName = name
	if rgbConn then
		rgbConn:Disconnect()
		rgbConn = nil
	end
	styleAccent()

	if themeName == "RGB" then
		rgbConn = RunService.RenderStepped:Connect(function()
			if not alive then
				return
			end
			styleAccent()
		end)
	end
end

local function disableAll(reason)
	alive = false
	skullEnabled = false
	liveEnabled = false
	if skullConn then skullConn:Disconnect() skullConn = nil end
	if liveConn then liveConn:Disconnect() liveConn = nil end
	if rgbConn then rgbConn:Disconnect() rgbConn = nil end
	if statusLabel then
		statusLabel.Text = reason or "Disabled"
		statusLabel.TextColor3 = Color3.fromRGB(255, 120, 120)
	end
	if screenGui then
		screenGui.Enabled = false
	end
end

local function buildGui()
	screenGui = playerGui:FindFirstChild(SCREEN_GUI_NAME)
	if not screenGui then
		screenGui = Instance.new("ScreenGui")
		screenGui.Name = SCREEN_GUI_NAME
		screenGui.ResetOnSpawn = false
		screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
		screenGui.Parent = playerGui
	end

	panel = screenGui:FindFirstChild("MainPanel")
	if not panel then
		panel = Instance.new("Frame")
		panel.Name = "MainPanel"
		panel.Parent = screenGui
	end
	panel.Position = UDim2.new(0, 18, 0, 18)
	panel.Size = UDim2.new(0, 408, 0, 208)
	panel.BackgroundColor3 = Color3.fromRGB(6, 6, 6)
	panel.BackgroundTransparency = 0.02
	panel.BorderSizePixel = 0
	panel.Active = true
	panel.ClipsDescendants = true
	addCorner(panel, UDim.new(0, 16))
	addStroke(panel, getAccent(), 0.02, 1.6)

	local titleBar = panel:FindFirstChild("TitleBar")
	if not titleBar then
		titleBar = Instance.new("Frame")
		titleBar.Name = "TitleBar"
		titleBar.Parent = panel
	end
	titleBar.BackgroundTransparency = 1
	titleBar.Size = UDim2.new(1, 0, 0, 58)
	titleBar.ZIndex = 2

	titleLabel = titleBar:FindFirstChild("Title")
	if not titleLabel then
		titleLabel = Instance.new("TextLabel")
		titleLabel.Name = "Title"
		titleLabel.Parent = titleBar
	end
	titleLabel.BackgroundTransparency = 1
	titleLabel.Position = UDim2.new(0, 16, 0, 11)
	titleLabel.Size = UDim2.new(0, 190, 0, 24)
	titleLabel.Font = Enum.Font.GothamBold
	titleLabel.TextSize = 22
	titleLabel.TextXAlignment = Enum.TextXAlignment.Left
	titleLabel.RichText = true
	titleLabel.ZIndex = 3

	themeLabel = titleBar:FindFirstChild("Theme")
	if not themeLabel then
		themeLabel = Instance.new("TextLabel")
		themeLabel.Name = "Theme"
		themeLabel.Parent = titleBar
	end
	themeLabel.BackgroundTransparency = 1
	themeLabel.Position = UDim2.new(0, 16, 0, 34)
	themeLabel.Size = UDim2.new(0, 150, 0, 14)
	themeLabel.Font = Enum.Font.Gotham
	themeLabel.TextSize = 12
	themeLabel.TextXAlignment = Enum.TextXAlignment.Left
	themeLabel.ZIndex = 3

	creditLabel = titleBar:FindFirstChild("Credit")
	if not creditLabel then
		creditLabel = Instance.new("TextLabel")
		creditLabel.Name = "Credit"
		creditLabel.Parent = titleBar
	end
	creditLabel.BackgroundTransparency = 1
	creditLabel.Position = UDim2.new(1, -168, 0, 20)
	creditLabel.Size = UDim2.new(0, 152, 0, 14)
	creditLabel.Font = Enum.Font.Gotham
	creditLabel.Text = CREDIT_TEXT
	creditLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
	creditLabel.TextSize = 12
	creditLabel.TextXAlignment = Enum.TextXAlignment.Right
	creditLabel.ZIndex = 3

	local tabBar = panel:FindFirstChild("TabBar")
	if not tabBar then
		tabBar = Instance.new("Frame")
		tabBar.Name = "TabBar"
		tabBar.Parent = panel
	end
	tabBar.BackgroundTransparency = 1
	tabBar.Position = UDim2.new(0, 256, 0, 14)
	tabBar.Size = UDim2.new(0, 132, 0, 28)
	tabBar.ZIndex = 3

	local tabLayout = tabBar:FindFirstChildOfClass("UIListLayout")
	if not tabLayout then
		tabLayout = Instance.new("UIListLayout")
		tabLayout.Parent = tabBar
	end
	tabLayout.FillDirection = Enum.FillDirection.Horizontal
	tabLayout.Padding = UDim.new(0, 8)

	local function makeTabButton(name, text)
		local button = tabBar:FindFirstChild(name)
		if not button then
			button = Instance.new("TextButton")
			button.Name = name
			button.Parent = tabBar
		end
		button.Size = UDim2.new(0, 62, 1, 0)
		button.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
		button.BorderSizePixel = 0
		button.AutoButtonColor = false
		button.Font = Enum.Font.GothamBold
		button.Text = text
		button.TextSize = 13
		button.TextColor3 = Color3.fromRGB(255, 255, 255)
		button.ZIndex = 3
		addCorner(button, UDim.new(0, 8))
		addStroke(button, getAccent(), 0.02, 1.5)
		return button
	end

	mainTab = makeTabButton("MainTab", "Main")
	settingsTab = makeTabButton("SettingsTab", "Settings")

	mainPage = panel:FindFirstChild("MainPage")
	if not mainPage then
		mainPage = Instance.new("Frame")
		mainPage.Name = "MainPage"
		mainPage.Parent = panel
	end
	mainPage.BackgroundTransparency = 1
	mainPage.Position = UDim2.new(0, 16, 0, 64)
	mainPage.Size = UDim2.new(1, -32, 1, -72)
	mainPage.ZIndex = 2

	settingsPage = panel:FindFirstChild("SettingsPage")
	if not settingsPage then
		settingsPage = Instance.new("Frame")
		settingsPage.Name = "SettingsPage"
		settingsPage.Parent = panel
	end
	settingsPage.BackgroundTransparency = 1
	settingsPage.Position = UDim2.new(0, 16, 0, 64)
	settingsPage.Size = UDim2.new(1, -32, 1, -72)
	settingsPage.Visible = false
	settingsPage.ZIndex = 2

	local mainButtons = mainPage:FindFirstChild("Buttons")
	if not mainButtons then
		mainButtons = Instance.new("Frame")
		mainButtons.Name = "Buttons"
		mainButtons.Parent = mainPage
	end
	mainButtons.BackgroundTransparency = 1
	mainButtons.Size = UDim2.new(1, 0, 0, 60)
	mainButtons.ZIndex = 2

	local mainLayout = mainButtons:FindFirstChildOfClass("UIListLayout")
	if not mainLayout then
		mainLayout = Instance.new("UIListLayout")
		mainLayout.Parent = mainButtons
	end
	mainLayout.FillDirection = Enum.FillDirection.Horizontal
	mainLayout.Padding = UDim.new(0, 10)

	local function makeMainButton(name, text)
		local button = mainButtons:FindFirstChild(name)
		if not button then
			button = Instance.new("TextButton")
			button.Name = name
			button.Parent = mainButtons
		end
		button.Size = UDim2.new(0.5, -5, 1, 0)
		button.BackgroundColor3 = Color3.fromRGB(12, 12, 12)
		button.BorderSizePixel = 0
		button.AutoButtonColor = false
		button.Font = Enum.Font.GothamBold
		button.Text = text
		button.TextSize = 14
		button.TextColor3 = Color3.fromRGB(255, 255, 255)
		button.ZIndex = 2
		addCorner(button, UDim.new(0, 10))
		addStroke(button, getAccent(), 0.02, 1.9)
		return button
	end

	skullButton = makeMainButton("SkullButton", "MZ Skull Marker")
	liveButton = makeMainButton("LiveButton", "Easy Level Viewer")

	local desc = mainPage:FindFirstChild("Description")
	if not desc then
		desc = Instance.new("TextLabel")
		desc.Name = "Description"
		desc.Parent = mainPage
	end
	desc.BackgroundTransparency = 1
	desc.Position = UDim2.new(0, 0, 1, -18)
	desc.Size = UDim2.new(1, 0, 0, 14)
	desc.Font = Enum.Font.Gotham
	desc.Text = "Toggle each feature on or off"
	desc.TextColor3 = Color3.fromRGB(200, 200, 200)
	desc.TextSize = 12
	desc.TextXAlignment = Enum.TextXAlignment.Left
	desc.ZIndex = 2

	local settingsHeader = settingsPage:FindFirstChild("Header")
	if not settingsHeader then
		settingsHeader = Instance.new("TextLabel")
		settingsHeader.Name = "Header"
		settingsHeader.Parent = settingsPage
	end
	settingsHeader.BackgroundTransparency = 1
	settingsHeader.Position = UDim2.new(0, 0, 0, 0)
	settingsHeader.Size = UDim2.new(1, 0, 0, 16)
	settingsHeader.Font = Enum.Font.GothamBold
	settingsHeader.Text = "Theme"
	settingsHeader.TextColor3 = Color3.fromRGB(255, 255, 255)
	settingsHeader.TextSize = 14
	settingsHeader.TextXAlignment = Enum.TextXAlignment.Left
	settingsHeader.ZIndex = 2

	local themeRow = settingsPage:FindFirstChild("ThemeRow")
	if not themeRow then
		themeRow = Instance.new("Frame")
		themeRow.Name = "ThemeRow"
		themeRow.Parent = settingsPage
	end
	themeRow.BackgroundTransparency = 1
	themeRow.Position = UDim2.new(0, 0, 0, 24)
	themeRow.Size = UDim2.new(1, 0, 0, 36)
	themeRow.ZIndex = 2

	local themeLayout = themeRow:FindFirstChildOfClass("UIListLayout")
	if not themeLayout then
		themeLayout = Instance.new("UIListLayout")
		themeLayout.Parent = themeRow
	end
	themeLayout.FillDirection = Enum.FillDirection.Horizontal
	themeLayout.Padding = UDim.new(0, 8)

	local function makeThemeButton(name)
		local button = themeRow:FindFirstChild(name)
		if not button then
			button = Instance.new("TextButton")
			button.Name = name
			button.Parent = themeRow
		end
		button.Size = UDim2.new(0, 72, 0, 36)
		button.BackgroundColor3 = Color3.fromRGB(12, 12, 12)
		button.BorderSizePixel = 0
		button.AutoButtonColor = false
		button.Font = Enum.Font.GothamBold
		button.Text = name
		button.TextSize = 13
		button.TextColor3 = Color3.fromRGB(255, 255, 255)
		button.ZIndex = 2
		addCorner(button, UDim.new(0, 10))
		addStroke(button, getAccent(), 0.02, 1.5)
		return button
	end

	redTheme = makeThemeButton("Red")
	greenTheme = makeThemeButton("Green")
	blueTheme = makeThemeButton("Blue")
	rgbTheme = makeThemeButton("RGB")

	statusLabel = panel:FindFirstChild("Status")
	if not statusLabel then
		statusLabel = Instance.new("TextLabel")
		statusLabel.Name = "Status"
		statusLabel.Parent = panel
	end
	statusLabel.BackgroundTransparency = 1
	statusLabel.Position = UDim2.new(0, 16, 1, -18)
	statusLabel.Size = UDim2.new(1, -32, 0, 14)
	statusLabel.Font = Enum.Font.Gotham
	statusLabel.Text = "Ready"
	statusLabel.TextColor3 = Color3.fromRGB(220, 220, 220)
	statusLabel.TextSize = 12
	statusLabel.TextXAlignment = Enum.TextXAlignment.Left
	statusLabel.ZIndex = 2

	local function pulse(button)
		local original = button.BackgroundColor3
		button.BackgroundColor3 = Color3.fromRGB(24, 24, 24)
		task.delay(0.12, function()
			if button and button.Parent then
				button.BackgroundColor3 = original
			end
		end)
	end

	local function setButtonState(button, enabled)
		button.TextColor3 = enabled and getAccent() or Color3.fromRGB(255, 255, 255)
	end

	mainTab.MouseButton1Click:Connect(function()
		setTab("Main")
	end)

	settingsTab.MouseButton1Click:Connect(function()
		setTab("Settings")
	end)

	redTheme.MouseButton1Click:Connect(function()
		setTheme("Red")
	end)

	greenTheme.MouseButton1Click:Connect(function()
		setTheme("Green")
	end)

	blueTheme.MouseButton1Click:Connect(function()
		setTheme("Blue")
	end)

	rgbTheme.MouseButton1Click:Connect(function()
		setTheme("RGB")
	end)

	skullButton.MouseButton1Click:Connect(function()
		setSkullEnabled(not skullEnabled)
		setButtonState(skullButton, skullEnabled)
		statusLabel.Text = skullEnabled and "Skull marker on" or "Skull marker off"
		statusLabel.TextColor3 = skullEnabled and getAccent() or Color3.fromRGB(220, 220, 220)
		pulse(skullButton)
	end)

	liveButton.MouseButton1Click:Connect(function()
		setLiveEnabled(not liveEnabled)
		setButtonState(liveButton, liveEnabled)
		statusLabel.Text = liveEnabled and "Easy level viewer on" or "Easy level viewer off"
		statusLabel.TextColor3 = liveEnabled and getAccent() or Color3.fromRGB(220, 220, 220)
		pulse(liveButton)
	end)

	creditLabel:GetPropertyChangedSignal("Text"):Connect(function()
		if creditLabel.Text ~= CREDIT_TEXT then
			disableAll("Tamper detected")
		end
	end)

	local dragging = false
	local dragStart
	local startPos

	panel.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			dragStart = input.Position
			startPos = panel.Position
			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
				end
			end)
		end
	end)

	UserInputService.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			local delta = input.Position - dragStart
			panel.Position = UDim2.new(
				startPos.X.Scale,
				startPos.X.Offset + delta.X,
				startPos.Y.Scale,
				startPos.Y.Offset + delta.Y
			)
		end
	end)

	setTab("Main")
	styleAccent()
	setButtonState(skullButton, false)
	setButtonState(liveButton, false)
end

buildGui()
