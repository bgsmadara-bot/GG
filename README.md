local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local eggsFolder = workspace:WaitForChild("Eggs")
local belt = workspace:WaitForChild("Belt")

local ALLOWED_RARITIES = {
	Secreto = true,
	Eterno = true,
	Divino = true,
}

local LOOP_DELAY = 1.5
local TELEPORT_HEIGHT = 3
local COLLECTION_DELAY = 0.3

-- Cria o evento usado pelo botão da interface
local toggleEvent = ReplicatedStorage:FindFirstChild("EggBotToggle")

if not toggleEvent then
	toggleEvent = Instance.new("RemoteEvent")
	toggleEvent.Name = "EggBotToggle"
	toggleEvent.Parent = ReplicatedStorage
end

local botEnabled = {}

local function getObjectPosition(object)
	if not object or not object.Parent then
		return nil
	end

	if object:IsA("BasePart") then
		return object.Position
	end

	if object:IsA("Model") then
		return object:GetPivot().Position
	end

	return nil
end

local function findFarthestEgg(playerPosition)
	local farthestEgg = nil
	local greatestDistance = -1

	for _, egg in ipairs(eggsFolder:GetChildren()) do
		local rarity = egg:GetAttribute("Rarity")
		local collected = egg:GetAttribute("Collected")
		local enabled = egg:GetAttribute("Enabled")

		local eggPosition = getObjectPosition(egg)

		if eggPosition
			and ALLOWED_RARITIES[rarity]
			and collected ~= true
			and enabled ~= false then

			local distance = (eggPosition - playerPosition).Magnitude

			if distance > greatestDistance then
				greatestDistance = distance
				farthestEgg = egg
			end
		end
	end

	return farthestEgg
end

local function teleportToPosition(player, position)
	local character = player.Character
	if not character or not position then
		return false
	end

	local rootPart = character:FindFirstChild("HumanoidRootPart")

	if not rootPart then
		return false
	end

	local targetPosition = position + Vector3.new(0, TELEPORT_HEIGHT, 0)

	character:PivotTo(CFrame.new(targetPosition))
	rootPart.AssemblyLinearVelocity = Vector3.zero

	return true
end

local function collectEgg(player, egg)
	if not egg or not egg.Parent then
		return
	end

	if egg:GetAttribute("Collected") == true then
		return
	end

	-- Impede duas coletas simultâneas
	egg:SetAttribute("Collected", true)

	-- Recompensa de exemplo
	local leaderstats = player:FindFirstChild("leaderstats")
	local eggsCollected = leaderstats
		and leaderstats:FindFirstChild("EggsCollected")

	if eggsCollected and eggsCollected:IsA("IntValue") then
		eggsCollected.Value += 1
	end

	-- Remove o egg após a coleta
	egg:Destroy()
end

-- Recebe o comando do botão de ligar/desligar
toggleEvent.OnServerEvent:Connect(function(player, requestedState)
	if typeof(requestedState) ~= "boolean" then
		return
	end

	botEnabled[player] = requestedState
end)

local function startBotForPlayer(player)
	botEnabled[player] = false

	task.spawn(function()
		while player.Parent do
			if botEnabled[player] then
				local character = player.Character
				local rootPart = character
					and character:FindFirstChild("HumanoidRootPart")

				local beltPosition = getObjectPosition(belt)

				if rootPart and beltPosition then
					-- Procura o egg permitido mais distante
					local farthestEgg = findFarthestEgg(rootPart.Position)

					if farthestEgg then
						local eggPosition = getObjectPosition(farthestEgg)

						if eggPosition then
							-- Vai até o egg
							local arrivedAtEgg = teleportToPosition(
								player,
								eggPosition
							)

							if arrivedAtEgg then
								task.wait(COLLECTION_DELAY)

								-- Coleta o egg
								collectEgg(player, farthestEgg)

								task.wait(COLLECTION_DELAY)

								-- Volta até o Belt
								teleportToPosition(player, beltPosition)
							end
						end
					end
				end
			end

			task.wait(LOOP_DELAY)
		end
	end)
end

Players.PlayerAdded:Connect(function(player)
	startBotForPlayer(player)
end)

Players.PlayerRemoving:Connect(function(player)
	botEnabled[player] = nil
end)

-- Também funciona se o script for iniciado com jogadores já conectados
for _, player in ipairs(Players:GetPlayers()) do
	startBotForPlayer(player)
end
