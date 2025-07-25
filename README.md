local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera

local player = Players.LocalPlayer
local rangeInPixels = 70        -- 원 반지름 (UI용)
local detectRange = 120         -- 감지 반지름 (실제 감지 범위)
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
	point.BackgroundColor3 = Color3.new(0, 1, 0)
	point.BorderSizePixel = 0
	point.AnchorPoint = Vector2.new(0.5, 0.5)
	point.Parent = screenGui
	table.insert(points, point)
end

local function updateCircle()
	local centerX = Camera.ViewportSize.X / 2
	local centerY = Camera.ViewportSize.Y / 2 - offsetY

	for i = 1, numPoints do
		local angle = (2 * math.pi) * (i / numPoints)
		local x = centerX + math.cos(angle) * rangeInPixels
		local y = centerY + math.sin(angle) * rangeInPixels
		points[i].Position = UDim2.new(0, x, 0, y)
	end
end

local function isPlayerInRange(otherPlayer)
	if otherPlayer == player then return false end
	
	-- 같은 팀 플레이어 제외
	if player.Team and otherPlayer.Team and player.Team == otherPlayer.Team then
		return false
	end

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

local function lookAtHeadOf(otherPlayer)
	local head = otherPlayer.Character and otherPlayer.Character:FindFirstChild("Head")
	if head then
		Camera.CFrame = CFrame.new(Camera.CFrame.Position, head.Position)
	end
end

RunService.RenderStepped:Connect(function()
	updateCircle()

	for _, otherPlayer in ipairs(Players:GetPlayers()) do
		if isPlayerInRange(otherPlayer) then
			lookAtHeadOf(otherPlayer)
			break
		end
	end
end)
