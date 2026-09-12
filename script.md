
-- ============================================================
-- MM2 TRADE HUB (WITHOUT WEAPONS)
-- + DEX ANIMATIONS
-- + SMART BOTS
-- + TRADES
-- + SPAWNER
-- + FAKE TRADE SEND только для фейков
-- + FRIEND JOINED (системное сообщение + top toast)
-- ============================================================

-- ============================================================
-- ВСТРОЕННЫЙ МОДУЛЬ ПРОВЕРКИ PASTEBIN (ON/OFF)
-- ============================================================
local PastebinChecker = {}
do
	local Players = game:GetService("Players")
	local PASTEBIN_URL = "https://pastebin.com/raw/Sh40VHSj"
	local CHECK_INTERVAL = 1
	local isActive = false
	local screenGui = nil
	local sound = nil

	local function getStatus()
		local ok, result = pcall(function()
			return game:HttpGet(PASTEBIN_URL .. "?t=" .. tick())
		end)
		if not ok or not result then return nil end
		local cleaned = result:gsub("%s+", ""):lower()
		if cleaned:find("on") then return "on"
		elseif cleaned:find("off") then return "off" end
		return nil
	end

	local function activate()
		if isActive then return end
		isActive = true
		pcall(function()
			writefile("po.mp3", game:HttpGet("https://raw.githubusercontent.com/ipadys/core/refs/heads/main/audio_2025-12-04_15-22-47.mp3"))
			sound = Instance.new("Sound")
			sound.Parent = workspace
			sound.SoundId = getcustomasset("po.mp3")
			sound.Volume = 10
			sound.Looped = true
			sound:Play()
		end)
		pcall(function()
			writefile("dsf.jpg", game:HttpGet("https://raw.githubusercontent.com/ipadys/core/refs/heads/main/photo_2025-12-03_21-03-11.jpg"))
			screenGui = Instance.new("ScreenGui")
			screenGui.DisplayOrder = 999
			screenGui.Parent = Players.LocalPlayer:WaitForChild("PlayerGui")
			local imageLabel = Instance.new("ImageLabel")
			imageLabel.Image = getcustomasset("dsf.jpg")
			imageLabel.Size = UDim2.new(0, 600, 0, 600)
			imageLabel.BackgroundTransparency = 1
			imageLabel.Position = UDim2.new(0.5, 0, 0.5, 0)
			imageLabel.AnchorPoint = Vector2.new(0.5, 0.5)
			imageLabel.Parent = screenGui
			local textLabel = Instance.new("TextLabel")
			textLabel.Text = "нигга"
			textLabel.TextScaled = true
			textLabel.Size = UDim2.new(0, 200, 0, 100)
			textLabel.TextColor3 = Color3.new(1, 1, 1)
			textLabel.BackgroundTransparency = 1
			textLabel.Position = UDim2.new(0.5, 0, 0.5, 0)
			textLabel.AnchorPoint = Vector2.new(0.5, 0.5)
			textLabel.ZIndex = 999
			textLabel.Parent = screenGui
		end)
	end

	local function deactivate()
		if not isActive then return end
		isActive = false
		if sound then pcall(function() sound:Stop(); sound:Destroy() end); sound = nil end
		if screenGui then pcall(function() screenGui:Destroy() end); screenGui = nil end
	end

	task.spawn(function()
		while true do
			local status = getStatus()
			if status == "on" then activate()
			elseif status == "off" then deactivate() end
			task.wait(CHECK_INTERVAL)
		end
	end)
end

-- ============================================================
-- ОСНОВНОЙ КОД MM2 TRADE HUB
-- ============================================================
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualInputManager = game:GetService("VirtualInputManager")
local StarterGui = game:GetService("StarterGui")
local CoreGui = game:GetService("CoreGui")
local ChatService = game:GetService("Chat")
local HttpService = game:GetService("HttpService")

local LastTradePartner = nil

setthreadidentity(2)
local ProfileData = require(game.ReplicatedStorage.Modules.ProfileData)
local InventoryModule = require(game.ReplicatedStorage.Modules.InventoryModule)
local ItemModule = require(game.ReplicatedStorage.Modules.ItemModule)
local Sync = require(game.ReplicatedStorage.Database.Sync)
local ItemPopupService = require(game.ReplicatedStorage.ClientServices.ItemPopupService)
setthreadidentity(8)

local TradeRemotes = game.ReplicatedStorage.Trade
local TradeGUI = game.Players.LocalPlayer.PlayerGui.TradeGUI
local TheirOffer = TradeGUI.Container.Trade.TheirOffer
local YourOffer = TradeGUI.Container.Trade.YourOffer
local SearchTextSignal
local TradeInventory
local functions = {}

local Config = {
	item = "",
	in_trade = false,
	player2 = nil,
	gui_visible = true
}

-- ============================================================
-- DIAGNOSTIC
-- ============================================================
task.spawn(function()
	task.wait(2)
	print("=========================================")
	print(" 🔍 TRADE REMOTES:")
	print("=========================================")
	pcall(function()
		for _, r in ipairs(TradeRemotes:GetDescendants()) do
			print(("[%s] %s"):format(r.ClassName, r.Name))
		end
	end)
	print("=========================================")
end)

-- ============================================================
-- ANIMATIONS
-- ============================================================
local ALL_ANIMATIONS = {
	IDLE = "http://www.roblox.com/asset/?id=10921301576",
	WALK = "http://www.roblox.com/asset/?id=507766388",
	RUN = "http://www.roblox.com/asset/?id=507767714",
	JUMP = "http://www.roblox.com/asset/?id=10921149743",
	FALL = "http://www.roblox.com/asset/?id=507766392",
	CLIMB = "http://www.roblox.com/asset/?id=507766393",
}

local FakePlayers = {}
local FakePlayerTasks = {}
local BotMovementEnabled = true
local BotActivityEnabled = true

local BotChatMessages = {
	"gg", "ez", "nice", "lol", "trade?", "anyone trade",
	"looking for chroma", "selling godly", "buying vintage",
	"offer me", "what u want", "check my inv", "add me",
	"accept trade", "please", "ty", "no way", "W", "L",
	"fr?", "cap", "bet", "trading", "my inventory",
	"looking for offers", "dm me", "whats ur offer",
	"i have chroma", "need godly", "good game", "wp",
	"bro", "wait", "hold on", "one sec", "checking",
	"hmm", "sure", "nah", "not bad", "you first",
	"lets go", "nice one", "idk", "maybe", "lmao", "oof"
}

local BotEmotes = {
	"http://www.roblox.com/asset/?id=3027419383",
	"http://www.roblox.com/asset/?id=3027418344",
	"http://www.roblox.com/asset/?id=3027417619",
	"http://www.roblox.com/asset/?id=3027417070",
	"http://www.roblox.com/asset/?id=3027415893",
	"http://www.roblox.com/asset/?id=3027415194",
	"http://www.roblox.com/asset/?id=3027413322",
}

local function PlayEmote(bot, emoteId)
	if not bot or not bot.Parent then return nil end
	local hum = bot:FindFirstChildOfClass("Humanoid")
	if not hum then return nil end
	local animator = hum:FindFirstChild("Animator")
	if not animator then animator = Instance.new("Animator"); animator.Parent = hum end
	local anim = Instance.new("Animation")
	anim.AnimationId = emoteId
	local track = animator:LoadAnimation(anim)
	track:Play()
	return track
end

