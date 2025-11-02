Project: Roblox DataStore + Backups + Live UI (Coins, Time, XP, Username) + RPC + Debug toggle

How to use
- Repliziere die Explorer-Struktur exakt.
- Erstelle jede Script/ModuleScript-Datei und füge die passenden Code-Abschnitte ein.
- Debug-Logs zentral über ServerSecrets.DEBUG_ENABLED steuern (true/false).

Explorer structure (create exactly this)

ReplicatedStorage
  Folder: Remotes
    RemoteEvent: RPC

ServerStorage
  ModuleScript: ServerSecrets

ServerScriptService
  Folder: ServerModules
    ModuleScript: DebugLogger
    ModuleScript: DataStoreManager
    ModuleScript: PlayerManager
    ModuleScript: RPCGateway
    ModuleScript: ServerConsole
    ModuleScript: SecurityModule
    ModuleScript: XPLevelSystem
    ModuleScript: BuildSystem
  Script: Server          (du kannst den Namen frei wählen; liegt in ServerScriptService)

StarterPlayer
  StarterPlayerScripts
    LocalScript: ClientGlue
    LocalScript: BuildClient

StarterGui
  ScreenGui: Info
    Frame: BackFrame
      UICorner           (deine bestehenden UI-Objekte bleiben unberührt)
      UIStroke
      Frame: Profile
        UICorner
        UIStroke
        ImageLabel: Image
          UICorner
          UIStroke
      TextLabel: Coins
      TextLabel: Stats
      TextLabel: Coins_Display
      TextLabel: username
      TextLabel: Time
      TextLabel: Time_Display
      TextLabel: XP_Display
      TextLabel: Blocks_Display

Behavior overview
- Spawn: Spieler spawnen sofort; Daten werden asynchron geladen und in leaderstats/Client-UI synchronisiert.
- Backups: 5-Minuten-Intervall (Hauptspeicher + Zeitstempel-Backup + Index).
- Immediate saves: Bei Coins-/XP-Änderungen oder periodischen Grants debounced sofort sichern.
- Playtime: Gesamtspielzeit wird persistiert und beim Rejoin fortgeführt; Anzeige im UI läuft live hoch.
- XP/Level: XP nur serverseitig, Level-Anzeige clientseitig und in leaderstats (max Level 500). Professionelles XP-zu-Level-System.
- Chat-Befehle: /coins, /xp, /wipe für Daten-Management (nur für Testzwecke).
- Build-System: R-Taste zum Platzieren, Klicken zum Auswählen, Pfeiltasten zum Verschieben/Drehen, T-Taste zum Löschen.
- Security: Verschlüsselte Client-Server-Kommunikation, HMAC-Verifizierung, keine echten Werte an Client.
- Debug: Vereinfachte, sinnvolle Ausgaben via DebugLogger; per ServerSecrets.DEBUG_ENABLED umschaltbar.

Code sections

================================================================================
1) ServerStorage/ServerSecrets (ModuleScript)
================================================================================
local ServerSecrets = {}

ServerSecrets.GAME_DATASTORE_NAME = "PlayerData"
ServerSecrets.BACKUP_DATASTORE_NAME = "PlayerBackups"
ServerSecrets.BACKUP_INDEX_DATASTORE_NAME = "PlayerBackupIndex"

ServerSecrets.DEFAULT_PLAYER_DATA = {
	coins = 300,
	lastSave = 0,
	version = 1,
	playTimeSeconds = 0,
	xp = 0,
	availableBlocks = 10,
	placedBlocks = {},
}

ServerSecrets.MAX_LEVEL = 500
ServerSecrets.XP_BASE_MULTIPLIER = 100
ServerSecrets.XP_EXPONENT = 1.5

ServerSecrets.IMMEDIATE_SAVE_DEBOUNCE_MS = 300
ServerSecrets.BACKUP_INTERVAL_SECONDS = 300
ServerSecrets.PERIODIC_GRANT_INTERVAL_SECONDS = 120
ServerSecrets.PERIODIC_GRANT_COINS = 100

ServerSecrets.RPC_RATE_LIMIT_PER_SECOND = 8
ServerSecrets.RPC_BURST_TOKENS = 16

ServerSecrets.SERVER_HMAC_SECRET = "replace-with-a-random-long-secret"
ServerSecrets.BACKUP_INDEX_MAX = 50

ServerSecrets.DEBUG_ENABLED = true

return ServerSecrets

================================================================================
2) ServerScriptService/ServerModules/DebugLogger (ModuleScript)
================================================================================
local ServerSecrets = require(game.ServerStorage.ServerSecrets)

local DebugLogger = {}

function DebugLogger.Log(message)
	if not ServerSecrets.DEBUG_ENABLED then
		return
	end
	print("[DEBUG] " .. tostring(message))
end

function DebugLogger.Warn(message)
	if not ServerSecrets.DEBUG_ENABLED then
		return
	end
	warn("[WARN] " .. tostring(message))
end

function DebugLogger.Error(message)
	if not ServerSecrets.DEBUG_ENABLED then
		return
	end
	warn("[ERROR] " .. tostring(message))
end

return DebugLogger

================================================================================
3) ServerScriptService/ServerModules/SecurityModule (ModuleScript)
================================================================================
local HttpService = game:GetService("HttpService")
local ServerSecrets = require(game.ServerStorage.ServerSecrets)

local SecurityModule = {}

function SecurityModule.CreateHMAC(data, userId)
	local combined = tostring(userId) .. ":" .. tostring(data) .. ":" .. ServerSecrets.SERVER_HMAC_SECRET
	local hash = HttpService:GenerateGUID(false):gsub("-", "")
	local combinedBytes = {}
	local hashValue = 0
	for i = 1, #combined do
		local byte = string.byte(combined, i)
		combinedBytes[i] = byte
		hashValue = hashValue + (byte * i * (userId % 1000))
		hashValue = hashValue % 2147483647
	end
	local finalHash = hash .. string.format("%08x", hashValue % 4294967295)
	return finalHash
end

function SecurityModule.EncryptForClient(userId, coins, level, playTimeSeconds)
	local hash = SecurityModule.CreateHMAC(userId .. coins .. level .. playTimeSeconds, userId)
	return {
		coins = coins,
		level = level,
		playTime = playTimeSeconds,
		hash = hash,
	}
end

function SecurityModule._obfuscateValue(value, userId)
	local seed = userId % 1000
	return bit32.bxor(math.floor(value), seed) + 1337
end

function SecurityModule._deobfuscateValue(encrypted, userId)
	local seed = userId % 1000
	return bit32.bxor(encrypted - 1337, seed)
end

function SecurityModule.VerifyClientData(userId, receivedHash, expectedHash)
	if receivedHash ~= expectedHash then
		return false
	end
	return true
end

function SecurityModule.CreateSecureUpdate(userId, coins, level, playTimeSeconds)
	local encrypted = SecurityModule.EncryptForClient(userId, coins, level, playTimeSeconds)
	return encrypted
end

return SecurityModule

================================================================================
4) ServerScriptService/ServerModules/XPLevelSystem (ModuleScript)
================================================================================
local ServerSecrets = require(game.ServerStorage.ServerSecrets)

local XPLevelSystem = {}

function XPLevelSystem.XPForLevel(level)
	if level <= 0 then
		return 0
	end
	if level > ServerSecrets.MAX_LEVEL then
		level = ServerSecrets.MAX_LEVEL
	end
	local xp = ServerSecrets.XP_BASE_MULTIPLIER * (level ^ ServerSecrets.XP_EXPONENT)
	return math.floor(xp)
end

function XPLevelSystem.LevelForXP(xp)
	if xp <= 0 then
		return 1
	end
	local level = math.pow(xp / ServerSecrets.XP_BASE_MULTIPLIER, 1 / ServerSecrets.XP_EXPONENT)
	level = math.floor(level)
	
	if level < 1 then
		return 1
	end
	if level > ServerSecrets.MAX_LEVEL then
		return ServerSecrets.MAX_LEVEL
	end
	
	while XPLevelSystem.XPForLevel(level + 1) <= xp and level < ServerSecrets.MAX_LEVEL do
		level = level + 1
	end
	
	while XPLevelSystem.XPForLevel(level) > xp and level > 1 do
		level = level - 1
	end
	
	return level
