--==================================================================--
--   MXME PVP  ·  v1.0
--   Premium PvP companion menu for Roblox
--
--   Install : Roblox Studio > StarterPlayer > StarterPlayerScripts
--             > Insert a LocalScript > paste this whole file
--   Toggle  : RightShift  (rebind it in Settings)
--==================================================================--

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local Lighting         = game:GetService("Lighting")
local StarterGui       = game:GetService("StarterGui")
local GuiService       = game:GetService("GuiService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui   = LocalPlayer:WaitForChild("PlayerGui")
local Mouse       = LocalPlayer:GetMouse()

local VERSION = "1.0"
local WINDOW_W, WINDOW_H = 740, 480
local SIDEBAR_W, TOPBAR_H = 200, 68

--==================================================================--
-- THEME & STATE
--==================================================================--

local Theme = {
	Bg        = Color3.fromRGB(12, 12, 18),
	Panel     = Color3.fromRGB(18, 18, 27),
	Card      = Color3.fromRGB(24, 24, 36),
	CardHover = Color3.fromRGB(31, 31, 46),
	Stroke    = Color3.fromRGB(40, 40, 58),
	Text      = Color3.fromRGB(238, 238, 250),
	SubText   = Color3.fromRGB(148, 148, 172),
	Muted     = Color3.fromRGB(92, 92, 116),
	Accent    = Color3.fromRGB(124, 92, 255),
	Success   = Color3.fromRGB(62, 214, 140),
	Warning   = Color3.fromRGB(255, 186, 66),
	Danger    = Color3.fromRGB(255, 78, 102),
}

local AccentPresets = {
	{ Name = "Violet",  Color = Color3.fromRGB(124, 92, 255) },
	{ Name = "Azure",   Color = Color3.fromRGB(64, 140, 255) },
	{ Name = "Cyan",    Color = Color3.fromRGB(40, 200, 230) },
	{ Name = "Mint",    Color = Color3.fromRGB(52, 211, 153) },
	{ Name = "Crimson", Color = Color3.fromRGB(255, 70, 95) },
	{ Name = "Rose",    Color = Color3.fromRGB(255, 92, 190) },
	{ Name = "Amber",   Color = Color3.fromRGB(255, 160, 50) },
	{ Name = "Gold",    Color = Color3.fromRGB(240, 205, 70) },
}

local Settings = {
	MenuKey        = Enum.KeyCode.RightShift,
	AccentColor    = Theme.Accent,
	UIScale        = 1,
	Notifications  = true,
	JoinAlerts     = true,
	MenuBlur       = true,
	Watermark      = true,
	RainbowAccent  = false,

	Crosshair      = false,
	CrossStyle     = "Cross",
	CrossSize      = 8,
	CrossThickness = 2,
	CrossGap       = 4,
	CrossAccent    = true,
	CrossOutline   = true,

	KillFeed       = true,
	TargetHud      = true,
	LowHpWarning   = true,
	LowHpThreshold = 30,
	DamageFlash    = true,
}

local Stats = { Kills = 0, Deaths = 0, Streak = 0, BestStreak = 0 }
local SessionStart = os.clock()
local Running = true
local MenuOpen = false
local KeybindListening = false
local CurrentTab = nil
local FPS = 60
local Conns = {}

--==================================================================--
-- UTILITIES
--==================================================================--

local function Track(connection)
	table.insert(Conns, connection)
	return connection
end

local function New(className, props, children)
	local inst = Instance.new(className)
	if inst:IsA("GuiObject") then inst.BorderSizePixel = 0 end
	if inst:IsA("GuiButton") then inst.AutoButtonColor = false end
	local parent
	for key, value in pairs(props or {}) do
		if key == "Parent" then parent = value else inst[key] = value end
	end
	for _, child in ipairs(children or {}) do child.Parent = inst end
	if parent then inst.Parent = parent end
	return inst
end

local function Corner(radius) return New("UICorner", { CornerRadius = UDim.new(0, radius or 8) }) end
local function Round() return New("UICorner", { CornerRadius = UDim.new(1, 0) }) end

local function Stroke(color, transparency, thickness)
	return New("UIStroke", {
		Color = color or Theme.Stroke,
		Transparency = transparency or 0,
		Thickness = thickness or 1,
		ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
	})
end

local function HPadding(left, right)
	return New("UIPadding", { PaddingLeft = UDim.new(0, left), PaddingRight = UDim.new(0, right or left) })
end

local function Tween(inst, props, duration, style, direction)
	local tween = TweenService:Create(inst, TweenInfo.new(
		duration or 0.25,
		style or Enum.EasingStyle.Quint,
		direction or Enum.EasingDirection.Out
	), props)
	tween:Play()
	return tween
end

local function Label(props)
	local p = {
		BackgroundTransparency = 1,
		Font = Enum.Font.GothamMedium,
		Text = "",
		TextSize = 13,
		TextColor3 = Theme.Text,
		TextXAlignment = Enum.TextXAlignment.Left,
		TextTruncate = Enum.TextTruncate.AtEnd,
	}
	for key, value in pairs(props) do p[key] = value end
	if props.AutomaticSize then p.TextTruncate = Enum.TextTruncate.None end
	return New("TextLabel", p)
end

local function Headshot(userId)
	return ("rbxthumb://type=AvatarHeadShot&id=%d&w=150&h=150"):format(userId)
end

local function FormatTime(seconds)
	seconds = math.floor(seconds)
	local h, m, s = seconds // 3600, (seconds % 3600) // 60, seconds % 60
	if h > 0 then return ("%d:%02d:%02d"):format(h, m, s) end
	return ("%02d:%02d"):format(m, s)
end

local function Shade(color, amount) return color:Lerp(Color3.new(0, 0, 0), amount) end
local function Tint(color, amount) return color:Lerp(Color3.new(1, 1, 1), amount) end

local function HealthColor(ratio)
	if ratio > 0.6 then return Theme.Success end
	if ratio > 0.3 then return Theme.Warning end
	return Theme.Danger
end

local function Escape(text)
	return (text:gsub("&", "&amp;"):gsub("<", "&lt;"):gsub(">", "&gt;"))
end

local function GetHumanoid(player)
	local character = player and player.Character
	return character and character:FindFirstChildOfClass("Humanoid")
end

local function GetRoot(player)
	local character = player and player.Character
	return character and character:FindFirstChild("HumanoidRootPart")
end

local function HealthOf(player)
	local humanoid = GetHumanoid(player)
	if humanoid then return humanoid.Health, humanoid.MaxHealth end
	return 0, 100
end

local function DistanceTo(player)
	local a, b = GetRoot(LocalPlayer), GetRoot(player)
	if a and b then return (a.Position - b.Position).Magnitude end
	return nil
end

local function PlayerFromPart(part)
	local model = part and part:FindFirstAncestorOfClass("Model")
	while model do
		local player = Players:GetPlayerFromCharacter(model)
		if player then return player end
		model = model:FindFirstAncestorOfClass("Model")
	end
	return nil
end

--==================================================================--
-- ACCENT SYSTEM (every accent-colored element follows the theme)
--==================================================================--

local AccentBindings, AccentListeners = {}, {}

local function BindAccent(inst, prop, transform)
	table.insert(AccentBindings, { inst, prop, transform })
	inst[prop] = transform and transform(Theme.Accent) or Theme.Accent
	return inst
end

local function OnAccent(fn)
	table.insert(AccentListeners, fn)
	fn(Theme.Accent)
end

local function SetAccent(color, instant)
	Theme.Accent = color
	for i = #AccentBindings, 1, -1 do
		local inst, prop, transform = AccentBindings[i][1], AccentBindings[i][2], AccentBindings[i][3]
		if inst:IsDescendantOf(game) then
			local target = transform and transform(color) or color
			if instant then inst[prop] = target else Tween(inst, { [prop] = target }, 0.35) end
		else
			table.remove(AccentBindings, i)
		end
	end
	for _, fn in ipairs(AccentListeners) do fn(color) end
end

--==================================================================--
-- SCREEN GUIS
--==================================================================--

for _, name in ipairs({ "MxmePvP", "MxmePvP_HUD" }) do
	local old = PlayerGui:FindFirstChild(name)
	if old then old:Destroy() end
end
local oldBlur = Lighting:FindFirstChild("MxmePvP_Blur")
if oldBlur then oldBlur:Destroy() end

local Gui = New("ScreenGui", {
	Name = "MxmePvP", ResetOnSpawn = false, DisplayOrder = 100,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling, Parent = PlayerGui,
})
local HudGui = New("ScreenGui", {
	Name = "MxmePvP_HUD", ResetOnSpawn = false, DisplayOrder = 99, IgnoreGuiInset = true,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling, Parent = PlayerGui,
})
local Blur = New("BlurEffect", { Name = "MxmePvP_Blur", Size = 0, Parent = Lighting })

local viewport = workspace.CurrentCamera.ViewportSize
if viewport.X > 100 then
	Settings.UIScale = math.clamp(math.min((viewport.X - 40) / WINDOW_W, (viewport.Y - 100) / WINDOW_H), 0.6, 1)
end

--==================================================================--
-- WINDOW
--==================================================================--

local Holder = New("Frame", {
	Name = "Window", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
	Size = UDim2.fromOffset(WINDOW_W, WINDOW_H), BackgroundTransparency = 1, Visible = false, Parent = Gui,
})
local WindowScale = New("UIScale", { Scale = Settings.UIScale, Parent = Holder })

New("ImageLabel", {
	Name = "Shadow", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, 0.5, 8),
	Size = UDim2.new(1, 80, 1, 80), BackgroundTransparency = 1, Image = "rbxassetid://6014261993",
	ImageColor3 = Color3.new(0, 0, 0), ImageTransparency = 0.3, ScaleType = Enum.ScaleType.Slice,
	SliceCenter = Rect.new(49, 49, 450, 450), Parent = Holder,
})

local Main = New("Frame", { Name = "Main", Size = UDim2.fromScale(1, 1), BackgroundColor3 = Theme.Bg, Parent = Holder }, { Corner(14) })
BindAccent(New("UIStroke", { Thickness = 1.2, Transparency = 0.55, ApplyStrokeMode = Enum.ApplyStrokeMode.Border, Parent = Main }), "Color")

-- Sidebar ----------------------------------------------------------
local Sidebar = New("Frame", { Size = UDim2.new(0, SIDEBAR_W, 1, 0), BackgroundColor3 = Theme.Panel, Parent = Main }, { Corner(14) })
New("Frame", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromScale(1, 0), Size = UDim2.new(0, 14, 1, 0), BackgroundColor3 = Theme.Panel, Parent = Sidebar })
New("Frame", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.fromScale(1, 0), Size = UDim2.new(0, 1, 1, 0), BackgroundColor3 = Theme.Stroke, Parent = Sidebar })

local Brand = New("Frame", { Position = UDim2.fromOffset(18, 20), Size = UDim2.new(1, -36, 0, 44), BackgroundTransparency = 1, Parent = Sidebar })
local LogoBadge = New("Frame", { Size = UDim2.fromOffset(44, 44), BackgroundColor3 = Color3.new(1, 1, 1), Parent = Brand }, { Corner(12) })
local LogoGradient = New("UIGradient", { Parent = LogoBadge })
Label({ Text = "M", Font = Enum.Font.GothamBlack, TextSize = 24, TextXAlignment = Enum.TextXAlignment.Center, Size = UDim2.fromScale(1, 1), Parent = LogoBadge })
local BrandTitle = Label({ Text = "MXME PVP", Font = Enum.Font.GothamBlack, TextSize = 18, TextColor3 = Color3.new(1, 1, 1), Position = UDim2.fromOffset(56, 4), Size = UDim2.new(1, -56, 0, 20), Parent = Brand })
local TitleShine = New("UIGradient", { Parent = BrandTitle })
Label({ Text = "Premium Combat Suite", Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(56, 25), Size = UDim2.new(1, -56, 0, 14), Parent = Brand })
OnAccent(function(c)
	LogoGradient.Color = ColorSequence.new(Tint(c, 0.25), Shade(c, 0.35))
	TitleShine.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Tint(c, 0.45)),
		ColorSequenceKeypoint.new(0.5, Color3.new(1, 1, 1)),
		ColorSequenceKeypoint.new(1, Tint(c, 0.45)),
	})