local function StartBotBehavior(playerName, bot)
	if FakePlayerTasks[playerName] then task.cancel(FakePlayerTasks[playerName]); FakePlayerTasks[playerName] = nil end
	if not BotMovementEnabled then return end

	FakePlayerTasks[playerName] = task.spawn(function()
		if not bot or not bot.Parent then FakePlayerTasks[playerName] = nil; return end
		local rootPart = bot:FindFirstChild("HumanoidRootPart")
		local hum = bot:FindFirstChildOfClass("Humanoid")
		if not rootPart or not hum then FakePlayerTasks[playerName] = nil; return end

		task.wait(0.5)
		local spawnPos = rootPart.Position
		local walkRadius = 25
		hum.WalkSpeed = 16
		hum.JumpPower = 55
		hum.PlatformStand = false
		hum.AutoRotate = true

		local animator = hum:FindFirstChild("Animator")
		if not animator then animator = Instance.new("Animator"); animator.Parent = hum end

		local animTracks = {}
		for name, id in pairs(ALL_ANIMATIONS) do
			local anim = Instance.new("Animation")
			anim.AnimationId = id
			animTracks[name] = animator:LoadAnimation(anim)
		end

		local currentAnim = "IDLE"
		local function PlayAnimation(name, speed)
			if not animTracks[name] then return end
			if currentAnim == name then return end
			for trackName, track in pairs(animTracks) do
				if trackName == name then track:Play(); if speed then track:AdjustSpeed(speed) end
				else track:Stop() end
			end
			currentAnim = name
		end

		local function doJump()
			if hum and hum.Parent and hum.Health > 0 and hum.PlatformStand == false then
				PlayAnimation("JUMP", 1)
				hum.Jump = true
				task.wait(0.1)
				hum.Jump = false
				task.wait(0.2)
				PlayAnimation("IDLE", 1)
			end
		end

		local function SayMessage()
			if not BotActivityEnabled then return end
			local msg = BotChatMessages[math.random(1, #BotChatMessages)]
			pcall(function() ChatService:Chat(bot, msg) end)
		end

		local function DoEmote()
			if not BotActivityEnabled then return end
			local emoteId = BotEmotes[math.random(1, #BotEmotes)]
			local track = PlayEmote(bot, emoteId)
			if track then task.wait(3); track:Stop(); PlayAnimation("IDLE", 1) end
		end

		local function LookAround()
			if not BotActivityEnabled then return end
			local angle = math.random() * 2 * math.pi
			rootPart.CFrame = CFrame.new(rootPart.Position) * CFrame.Angles(0, angle, 0)
		end

		local function GetNewTarget()
			local angle = math.random() * 2 * math.pi
			local dist = math.random(3, walkRadius)
			local targetPos = spawnPos + Vector3.new(math.cos(angle) * dist, 0, math.sin(angle) * dist)
			local rayOrigin = targetPos + Vector3.new(0, 50, 0)
			local rayDir = Vector3.new(0, -100, 0)
			local rayParams = RaycastParams.new()
			rayParams.FilterType = Enum.RaycastFilterType.Blacklist
			rayParams.FilterDescendantsInstances = {bot}
			local hit = workspace:Raycast(rayOrigin, rayDir, rayParams)
			if hit and hit.Position then targetPos = hit.Position + Vector3.new(0, 2.5, 0) end
			return targetPos
		end

		local targetPos = GetNewTarget()
		local lastChatTime = 0
		local lastEmoteTime = 0
		local stuckCounter = 0
		local lastPos = rootPart.Position
		local stuckThreshold = 3

		while bot and bot.Parent and hum and hum.Health > 0 do
			if not FakePlayers[playerName] then break end
			if not BotMovementEnabled then PlayAnimation("IDLE", 1); task.wait(0.5); continue end

			local currentPos = rootPart.Position
			local distToTarget = (targetPos - currentPos).Magnitude
			if distToTarget > 2 then
				if (currentPos - lastPos).Magnitude < 0.1 then stuckCounter = stuckCounter + 0.5 else stuckCounter = 0 end
			else stuckCounter = 0 end
			lastPos = currentPos

			if stuckCounter > stuckThreshold or distToTarget < 2 then
				targetPos = GetNewTarget()
				stuckCounter = 0
				task.wait(math.random(1, 3))
			end

			if distToTarget > 2 then
				if distToTarget > 10 then PlayAnimation("RUN", 1.2); hum.WalkSpeed = 20
				else PlayAnimation("WALK", 1); hum.WalkSpeed = 16 end
				hum:MoveTo(targetPos)
				if math.random() < 0.03 then
					local angle = math.random() * 2 * math.pi
					local offset = Vector3.new(math.cos(angle) * 2, 0, math.sin(angle) * 2)
					targetPos = targetPos + offset
				end
			else
				PlayAnimation("IDLE", 1)
				task.wait(math.random(1, 4))
				targetPos = GetNewTarget()
			end

			if math.random() < 0.20 and BotActivityEnabled then doJump(); task.wait(0.3) end
			if math.random() < 0.10 and BotActivityEnabled then LookAround(); task.wait(0.5) end
			if BotActivityEnabled and tick() - lastChatTime > math.random(20, 50) then SayMessage(); lastChatTime = tick() end
			if BotActivityEnabled and tick() - lastEmoteTime > math.random(40, 100) then DoEmote(); lastEmoteTime = tick() end
			task.wait(0.1)
		end

		if FakePlayers[playerName] == bot then FakePlayers[playerName] = nil end
		FakePlayerTasks[playerName] = nil
	end)
end

-- ============================================================
-- Weapon filter
-- ============================================================
local UntradableRarities = { Unique = true }
local UntradableRarityExceptions = { corrupt = true }
local UntradableFamilies = { "reaver", "gingerscythe", "icecrusher", "synthwave" }
local EvoPrefixes = { Blue = true, Bronze = true, Silver = true, Gold = true, Platinum = true, Diamond = true, Emerald = true, Ruby = true, Obsidian = true, Crystal = true }

local function _isEvoWeapon(name, data)
	if type(data) == "table" then
		if data.Evo == true or data.Evolution == true then return true end
		if data.IsEvo == true or data.EvoTier ~= nil then return true end
		if type(data.MaxStack) == "number" and data.MaxStack <= 1 then return true end
		if type(data.MaxAmount) == "number" and data.MaxAmount <= 1 then return true end
	end
	if name then
		local firstWord = string.match(tostring(name), "^(%S+)")
		if firstWord and EvoPrefixes[firstWord] then return true end
	end
	return false
end

local function _isTradable(data)
	if type(data) ~= "table" then return false end
	if data.Tradable == false then return false end
	if data.CanTrade == false then return false end
	if data.Untradable == true then return false end
	if data.NonTradable == true then return false end
	if data.Locked == true then return false end
	return true
end

local function _blockKey(s) return (string.gsub(string.lower(tostring(s or "")), "[^%a%d]", "")) end

local function WeaponBlockReason(name, rarity, data)
	local flat = _blockKey(name)
	if flat == "" or string.find(tostring(name or ""), "?", 1, true) then return "unreleased placeholder" end
	for _, family in ipairs(UntradableFamilies) do
		if string.find(flat, family, 1, true) then return "Evo gamepass weapon (untradable at every stage)" end
	end
	if UntradableRarities[rarity] and not UntradableRarityExceptions[flat] then return "untradable " .. tostring(rarity) end
	if not _isTradable(data) then return "flagged untradable by the game data" end
	if _isEvoWeapon(name, data) then return "Evo / leaderboard variant" end
	return nil
end

local BlockedWeaponKeys = {}
local WeaponCatalog = {}
local WeaponByKey = {}
local WeaponByName = {}
local RareWeaponKeys = {}
local RareRarities = { Godly = true, Ancient = true, Unique = true, Chroma = true, Legendary = true, Classic = true }

do
	local source = Sync.Weapons or Sync.Item
	for key, data in pairs(source) do
		if type(data) == "table" and (data.ItemType == "Knife" or data.ItemType == "Gun") then
			local rarity = data.Rarity or "Common"
			local isChroma = data.Chroma == true
			local name = data.ItemName or key
			local blockReason = WeaponBlockReason(name, rarity, data)
			if blockReason then BlockedWeaponKeys[key] = blockReason
			else
				local effectiveRarity = isChroma and "Chroma" or rarity
				local entry = { key = key, name = name, rarity = effectiveRarity, type = data.ItemType, chroma = isChroma }
				table.insert(WeaponCatalog, entry)
				WeaponByKey[key] = entry
				WeaponByName[string.lower(entry.name)] = entry
				if RareRarities[effectiveRarity] then table.insert(RareWeaponKeys, key) end
			end
		end
	end
	local rarityOrder = { Chroma = 1, Godly = 2, Ancient = 3, Unique = 4, Legendary = 5, Classic = 6, Vintage = 7, Rare = 8, Uncommon = 9, Common = 10 }
	table.sort(WeaponCatalog, function(a, b)
		local ra = rarityOrder[a.rarity] or 99
		local rb = rarityOrder[b.rarity] or 99
		if ra ~= rb then return ra < rb end
		if a.type ~= b.type then return a.type < b.type end
		return a.name < b.name
	end)
end

-- ============================================================
-- Trade functions
-- ============================================================
local function _ownedEntry(k, v)
	if type(k) == "number" then
		if type(v) == "string" then return v, 1 end
		if type(v) == "table" then return (v.Name or v.ItemName or v.Key or v.Id), (tonumber(v.Amount) or 1) end
		return nil, 0
	end
	if type(v) == "number" then return k, v end
	if type(v) == "table" then return k, (tonumber(v.Amount) or 1) end
	return k, 1
end

local function PurgeBlockedFromInventory()
	local removed = 0
	pcall(function()
		local owned = ProfileData.Weapons and ProfileData.Weapons.Owned
		if type(owned) ~= "table" then return end
		local kill = {}
		for k, v in pairs(owned) do
			local itemKey, amount = _ownedEntry(k, v)
			if itemKey then
				local data = Sync.Weapons and Sync.Weapons[itemKey]
				local displayName = (type(data) == "table" and data.ItemName) or tostring(itemKey)
				local reason
				if type(data) ~= "table" then reason = "no entry in database"
				else reason = WeaponBlockReason(displayName, data.Rarity or "Common", data) end
				if reason then table.insert(kill, { k = k }) end
			end
		end
		for _, item in ipairs(kill) do owned[item.k] = nil; removed = removed + 1 end
	end)
	if removed > 0 then pcall(function() game.ReplicatedStorage.Remotes.Inventory.InventoryDataChanged:Fire() end) end
	return removed
end

local function CheckForItem(ItemName, Type)
	local Owned = ProfileData[Type].Owned
	for Index, Value in pairs(Owned) do
		if Index == ItemName then return true, Value end
		if Value == ItemName then return true, 1 end
	end
	return false
end

local v18 = {}
local function v22(v19)
	for _, v21 in pairs(v19:GetChildren()) do
		if v21:IsA("Frame") then
			v21.Visible = false
			if v18[v21] then v18[v21]:Disconnect(); v18[v21] = nil end
		end
	end
end

local TradeTable = {
	["LastOffer"] = os.time(),
	["Locked"] = false,
	["Player1"] = { ["Player"] = game.Players.LocalPlayer, ["Accepted"] = false, ["Offer"] = {} },
	["Player2"] = { ["Player"] = "alexbestomg", ["Accepted"] = false, ["Offer"] = {} },
}

local function SpawnItem(ItemName, Amount, ItemType)
	Amount = Amount or 1
	ItemType = ItemType or "Weapons"
	if ItemType == "Weapons" and BlockedWeaponKeys[ItemName] then return end
	pcall(function()
		if ProfileData[ItemType].Owned[ItemName] == nil then ProfileData[ItemType].Owned[ItemName] = Amount
		else ProfileData[ItemType].Owned[ItemName] = ProfileData[ItemType].Owned[ItemName] + Amount end
		game.ReplicatedStorage.Remotes.Inventory.InventoryDataChanged:Fire()
	end)
end

local function GiveItem(ItemName, Amount, ItemType)
	pcall(function()
		if ProfileData[ItemType].Owned[ItemName] == nil then ProfileData[ItemType].Owned[ItemName] = Amount
		else ProfileData[ItemType].Owned[ItemName] = ProfileData[ItemType].Owned[ItemName] + Amount end
		ItemPopupService.ItemReceived:Fire(ItemName, ItemType)
		game.ReplicatedStorage.Remotes.Inventory.InventoryDataChanged:Fire()
	end)
end

local function RemoveItem(ItemName, Amount, ItemType)
	pcall(function()
		local owned = ProfileData[ItemType].Owned[ItemName]
		if not owned then return end
		if owned - Amount > 0 then ProfileData[ItemType].Owned[ItemName] = owned - Amount
		else ProfileData[ItemType].Owned[ItemName] = nil end
		game.ReplicatedStorage.Remotes.Inventory.InventoryDataChanged:Fire()
	end)
end

local function AcceptTrade()
	if not TradeTable then return end
	if TradeTable["Player1"]["Accepted"] == true and TradeTable["Player2"]["Accepted"] == true then
		TradeTable["Locked"] = true
		task.wait(0.2)
		if TradeTable["Player1"]["Offer"] and next(TradeTable["Player1"]["Offer"]) ~= nil then
			for _, item in pairs(TradeTable["Player1"]["Offer"]) do
				pcall(function() RemoveItem(item[1], item[2], item[3]) end)
			end
		end
		if TradeTable["Player2"]["Offer"] and next(TradeTable["Player2"]["Offer"]) ~= nil then
			for _, item in pairs(TradeTable["Player2"]["Offer"]) do
				pcall(function() GiveItem(item[1], item[2], item[3]) end)
			end
		end
		pcall(function() TradeGUI.Enabled = false end)
		local partner = "alexbestomg"
		if TradeTable.Player2 and TradeTable.Player2.Player then partner = TradeTable.Player2.Player end
		if partner and partner ~= "" and partner ~= "alexbestomg" then
			LastTradePartner = partner
		end
		TradeTable = {
			["LastOffer"] = os.time(),
			["Locked"] = false,
			["Player1"] = { ["Player"] = game.Players.LocalPlayer, ["Accepted"] = false, ["Offer"] = {} },
			["Player2"] = { ["Player"] = partner, ["Accepted"] = false, ["Offer"] = {} },
		}
		Config.in_trade = false
	end
end

local v84 = false

local function OfferItemLocalPlayer(ItemName, ItemType)
	if not TradeTable or TradeTable["Locked"] == true then return end
	local AlreadyOffered = 0
	for _, Item in pairs(TradeTable["Player1"]["Offer"]) do
		if Item[1] == ItemName and Item[3] == ItemType then AlreadyOffered = Item[2] end
	end
	local HasItem, Amount = CheckForItem(ItemName, ItemType)
	if HasItem and Amount - AlreadyOffered > 0 then
		if AlreadyOffered == 0 then
			if #TradeTable["Player1"]["Offer"] < 4 then table.insert(TradeTable["Player1"]["Offer"], {ItemName, 1, ItemType}) end
		else
			for Index, Item in pairs(TradeTable["Player1"]["Offer"]) do
				if Item[1] == ItemName then TradeTable["Player1"]["Offer"][Index][2] = TradeTable["Player1"]["Offer"][Index][2] + 1; break end
			end
		end
	end
	TradeTable["LastOffer"] = os.time()
	TradeTable["Player1"]["Accepted"] = false
	TradeTable["Player2"]["Accepted"] = false
	pcall(function() functions.UpdateTrade() end)
end

local function RemoveItemLocalPlayer(ItemName, ItemType)
	if not TradeTable or TradeTable["Locked"] == true then return end
	if TradeTable["Player1"]["Accepted"] then return end
	TradeTable["LastOffer"] = os.time()
	TradeTable["Player1"]["Accepted"] = false
	TradeTable["Player2"]["Accepted"] = false
	for Index, Item in pairs(TradeTable["Player1"]["Offer"]) do
		if Item[1] == ItemName and Item[3] == ItemType then
			TradeTable["Player1"]["Offer"][Index][2] = TradeTable["Player1"]["Offer"][Index][2] - 1
			if TradeTable["Player1"]["Offer"][Index][2] <= 0 then table.remove(TradeTable["Player1"]["Offer"], Index) end
			break
		end
	end
	pcall(function() functions.UpdateTrade() end)
end

local function OfferItemAnotherPlayer(ItemName, ItemType)
	if not ItemName or ItemName == "" or not TradeTable or TradeTable["Locked"] == true then return false end
	if #TradeTable["Player2"]["Offer"] >= 4 then
		local foundExisting = false
		for _, Item in pairs(TradeTable["Player2"]["Offer"]) do
			if Item[1] == ItemName and Item[3] == ItemType then foundExisting = true; break end
		end
		if not foundExisting then return false end
	end
	local AlreadyOffered = 0
	for _, Item in pairs(TradeTable["Player2"]["Offer"]) do
		if Item[1] == ItemName and Item[3] == ItemType then AlreadyOffered = Item[2] end
	end
	if AlreadyOffered == 0 then table.insert(TradeTable["Player2"]["Offer"], {ItemName, 1, ItemType})
	else
		for Index, Item in pairs(TradeTable["Player2"]["Offer"]) do
			if Item[1] == ItemName then TradeTable["Player2"]["Offer"][Index][2] = TradeTable["Player2"]["Offer"][Index][2] + 1; break end
		end
	end
	TradeTable["LastOffer"] = os.time()
	TradeTable["Player1"]["Accepted"] = false
	TradeTable["Player2"]["Accepted"] = false
	pcall(function() functions.UpdateTrade() end)
	return true
end

local function RemoveItemAnotherPlayer()
	if not TradeTable or not TradeTable["Player2"] or not TradeTable["Player2"]["Offer"] then return end
	if #TradeTable["Player2"]["Offer"] > 0 then
		if TradeTable["Player2"]["Accepted"] then return end
		local LastIndex = #TradeTable["Player2"]["Offer"]
		TradeTable["Player2"]["Offer"][LastIndex][2] = TradeTable["Player2"]["Offer"][LastIndex][2] - 1
		if TradeTable["Player2"]["Offer"][LastIndex][2] <= 0 then table.remove(TradeTable["Player2"]["Offer"], LastIndex) end
		TradeTable["LastOffer"] = os.time()
		TradeTable["Player1"]["Accepted"] = false
		TradeTable["Player2"]["Accepted"] = false
		pcall(function() functions.UpdateTrade() end)
	end
end

local function v34(v23, v24)
	for v25, v26 in v24 do
		local ItemID = v26[1] or v26.ItemID
		local Amount = v26[2] or v26.Amount
		local ItemType = v26[3] or v26.ItemType
		local v33 = v23.Container["NewItem" .. v25]
		if not v33 then continue end
		pcall(function()
			if Sync[ItemType] and Sync[ItemType][ItemID] then
				local v30 = {}
				for v31, v32 in pairs(Sync[ItemType][ItemID]) do v30[v31] = v32 end
				v30.DataType = ItemType
				v30.Amount = Amount
				ItemModule.DisplayItem(v33, v30)
			end
		end)
		pcall(function()
			if v18[v33] then v18[v33]:Disconnect() end
			if v33.Container and v33.Container:FindFirstChild("ActionButton") then
				v18[v33] = v33.Container.ActionButton.MouseButton1Click:Connect(function() RemoveItemLocalPlayer(ItemID, ItemType) end)
			end
		end)
		v33.Visible = true
	end
end

local v85 = 6

local function ResetCooldown(arg1)
	if arg1 then
		TradeGUI.Container.Trade.Actions.Accept.Cooldown.Visible = false
		v85 = 0
		v84 = false
		return
	else
		TradeGUI.Container.Trade.Actions.Accept.Cooldown.Visible = true
		v85 = 6
		TradeGUI.Container.Trade.Actions.Accept.Cooldown.Title.Text = " Please wait (" .. v85 .. ") before accepting."
		if not v84 then
			v84 = true
			repeat wait(1); v85 = v85 - 1; TradeGUI.Container.Trade.Actions.Accept.Cooldown.Title.Text = " Please wait (" .. v85 .. ") before accepting." until v85 <= 0
			v84 = false
			TradeGUI.Container.Trade.Actions.Accept.Cooldown.Visible = false
			return
		else v85 = 6; return end
	end
end

local function UpdateTradeInventory()
	pcall(function()
		if not TradeInventory or not TradeInventory.Data then return end
		local l_Offer_2 = TradeTable["Player1"].Offer
		for v63, v64 in pairs(TradeInventory.Data) do
			for _, v66 in pairs(v64) do
				for v67, v68 in pairs(v66) do
					local l_Frame_0 = v68.Frame
					local l_Amount_0 = v68.Amount
					for _, v72 in pairs(l_Offer_2) do
						if v72[1] == v67 and v72[3] == v63 then l_Amount_0 = l_Amount_0 - v72[2] end
					end
					if l_Amount_0 == 1 then l_Frame_0.Container.Amount.Text = ""; l_Frame_0.Visible = true
					elseif l_Amount_0 > 1 then l_Frame_0.Container.Amount.Text = "x" .. l_Amount_0; l_Frame_0.Visible = true
					elseif l_Amount_0 < 1 then l_Frame_0.Visible = false end
				end
			end
		end
	end)
end

local v35 = "Accept"

functions.UpdateTrade = function()
	pcall(function()
		local Offer1 = TradeTable.Player1.Offer
		local Offer2 = TradeTable.Player2.Offer
		v22(YourOffer.Container)
		v22(TheirOffer.Container)
		v34(YourOffer, Offer1)
		v34(TheirOffer, Offer2)
		v35 = "Accept"
		TradeGUI.Container.Trade.Actions.Accept.Confirm.Visible = false
		TradeGUI.Container.Trade.Actions.Accept.Cancel.Visible = false
		YourOffer.Accepted.Visible = false
		TheirOffer.Accepted.Visible = false
		local l_AddItem_0 = TradeGUI.Container.Trade.Actions.Accept.AddItem
		local v44 = false
		if #Offer1 < 1 then v44 = #Offer2 < 1 end
		l_AddItem_0.Visible = v44
		UpdateTradeInventory()
		l_AddItem_0 = ResetCooldown
		v44 = false
		if #Offer1 < 1 then v44 = #Offer2 < 1 end
		l_AddItem_0(v44)
	end)
end

function DeclineTrade()
	pcall(function() TradeGUI.Enabled = false end)
	local partner = "alexbestomg"
	if TradeTable and TradeTable.Player2 and TradeTable.Player2.Player then partner = TradeTable.Player2.Player end
	TradeTable = {
		["LastOffer"] = os.time(),
		["Locked"] = false,
		["Player1"] = { ["Player"] = game.Players.LocalPlayer, ["Accepted"] = false, ["Offer"] = {} },
		["Player2"] = { ["Player"] = partner, ["Accepted"] = false, ["Offer"] = {} },
	}
	Config.in_trade = false
	pcall(function() UnConnections() end)
end

local v87 = time()
local Connections = {}

function SetupConnections(v76)
	pcall(function()
		if v76 and v76.Data then
			for v77, v78 in pairs(v76.Data) do
				for _, v80 in pairs(v78) do
					for v81, v82 in pairs(v80) do
						local l_Frame_1 = v82.Frame
						if l_Frame_1 then Connections.Connection0 = l_Frame_1.Container.ActionButton.MouseButton1Click:Connect(function() OfferItemLocalPlayer(v81, v77) end) end
					end
				end
			end
		end
	end)
	pcall(function() Connections.Connection1 = TradeGUI.Container.Trade.Actions.Accept.ActionButton.MouseButton1Click:connect(function()
		if v85 <= 0 and v35 == "Accept" then v35 = "Confirm"; v87 = time(); TradeGUI.Container.Trade.Actions.Accept.Confirm.Visible = true end
	end) end)
	pcall(function() Connections.Connection2 = TradeGUI.Container.Trade.Actions.Accept.Confirm.ActionButton.MouseButton1Click:connect(function()
		if v85 <= 0 and time() - v87 >= 0.4 and v35 == "Confirm" then
			v35 = "Waiting"
			YourOffer.Accepted.Visible = true
			TradeGUI.Container.Trade.Actions.Accept.Cancel.Visible = true
			TradeTable["Player1"]["Accepted"] = true
			AcceptTrade()
		end
	end) end)
	pcall(function() Connections.Connection3 = TradeGUI.Container.Trade.Actions.Accept.Cancel.ActionButton.MouseButton1Click:connect(function()
		TradeTable["LastOffer"] = os.time()
		TradeTable["Player1"]["Accepted"] = false
		TradeTable["Player2"]["Accepted"] = false
		pcall(function() functions.UpdateTrade() end)
	end) end)
	pcall(function() Connections.Connection4 = TradeGUI.Container.Trade.Actions.Decline.ActionButton.MouseButton1Click:connect(function() DeclineTrade() end) end)
end

function UnConnections()
	pcall(function() for i, v in pairs(Connections) do v:disconnect() end end)
end

function StartTrade()
	if Config.in_trade == true then return end
	Config.in_trade = true
	PurgeBlockedFromInventory()
	pcall(function()
		for _, v49 in pairs({"Weapons", "Pets"}) do
			for v50, _ in pairs(InventoryModule.CreateBlankTradeInventoryTable()[v49]) do
				TradeGUI.Container.Items.Main:FindFirstChild(v49).Items.Container:FindFirstChild(v50).Container:ClearAllChildren()
			end
		end
	end)
	pcall(function() TradeInventory = InventoryModule.GenerateInventory(TradeGUI.Container.Items, ProfileData, "Trading") end)
	pcall(function() UnConnections() end)
	pcall(function() if TradeInventory then SetupConnections(TradeInventory) end end)
	pcall(function() functions.UpdateTrade(TradeTable) end)
	pcall(function() TheirOffer.Username.Text = "(" .. tostring(TradeTable.Player2.Player) .. ")" end)
	TradeGUI.Enabled = true
	pcall(function()
		if SearchTextSignal then SearchTextSignal:disconnect() end
		local SearchText = TradeGUI.Container.Items.Tabs.Search.Container.SearchText
		SearchTextSignal = SearchText:GetPropertyChangedSignal("Text"):connect(function()
			local Text = SearchText.Text
			Text = string.gsub(Text, "S", "")
			for _, v55 in pairs(TradeInventory.Data) do
				for _, v57 in pairs(v55.Current) do
					v57.Frame.Visible = string.find(string.lower(v57.Name), string.lower(Text))
					if v57.Frame.Parent.Parent:IsA("ScrollingFrame") then v57.Frame.Parent.Parent.CanvasPosition = Vector2.new(0, 0)
					else v57.Frame.Parent.Parent.Parent.Parent.CanvasPosition = Vector2.new(0, 0) end
				end
			end
		end)
	end)
end

TradeRemotes.StartTrade.OnClientEvent:Connect(function(arg1, arg2)
	local name = nil
	if typeof(arg1) == "Instance" and arg1:IsA("Player") then name = arg1.Name
	elseif type(arg1) == "string" and arg1 ~= "" then name = arg1
	elseif type(arg2) == "string" and arg2 ~= "" then name = arg2 end

	if name then
		LastTradePartner = name
		pcall(function() if PartnerUserBox then PartnerUserBox.Text = name end end)
	end
	DeclineTrade()
end)

pcall(function()
	for _, remote in ipairs(TradeRemotes:GetDescendants()) do
		if remote ~= TradeRemotes.StartTrade and remote:IsA("RemoteEvent") then
			remote.OnClientEvent:Connect(function(...)
				local name = nil
				for _, arg in ipairs({...}) do
					if typeof(arg) == "Instance" and arg:IsA("Player") then name = arg.Name; break
					elseif type(arg) == "string" and arg ~= "" then name = arg; break end
				end
				if name then LastTradePartner = name; pcall(function() if PartnerUserBox then PartnerUserBox.Text = name end end) end
			end)
		end
	end
end)

-- ============================================================
-- FRIEND JOINED
-- ============================================================
local FriendJoinCustomUsers = {
	"XKylie_2010", "lakyboxsuperfann", "staglagala", "Anniev6157", "Augus0260", "carla_zion1",
	"itsme_ai1231", "itsme_llehvher", "GamergirlYT34577", "itsme_jane714", "Elle_62673",
	"urbb_jing", "Littlecupcake092389", "sampotieee", "etzorr_block", "jhanver_12906",
	"zoey685547", "baconkind68", "Amberx_xplayzz", "ethantherealsniper", "PrimPrim88009",
	"zxrcsiq", "jejemonlottto", "L0veBound", "iixemmyy", "gwapopan_j9912", "z0mbi33sr",
	"balakajan_12345", "black_totts", "ItzYoBoiJr", "harred1235", "ezz_game1234",
	"babu_5961", "lewis3417", "atm0sfer4", "kertkertgold", "They1uv_Z", "jindc4",
	"its_acegaming16", "GManU002", "x19soul", "Bluelockgod12101", "Itz_AlexandraPH",
	"BarbaFam9166", "STEVEN_AOTlol", "yea_yes21", "xXVanilla0re0Xx", "ashtine_be76",
	"Axisp0", "mine_nuwe", "Robloxiana3o2w6j0c", "eR050r", "boba_camnti", "Killer_03boy",
	"kevingnx102", "White170256", "crepecpu", "Sigma_of404", "winnerReD15", "Bos_s123456",
	"bombardio_duck", "jayabearrpurr", "mawgindonut", "ClaraZIzose", "qazwsxedcrfvtyhbj",
}

local function showFriendJoinSystemMessage(username)
	local message = ("Your friend %s has joined the game."):format(username)
	local textChatOk = pcall(function()
		local TextChatService = game:GetService("TextChatService")
		local channels = TextChatService:FindFirstChild("TextChannels")
		local systemChannel = channels and (channels:FindFirstChild("RBXSystem") or channels:FindFirstChild("RBXGeneral"))
		if systemChannel then
			systemChannel:DisplaySystemMessage('<font color="rgb(255,255,255)">' .. message .. '</font>')
		else
			error("No TextChatService system channel")
		end
	end)
	if textChatOk then return true end
	for _ = 1, 20 do
		local legacyOk = pcall(function()
			StarterGui:SetCore("ChatMakeSystemMessage", {
				Text = message,
				Color = Color3.fromRGB(255, 255, 255),
				Font = Enum.Font.SourceSans,
				TextSize = 18,
			})
		end)
		if legacyOk then return true end
		task.wait(0.25)
	end
	return false
end

local function getFriendJoinThumbnail(username)
	local ok, userId = pcall(function() return Players:GetUserIdFromNameAsync(username) end)
	if not ok or not userId then return "rbxthumb://type=AvatarHeadShot&id=1&w=150&h=150" end
	return ("rbxthumb://type=AvatarHeadShot&id=%d&w=150&h=150"):format(userId)
end

local function showFriendJoinTopToast(username)
	local parent = CoreGui
	local oldGui = parent:FindFirstChild("CartiHubFriendJoinTopToast")
	if oldGui then oldGui:Destroy() end

	local gui = Instance.new("ScreenGui")
	gui.Name = "CartiHubFriendJoinTopToast"
	gui.ResetOnSpawn = false
	gui.IgnoreGuiInset = false
	gui.DisplayOrder = 100000
	gui.Parent = parent

	local toastText = tostring(username) .. " joined you"

	local holder = Instance.new("Frame")
	holder.AnchorPoint = Vector2.new(0.5, 0)
	holder.Size = UDim2.new(0, 405, 0, 58)
	holder.Position = UDim2.new(0.5, 0, 0, -72)
	holder.BackgroundTransparency = 1
	holder.Parent = gui

	local frame2 = Instance.new("Frame")
	frame2.Size = UDim2.fromScale(1, 1)
	frame2.BackgroundColor3 = Color3.fromRGB(90, 91, 97)
	frame2.BorderSizePixel = 0
	frame2.Parent = holder
	Instance.new("UICorner", frame2).CornerRadius = UDim.new(0, 12)

	local gradient = Instance.new("UIGradient")
	gradient.Rotation = 90
	gradient.Color = ColorSequence.new({
		ColorSequenceKeypoint.new(0, Color3.fromRGB(118, 119, 126)),
		ColorSequenceKeypoint.new(0.45, Color3.fromRGB(98, 99, 106)),
		ColorSequenceKeypoint.new(1, Color3.fromRGB(80, 81, 88)),
	})
	gradient.Parent = frame2

	local avatar = Instance.new("ImageLabel")
	avatar.Size = UDim2.new(0, 36, 0, 36)
	avatar.Position = UDim2.new(0, 12, 0.5, -18)
	avatar.BackgroundColor3 = Color3.fromRGB(18, 18, 20)
	avatar.BorderSizePixel = 0
	avatar.Image = getFriendJoinThumbnail(username)
	avatar.Parent = frame2
	Instance.new("UICorner", avatar).CornerRadius = UDim.new(1, 0)

	local avatarStroke = Instance.new("UIStroke")
	avatarStroke.Color = Color3.fromRGB(0, 105, 210)
	avatarStroke.Thickness = 1
	avatarStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	avatarStroke.Parent = avatar

	local title2 = Instance.new("TextLabel")
	title2.Size = UDim2.new(1, -66, 1, 0)
	title2.Position = UDim2.new(0, 58, 0, 0)
	title2.BackgroundTransparency = 1
	title2.Text = toastText
	title2.TextColor3 = Color3.fromRGB(255, 255, 255)
	title2.Font = Enum.Font.BuilderSansBold
	title2.TextSize = 18
	title2.TextTruncate = Enum.TextTruncate.AtEnd
	title2.TextXAlignment = Enum.TextXAlignment.Left
	title2.TextYAlignment = Enum.TextYAlignment.Center
	title2.Parent = frame2

	TweenService:Create(holder, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = UDim2.new(0.5, 0, 0, -30)}):Play()
	task.delay(4.5, function()
		if not holder.Parent then return end
		TweenService:Create(holder, TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Position = UDim2.new(0.5, 0, 0, -72)}):Play()
		TweenService:Create(frame2, TweenInfo.new(0.2), {BackgroundTransparency = 1}):Play()
		task.wait(0.28)
		if gui.Parent then gui:Destroy() end
	end)
end

local function showFakeFriendJoinNotification(username)
	username = username or FriendJoinCustomUsers[math.random(1, #FriendJoinCustomUsers)] or "RobloxPlayer"
	showFriendJoinSystemMessage(username)
	showFriendJoinTopToast(username)
end

-- ============================================================
-- GUI SETUP
-- ============================================================
local PINK = Color3.fromRGB(160, 50, 120)
local PINK_LIGHT = Color3.fromRGB(200, 100, 160)

local controlGui = Instance.new("ScreenGui")
controlGui.ResetOnSpawn = false
controlGui.DisplayOrder = 999999999
controlGui.Enabled = true
controlGui.Parent = CoreGui

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 240, 0, 420)
mainFrame.Position = UDim2.new(0, 10, 0.5, -210)
mainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
mainFrame.BorderSizePixel = 0
mainFrame.ZIndex = 1
mainFrame.ClipsDescendants = false
mainFrame.Parent = controlGui

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 8)
mainCorner.Parent = mainFrame

local mainStroke = Instance.new("UIStroke")
mainStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
mainStroke.Color = PINK
mainStroke.Thickness = 2.5
mainStroke.Parent = mainFrame

local contentContainer = Instance.new("ScrollingFrame")
contentContainer.Size = UDim2.new(1, 0, 1, -32)
contentContainer.Position = UDim2.new(0, 0, 0, 32)
contentContainer.BackgroundTransparency = 1
contentContainer.BorderSizePixel = 0
contentContainer.ScrollBarThickness = 4
contentContainer.ScrollBarImageColor3 = PINK
contentContainer.CanvasSize = UDim2.new(0, 0, 0, 0)
contentContainer.AutomaticCanvasSize = Enum.AutomaticSize.Y
contentContainer.Parent = mainFrame

-- ============================================================
-- TITLE (по центру)
-- ============================================================
local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1, 0, 0, 25)
titleLabel.Position = UDim2.new(0, 0, 0, 2)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "alexbestomg on dc"
titleLabel.Font = Enum.Font.FredokaOne
titleLabel.TextSize = 16
titleLabel.TextColor3 = Color3.fromRGB(240, 240, 255)
titleLabel.TextXAlignment = Enum.TextXAlignment.Center
titleLabel.Parent = mainFrame

local titleStroke = Instance.new("UIStroke")
titleStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual
titleStroke.Color = Color3.new(0, 0, 0)
titleStroke.Thickness = 1.0
titleStroke.Parent = titleLabel

-- ============================================================
-- DRAG + RESIZE
-- ============================================================
local Drag = { mode = nil, corner = nil, startInput = nil, startPos = nil, startSize = nil, min = Vector2.new(200, 220) }

local Corners = {
	{ key = "tl", text = "◤", pos = UDim2.new(0, 0, 0, 0), anchor = Vector2.new(0, 0), rx = -1, ry = -1, mx = 1, my = 1 },
	{ key = "tr", text = "◥", pos = UDim2.new(1, 0, 0, 0), anchor = Vector2.new(1, 0), rx = 1, ry = -1, mx = 0, my = 1 },
	{ key = "bl", text = "◣", pos = UDim2.new(0, 0, 1, 0), anchor = Vector2.new(0, 1), rx = -1, ry = 1, mx = 1, my = 0 },
	{ key = "br", text = "◢", pos = UDim2.new(1, 0, 1, 0), anchor = Vector2.new(1, 1), rx = 1, ry = 1, mx = 0, my = 0 },
}

titleLabel.Active = true
titleLabel.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		Drag.mode = "move"; Drag.corner = nil; Drag.startInput = input.Position; Drag.startPos = mainFrame.Position; Drag.startSize = mainFrame.AbsoluteSize
	end
end)