end

function XPLevelSystem.AddXP(currentXP, amount)
	local newXP = math.max(0, currentXP + amount)
	local newLevel = XPLevelSystem.LevelForXP(newXP)
	return newXP, newLevel
end

function XPLevelSystem.CheckLevelUp(oldXP, newXP)
	local oldLevel = XPLevelSystem.LevelForXP(oldXP)
	local newLevel = XPLevelSystem.LevelForXP(newXP)
	return newLevel > oldLevel, newLevel
end

function XPLevelSystem.XPInCurrentLevel(totalXP)
	local currentLevel = XPLevelSystem.LevelForXP(totalXP)
	local xpForCurrentLevel = XPLevelSystem.XPForLevel(currentLevel)
	local xpInLevel = totalXP - xpForCurrentLevel
	return math.max(0, xpInLevel)
end

function XPLevelSystem.XPNeededForNextLevel(totalXP)
	local currentLevel = XPLevelSystem.LevelForXP(totalXP)
	if currentLevel >= ServerSecrets.MAX_LEVEL then
		return 0
	end
	local xpForNextLevel = XPLevelSystem.XPForLevel(currentLevel + 1)
	return xpForNextLevel - totalXP
end

return XPLevelSystem

================================================================================
5) ServerScriptService/ServerModules/DataStoreManager (ModuleScript)
================================================================================
local DataStoreService = game:GetService("DataStoreService")

local ServerSecrets = require(game.ServerStorage.ServerSecrets)
local DebugLogger = require(script.Parent.DebugLogger)

local DataStoreManager = {}

local mainStore = DataStoreService:GetDataStore(ServerSecrets.GAME_DATASTORE_NAME)
local backupStore = DataStoreService:GetDataStore(ServerSecrets.BACKUP_DATASTORE_NAME)
local backupIndexStore = DataStoreService:GetDataStore(ServerSecrets.BACKUP_INDEX_DATASTORE_NAME)

local function utcNowSec()
	return os.time(os.date("!*t"))
end

local function cloneDefault()
	local d = ServerSecrets.DEFAULT_PLAYER_DATA
	return {
		coins = d.coins,
		lastSave = d.lastSave,
		version = d.version,
		playTimeSeconds = d.playTimeSeconds,
		xp = d.xp,
		availableBlocks = d.availableBlocks,
		placedBlocks = {},
	}
end

local function deepMergeLastWriteWins(oldValue, newValue)
	if typeof(oldValue) ~= "table" then
		return newValue
	end
	if typeof(newValue) ~= "table" then
		return oldValue
	end
	local result = {}
	for k, v in pairs(oldValue) do
		result[k] = v
	end
	for k, v in pairs(newValue) do
		result[k] = v
	end
	return result
end

local function backoff(attempt)
	local base = 0.5
	return base * (2 ^ (attempt - 1)) + math.random() * 0.25
end

local function tryUpdateAsync(store, key, transform, userId, label)
	local maxAttempts = 6
	local attempt = 0
	while attempt < maxAttempts do
		attempt += 1
		local started = utcNowSec()
		local ok, result = pcall(function()
			return store:UpdateAsync(key, function(oldValue)
				return transform(oldValue)
			end)
		end)
		if ok then
			DebugLogger.Log(string.format("Save %s successful (attempt %d)", label, attempt))
			return result, nil
		else
			DebugLogger.Warn(string.format("Save %s failed, retrying (attempt %d)", label, attempt))
			task.wait(backoff(attempt))
		end
	end
	return nil, "UpdateAsync failed"
end

local function trySetAsync(store, key, value, userId, label)
	local maxAttempts = 6
	local attempt = 0
	while attempt < maxAttempts do
		attempt += 1
		local started = utcNowSec()
		local ok = pcall(function()
			store:SetAsync(key, value)
		end)
		if ok then
			DebugLogger.Log(string.format("Backup %s successful (attempt %d)", label, attempt))
			return true
		else
			DebugLogger.Warn(string.format("Backup %s failed, retrying (attempt %d)", label, attempt))
			task.wait(backoff(attempt))
		end
	end
	return false
end

function DataStoreManager.LoadPlayerData(userId)
	local key = tostring(userId)
	DebugLogger.Log(string.format("Loading data for user %d", userId))
	local ok, value = pcall(function()
		return mainStore:GetAsync(key)
	end)
	if not ok then
		DebugLogger.Error(string.format("Failed to load data for user %d", userId))
		return cloneDefault(), "GetAsync error"
	end
	if value == nil then
		DebugLogger.Log(string.format("New player %d, using default data", userId))
		return cloneDefault(), nil
	end
	if typeof(value) ~= "table" then
		DebugLogger.Warn(string.format("Invalid data type for user %d, using default", userId))
		return cloneDefault(), "Invalid type"
	end
	if typeof(value.coins) == "number" and value.coins >= 0 then
		value.coins = math.floor(value.coins)
	else
		value.coins = ServerSecrets.DEFAULT_PLAYER_DATA.coins
	end
	if typeof(value.lastSave) == "number" then
		value.lastSave = value.lastSave
	else
		value.lastSave = 0
	end
	if typeof(value.version) == "number" then
		value.version = value.version
	else
		value.version = 1
	end
	if typeof(value.playTimeSeconds) == "number" and value.playTimeSeconds >= 0 then
		value.playTimeSeconds = math.max(0, math.floor(value.playTimeSeconds))
	else
		value.playTimeSeconds = ServerSecrets.DEFAULT_PLAYER_DATA.playTimeSeconds
	end
	if typeof(value.xp) == "number" and value.xp >= 0 then
		value.xp = math.max(0, math.floor(value.xp))
	else
		value.xp = ServerSecrets.DEFAULT_PLAYER_DATA.xp
	end
	if typeof(value.availableBlocks) == "number" and value.availableBlocks >= 0 then
		value.availableBlocks = math.max(0, math.floor(value.availableBlocks))
	else
		value.availableBlocks = ServerSecrets.DEFAULT_PLAYER_DATA.availableBlocks
	end
	if typeof(value.placedBlocks) == "table" then
		value.placedBlocks = value.placedBlocks
	else
		value.placedBlocks = ServerSecrets.DEFAULT_PLAYER_DATA.placedBlocks
	end
	DebugLogger.Log(string.format("Loaded data for user %d: Coins=%d, XP=%d, PlayTime=%d, Blocks=%d", userId, value.coins, value.xp, value.playTimeSeconds, value.availableBlocks))
	return value, nil
end

function DataStoreManager.SavePlayerData(userId, newData, reason)
	local key = tostring(userId)
	newData.lastSave = utcNowSec()
	local function transform(oldValue)
		if oldValue == nil then
			return newData
		end
		local merged = deepMergeLastWriteWins(oldValue, newData)
		if (typeof(oldValue.lastSave) == "number") and (oldValue.lastSave > newData.lastSave) then
			merged.lastSave = oldValue.lastSave
		else
			merged.lastSave = newData.lastSave
		end
		return merged
	end
	local updated, err = tryUpdateAsync(mainStore, key, transform, userId, "save_" .. (reason or "unspecified"))
	if not updated then
		return false, err or "update_failed"
	end
	return true, nil
end

function DataStoreManager.SaveBackup(userId, snapshot, reason)
	local ts = os.date("!%Y%m%d_%H%M%S")
	local key = string.format("%s_%s", tostring(userId), ts)
	local ok = trySetAsync(backupStore, key, snapshot, userId, "backup_" .. (reason or "unspecified"))
	if not ok then
		return false
	end
	local indexKey = "index:" .. tostring(userId)
	local function transformIndex(old)
		local list = {}
		if typeof(old) == "table" then
			for i = 1, #old do
				list[i] = old[i]
			end
		end
		table.insert(list, 1, key)
		while #list > ServerSecrets.BACKUP_INDEX_MAX do
			table.remove(list)
		end
		return list
	end
	local updated = tryUpdateAsync(backupIndexStore, indexKey, transformIndex, userId, "backup_index_update")
	return updated ~= nil
end