end)
New("Frame", { Position = UDim2.fromOffset(18, 80), Size = UDim2.new(1, -36, 0, 1), BackgroundColor3 = Theme.Stroke, Parent = Sidebar })

local NAV_TOP, NAV_STEP = 22, 42
local NavHolder = New("Frame", { Position = UDim2.fromOffset(14, 96), Size = UDim2.new(1, -28, 0, 260), BackgroundTransparency = 1, Parent = Sidebar })
Label({ Text = "MENU", Font = Enum.Font.GothamBold, TextSize = 10, TextColor3 = Theme.Muted, Position = UDim2.fromOffset(10, 0), Size = UDim2.new(1, -10, 0, 14), Parent = NavHolder })
local NavIndicator = New("Frame", { Position = UDim2.fromOffset(0, NAV_TOP), Size = UDim2.new(1, 0, 0, 38), BackgroundTransparency = 0.82, Parent = NavHolder }, { Corner(9) })
BindAccent(NavIndicator, "BackgroundColor3")
BindAccent(New("Frame", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.fromScale(0, 0.5), Size = UDim2.fromOffset(3, 18), Parent = NavIndicator }, { Round() }), "BackgroundColor3")

local UserCard = New("Frame", { AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 14, 1, -14), Size = UDim2.new(1, -28, 0, 58), BackgroundColor3 = Theme.Card, Parent = Sidebar }, { Corner(12), Stroke() })
New("ImageLabel", { Position = UDim2.fromOffset(10, 10), Size = UDim2.fromOffset(38, 38), BackgroundColor3 = Theme.Panel, Image = Headshot(LocalPlayer.UserId), Parent = UserCard }, { Round() })
New("Frame", { Position = UDim2.fromOffset(37, 37), Size = UDim2.fromOffset(12, 12), BackgroundColor3 = Theme.Success, ZIndex = 2, Parent = UserCard }, { Round(), Stroke(Theme.Card, 0, 2) })
Label({ Text = LocalPlayer.DisplayName, Font = Enum.Font.GothamBold, TextSize = 13, Position = UDim2.fromOffset(58, 12), Size = UDim2.new(1, -66, 0, 16), Parent = UserCard })
local UserCardSub = Label({ Text = "Online", Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(58, 30), Size = UDim2.new(1, -66, 0, 14), Parent = UserCard })

-- Content + top bar --------------------------------------------------
local Content = New("Frame", { Position = UDim2.fromOffset(SIDEBAR_W, 0), Size = UDim2.new(1, -SIDEBAR_W, 1, 0), BackgroundTransparency = 1, Parent = Main })
local Topbar = New("Frame", { Size = UDim2.new(1, 0, 0, TOPBAR_H), BackgroundTransparency = 1, Parent = Content })
local PageTitle = Label({ Font = Enum.Font.GothamBold, TextSize = 20, Position = UDim2.fromOffset(24, 16), Size = UDim2.new(1, -230, 0, 24), Parent = Topbar })
local PageSubtitle = Label({ Font = Enum.Font.Gotham, TextSize = 12, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(24, 40), Size = UDim2.new(1, -230, 0, 16), Parent = Topbar })
New("Frame", { Position = UDim2.new(0, 24, 1, -1), Size = UDim2.new(1, -48, 0, 1), BackgroundColor3 = Theme.Stroke, Parent = Topbar })

local Controls = New("Frame", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -20, 0, 20), Size = UDim2.fromOffset(200, 30), BackgroundTransparency = 1, Parent = Topbar }, {
	New("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, HorizontalAlignment = Enum.HorizontalAlignment.Right, VerticalAlignment = Enum.VerticalAlignment.Center, Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }),
})
local ClockPill = Label({ Text = "00:00", Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = Theme.SubText, AutomaticSize = Enum.AutomaticSize.X, Size = UDim2.fromOffset(0, 30), BackgroundTransparency = 0, BackgroundColor3 = Theme.Card, LayoutOrder = 1, Parent = Controls })
Corner(8).Parent = ClockPill
Stroke().Parent = ClockPill
HPadding(12).Parent = ClockPill

local function TopButton(text, hoverColor, order)
	local button = New("TextButton", { Size = UDim2.fromOffset(30, 30), BackgroundColor3 = Theme.Card, Text = text, Font = Enum.Font.GothamBold, TextSize = 13, TextColor3 = Theme.SubText, LayoutOrder = order, Parent = Controls }, { Corner(8), Stroke() })
	button.MouseEnter:Connect(function() Tween(button, { BackgroundColor3 = hoverColor, TextColor3 = Color3.new(1, 1, 1) }, 0.15) end)
	button.MouseLeave:Connect(function() Tween(button, { BackgroundColor3 = Theme.Card, TextColor3 = Theme.SubText }, 0.15) end)
	return button
end
local MinimizeButton = TopButton("—", Theme.CardHover, 2)
local CloseButton = TopButton("✕", Theme.Danger, 3)

local PageHolder = New("Frame", { Position = UDim2.fromOffset(0, TOPBAR_H), Size = UDim2.new(1, 0, 1, -TOPBAR_H), BackgroundTransparency = 1, ClipsDescendants = true, Parent = Content })

-- Dragging -----------------------------------------------------------
local function MakeDraggable(handle)
	local dragging, dragStart, startPos = false, nil, nil
	handle.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging, dragStart, startPos = true, input.Position, Holder.Position
			local connection
			connection = input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragging = false
					connection:Disconnect()
				end
			end)
		end
	end)
	Track(UserInputService.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			local delta = input.Position - dragStart
			Holder.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
		end
	end))
end
MakeDraggable(Topbar)
MakeDraggable(Brand)

--==================================================================--
-- NOTIFICATIONS & DIALOGS
--==================================================================--

local NotifyStack = New("Frame", { AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -18, 1, -18), Size = UDim2.fromOffset(300, 500), BackgroundTransparency = 1, Parent = Gui }, {
	New("UIListLayout", { VerticalAlignment = Enum.VerticalAlignment.Bottom, Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder }),
})
local notifyOrder = 0
local NotifyColors = { success = Theme.Success, warning = Theme.Warning, danger = Theme.Danger }

local function Notify(title, message, kind, duration)
	if not Settings.Notifications or not Running then return end
	duration = duration or 3.5
	notifyOrder += 1
	local color = NotifyColors[kind] or Theme.Accent
	local slot = New("Frame", { Size = UDim2.new(1, 0, 0, 64), BackgroundTransparency = 1, LayoutOrder = notifyOrder, Parent = NotifyStack })
	local card = New("Frame", { Position = UDim2.fromOffset(330, 0), Size = UDim2.fromScale(1, 1), BackgroundColor3 = Theme.Panel, Parent = slot }, { Corner(12), Stroke() })
	New("Frame", { Position = UDim2.fromOffset(12, 14), Size = UDim2.fromOffset(4, 34), BackgroundColor3 = color, Parent = card }, { Round() })
	Label({ Text = title, Font = Enum.Font.GothamBold, TextSize = 13, Position = UDim2.fromOffset(28, 12), Size = UDim2.new(1, -40, 0, 18), Parent = card })
	Label({ Text = message, Font = Enum.Font.Gotham, TextSize = 12, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(28, 31), Size = UDim2.new(1, -40, 0, 16), Parent = card })
	local progress = New("Frame", { AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 12, 1, -6), Size = UDim2.new(1, -24, 0, 2), BackgroundColor3 = color, BackgroundTransparency = 0.4, Parent = card }, { Round() })

	Tween(card, { Position = UDim2.fromOffset(0, 0) }, 0.5, Enum.EasingStyle.Back)
	Tween(progress, { Size = UDim2.new(0, 0, 0, 2) }, duration, Enum.EasingStyle.Linear)
	task.delay(duration, function()
		if not slot.Parent then return end
		Tween(card, { Position = UDim2.fromOffset(330, 0) }, 0.3, Enum.EasingStyle.Quint, Enum.EasingDirection.In)
		task.wait(0.3)
		Tween(slot, { Size = UDim2.new(1, 0, 0, 0) }, 0.2)
		task.wait(0.2)
		slot:Destroy()
	end)
end

local function Confirm(title, message, confirmText, onConfirm)
	local overlay = New("TextButton", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0), BackgroundTransparency = 1, Text = "", ZIndex = 50, Parent = Main }, { Corner(14) })
	local box = New("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, 0.5, 20), Size = UDim2.fromOffset(350, 168), BackgroundColor3 = Theme.Panel, Parent = overlay }, { Corner(14), Stroke() })
	local scale = New("UIScale", { Scale = 0.92, Parent = box })

	local icon = Label({ Text = "!", Font = Enum.Font.GothamBlack, TextSize = 18, TextColor3 = Theme.Danger, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.fromOffset(20, 20), Size = UDim2.fromOffset(36, 36), BackgroundTransparency = 0.82, BackgroundColor3 = Theme.Danger, Parent = box })
	Round().Parent = icon
	Label({ Text = title, Font = Enum.Font.GothamBold, TextSize = 15, Position = UDim2.fromOffset(70, 20), Size = UDim2.new(1, -90, 0, 20), Parent = box })
	Label({ Text = message, Font = Enum.Font.Gotham, TextSize = 12, TextColor3 = Theme.SubText, TextWrapped = true, TextTruncate = Enum.TextTruncate.None, TextYAlignment = Enum.TextYAlignment.Top, Position = UDim2.fromOffset(70, 44), Size = UDim2.new(1, -90, 0, 54), Parent = box })

	local cancel = New("TextButton", { AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -134, 1, -18), Size = UDim2.fromOffset(104, 34), BackgroundColor3 = Theme.Card, Text = "Cancel", Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = Theme.Text, Parent = box }, { Corner(8), Stroke() })
	local confirm = New("TextButton", { AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -18, 1, -18), Size = UDim2.fromOffset(108, 34), BackgroundColor3 = Theme.Danger, Text = confirmText, Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = Color3.new(1, 1, 1), Parent = box }, { Corner(8) })

	Tween(overlay, { BackgroundTransparency = 0.35 }, 0.2)
	Tween(box, { Position = UDim2.fromScale(0.5, 0.5) }, 0.35, Enum.EasingStyle.Back)
	Tween(scale, { Scale = 1 }, 0.35, Enum.EasingStyle.Back)

	local function close()
		Tween(overlay, { BackgroundTransparency = 1 }, 0.2)
		Tween(scale, { Scale = 0.9 }, 0.2)
		task.delay(0.2, function() overlay:Destroy() end)
	end
	cancel.MouseButton1Click:Connect(close)
	confirm.MouseButton1Click:Connect(function()
		close()
		onConfirm()
	end)
end

--==================================================================--
-- COMPONENTS
--==================================================================--

local orderCounter = 0
local function NextOrder()
	orderCounter += 1
	return orderCounter
end

local function Row(parent, height)
	return New("Frame", { Size = UDim2.new(1, 0, 0, height), BackgroundColor3 = Theme.Card, LayoutOrder = NextOrder(), Parent = parent }, { Corner(9) })
