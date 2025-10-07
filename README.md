---

1️⃣ Booth System Script (BoothSystem.lua)

local Players = game:GetService("Players")
local MarketplaceService = game:GetService("MarketplaceService")
local DataStoreService = game:GetService("DataStoreService")

local booth = workspace:WaitForChild("YourBoothModel")
local BoothEarningsStore = DataStoreService:GetDataStore("BoothEarnings")
local boothOwner = nil

local donationTiers = {
    {Name = "SmallDonateButton", ProductId = 0, Offset = Vector3.new(-6,2,0)},
    {Name = "MediumDonateButton", ProductId = 0, Offset = Vector3.new(0,2,0)},
    {Name = "LargeDonateButton", ProductId = 0, Offset = Vector3.new(6,2,0)}
}

for i, tier in ipairs(donationTiers) do
    local existing = booth:FindFirstChild(tier.Name)
    if not existing then
        local part = Instance.new("Part")
        part.Name = tier.Name
        part.Size = Vector3.new(4,1,4)
        part.Anchored = true
        part.CanCollide = false
        part.Position = (booth.PrimaryPart and booth.PrimaryPart.Position or booth:GetModelCFrame().p)+tier.Offset
        part.Parent = booth
    end
    local part = booth:FindFirstChild(tier.Name)
    if part and not part:FindFirstChildOfClass("ClickDetector") then
        local click = Instance.new("ClickDetector")
        click.MaxActivationDistance = 32
        click.Parent = part
        click.MouseClick:Connect(function(player)
            if tier.ProductId and tier.ProductId~=0 then
                MarketplaceService:PromptProductPurchase(player,tier.ProductId)
            else
                warn("ProductId not set:",tier.Name)
            end
        end)
    end
end

MarketplaceService.ProcessReceipt=function(receiptInfo)
    local buyer=Players:GetPlayerByUserId(receiptInfo.PlayerId)
    if not buyer then return Enum.ProductPurchaseDecision.NotProcessedYet end
    if boothOwner then
        local key=tostring(boothOwner.UserId)
        local ok,err=pcall(function()
            BoothEarningsStore:UpdateAsync(key,function(old) return (old or 0)+receiptInfo.CurrencySpent end)
        end)
        if not ok then warn("Failed credit owner:",err)
        else print(buyer.Name.." donated "..receiptInfo.CurrencySpent.." to "..boothOwner.Name)
        end
    else
        print(buyer.Name.." donated but no owner. Granted.")
    end
    return Enum.ProductPurchaseDecision.PurchaseGranted
end

local claimPart=booth:FindFirstChild("ClaimPart")
if not claimPart then
    claimPart=Instance.new("Part")
    claimPart.Name="ClaimPart"
    claimPart.Size=Vector3.new(4,1,4)
    claimPart.Anchored=true
    claimPart.Position=(booth.PrimaryPart and booth.PrimaryPart.Position or booth:GetModelCFrame().p)+Vector3.new(0,2,-6)
    claimPart.Parent=booth
end

if not claimPart:FindFirstChildOfClass("ProximityPrompt") then
    local prompt=Instance.new("ProximityPrompt")
    prompt.ActionText="Claim Booth"
    prompt.ObjectText="Booth"
    prompt.KeyboardKeyCode=Enum.KeyCode.E
    prompt.HoldDuration=0
    prompt.Parent=claimPart
    prompt.Triggered:Connect(function(player)
        if not boothOwner then
            boothOwner=player
            print(player.Name.." is now the booth owner")
        end
    end)
end


---

2️⃣ Earnings Claim Script (EarningsClaim.lua)

local DataStoreService = game:GetService("DataStoreService")
local booth = workspace:WaitForChild("YourBoothModel")
local earningsPart = booth:FindFirstChild("EarningsPart")
local BoothEarningsStore = DataStoreService:GetDataStore("BoothEarnings")

if not earningsPart then
    earningsPart = Instance.new("Part")
    earningsPart.Name = "EarningsPart"
    earningsPart.Size = Vector3.new(4,1,4)
    earningsPart.Anchored = true
    earningsPart.Position = (booth.PrimaryPart and booth.PrimaryPart.Position or booth:GetModelCFrame().p)+Vector3.new(0,2,12)
    earningsPart.Parent = booth
end

if not earningsPart:FindFirstChildOfClass("ProximityPrompt") then
    local prompt = Instance.new("ProximityPrompt")
    prompt.ActionText = "Claim Earnings"
    prompt.ObjectText = "Booth"
    prompt.KeyboardKeyCode = Enum.KeyCode.E
    prompt.HoldDuration = 0
    prompt.Parent = earningsPart

    prompt.Triggered:Connect(function(player)
        local key=tostring(player.UserId)
        local success,pending=pcall(function() return BoothEarningsStore:GetAsync(key) end)
        if not success then warn("Failed read for",player.Name) return end
        pending=pending or 0
        if pending<=0 then return end

        local ok,err=pcall(function()
            BoothEarningsStore:UpdateAsync(key,function() return 0 end)
        end)
        if not ok then warn("Failed clear for",player.Name,err) return end

        local claimedStore=DataStoreService:GetDataStore("BoothClaimed")
        pcall(function() claimedStore:UpdateAsync(key,function(old) return (old or 0)+pending end) end)

        print(player.Name.." claimed "..pending.." Robux (recorded).")
    end)
end


---

3️⃣ README.md

# Roblox Booth & Earnings System

## Setup

1. Place your booth model in `Workspace` and set a `PrimaryPart`.
2. Add these scripts to `ServerScriptService`:
   - `BoothSystem.lua` → handles donations + claiming booth ownership
   - `EarningsClaim.lua` → handles claiming pending earnings
3. Replace `ProductId` in `BoothSystem.lua` with your DevProduct IDs.
4. Enable API Services (DataStore) in Game Settings.
5. Optional: Add `ClaimPart` or `EarningsPart` in your model — scripts auto-create if missing.

## How it works

- Players claim the booth with `E` on `ClaimPart`.
- Donation buttons appear above booth automatically.
- DevProduct purchases credit the current booth owner.
- Booth owner collects earnings via `EarningsPart`.


---

If you want, I can also make a tiny BillboardGui script that automatically shows the current booth owner + pending earnings above the booth — perfect for GitHub demo.

Do you want me to add that?

