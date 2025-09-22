--=== Module: YouTuber Database ===--
local YouTuberDatabase = {
    list = {
        {id = "PewDiePie", displayName = "PewDiePie", subs = 111000000, isFlying = true},
        {id = "GamerOne", displayName = "GamerOne", subs = 78000, isFlying = false},
        {id = "VlogStar", displayName = "VlogStar", subs = 250000, isFlying = false},
        {id = "ProStreamer", displayName = "ProStreamer", subs = 1200000, isFlying = true},
    }
}
local RARITY_TIERS = {
    {name = "Common", min = 50000, max = 100000},
    {name = "Uncommon", min = 100001, max = 500000},
    {name = "Rare", min = 500001, max = 900000},
    {name = "Legendary", min = 900001, max = 1000000},
    {name = "Mythic", min = 1000001, max = 10000000},
    {name = "Godly", min = 10000001, max = 30000000},
    {name = "Secret", min = 30000001, max = 50000000},
    {name = "OG", min = 50000001, max = 9999999999}
}
local function getRarityBySubs(subs)
    for _, tier in ipairs(RARITY_TIERS) do
        if subs >= tier.min and subs <= tier.max then
            return tier.name
        end
    end
    return "Common"
end
local function computeIncomeFromSubs(subs)
    local log = math.log10(math.max(subs, 50000))
    local income = math.floor((log ^ 3) * 8)
    return math.max(1, income)
end
YouTuberDatabase.byId = {}
for _, entry in ipairs(YouTuberDatabase.list) do
    entry.rarity = getRarityBySubs(entry.subs)
    entry.baseIncome = computeIncomeFromSubs(entry.subs)
    entry.sellPrice = math.floor(entry.baseIncome * 20)
    YouTuberDatabase.byId[entry.id] = entry
end

--=== Module: Items Database ===--
local ItemDatabase = {
    ["SpeedPotion"] = {displayName = "Speed Potion", coinPrice = 50, effect = "speed"},
    ["DoubleIncome"] = {displayName = "Double Income", coinPrice = 200, effect = "double_income"},
    ["MegaYT"] = {displayName = "Mega YouTuber", robuxPrice = 99, devProductId = 12345678, effect = "mega_yt"}, -- Replace with your dev product ID
}

--=== Remotes Setup ===--
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RemoteFolder = ReplicatedStorage:FindFirstChild("YT_Tycoon_Remotes") or Instance.new("Folder", ReplicatedStorage)
RemoteFolder.Name = "YT_Tycoon_Remotes"

local BuyYTRemote = RemoteFolder:FindFirstChild("BuyYouTuber") or Instance.new("RemoteEvent", RemoteFolder)
BuyYTRemote.Name = "BuyYouTuber"
local SellYTRemote = RemoteFolder:FindFirstChild("SellYouTuber") or Instance.new("RemoteEvent", RemoteFolder)
SellYTRemote.Name = "SellYouTuber"
local GetOwnedYTRemote = RemoteFolder:FindFirstChild("GetOwnedYouTubers") or Instance.new("RemoteFunction", RemoteFolder)
GetOwnedYTRemote.Name = "GetOwnedYouTubers"
local RequestIndexRemote = RemoteFolder:FindFirstChild("RequestIndex") or Instance.new("RemoteFunction", RemoteFolder)
RequestIndexRemote.Name = "RequestIndex"
local RebirthRemote = RemoteFolder:FindFirstChild("Rebirth") or Instance.new("RemoteEvent", RemoteFolder)
RebirthRemote.Name = "Rebirth"
local BuyItemRemote = RemoteFolder:FindFirstChild("BuyItem") or Instance.new("RemoteEvent", RemoteFolder)
BuyItemRemote.Name = "BuyItem"

--=== Leaderstats & Rebirths ===--
local Players = game:GetService("Players")
local runtime = {}

Players.PlayerAdded:Connect(function(player)
    local leaderstats = Instance.new("Folder")
    leaderstats.Name = "leaderstats"
    leaderstats.Parent = player
    local cash = Instance.new("IntValue")
    cash.Name = "Cash"
    cash.Value = 30000
    cash.Parent = leaderstats
    local coins = Instance.new("IntValue")
    coins.Name = "Coins"
    coins.Value = 0
    coins.Parent = leaderstats
    local rebirths = Instance.new("IntValue")
    rebirths.Name = "Rebirths"
    rebirths.Value = 0
    rebirths.Parent = leaderstats
    runtime[player.UserId] = {cash = cash.Value, coins = coins.Value, rebirths = rebirths.Value, owned = {}}
end)
Players.PlayerRemoving:Connect(function(player)
    runtime[player.UserId] = nil
end)