end

local function RowText(row, title, desc, rightSpace)
	local height = row.Size.Y.Offset
	local titleLabel = Label({ Text = title, TextSize = 13, Position = UDim2.fromOffset(12, desc and 8 or 0), Size = UDim2.new(1, -(rightSpace + 12), 0, desc and 18 or height), Parent = row })
	if desc then
		Label({ Text = desc, Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(12, 27), Size = UDim2.new(1, -(rightSpace + 12), 0, 14), Parent = row })
	end
	return titleLabel
end

local function RowHover(row, hit)
	hit.MouseEnter:Connect(function() Tween(row, { BackgroundColor3 = Theme.CardHover }, 0.15) end)
	hit.MouseLeave:Connect(function() Tween(row, { BackgroundColor3 = Theme.Card }, 0.15) end)
end

local function Section(page, title, desc)
	local card = New("Frame", { Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundColor3 = Theme.Panel, LayoutOrder = NextOrder(), Parent = page }, {
		Corner(12), Stroke(),
		New("UIPadding", { PaddingTop = UDim.new(0, 14), PaddingBottom = UDim.new(0, 14), PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14) }),
		New("UIListLayout", { Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder }),
	})
	local head = New("Frame", { Size = UDim2.new(1, 0, 0, desc and 36 or 20), BackgroundTransparency = 1, LayoutOrder = 0, Parent = card })
	BindAccent(New("Frame", { Position = UDim2.fromOffset(0, 5), Size = UDim2.fromOffset(8, 8), Parent = head }, { Corner(3) }), "BackgroundColor3")
	Label({ Text = title:upper(), Font = Enum.Font.GothamBold, TextSize = 11, Position = UDim2.fromOffset(16, 0), Size = UDim2.new(1, -16, 0, 18), Parent = head })
	if desc then
		Label({ Text = desc, Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(16, 19), Size = UDim2.new(1, -16, 0, 14), Parent = head })
	end
	return card
end

local function Toggle(parent, title, desc, default, callback)
	local row = Row(parent, desc and 50 or 40)
	RowText(row, title, desc, 70)
	local track = New("Frame", { AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0), Size = UDim2.fromOffset(42, 22), BackgroundColor3 = Theme.Stroke, Parent = row }, { Round() })
	local knob = New("Frame", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 3, 0.5, 0), Size = UDim2.fromOffset(16, 16), BackgroundColor3 = Color3.fromRGB(190, 190, 205), Parent = track }, { Round() })
	local state = default

	local function render(duration)
		Tween(track, { BackgroundColor3 = state and Theme.Accent or Theme.Stroke }, duration)
		Tween(knob, {
			Position = state and UDim2.new(1, -19, 0.5, 0) or UDim2.new(0, 3, 0.5, 0),
			BackgroundColor3 = state and Color3.new(1, 1, 1) or Color3.fromRGB(190, 190, 205),
		}, duration)
	end
	local function set(value, silent)
		state = value
		render(0.25)
		if not silent and callback then callback(state) end
	end

	table.insert(AccentListeners, function(color) if state then track.BackgroundColor3 = color end end)
	local hit = New("TextButton", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "", Parent = row })
	RowHover(row, hit)
	hit.MouseButton1Click:Connect(function() set(not state) end)
	render(0)
	return set
end

local function Slider(parent, title, min, max, default, suffix, callback)
	local row = Row(parent, 58)
	Label({ Text = title, TextSize = 13, Position = UDim2.fromOffset(12, 9), Size = UDim2.new(1, -100, 0, 18), Parent = row })
	local valuePill = Label({ Font = Enum.Font.GothamBold, TextSize = 12, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -12, 0, 8), Size = UDim2.fromOffset(0, 20), AutomaticSize = Enum.AutomaticSize.X, BackgroundTransparency = 0, BackgroundColor3 = Theme.Panel, Parent = row })
	Corner(6).Parent = valuePill
	HPadding(8).Parent = valuePill
	BindAccent(valuePill, "TextColor3")

	local bar = New("Frame", { Position = UDim2.new(0, 12, 0, 40), Size = UDim2.new(1, -24, 0, 6), BackgroundColor3 = Theme.Stroke, Parent = row }, { Round() })
	local fill = New("Frame", { Size = UDim2.fromScale(0, 1), Parent = bar }, { Round() })
	BindAccent(fill, "BackgroundColor3")
	local knob = New("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0, 0.5), Size = UDim2.fromOffset(14, 14), BackgroundColor3 = Color3.new(1, 1, 1), Parent = bar }, { Round() })
	local knobStroke = Stroke(Theme.Accent, 0, 2)
	knobStroke.Parent = knob
	BindAccent(knobStroke, "Color")

	local function set(value, silent)
		value = math.clamp(math.floor(value + 0.5), min, max)
		local alpha = (value - min) / (max - min)
		fill.Size = UDim2.fromScale(alpha, 1)
		knob.Position = UDim2.fromScale(alpha, 0.5)
		valuePill.Text = tostring(value) .. (suffix or "")
		if not silent and callback then callback(value) end
	end

	local dragging = false
	local function fromX(x)
		local alpha = math.clamp((x - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
		set(min + alpha * (max - min))
	end
	local hit = New("TextButton", { Position = UDim2.fromOffset(0, 30), Size = UDim2.new(1, 0, 0, 26), BackgroundTransparency = 1, Text = "", Parent = row })
	RowHover(row, hit)
	hit.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			fromX(input.Position.X)
			Tween(knob, { Size = UDim2.fromOffset(18, 18) }, 0.15)
		end
	end)
	Track(UserInputService.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			fromX(input.Position.X)
		end
	end))
	Track(UserInputService.InputEnded:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch) then
			dragging = false
			Tween(knob, { Size = UDim2.fromOffset(14, 14) }, 0.15)
		end
	end))
	set(default, true)
	return set
end