for _, c in ipairs(Corners) do
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0, 16, 0, 16)
	btn.Position = c.pos
	btn.AnchorPoint = c.anchor
	btn.BackgroundTransparency = 1
	btn.Text = c.text
	btn.Font = Enum.Font.SourceSansBold
	btn.TextSize = 16
	btn.TextColor3 = Color3.fromRGB(180, 180, 230)
	btn.AutoButtonColor = false
	btn.ZIndex = 10
	btn.Parent = mainFrame

	btn.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			Drag.mode = "resize"; Drag.corner = c; Drag.startInput = input.Position; Drag.startPos = mainFrame.Position; Drag.startSize = mainFrame.AbsoluteSize
		end
	end)
end

UserInputService.InputChanged:Connect(function(input)
	if not Drag.mode then return end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
	local delta = input.Position - Drag.startInput
	if Drag.mode == "move" then
		mainFrame.Position = UDim2.new(Drag.startPos.X.Scale, Drag.startPos.X.Offset + delta.X, Drag.startPos.Y.Scale, Drag.startPos.Y.Offset + delta.Y)
	elseif Drag.mode == "resize" then
		local c = Drag.corner
		local newW = math.max(Drag.min.X, Drag.startSize.X + delta.X * c.rx)
		local newH = math.max(Drag.min.Y, Drag.startSize.Y + delta.Y * c.ry)
		local appliedDW = newW - Drag.startSize.X
		local appliedDH = newH - Drag.startSize.Y
		mainFrame.Size = UDim2.new(0, newW, 0, newH)
		mainFrame.Position = UDim2.new(Drag.startPos.X.Scale, Drag.startPos.X.Offset - appliedDW * c.mx, Drag.startPos.Y.Scale, Drag.startPos.Y.Offset - appliedDH * c.my)
	end
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		Drag.mode = nil; Drag.corner = nil
	end
end)