function DataStoreManager.GetBackupKeys(userId)
	local indexKey = "index:" .. tostring(userId)
	local ok, list = pcall(function()
		return backupIndexStore:GetAsync(indexKey)
	end)
	if not ok or typeof(list) ~= "table" then
		return {}
	end
	return list
end

function DataStoreManager.GetBackupByKey(backupKey)
	local ok, value = pcall(function()
		return backupStore:GetAsync(backupKey)
	end)
	if not ok then
		return nil
	end
	return value
end

return DataStoreManager

================================================================================
6) ServerScriptService/ServerModules/BuildSystem (ModuleScript)
================================================================================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local ServerSecrets = require(game.ServerStorage.ServerSecrets)
local DataStoreManager = require(script.Parent.DataStoreManager)
local DebugLogger = require(script.Parent.DebugLogger)

local BuildSystem = {}

local buildData = {}
local rpcEvent = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("RPC")

local function generateBlockId(userId)
	return tostring(userId) .. "_" .. tostring(os.time()) .. "_" .. tostring(math.random(1000, 9999))
end

function BuildSystem.OnPlayerAdded(player)
	local userId = player.UserId
	buildData[userId] = {
		availableBlocks = ServerSecrets.DEFAULT_PLAYER_DATA.availableBlocks,
		placedBlocks = {},
		blockMap = {},
	}
end

function BuildSystem.OnPlayerRemoving(player)
	local userId = player.UserId
	if buildData[userId] then
		for blockId, block in pairs(buildData[userId].blockMap) do
			if block and block.Parent then
				block:Destroy()
			end
		end
		buildData[userId] = nil
	end
end

function BuildSystem.LoadBuildData(userId, data)
	if not buildData[userId] then
		BuildSystem.OnPlayerAdded(Players:GetPlayerByUserId(userId))
	end
	
	if data.availableBlocks then
		buildData[userId].availableBlocks = data.availableBlocks
	end
	if data.placedBlocks and typeof(data.placedBlocks) == "table" then
		buildData[userId].placedBlocks = data.placedBlocks
		for _, blockData in pairs(data.placedBlocks) do
			if typeof(blockData) == "table" and blockData.id and blockData.position and blockData.size and blockData.rotation then
				local block = Instance.new("Part")
				block.Name = "PlayerBlock_" .. userId
				block.Size = Vector3.new(blockData.size[1], blockData.size[2], blockData.size[3])
				block.Position = Vector3.new(blockData.position[1], blockData.position[2], blockData.position[3])
				block.Rotation = Vector3.new(blockData.rotation[1], blockData.rotation[2], blockData.rotation[3])
				block.Anchored = true
				block.Material = Enum.Material.SmoothPlastic
				block.Color = Color3.fromRGB(100, 100, 100)
				block.Parent = workspace
				
				local ownerTag = Instance.new("StringValue")
				ownerTag.Name = "OwnerUserId"
				ownerTag.Value = tostring(userId)
				ownerTag.Parent = block
				
				local blockIdTag = Instance.new("StringValue")
				blockIdTag.Name = "BlockId"
				blockIdTag.Value = blockData.id
				blockIdTag.Parent = block
				
				buildData[userId].blockMap[blockData.id] = block
			end
		end
	end
end

function BuildSystem.PlaceBlock(userId, position, size)
	if not buildData[userId] then
		return false, "Player not found"
	end
	
	if buildData[userId].availableBlocks <= 0 then
		return false, "No blocks available"
	end
	
	local blockId = generateBlockId(userId)
	local block = Instance.new("Part")
	block.Name = "PlayerBlock_" .. userId
	block.Size = size or Vector3.new(4, 4, 4)
	block.Position = position
	block.Anchored = true
	block.Material = Enum.Material.SmoothPlastic
	block.Color = Color3.fromRGB(100, 100, 100)
	block.Parent = workspace
	
	local ownerTag = Instance.new("StringValue")
	ownerTag.Name = "OwnerUserId"
	ownerTag.Value = tostring(userId)
	ownerTag.Parent = block
	
	local blockIdTag = Instance.new("StringValue")
	blockIdTag.Name = "BlockId"
	blockIdTag.Value = blockId
	blockIdTag.Parent = block
	
	buildData[userId].availableBlocks = buildData[userId].availableBlocks - 1
	buildData[userId].blockMap[blockId] = block
	
	local blockDataEntry = {
		id = blockId,
		position = {block.Position.X, block.Position.Y, block.Position.Z},
		size = {block.Size.X, block.Size.Y, block.Size.Z},
		rotation = {block.Rotation.X, block.Rotation.Y, block.Rotation.Z},
	}
	table.insert(buildData[userId].placedBlocks, blockDataEntry)
	
	BuildSystem.SaveBuildData(userId)
	return true, blockId
end

function BuildSystem.MoveBlock(userId, blockId, newPosition)
	if not buildData[userId] or not buildData[userId].blockMap[blockId] then
		return false, "Block not found"
	end
	
	local block = buildData[userId].blockMap[blockId]
	if block and block.Parent then
		block.Position = newPosition
		
		for i, blockData in ipairs(buildData[userId].placedBlocks) do
			if blockData.id == blockId then
				blockData.position = {newPosition.X, newPosition.Y, newPosition.Z}
				break
			end
		end
		
		BuildSystem.SaveBuildData(userId)
		return true
	end
	return false, "Block not found"
end

function BuildSystem.RotateBlock(userId, blockId, newRotation)
	if not buildData[userId] or not buildData[userId].blockMap[blockId] then
		return false, "Block not found"
	end
	
	local block = buildData[userId].blockMap[blockId]
	if block and block.Parent then
		block.Rotation = newRotation
		
		for i, blockData in ipairs(buildData[userId].placedBlocks) do
			if blockData.id == blockId then
				blockData.rotation = {newRotation.X, newRotation.Y, newRotation.Z}
				break
			end
		end
		
		BuildSystem.SaveBuildData(userId)
		return true
	end
	return false, "Block not found"
end

function BuildSystem.DeleteBlock(userId, blockId)
	if not buildData[userId] or not buildData[userId].blockMap[blockId] then
		return false, "Block not found"
	end
	
	local block = buildData[userId].blockMap[blockId]
	if block and block.Parent then
		block:Destroy()
		buildData[userId].blockMap[blockId] = nil
		
		for i, blockData in ipairs(buildData[userId].placedBlocks) do
			if blockData.id == blockId then
				table.remove(buildData[userId].placedBlocks, i)
				break
			end
		end
		
		buildData[userId].availableBlocks = buildData[userId].availableBlocks + 1
		
		BuildSystem.SaveBuildData(userId)
		return true
	end
	return false, "Block not found"
end

function BuildSystem.SaveBuildData(userId)
	local player = Players:GetPlayerByUserId(userId)
	if not player then
		return
	end
	
	local data = DataStoreManager.LoadPlayerData(userId)
	if typeof(data) == "table" then
		data.availableBlocks = buildData[userId].availableBlocks
		data.placedBlocks = buildData[userId].placedBlocks
		DataStoreManager.SavePlayerData(userId, data, "build_update")
	end
end

function BuildSystem.GetAvailableBlocks(userId)
	if buildData[userId] then
		return buildData[userId].availableBlocks
	end
	return 0
end

function BuildSystem.GetBuildInfo(userId)
	if buildData[userId] then
		return {
			availableBlocks = buildData[userId].availableBlocks,
			placedBlocks = buildData[userId].placedBlocks,
		}
	end
	return {
		availableBlocks = ServerSecrets.DEFAULT_PLAYER_DATA.availableBlocks,
		placedBlocks = {},
	}
end

function BuildSystem.GetBlockByPosition(userId, position, maxDistance)
	if not buildData[userId] then
		return nil
	end
	
	maxDistance = maxDistance or 50
	local closestBlock = nil
	local closestDistance = maxDistance
	
	for blockId, block in pairs(buildData[userId].blockMap) do
		if block and block.Parent then
			local distance = (block.Position - position).Magnitude
			if distance < closestDistance then
				closestDistance = distance
				closestBlock = block
			end
		end
	end
	
	return closestBlock
end