local function MakeSegment(parent, options, default, callback, segmentWidth, height)
	local holder = New("Frame", { Size = UDim2.fromOffset(#options * segmentWidth + 6, height), BackgroundColor3 = Theme.Panel, Parent = parent }, { Corner(8), Stroke() })
	local pill = New("Frame", { Position = UDim2.fromOffset(3, 3), Size = UDim2.fromOffset(segmentWidth, height - 6), Parent = holder }, { Corner(6) })
	BindAccent(pill, "BackgroundColor3")
	local buttons = {}
	local function set(option, silent)
		for i, name in ipairs(options) do
			local active = name == option
			if active then Tween(pill, { Position = UDim2.fromOffset(3 + (i - 1) * segmentWidth, 3) }, 0.3) end
			Tween(buttons[i], { TextColor3 = active and Color3.new(1, 1, 1) or Theme.SubText }, 0.2)
		end
		if not silent and callback then callback(option) end
	end
	for i, name in ipairs(options) do
		local button = New("TextButton", { Position = UDim2.fromOffset(3 + (i - 1) * segmentWidth, 3), Size = UDim2.fromOffset(segmentWidth, height - 6), BackgroundTransparency = 1, Text = name, Font = Enum.Font.GothamMedium, TextSize = 12, TextColor3 = Theme.SubText, ZIndex = 2, Parent = holder })
		button.MouseButton1Click:Connect(function() set(name) end)
		buttons[i] = button
	end
	set(default, true)
	return holder, set
end

local function Segmented(parent, title, options, default, callback)
	local row = Row(parent, 46)
	RowText(row, title, nil, #options * 64 + 24)
	local holder = MakeSegment(row, options, default, callback, 64, 32)
	holder.AnchorPoint = Vector2.new(1, 0.5)
	holder.Position = UDim2.new(1, -7, 0.5, 0)
end

local function Button(parent, title, desc, callback, danger)
	local row = Row(parent, desc and 50 or 40)
	local titleLabel = RowText(row, title, desc, 40)
	if danger then titleLabel.TextColor3 = Theme.Danger end
	local arrow = Label({ Text = "›", Font = Enum.Font.GothamBold, TextSize = 22, TextXAlignment = Enum.TextXAlignment.Center, TextColor3 = danger and Theme.Danger or Theme.Muted, AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, -1), Size = UDim2.fromOffset(16, 22), Parent = row })
	local hit = New("TextButton", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "", Parent = row })
	RowHover(row, hit)
	hit.MouseEnter:Connect(function() Tween(arrow, { Position = UDim2.new(1, -7, 0.5, -1) }, 0.2) end)
	hit.MouseLeave:Connect(function() Tween(arrow, { Position = UDim2.new(1, -12, 0.5, -1) }, 0.2) end)
	hit.MouseButton1Click:Connect(function()
		row.BackgroundColor3 = Theme.Stroke
		Tween(row, { BackgroundColor3 = Theme.CardHover }, 0.3)
		callback()
	end)
end

local function Keybind(parent, title, desc, default, callback)
	local row = Row(parent, desc and 50 or 40)
	RowText(row, title, desc, 150)
	local current = default
	local button = New("TextButton", { AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0), Size = UDim2.fromOffset(0, 28), AutomaticSize = Enum.AutomaticSize.X, BackgroundColor3 = Theme.Panel, Text = current.Name, Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = Theme.Text, Parent = row }, { Corner(7), Stroke(), HPadding(12) })
	local listening = false
	button.MouseButton1Click:Connect(function()
		if listening then return end
		listening, KeybindListening = true, true
		button.Text = "Press a key..."
		Tween(button, { TextColor3 = Theme.Accent }, 0.15)
	end)
	Track(UserInputService.InputBegan:Connect(function(input)
		if not listening or input.UserInputType ~= Enum.UserInputType.Keyboard then return end
		listening = false
		if input.KeyCode ~= Enum.KeyCode.Escape then
			current = input.KeyCode
			callback(current)
		end
		button.Text = current.Name
		Tween(button, { TextColor3 = Theme.Text }, 0.15)
		task.delay(0.15, function() KeybindListening = false end)
	end))
end

local function Swatches(parent, title, presets, defaultIndex, callback)
	local row = Row(parent, 80)
	local titleLabel = Label({ TextSize = 13, Position = UDim2.fromOffset(12, 10), Size = UDim2.new(1, -24, 0, 18), Parent = row })
	local holder = New("Frame", { Position = UDim2.fromOffset(12, 38), Size = UDim2.new(1, -24, 0, 30), BackgroundTransparency = 1, Parent = row }, {
		New("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, VerticalAlignment = Enum.VerticalAlignment.Center, Padding = UDim.new(0, 12), SortOrder = Enum.SortOrder.LayoutOrder }),
		New("UIPadding", { PaddingLeft = UDim.new(0, 3) }),
	})
	local rings = {}
	local function select(index, silent)
		for i, ring in ipairs(rings) do Tween(ring, { Transparency = i == index and 0 or 1 }, 0.2) end
		titleLabel.Text = ("%s  ·  %s"):format(title, presets[index].Name)
		if not silent then callback(presets[index]) end
	end
	for i, preset in ipairs(presets) do
		local swatch = New("TextButton", { Size = UDim2.fromOffset(26, 26), BackgroundColor3 = preset.Color, Text = "", LayoutOrder = i, Parent = holder }, { Round() })
		rings[i] = Stroke(Color3.new(1, 1, 1), 1, 2)
		rings[i].Parent = swatch
		swatch.MouseButton1Click:Connect(function() select(i) end)
	end
	select(defaultIndex, true)
end

local function StatGrid(page)
	return New("Frame", { Size = UDim2.new(1, 0, 0, 78), BackgroundTransparency = 1, LayoutOrder = NextOrder(), Parent = page }, {
		New("UIGridLayout", { CellSize = UDim2.new(0.25, -8, 1, 0), CellPadding = UDim2.fromOffset(10, 0), SortOrder = Enum.SortOrder.LayoutOrder }),
	})
end

local function StatCard(grid, title, order)
	local card = New("Frame", { BackgroundColor3 = Theme.Panel, LayoutOrder = order, Parent = grid }, { Corner(12), Stroke() })
	BindAccent(New("Frame", { Position = UDim2.fromOffset(14, 16), Size = UDim2.fromOffset(6, 6), Parent = card }, { Round() }), "BackgroundColor3")
	Label({ Text = title, Font = Enum.Font.GothamBold, TextSize = 10, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(26, 10), Size = UDim2.new(1, -30, 0, 18), Parent = card })
	return Label({ Text = "—", Font = Enum.Font.GothamBlack, TextSize = 22, Position = UDim2.fromOffset(14, 34), Size = UDim2.new(1, -28, 0, 30), Parent = card })
end

local function InfoRow(parent, key, value)
	local row = Row(parent, 34)
	Label({ Text = key, TextSize = 12, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(12, 0), Size = UDim2.new(0.5, -12, 1, 0), Parent = row })
	return Label({ Text = value, Font = Enum.Font.GothamBold, TextSize = 12, TextXAlignment = Enum.TextXAlignment.Right, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -12, 0, 0), Size = UDim2.new(0.5, -12, 1, 0), Parent = row })
end

local function HintRow(parent, keyText, text)
	local row = Row(parent, 40)
	local key = Label({ Text = keyText, Font = Enum.Font.GothamBold, TextSize = 11, Position = UDim2.fromOffset(10, 9), Size = UDim2.fromOffset(0, 22), AutomaticSize = Enum.AutomaticSize.X, BackgroundTransparency = 0, BackgroundColor3 = Theme.Panel, Parent = row })
	Corner(6).Parent = key
	Stroke().Parent = key
	HPadding(8).Parent = key
	BindAccent(key, "TextColor3")
	Label({ Text = text, TextSize = 12, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(122, 0), Size = UDim2.new(1, -132, 1, 0), Parent = row })
	return key
end

local function ActionButton(parent, text, props, primary)
	local button = New("TextButton", { BackgroundColor3 = primary and Theme.Accent or Theme.Card, Text = text, Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = Color3.new(1, 1, 1), Parent = parent }, { Corner(8) })
	for key, value in pairs(props) do button[key] = value end
	if primary then
		BindAccent(button, "BackgroundColor3")
		button.MouseEnter:Connect(function() Tween(button, { BackgroundColor3 = Tint(Theme.Accent, 0.15) }, 0.15) end)
		button.MouseLeave:Connect(function() Tween(button, { BackgroundColor3 = Theme.Accent }, 0.15) end)
	else
		Stroke().Parent = button
		button.MouseEnter:Connect(function() Tween(button, { BackgroundColor3 = Theme.CardHover }, 0.15) end)
		button.MouseLeave:Connect(function() Tween(button, { BackgroundColor3 = Theme.Card }, 0.15) end)
	end
	return button
end

--==================================================================--
-- TABS
--==================================================================--

local TabDefs = {
	{ Id = "Home",     Icon = "◈", Name = "Dashboard", Subtitle = "Your session and the server at a glance" },
	{ Id = "Players",  Icon = "◉", Name = "Players",   Subtitle = "Everyone in this server, updated live" },
	{ Id = "PvP",      Icon = "✦", Name = "PvP Tools", Subtitle = "Crosshair, kill feed and combat HUD" },
	{ Id = "Visuals",  Icon = "◐", Name = "Visuals",   Subtitle = "Camera, overlays and effects" },
	{ Id = "Settings", Icon = "⚙", Name = "Settings",  Subtitle = "Make mxme pvp yours" },
}
local Pages, NavButtons = {}, {}

local function CreateScrollPage(id)
	return New("ScrollingFrame", {
		Name = id, Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false,
		ScrollBarThickness = 3, ScrollBarImageColor3 = Theme.Stroke, CanvasSize = UDim2.new(),
		AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollingDirection = Enum.ScrollingDirection.Y, Parent = PageHolder,
	}, {
		New("UIListLayout", { Padding = UDim.new(0, 12), SortOrder = Enum.SortOrder.LayoutOrder }),
		New("UIPadding", { PaddingTop = UDim.new(0, 18), PaddingBottom = UDim.new(0, 22), PaddingLeft = UDim.new(0, 24), PaddingRight = UDim.new(0, 20) }),
	})
end

local function SelectTab(id)
	if CurrentTab == id then return end
	for _, def in ipairs(TabDefs) do
		local nav = NavButtons[def.Id]
		local active = def.Id == id
		Tween(nav.Text, { TextColor3 = active and Theme.Text or Theme.SubText }, 0.2)
		Tween(nav.Icon, { TextColor3 = active and Theme.Accent or Theme.Muted }, 0.2)
		if active then
			Tween(NavIndicator, { Position = UDim2.fromOffset(0, nav.Button.Position.Y.Offset) }, 0.35, Enum.EasingStyle.Quint)
			PageTitle.Text = def.Name
			PageSubtitle.Text = def.Subtitle
		end
	end
	if CurrentTab then Pages[CurrentTab].Visible = false end
	local page = Pages[id]
	page.Visible = true
	page.Position = UDim2.fromOffset(0, 16)
	Tween(page, { Position = UDim2.fromOffset(0, 0) }, 0.35)
	CurrentTab = id
end

for i, def in ipairs(TabDefs) do
	local button = New("TextButton", { Position = UDim2.fromOffset(0, NAV_TOP + (i - 1) * NAV_STEP), Size = UDim2.new(1, 0, 0, 38), BackgroundTransparency = 1, Text = "", ZIndex = 2, Parent = NavHolder })
	local icon = Label({ Text = def.Icon, Font = Enum.Font.GothamBold, TextSize = 15, TextColor3 = Theme.Muted, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.fromOffset(12, 0), Size = UDim2.new(0, 20, 1, 0), Parent = button })
	local text = Label({ Text = def.Name, TextSize = 13, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(42, 0), Size = UDim2.new(1, -42, 1, 0), Parent = button })
	NavButtons[def.Id] = { Button = button, Icon = icon, Text = text }

	button.MouseEnter:Connect(function() if CurrentTab ~= def.Id then Tween(text, { TextColor3 = Theme.Text }, 0.15) end end)
	button.MouseLeave:Connect(function() if CurrentTab ~= def.Id then Tween(text, { TextColor3 = Theme.SubText }, 0.15) end end)
	button.MouseButton1Click:Connect(function() SelectTab(def.Id) end)

	if def.Id == "Players" then
		Pages[def.Id] = New("Frame", { Name = def.Id, Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, Parent = PageHolder })
	else
		Pages[def.Id] = CreateScrollPage(def.Id)
	end
end
OnAccent(function(color)
	if CurrentTab then NavButtons[CurrentTab].Icon.TextColor3 = color end
end)

--==================================================================--
-- HUD: CROSSHAIR
--==================================================================--

local Crosshair = New("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(2, 2), BackgroundTransparency = 1, Parent = HudGui })
local CrossParts = {}

local function BuildCrosshair()
	Crosshair:ClearAllChildren()
	table.clear(CrossParts)
	Crosshair.Visible = Settings.Crosshair
	if not Settings.Crosshair then return end

	local color = Settings.CrossAccent and Theme.Accent or Color3.new(1, 1, 1)
	local size, thick, gap, style = Settings.CrossSize, Settings.CrossThickness, Settings.CrossGap, Settings.CrossStyle

	local function bar(x, y, w, h, rounded)
		local part = New("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, x, 0.5, y), Size = UDim2.fromOffset(w, h), BackgroundColor3 = color, Parent = Crosshair })
		if rounded then Round().Parent = part end
		if Settings.CrossOutline then Stroke(Color3.new(0, 0, 0), 0.35, 1).Parent = part end
		table.insert(CrossParts, { part, "BackgroundColor3" })
	end

	if style == "Cross" or style == "Hybrid" then
		local offset = gap + size / 2
		bar(0, -offset, thick, size)
		bar(0, offset, thick, size)
		bar(-offset, 0, size, thick)
		bar(offset, 0, size, thick)
	end
	if style == "Dot" or style == "Hybrid" then
		bar(0, 0, thick + 2, thick + 2, true)
	end
	if style == "Circle" then
		local diameter = (gap + size) * 2
		local ring = New("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(diameter, diameter), BackgroundTransparency = 1, Parent = Crosshair }, { Round() })
		local ringStroke = New("UIStroke", { Color = color, Thickness = thick, Parent = ring })
		table.insert(CrossParts, { ringStroke, "Color" })
		bar(0, 0, 2, 2, true)
	end
end
OnAccent(function(color)
	if Settings.CrossAccent then
		for _, part in ipairs(CrossParts) do part[1][part[2]] = color end
	end
end)

--==================================================================--
-- HUD: LOW HEALTH VIGNETTE
--==================================================================--

local VignetteEdges = {}
local function VignetteEdge(position, size, rotation)
	local edge = New("Frame", { Position = position, Size = size, BackgroundColor3 = Theme.Danger, BackgroundTransparency = 1, Parent = HudGui })
	New("UIGradient", { Rotation = rotation, Transparency = NumberSequence.new(0, 1), Parent = edge })
	table.insert(VignetteEdges, edge)
end
VignetteEdge(UDim2.fromScale(0, 0), UDim2.fromScale(0.22, 1), 0)
VignetteEdge(UDim2.fromScale(0.78, 0), UDim2.fromScale(0.22, 1), 180)
VignetteEdge(UDim2.fromScale(0, 0), UDim2.fromScale(1, 0.25), 90)
VignetteEdge(UDim2.fromScale(0, 0.75), UDim2.fromScale(1, 0.25), 270)

local DamageFlash, VignetteAlpha = 0, 0
local function UpdateVignette(dt)
	local target = 0
	local humanoid = GetHumanoid(LocalPlayer)
	if Settings.LowHpWarning and humanoid and humanoid.Health > 0 and humanoid.MaxHealth > 0
		and humanoid.Health / humanoid.MaxHealth * 100 <= Settings.LowHpThreshold then
		target = 0.35 + math.sin(os.clock() * 5) * 0.15
	end
	DamageFlash = math.max(DamageFlash - dt * 1.6, 0)
	target = math.max(target, DamageFlash)
	VignetteAlpha += (target - VignetteAlpha) * math.min(dt * 12, 1)
	for _, edge in ipairs(VignetteEdges) do edge.BackgroundTransparency = 1 - VignetteAlpha end
end

--==================================================================--
-- HUD: WATERMARK
--==================================================================--

local Watermark = New("Frame", { AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 16, 1, -16), Size = UDim2.fromOffset(0, 32), AutomaticSize = Enum.AutomaticSize.X, BackgroundColor3 = Theme.Panel, BackgroundTransparency = 0.08, Parent = Gui }, { Corner(9), Stroke(), HPadding(14) })
local WatermarkText = Label({ RichText = true, TextSize = 12, AutomaticSize = Enum.AutomaticSize.X, Size = UDim2.fromOffset(0, 32), Parent = Watermark })

--==================================================================--
-- HUD: KILL FEED
--==================================================================--

local KillFeed = New("Frame", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -16, 0.28, 0), Size = UDim2.fromOffset(340, 220), BackgroundTransparency = 1, Parent = Gui }, {
	New("UIListLayout", { HorizontalAlignment = Enum.HorizontalAlignment.Right, Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }),
})
local feedOrder = 0

local function NameColor(player)
	return player.Team and player.TeamColor.Color or Theme.Text
end

local function AddKillFeed(killer, victim)
	if not Settings.KillFeed then return end
	feedOrder += 1
	local involvesMe = killer == LocalPlayer or victim == LocalPlayer
	local text
	if killer then
		text = ('<font color="#%s"><b>%s</b></font>   <font color="#%s">⚔</font>   <font color="#%s"><b>%s</b></font>'):format(
			NameColor(killer):ToHex(), Escape(killer.DisplayName), Theme.Accent:ToHex(), NameColor(victim):ToHex(), Escape(victim.DisplayName))
	else
		text = ('<font color="#%s"><b>%s</b></font>  <font color="#%s">was eliminated</font>'):format(
			NameColor(victim):ToHex(), Escape(victim.DisplayName), Theme.SubText:ToHex())
	end

	local entry = Label({ RichText = true, Text = text, TextSize = 13, AutomaticSize = Enum.AutomaticSize.X, Size = UDim2.fromOffset(0, 30), BackgroundColor3 = Theme.Panel, BackgroundTransparency = 1, TextTransparency = 1, LayoutOrder = feedOrder, Parent = KillFeed })
	Corner(8).Parent = entry
	HPadding(12).Parent = entry
	local entryStroke = Stroke(involvesMe and Theme.Accent or Theme.Stroke, involvesMe and 0.2 or 0)
	entryStroke.Parent = entry
	Tween(entry, { TextTransparency = 0, BackgroundTransparency = 0.08 }, 0.3)

	local entries = {}
	for _, child in ipairs(KillFeed:GetChildren()) do
		if child:IsA("TextLabel") then table.insert(entries, child) end
	end
	if #entries > 5 then
		table.sort(entries, function(a, b) return a.LayoutOrder < b.LayoutOrder end)
		entries[1]:Destroy()
	end

	task.delay(6, function()
		if not entry.Parent then return end
		Tween(entry, { TextTransparency = 1, BackgroundTransparency = 1 }, 0.4)
		Tween(entryStroke, { Transparency = 1 }, 0.4)
		task.wait(0.4)
		entry:Destroy()
	end)
end

--==================================================================--
-- HUD: TARGET INFO (player under your mouse)
--==================================================================--

local TargetHud = New("Frame", { AnchorPoint = Vector2.new(0.5, 1), Position = UDim2.new(0.5, 0, 1, -130), Size = UDim2.fromOffset(280, 64), BackgroundColor3 = Theme.Panel, BackgroundTransparency = 0.05, Visible = false, Parent = Gui }, { Corner(12), Stroke() })
local TargetAvatar = New("ImageLabel", { Position = UDim2.fromOffset(10, 10), Size = UDim2.fromOffset(44, 44), BackgroundColor3 = Theme.Card, Parent = TargetHud }, { Round() })
local TargetName = Label({ Font = Enum.Font.GothamBold, TextSize = 14, Position = UDim2.fromOffset(64, 10), Size = UDim2.new(1, -150, 0, 18), Parent = TargetHud })
local TargetDistance = Label({ TextSize = 11, TextColor3 = Theme.SubText, TextXAlignment = Enum.TextXAlignment.Right, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -12, 0, 12), Size = UDim2.fromOffset(80, 14), Parent = TargetHud })
local TargetBarBg = New("Frame", { Position = UDim2.fromOffset(64, 38), Size = UDim2.new(1, -140, 0, 6), BackgroundColor3 = Theme.Stroke, Parent = TargetHud }, { Round() })
local TargetBar = New("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = Theme.Success, Parent = TargetBarBg }, { Round() })
local TargetHp = Label({ Font = Enum.Font.GothamBold, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Right, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -12, 0, 34), Size = UDim2.fromOffset(62, 14), Parent = TargetHud })
local CurrentTarget = nil

