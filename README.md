local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera

local player = Players.LocalPlayer
local rangeInPixels = 70        -- 원 반지름 (UI)
local detectRange = 90          -- 감지 반지름
local offsetY = 55              -- 중심 Y 오프셋

local myCharacter = player.Character or player.CharacterAdded:Wait()
player.CharacterAdded:Connect(function(char)
	myCharacter = char
end)

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "RangeIndicatorGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

local numPoints = 36
local points = {}

for i = 1, numPoints do
	local point = Instance.new("Frame")
	point.Size = UDim2.new(0, 3, 0, 3)
	point.BackgroundColor3 = Color3.new(0, 1, 0) -- 기본 초록색
	point.BorderSizePixel = 0
	point.AnchorPoint = Vector2.new(0.5, 0.5)
	point.Parent = screenGui
	table.insert(points, point)
end

-- ⭕ 원 위치와 색 갱신
local function updateCircle(color)
	color = color or Color3.new(0, 1, 0) -- 기본 초록색
	local centerX = Camera.ViewportSize.X / 2
	local centerY = Camera.ViewportSize.Y / 2 - offsetY

	for i = 1, numPoints do
		local angle = (2 * math.pi) * (i / numPoints)
		local x = centerX + math.cos(angle) * rangeInPixels
		local y = centerY + math.sin(angle) * rangeInPixels
		points[i].Position = UDim2.new(0, x, 0, y)
		points[i].BackgroundColor3 = color
	end
end

-- 🎯 감지 조건
local function isPlayerInRange(otherPlayer)
	if otherPlayer == player then return false end

	local char = otherPlayer.Character
	if not char then return false end

	local head = char:FindFirstChild("Head")
	local humanoid = char:FindFirstChild("Humanoid")
	if not head or not humanoid or humanoid.Health <= 0 then return false end

	if not Players:GetPlayerFromCharacter(char) then return false end

	local screenPoint, onScreen = Camera:WorldToViewportPoint(head.Position)
	if not onScreen then return false end

	local centerX = Camera.ViewportSize.X / 2
	local centerY = Camera.ViewportSize.Y / 2 - offsetY
	local dist = (Vector2.new(screenPoint.X, screenPoint.Y) - Vector2.new(centerX, centerY)).Magnitude
	if dist > detectRange then return false end

	local origin = Camera.CFrame.Position
	local direction = (head.Position - origin)

	local rayParams = RaycastParams.new()
	rayParams.FilterType = Enum.RaycastFilterType.Blacklist
	rayParams.FilterDescendantsInstances = {player.Character}

	local result = workspace:Raycast(origin, direction, rayParams)
	if result and result.Instance and not head:IsDescendantOf(result.Instance.Parent) then
		return false
	end

	return true
end

-- 🎯 조준 함수
local function lookAtHeadOf(otherPlayer)
	local head = otherPlayer.Character and otherPlayer.Character:FindFirstChild("Head")
	if head then
		Camera.CFrame = CFrame.new(Camera.CFrame.Position, head.Position)
	end
end

-- 🔁 매 프레임마다 감지 & 조준 & 색 변경
RunService.RenderStepped:Connect(function()
	local targetFound = false

	for _, otherPlayer in ipairs(Players:GetPlayers()) do
		if isPlayerInRange(otherPlayer) then
			lookAtHeadOf(otherPlayer)
			targetFound = true
			break
		end
	end

	if targetFound then
		updateCircle(Color3.new(1, 0, 0)) -- 빨간색
	else
		updateCircle(Color3.new(0, 1, 0)) -- 초록색
	end
end)