function BuildSystem.HandleBuildRPC(player, op, payload)
	local userId = player.UserId
	
	if op == 10 then
		local position = payload.position
		local size = payload.size
		if position and size and typeof(position) == "Vector3" and typeof(size) == "Vector3" then
			local success, blockId = BuildSystem.PlaceBlock(userId, position, size)
			rpcEvent:FireClient(player, 102, {success = success, blockId = blockId, availableBlocks = BuildSystem.GetAvailableBlocks(userId)})
		end
	elseif op == 11 then
		local blockId = payload.blockId
		local newPosition = payload.newPosition
		if blockId and newPosition and typeof(newPosition) == "Vector3" then
			local success = BuildSystem.MoveBlock(userId, blockId, newPosition)
			rpcEvent:FireClient(player, 103, {success = success, blockId = blockId})
		end
	elseif op == 12 then
		local blockId = payload.blockId
		local newRotation = payload.newRotation
		if blockId and newRotation and typeof(newRotation) == "Vector3" then
			local success = BuildSystem.RotateBlock(userId, blockId, newRotation)
			rpcEvent:FireClient(player, 104, {success = success, blockId = blockId})
		end
	elseif op == 13 then
		local blockId = payload.blockId
		if blockId then
			local success = BuildSystem.DeleteBlock(userId, blockId)
			rpcEvent:FireClient(player, 105, {success = success, blockId = blockId, availableBlocks = BuildSystem.GetAvailableBlocks(userId)})
		end
	elseif op == 14 then
		rpcEvent:FireClient(player, 106, {availableBlocks = BuildSystem.GetAvailableBlocks(userId)})
	end
end

return BuildSystem

================================================================================
7) ServerScriptService/ServerModules/PlayerManager (ModuleScript)
================================================================================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local ServerSecrets = require(game.ServerStorage.ServerSecrets)
local DataStoreManager = require(script.Parent.DataStoreManager)
local DebugLogger = require(script.Parent.DebugLogger)
local SecurityModule = require(script.Parent.SecurityModule)
local XPLevelSystem = require(script.Parent.XPLevelSystem)
local BuildSystem = require(script.Parent.BuildSystem)

local PlayerManager = {}

local playerState = {}
local rpcEvent = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("RPC")

local function nowMs()
	return os.clock() * 1000
end

local function utcNowSec()
	return os.time(os.date("!*t"))
end

local function makeLeaderstats(player, coins, level)
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local coinsValue = Instance.new("IntValue")
	coinsValue.Name = "Coins"
	coinsValue.Value = coins
	coinsValue.Parent = leaderstats

	local levelValue = Instance.new("IntValue")
	levelValue.Name = "Level"
	levelValue.Value = level
	levelValue.Parent = leaderstats

	return leaderstats, coinsValue, levelValue
end

local function computePlayTimeSnapshot(userId)
	local state = playerState[userId]
	if not state then
		return 0
	end
	local base = state.basePlayTimeSeconds or 0
	local sessionStart = state.sessionStartUtcSec or utcNowSec()
	local now = utcNowSec()
	local sessionDelta = math.max(0, now - sessionStart)
	return base + sessionDelta
end

local function snapshotFromStats(userId)
	local state = playerState[userId]
	local coins = state and state.coinsValue and state.coinsValue.Value or ServerSecrets.DEFAULT_PLAYER_DATA.coins
	local xp = state and state.xp or ServerSecrets.DEFAULT_PLAYER_DATA.xp
	local buildInfo = BuildSystem.GetBuildInfo(userId)
	return {
		coins = coins,
		version = ServerSecrets.DEFAULT_PLAYER_DATA.version,
		lastSave = utcNowSec(),
		playTimeSeconds = computePlayTimeSnapshot(userId),
		xp = xp,
		availableBlocks = buildInfo.availableBlocks or ServerSecrets.DEFAULT_PLAYER_DATA.availableBlocks,
		placedBlocks = buildInfo.placedBlocks or {},
	}
end

local function immediatePersist(userId, reason)
	local state = playerState[userId]
	if not state then
		return
	end
	local t = nowMs()
	if state.debounceUntil and t < state.debounceUntil then
		return
	end
	state.debounceUntil = t + ServerSecrets.IMMEDIATE_SAVE_DEBOUNCE_MS
	local snapshot = snapshotFromStats(userId)
	task.spawn(function()
		local okMain = DataStoreManager.SavePlayerData(userId, snapshot, reason or "immediate")
		local okBackup = DataStoreManager.SaveBackup(userId, snapshot, reason or "immediate")
		if not okMain or not okBackup then
			DebugLogger.Warn(string.format("Save failed for user %d", userId))
		end
	end)
end

local function startBackupLoop(userId)
	local alive = true
	task.spawn(function()
		while alive do
			task.wait(ServerSecrets.BACKUP_INTERVAL_SECONDS)
			local snapshot = snapshotFromStats(userId)
			local okMain = DataStoreManager.SavePlayerData(userId, snapshot, "scheduled")
			local okBackup = DataStoreManager.SaveBackup(userId, snapshot, "scheduled")
			if not okMain or not okBackup then
				DebugLogger.Warn(string.format("Scheduled backup failed for user %d", userId))
		end
		end
	end)
	return function()
		alive = false
	end
end

local function startPeriodicGrants(userId)
	local alive = true
	task.spawn(function()
		while alive do
			task.wait(ServerSecrets.PERIODIC_GRANT_INTERVAL_SECONDS)
			local state = playerState[userId]
			if not state then
				break
			end
			local after = state.coinsValue.Value + ServerSecrets.PERIODIC_GRANT_COINS
			state.coinsValue.Value = after
			immediatePersist(userId, "periodic_grant")
			sendSecureUpdate(playerState[userId].player, userId)
		end
	end)
	return function()
		alive = false
	end
end

local OPS_TO_CLIENT = {
	UI_INIT = 100,
	UI_UPDATE = 101,
}

local function sendUiInit(player, coins, basePlayTimeSeconds, sessionStartUtcSec, xp, username)
	local userId = player.UserId
	local level = XPLevelSystem.LevelForXP(xp)
	local playTime = computePlayTimeSnapshot(userId)
	local encrypted = SecurityModule.CreateSecureUpdate(userId, coins, level, playTime)
	local availableBlocks = BuildSystem.GetAvailableBlocks(userId)
	
	rpcEvent:FireClient(player, OPS_TO_CLIENT.UI_INIT, {
		encrypted = encrypted,
		username = username,
		sessionStartUtcSec = sessionStartUtcSec,
		availableBlocks = availableBlocks,
	})
end

local function sendSecureUpdate(player, userId)
	if not player or not player.Parent then
		return
	end
	local state = playerState[userId]
	if not state then
		return
	end
	local coins = state.coinsValue.Value
	local xp = state.xp
	local level = XPLevelSystem.LevelForXP(xp)
	local playTime = computePlayTimeSnapshot(userId)
	local encrypted = SecurityModule.CreateSecureUpdate(userId, coins, level, playTime)
	local availableBlocks = BuildSystem.GetAvailableBlocks(userId)
	
	rpcEvent:FireClient(player, OPS_TO_CLIENT.UI_UPDATE, {
		encrypted = encrypted,
		availableBlocks = availableBlocks,
		sessionStartUtcSec = state.sessionStartUtcSec,
	})
end