local function UpdateTargetHud()
	local target = Settings.TargetHud and PlayerFromPart(Mouse.Target) or nil
	if target == LocalPlayer then target = nil end
	local humanoid = GetHumanoid(target)
	if not humanoid then
		TargetHud.Visible = false
		CurrentTarget = nil
		return
	end
	if target ~= CurrentTarget then
		CurrentTarget = target
		TargetAvatar.Image = Headshot(target.UserId)
		TargetName.Text = target.DisplayName
	end
	local ratio = humanoid.MaxHealth > 0 and math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1) or 0
	Tween(TargetBar, { Size = UDim2.fromScale(ratio, 1), BackgroundColor3 = HealthColor(ratio) }, 0.15)
	TargetHp.Text = math.floor(humanoid.Health + 0.5) .. " HP"
	local distance = DistanceTo(target)
	TargetDistance.Text = distance and (math.floor(distance) .. " studs") or ""
	TargetHud.Visible = true
end

--==================================================================--
-- SPECTATE
--==================================================================--

local SpectateTarget = nil
local SpectateBar = New("Frame", { AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, -60), Size = UDim2.fromOffset(0, 42), AutomaticSize = Enum.AutomaticSize.X, BackgroundColor3 = Theme.Panel, Visible = false, Parent = Gui }, {
	Corner(12), Stroke(), HPadding(14, 7),
	New("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, VerticalAlignment = Enum.VerticalAlignment.Center, Padding = UDim.new(0, 10), SortOrder = Enum.SortOrder.LayoutOrder }),
})
local SpectateDot = New("Frame", { Size = UDim2.fromOffset(8, 8), BackgroundColor3 = Theme.Danger, LayoutOrder = 1, Parent = SpectateBar }, { Round() })
Label({ Text = "SPECTATING", Font = Enum.Font.GothamBold, TextSize = 11, TextColor3 = Theme.SubText, AutomaticSize = Enum.AutomaticSize.X, Size = UDim2.fromOffset(0, 42), LayoutOrder = 2, Parent = SpectateBar })
local SpectateName = Label({ Font = Enum.Font.GothamBold, TextSize = 14, AutomaticSize = Enum.AutomaticSize.X, Size = UDim2.fromOffset(0, 42), LayoutOrder = 3, Parent = SpectateBar })
local SpectateStop = New("TextButton", { Size = UDim2.fromOffset(64, 28), BackgroundColor3 = Theme.Danger, Text = "Stop", Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = Color3.new(1, 1, 1), LayoutOrder = 4, Parent = SpectateBar }, { Corner(8) })

local function StopSpectate()
	if not SpectateTarget then return end
	SpectateTarget = nil
	local humanoid = GetHumanoid(LocalPlayer)
	if humanoid then workspace.CurrentCamera.CameraSubject = humanoid end
	Tween(SpectateBar, { Position = UDim2.new(0.5, 0, 0, -60) }, 0.3, Enum.EasingStyle.Quint, Enum.EasingDirection.In)
	task.delay(0.3, function() if not SpectateTarget then SpectateBar.Visible = false end end)
end

local function StartSpectate(player)
	if player == LocalPlayer then
		Notify("Spectate", "You can't spectate yourself.", "warning")
		return
	end
	local humanoid = GetHumanoid(player)
	if not humanoid then
		Notify("Spectate", player.DisplayName .. " has no character right now.", "warning")
		return
	end
	SpectateTarget = player
	workspace.CurrentCamera.CameraSubject = humanoid
	SpectateName.Text = player.DisplayName
	SpectateBar.Visible = true
	Tween(SpectateBar, { Position = UDim2.new(0.5, 0, 0, 14) }, 0.45, Enum.EasingStyle.Back)
	Notify("Spectating", "Now watching " .. player.DisplayName, "info")
end
SpectateStop.MouseButton1Click:Connect(StopSpectate)

--==================================================================--
-- MENU VISIBILITY & UNLOAD
--==================================================================--

local function SetMenu(open)
	MenuOpen = open
	if open then
		Holder.Visible = true
		WindowScale.Scale = Settings.UIScale * 0.92
		Tween(WindowScale, { Scale = Settings.UIScale }, 0.45, Enum.EasingStyle.Back)
		if Settings.MenuBlur then Tween(Blur, { Size = 14 }, 0.4) end
	else
		Tween(Blur, { Size = 0 }, 0.3)
		local tween = Tween(WindowScale, { Scale = Settings.UIScale * 0.88 }, 0.2, Enum.EasingStyle.Quint, Enum.EasingDirection.In)
		tween.Completed:Connect(function(state)
			if state == Enum.PlaybackState.Completed and not MenuOpen then Holder.Visible = false end
		end)
	end
end

local function Unload()
	if not Running then return end
	StopSpectate()
	Running = false
	for _, connection in ipairs(Conns) do connection:Disconnect() end
	Tween(Blur, { Size = 0 }, 0.25)
	Tween(WindowScale, { Scale = Settings.UIScale * 0.85 }, 0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.In)
	task.delay(0.3, function()
		Blur:Destroy()
		Gui:Destroy()
		HudGui:Destroy()
	end)
end

--==================================================================--
-- PAGE: DASHBOARD
--==================================================================--

local HomePage = Pages.Home
local KeyHints = {}

local Hero = New("Frame", { Size = UDim2.new(1, 0, 0, 112), BackgroundColor3 = Color3.new(1, 1, 1), LayoutOrder = NextOrder(), Parent = HomePage }, { Corner(14) })
local HeroGradient = New("UIGradient", { Rotation = 20, Parent = Hero })
OnAccent(function(c)
	HeroGradient.Color = ColorSequence.new({ ColorSequenceKeypoint.new(0, Shade(c, 0.3)), ColorSequenceKeypoint.new(1, Theme.Panel) })
end)
local heroStroke = Stroke(Theme.Accent, 0.6)
heroStroke.Parent = Hero
BindAccent(heroStroke, "Color")
New("ImageLabel", { Position = UDim2.fromOffset(20, 21), Size = UDim2.fromOffset(70, 70), BackgroundColor3 = Theme.Card, Image = Headshot(LocalPlayer.UserId), Parent = Hero }, { Round(), Stroke(Color3.new(1, 1, 1), 0.7, 2) })
Label({ Text = "WELCOME BACK", Font = Enum.Font.GothamBold, TextSize = 11, TextColor3 = Color3.new(1, 1, 1), TextTransparency = 0.35, Position = UDim2.fromOffset(106, 24), Size = UDim2.new(1, -200, 0, 14), Parent = Hero })
Label({ Text = LocalPlayer.DisplayName, Font = Enum.Font.GothamBlack, TextSize = 24, Position = UDim2.fromOffset(106, 40), Size = UDim2.new(1, -200, 0, 28), Parent = Hero })
Label({
	Text = ("@%s  ·  %d days on Roblox%s"):format(LocalPlayer.Name, LocalPlayer.AccountAge, LocalPlayer.MembershipType == Enum.MembershipType.Premium and "  ·  Premium" or ""),
	Font = Enum.Font.Gotham, TextSize = 12, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(106, 70), Size = UDim2.new(1, -126, 0, 16), Parent = Hero,
})
local versionPill = Label({ Text = "v" .. VERSION, Font = Enum.Font.GothamBold, TextSize = 11, TextXAlignment = Enum.TextXAlignment.Center, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -16, 0, 16), Size = UDim2.fromOffset(52, 22), BackgroundTransparency = 0.85, BackgroundColor3 = Color3.new(1, 1, 1), Parent = Hero })
Corner(11).Parent = versionPill

local homeGrid = StatGrid(HomePage)
local HomeStats = {
	Players = StatCard(homeGrid, "PLAYERS", 1),
	FPS     = StatCard(homeGrid, "FPS", 2),
	Ping    = StatCard(homeGrid, "PING", 3),
	Session = StatCard(homeGrid, "SESSION", 4),
}