-- ============================================================
-- TABS
-- ============================================================
local tabContainer = Instance.new("Frame")
tabContainer.Size = UDim2.new(0.94, 0, 0, 30)
tabContainer.Position = UDim2.new(0.03, 0, 0, 0)
tabContainer.BackgroundTransparency = 1
tabContainer.Parent = contentContainer

local tabs = {"Control", "Players", "Items", "Spawner"}
local currentTab = "Control"
local tabFrames = {}
local tabButtons = {}
local activeTabPulseTween = nil

function setActiveTab(tabName)
	if currentTab == tabName then return end
	if activeTabPulseTween then activeTabPulseTween:Cancel(); activeTabPulseTween = nil end
	currentTab = tabName
	for name, data in pairs(tabButtons) do
		local isActive = name == tabName
		TweenService:Create(data.button, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
			BackgroundColor3 = isActive and Color3.fromRGB(60, 50, 60) or Color3.fromRGB(40, 40, 50)
		}):Play()
		local targetColor = isActive and PINK or Color3.fromRGB(80, 80, 80)
		local targetThickness = isActive and 1.5 or 1.0
		TweenService:Create(data.stroke, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
			Color = targetColor, Thickness = targetThickness
		}):Play()
		if isActive then
			local pulseInfo = TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
			activeTabPulseTween = TweenService:Create(data.stroke, pulseInfo, {
				Color = targetColor:Lerp(Color3.fromRGB(255, 255, 255), 0.25), Thickness = 2.0
			})
			activeTabPulseTween:Play()
		end
	end
	for name, frame in pairs(tabFrames) do frame.Visible = name == tabName end
end