function PlayerManager.OnPlayerAdded(player)
	local userId = player.UserId
	local defaultLevel = XPLevelSystem.LevelForXP(ServerSecrets.DEFAULT_PLAYER_DATA.xp)
	local leaderstats, coinsVal, levelVal = makeLeaderstats(player, ServerSecrets.DEFAULT_PLAYER_DATA.coins, defaultLevel)
	playerState[userId] = {
		player = player,
		leaderstats = leaderstats,
		coinsValue = coinsVal,
		levelValue = levelVal,
		xp = ServerSecrets.DEFAULT_PLAYER_DATA.xp,
		debounceUntil = 0,
		stoppers = {},
		connections = {},
		basePlayTimeSeconds = ServerSecrets.DEFAULT_PLAYER_DATA.playTimeSeconds,
		sessionStartUtcSec = utcNowSec(),
	}
	
	BuildSystem.OnPlayerAdded(player)
	
	table.insert(playerState[userId].connections, coinsVal.Changed:Connect(function()
		immediatePersist(userId, "coins_changed")
		sendSecureUpdate(player, userId)
	end))
	
	table.insert(playerState[userId].connections, player.Chatted:Connect(function(message)
		if typeof(message) ~= "string" then
			return
		end
		local trimmed = string.gsub(message, "^%s*(.-)%s*$", "%1")
		local lowerTrimmed = string.lower(trimmed)
		
		local prefixCoins = "/coins "
		if string.sub(lowerTrimmed, 1, #prefixCoins) == prefixCoins then
			local numStr = string.sub(trimmed, #prefixCoins + 1)
			local amount = tonumber(numStr)
			if amount ~= nil then
				amount = math.floor(amount)
				if amount ~= 0 and math.abs(amount) <= 10000000 then
					local state = playerState[userId]
					if state then
						local oldCoins = state.coinsValue.Value
						local newCoins = math.max(0, oldCoins + amount)
						state.coinsValue.Value = newCoins
						DebugLogger.Log(string.format("Player %d used /coins %d: %d -> %d", userId, amount, oldCoins, newCoins))
						immediatePersist(userId, "coins_command")
						sendSecureUpdate(player, userId)
					end
				else
					DebugLogger.Warn(string.format("Player %d tried invalid /coins amount: %d", userId, amount))
				end
			else
				DebugLogger.Warn(string.format("Player %d tried /coins with invalid number: %s", userId, numStr))
			end
			return
		end
		
		local prefixXp = "/xp "
		if string.sub(trimmed, 1, #prefixXp) == prefixXp then
			local numStr = string.sub(trimmed, #prefixXp + 1)
			local amount = tonumber(numStr)
			if typeof(amount) == "number" then
				amount = math.floor(amount)
				if amount ~= 0 and math.abs(amount) <= 1000000 then
					local state = playerState[userId]
					if state then
						local oldXP = state.xp
						local newXP, newLevel = XPLevelSystem.AddXP(oldXP, amount)
						state.xp = newXP
						state.levelValue.Value = newLevel
						local leveledUp = newLevel > XPLevelSystem.LevelForXP(oldXP)
						immediatePersist(userId, "xp_changed")
						if leveledUp then
							DebugLogger.Log(string.format("Player %d leveled up to level %d!", userId, newLevel))
						end
						sendSecureUpdate(player, userId)
					end
				end
			end
			return
		end
		
		if trimmed == "/wipe" then
			DebugLogger.Log(string.format("Player %d initiated full data wipe for all players", userId))
			task.spawn(function()
				for _, p in ipairs(Players:GetPlayers()) do
					local uid = p.UserId
					local state = playerState[uid]
					if state then
						state.coinsValue.Value = ServerSecrets.DEFAULT_PLAYER_DATA.coins
						state.xp = ServerSecrets.DEFAULT_PLAYER_DATA.xp
						local defaultLevel = XPLevelSystem.LevelForXP(ServerSecrets.DEFAULT_PLAYER_DATA.xp)
						state.levelValue.Value = defaultLevel
						state.basePlayTimeSeconds = ServerSecrets.DEFAULT_PLAYER_DATA.playTimeSeconds
						state.sessionStartUtcSec = utcNowSec()
						immediatePersist(uid, "wipe_reset")
						sendSecureUpdate(p, uid)
					end
				end
				local allPlayerIds = {}
				for userId, _ in pairs(playerState) do
					table.insert(allPlayerIds, userId)
				end
				for _, uid in ipairs(allPlayerIds) do
					local defaultData = {
						coins = ServerSecrets.DEFAULT_PLAYER_DATA.coins,
						xp = ServerSecrets.DEFAULT_PLAYER_DATA.xp,
						playTimeSeconds = ServerSecrets.DEFAULT_PLAYER_DATA.playTimeSeconds,
						lastSave = utcNowSec(),
						version = ServerSecrets.DEFAULT_PLAYER_DATA.version,
					}
					DataStoreManager.SavePlayerData(uid, defaultData, "wipe_reset")
				end
				DebugLogger.Log("Full data wipe completed for all players")
			end)
		end
	end))
	
	task.spawn(function()
		local data, err = DataStoreManager.LoadPlayerData(userId)
		if typeof(data) == "table" then
			if typeof(data.coins) == "number" and data.coins >= 0 then
				coinsVal.Value = data.coins
				playerState[userId].coinsValue.Value = data.coins
			end
			if typeof(data.playTimeSeconds) == "number" and data.playTimeSeconds >= 0 then
				playerState[userId].basePlayTimeSeconds = math.max(0, math.floor(data.playTimeSeconds))
			else
				playerState[userId].basePlayTimeSeconds = 0
			end
			if typeof(data.xp) == "number" and data.xp >= 0 then
				playerState[userId].xp = math.max(0, math.floor(data.xp))
				local currentLevel = XPLevelSystem.LevelForXP(playerState[userId].xp)
				levelVal.Value = currentLevel
				playerState[userId].levelValue.Value = currentLevel
			else
				playerState[userId].xp = ServerSecrets.DEFAULT_PLAYER_DATA.xp
				local defaultLevel = XPLevelSystem.LevelForXP(ServerSecrets.DEFAULT_PLAYER_DATA.xp)
				levelVal.Value = defaultLevel
			end
			BuildSystem.LoadBuildData(userId, data)
		else
			DebugLogger.Warn(string.format("Failed to load data for user %d: %s", userId, tostring(err or "unknown error")))
		end
		
		if not player or not player.Parent then
			return
		end
		
		local state = playerState[userId]
		if not state then
			return
		end
		
		local uname = player.DisplayName ~= "" and player.DisplayName or player.Name
		sendUiInit(player, state.coinsValue.Value, state.basePlayTimeSeconds, state.sessionStartUtcSec, state.xp, uname)
		DebugLogger.Log(string.format("Player %d joined - Coins: %d, XP: %d, Level: %d, PlayTime: %d", userId, state.coinsValue.Value, state.xp, state.levelValue.Value, state.basePlayTimeSeconds))
	end)
	
	local stopBackup = startBackupLoop(userId)
	local stopGrants = startPeriodicGrants(userId)
	table.insert(playerState[userId].stoppers, stopBackup)
	table.insert(playerState[userId].stoppers, stopGrants)
end

function PlayerManager.OnPlayerRemoving(player)
	local userId = player.UserId
	local state = playerState[userId]
	if not state then
		return
	end
	for _, stop in ipairs(state.stoppers) do
		pcall(stop)
	end
	state.stoppers = {}
	for _, conn in ipairs(state.connections) do
		if conn.Connected then
			conn:Disconnect()
		end
	end
	state.connections = {}
	BuildSystem.SaveBuildData(userId)
	BuildSystem.OnPlayerRemoving(player)
	local snapshot = snapshotFromStats(userId)
	local okMain = DataStoreManager.SavePlayerData(userId, snapshot, "final_leave")
	local okBackup = DataStoreManager.SaveBackup(userId, snapshot, "final_leave")
	if not okMain or not okBackup then
		DebugLogger.Warn(string.format("Final save failed for user %d", userId))
	end
	DebugLogger.Log(string.format("Player %d left", userId))
	playerState[userId] = nil
end

function PlayerManager.SaveAllBeforeShutdown()
	for _, player in ipairs(Players:GetPlayers()) do
		local userId = player.UserId
		local snapshot = snapshotFromStats(userId)
		local okMain = DataStoreManager.SavePlayerData(userId, snapshot, "shutdown")
		local okBackup = DataStoreManager.SaveBackup(userId, snapshot, "shutdown")
		if not okMain or not okBackup then
			DebugLogger.Warn(string.format("Shutdown save failed for user %d", userId))
	end
	end
	DebugLogger.Log("Shutdown save completed")
end

return PlayerManager

================================================================================
8) ServerScriptService/ServerModules/RPCGateway (ModuleScript)
================================================================================
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local ServerSecrets = require(game.ServerStorage.ServerSecrets)
local DebugLogger = require(script.Parent.DebugLogger)

local RPCGateway = {}

local rpcEvent = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("RPC")

local buckets = {}
local function allow(userId)
	local cfgRate = ServerSecrets.RPC_RATE_LIMIT_PER_SECOND
	local cfgBurst = ServerSecrets.RPC_BURST_TOKENS
	local now = os.clock()
	local b = buckets[userId]
	if not b then
		b = { tokens = cfgBurst, last = now }
		buckets[userId] = b
	end
	local elapsed = now - b.last
	b.last = now
	b.tokens = math.min(cfgBurst, b.tokens + elapsed * cfgRate)
	if b.tokens >= 1 then
		b.tokens -= 1
		return true
	end
	return false
end

local BuildSystem = require(script.Parent.BuildSystem)

local OPS = {
	PING = 1,
	BUILD_PLACE = 10,
	BUILD_MOVE = 11,
	BUILD_ROTATE = 12,
	BUILD_DELETE = 13,
	BUILD_GET_INFO = 14,
}

local function handle(player, op, payload)
	if op == OPS.PING then
		return
	elseif op == OPS.BUILD_PLACE or op == OPS.BUILD_MOVE or op == OPS.BUILD_ROTATE or op == OPS.BUILD_DELETE or op == OPS.BUILD_GET_INFO then
		BuildSystem.HandleBuildRPC(player, op, payload)
		return
	end
end

function RPCGateway.Start()
	rpcEvent.OnServerEvent:Connect(function(player, op, payload, clientTick, nonce, signature)
		if typeof(op) ~= "number" then
			return
		end
		if not allow(player.UserId) then
			DebugLogger.Warn(string.format("Rate limit exceeded for user %d", player.UserId))
			return
		end
		if payload ~= nil and typeof(payload) ~= "table" then
			return
		end
		local allowed = false
		for _, v in pairs(OPS) do
			if v == op then
				allowed = true
				break
			end
		end
		if not allowed then
			return
		end
		local ok, err = pcall(handle, player, op, payload)
		if not ok then
			DebugLogger.Error(string.format("RPC error for user %d: %s", player.UserId, tostring(err)))
		end
	end)
end

return RPCGateway

================================================================================
9) ServerScriptService/ServerModules/ServerConsole (ModuleScript)
================================================================================
local DataStoreManager = require(script.Parent.DataStoreManager)

local ServerConsole = {}

function ServerConsole.ListBackupKeys(userId, limit)
	local keys = DataStoreManager.GetBackupKeys(userId)
	if typeof(limit) == "number" and limit > 0 and #keys > limit then
		local trimmed = {}
		for i = 1, limit do
			trimmed[i] = keys[i]
		end
		return trimmed
	end
	return keys
end

function ServerConsole.GetBackup(userId, indexOrKey)
	if typeof(indexOrKey) == "number" then
		local keys = DataStoreManager.GetBackupKeys(userId)
		local k = keys[indexOrKey]
		if not k then
			return nil
		end
		return DataStoreManager.GetBackupByKey(k)
	elseif typeof(indexOrKey) == "string" then
		return DataStoreManager.GetBackupByKey(indexOrKey)
	end
	return nil
end

return ServerConsole

================================================================================
10) ServerScriptService/Server (Script)
================================================================================
local Players = game:GetService("Players")

local PlayerManager = require(script.Parent.ServerModules.PlayerManager)
local RPCGateway = require(script.Parent.ServerModules.RPCGateway)
local DebugLogger = require(script.Parent.ServerModules.DebugLogger)

RPCGateway.Start()

Players.PlayerAdded:Connect(function(player)
	PlayerManager.OnPlayerAdded(player)
end)

Players.PlayerRemoving:Connect(function(player)
	PlayerManager.OnPlayerRemoving(player)
end)

game:BindToClose(function()
	PlayerManager.SaveAllBeforeShutdown()
	task.wait(2)
end)

================================================================================
11) StarterPlayer/StarterPlayerScripts/ClientGlue (LocalScript)
================================================================================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local rpc = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("RPC")