local serverSection = Section(HomePage, "Server", "Live information about the current server")
InfoRow(serverSection, "Place ID", tostring(game.PlaceId))
InfoRow(serverSection, "Server ID", game.JobId ~= "" and game.JobId:sub(1, 8):upper() or "Studio")
local UptimeValue = InfoRow(serverSection, "Server Uptime", "—")
local TeamValue = InfoRow(serverSection, "Your Team", "—")
InfoRow(serverSection, "Max Players", tostring(Players.MaxPlayers))

local quickSection = Section(HomePage, "Quick Start", "Everything you need to know in 10 seconds")
table.insert(KeyHints, HintRow(quickSection, Settings.MenuKey.Name, "Show or hide the menu"))
HintRow(quickSection, "DRAG", "Move the window from the top bar")
HintRow(quickSection, "CLICK", "Open a full profile from the Players tab")

--==================================================================--
-- PAGE: PLAYERS
--==================================================================--

local PlayersPage = Pages.Players
local PlayerRows = {}
local SortMode = "Name"
local SearchQuery = ""
local DrawerPlayer = nil

local SearchFrame = New("Frame", { Position = UDim2.fromOffset(24, 16), Size = UDim2.new(1, -272, 0, 36), BackgroundColor3 = Theme.Panel, Parent = PlayersPage }, { Corner(9) })
local SearchStroke = Stroke()
SearchStroke.Parent = SearchFrame
local lens = New("Frame", { Position = UDim2.fromOffset(13, 11), Size = UDim2.fromOffset(11, 11), BackgroundTransparency = 1, Parent = SearchFrame }, { Round(), Stroke(Theme.Muted, 0, 2) })
New("Frame", { Position = UDim2.fromOffset(10, 11), Size = UDim2.fromOffset(2, 6), Rotation = -45, BackgroundColor3 = Theme.Muted, Parent = lens }, { Round() })
local SearchBox = New("TextBox", { Position = UDim2.fromOffset(36, 0), Size = UDim2.new(1, -46, 1, 0), BackgroundTransparency = 1, Text = "", PlaceholderText = "Search by name or display name...", PlaceholderColor3 = Theme.Muted, TextColor3 = Theme.Text, Font = Enum.Font.GothamMedium, TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left, ClearTextOnFocus = false, Parent = SearchFrame })
SearchBox.Focused:Connect(function() Tween(SearchStroke, { Color = Theme.Accent }, 0.2) end)
SearchBox.FocusLost:Connect(function() Tween(SearchStroke, { Color = Theme.Stroke }, 0.2) end)

Label({ Text = "IN THIS SERVER", Font = Enum.Font.GothamBold, TextSize = 10, TextColor3 = Theme.Muted, Position = UDim2.fromOffset(24, 66), Size = UDim2.fromOffset(200, 14), Parent = PlayersPage })
local CountBadge = Label({ Text = "0", Font = Enum.Font.GothamBold, TextSize = 11, TextColor3 = Color3.new(1, 1, 1), AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -20, 0, 62), Size = UDim2.fromOffset(0, 20), AutomaticSize = Enum.AutomaticSize.X, BackgroundTransparency = 0, Parent = PlayersPage })
Corner(6).Parent = CountBadge
HPadding(8).Parent = CountBadge
BindAccent(CountBadge, "BackgroundColor3")

local PlayerList = New("ScrollingFrame", { Position = UDim2.fromOffset(24, 90), Size = UDim2.new(1, -44, 1, -102), BackgroundTransparency = 1, ScrollBarThickness = 3, ScrollBarImageColor3 = Theme.Stroke, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollingDirection = Enum.ScrollingDirection.Y, Parent = PlayersPage }, {
	New("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }),
	New("UIPadding", { PaddingRight = UDim.new(0, 8), PaddingBottom = UDim.new(0, 4) }),
})
local EmptyState = Label({ Text = "No players match your search.", TextColor3 = Theme.Muted, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.fromOffset(24, 130), Size = UDim2.new(1, -44, 0, 20), Visible = false, Parent = PlayersPage })

-- Profile drawer -----------------------------------------------------
local DrawerShade = New("TextButton", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0), BackgroundTransparency = 1, Text = "", Visible = false, ZIndex = 10, Parent = PlayersPage }, { Corner(14) })
local Drawer = New("Frame", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, 300, 0, 0), Size = UDim2.new(0, 280, 1, 0), BackgroundColor3 = Theme.Panel, ZIndex = 11, Parent = PlayersPage }, { Corner(14), Stroke() })
local DrawerBanner = New("Frame", { Position = UDim2.fromOffset(8, 8), Size = UDim2.new(1, -16, 0, 64), BackgroundColor3 = Color3.new(1, 1, 1), Parent = Drawer }, { Corner(11) })
local DrawerBannerGradient = New("UIGradient", { Rotation = 25, Parent = DrawerBanner })
OnAccent(function(c) DrawerBannerGradient.Color = ColorSequence.new(Shade(c, 0.1), Shade(c, 0.6)) end)
local DrawerClose = New("TextButton", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -8, 0, 8), Size = UDim2.fromOffset(26, 26), BackgroundColor3 = Color3.new(0, 0, 0), BackgroundTransparency = 0.6, Text = "✕", Font = Enum.Font.GothamBold, TextSize = 12, TextColor3 = Color3.new(1, 1, 1), Parent = DrawerBanner }, { Round() })
local DrawerAvatar = New("ImageLabel", { AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 36), Size = UDim2.fromOffset(72, 72), BackgroundColor3 = Theme.Card, Parent = Drawer }, { Round(), Stroke(Theme.Panel, 0, 4) })
local DrawerName = Label({ Font = Enum.Font.GothamBold, TextSize = 16, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.fromOffset(16, 114), Size = UDim2.new(1, -32, 0, 20), Parent = Drawer })
local DrawerUser = Label({ Font = Enum.Font.Gotham, TextSize = 12, TextColor3 = Theme.SubText, TextXAlignment = Enum.TextXAlignment.Center, Position = UDim2.fromOffset(16, 134), Size = UDim2.new(1, -32, 0, 14), Parent = Drawer })

local function DrawerInfo(key, y)
	New("Frame", { Position = UDim2.new(0, 16, 0, y + 21), Size = UDim2.new(1, -32, 0, 1), BackgroundColor3 = Theme.Stroke, BackgroundTransparency = 0.4, Parent = Drawer })
	Label({ Text = key, TextSize = 12, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(16, y), Size = UDim2.new(0.45, -16, 0, 20), Parent = Drawer })
	return Label({ Text = "—", Font = Enum.Font.GothamBold, TextSize = 12, TextXAlignment = Enum.TextXAlignment.Right, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -16, 0, y), Size = UDim2.new(0.55, -16, 0, 20), Parent = Drawer })
end
local DrawerFields = {
	UserId   = DrawerInfo("User ID", 162),
	Age      = DrawerInfo("Account Age", 184),
	Team     = DrawerInfo("Team", 206),
	Health   = DrawerInfo("Health", 228),
	Distance = DrawerInfo("Distance", 250),
	Status   = DrawerInfo("Status", 272),
}
local SpectateButton = ActionButton(Drawer, "Spectate", { Position = UDim2.fromOffset(16, 304), Size = UDim2.new(1, -32, 0, 34) }, true)
local AvatarButton = ActionButton(Drawer, "View Avatar", { Position = UDim2.fromOffset(16, 346), Size = UDim2.new(0.5, -20, 0, 34) })
local FriendButton = ActionButton(Drawer, "Add Friend", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -16, 0, 346), Size = UDim2.new(0.5, -20, 0, 34) })

local function UpdateDrawer()
	local player = DrawerPlayer
	if not player then return end
	local humanoid = GetHumanoid(player)
	local hp, maxHp = HealthOf(player)
	local alive = humanoid ~= nil and hp > 0
	local distance = DistanceTo(player)
	DrawerFields.Health.Text = humanoid and ("%d / %d"):format(math.floor(hp + 0.5), math.floor(maxHp + 0.5)) or "—"
	DrawerFields.Distance.Text = player == LocalPlayer and "That's you" or (distance and (math.floor(distance) .. " studs") or "—")
	DrawerFields.Team.Text = player.Team and player.Team.Name or "None"
	DrawerFields.Team.TextColor3 = player.Team and player.TeamColor.Color or Theme.Text
	DrawerFields.Status.Text = alive and "Alive" or "Dead"
	DrawerFields.Status.TextColor3 = alive and Theme.Success or Theme.Danger
	SpectateButton.Text = SpectateTarget == player and "Stop Spectating" or "Spectate"
end

local function OpenDrawer(player)
	DrawerPlayer = player
	DrawerAvatar.Image = Headshot(player.UserId)
	DrawerName.Text = player.DisplayName
	DrawerUser.Text = "@" .. player.Name
	DrawerFields.UserId.Text = tostring(player.UserId)
	DrawerFields.Age.Text = player.AccountAge .. " days"
	UpdateDrawer()
	DrawerShade.Visible = true
	Tween(DrawerShade, { BackgroundTransparency = 0.45 }, 0.25)
	Tween(Drawer, { Position = UDim2.new(1, 0, 0, 0) }, 0.4)
end

local function CloseDrawer()
	DrawerPlayer = nil
	Tween(DrawerShade, { BackgroundTransparency = 1 }, 0.25)
	Tween(Drawer, { Position = UDim2.new(1, 300, 0, 0) }, 0.3, Enum.EasingStyle.Quint, Enum.EasingDirection.In)
	task.delay(0.25, function() if not DrawerPlayer then DrawerShade.Visible = false end end)
end

DrawerShade.MouseButton1Click:Connect(CloseDrawer)
DrawerClose.MouseButton1Click:Connect(CloseDrawer)
SpectateButton.MouseButton1Click:Connect(function()
	if not DrawerPlayer then return end
	if SpectateTarget == DrawerPlayer then StopSpectate() else StartSpectate(DrawerPlayer) end
	UpdateDrawer()
end)
AvatarButton.MouseButton1Click:Connect(function()
	if not DrawerPlayer then return end
	local ok = pcall(function() GuiService:InspectPlayerFromUserId(DrawerPlayer.UserId) end)
	if not ok then Notify("View Avatar", "Avatar inspect is not available here.", "warning") end
end)
FriendButton.MouseButton1Click:Connect(function()
	if not DrawerPlayer then return end
	if DrawerPlayer == LocalPlayer then
		Notify("Add Friend", "You can't befriend yourself... yet.", "warning")
		return
	end
	local ok = pcall(function() StarterGui:SetCore("PromptSendFriendRequest", DrawerPlayer) end)
	if not ok then Notify("Add Friend", "Friend requests are not available here.", "warning") end
end)