--=== Rebirth Logic ===--
RebirthRemote.OnServerEvent:Connect(function(player)
    local leaderstats = player:WaitForChild("leaderstats")
    local cash = leaderstats:FindFirstChild("Cash")
    local rebirths = leaderstats:FindFirstChild("Rebirths")
    local coins = leaderstats:FindFirstChild("Coins")
    local requiredCash = 100000 * (rebirths.Value + 1)
    if cash.Value >= requiredCash then
        cash.Value = 0
        rebirths.Value = rebirths.Value + 1
        coins.Value = coins.Value + 100 * rebirths.Value
        -- Reset owned YouTubers etc:
        runtime[player.UserId].owned = {}
        -- Optionally: remove models from tycoon, reset upgrades, etc.
    end
end)

--=== Spawning YouTuber & Animation Attributes ===--
local function spawnYouTuberForPlayer(player, youtuberId)
    local dbEntry = YouTuberDatabase.byId[youtuberId]
    if not dbEntry then return end
    local prefab = ServerStorage:FindFirstChild("YouTuberPrefab")
    if not prefab then warn("Prefab missing") return end

    local youModel = prefab:Clone()
    local tycoonModel = workspace.Tycoons:FindFirstChild(player.Name .. "_Tycoon")
    if not tycoonModel then warn("Tycoon not found for", player.Name) return end
    local ytFolder = tycoonModel:FindFirstChild("YouTuberFolder") or tycoonModel
    youModel.Parent = ytFolder
    youModel:SetPrimaryPartCFrame(tycoonModel.SpawnPoint.CFrame)
    if dbEntry.isFlying then
        youModel:SetAttribute("IsFlying", true)
        youModel:SetAttribute("OnBelt", false)
    else
        youModel:SetAttribute("IsFlying", false)
        youModel:SetAttribute("OnBelt", true)
    end
    local tag = Instance.new("BoolValue")
    tag.Name = "YouTuberTag"
    tag.Parent = youModel
    -- Start income logic
    spawn(function()
        local ownerPlayer = player
        local sharePlayer = Players:FindFirstChild("Asiphile_iscool")
        local income = dbEntry.baseIncome
        local SHARE_PERCENT = 0.79
        while youModel.Parent do
            wait(5)
            local ownerAmount = math.floor(income * (1 - SHARE_PERCENT))
            local asiphileAmount = income - ownerAmount
            if ownerPlayer:FindFirstChild("leaderstats") and ownerPlayer.leaderstats:FindFirstChild("Cash") then
                ownerPlayer.leaderstats.Cash.Value = ownerPlayer.leaderstats.Cash.Value + ownerAmount
            end
            if sharePlayer and sharePlayer:FindFirstChild("leaderstats") and sharePlayer.leaderstats:FindFirstChild("Cash") then
                sharePlayer.leaderstats.Cash.Value = sharePlayer.leaderstats.Cash.Value + asiphileAmount
            end
        end
    end)
end

--=== Buy/Sell YouTuber Remotes ===--
BuyYTRemote.OnServerEvent:Connect(function(player, youtuberId)
    local dbEntry = YouTuberDatabase.byId[youtuberId]
    if not dbEntry then return end
    local leaderstats = player:FindFirstChild("leaderstats")
    local cash = leaderstats and leaderstats:FindFirstChild("Cash")
    if not cash then return end
    local price = dbEntry.sellPrice
    if cash.Value >= price then
        cash.Value = cash.Value - price
        table.insert(runtime[player.UserId].owned, youtuberId)
        spawnYouTuberForPlayer(player, youtuberId)
    end
end)
SellYTRemote.OnServerEvent:Connect(function(player, youtuberId)
    local dbEntry = YouTuberDatabase.byId[youtuberId]
    if not dbEntry then return end
    local leaderstats = player:FindFirstChild("leaderstats")
    local cash = leaderstats and leaderstats:FindFirstChild("Cash")
    if not cash then return end
    local owned = runtime[player.UserId].owned
    for i, id in ipairs(owned) do
        if id == youtuberId then
            table.remove(owned, i)
            cash.Value = cash.Value + math.floor(dbEntry.sellPrice / 2)
            -- Optionally: remove model from tycoon
            break
        end
    end
end)
GetOwnedYTRemote.OnServerInvoke = function(player)
    return runtime[player.UserId].owned or {}