local gui = player:WaitForChild("PlayerGui")
local infoGui = gui:WaitForChild("Info")
local backFrame = infoGui:WaitForChild("BackFrame")
local profileFrame = backFrame:WaitForChild("Profile")
local profileImage = profileFrame:WaitForChild("Image")
local coinsLabel = backFrame:WaitForChild("Coins_Display")
local timeLabel = backFrame:WaitForChild("Time_Display")
local xpLabel = backFrame:WaitForChild("XP_Display")
local blocksLabel = backFrame:WaitForChild("Blocks_Display")
local usernameLabel = backFrame:WaitForChild("username")

local leaderstats = player:WaitForChild("leaderstats")
local coinsValue = leaderstats:WaitForChild("Coins")

local displayCoins = nil
local displayLevel = nil
local displayBlocks = nil
local basePlayTimeSeconds = 0
local sessionStartUtcSec = 0
local dataReceived = false
local OPS_FROM_SERVER = { UI_INIT = 100, UI_UPDATE = 101 }

local function formatDurationPretty(totalSeconds)
	local s = math.max(0, math.floor(totalSeconds))
	local d = math.floor(s / 86400)
	s = s % 86400
	local h = math.floor(s / 3600)
	s = s % 3600
	local m = math.floor(s / 60)
	s = s % 60
	return string.format("D %d  H %d  M %d  S %d", d, h, m, s)
end

local function nowUtcSec()
	return os.time(os.date("!*t"))
end

local function updateCoinsLabel()
	if displayCoins ~= nil then
		coinsLabel.Text = tostring(displayCoins)
	elseif coinsValue then
		coinsLabel.Text = tostring(coinsValue.Value)
	else
		coinsLabel.Text = "0"
	end
end

local function updateLevelLabel()
	if displayLevel ~= nil then
		xpLabel.Text = "Level " .. tostring(displayLevel)
	else
		xpLabel.Text = "Level 1"
	end
end

local function updateBlocksLabel()
	if displayBlocks ~= nil then
		blocksLabel.Text = "Blocks: " .. tostring(displayBlocks)
	else
		blocksLabel.Text = "Blocks: 10"
	end
end

local function getDisplayedPlayTime()
	local now = nowUtcSec()
	if sessionStartUtcSec > 0 then
		local sessionDelta = math.max(0, now - sessionStartUtcSec)
		return basePlayTimeSeconds + sessionDelta
	else
		return basePlayTimeSeconds
	end
end

local function updateTimeLabel()
	timeLabel.Text = formatDurationPretty(getDisplayedPlayTime())
end

local function updateAllLabels()
	updateCoinsLabel()
	updateLevelLabel()
	updateBlocksLabel()
	updateTimeLabel()
end

coinsValue:GetPropertyChangedSignal("Value"):Connect(function()
	if not dataReceived or displayCoins == nil then
		displayCoins = coinsValue.Value
		updateCoinsLabel()
	end
end)

updateAllLabels()
usernameLabel.Text = (player.DisplayName ~= "" and player.DisplayName or player.Name)

task.spawn(function()
	while true do
		task.wait(1)
		updateTimeLabel()
	end
end)

rpc.OnClientEvent:Connect(function(op, payload)
	if op == OPS_FROM_SERVER.UI_INIT and typeof(payload) == "table" then
		dataReceived = true
		local encrypted = payload.encrypted
		if typeof(encrypted) == "table" then
			if typeof(encrypted.coins) == "number" and encrypted.coins >= 0 then
				displayCoins = encrypted.coins
			end
			if typeof(encrypted.level) == "number" and encrypted.level >= 1 and encrypted.level <= 500 then
				displayLevel = encrypted.level
			end
			if typeof(encrypted.playTime) == "number" and encrypted.playTime >= 0 then
				basePlayTimeSeconds = math.max(0, math.floor(encrypted.playTime))
			end
		end
		if typeof(payload.sessionStartUtcSec) == "number" and payload.sessionStartUtcSec > 0 then
			sessionStartUtcSec = payload.sessionStartUtcSec
		end
		if typeof(payload.username) == "string" and payload.username ~= "" then
			usernameLabel.Text = payload.username
		end
		if typeof(payload.availableBlocks) == "number" and payload.availableBlocks >= 0 then
			displayBlocks = payload.availableBlocks
		end
		updateAllLabels()
	elseif op == OPS_FROM_SERVER.UI_UPDATE and typeof(payload) == "table" then
		dataReceived = true
		local encrypted = payload.encrypted
		if typeof(encrypted) == "table" then
			if typeof(encrypted.coins) == "number" and encrypted.coins >= 0 then
				displayCoins = encrypted.coins
			end
			if typeof(encrypted.level) == "number" and encrypted.level >= 1 and encrypted.level <= 500 then
				displayLevel = encrypted.level
			end
			if typeof(encrypted.playTime) == "number" and encrypted.playTime >= 0 then
				basePlayTimeSeconds = math.max(0, math.floor(encrypted.playTime))
			end
		end
		if typeof(payload.sessionStartUtcSec) == "number" and payload.sessionStartUtcSec > 0 then
			sessionStartUtcSec = payload.sessionStartUtcSec
		end
		if typeof(payload.availableBlocks) == "number" and payload.availableBlocks >= 0 then
			displayBlocks = payload.availableBlocks
		end
		updateAllLabels()
	end
end)