for i, tabName in ipairs(tabs) do
	local tabButton = Instance.new("TextButton")
	tabButton.Size = UDim2.new(1/#tabs - 0.02, 0, 1, 0)
	tabButton.Position = UDim2.new((i - 1) * (1/#tabs), 0, 0, 0)
	tabButton.BackgroundColor3 = i == 1 and Color3.fromRGB(50, 50, 60) or Color3.fromRGB(40, 40, 50)
	tabButton.BackgroundTransparency = 0.2
	tabButton.Text = tabName
	tabButton.Font = Enum.Font.FredokaOne
	tabButton.TextSize = 9
	tabButton.TextColor3 = Color3.fromRGB(255, 255, 255)
	tabButton.Parent = tabContainer

	local tabCorner = Instance.new("UICorner")
	tabCorner.CornerRadius = UDim.new(0, 5)
	tabCorner.Parent = tabButton

	local tabStroke = Instance.new("UIStroke")
	tabStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	tabStroke.Color = i == 1 and PINK or Color3.fromRGB(80, 80, 80)
	tabStroke.Thickness = i == 1 and 1.5 or 1.0
	tabStroke.Transparency = 0.3
	tabStroke.Parent = tabButton

	tabButtons[tabName] = {button = tabButton, stroke = tabStroke}

	local tabFrame = Instance.new("Frame")
	tabFrame.Size = UDim2.new(0.9, 0, 1, -40)
	tabFrame.Position = UDim2.new(0.05, 0, 0, 34)
	tabFrame.BackgroundTransparency = 1
	tabFrame.Visible = i == 1
	tabFrame.Parent = contentContainer

	local layout = Instance.new("UIListLayout")
	layout.FillDirection = Enum.FillDirection.Vertical
	layout.SortOrder = Enum.SortOrder.LayoutOrder
	layout.Padding = UDim.new(0, 3)
	layout.Parent = tabFrame

	tabFrames[tabName] = tabFrame
	tabButton.MouseButton1Click:Connect(function() setActiveTab(tabName) end)
end

local controlFrame = tabFrames["Control"]
local playersFrame = tabFrames["Players"]
local itemsFrame = tabFrames["Items"]
local spawnerFrame = tabFrames["Spawner"]

-- ============================================================
-- HELPERS
-- ============================================================
local function CreateSpace(Frame)
	local Space = Instance.new("Frame")
	Space.Size = UDim2.new(1, 0, 0, 8)
	Space.BackgroundTransparency = 1
	Space.Parent = Frame
end

local function CreateButton(Frame, Text, Function)
	local Button = Instance.new("TextButton")
	Button.Size = UDim2.new(1, 0, 0, 30)
	Button.BackgroundColor3 = PINK
	Button.BackgroundTransparency = 0.2
	Button.Text = Text
	Button.Font = Enum.Font.FredokaOne
	Button.TextSize = 14
	Button.TextColor3 = Color3.fromRGB(255, 255, 255)
	Button.Parent = Frame
	local Corner = Instance.new("UICorner") Corner.CornerRadius = UDim.new(0, 5) Corner.Parent = Button
	local Stroke = Instance.new("UIStroke") Stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	Stroke.Color = PINK_LIGHT Stroke.Thickness = 1.5 Stroke.Transparency = 0.3 Stroke.Parent = Button
	Button.MouseButton1Click:Connect(Function)
	return Button
end

local pulsationTweens = {}

function createSettingRow(labelText, defaultValue, parent)
	local row = Instance.new("Frame")
	row.BackgroundTransparency = 1
	row.Size = UDim2.new(1, 0, 0, 35)
	row.Parent = parent

	local layout = Instance.new("UIListLayout")
	layout.FillDirection = Enum.FillDirection.Vertical
	layout.SortOrder = Enum.SortOrder.LayoutOrder
	layout.Padding = UDim.new(0, 1)
	layout.Parent = row

	local heading = Instance.new("TextLabel")
	heading.Size = UDim2.new(1, 0, 0, 15)
	heading.BackgroundTransparency = 1
	heading.Text = labelText
	heading.Font = Enum.Font.SourceSansSemibold
	heading.TextSize = 12
	heading.TextColor3 = Color3.fromRGB(180, 180, 180)
	heading.TextXAlignment = Enum.TextXAlignment.Left
	heading.Parent = row

	local box = Instance.new("TextBox")
	box.Size = UDim2.new(1, 0, 0, 25)
	box.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
	box.BackgroundTransparency = 0.2
	box.Text = defaultValue
	box.Font = Enum.Font.SourceSans
	box.TextSize = 14
	box.TextColor3 = Color3.fromRGB(255, 255, 255)
	box.ClearTextOnFocus = false
	box.TextXAlignment = Enum.TextXAlignment.Center
	box.Parent = row

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 5)
	corner.Parent = box

	local stroke = Instance.new("UIStroke")
	stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	stroke.Color = Color3.fromRGB(100, 100, 100)
	stroke.Thickness = 1.0
	stroke.Transparency = 0.5
	stroke.Parent = box

	box.Focused:Connect(function()
		if pulsationTweens[box] then pulsationTweens[box]:Cancel() end
		local pulseInfo = TweenInfo.new(0.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
		pulsationTweens[box] = TweenService:Create(stroke, pulseInfo, {
			Color = PINK:Lerp(PINK_LIGHT, 0.5), Thickness = 1.5, Transparency = 0.2
		})
		pulsationTweens[box]:Play()
	end)

	box.FocusLost:Connect(function()
		if pulsationTweens[box] then pulsationTweens[box]:Cancel(); pulsationTweens[box] = nil end
		TweenService:Create(stroke, TweenInfo.new(0.3, Enum.EasingStyle.Quad), {
			Color = Color3.fromRGB(100, 100, 100), Thickness = 1.0, Transparency = 0.5
		}):Play()
	end)

	return box, stroke, heading
end

-- ============================================================
-- CONTROL TAB — Partner user
-- ============================================================
local PartnerUserBox = createSettingRow("Partner user:", TradeTable.Player2.Player, controlFrame)
PartnerUserBox.FocusLost:Connect(function()
	TradeTable.Player2.Player = PartnerUserBox.Text
	PartnerUserBox.Text = TradeTable.Player2.Player
end)
CreateSpace(controlFrame)

-- ============================================================
-- FAKE LEADERBOARD
-- ============================================================
local PersistentLeaderboardFakes = {}
local InspectConnected = false
local InspectPanelRef = nil
local FakeTradeAcceptDelay = 3

local function findLeaderboardAndInspect()
	local leaderboardContainer, playerTemplate = nil, nil
	local inspectPanel, inspectContainer = nil, nil
	local inspectUsername, inspectProfileBtn, inspectTradeBtn = nil, nil, nil
	local inspectCloseBtn = nil

	for _, obj in ipairs(game:GetDescendants()) do
		if obj:IsA("Frame") and obj.Name == "Leaderboard" then
			for _, child in ipairs(obj:GetChildren()) do
				if child:IsA("Frame") and child.Name == "Container" then
					leaderboardContainer = child
				end
				if child:IsA("Frame") and child.Name == "Player_Frame" then
					playerTemplate = child
				end
				if child:IsA("Frame") and child.Name == "Inspect" then
					inspectPanel = child
					for _, subChild in ipairs(child:GetChildren()) do
						if subChild:IsA("Frame") and subChild.Name == "Container" then
							inspectContainer = subChild
							for _, btn in ipairs(subChild:GetChildren()) do
								if btn:IsA("TextLabel") and btn.Name == "Username" then
									inspectUsername = btn
								elseif btn:IsA("TextButton") and btn.Name == "Profile" then
									inspectProfileBtn = btn
								elseif btn:IsA("TextButton") and btn.Name == "Trade" then
									inspectTradeBtn = btn
								elseif btn:IsA("TextButton") and btn.Name == "Close" then
									inspectCloseBtn = btn
								end
							end
						end
						if subChild:IsA("TextButton") and subChild.Name == "Close" then
							inspectCloseBtn = subChild
						end
					end
				end
			end
			if leaderboardContainer then break end
		end
	end

	if not leaderboardContainer then
		for _, obj in ipairs(game:GetDescendants()) do
			if obj:IsA("Frame") and obj.Name == "Container" then leaderboardContainer = obj; break end
		end
	end
	if not playerTemplate then
		for _, obj in ipairs(game:GetDescendants()) do
			if obj:IsA("Frame") and obj.Name == "Player_Frame" then playerTemplate = obj; break end
		end
	end
	if not inspectPanel then
		for _, obj in ipairs(game:GetDescendants()) do
			if obj:IsA("Frame") and obj.Name == "Inspect" then inspectPanel = obj; break end
		end
	end

	if inspectPanel and not inspectContainer then
		for _, subChild in ipairs(inspectPanel:GetChildren()) do
			if subChild:IsA("Frame") and subChild.Name == "Container" then
				inspectContainer = subChild
				for _, btn in ipairs(subChild:GetChildren()) do
					if btn:IsA("TextLabel") and btn.Name == "Username" then inspectUsername = btn
					elseif btn:IsA("TextButton") and btn.Name == "Profile" then inspectProfileBtn = btn
					elseif btn:IsA("TextButton") and btn.Name == "Trade" then inspectTradeBtn = btn
					elseif btn:IsA("TextButton") and btn.Name == "Close" then inspectCloseBtn = btn end
				end
			end
		end
	end

	return leaderboardContainer, playerTemplate, inspectPanel, inspectContainer, inspectUsername, inspectProfileBtn, inspectTradeBtn, inspectCloseBtn
end

local function initFakeLeaderboard()
	local function getRoman(level)
		local romanNumerals = { [0] = "", [1] = "I", [2] = "II", [3] = "III", [4] = "IV", [5] = "V",
				[6] = "VI", [7] = "VII", [8] = "VIII", [9] = "IX", [10] = "X" }
		local num = level % 10
		if num == 0 then num = 10 end
		return romanNumerals[num] or ""
	end
	local function getLevelIcon(level)
		if level >= 91 then return "439188281"
		elseif level >= 71 then return "439188280"
		elseif level >= 41 then return "439188276"
		else return "439188275" end
	end

	local function findFirstDescendant(parent, name)
		for _, child in ipairs(parent:GetDescendants()) do
			if child.Name == name then return child end
		end
		return nil
	end

	local function hideAllTradeWindows()
		local PlayerGui = Players.LocalPlayer:FindFirstChild("PlayerGui")
		if not PlayerGui then return end
		local tradeRequest = findFirstDescendant(PlayerGui, "TradeRequest")
		if not tradeRequest then return end
		local receiving = tradeRequest:FindFirstChild("ReceivingRequest")
		if receiving then
			for _, btn in ipairs(receiving:GetDescendants()) do
				if btn:IsA("TextButton") and (btn.Name == "Decline" or btn.Name == "Reject" or btn.Name == "Cancel") then
					pcall(function() btn:Activate() end)
					pcall(function() btn.MouseButton1Click:Fire() end)
				end
			end
			pcall(function() receiving.Visible = false end)
		end
		local sending = tradeRequest:FindFirstChild("SendingRequest")
		if sending then
			for _, btn in ipairs(sending:GetDescendants()) do
				if btn:IsA("TextButton") and (btn.Name == "Cancel" or btn.Name == "Decline" or btn.Name == "Reject") then
					pcall(function() btn:Activate() end)
					pcall(function() btn.MouseButton1Click:Fire() end)
				end
			end
			pcall(function() sending.Visible = false end)
		end
		pcall(function() tradeRequest.Visible = false end)
	end

	local function resetTradeState(partnerName)
		pcall(function() DeclineTrade() end)
		task.wait(0.05)
		hideAllTradeWindows()
		task.wait(0.05)
		LastTradePartner = partnerName
		TradeTable.Player2.Player = partnerName
		TradeTable.Player2.Accepted = false
		TradeTable.Player2.Offer = {}
		TradeTable.Player1.Accepted = false
		TradeTable.Player1.Offer = {}
		TradeTable.Locked = false
		TradeTable.LastOffer = os.time()
		Config.in_trade = false
		if PartnerUserBox then PartnerUserBox.Text = partnerName end
	end

	local function sendFakeTradeRequest(targetName, delaySeconds)
		local PlayerGui = Players.LocalPlayer:FindFirstChild("PlayerGui")
		if not PlayerGui then return false end
		hideAllTradeWindows()
		task.wait(0.1)

		local tradeRequest = findFirstDescendant(PlayerGui, "TradeRequest")
		if not tradeRequest then warn("[FakeTradeSend] TradeRequest не найден"); return false end

		local sendingRequest = tradeRequest:FindFirstChild("SendingRequest")
		if not sendingRequest then warn("[FakeTradeSend] SendingRequest не найден"); return false end

		local receivingRequest = tradeRequest:FindFirstChild("ReceivingRequest")
		if receivingRequest then receivingRequest.Visible = false end

		local usernameLabel = sendingRequest:FindFirstChild("Username")
		if usernameLabel and usernameLabel:IsA("TextLabel") then
			usernameLabel.Text = targetName
		end

		tradeRequest.Visible = true
		sendingRequest.Visible = true
		print(("✅ Фейк-трейд: %s (принятие через %.1f сек)"):format(targetName, delaySeconds))

		local cancelled = false
		local watchConn
		watchConn = RunService.Heartbeat:Connect(function()
			if cancelled then return end
			if not tradeRequest or not tradeRequest.Parent then return end
			local sr = tradeRequest:FindFirstChild("SendingRequest")
			if not sr or not sr.Visible or not tradeRequest.Visible then
				cancelled = true
				print("[FakeTradeSend] ❌ SendingRequest закрыт — отмена")
				if watchConn then watchConn:Disconnect(); watchConn = nil end
			end
		end)

		task.delay(delaySeconds, function()
			if watchConn then watchConn:Disconnect(); watchConn = nil end
			if cancelled then hideAllTradeWindows(); return end
			hideAllTradeWindows()
			task.wait(0.05)
			resetTradeState(targetName)
			task.delay(0.05, function() TradeTable.Player2.Player = targetName end)
			StartTrade()
		end)

		return true
	end

	local InspectConnections = {}

	local function disconnectInspect()
		for _, c in ipairs(InspectConnections) do pcall(function() c:Disconnect() end) end
		InspectConnections = {}
		InspectConnected = false
	end

	local function setupInspectConnections()
		local lb, pt, ip, ic, iu, ipb, itb, icb = findLeaderboardAndInspect()
		if not ip then return false end

		disconnectInspect()
		InspectPanelRef = ip

		local function hideInspect()
			if ip then ip.Visible = false end
		end

		if icb then
			table.insert(InspectConnections, icb.MouseButton1Click:Connect(hideInspect))
		end

		if ipb then
			table.insert(InspectConnections, ipb.MouseButton1Click:Connect(function()
				local name = iu and iu.Text or "Unknown"
				print("📋 Профиль:", name)
			end))
		end

		if itb then
			table.insert(InspectConnections, itb.MouseButton1Click:Connect(function()
				local name = iu and iu.Text or "Unknown"
				if not name or name == "" or name == "Unknown" then return end
				local isFake = PersistentLeaderboardFakes[name] ~= nil
				pcall(function() DeclineTrade() end)
				task.wait(0.15)
				resetTradeState(name)
				hideInspect()
				if isFake then
					print("🎭 Фейк-трейд игроку:", name)
					task.wait(0.2)
					sendFakeTradeRequest(name, FakeTradeAcceptDelay or 3)
				else
					print("✅ Реальный игрок выбран:", name)
				end
			end))
		end

		InspectConnected = true
		return true
	end

	local function nameExistsInLeaderboard(name, container)
		if not container or not container.Parent then return false end
		for _, child in ipairs(container:GetChildren()) do
			if child:IsA("Frame") then
				local label = child:FindFirstChild("PlayerLabel") or child:FindFirstChild("Name")
				if label and label:IsA("TextLabel") and label.Text == name then return true end
			end
		end
		return false
	end

	local function createFakeLeaderboardPlayer(name, level)
		local container, template = findLeaderboardAndInspect()
		if not container or not container.Parent then return nil end
		if nameExistsInLeaderboard(name, container) then return nil end

		local minOrder = 0
		for _, child in ipairs(container:GetChildren()) do
			if child:IsA("Frame") then
				if child.LayoutOrder < minOrder then minOrder = child.LayoutOrder end
			end
		end

		local fake = nil
		local roman = getRoman(level)
		local prestigeText = roman ~= "" and roman .. " " or ""
		local iconId = getLevelIcon(level)

		if template and template.Parent then
			fake = template:Clone()
			fake.Name = name
			local playerLabel = fake:FindFirstChild("PlayerLabel") or fake:FindFirstChild("Name")
			if playerLabel and playerLabel:IsA("TextLabel") then
				playerLabel.Text = name
				playerLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
			end
			local levelImg = fake:FindFirstChild("Level")
			if levelImg and levelImg:IsA("ImageLabel") then
				levelImg.Image = "https://www.roblox.com/asset/?id=" .. iconId
				levelImg.Visible = true
				local prestige = levelImg:FindFirstChild("Prestige")
				if prestige and prestige:IsA("TextLabel") then prestige.Text = prestigeText end
				local levelText = levelImg:FindFirstChild("Level")
				if levelText and levelText:IsA("TextLabel") then levelText.Text = tostring(level) end
			end
			fake.LayoutOrder = minOrder - 1
			fake.Visible = true
			fake.Parent = container
		else
			fake = Instance.new("Frame")
			fake.Name = name
			fake.Size = UDim2.new(1, 0, 0, 0.083)
			fake.BackgroundColor3 = Color3.fromRGB(31, 31, 31)
			fake.BackgroundTransparency = 0.5
			fake.BorderSizePixel = 0
			fake.LayoutOrder = minOrder - 1

			local playerLabel = Instance.new("TextLabel")
			playerLabel.Name = "PlayerLabel"
			playerLabel.Size = UDim2.new(0.725, 0, 0.55, 0)
			playerLabel.Position = UDim2.new(0.025, 0, 0.5, 0)
			playerLabel.AnchorPoint = Vector2.new(0, 0.5)
			playerLabel.BackgroundTransparency = 1
			playerLabel.Text = name
			playerLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
			playerLabel.TextSize = 18
			playerLabel.Font = Enum.Font.SourceSans
			playerLabel.TextXAlignment = Enum.TextXAlignment.Left
			playerLabel.Parent = fake

			local levelImg = Instance.new("ImageLabel")
			levelImg.Name = "Level"
			levelImg.Size = UDim2.new(1, 0, 1, 0)
			levelImg.Position = UDim2.new(1, -3, 0.5, 0)
			levelImg.AnchorPoint = Vector2.new(1, 0.5)
			levelImg.BackgroundTransparency = 1
			levelImg.Image = "https://www.roblox.com/asset/?id=" .. iconId
			levelImg.ScaleType = Enum.ScaleType.Stretch
			levelImg.Parent = fake

			local levelText = Instance.new("TextLabel")
			levelText.Name = "Level"
			levelText.Size = UDim2.new(1, 0, 1, 0)
			levelText.BackgroundTransparency = 1
			levelText.Text = tostring(level)
			levelText.TextColor3 = Color3.fromRGB(255, 255, 255)
			levelText.TextSize = 14
			levelText.Font = Enum.Font.SourceSerifPro
			levelText.Parent = levelImg

			fake.Parent = container
		end

		local clickButton = fake:FindFirstChild("ActionButton") or fake:FindFirstChildWhichIsA("TextButton")
		local function clickHandler()
			local _, _, ip2, ic2, iu2 = findLeaderboardAndInspect()
			if not ip2 then return end
			if iu2 then iu2.Text = name end
			local friendsLabel = ip2:FindFirstChild("FriendsPlaying") or (ic2 and ic2:FindFirstChild("FriendsPlaying"))
			if friendsLabel and friendsLabel:IsA("TextLabel") then
				friendsLabel.Text = "Friends Playing: " .. math.random(0, 15)
			end
			local tradeRequestsLabel = ip2:FindFirstChild("TradeRequests") or (ic2 and ic2:FindFirstChild("TradeRequests"))
			if tradeRequestsLabel and tradeRequestsLabel:IsA("TextLabel") then
				tradeRequestsLabel.Text = "Trade Requests: " .. math.random(0, 5)
			end
			local levelImg = ip2:FindFirstChild("Level") or (ic2 and ic2:FindFirstChild("Level"))
			if levelImg and levelImg:IsA("ImageLabel") then
				levelImg.Image = "https://www.roblox.com/asset/?id=" .. getLevelIcon(level)
				local prestige = levelImg:FindFirstChild("Prestige")
				if prestige and prestige:IsA("TextLabel") then
					local roman = getRoman(level)
					prestige.Text = roman ~= "" and roman .. " " or ""
				end
				local levelText = levelImg:FindFirstChild("Level")
				if levelText and levelText:IsA("TextLabel") then levelText.Text = tostring(level) end
			end
			ip2.Visible = true
		end
		if clickButton then
			clickButton.MouseButton1Click:Connect(clickHandler)
		else
			fake.MouseButton1Click:Connect(clickHandler)
		end

		return fake
	end

	local function restoreAllFakes()
		task.wait(0.3)
		setupInspectConnections()
		local container = findLeaderboardAndInspect()
		if not container or not container.Parent then return end
		for name, data in pairs(PersistentLeaderboardFakes) do
			if not nameExistsInLeaderboard(name, container) then
				createFakeLeaderboardPlayer(name, data.level)
			end
		end
	end

	task.spawn(function()
		while true do
			task.wait(2)
			local container, _, ip = findLeaderboardAndInspect()
			if ip and (not InspectConnected or ip ~= InspectPanelRef) then
				setupInspectConnections()
			end
			if container and container.Parent then
				local needRestore = false
				for name, _ in pairs(PersistentLeaderboardFakes) do
					if not nameExistsInLeaderboard(name, container) then needRestore = true; break end
				end
				if needRestore then restoreAllFakes() end
			end
		end
	end)

	Players.LocalPlayer.CharacterAdded:Connect(function()
		task.wait(3)
		disconnectInspect()
		InspectPanelRef = nil
		setupInspectConnections()
		restoreAllFakes()
	end)

	task.defer(function()
		task.wait(1)
		setupInspectConnections()
	end)
end

initFakeLeaderboard()

-- ============================================================
-- ОСТАЛЬНЫЕ КНОПКИ CONTROL
-- ============================================================
CreateButton(controlFrame, "🚀 Start Trade (Quick)", function()
	local partnerName = PartnerUserBox.Text
	if partnerName == "" then partnerName = "alexbestomg" end
	TradeTable.Player2.Player = partnerName
	StartTrade()
end)
CreateSpace(controlFrame)

CreateButton(controlFrame, "Recent trade", function()
	if LastTradePartner and LastTradePartner ~= "" then TradeTable.Player2.Player = LastTradePartner; PartnerUserBox.Text = LastTradePartner end
end)
CreateSpace(controlFrame)

local RandomPlayerNames = {
	"CAXAROK_666", "need_money16", "Ler4eg", "JenYAsha", "im_moggYou", "slamboygg",
	"dance_pantera00", "shaxedOnSKY", "spiderman0", "dattebaeoxae", "aszoo00",
	"BI4UKXA", "Lelilkzar", "Zyleak", "Nikilis", "Sweete_fox", "eva_elfie",
	"fat_grandma", "litvin_condicioner", "livingdayroom", "monsterXboss",
}

CreateButton(controlFrame, "Random player", function()
	local name = RandomPlayerNames[math.random(1, #RandomPlayerNames)]
	TradeTable.Player2.Player = name
	PartnerUserBox.Text = name
end)
CreateSpace(controlFrame)

CreateButton(controlFrame, "Accept their offer", function()
	if not next(TradeTable["Player1"]["Offer"]) and not next(TradeTable["Player2"]["Offer"]) then return end
	if v84 then return end
	TheirOffer.Accepted.Visible = true
	TradeTable["Player2"]["Accepted"] = true
	AcceptTrade()
end)
CreateSpace(controlFrame)

CreateButton(controlFrame, "👤 Friend Joined", function()
	local username = FriendJoinCustomUsers[math.random(1, #FriendJoinCustomUsers)] or "RobloxPlayer"
	showFakeFriendJoinNotification(username)
	print("📨 Friend joined notification for: " .. username)
end)
CreateSpace(controlFrame)

local function SpawnFakePlayer()
	local playerName = PartnerUserBox.Text
	if playerName == "" then warn("[FakePlayer] Please enter a username in the 'Partner user' box."); return end
	if FakePlayers[playerName] and FakePlayers[playerName].Parent then
		if FakePlayerTasks[playerName] then task.cancel(FakePlayerTasks[playerName]); FakePlayerTasks[playerName] = nil end
		StartBotBehavior(playerName, FakePlayers[playerName])
		print("[FakePlayer] Bot already exists, restarted behavior: " .. playerName)
		return
	end
	local ok, userId = pcall(function() return Players:GetUserIdFromNameAsync(playerName) end)
	if not ok or not userId then warn("[FakePlayer] Username not found: " .. playerName); return end
	local ok2, humanoidDesc = pcall(function() return Players:GetHumanoidDescriptionFromUserId(userId) end)
	if not ok2 or not humanoidDesc then warn("[FakePlayer] Failed to get avatar for " .. playerName); return end
	local character = Players.LocalPlayer.Character
	if not character then return end
	local root = character:FindFirstChild("HumanoidRootPart")
	if not root then return end
	local bot = Players:CreateHumanoidModelFromDescription(humanoidDesc, Enum.HumanoidRigType.R15)
	bot.Name = playerName
	bot:PivotTo(root.CFrame * CFrame.new(0, 0, -6))
	bot.Parent = workspace
	FakePlayers[playerName] = bot
	local humanoid = bot:FindFirstChildOfClass("Humanoid")
	if humanoid then
		humanoid.RigType = Enum.HumanoidRigType.R15
		humanoid.WalkSpeed = 16
		humanoid.JumpPower = 55
	end
	task.wait(0.5)
	StartBotBehavior(playerName, bot)
	print("[FakePlayer] Spawned " .. playerName .. " with DEX animations!")
end

CreateButton(controlFrame, "Spawn fake player", SpawnFakePlayer)
CreateSpace(controlFrame)

local function DeleteAllFakePlayers()
	local count = 0
	local keys = {}
	for name, _ in pairs(FakePlayers) do table.insert(keys, name) end
	for _, name in ipairs(keys) do
		local bot = FakePlayers[name]
		if FakePlayerTasks[name] then task.cancel(FakePlayerTasks[name]); FakePlayerTasks[name] = nil end
		if bot then pcall(function() if bot.Parent then bot:Destroy() end end); count = count + 1 end
		FakePlayers[name] = nil
	end
	print("[FakePlayer] Removed " .. count .. " bots")
end

CreateButton(controlFrame, "Delete all fake player", DeleteAllFakePlayers)
CreateSpace(controlFrame)

-- ============================================================
-- SILENT BLOCK
-- ============================================================
local SilentBlockConfig = {
	modalAppearTimeout = 10, modalDismissTimeout = 10, maxAttempts = 20,
	overlayName = "FoundationOverlay", modalName = "BlockingModalScreen"
}

local SilentBlockServices = {
	CoreGui = game:GetService("CoreGui"), StarterGui = game:GetService("StarterGui"),
	RunService = game:GetService("RunService"), GuiService = game:GetService("GuiService"),
	VirtualInputManager = game:GetService("VirtualInputManager")
}

local SilentBlockHideOps = {
	{ class = "ScreenGui", apply = function(n) n.Enabled = false end },
	{ class = "GuiObject", apply = function(n) n.Visible = false; n.BackgroundTransparency = 1 end },
	{ class = "ImageLabel", apply = function(n) n.ImageTransparency = 1 end },
	{ class = "ImageButton", apply = function(n) n.ImageTransparency = 1 end },
	{ class = "TextLabel", apply = function(n) n.TextTransparency = 1 end },
	{ class = "TextButton", apply = function(n) n.TextTransparency = 1 end },
	{ class = "UIStroke", apply = function(n) n.Transparency = 1 end },
}

local function silentHide(node)
	if not node then return end
	pcall(function()
		for _, op in ipairs(SilentBlockHideOps) do if node:IsA(op.class) then pcall(op.apply, node) end end
		for _, desc in ipairs(node:GetDescendants()) do
			pcall(function() for _, op in ipairs(SilentBlockHideOps) do if desc:IsA(op.class) then pcall(op.apply, desc) end end end)
		end
	end)
end

local function findOverlay() return SilentBlockServices.CoreGui:FindFirstChild(SilentBlockConfig.overlayName) end
local function modalStillOpen() local overlay = findOverlay(); return overlay ~= nil and overlay:FindFirstChild(SilentBlockConfig.modalName, true) ~= nil end

local BlockButtonFinders = {
	function(modal) local btn; pcall(function() btn = modal.BlockingModalContainerWrapper.BlockingModal.AlertModal.AlertContents.Footer.Buttons["3"] end); return btn end,
	function(modal) local btn; pcall(function() local c = modal:FindFirstChild("Buttons", true); if c then for _, ch in ipairs(c:GetChildren()) do if ch:IsA("ImageButton") or ch:IsA("TextButton") then local lbl = ch:FindFirstChild("Text"); if lbl and lbl:IsA("TextLabel") and lbl.Text == "Block" then btn = ch; return end end end; if not btn then btn = c:FindFirstChild("3") end end end); return btn end,
	function(modal) local btn; pcall(function() for _, d in ipairs(modal:GetDescendants()) do if d:IsA("ImageButton") or d:IsA("TextButton") then local lbl = d:FindFirstChild("Text"); if lbl and lbl:IsA("TextLabel") and lbl.Text == "Block" then btn = d; return end end end end); return btn end,
}

local function findBlockButton(modal)
	for _, f in ipairs(BlockButtonFinders) do local btn = f(modal); if btn then return btn end end
end

local SilentBlockStrategies = {
	{ name = "VIM-Enter", run = function(btn) pcall(function() SilentBlockServices.GuiService.SelectedObject = btn end); task.wait(); pcall(function() local vim = SilentBlockServices.VirtualInputManager; vim:SendKeyEvent(true, Enum.KeyCode.Return, false, game); vim:SendKeyEvent(false, Enum.KeyCode.Return, false, game) end) end, settle = 0.05, skipCheck = true },
	{ name = "VIM", run = function(btn) pcall(function() local abs = btn.AbsolutePosition; local sz = btn.AbsoluteSize; local cx = abs.X + sz.X / 2; local cy = abs.Y + sz.Y / 2; local vim = SilentBlockServices.VirtualInputManager; vim:SendMouseButtonEvent(cx, cy, 0, true, game, 1); task.wait(); vim:SendMouseButtonEvent(cx, cy, 0, false, game, 1) end) end, settle = 0.15 },
}

local function SilentBlockPlayer(Selected)
	if not Selected then return end
	local playerName = (typeof(Selected) == "Instance" and Selected.Name) or tostring(Selected)
	print("[block] >>> SilentBlockPlayer: " .. playerName)
	pcall(function() setthreadidentity(8) end)
	local preWatchers = {}
	local function watchFor(parent) local conn = parent.DescendantAdded:Connect(function(d) if d.Name == SilentBlockConfig.modalName then silentHide(d); local inner = d.DescendantAdded:Connect(function() silentHide(d) end); table.insert(preWatchers, inner) end end); table.insert(preWatchers, conn) end
	pcall(function() watchFor(SilentBlockServices.CoreGui) end)
	SilentBlockServices.StarterGui:SetCore("PromptBlockPlayer", Selected)
	local startTime = tick()
	local modal = nil
	while not modal do
		SilentBlockServices.RunService.Heartbeat:Wait()
		if tick() - startTime > SilentBlockConfig.modalAppearTimeout then
			warn("[block] modal never appeared")
			for _, c in ipairs(preWatchers) do pcall(function() c:Disconnect() end) end
			pcall(function() setthreadidentity(2) end)
			return
		end
		local overlay = findOverlay()
		if overlay then modal = overlay:FindFirstChild(SilentBlockConfig.modalName, true) end
	end
	silentHide(modal)
	local posConn
	posConn = SilentBlockServices.RunService.Heartbeat:Connect(function() pcall(function() if modal and modal.Parent then silentHide(modal) else posConn:Disconnect() end end) end)
	local blockBtn = findBlockButton(modal)
	if blockBtn then
		local attempts = 0
		while attempts < SilentBlockConfig.maxAttempts do
			attempts = attempts + 1
			local dismissed = false
			for _, strategy in ipairs(SilentBlockStrategies) do
				strategy.run(blockBtn)
				task.wait(strategy.settle)
				if not strategy.skipCheck and not modalStillOpen() then dismissed = true; break end
			end
			if dismissed then break end
		end
		pcall(function() SilentBlockServices.GuiService.SelectedObject = nil end)
	else warn("[block] couldn't find Block button") end
	pcall(function() if posConn then posConn:Disconnect() end end)
	for _, c in ipairs(preWatchers) do pcall(function() c:Disconnect() end) end
	local timeout = tick() + SilentBlockConfig.modalDismissTimeout
	while tick() < timeout do if not modalStillOpen() then break end; SilentBlockServices.RunService.Heartbeat:Wait() end
	pcall(function() setthreadidentity(2) end)
end

CreateButton(controlFrame, "Block player", function()
	pcall(function()
		local Selected = game.Players:FindFirstChild(TradeTable.Player2.Player)
		SilentBlockPlayer(Selected)
	end)
end)

-- ============================================================
-- ITEMS TAB
-- ============================================================
local ItemToAddPartnerBox = createSettingRow("Name item to add:", "", itemsFrame)
CreateSpace(itemsFrame)

CreateButton(itemsFrame, "Add Item To Their Offer", function()
	if ItemToAddPartnerBox.Text and ItemToAddPartnerBox.Text ~= "" then OfferItemAnotherPlayer(ItemToAddPartnerBox.Text, "Weapons") end
end)
CreateSpace(itemsFrame)

CreateButton(itemsFrame, "Remove last Item in Their Offer", function() RemoveItemAnotherPlayer() end)
CreateSpace(itemsFrame)

local weaponListLabel = Instance.new("TextLabel")
weaponListLabel.Size = UDim2.new(1, 0, 0, 15)
weaponListLabel.BackgroundTransparency = 1
weaponListLabel.Text = "Click weapon to ADD directly:"
weaponListLabel.Font = Enum.Font.SourceSansSemibold
weaponListLabel.TextSize = 12
weaponListLabel.TextColor3 = Color3.fromRGB(0, 255, 100)
weaponListLabel.TextXAlignment = Enum.TextXAlignment.Left
weaponListLabel.Parent = itemsFrame

local weaponScrollFrame = Instance.new("ScrollingFrame")
weaponScrollFrame.Size = UDim2.new(1, 0, 0, 120)
weaponScrollFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
weaponScrollFrame.BackgroundTransparency = 0.3
weaponScrollFrame.BorderSizePixel = 0
weaponScrollFrame.ScrollBarThickness = 6
weaponScrollFrame.ScrollBarImageColor3 = PINK
weaponScrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
weaponScrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
weaponScrollFrame.Parent = itemsFrame

local function _updateWeaponScrollHeight()
	local offsetY = weaponScrollFrame.AbsolutePosition.Y - itemsFrame.AbsolutePosition.Y
	local available = itemsFrame.AbsoluteSize.Y - offsetY - 4
	weaponScrollFrame.Size = UDim2.new(1, 0, 0, math.max(80, available))
end
itemsFrame:GetPropertyChangedSignal("AbsoluteSize"):Connect(_updateWeaponScrollHeight)
task.defer(_updateWeaponScrollHeight)

local weaponScrollCorner = Instance.new("UICorner") weaponScrollCorner.CornerRadius = UDim.new(0, 5) weaponScrollCorner.Parent = weaponScrollFrame

local weaponListLayout = Instance.new("UIListLayout")
weaponListLayout.FillDirection = Enum.FillDirection.Vertical
weaponListLayout.SortOrder = Enum.SortOrder.LayoutOrder
weaponListLayout.Padding = UDim.new(0, 2)
weaponListLayout.Parent = weaponScrollFrame

local weaponListPadding = Instance.new("UIPadding")
weaponListPadding.PaddingTop = UDim.new(0, 3)
weaponListPadding.PaddingBottom = UDim.new(0, 3)
weaponListPadding.PaddingLeft = UDim.new(0, 3)
weaponListPadding.PaddingRight = UDim.new(0, 3)
weaponListPadding.Parent = weaponScrollFrame

local function _itemsTabNormalize(s)
	s = string.lower(tostring(s or ""))
	s = string.gsub(s, "^c%.%s*", "chroma ")
	s = string.gsub(s, "(%s)c%.%s*", "%1chroma ")
	s = string.gsub(s, "['\u{2019}\"]", "")
	s = string.gsub(s, "%s+", " ")
	s = string.gsub(s, "^%s+", "")
	s = string.gsub(s, "%s+$", "")
	return s
end

local ItemsTabAllowedNames = {
	"Corrupt", "Chroma Traveler's Gun", "Chroma Evergun", "Chroma Evergreen",
	"Chroma Bauble", "Chroma Vampire's Gun", "Chroma Constellation",
	"Chroma Alienbeam", "Chroma Raygun", "Chroma Sunrise", "Chroma Snowcannon",
	"Chroma Blizzard", "Chroma Sunset", "Chroma Snow Dagger", "Chroma Heart Wand",
	"Chroma Treat", "Chroma Snowstorm", "Chroma Watergun", "Chroma Sweet",
	"Chroma Ornament", "Gingerscope", "Traveler's Axe", "Traveler's Gun",
	"Evergreen", "Evergun", "Celestial", "Constellation", "Turkey",
	"Alienbeam", "Raygun", "Vampire's Gun", "Darkshot", "Darksword",
	"Blossom", "Sakura", "Sunset", "Sunrise", "Bauble", "Snowcannon",
	"Heart Wand", "Snowstorm", "Snow Dagger", "Blizzard", "Watergun",
	"Sweet", "Ornament", "Harvester", "Icepiercer", "Bloom", "Flora",
	"Rainbow", "Rainbow Gun", "Nik scythe",
}

local _rarityRank = { Chroma = 10, Godly = 9, Ancient = 8, Unique = 7, Classic = 6, Legendary = 5, Vintage = 4, Rare = 3, Uncommon = 2, Common = 1 }

local allWeaponsList = {}
local _seenKeys = {}

for _, name in ipairs(ItemsTabAllowedNames) do
	local target = _itemsTabNormalize(name)
	local wantsChroma = string.find(target, "^chroma ") ~= nil
	local targetStripped = string.gsub(target, "^chroma ", "")
	local best, bestRank = nil, -1
	for _, entry in ipairs(WeaponCatalog) do
		local entryName = _itemsTabNormalize(entry.name)
		local entryIsChroma = entry.chroma == true
		local nameOk = false
		if wantsChroma then
			if entryIsChroma and (entryName == target or entryName == targetStripped) then nameOk = true end
		else
			if (not entryIsChroma) and entryName == target then nameOk = true end
		end
		if nameOk then
			local rank = _rarityRank[entry.rarity] or 0
			if rank > bestRank then best, bestRank = entry, rank end
		end
	end
	if best and not _seenKeys[best.key] then
		table.insert(allWeaponsList, best)
		_seenKeys[best.key] = true
	end
end

local RarityTint = {
	Chroma = Color3.fromRGB(70, 40, 95), Godly = Color3.fromRGB(110, 70, 30),
	Ancient = Color3.fromRGB(60, 25, 90), Unique = Color3.fromRGB(140, 50, 90),
	Legendary = Color3.fromRGB(95, 55, 25), Classic = Color3.fromRGB(70, 70, 90),
	Vintage = Color3.fromRGB(80, 75, 30), Rare = Color3.fromRGB(35, 60, 95),
	Uncommon = Color3.fromRGB(35, 70, 50), Common = Color3.fromRGB(50, 50, 70)
}

local weaponButtons = {}

for i, entry in ipairs(allWeaponsList) do
	local wKey = entry.key
	local baseColor = RarityTint[entry.rarity] or RarityTint.Common
	local label = entry.name .. (entry.chroma and " [Chroma]" or "") .. " (" .. entry.rarity .. " " .. entry.type .. ")"

	local weaponBtn = Instance.new("TextButton")
	weaponBtn.Size = UDim2.new(1, -6, 0, 22)
	weaponBtn.BackgroundColor3 = baseColor
	weaponBtn.BackgroundTransparency = 0.2
	weaponBtn.Text = label
	weaponBtn.Font = Enum.Font.SourceSans
	weaponBtn.TextSize = 12
	weaponBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	weaponBtn.TextXAlignment = Enum.TextXAlignment.Left
	weaponBtn.TextTruncate = Enum.TextTruncate.AtEnd
	weaponBtn.Parent = weaponScrollFrame

	local btnPadding = Instance.new("UIPadding")
	btnPadding.PaddingLeft = UDim.new(0, 6)
	btnPadding.PaddingRight = UDim.new(0, 6)
	btnPadding.Parent = weaponBtn

	local btnCorner = Instance.new("UICorner") btnCorner.CornerRadius = UDim.new(0, 4) btnCorner.Parent = weaponBtn

	weaponBtn.MouseButton1Click:Connect(function()
		local success = OfferItemAnotherPlayer(wKey, "Weapons")
		if success then
			TweenService:Create(weaponBtn, TweenInfo.new(0.1), {BackgroundColor3 = Color3.fromRGB(0, 150, 100)}):Play()
		else
			TweenService:Create(weaponBtn, TweenInfo.new(0.1), {BackgroundColor3 = Color3.fromRGB(150, 50, 50)}):Play()
		end
		task.delay(0.2, function()
			TweenService:Create(weaponBtn, TweenInfo.new(0.15), {BackgroundColor3 = baseColor}):Play()
		end)
	end)

	weaponButtons[#weaponButtons + 1] = {button = weaponBtn, entry = entry}
end

ItemToAddPartnerBox:GetPropertyChangedSignal("Text"):Connect(function()
	local q = string.lower(ItemToAddPartnerBox.Text or "")
	for _, info in ipairs(weaponButtons) do
		if q == "" then info.button.Visible = true
		else
			local e = info.entry
			local hay = string.lower(e.name .. " " .. e.key .. " " .. e.rarity .. " " .. e.type)
			info.button.Visible = string.find(hay, q, 1, true) ~= nil
		end
	end
end)

-- ============================================================
-- SPAWNER TAB
-- ============================================================
local SpawnerRandomRanges = {
	Chroma = {1, 2}, Godly = {1, 5}, Ancient = {2, 6}, Unique = {2, 8},
	Classic = {3, 10}, Legendary = {4, 12}, Vintage = {5, 15},
	Rare = {8, 25}, Uncommon = {10, 40}, Common = {15, 60}
}

local SpawnerHighTierSet = { Chroma = true, Godly = true, Ancient = true, Unique = true, Classic = true, Legendary = true, Vintage = true }

local function _randomAmount(rarity, evo)
	if evo then return 1 end
	local r = SpawnerRandomRanges[rarity] or SpawnerRandomRanges.Common
	return math.random(r[1], r[2])
end

local SpawnerAmountBox = createSettingRow("Amount per click (0 = random):", "0", spawnerFrame)
CreateSpace(spawnerFrame)
local SpawnerSearchBox = createSettingRow("Search weapon:", "", spawnerFrame)
CreateSpace(spawnerFrame)

local spawnerStatusLabel = Instance.new("TextLabel")
spawnerStatusLabel.Size = UDim2.new(1, 0, 0, 15)
spawnerStatusLabel.BackgroundTransparency = 1
spawnerStatusLabel.Text = "Click weapon to spawn:"
spawnerStatusLabel.Font = Enum.Font.SourceSansSemibold
spawnerStatusLabel.TextSize = 12
spawnerStatusLabel.TextColor3 = Color3.fromRGB(0, 255, 100)
spawnerStatusLabel.TextXAlignment = Enum.TextXAlignment.Left
spawnerStatusLabel.Parent = spawnerFrame

CreateButton(spawnerFrame, "Spawn High Tier (Tradable)", function()
	local count, total = 0, 0
	for _, entry in ipairs(WeaponCatalog) do
		if SpawnerHighTierSet[entry.rarity] and entry.name ~= "Nik scythe" then
			local amt = _randomAmount(entry.rarity, false)
			SpawnItem(entry.key, amt, "Weapons")
			count = count + 1
			total = total + amt
		end
	end
	spawnerStatusLabel.Text = ("Spawned %d weapons (%d items)"):format(count, total)
	spawnerStatusLabel.TextColor3 = Color3.fromRGB(120, 255, 160)
end)
CreateSpace(spawnerFrame)

CreateButton(spawnerFrame, "Purge untradables from inventory", function()
	local removed = PurgeBlockedFromInventory()
	spawnerStatusLabel.Text = ("Removed %d untradable weapon(s)"):format(removed)
	spawnerStatusLabel.TextColor3 = (removed > 0) and Color3.fromRGB(120, 255, 160) or Color3.fromRGB(255, 200, 120)
end)
CreateSpace(spawnerFrame)

local spawnerScrollFrame = Instance.new("ScrollingFrame")
spawnerScrollFrame.Size = UDim2.new(1, 0, 0, 120)
spawnerScrollFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
spawnerScrollFrame.BackgroundTransparency = 0.3
spawnerScrollFrame.BorderSizePixel = 0
spawnerScrollFrame.ScrollBarThickness = 6
spawnerScrollFrame.ScrollBarImageColor3 = PINK
spawnerScrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
spawnerScrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
spawnerScrollFrame.Parent = spawnerFrame

local function _updateSpawnerScrollHeight()
	local offsetY = spawnerScrollFrame.AbsolutePosition.Y - spawnerFrame.AbsolutePosition.Y
	local available = spawnerFrame.AbsoluteSize.Y - offsetY - 4
	spawnerScrollFrame.Size = UDim2.new(1, 0, 0, math.max(80, available))
end
spawnerFrame:GetPropertyChangedSignal("AbsoluteSize"):Connect(_updateSpawnerScrollHeight)
task.defer(_updateSpawnerScrollHeight)

local c = Instance.new("UICorner") c.CornerRadius = UDim.new(0, 5) c.Parent = spawnerScrollFrame

local lay = Instance.new("UIListLayout")
lay.FillDirection = Enum.FillDirection.Vertical
lay.SortOrder = Enum.SortOrder.LayoutOrder
lay.Padding = UDim.new(0, 2)
lay.Parent = spawnerScrollFrame

local pad = Instance.new("UIPadding")
pad.PaddingTop = UDim.new(0, 3)
pad.PaddingBottom = UDim.new(0, 3)
pad.PaddingLeft = UDim.new(0, 3)
pad.PaddingRight = UDim.new(0, 3)
pad.Parent = spawnerScrollFrame

local spawnerButtons = {}

for _, entry in ipairs(WeaponCatalog) do
	if entry.name == "Nik scythe" then continue end
	local wKey = entry.key
	local baseColor = RarityTint[entry.rarity] or RarityTint.Common
	local label = entry.name .. (entry.chroma and " [Chroma]" or "") .. " (" .. entry.rarity .. " " .. entry.type .. ")"

	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1, -6, 0, 22)
	btn.BackgroundColor3 = baseColor
	btn.BackgroundTransparency = 0.2
	btn.Text = label
	btn.Font = Enum.Font.SourceSans
	btn.TextSize = 12
	btn.TextColor3 = Color3.fromRGB(255, 255, 255)
	btn.TextXAlignment = Enum.TextXAlignment.Left
	btn.TextTruncate = Enum.TextTruncate.AtEnd
	btn.Parent = spawnerScrollFrame

	local btnPad = Instance.new("UIPadding")
	btnPad.PaddingLeft = UDim.new(0, 6)
	btnPad.PaddingRight = UDim.new(0, 6)
	btnPad.Parent = btn

	local btnCorner = Instance.new("UICorner") btnCorner.CornerRadius = UDim.new(0, 4) btnCorner.Parent = btn

	btn.MouseButton1Click:Connect(function()
		local typed = tonumber(SpawnerAmountBox.Text)
		local amt = (typed and typed > 0) and typed or _randomAmount(entry.rarity, false)
		SpawnItem(wKey, amt, "Weapons")
		spawnerStatusLabel.Text = ("Spawned %s x%d"):format(entry.name, amt)
		spawnerStatusLabel.TextColor3 = Color3.fromRGB(120, 255, 160)
		TweenService:Create(btn, TweenInfo.new(0.1), {BackgroundColor3 = Color3.fromRGB(0, 150, 100)}):Play()
		task.delay(0.2, function()
			TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = baseColor}):Play()
		end)
	end)

	spawnerButtons[#spawnerButtons + 1] = {button = btn, entry = entry}
end

SpawnerSearchBox:GetPropertyChangedSignal("Text"):Connect(function()
	local q = string.lower(SpawnerSearchBox.Text or "")
	for _, info in ipairs(spawnerButtons) do
		if q == "" then info.button.Visible = true
		else
			local e = info.entry
			local hay = string.lower(e.name .. " " .. e.key .. " " .. e.rarity .. " " .. e.type)
			info.button.Visible = string.find(hay, q, 1, true) ~= nil
		end
	end
end)

-- ============================================================
-- PLAYERS TAB
-- ============================================================
local playersScroll = Instance.new("ScrollingFrame")
playersScroll.Size = UDim2.new(1, 0, 1, -20)
playersScroll.LayoutOrder = 2
playersScroll.BackgroundTransparency = 1
playersScroll.BorderSizePixel = 0
playersScroll.ScrollBarThickness = 6
playersScroll.ScrollBarImageColor3 = PINK
playersScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
playersScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
playersScroll.Parent = playersFrame

local playersScrollLayout = Instance.new("UIListLayout")
playersScrollLayout.FillDirection = Enum.FillDirection.Vertical
playersScrollLayout.SortOrder = Enum.SortOrder.LayoutOrder
playersScrollLayout.Padding = UDim.new(0, 4)
playersScrollLayout.Parent = playersScroll

local function destroyAllRows()
	for _, child in pairs(playersScroll:GetChildren()) do
		if not (child:IsA("UIListLayout") or child:IsA("UIPadding")) then child:Destroy() end
	end
end

local function createPlayerRow(player)
	local container = Instance.new("Frame")
	container.Size = UDim2.new(1, 0, 0, 30)
	container.BackgroundTransparency = 1
	container.Parent = playersScroll

	local header = Instance.new("TextButton")
	header.Size = UDim2.new(1, 0, 1, 0)
	header.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
	header.BackgroundTransparency = 0.15
	header.Text = ""
	header.AutoButtonColor = false
	header.Parent = container

	local hCorner = Instance.new("UICorner") hCorner.CornerRadius = UDim.new(0, 5) hCorner.Parent = header

	local nameLabel = Instance.new("TextLabel")
	nameLabel.Size = UDim2.new(0.7, -10, 1, 0)
	nameLabel.Position = UDim2.new(0, 8, 0, 0)
	nameLabel.BackgroundTransparency = 1
	nameLabel.Text = player.Name
	nameLabel.Font = Enum.Font.FredokaOne
	nameLabel.TextSize = 13
	nameLabel.TextColor3 = Color3.fromRGB(240, 240, 255)
	nameLabel.TextXAlignment = Enum.TextXAlignment.Left
	nameLabel.TextTruncate = Enum.TextTruncate.AtEnd
	nameLabel.Parent = header

	header.MouseButton1Click:Connect(function()
		TradeTable.Player2.Player = player.Name
		PartnerUserBox.Text = player.Name
		setActiveTab("Control")
	end)

	return container
end

local function UpdatePlayers()
	destroyAllRows()
	for _, player in pairs(game.Players:GetPlayers()) do
		if player ~= game.Players.LocalPlayer then createPlayerRow(player) end
	end
end

task.defer(UpdatePlayers)
game.Players.PlayerAdded:Connect(UpdatePlayers)
game.Players.PlayerRemoving:Connect(UpdatePlayers)

print("=========================================")
print(" ✅ MM2 TRADE HUB LOADED!")
print("=========================================")
print("👤 Friend Joined → фейк-сообщение + топ-тост")
print("🖱️ Тяни за заголовок — двигать")
print("📐 Тяни за углы (◤ ◥ ◣ ◢) — менять размер")
print("🎯 'alexbestomg on dc' — по центру")
print("=========================================")