-- Player rows --------------------------------------------------------
local function CreatePlayerRow(player)
	local isMe = player == LocalPlayer
	local row = New("TextButton", { Size = UDim2.new(1, 0, 0, 58), BackgroundColor3 = Theme.Panel, Text = "", LayoutOrder = isMe and 0 or 1, Parent = PlayerList }, { Corner(10) })
	local rowStroke = Stroke(Theme.Stroke, isMe and 0.45 or 0)
	rowStroke.Parent = row
	if isMe then BindAccent(rowStroke, "Color") end

	local avatar = New("ImageLabel", { Position = UDim2.fromOffset(10, 9), Size = UDim2.fromOffset(40, 40), BackgroundColor3 = Theme.Card, Image = Headshot(player.UserId), Parent = row }, { Round() })
	local ring = Stroke(Theme.Success, 0, 2)
	ring.Parent = avatar

	Label({ Text = player.DisplayName, Font = Enum.Font.GothamBold, TextSize = 13, Position = UDim2.fromOffset(62, 11), Size = UDim2.new(1, -350, 0, 18), Parent = row })
	Label({ Text = "@" .. player.Name .. (isMe and "  ·  You" or ""), Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(62, 30), Size = UDim2.new(1, -350, 0, 14), Parent = row })

	local teamPill = Label({ Font = Enum.Font.GothamBold, TextSize = 10, AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -200, 0.5, 0), Size = UDim2.fromOffset(0, 20), AutomaticSize = Enum.AutomaticSize.X, BackgroundTransparency = 0.8, Parent = row })
	Corner(6).Parent = teamPill
	HPadding(8).Parent = teamPill

	local hpText = Label({ Font = Enum.Font.GothamBold, TextSize = 11, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -100, 0, 13), Size = UDim2.fromOffset(84, 14), Parent = row })
	local hpBg = New("Frame", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -100, 0, 33), Size = UDim2.fromOffset(84, 5), BackgroundColor3 = Theme.Stroke, Parent = row }, { Round() })
	local hpFill = New("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = Theme.Success, Parent = hpBg }, { Round() })
	local distance = Label({ Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = Theme.SubText, TextXAlignment = Enum.TextXAlignment.Right, AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -30, 0.5, 0), Size = UDim2.fromOffset(60, 14), Parent = row })
	local chevron = Label({ Text = "›", Font = Enum.Font.GothamBold, TextSize = 20, TextColor3 = Theme.Muted, TextXAlignment = Enum.TextXAlignment.Center, AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -8, 0.5, -1), Size = UDim2.fromOffset(16, 20), Parent = row })

	row.MouseEnter:Connect(function()
		Tween(row, { BackgroundColor3 = Theme.Card }, 0.15)
		Tween(chevron, { Position = UDim2.new(1, -4, 0.5, -1), TextColor3 = Theme.Accent }, 0.2)
	end)
	row.MouseLeave:Connect(function()
		Tween(row, { BackgroundColor3 = Theme.Panel }, 0.15)
		Tween(chevron, { Position = UDim2.new(1, -8, 0.5, -1), TextColor3 = Theme.Muted }, 0.2)
	end)
	row.MouseButton1Click:Connect(function() OpenDrawer(player) end)

	PlayerRows[player] = { Row = row, Ring = ring, TeamPill = teamPill, HpText = hpText, HpFill = hpFill, Distance = distance }
end

local function RemovePlayerRow(player)
	local data = PlayerRows[player]
	if data then
		data.Row:Destroy()
		PlayerRows[player] = nil
	end
end

local function RefreshPlayerList()
	local query = SearchQuery:lower()
	local others, visible = {}, 0
	for player, r in pairs(PlayerRows) do
		local hp, maxHp = HealthOf(player)
		local ratio = maxHp > 0 and math.clamp(hp / maxHp, 0, 1) or 0
		local alive = GetHumanoid(player) ~= nil and hp > 0
		local distance = DistanceTo(player)

		r.HpText.Text = alive and ("%d / %d HP"):format(math.floor(hp + 0.5), math.floor(maxHp + 0.5)) or "Dead"
		r.HpText.TextColor3 = alive and Theme.Text or Theme.Danger
		Tween(r.HpFill, { Size = UDim2.fromScale(alive and ratio or 0, 1), BackgroundColor3 = HealthColor(ratio) }, 0.2)
		r.Ring.Color = alive and Theme.Success or Theme.Danger
		r.Distance.Text = player == LocalPlayer and "—" or (distance and (math.floor(distance) .. " st") or "—")

		if player.Team then
			r.TeamPill.Text = player.Team.Name:upper():sub(1, 12)
			r.TeamPill.BackgroundColor3 = player.TeamColor.Color
			r.TeamPill.TextColor3 = Tint(player.TeamColor.Color, 0.45)
		else
			r.TeamPill.Text = "NO TEAM"
			r.TeamPill.BackgroundColor3 = Theme.Muted
			r.TeamPill.TextColor3 = Theme.SubText
		end

		local match = query == "" or player.Name:lower():find(query, 1, true) ~= nil or player.DisplayName:lower():find(query, 1, true) ~= nil
		r.Row.Visible = match
		if match then visible += 1 end
		if player ~= LocalPlayer then
			table.insert(others, { Player = player, Hp = alive and hp or -1, Distance = distance or math.huge })
		end
	end

	table.sort(others, function(a, b)
		if SortMode == "Distance" then return a.Distance < b.Distance end
		if SortMode == "Health" then return a.Hp > b.Hp end
		return a.Player.DisplayName:lower() < b.Player.DisplayName:lower()
	end)
	for i, entry in ipairs(others) do PlayerRows[entry.Player].Row.LayoutOrder = i end

	CountBadge.Text = ("%d / %d"):format(#Players:GetPlayers(), Players.MaxPlayers)
	EmptyState.Visible = visible == 0
end

local sortHolder = MakeSegment(PlayersPage, { "Name", "Distance", "Health" }, "Name", function(mode)
	SortMode = mode
	RefreshPlayerList()
end, 72, 36)
sortHolder.AnchorPoint = Vector2.new(1, 0)
sortHolder.Position = UDim2.new(1, -20, 0, 16)

SearchBox:GetPropertyChangedSignal("Text"):Connect(function()
	SearchQuery = SearchBox.Text
	RefreshPlayerList()
end)

--==================================================================--
-- PAGE: PVP TOOLS
--==================================================================--

local PvpPage = Pages.PvP
local pvpGrid = StatGrid(PvpPage)
local PvpStats = {
	Kills  = StatCard(pvpGrid, "KILLS", 1),
	Deaths = StatCard(pvpGrid, "DEATHS", 2),
	KD     = StatCard(pvpGrid, "K/D RATIO", 3),
	Best   = StatCard(pvpGrid, "BEST STREAK", 4),
}

local function UpdateSessionStats()
	PvpStats.Kills.Text = tostring(Stats.Kills)
	PvpStats.Deaths.Text = tostring(Stats.Deaths)
	PvpStats.KD.Text = Stats.Deaths == 0 and ("%.2f"):format(Stats.Kills) or ("%.2f"):format(Stats.Kills / Stats.Deaths)
	PvpStats.Best.Text = tostring(Stats.BestStreak)
end
UpdateSessionStats()

local crossSection = Section(PvpPage, "Crosshair", "A clean custom crosshair at the center of your screen")
Toggle(crossSection, "Enable Crosshair", "Draw the custom crosshair", Settings.Crosshair, function(v) Settings.Crosshair = v BuildCrosshair() end)
Segmented(crossSection, "Style", { "Cross", "Dot", "Circle", "Hybrid" }, Settings.CrossStyle, function(v) Settings.CrossStyle = v BuildCrosshair() end)
Slider(crossSection, "Size", 2, 30, Settings.CrossSize, " px", function(v) Settings.CrossSize = v BuildCrosshair() end)
Slider(crossSection, "Thickness", 1, 6, Settings.CrossThickness, " px", function(v) Settings.CrossThickness = v BuildCrosshair() end)
Slider(crossSection, "Gap", 0, 20, Settings.CrossGap, " px", function(v) Settings.CrossGap = v BuildCrosshair() end)
Toggle(crossSection, "Use Accent Color", "Off = classic white crosshair", Settings.CrossAccent, function(v) Settings.CrossAccent = v BuildCrosshair() end)
Toggle(crossSection, "Outline", "Thin dark border for visibility", Settings.CrossOutline, function(v) Settings.CrossOutline = v BuildCrosshair() end)

local hudSection = Section(PvpPage, "Combat HUD", "Live combat information on your screen")
Toggle(hudSection, "Target Info", "Shows health and distance of the player under your mouse", Settings.TargetHud, function(v) Settings.TargetHud = v end)
Toggle(hudSection, "Kill Feed", "Shows every elimination in the server", Settings.KillFeed, function(v) Settings.KillFeed = v end)

local alertSection = Section(PvpPage, "Alerts", "Feel the fight")
Toggle(alertSection, "Low Health Warning", "Red pulse on the screen edges when you are low", Settings.LowHpWarning, function(v) Settings.LowHpWarning = v end)
Slider(alertSection, "Warning Threshold", 10, 60, Settings.LowHpThreshold, "%", function(v) Settings.LowHpThreshold = v end)
Toggle(alertSection, "Damage Flash", "Quick red flash whenever you take damage", Settings.DamageFlash, function(v) Settings.DamageFlash = v end)
Button(alertSection, "Reset Session Stats", "Set kills, deaths and streaks back to zero", function()
	Stats.Kills, Stats.Deaths, Stats.Streak, Stats.BestStreak = 0, 0, 0, 0
	UpdateSessionStats()
	Notify("Stats reset", "Your session stats are back to zero.", "info")
end)

--==================================================================--
-- PAGE: VISUALS
--==================================================================--

local VisualsPage = Pages.Visuals
local SetRainbow

local cameraSection = Section(VisualsPage, "Camera", "Tune how you see the battlefield")
local defaultFov = math.clamp(math.floor(workspace.CurrentCamera.FieldOfView + 0.5), 50, 120)
local SetFovSlider = Slider(cameraSection, "Field of View", 50, 120, defaultFov, "°", function(v)
	workspace.CurrentCamera.FieldOfView = v
end)
Button(cameraSection, "Reset Camera", "Default FOV and back to your character", function()
	SetFovSlider(70)
	StopSpectate()
	Notify("Camera reset", "Field of view set back to 70.", "info")
end)

local overlaySection = Section(VisualsPage, "Overlays", "On-screen extras")
Toggle(overlaySection, "Watermark", "FPS, ping and clock in the bottom-left corner", Settings.Watermark, function(v) Settings.Watermark = v end)
Toggle(overlaySection, "Menu Background Blur", "Blur the game while the menu is open", Settings.MenuBlur, function(v)
	Settings.MenuBlur = v
	Tween(Blur, { Size = (v and MenuOpen) and 14 or 0 }, 0.3)
end)

local fxSection = Section(VisualsPage, "Effects", "Extra flair")
SetRainbow = Toggle(fxSection, "Rainbow Accent", "Smoothly cycles the accent color", Settings.RainbowAccent, function(v)
	Settings.RainbowAccent = v
	if not v then SetAccent(Settings.AccentColor) end
end)

--==================================================================--
-- PAGE: SETTINGS
--==================================================================--

local SettingsPage = Pages.Settings

local appearanceSection = Section(SettingsPage, "Appearance", "Colors and size of the interface")
Swatches(appearanceSection, "Accent Color", AccentPresets, 1, function(preset)
	Settings.AccentColor = preset.Color
	if Settings.RainbowAccent then SetRainbow(false, true) Settings.RainbowAccent = false end
	SetAccent(preset.Color)
end)
Slider(appearanceSection, "Interface Scale", 60, 130, math.floor(Settings.UIScale * 100 + 0.5), "%", function(v)
	Settings.UIScale = v / 100
	if MenuOpen then WindowScale.Scale = Settings.UIScale end
end)

local controlSection = Section(SettingsPage, "Controls", "Keyboard shortcuts")
Keybind(controlSection, "Toggle Menu", "Click, then press any key (Esc cancels)", Settings.MenuKey, function(key)
	Settings.MenuKey = key
	for _, hint in ipairs(KeyHints) do hint.Text = key.Name end
	Notify("Keybind updated", "The menu now toggles with " .. key.Name, "success")
end)

local notifySection = Section(SettingsPage, "Notifications", "What mxme pvp tells you about")
Toggle(notifySection, "Notifications", "Toasts in the bottom-right corner", Settings.Notifications, function(v) Settings.Notifications = v end)
Toggle(notifySection, "Join & Leave Alerts", "Get notified when players come and go", Settings.JoinAlerts, function(v) Settings.JoinAlerts = v end)
Button(notifySection, "Send Test Notification", nil, function()
	Notify("Looking good!", "This is how mxme pvp notifications look.", "success")
end)

local dangerSection = Section(SettingsPage, "Danger Zone")
Button(dangerSection, "Unload mxme pvp", "Remove the menu and switch every feature off", function()
	Confirm("Unload mxme pvp?", "The menu will be removed and every feature turned off. Re-run the script to load it again.", "Unload", Unload)
end, true)

local About = New("Frame", { Size = UDim2.new(1, 0, 0, 92), BackgroundColor3 = Color3.new(1, 1, 1), LayoutOrder = NextOrder(), Parent = SettingsPage }, { Corner(12) })
local AboutGradient = New("UIGradient", { Rotation = 15, Parent = About })
OnAccent(function(c) AboutGradient.Color = ColorSequence.new(Shade(c, 0.55), Theme.Panel) end)
local aboutStroke = Stroke(Theme.Accent, 0.6)
aboutStroke.Parent = About
BindAccent(aboutStroke, "Color")
Label({ Text = "MXME PVP", Font = Enum.Font.GothamBlack, TextSize = 18, Position = UDim2.fromOffset(18, 16), Size = UDim2.new(1, -36, 0, 22), Parent = About })
Label({ Text = "Version " .. VERSION .. "  ·  Premium Combat Suite", Font = Enum.Font.GothamMedium, TextSize = 12, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(18, 42), Size = UDim2.new(1, -36, 0, 16), Parent = About })
Label({ Text = "Made by mxme  ·  Thanks for using it!", Font = Enum.Font.Gotham, TextSize = 11, TextColor3 = Theme.Muted, Position = UDim2.fromOffset(18, 62), Size = UDim2.new(1, -36, 0, 14), Parent = About })

--==================================================================--
-- ELIMINATIONS & PLAYER TRACKING
--==================================================================--

local StreakNames = { [3] = "Killing Spree", [5] = "Rampage", [7] = "Dominating", [10] = "Unstoppable" }

local function OnElimination(victim, killer)
	if victim == LocalPlayer then
		Stats.Deaths += 1
		Stats.Streak = 0
		Notify("You were eliminated", killer and ("Taken down by " .. killer.DisplayName) or "Respawning soon...", "danger")
	elseif killer == LocalPlayer then
		Stats.Kills += 1
		Stats.Streak += 1
		Stats.BestStreak = math.max(Stats.BestStreak, Stats.Streak)
		Notify(StreakNames[Stats.Streak] or "Elimination", ("You eliminated %s  ·  Streak %d"):format(victim.DisplayName, Stats.Streak), "success")
	end
	AddKillFeed(killer, victim)
	UpdateSessionStats()
end

local function HookCharacter(player, character)
	local humanoid = character:WaitForChild("Humanoid", 10)
	if not humanoid or not Running then return end

	if player == LocalPlayer then
		local lastHealth = humanoid.Health
		humanoid.HealthChanged:Connect(function(health)
			if health < lastHealth and Settings.DamageFlash then DamageFlash = math.max(DamageFlash, 0.5) end
			lastHealth = health
		end)
	end

	humanoid.Died:Connect(function()
		if not Running then return end
		local tag = humanoid:FindFirstChild("creator")
		local killer = tag and tag:IsA("ObjectValue") and tag.Value or nil
		if killer and (not killer:IsA("Player") or killer == player) then killer = nil end
		OnElimination(player, killer)
	end)
end

local function OnPlayerAdded(player)
	CreatePlayerRow(player)
	if player.Character then task.spawn(HookCharacter, player, player.Character) end
	Track(player.CharacterAdded:Connect(function(character) HookCharacter(player, character) end))
end

for _, player in ipairs(Players:GetPlayers()) do OnPlayerAdded(player) end
RefreshPlayerList()

Track(Players.PlayerAdded:Connect(function(player)
	OnPlayerAdded(player)
	RefreshPlayerList()
	if Settings.JoinAlerts then Notify("Player joined", player.DisplayName .. " joined the server", "info") end
end))

Track(Players.PlayerRemoving:Connect(function(player)
	RemovePlayerRow(player)
	if SpectateTarget == player then StopSpectate() end
	if DrawerPlayer == player then CloseDrawer() end
	RefreshPlayerList()
	if Settings.JoinAlerts then Notify("Player left", player.DisplayName .. " left the server", "warning") end
end))

--==================================================================--
-- INPUT
--==================================================================--

Track(UserInputService.InputBegan:Connect(function(input)
	if KeybindListening or UserInputService:GetFocusedTextBox() then return end
	if input.KeyCode == Settings.MenuKey then SetMenu(not MenuOpen) end
end))

MinimizeButton.MouseButton1Click:Connect(function()
	SetMenu(false)
	Notify("Menu hidden", "Press " .. Settings.MenuKey.Name .. " to bring it back.", "info")
end)
CloseButton.MouseButton1Click:Connect(function()
	Confirm("Unload mxme pvp?", "The menu will be removed and every feature turned off. Re-run the script to load it again.", "Unload", Unload)
end)

if UserInputService.TouchEnabled then
	local MobileToggle = New("TextButton", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 12, 0.5, 0), Size = UDim2.fromOffset(46, 46), BackgroundColor3 = Theme.Panel, Text = "M", Font = Enum.Font.GothamBlack, TextSize = 20, TextColor3 = Color3.new(1, 1, 1), Parent = Gui }, { Corner(14) })
	local mobileStroke = Stroke(Theme.Accent, 0.3, 1.5)
	mobileStroke.Parent = MobileToggle
	BindAccent(mobileStroke, "Color")
	MobileToggle.MouseButton1Click:Connect(function() SetMenu(not MenuOpen) end)