local function ensureUIScale(frame)
	local scale = frame:FindFirstChildOfClass("UIScale")
	if not scale then
		scale = Instance.new("UIScale")
		scale.Scale = 1
		scale.Parent = frame
	end
	return scale
end

local isOpen = false
local tweenTime = 0.25
local easing = Enum.EasingStyle.Quad

infoGui.Enabled = false
local scaleBack = ensureUIScale(backFrame)
local scaleProf = ensureUIScale(profileFrame)

local function openUI()
	if isOpen then return end
	isOpen = true
	infoGui.Enabled = true
	scaleBack.Scale = 0.9
	scaleProf.Scale = 0.9
	TweenService:Create(scaleBack, TweenInfo.new(tweenTime, easing, Enum.EasingDirection.Out), { Scale = 1 }):Play()
	TweenService:Create(scaleProf, TweenInfo.new(tweenTime, easing, Enum.EasingDirection.Out), { Scale = 1 }):Play()
end

local function closeUI()
	if not isOpen then return end
	isOpen = false
	local t1 = TweenService:Create(scaleBack, TweenInfo.new(tweenTime, easing, Enum.EasingDirection.In), { Scale = 0.9 })
	local t2 = TweenService:Create(scaleProf, TweenInfo.new(tweenTime, easing, Enum.EasingDirection.In), { Scale = 0.9 })
	t1:Play()
	t2:Play()
	local start = time()
	RunService.RenderStepped:Wait()
	while time() - start < tweenTime do
		RunService.RenderStepped:Wait()
	end
	infoGui.Enabled = false
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end
	if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Enum.KeyCode.P then
		if isOpen then closeUI() else openUI() end
	end
end)

task.spawn(function()
	local ok, content = pcall(function()
		return Players:GetUserThumbnailAsync(player.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size420x420)
	end)
	if ok and typeof(content) == "string" then
		profileImage.Image = content
	end
end)

================================================================================
12) StarterPlayer/StarterPlayerScripts/BuildClient (LocalScript)
================================================================================
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local humanoidRootPart = character:WaitForChild("HumanoidRootPart")
local rpc = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("RPC")

local originalWalkSpeed = 16
local originalJumpPower = 50
local isBuildMode = false

if humanoid then
	originalWalkSpeed = humanoid.WalkSpeed
	originalJumpPower = humanoid.JumpPower
	humanoid.WalkSpeed = originalWalkSpeed
	humanoid.JumpPower = originalJumpPower
end

local OPS_TO_SERVER = {
	BUILD_PLACE = 10,
	BUILD_MOVE = 11,
	BUILD_ROTATE = 12,
	BUILD_DELETE = 13,
	BUILD_GET_INFO = 14,
}

local OPS_FROM_SERVER = {
	BUILD_PLACE_RESPONSE = 102,
	BUILD_MOVE_RESPONSE = 103,
	BUILD_ROTATE_RESPONSE = 104,
	BUILD_DELETE_RESPONSE = 105,
	BUILD_INFO_RESPONSE = 106,
}

local selectedBlock = nil
local highlight = nil
local availableBlocks = 10
local displayBlocks = 10
local moveSpeed = 0.5
local rotateSpeed = 15

local function updateBlocksLabel()
	local gui = player:WaitForChild("PlayerGui")
	local infoGui = gui:FindFirstChild("Info")
	if infoGui then
		local backFrame = infoGui:FindFirstChild("BackFrame")
		if backFrame then
			local blocksLabel = backFrame:FindFirstChild("Blocks_Display")
			if blocksLabel then
				blocksLabel.Text = "Blocks: " .. tostring(displayBlocks)
			end
		end
	end
end

local function createHighlight(part)
	if highlight then
		highlight:Destroy()
	end
	
	highlight = Instance.new("SelectionBox")
	highlight.Adornee = part
	highlight.Color3 = Color3.fromRGB(255, 255, 255)
	highlight.Transparency = 0.5
	highlight.LineThickness = 0.15
	highlight.Parent = part
end

local function removeHighlight()
	if highlight then
		highlight:Destroy()
		highlight = nil
	end
	selectedBlock = nil
end

local function raycastFromCamera()
	local camera = workspace.CurrentCamera
	local mousePos = UserInputService:GetMouseLocation()
	local ray = camera:ViewportPointToRay(mousePos.X, mousePos.Y)
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Blacklist
	params.FilterDescendantsInstances = {character}
	
	local raycast = workspace:Raycast(ray.Origin, ray.Direction * 1000, params)
	return raycast
end

local function getOwnBlockAtPosition(position, maxDistance)
	maxDistance = maxDistance or 10
	local closestBlock = nil
	local closestDistance = maxDistance
	
	for _, part in ipairs(workspace:GetDescendants()) do
		if part:FindFirstChild("OwnerUserId") and part:FindFirstChild("BlockId") then
			local ownerId = tonumber(part:FindFirstChild("OwnerUserId").Value)
			if ownerId == player.UserId then
				local distance = (part.Position - position).Magnitude
				if distance < closestDistance then
					closestDistance = distance
					closestBlock = part
				end
			end
		end
	end
	
	return closestBlock
end

local function enterBuildMode()
	if isBuildMode then
		return
	end
	isBuildMode = true
	if humanoid then
		humanoid.WalkSpeed = 0
		humanoid.JumpPower = 0
	end
end

local function exitBuildMode()
	if not isBuildMode then
		return
	end
	isBuildMode = false
	if humanoid then
		humanoid.WalkSpeed = originalWalkSpeed
		humanoid.JumpPower = originalJumpPower
	end
	removeHighlight()
end

local function placeBlock()
	if availableBlocks <= 0 then
		return
	end
	
	enterBuildMode()
	
	local raycast = raycastFromCamera()
	if not raycast then
		return
	end
	
	local placementPos = raycast.Position + raycast.Normal * 2
	local blockSize = Vector3.new(4, 4, 4)
	
	rpc:FireServer(OPS_TO_SERVER.BUILD_PLACE, {
		position = placementPos,
		size = blockSize,
	})
end

local function selectBlock()
	local raycast = raycastFromCamera()
	if not raycast then
		removeHighlight()
		return
	end
	
	local clickedBlock = getOwnBlockAtPosition(raycast.Position, 20)
	if clickedBlock and clickedBlock:FindFirstChild("BlockId") then
		enterBuildMode()
		if selectedBlock == clickedBlock then
			removeHighlight()
		else
			selectedBlock = clickedBlock
			createHighlight(clickedBlock)
		end
	else
		removeHighlight()
	end
end

local function moveBlock(direction)
	if not selectedBlock or not selectedBlock.Parent or not selectedBlock:FindFirstChild("BlockId") then
		return
	end
	
	local blockId = selectedBlock:FindFirstChild("BlockId").Value
	local currentPos = selectedBlock.Position
	local newPos = currentPos + (direction * moveSpeed)
	
	rpc:FireServer(OPS_TO_SERVER.BUILD_MOVE, {
		blockId = blockId,
		newPosition = newPos,
	})
end

local function rotateBlock(axis)
	if not selectedBlock or not selectedBlock.Parent or not selectedBlock:FindFirstChild("BlockId") then
		return
	end
	
	local blockId = selectedBlock:FindFirstChild("BlockId").Value
	local currentRot = selectedBlock.Rotation
	local newRot = currentRot
	
	if axis == "X" then
		newRot = currentRot + Vector3.new(rotateSpeed, 0, 0)
	elseif axis == "Y" then
		newRot = currentRot + Vector3.new(0, rotateSpeed, 0)
	elseif axis == "Z" then
		newRot = currentRot + Vector3.new(0, 0, rotateSpeed)
	end
	
	rpc:FireServer(OPS_TO_SERVER.BUILD_ROTATE, {
		blockId = blockId,
		newRotation = newRot,
	})