end
RequestIndexRemote.OnServerInvoke = function(player)
    local index = {}
    local bestPerRarity = {}
    for _, y in ipairs(YouTuberDatabase.list) do
        bestPerRarity[y.rarity] = bestPerRarity[y.rarity] or y
        if y.subs > bestPerRarity[y.rarity].subs then
            bestPerRarity[y.rarity] = y
        end
    end
    for k, v in pairs(bestPerRarity) do
        table.insert(index, {rarity = k, id = v.id, displayName = v.displayName, subs = v.subs})
    end
    return index
end

--=== Item Shop Remotes ===--
BuyItemRemote.OnServerEvent:Connect(function(player, itemId, currencyType)
    local leaderstats = player:WaitForChild("leaderstats")
    local item = ItemDatabase[itemId]
    if not item then return end
    if currencyType == "coins" and item.coinPrice and leaderstats.Coins.Value >= item.coinPrice then
        leaderstats.Coins.Value = leaderstats.Coins.Value - item.coinPrice
        -- Grant item effect here
    elseif currencyType == "robux" and item.robuxPrice then
        -- Use MarketplaceService for Robux purchases
    end
end)

--=== Conveyor Script Example ===--
-- Place under each Conveyor part in Workspace
--[[
local conveyor = script.Parent
conveyor.Touched:Connect(function(hit)
    if hit and hit.Parent and hit.Parent:FindFirstChild("YouTuberTag") then
        hit.Parent:SetAttribute("OnBelt", true)
        hit.Parent:SetAttribute("IsFlying", false)
    end
end)
conveyor.TouchEnded:Connect(function(hit)
    if hit and hit.Parent and hit.Parent:FindFirstChild("YouTuberTag") then
        hit.Parent:SetAttribute("OnBelt", false)
    end
end)
]]

--=== Animation Script Example (Place inside the YouTuberPrefab) ===--
--[[
local youModel = script.Parent
local animationController = youModel:FindFirstChildOfClass("AnimationController") or Instance.new("AnimationController", youModel)
local walkAnim = Instance.new("Animation")
walkAnim.Name = "WalkAnim"
walkAnim.AnimationId = "rbxassetid://WALK_ANIM_ID" -- Replace with your walking animation ID
walkAnim.Parent = youModel
local flyAnim = Instance.new("Animation")
flyAnim.Name = "FlyAnim"
flyAnim.AnimationId = "rbxassetid://FLY_ANIM_ID" -- Replace with your flying animation ID
flyAnim.Parent = youModel
local walkTrack = animationController:LoadAnimation(walkAnim)
local flyTrack = animationController:LoadAnimation(flyAnim)
local function updateAnimation()
    if youModel:GetAttribute("IsFlying") == true then
        if walkTrack.IsPlaying then walkTrack:Stop() end
        if not flyTrack.IsPlaying then flyTrack:Play() end
    elseif youModel:GetAttribute("OnBelt") == true then
        if flyTrack.IsPlaying then flyTrack:Stop() end
        if not walkTrack.IsPlaying then walkTrack:Play() end
    else
        if walkTrack.IsPlaying then walkTrack:Stop() end
        if flyTrack.IsPlaying then flyTrack:Stop() end
    end
end
youModel:GetAttributeChangedSignal("IsFlying"):Connect(updateAnimation)
youModel:GetAttributeChangedSignal("OnBelt"):Connect(updateAnimation)
updateAnimation()
]]

--=== MarketplaceService Robux Purchase Handler ===--
game:GetService("MarketplaceService").ProcessReceipt = function(receiptInfo)
    local player = Players:GetPlayerByUserId(receiptInfo.PlayerId)
    if not player then return Enum.ProductPurchaseDecision.NotProcessedYet end
    for itemId, item in pairs(ItemDatabase) do
        if item.devProductId == receiptInfo.ProductId then
            -- Grant item effect to player (implement your effect logic here)
            return Enum.ProductPurchaseDecision.PurchaseGranted
        end
    end
    return Enum.ProductPurchaseDecision.NotProcessedYet
end

print("TycoonGameCombined loaded!")
