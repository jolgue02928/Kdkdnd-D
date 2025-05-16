local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local hrp = character:WaitForChild("HumanoidRootPart")
local hum = character:WaitForChild("Humanoid")

-- Variável global para sinalizar quando canhão foi encontrado e pegado
_G.CannonFound = false

-- FUNÇÃO PARA VER SE UM MODELO TEM PARTES NÃO ANCORADAS
local function isUnanchored(model)
	for _, p in pairs(model:GetDescendants()) do
		if p:IsA("BasePart") and not p.Anchored then
			return true
		end
	end
	return false
end

-- FUNÇÃO PARA ENCONTRAR O CANHÃO NO WORKSPACE
local function findCannon()
	local exclude = nil
	local fort = workspace:FindFirstChild("FortConstitution")
	if fort then
		exclude = fort:FindFirstChild("Cannon", true)
	end

	for _, d in ipairs(workspace:GetDescendants()) do
		if d:IsA("Model") and d.Name == "Cannon" and d ~= exclude then
			return d
		end
	end
	return nil
end

-- MOVER O PERSONAGEM ATÉ UMA POSIÇÃO LONGE (20 segundos de tween)
local tweenInfo = TweenInfo.new(20, Enum.EasingStyle.Linear)
local targetCFrame = CFrame.new(-9, 3, -50000)
local tween = TweenService:Create(hrp, tweenInfo, { CFrame = targetCFrame })
tween:Play()

local conn
conn = RunService.RenderStepped:Connect(function()
	local cannon = findCannon()
	if cannon then
		if isUnanchored(cannon) then
			local seat = cannon:FindFirstChildWhichIsA("VehicleSeat", true)
			if seat and not seat.Occupant then
				tween:Cancel()
				conn:Disconnect()

				hrp.CFrame = seat.CFrame
				seat:Sit(hum)

				task.delay(1, function()
					if hum.Sit then
						hum.Sit = false
						hum:ChangeState(Enum.HumanoidStateType.Jumping)
					end

					task.delay(1, function()
						seat:Sit(hum)

						task.delay(1, function()
							hrp.CFrame = targetCFrame -- volta para posição longe
							hum.JumpPower = 0
							hum:SetStateEnabled(Enum.HumanoidStateType.Jumping, false)
							task.wait(1)
							_G.CannonFound = true
						end)
					end)
				end)
			end
		end
	end
end)

-- ESPERA ATÉ PEGAR O CANHÃO
repeat task.wait() until _G.CannonFound

-- --------------------------------------------------------
-- COMEÇA O FARM DAS BONDS
-- --------------------------------------------------------

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "BondUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = cloneref(game:GetService("CoreGui"))

local bondLabel = Instance.new("TextLabel")
bondLabel.Size = UDim2.new(0.3, 0, 0.1, 0)
bondLabel.Position = UDim2.new(0.35, 0, 0.45, 0)
bondLabel.BackgroundTransparency = 1
bondLabel.TextScaled = true
bondLabel.Font = Enum.Font.SourceSansBold
bondLabel.TextColor3 = Color3.new(1, 1, 1)
bondLabel.TextStrokeTransparency = 0.5
bondLabel.Text = "Bond Collected: 0"
bondLabel.Parent = screenGui

local activateRemote = ReplicatedStorage:WaitForChild("Shared")
	:WaitForChild("Network")
	:WaitForChild("RemotePromise")
	:WaitForChild("Remotes")
	:WaitForChild("C_ActivateObject")

local character = player.Character or player.CharacterAdded:Wait()
local root = character:WaitForChild("HumanoidRootPart")
local lockedY = root.Position.Y