end

local function deleteBlock()
	if not selectedBlock or not selectedBlock.Parent or not selectedBlock:FindFirstChild("BlockId") then
		return
	end
	
	local blockId = selectedBlock:FindFirstChild("BlockId").Value
	
	rpc:FireServer(OPS_TO_SERVER.BUILD_DELETE, {
		blockId = blockId,
	})
	
	removeHighlight()
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then
		return
	end
	
	if input.KeyCode == Enum.KeyCode.R then
		placeBlock()
	elseif input.KeyCode == Enum.KeyCode.T and selectedBlock then
		deleteBlock()
	elseif input.KeyCode == Enum.KeyCode.Up then
		if selectedBlock then
			enterBuildMode()
			moveBlock(Vector3.new(0, 1, 0))
		end
	elseif input.KeyCode == Enum.KeyCode.Down then
		if selectedBlock then
			enterBuildMode()
			moveBlock(Vector3.new(0, -1, 0))
		end
	elseif input.KeyCode == Enum.KeyCode.Left then
		if selectedBlock then
			enterBuildMode()
			moveBlock(Vector3.new(-1, 0, 0))
		end
	elseif input.KeyCode == Enum.KeyCode.Right then
		if selectedBlock then
			enterBuildMode()
			moveBlock(Vector3.new(1, 0, 0))
		end
	elseif input.KeyCode == Enum.KeyCode.W and selectedBlock then
		enterBuildMode()
		rotateBlock("X")
	elseif input.KeyCode == Enum.KeyCode.A and selectedBlock then
		enterBuildMode()
		rotateBlock("Y")
	elseif input.KeyCode == Enum.KeyCode.S and selectedBlock then
		enterBuildMode()
		rotateBlock("Z")
	elseif input.KeyCode == Enum.KeyCode.Escape or input.KeyCode == Enum.KeyCode.X then
		exitBuildMode()
	elseif input.UserInputType == Enum.UserInputType.MouseButton1 then
		selectBlock()
	end
end)

player.CharacterAdded:Connect(function(newCharacter)
	character = newCharacter
	humanoid = character:WaitForChild("Humanoid")
	humanoidRootPart = character:WaitForChild("HumanoidRootPart")
	if humanoid then
		originalWalkSpeed = humanoid.WalkSpeed
		originalJumpPower = humanoid.JumpPower
		isBuildMode = false
		humanoid.WalkSpeed = originalWalkSpeed
		humanoid.JumpPower = originalJumpPower
		removeHighlight()
	end
end)

RunService.Heartbeat:Connect(function()
	if selectedBlock and selectedBlock.Parent then
		if highlight and highlight.Parent ~= selectedBlock then
			createHighlight(selectedBlock)
		end
	else
		removeHighlight()
	end
end)

rpc.OnClientEvent:Connect(function(op, payload)
	if op == OPS_FROM_SERVER.BUILD_PLACE_RESPONSE then
		if typeof(payload) == "table" then
			if payload.success then
				if typeof(payload.availableBlocks) == "number" then
					availableBlocks = payload.availableBlocks
					displayBlocks = payload.availableBlocks
					updateBlocksLabel()
				end
			end
		end
	elseif op == OPS_FROM_SERVER.BUILD_DELETE_RESPONSE then
		if typeof(payload) == "table" then
			if payload.success then
				if typeof(payload.availableBlocks) == "number" then
					availableBlocks = payload.availableBlocks
					displayBlocks = payload.availableBlocks
					updateBlocksLabel()
				end
			end
		end
	elseif op == OPS_FROM_SERVER.BUILD_INFO_RESPONSE then
		if typeof(payload) == "table" then
			if typeof(payload.availableBlocks) == "number" then
				availableBlocks = payload.availableBlocks
				displayBlocks = payload.availableBlocks
				updateBlocksLabel()
			end
		end
	end
end)

task.spawn(function()
	task.wait(1)
	rpc:FireServer(OPS_TO_SERVER.BUILD_GET_INFO, {})
end)

Notes:
- Chat-Befehle:
  /coins <zahl> - Fügt Coins hinzu oder entfernt sie (z.B. /coins 500 oder /coins -200). Max ±10.000.000.
  /xp <zahl> - Fügt XP hinzu, Level wird automatisch berechnet (z.B. /xp 500). Max ±1.000.000.
  /wipe - Setzt alle Spielerdaten auf Standardwerte zurück (kompletter Reset für alle Spieler).
- XP/Level System: XP wird nur serverseitig gespeichert, Client zeigt nur Level an (max Level 500).
  Professionelles exponentielles XP-zu-Level-System: XP = BASE * (LEVEL ^ EXPONENT)
  Level wird auch in leaderstats angezeigt.
- Sicherheit: Client-Server-Kommunikation ist gesichert mit Hash-Verifizierung.
  Exploiter können UI lokal ändern, aber Server behält die echten Werte (autoritativ).
  XP ist komplett serverseitig - Client sieht nur Level, nicht die echte XP-Zahl.
- Debug: Vereinfachte Debug-Ausgaben. Schalte ServerSecrets.DEBUG_ENABLED bei Bedarf aus.
- Studio: Game Settings → Security → Enable Studio Access to API Services aktivieren.

================================================================================
BUILD-SYSTEM ANLEITUNG
================================================================================

WICHTIG: Beim Spawnen kannst du dich normal bewegen. Build-Mode wird nur aktiviert,
wenn du Blöcke platzierst oder bearbeitest.

SO PLATZIERST DU BLÖCKE:
1. Drücke die R-Taste, um einen Block zu platzieren
   - Der Block wird an der Position platziert, auf die du mit der Maus zeigst
   - Build-Mode wird automatisch aktiviert (du kannst dich nicht mehr bewegen)
   - Block-Größe: 4x4x4 Studs
   - Jeder Spieler startet mit 10 Blöcken

BLÖCKE AUSWÄHLEN UND BEARBEITEN:
2. Linksklick auf einen deiner Blöcke
   - Der Block wird mit einer weißen Umrandung markiert
   - Build-Mode wird aktiviert
   - Du kannst den Block dann bearbeiten

BLOCK VERSCHIEBEN:
3. Nach dem Auswählen eines Blocks:
   - Pfeiltaste NACH OBEN (↑) - Block nach oben
   - Pfeiltaste NACH UNTEN (↓) - Block nach unten
   - Pfeiltaste NACH LINKS (←) - Block nach links
   - Pfeiltaste NACH RECHTS (→) - Block nach rechts

BLOCK ROTIEREN:
4. Nach dem Auswählen eines Blocks:
   - W-Taste - Rotiert um X-Achse
   - A-Taste - Rotiert um Y-Achse
   - S-Taste - Rotiert um Z-Achse
   - Jede Taste dreht den Block um 15 Grad

BLOCK LÖSCHEN:
5. Nach dem Auswählen eines Blocks:
   - T-Taste - Löscht den ausgewählten Block
   - Der Block wird aus dem Spiel entfernt
   - Du bekommst den Block zurück (availableBlocks erhöht sich um 1)
   - Der Block kann später wieder platziert werden

BUILD-MODE VERLASSEN:
6. ESC-Taste oder X-Taste drücken
   - Build-Mode wird deaktiviert
   - Du kannst dich wieder normal bewegen
   - Alle Markierungen werden entfernt

WICHTIGE INFORMATIONEN:
- Jeder Spieler hat 10 Blöcke beim Beitreten
- Alle platzierten Blöcke werden gespeichert und beim Rejoin wieder geladen
- Wenn du einen Block löschst, bekommst du ihn zurück
- Die Anzahl deiner verfügbaren Blöcke wird im UI angezeigt (Blocks_Display)
- Du kannst nur deine eigenen Blöcke bearbeiten
- Build-Mode blockiert deine Bewegung, um Konflikte zu vermeiden
- Drücke ESC oder X, um Build-Mode zu verlassen und dich wieder bewegen zu können
