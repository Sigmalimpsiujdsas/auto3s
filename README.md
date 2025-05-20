
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- State
local JumpEnabled = false
local JumpPower = 45
local DeFollowEnabled = false
local AimFOV = 100
local MagEnabled = true
local MagDistance = 20

-- UI
local screenGui = Instance.new("ScreenGui", game.CoreGui)
screenGui.Name = "LimpGui"
screenGui.ResetOnSpawn = false

local mainFrame = Instance.new("Frame", screenGui)
mainFrame.Size = UDim2.new(0, 300, 0, 300)
mainFrame.Position = UDim2.new(0.5, -150, 0.5, -125)
mainFrame.BackgroundColor3 = Color3.new(0.2, 0.2, 0.2)
mainFrame.Active = true
mainFrame.Draggable = true

local title = Instance.new("TextLabel", mainFrame)
title.Size = UDim2.new(1, 0, 0, 30)
title.Text = "limps auto 3 script"
title.TextColor3 = Color3.new(1,1,1)
title.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)

local function createToggle(name, default, callback, position)
	local toggle = Instance.new("TextButton", mainFrame)
	toggle.Size = UDim2.new(1, -20, 0, 30)
	toggle.Position = UDim2.new(0, 10, 0, position)
	toggle.Text = name .. ": " .. (default and "ON" or "OFF")
	toggle.BackgroundColor3 = Color3.new(0.3,0.3,0.3)
	toggle.TextColor3 = Color3.new(1,1,1)
	local state = default
	toggle.MouseButton1Click:Connect(function()
		state = not state
		toggle.Text = name .. ": " .. (state and "ON" or "OFF")
		callback(state)
	end)
end

local function createSlider(name, min, max, default, callback, position)
	local label = Instance.new("TextLabel", mainFrame)
	label.Size = UDim2.new(1, -20, 0, 20)
	label.Position = UDim2.new(0, 10, 0, position)
	label.Text = name .. ": " .. default
	label.TextColor3 = Color3.new(1,1,1)
	label.BackgroundTransparency = 1

	local slider = Instance.new("TextButton", mainFrame)
	slider.Size = UDim2.new(1, -20, 0, 20)
	slider.Position = UDim2.new(0, 10, 0, position + 20)
	slider.BackgroundColor3 = Color3.new(0.4, 0.4, 0.4)
	slider.Text = ""
	local val = default

	slider.MouseButton1Down:Connect(function()
		local conn
		conn = RunService.RenderStepped:Connect(function()
			if UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1) then
				local rel = UserInputService:GetMouseLocation().X - slider.AbsolutePosition.X
				local percent = math.clamp(rel / slider.AbsoluteSize.X, 0, 1)
				val = math.floor(min + (max - min) * percent)
				label.Text = name .. ": " .. val
				callback(val)
			else
				conn:Disconnect()
			end
		end)
	end)
end

-- FOV Circle
local circle = Drawing.new("Circle")
circle.Visible = false
circle.Thickness = 1
circle.Transparency = 1
circle.Color = Color3.new(1, 1, 1)
circle.Filled = false
circle.NumSides = 100
circle.Radius = AimFOV

-- UI Components
createToggle("High Jump", false, function(v) JumpEnabled = v end, 40)
createToggle("De Follow", false, function(v)
	DeFollowEnabled = v
	circle.Visible = v
end, 80)
createSlider("FOV", 30, 300, 100, function(v)
	AimFOV = v
	if DeFollowEnabled then
		circle.Radius = v
	end
end, 120)
createToggle("Mag Catch", true, function(v) MagEnabled = v end, 170)
createSlider("Mag Distance", 10, 50, 20, function(v) MagDistance = v end, 210)
createSlider("Jump Power", 10, 150, 45, function(v) JumpPower = v end, 250)

-- Anti-Teleport Down
RunService.Stepped:Connect(function()
	local character = LocalPlayer.Character
	if character and character:FindFirstChild("HumanoidRootPart") and JumpEnabled then
		local hrp = character.HumanoidRootPart
		if hrp.Velocity.Y < -50 then
			hrp.Velocity = Vector3.new(hrp.Velocity.X, 0, hrp.Velocity.Z)
		end
	end
end)

-- Jump Modifier
UserInputService.InputBegan:Connect(function(input)
	if input.KeyCode == Enum.KeyCode.Space and JumpEnabled then
		local char = LocalPlayer.Character
		if char and char:FindFirstChild("Humanoid") and char:FindFirstChild("HumanoidRootPart") then
			local hum = char.Humanoid
			if hum:GetState() == Enum.HumanoidStateType.Jumping or hum:GetState() == Enum.HumanoidStateType.Freefall then
				local hrp = char.HumanoidRootPart
				if hrp.Velocity.Y > -0.1 then
					hrp.Velocity = Vector3.new(0, JumpPower, 0)
				end
			end
		end
	end
end)

-- Closest Player to Mouse
local function getClosestToMouse()
	local shortest, closest = AimFOV, nil
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("Head") then
			local head = p.Character.Head
			local pos, onscreen = Camera:WorldToViewportPoint(head.Position)
			if onscreen then
				local dist = (Vector2.new(pos.X, pos.Y) - Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)).Magnitude
				if dist < shortest then
					shortest = dist
					closest = head
				end
			end
		end
	end
	return closest
end

-- Ball Finder
local function getBall()
	for _, obj in ipairs(workspace:GetDescendants()) do
		if obj:IsA("BasePart") and obj.Name:lower():find("ball") then
			return obj
		end
	end
end

-- Main loop
RunService.RenderStepped:Connect(function()
	circle.Position = UserInputService:GetMouseLocation()

	if DeFollowEnabled and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
		local target = getClosestToMouse()
		if target then
			Camera.CFrame = CFrame.new(Camera.CFrame.Position, target.Position + Vector3.new(0, 0.2, 0))
		end
	end

	if MagEnabled then
		local char = LocalPlayer.Character
		local ball = getBall()
		if char and ball and char:FindFirstChild("HumanoidRootPart") then
			local hrp = char.HumanoidRootPart
			if ball:IsDescendantOf(workspace) and (hrp.Position - ball.Position).Magnitude <= MagDistance then
				hrp.CFrame = CFrame.new(ball.Position + Vector3.new(0, 2, 0))
			end
		end
	end
end)


UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if not gameProcessed and input.KeyCode == Enum.KeyCode.K then
        mainFrame.Visible = not mainFrame.Visible
    end
end)