-- FUNÇÃO PARA ENCONTRAR A BOND MAIS PRÓXIMA
local function findNearestBond()
	local closest, shortestDist = nil, math.huge
	for _, item in ipairs(workspace.RuntimeItems:GetDescendants()) do
		if item:IsA("Model") and item.Name:lower() == "bond" then
			local primary = item.PrimaryPart or item:FindFirstChildWhichIsA("BasePart")
			if primary then
				local dist = (primary.Position - root.Position).Magnitude
				if dist < shortestDist then
					shortestDist = dist
					closest = item
				end
			end
		end
	end
	return closest
end

-- LOCK Y para evitar cair durante tween
task.spawn(function()
	while true do
		if root and root.Parent then
			local pos = root.Position
			root.Velocity = Vector3.new(root.Velocity.X, 0, root.Velocity.Z)
			root.CFrame = CFrame.new(pos.X, lockedY, pos.Z)
		end
		task.wait()
	end
end)

local bondCount = 0

-- TELEPORT PARA A BOND E ATIVA-LA
local function teleportTo(bond)
	local primary = bond.PrimaryPart or bond:FindFirstChildWhichIsA("BasePart")
	if not primary then return end

	root.CFrame = primary.CFrame + Vector3.new(0, 5, 0)

	local startTime = os.clock()
	while bond.Parent and os.clock() - startTime < 1 do
		activateRemote:FireServer(bond)
		task.wait(0.1)
	end

	if not bond.Parent then
		bondCount += 1
		bondLabel.Text = "Bond Collected: " .. bondCount
		print("Bond taken! Total:", bondCount)
	else
		print("Failed to take bond within timeout.")
	end
end

local activeTween = nil
local travelTime = 1

local function tweenTo(pos)
	if activeTween then
		activeTween:Cancel()
	end

	local tweenInfo = TweenInfo.new(travelTime, Enum.EasingStyle.Linear)
	local goal = {CFrame = CFrame.new(pos)}
	activeTween = TweenService:Create(root, tweenInfo, goal)

	local finished = false
	local lastTweenStart = os.clock()

	local connection
	connection = activeTween.Completed:Connect(function()
		finished = true
		if connection then connection:Disconnect() end
	end)

	activeTween:Play()

	while not finished do
		local bond = findNearestBond()
		if bond then
			activeTween:Cancel()
			teleportTo(bond)
			return
		end

		if os.clock() - lastTweenStart > 5 and not finished then
			warn("Tween possibly stuck, retrying...")
			return tweenTo(pos)
		end

		task.wait(0.1)
	end
end

-- PRIMEIRA PASSAGEM: Pega as Bonds que já estejam perto (antes de varrer o mapa)
while true do
	local bond = findNearestBond()
	if bond then
		teleportTo(bond)
	else
		break
	end
	task.wait(0.05)
end

-- Varredura em zig-zag para pegar todas as bonds no mapa
local layerSize = 2048
local halfSize = layerSize / 2
local xStart = -halfSize
local xEnd = halfSize
local y = -50
local zStart = 30000
local zEnd = -49872
local zStep = -layerSize

local z = zStart
local direction = 1

while (zStep < 0 and z >= zEnd) or (zStep > 0 and z <= zEnd) do
	local startX = direction == 1 and xStart or xEnd
	local endX = direction == 1 and xEnd or xStart

	local startPos = Vector3.new(startX, y, z)
	local endPos = Vector3.new(endX, y, z)

	tweenTo(startPos)
	tweenTo(endPos)

	z += zStep
	direction *= -1
end

task.wait(2)

-- --------------------------------------------------------
-- FAZ REJOIN PARA NOVA PARTIDA
-- --------------------------------------------------------

local createPartyEvent = ReplicatedStorage:WaitForChild("Shared")
    :WaitForChild("Network")
    :WaitForChild("RemoteEvent")
    :WaitForChild("CreateParty")

local args = {
    [1] = {
        ["trainId"] = "default",
        ["maxMembers"] = 1,
        ["gameMode"] = "Normal"
    }
}

-- Faz rejoin: cria nova partida
createPartyEvent:FireServer(unpack(args))

print("Autofarm completo. Reingressando em nova partida...")