end

--==================================================================--
-- UPDATE LOOPS
--==================================================================--

local function SlowUpdate()
	local okPing, rawPing = pcall(function() return LocalPlayer:GetNetworkPing() end)
	local ping = okPing and math.floor(rawPing * 1000 + 0.5) or 0
	local session = FormatTime(os.clock() - SessionStart)

	ClockPill.Text = os.date("%H:%M")
	UserCardSub.Text = "Online  ·  " .. session

	if MenuOpen then
		if CurrentTab == "Home" then
			HomeStats.Players.Text = ("%d/%d"):format(#Players:GetPlayers(), Players.MaxPlayers)
			HomeStats.FPS.Text = tostring(FPS)
			HomeStats.FPS.TextColor3 = FPS >= 50 and Theme.Success or (FPS >= 30 and Theme.Warning or Theme.Danger)
			HomeStats.Ping.Text = ping .. "ms"
			HomeStats.Ping.TextColor3 = ping <= 80 and Theme.Success or (ping <= 160 and Theme.Warning or Theme.Danger)
			HomeStats.Session.Text = session
			UptimeValue.Text = FormatTime(workspace.DistributedGameTime)
			TeamValue.Text = LocalPlayer.Team and LocalPlayer.Team.Name or "None"
		elseif CurrentTab == "Players" then
			RefreshPlayerList()
			UpdateDrawer()
		end
	end

	Watermark.Visible = Settings.Watermark
	if Settings.Watermark then
		local sep = ('  <font color="#%s">|</font>  '):format(Theme.Muted:ToHex())
		WatermarkText.Text = ('<font color="#%s"><b>MXME</b></font> PVP'):format(Theme.Accent:ToHex())
			.. sep .. FPS .. " FPS" .. sep .. ping .. " ms" .. sep .. os.date("%H:%M")
	end

	if SpectateTarget then
		local humanoid = GetHumanoid(SpectateTarget)
		if humanoid and workspace.CurrentCamera.CameraSubject ~= humanoid then
			workspace.CurrentCamera.CameraSubject = humanoid
		end
	end
end

local frames, fpsTimer, slowTimer, targetTimer = 0, 0, 0, 0
Track(RunService.RenderStepped:Connect(function(dt)
	frames += 1
	fpsTimer += dt
	if fpsTimer >= 0.5 then
		FPS = math.floor(frames / fpsTimer + 0.5)
		frames, fpsTimer = 0, 0
	end

	if Settings.RainbowAccent then
		SetAccent(Color3.fromHSV((os.clock() * 0.08) % 1, 0.6, 1), true)
	end

	LogoGradient.Rotation = (os.clock() * 40) % 360
	SpectateDot.BackgroundTransparency = 0.5 + math.sin(os.clock() * 6) * 0.5
	UpdateVignette(dt)

	targetTimer += dt
	if targetTimer >= 0.05 then
		targetTimer = 0
		UpdateTargetHud()
	end

	slowTimer += dt
	if slowTimer >= 0.25 then
		slowTimer = 0
		SlowUpdate()
	end
end))

task.spawn(function()
	while Running do
		TitleShine.Offset = Vector2.new(-1, 0)
		Tween(TitleShine, { Offset = Vector2.new(1, 0) }, 1.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
		task.wait(4)
	end
end)

--==================================================================--
-- INTRO
--==================================================================--

SelectTab("Home")
BuildCrosshair()

local Splash = New("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(310, 132), BackgroundColor3 = Theme.Panel, Parent = Gui }, { Corner(16), Stroke(Theme.Accent, 0.5) })
local SplashScale = New("UIScale", { Scale = 0.8, Parent = Splash })
local splashBadge = New("Frame", { Position = UDim2.fromOffset(22, 22), Size = UDim2.fromOffset(48, 48), BackgroundColor3 = Color3.new(1, 1, 1), Parent = Splash }, { Corner(13) })
New("UIGradient", { Rotation = 45, Color = ColorSequence.new(Tint(Theme.Accent, 0.25), Shade(Theme.Accent, 0.35)), Parent = splashBadge })
Label({ Text = "M", Font = Enum.Font.GothamBlack, TextSize = 26, TextXAlignment = Enum.TextXAlignment.Center, Size = UDim2.fromScale(1, 1), Parent = splashBadge })
Label({ Text = "MXME PVP", Font = Enum.Font.GothamBlack, TextSize = 20, Position = UDim2.fromOffset(84, 26), Size = UDim2.new(1, -100, 0, 22), Parent = Splash })
local SplashStatus = Label({ Text = "Initializing...", Font = Enum.Font.Gotham, TextSize = 12, TextColor3 = Theme.SubText, Position = UDim2.fromOffset(84, 50), Size = UDim2.new(1, -100, 0, 16), Parent = Splash })
local splashBar = New("Frame", { Position = UDim2.new(0, 22, 1, -32), Size = UDim2.new(1, -44, 0, 6), BackgroundColor3 = Theme.Stroke, Parent = Splash }, { Round() })
local splashFill = New("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = Theme.Accent, Parent = splashBar }, { Round() })

Tween(SplashScale, { Scale = 1 }, 0.5, Enum.EasingStyle.Back)
local steps = { "Building interface...", "Syncing players...", "Loading PvP tools...", "Ready" }
for i, text in ipairs(steps) do
	SplashStatus.Text = text
	Tween(splashFill, { Size = UDim2.fromScale(i / #steps, 1) }, 0.3)
	task.wait(0.32)
end
task.wait(0.15)
Tween(SplashScale, { Scale = 0 }, 0.35, Enum.EasingStyle.Back, Enum.EasingDirection.In)
task.wait(0.35)
Splash:Destroy()

if Running then
	SetMenu(true)
	Notify("mxme pvp loaded", "Press " .. Settings.MenuKey.Name .. " to toggle the menu.", "success", 5)
end
