-- ===== CONFIG =====
pcall(function() if setfpscap then setfpscap(20) end end)
_G.main = {"TurboPanda97962Y", "LuckyViper95203U", "ShadowComet48801U", "wasd"}
_G.afk  = {"wasd", "asd"}
_G.pointcap = 1000
-- ==================

--- V.8.6.16 Tight Stack Farm + Fast Position Lock
repeat task.wait(0.1) until game:IsLoaded()

local TeleportService = game:GetService("TeleportService")
local Players = game:GetService("Players")
local EarlyLP = Players.LocalPlayer

local MAIN_PLACE_ID = 1458767429
local AFK_WORLD_PLACE_ID = 5411459567

if game.PlaceId == AFK_WORLD_PLACE_ID then
	pcall(function()
		TeleportService:Teleport(MAIN_PLACE_ID, EarlyLP)
	end)
	return
end

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VUser   = game:GetService("VirtualUser")
local Http    = game:GetService("HttpService")
local UIS     = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")
local SoundService = game:GetService("SoundService")
local LP      = Players.LocalPlayer

-- ===== Full Client Mute =====
pcall(function()
	UserSettings().GameSettings.MasterVolume=0
end)

local function muteSound(obj)
	if not obj or not obj:IsA("Sound") then return end
	pcall(function()
		obj.Volume=0
		obj:Stop()
	end)
end

task.spawn(function()
	for _,obj in ipairs(game:GetDescendants()) do
		if obj:IsA("Sound") then muteSound(obj) end
	end
end)

game.DescendantAdded:Connect(function(obj)
	if obj:IsA("Sound") then
		task.defer(muteSound,obj)
	end
end)

local resetSequence = 0
local actionNoticeLabel = nil
local actionNoticeToken = 0
local function notifyAction(title, message, duration)
	local msg=tostring(message or "")
	pcall(function()
		StarterGui:SetCore("SendNotification",{
			Title = title or "WWHub",
			Text = msg,
			Duration = duration or 5
		})
	end)
	warn(("[WWHub] %s | %s"):format(tostring(title or "WWHub"),msg))
	if actionNoticeLabel then
		actionNoticeToken+=1
		local token=actionNoticeToken
		actionNoticeLabel.Text=tostring(title or "WWHub").." • "..msg
		actionNoticeLabel.Visible=true
		task.delay(duration or 5,function()
			if actionNoticeLabel and token==actionNoticeToken then
				actionNoticeLabel.Visible=false
			end
		end)
	end
end

local myName = LP.Name
local IS_MAIN = false
local IS_AFK = false

for _, n in ipairs(_G.main) do
	if n == myName then IS_MAIN = true break end
end

for _, n in ipairs(_G.afk) do
	if n == myName and not IS_MAIN then IS_AFK = true break end
end

local IS_FARMER = IS_MAIN or IS_AFK
local FARM_ROLE = IS_AFK and "AFK" or (IS_MAIN and "MAIN" or "UNLISTED")
local manualOverride = false
warn("[WWHub] Role: " .. FARM_ROLE)

-- Random server hop disabled

-- ===== Vars =====
local FARM_BASE = Vector3.new(200, 2000, 200)
local FARM_PAIR_SPACING = 0.15
local FARM_ALT_OFFSET = 0.15

local function getRoleIndex(list)
	for i, n in ipairs(list) do
		if n == myName then return i end
	end
	return 1
end

local farmIndex = IS_AFK and getRoleIndex(_G.afk) or getRoleIndex(_G.main)
local pairX = (farmIndex - 1) * FARM_PAIR_SPACING
local farmPos = FARM_BASE + Vector3.new(pairX, 0, IS_AFK and FARM_ALT_OFFSET or 0)
local altCFrame = CFrame.new(farmPos)
local pauseCFrame = CFrame.new(156, 1, -43)
local baseName    = "WWHub_BasePlate"
local tpDist      = 0.55
local safeLimit   = 8


local loopMain      = false
local starting      = false
local selectingTeam = false
local pointsCapped  = false
local roundPaused   = false
local roundPauseReason = nil
local roundResetting   = false
local handledChar   = nil
local timerTpDone   = false
local endRoundResetDone = false
local gui           = nil
_G.pointcap = math.max(100, tonumber(_G.pointcap) or 1000)

-- ===== Gold Progress Tracking =====
local scriptStartTime = os.time()
local startGold       = nil   -- captured once real Gold value is available

local WebhookURL = "https://discord.com/api/webhooks/1453628734090514533/ddACObJX5Iuv966TcspBAEmkd5Er2ZfiVCMdoHzyONWLJ1CoqlDaAn3vg9D1GiZkvPoR"
local _request
if not pcall(function() _request = request or http_request or http.request end) then
	_request = function() end
end

local avatarUrl = ""
task.spawn(function()
	pcall(function()
		local url = ("https://thumbnails.roblox.com/v1/users/avatar-headshot?userIds=%d&size=420x420&format=Png&isCircular=false"):format(LP.UserId)
		local r = Http:JSONDecode(game:HttpGet(url))
		if r and r.data and r.data[1] then avatarUrl = r.data[1].imageUrl or "" end
	end)
end)

-- capture starting Gold as soon as it's available (retry up to ~50s)
task.spawn(function()
	for _ = 1, 50 do
		local ok, v = pcall(function()
			return LP:WaitForChild("ReplicatedStats"):WaitForChild("Gold").Value
		end)
		if ok and v then startGold = v break end
		task.wait(1)
	end
	if startGold == nil then startGold = 0 end
end)

-- ===== Helpers =====
local function getChar() return LP.Character end
local function getHRP()  local c = getChar() return c and c:FindFirstChild("HumanoidRootPart") end
local function getHum()  local c = getChar() return c and c:FindFirstChildOfClass("Humanoid") end
local function getInput()
	local bp = LP:FindFirstChild("Backpack") local c = getChar()
	return (bp and bp:FindFirstChild("Input")) or (c and c:FindFirstChild("Input")) or LP:FindFirstChild("Input")
end
local function fireInput(action, data)
	local inp = getInput() if not inp then return end
	pcall(function() if data then inp:FireServer(action, data) else inp:FireServer(action) end end)
end
local function makeBase()
	local b = workspace:FindFirstChild(baseName)
	if not b then
		b = Instance.new("Part") b.Name = baseName b.Anchored = true
		b.Color = Color3.fromRGB(35,35,35) b.Material = Enum.Material.SmoothPlastic b.Parent = workspace
	end
	b.Size = Vector3.new(500, 0.4, 500)
	b.CFrame = CFrame.new(altCFrame.Position - Vector3.new(0, 3, 0))
end
local function isNearCF(cf, limit)
	local hrp = getHRP() if not hrp or not cf then return false end
	return (hrp.Position - cf.Position).Magnitude <= (limit or tpDist)
end
local function tpToCF(cf)
	if not cf then return false end makeBase()
	local hrp = getHRP() if not hrp then return false end
	pcall(function()
		hrp.AssemblyLinearVelocity  = Vector3.new(0,0,0)
		hrp.AssemblyAngularVelocity = Vector3.new(0,0,0)
		hrp.CFrame = cf
	end)
	return true
end
local function getMainCF()
	local p = altCFrame.Position
	return CFrame.lookAt(p, p + Vector3.new(0,0,1))
end
local function tpToSafeZone()
	local cf = pauseCFrame
	task.spawn(function()
		for _ = 1, 20 do
			if not gui or not gui.Parent then break end
			local hrp = getHRP()
			if hrp then
				pcall(function()
					hrp.AssemblyLinearVelocity  = Vector3.new(0,0,0)
					hrp.AssemblyAngularVelocity = Vector3.new(0,0,0)
					hrp.CFrame = cf
				end)
				task.wait(0.25) hrp = getHRP()
				if hrp and (hrp.Position - cf.Position).Magnitude <= safeLimit then break end
			end
			task.wait(0.25)
		end
	end)
end

-- ===== Bakugou Q Remote =====
local function fireBakugouQ()
	if not IS_MAIN then return end
	local inp = getInput()
	local hrp = getHRP()
	if not inp or not hrp then return end
	pcall(function()
		inp:FireServer("dodge", {
			TagName = "ExplodDodge",
			explod = Enum.KeyCode.W,
			ServerSwoosh = false,
			pos = hrp.CFrame
		})
	end)
end

local function forceFieldOff()
	fireInput("ForceFieldOff")
end

-- Anti-AFK
pcall(function()
	LP.Idled:Connect(function()
		pcall(function()
			VUser:CaptureController()
			VUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
			task.wait(0.1)
			VUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
		end)
	end)
end)

-- ===== Farm Character Verify =====
local characterFixing = false

local function characterKey(name)
	return tostring(name or ""):lower():gsub("[^%w]","")
end

local function readEquippedCharacter()
	local char = LP.Character
	for _,container in ipairs({char or false,LP:FindFirstChild("ReplicatedStats") or false,LP}) do
		if container then
			for _,field in ipairs({"CharacterName","CurrentCharacter","EquippedCharacter"}) do
				local attr = container:GetAttribute(field)
				if type(attr) == "string" and attr ~= "" then return attr end
				local value = container:FindFirstChild(field)
				if value and value:IsA("StringValue") and value.Value ~= "" then return value.Value end
			end
		end
	end
	return nil
end

local function getChooseRemote()
	local bp = LP:FindFirstChild("Backpack")
	local c = LP.Character
	local st = (bp and bp:FindFirstChild("ServerTraits")) or (c and c:FindFirstChild("ServerTraits")) or LP:FindFirstChild("ServerTraits")
	return st and st:FindFirstChild("Choose")
end

local function fireSelectRemote(charName)
	local inp = getInput()
	if inp then
		local ok = pcall(function()
			inp:FireServer("CharacterButton",charName)
			task.wait(0.15)
			local fresh = getInput()
			if fresh then fresh:FireServer("ClickPlay") end
		end)
		if ok then return true end
	end

	local choose = getChooseRemote()
	if choose then
		return pcall(function()
			choose:FireServer(charName)
			task.wait(0.15)
			local fresh = getChooseRemote()
			if fresh then fresh:FireServer("PLAY") end
		end)
	end
	return false
end

local function farmCharacterName()
	return IS_MAIN and "Bakugou" or "Ichigo"
end

local function isKnownWrongFarmCharacter()
	local equipped = readEquippedCharacter()
	if not equipped then return false,nil end
	local target = farmCharacterName()
	return characterKey(equipped) ~= characterKey(target),equipped
end

local function ensureFarmCharacter(reason)
	if characterFixing then return false end
	characterFixing = true
	local target = farmCharacterName()
	local equipped = readEquippedCharacter()

	if equipped and characterKey(equipped) == characterKey(target) then
		characterFixing = false
		return true
	end

	if equipped then
		notifyAction("Character Fix","Wrong character: "..tostring(equipped).." -> "..target.." | "..tostring(reason or ""),6)
	end

	local deadline = os.clock()+8
	local sent = false
	while os.clock() < deadline do
		sent = fireSelectRemote(target) or sent

		pcall(function()
			local pg = LP:FindFirstChild("PlayerGui")
			local respawning = pg and pg:FindFirstChild("Respawning")
			local done = respawning and respawning:FindFirstChild("Done")
			if done then done:FireServer() end
		end)

		task.wait(0.35)
		equipped = readEquippedCharacter()
		if equipped and characterKey(equipped) == characterKey(target) then
			notifyAction("Character Ready",target.." confirmed",4)
			characterFixing = false
			return true
		end
	end

	-- Some ABA versions do not expose equipped character name.
	-- If remote was accepted, keep target selection instead of reverting to another character.
	if sent and not readEquippedCharacter() then
		for _=1,3 do
			fireSelectRemote(target)
			task.wait(0.2)
		end
		characterFixing = false
		return true
	end

	notifyAction("Character Error","Could not confirm "..target.." | observed "..tostring(readEquippedCharacter()),6)
	characterFixing = false
	return false
end

-- ===== Bakugou Auto Buy =====
local bakugouBuyTried = false

local function buyBakugou()
	if not IS_MAIN or bakugouBuyTried then return end
	local ok = pcall(function()
		local backpack = LP:WaitForChild("Backpack",10)
		local serverTraits = backpack and backpack:WaitForChild("ServerTraits",10)
		local choose = serverTraits and serverTraits:WaitForChild("Choose",10)
		if not choose then error("Choose remote missing") end
		choose:FireServer("Bakugou")
		task.wait(0.15)
		choose:FireServer("PLAY")
		task.wait(0.5)
	end)
	if ok then bakugouBuyTried = true end
end

-- ===== Character =====
local function fireRespawnDone()
	local r = LP:WaitForChild("PlayerGui"):FindFirstChild("Respawning")
	if r and r:FindFirstChild("Done") then pcall(function() r.Done:FireServer() end) end
end
local function afterCharLoaded(char)
	if not char or handledChar == char then return end
	handledChar = char
	char:WaitForChild("HumanoidRootPart", 10) char:WaitForChild("Humanoid", 10)
	task.wait(1) fireRespawnDone() task.wait(0.2) forceFieldOff()
end
local function resetChar(reason)
	resetSequence += 1
	notifyAction("WWHub Reset",("#%d | %s"):format(resetSequence,reason or "Unknown reason"),6)
	local c = getChar()
	if c then
		local h = c:FindFirstChildOfClass("Humanoid")
		if h and h.Health > 0 then h.Health = 0 end
	end
	pcall(function() game:GetService("ReplicatedStorage"):WaitForChild("Loaded"):FireServer() end)
	task.wait(1)
	local nc = getChar()
	if nc then afterCharLoaded(nc) end
end

-- ===== Stats =====
local function getPoints() local ok,v = pcall(function() return LP.leaderstats.Points.Value end) return ok and v or 0 end
local function getTimerValue()
	local ok,v = pcall(function()
		local hud = LP.PlayerGui:FindFirstChild("HUD") if not hud then return 0 end
		local t = hud:FindFirstChild("Timer") if not t then return 0 end
		return (t:IsA("TextLabel") or t:IsA("TextBox")) and (tonumber(t.Text) or 0) or (tonumber(t.Value) or 0)
	end)
	return ok and v or 0
end
local lastKnownLevel = nil
local function getLevel()
	local ok,v = pcall(function()
		-- Try replicated/non-GUI values first.
		local attr = LP:GetAttribute("Level")
		if tonumber(attr) then return tonumber(attr) end

		local replicatedStats = LP:FindFirstChild("ReplicatedStats")
		local leaderstats = LP:FindFirstChild("leaderstats")
		local char = LP.Character
		local charStats = char and char:FindFirstChild("Stats")

		for _,root in ipairs({replicatedStats or false,leaderstats or false,LP,char or false,charStats or false}) do
			if root then
				local level = root:FindFirstChild("Level")
				if level and (level:IsA("IntValue") or level:IsA("NumberValue")) then
					return tonumber(level.Value)
				end
			end
		end

		for _,obj in ipairs(LP:GetDescendants()) do
			if obj.Name == "Level" and (obj:IsA("IntValue") or obj:IsA("NumberValue")) then
				return tonumber(obj.Value)
			end
		end

		-- ABA currently exposes the actual displayed level through HUD.LocalHUDScript.
		local pg = LP:FindFirstChild("PlayerGui")
		local hud = pg and pg:FindFirstChild("HUD")
		local rb = hud and hud:FindFirstChild("RightBotCorner")
		local line2 = rb and rb:FindFirstChild("Line2")
		local lvl = line2 and line2:FindFirstChild("Lvl")
		if lvl and (lvl:IsA("TextLabel") or lvl:IsA("TextBox")) then
			local n = tonumber(tostring(lvl.Text):match("[Ll]vl%.?%s*(%d+)") or tostring(lvl.Text):match("(%d+)"))
			if n then return n end
		end

		return lastKnownLevel
	end)
	if ok and tonumber(v) then lastKnownLevel = tonumber(v) end
	return lastKnownLevel
end

local PRESTIGE_POOL = {
	["Achilles"]=true,["Chiaotzu"]=true,["Deidara"]=true,["Fubuki"]=true,
	["Hercule Satan"]=true,["Mr President"]=true,["Ritsu"]=true,
	["Satsuki Kiryuin"]=true,["Shadow DIO"]=true,["Shisui"]=true,
	["Super Dummy"]=true,["Tobi"]=true,["Tokita Ohma"]=true,
	["Zamasu [Fused]"]=true,["Zenitsu"]=true
}

local function getPrestigeCount()
	local ok,count = pcall(function()
		local rs = LP:FindFirstChild("ReplicatedStats")
		local unlocked = rs and rs:FindFirstChild("Unlocked")
		if not unlocked or type(unlocked.Value) ~= "string" then return 0 end
		local data = Http:JSONDecode(unlocked.Value)
		if type(data) ~= "table" then return 0 end
		local n = 0
		for name in pairs(PRESTIGE_POOL) do
			if data[name] ~= nil then n += 1 end
		end
		return n
	end)
	return ok and tonumber(count) or 0
end

-- ===== Point Cap Visibility Safety =====
local function isFarmAccount(name)
	for _, n in ipairs(_G.main) do
		if n == name then return true end
	end
	for _, n in ipairs(_G.afk) do
		if n == name then return true end
	end
	return false
end

local function hasOutsider()
	for _, plr in ipairs(Players:GetPlayers()) do
		if not isFarmAccount(plr.Name) then
			return true
		end
	end
	return false
end

-- ===== Gold / Single Point Cap =====
local function getGold()
	local ok,v=pcall(function()
		return LP:WaitForChild("ReplicatedStats"):WaitForChild("Gold").Value
	end)
	return ok and v or 0
end

local function getEffectiveCap()
	return math.max(100,tonumber(_G.pointcap) or 1000)
end

local function adjustEffectiveCap(delta)
	_G.pointcap=math.max(100,getEffectiveCap()+delta)
	if getPoints()<getEffectiveCap() then
		pointsCapped=false
	end
end

-- คืนค่าข้อความ progress: "+1,000 Gold ผ่านมาแล้ว 01 ชม 03 นาที"
local function getGoldProgressText()
	local current = getGold()
	local base = startGold or current
	local gained = current - base
	local elapsed = os.time() - scriptStartTime
	local hh = math.floor(elapsed / 3600)
	local mm = math.floor((elapsed % 3600) / 60)
	local sign = gained >= 0 and "+" or ""
	return string.format("%s%d Gold ผ่านมาแล้ว %02d ชม %02d นาที", sign, gained, hh, mm)
end

-- ===== Webhook =====
local function sendWebhook(label)
	task.spawn(function()
		pcall(function()
			local money, lvl, pts = "N/A","N/A","N/A"
			pcall(function()
				money = tostring(LP:WaitForChild("ReplicatedStats"):WaitForChild("Gold").Value)
				local realLevel = getLevel()
				if realLevel ~= nil then lvl = tostring(realLevel) else lvl = "?" end
				pts = tostring(LP.leaderstats.Points.Value)
			end)
			-- ถ้า level >= 100 ให้ติดตรา max level ต่อท้าย
			pcall(function()
				local lvlNum = tonumber(lvl:match("%d+"))
				if lvlNum and lvlNum >= 100 then
					lvl = lvl .. " 🟢"
				end
			end)
			local progressText = "N/A"
			pcall(function() progressText = getGoldProgressText() end)
			_request({
				Url = WebhookURL, Method = "POST",
				Headers = {["Content-Type"] = "application/json"},
				Body = Http:JSONEncode({
					username = "WW Hub", avatar_url = avatarUrl,
					embeds = {{
						title = "⚡ WWHub — "..label, color = 0x46C864,
						thumbnail = {url = avatarUrl},
						fields = {
							{name="👤 Player", value=LP.DisplayName.." (@"..LP.Name..")", inline=false},
							{name="💰 Money",  value=money, inline=true},
							{name="⭐ Level",  value=lvl,   inline=true},
							{name="🎯 Points", value=pts,   inline=true},
							{name="📈 Progress", value=progressText, inline=false},
						},
						footer    = {text = "WWHub • "..os.date("%d/%m/%Y %H:%M:%S")},
						timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
					}}
				})
			})
		end)
	end)
end

-- ===== Blocked Mode =====
local function isVisibleGuiObject(obj)
	if not obj or not obj:IsA("GuiObject") then return false end
	if not obj.Visible then return false end
	local p = obj.Parent
	while p and p ~= LP.PlayerGui do
		if p:IsA("GuiObject") and not p.Visible then return false end
		p = p.Parent
	end
	return true
end

local function getVisibleTeamModeHint()
	local pg=LP:FindFirstChild("PlayerGui")
	if not pg then return nil end
	local function matchMode(raw)
		local tx=tostring(raw or ""):lower()
		if tx:find("team battle",1,true) then return "Team Battle" end
		if tx:find("kills team",1,true) then return "Kills Team" end
		if tx:find("3 teams",1,true) or tx:find("three teams",1,true) then return "3 Teams" end
		return nil
	end
	for _,obj in ipairs(pg:GetDescendants()) do
		if (obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton")) and isVisibleGuiObject(obj) then
			local mode=matchMode(obj.Text)
			if mode then return mode end
		elseif obj:IsA("StringValue") then
			local nm=tostring(obj.Name or ""):lower()
			if nm:find("mode",1,true) or nm:find("game",1,true) then
				local mode=matchMode(obj.Value)
				if mode then return mode end
			end
		end
	end
	return nil
end

local function getBlockedMode()
	local pg = LP:FindFirstChild("PlayerGui")
	if not pg then return nil end
	local lb = pg:FindFirstChild("CustomLeaderboard")
	if not lb then return nil end
	local m = lb:FindFirstChild("Main") or lb

	-- Strong Juggernaut detection.
	for _,obj in ipairs(m:GetDescendants()) do
		local nm = tostring(obj.Name or ""):lower()
		if nm == "juggernaut" or nm:find("juggernaut",1,true) then
			return "Juggernaut"
		end
		if (obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton")) and isVisibleGuiObject(obj) then
			local tx = tostring(obj.Text or ""):lower()
			if tx:find("juggernaut",1,true) then return "Juggernaut" end
		end
	end

	-- In any detected team round, generic Lives must NEVER trigger the blocked-mode reset.
	local teamMode=getVisibleTeamModeHint()
	local teamPadPresent=workspace:FindFirstChild("Red Team") or workspace:FindFirstChild("Blue Team")
		or workspace:FindFirstChild("Green Team") or workspace:FindFirstChild("Yellow Team")
	if LP.Team~=nil or teamMode or teamPadPresent then
		return nil
	end

	-- Lives must contain an actual numeric lives value, not only an object named "Lives".
	for _,obj in ipairs(m:GetDescendants()) do
		if (obj:IsA("IntValue") or obj:IsA("NumberValue")) and tostring(obj.Name or ""):lower() == "lives" then
			local n = tonumber(obj.Value)
			if n and n > 0 then return "Lives" end
		elseif (obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton")) and isVisibleGuiObject(obj) then
			local tx = tostring(obj.Text or "")
			local n = tx:match("[Ll][Ii][Vv][Ee][Ss]%s*:%s*(%d+)") or tx:match("[Ll][Ii][Vv][Ee][Ss]%s+(%d+)")
			if tonumber(n) and tonumber(n) > 0 then return "Lives" end
		end
	end
	return nil
end

local function getLivesValue()
	local pg = LP:FindFirstChild("PlayerGui")
	if not pg then return nil end

	for _, obj in ipairs(pg:GetDescendants()) do
		if obj:IsA("IntValue") or obj:IsA("NumberValue") then
			if (obj.Name or ""):lower() == "lives" then
				return tonumber(obj.Value)
			end
		elseif obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton") then
			local tx = tostring(obj.Text or "")
			local n = tx:match("[Ll][Ii][Vv][Ee][Ss]%s*:%s*(%d+)")
				or tx:match("[Ll][Ii][Vv][Ee][Ss]%s+(%d+)")
			if n then return tonumber(n) end
		end
	end
	return nil
end

local function waitForRespawn(oldChar, timeout)
	local started = os.clock()
	timeout = timeout or 10
	while gui and gui.Parent and loopMain and (os.clock() - started) < timeout do
		local lives = getLivesValue()
		if lives ~= nil and lives <= 0 then
			return false
		end

		local c = getChar()
		local h = c and c:FindFirstChildOfClass("Humanoid")
		local hrp = c and c:FindFirstChild("HumanoidRootPart")
		if c and c ~= oldChar and h and hrp and h.Health > 0 then
			afterCharLoaded(c)
			return true
		end
		task.wait(0.1)
	end
	return false
end

local function drainLives()
	local initialLives = getLivesValue()
	local resetCount = 0
	local missingSince = nil

	while gui and gui.Parent and loopMain and roundPaused do
		local lives = getLivesValue()
		local bm = getBlockedMode()

		if lives ~= nil then
			missingSince = nil
			if lives <= 0 then break end
		elseif not bm then
			missingSince = missingSince or os.clock()

			-- ระหว่างตาย/เกิดใหม่ UI จะหายชั่วคราว ห้ามหยุด reset ทันที
			if initialLives then
				if resetCount >= initialLives and (os.clock() - missingSince) >= 2 then
					break
				end
			elseif (os.clock() - missingSince) >= 5 then
				break
			end
		else
			missingSince = nil
		end

		local oldChar = getChar()
		local hum = oldChar and oldChar:FindFirstChildOfClass("Humanoid")
		if hum and hum.Health > 0 then
			resetCount += 1
			resetSequence += 1
			notifyAction("WWHub Reset",("#%d | Blocked %s | Lives=%s | Drain #%d"):format(
				resetSequence,tostring(bm or roundPauseReason or "Unknown"),tostring(lives or "?"),resetCount
			),6)
			pcall(function() hum.Health = 0 end)

			pcall(function()
				ReplicatedStorage:WaitForChild("Loaded"):FireServer()
			end)

			waitForRespawn(oldChar, 10)
		else
			task.wait(0.25)
		end

		task.wait(0.5)
	end
end

local function pauseFarm(reason)
	if roundPaused then return end
	roundPaused = true roundPauseReason = reason or "Blocked"
	pointsCapped = false
	notifyAction("Farm Paused","Blocked mode confirmed: "..roundPauseReason.." -> reset/drain",6)
	sendWebhook("Farm Paused — "..roundPauseReason)
	if roundResetting then return end roundResetting = true
	task.spawn(function()
		-- Juggernaut/Lives: reset repeatedly until Lives reaches 0
		drainLives()
		tpToSafeZone()

		local clear = 0
		while gui and gui.Parent and loopMain and roundPaused do
			local bm = getBlockedMode()
			if bm then
				clear = 0
				tpToSafeZone()
				task.wait(1)
			else
				clear += 1
				if clear >= 3 then break end
				task.wait(1)
			end
		end

		local last = roundPauseReason
		roundPaused=false roundPauseReason=nil roundResetting=false
		if gui and gui.Parent and loopMain then
			notifyAction("Farm Resumed","Blocked mode cleared: "..tostring(last),5)
			sendWebhook("Farm Resumed — "..last)
		end
	end)
end

-- Timer watchdog (now uses dynamic Gold-based cap)
task.spawn(function()
	while true do
		task.wait(0.3)
		if loopMain then
			local pts = getPoints() local timer = getTimerValue()
			local capNow = getEffectiveCap()

			-- No outsiders = no point cap. Re-enable automatically if an outsider joins.
			if not hasOutsider() then
				pointsCapped = false
			elseif pts >= capNow then
				pointsCapped = true
			else
				pointsCapped = false
			end
			if timer > 0 and timer <= 2 and not timerTpDone and not roundPaused then
				timerTpDone = true
				notifyAction("Farm Action","End round timer <= 2 -> TP Safe Zone",4)
				tpToSafeZone()
				if not endRoundResetDone then
					endRoundResetDone = true
					task.spawn(function()
						task.wait(0.25)
						if loopMain then
							resetChar("End round: timer <= 2")
							task.wait(0.75)
							ensureFarmCharacter("After end-round reset")
						end
					end)
				end
			end
			if timerTpDone then
				-- Re-arm only after the next round timer is clearly running again.
				-- Team pads may remain/spawn during respawn/intermission; using them here caused reset loops.
				if timer >= 10 then
					timerTpDone=false
					endRoundResetDone=false
				end
			end
		else
			pointsCapped = false timerTpDone = false
		end
	end
end)

-- startFarm
local function startFarm()
	if not IS_FARMER then
		notifyAction("Not Listed","Press START to run this unlisted account manually as MAIN",8)
		return
	end
	if starting then return end starting=true loopMain=false makeBase()
	if IS_MAIN then
		buyBakugou()
		task.wait(0.5)
	end
	local farmCharacter = farmCharacterName()
	fireSelectRemote(farmCharacter)
	task.wait(2.5)
	resetChar("Startup character sync")
	task.wait(1.5)
	ensureFarmCharacter("Startup verify")
	task.wait(1)
	roundPaused=false roundPauseReason=nil roundResetting=false timerTpDone=false endRoundResetDone=false pointsCapped=false
	loopMain = true
	sendWebhook(FARM_ROLE.." Farm Started")
	starting = false
end

-- Team selection — MAIN and AFK use opposing teams
local allTeamPads = {"Red Team", "Blue Team", "Green Team", "Yellow Team"}

local function hasAnyTeamPad()
	for _, name in ipairs(allTeamPads) do
		if workspace:FindFirstChild(name) then return true end
	end
	return false
end

local function getMyTeamPad()
	local availablePads = {}
	for _, name in ipairs(allTeamPads) do
		if workspace:FindFirstChild(name) then
			table.insert(availablePads, name)
		end
	end
	if #availablePads == 0 then return nil end

	local padIndex = 1
	if IS_MAIN then
		local mainIndex = getRoleIndex(_G.main)
		padIndex = ((mainIndex - 1) % #availablePads) + 1
	elseif IS_AFK then
		local afkIndex = getRoleIndex(_G.afk)
		if #availablePads >= 2 then
			-- Put each ALT on the next color from its paired MAIN.
			padIndex = (afkIndex % #availablePads) + 1
		else
			padIndex = 1
		end
	end

	return workspace:FindFirstChild(availablePads[padIndex])
end

local function selectTeam()
	if selectingTeam then return end
	selectingTeam = true

	while gui and gui.Parent and loopMain and not roundPaused and hasAnyTeamPad() do
		local currentPad = getMyTeamPad()
		local hrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")

		if currentPad and hrp then
			local pp = currentPad:IsA("BasePart") and currentPad or currentPad:FindFirstChildWhichIsA("BasePart")
			if pp then
				pcall(function()
					hrp.AssemblyLinearVelocity = Vector3.new(0,0,0)
					hrp.AssemblyAngularVelocity = Vector3.new(0,0,0)
					hrp.CFrame = pp.CFrame + Vector3.new(0,3,0)
				end)
			end
		end

		task.wait(0.1)
	end

	-- Do not release farm TP until team pads are really gone.
	if gui and gui.Parent and loopMain and not roundPaused then
		task.wait(0.35)
	end
	selectingTeam = false
end

task.spawn(function()
	-- เรียง FFA ก่อน team เพื่อลด chance เจอ team mode
	local modes = {
		"Free For All",  -- 1st priority
		"Kills FFA",     -- 2nd
		"Kills Team",    -- 3rd
		"3 Teams",       -- 4th
		"Team Battle",   -- 5th
	}
	while gui.Parent do
		if loopMain and not roundPaused then
			for _, m in ipairs(modes) do fireInput("mode", m) task.wait(0.3) end
			task.wait(2)
		else task.wait(3) end
	end
end)

-- NoClip
local noClipOn = false
task.spawn(function()
	while gui and gui.Parent do
		task.wait(0.1)
		if loopMain and not roundPaused and not timerTpDone then
			if not noClipOn then noClipOn = true end
			local c = getChar()
			if c then
				for _, p in ipairs(c:GetDescendants()) do
					if p:IsA("BasePart") then p.CanCollide = false end
				end
				local hrp = getHRP()
				if hrp and hrp.Position.Y < -10 then
					pcall(function()
						hrp.CFrame = CFrame.new(hrp.Position.X, 5, hrp.Position.Z)
						hrp.AssemblyLinearVelocity = Vector3.new(0,0,0)
					end)
				end
			end
		else
			if noClipOn then
				noClipOn = false local c = getChar()
				if c then for _, p in ipairs(c:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end end
			end
		end
	end
end)


-- ===== GUI =====
gui=Instance.new("ScreenGui")
gui.Name="WWHub_GUI_v8_6_16"
gui.ResetOnSpawn=false
gui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling
gui.DisplayOrder=1000000
gui.Parent=game.CoreGui

local toggleBtn=Instance.new("TextButton")
toggleBtn.Size=UDim2.new(0,42,0,42)
toggleBtn.Position=UDim2.new(0,10,0.5,-21)
toggleBtn.BackgroundColor3=Color3.fromRGB(18,18,28)
toggleBtn.Text="⚡"
toggleBtn.TextSize=20
toggleBtn.TextColor3=Color3.fromRGB(150,70,255)
toggleBtn.Font=Enum.Font.GothamBold
toggleBtn.ZIndex=10
toggleBtn.Parent=gui
Instance.new("UICorner",toggleBtn).CornerRadius=UDim.new(0,10)
local tst=Instance.new("UIStroke",toggleBtn)
tst.Color=Color3.fromRGB(110,40,200)
tst.Thickness=2

local panel=Instance.new("Frame")
panel.Size=UDim2.new(0,310,0,456)
panel.Position=UDim2.new(0.5,-155,0.5,-228)
panel.BackgroundColor3=Color3.fromRGB(14,14,22)
panel.BorderSizePixel=0
panel.Active=true
panel.Parent=gui
Instance.new("UICorner",panel).CornerRadius=UDim.new(0,14)
local pst=Instance.new("UIStroke",panel)
pst.Color=Color3.fromRGB(100,35,190)
pst.Thickness=2

local dragging,dragStart,dragPos
panel.InputBegan:Connect(function(i)
	if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
		dragging=true dragStart=i.Position dragPos=panel.Position
	end
end)
panel.InputEnded:Connect(function(i)
	if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=false end
end)
if UIS then
	UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
			local d=i.Position-dragStart
			panel.Position=UDim2.new(dragPos.X.Scale,dragPos.X.Offset+d.X,dragPos.Y.Scale,dragPos.Y.Offset+d.Y)
		end
	end)
end

local header=Instance.new("Frame")
header.Size=UDim2.new(1,0,0,48)
header.BackgroundColor3=Color3.fromRGB(20,20,32)
header.BorderSizePixel=0
header.Parent=panel
Instance.new("UICorner",header).CornerRadius=UDim.new(0,14)

local titleLbl=Instance.new("TextLabel")
titleLbl.Size=UDim2.new(1,-50,1,0)
titleLbl.Position=UDim2.new(0,12,0,0)
titleLbl.BackgroundTransparency=1
titleLbl.Text="⚡ WW Hub v8.6.16"
titleLbl.TextColor3=Color3.fromRGB(155,80,255)
titleLbl.TextSize=18
titleLbl.Font=Enum.Font.GothamBold
titleLbl.TextXAlignment=Enum.TextXAlignment.Left
titleLbl.Parent=header

local closeBtn=Instance.new("TextButton")
closeBtn.Size=UDim2.new(0,32,0,32)
closeBtn.Position=UDim2.new(1,-40,0,8)
closeBtn.BackgroundColor3=Color3.fromRGB(190,35,55)
closeBtn.Text="✕"
closeBtn.TextColor3=Color3.fromRGB(255,255,255)
closeBtn.TextSize=15
closeBtn.Font=Enum.Font.GothamBold
closeBtn.Parent=header
Instance.new("UICorner",closeBtn).CornerRadius=UDim.new(0,8)

local function makeStatCard(x,title)
	local card=Instance.new("Frame")
	card.Size=UDim2.new(0,90,0,62)
	card.Position=UDim2.new(0,x,0,58)
	card.BackgroundColor3=Color3.fromRGB(22,24,38)
	card.BorderSizePixel=0
	card.Parent=panel
	Instance.new("UICorner",card).CornerRadius=UDim.new(0,10)
	local stroke=Instance.new("UIStroke",card)
	stroke.Color=Color3.fromRGB(48,52,78)
	stroke.Thickness=1

	local name=Instance.new("TextLabel")
	name.Size=UDim2.new(1,-8,0,18)
	name.Position=UDim2.new(0,4,0,6)
	name.BackgroundTransparency=1
	name.Text=title
	name.TextColor3=Color3.fromRGB(145,155,185)
	name.TextSize=10
	name.Font=Enum.Font.GothamBold
	name.Parent=card

	local value=Instance.new("TextLabel")
	value.Size=UDim2.new(1,-8,0,28)
	value.Position=UDim2.new(0,4,0,26)
	value.BackgroundTransparency=1
	value.Text="-"
	value.TextColor3=Color3.fromRGB(240,244,255)
	value.TextScaled=true
	value.Font=Enum.Font.GothamBold
	value.Parent=card
	return value
end

local lvValue=makeStatCard(10,"LEVEL")
local goldValue=makeStatCard(110,"GOLD")
local prestigeValue=makeStatCard(210,"PRESTIGE")

local statusLbl=Instance.new("TextLabel")
statusLbl.Size=UDim2.new(1,-20,0,20)
statusLbl.Position=UDim2.new(0,10,0,126)
statusLbl.BackgroundTransparency=1
statusLbl.Text="📊 Idle"
statusLbl.TextColor3=Color3.fromRGB(140,140,165)
statusLbl.TextSize=11
statusLbl.Font=Enum.Font.Gotham
statusLbl.TextXAlignment=Enum.TextXAlignment.Left
statusLbl.Parent=panel

actionNoticeLabel=Instance.new("TextLabel")
actionNoticeLabel.Size=UDim2.new(1,-20,0,28)
actionNoticeLabel.Position=UDim2.new(0,10,0,148)
actionNoticeLabel.BackgroundColor3=Color3.fromRGB(75,28,35)
actionNoticeLabel.BackgroundTransparency=0.05
actionNoticeLabel.TextColor3=Color3.fromRGB(255,225,225)
actionNoticeLabel.TextSize=10
actionNoticeLabel.Font=Enum.Font.GothamBold
actionNoticeLabel.TextWrapped=true
actionNoticeLabel.Visible=false
actionNoticeLabel.Parent=panel
Instance.new("UICorner",actionNoticeLabel).CornerRadius=UDim.new(0,8)

local function mkBtn(txt,col,y,h)
	local b=Instance.new("TextButton")
	b.Size=UDim2.new(1,-20,0,h or 46)
	b.Position=UDim2.new(0,10,0,y)
	b.BackgroundColor3=col
	b.Text=txt
	b.TextColor3=Color3.fromRGB(255,255,255)
	b.TextSize=17
	b.Font=Enum.Font.GothamBold
	b.Parent=panel
	Instance.new("UICorner",b).CornerRadius=UDim.new(0,10)
	return b
end

local startBtn=mkBtn("🎮 START",Color3.fromRGB(50,185,90),184,48)
local stopBtn=mkBtn("⏹ STOP",Color3.fromRGB(190,50,50),240,48)

local capLbl=Instance.new("TextLabel")
capLbl.Size=UDim2.new(1,-95,0,26)
capLbl.Position=UDim2.new(0,10,0,296)
capLbl.BackgroundTransparency=1
capLbl.Text="🎯 Cap: "..getEffectiveCap()
capLbl.TextColor3=Color3.fromRGB(255,200,60)
capLbl.TextSize=12
capLbl.Font=Enum.Font.GothamBold
capLbl.TextXAlignment=Enum.TextXAlignment.Left
capLbl.Parent=panel

local function mkSBtn(txt,xOff)
	local b=Instance.new("TextButton")
	b.Size=UDim2.new(0,36,0,28)
	b.Position=UDim2.new(1,xOff,0,295)
	b.BackgroundColor3=Color3.fromRGB(40,40,55)
	b.Text=txt
	b.TextColor3=Color3.fromRGB(255,255,255)
	b.TextSize=16
	b.Font=Enum.Font.GothamBold
	b.Parent=panel
	Instance.new("UICorner",b).CornerRadius=UDim.new(0,7)
	return b
end
local capMinus=mkSBtn("−",-90)
local capPlus=mkSBtn("+",-48)

local renderEnabled=true
local renderBtn=mkBtn("👁 Render: ON",Color3.fromRGB(0,120,210),330,38)
renderBtn.TextSize=14

-- ===== Admin WhiteScreen =====
local ADMIN_PASSWORD="hahaha123"
local adminUnlocked=false
local whiteScreenEnabled=true

local oldBlack=game.CoreGui:FindFirstChild("WWHub_BlackScreen")
if oldBlack then oldBlack:Destroy() end
local oldWhite=game.CoreGui:FindFirstChild("WWHub_WhiteScreen")
if oldWhite then oldWhite:Destroy() end

local whiteGui=Instance.new("ScreenGui")
whiteGui.Name="WWHub_WhiteScreen"
whiteGui.ResetOnSpawn=false
whiteGui.IgnoreGuiInset=true
whiteGui.DisplayOrder=999999
whiteGui.ZIndexBehavior=Enum.ZIndexBehavior.Global
whiteGui.Parent=game.CoreGui

local whiteFrame=Instance.new("Frame")
whiteFrame.Size=UDim2.fromScale(1,1)
whiteFrame.Position=UDim2.fromScale(0,0)
whiteFrame.BackgroundColor3=Color3.new(1,1,1)
whiteFrame.BorderSizePixel=0
whiteFrame.ZIndex=1
whiteFrame.Parent=whiteGui

local adminBox=Instance.new("TextBox")
adminBox.Size=UDim2.new(0,176,0,30)
adminBox.Position=UDim2.new(0.5,-88,0,376)
adminBox.BackgroundColor3=Color3.fromRGB(28,28,42)
adminBox.TextColor3=Color3.fromRGB(235,235,245)
adminBox.Text=""
adminBox.PlaceholderText=""
adminBox.ClearTextOnFocus=false
adminBox.TextSize=12
adminBox.Font=Enum.Font.Gotham
adminBox.Parent=panel
Instance.new("UICorner",adminBox).CornerRadius=UDim.new(0,8)
local abs=Instance.new("UIStroke",adminBox)
abs.Color=Color3.fromRGB(54,58,82)
abs.Thickness=1

local whiteToggleBtn=Instance.new("TextButton")
whiteToggleBtn.Size=UDim2.new(0,136,0,28)
whiteToggleBtn.Position=UDim2.new(0.5,-68,0,412)
whiteToggleBtn.BackgroundColor3=Color3.fromRGB(35,105,55)
whiteToggleBtn.Text="WHITE: ON"
whiteToggleBtn.TextColor3=Color3.fromRGB(255,255,255)
whiteToggleBtn.TextSize=11
whiteToggleBtn.Font=Enum.Font.GothamBold
whiteToggleBtn.Visible=false
whiteToggleBtn.Parent=panel
Instance.new("UICorner",whiteToggleBtn).CornerRadius=UDim.new(0,8)

local destroyBtn=Instance.new("TextButton")
destroyBtn.Size=UDim2.new(0,90,0,28)
destroyBtn.Position=UDim2.new(1,-100,0,412)
destroyBtn.BackgroundColor3=Color3.fromRGB(35,35,50)
destroyBtn.Text="DESTROY GUI"
destroyBtn.TextColor3=Color3.fromRGB(200,200,220)
destroyBtn.TextSize=11
destroyBtn.Font=Enum.Font.GothamBold
destroyBtn.Parent=panel
Instance.new("UICorner",destroyBtn).CornerRadius=UDim.new(0,8)

local function setWhiteScreen(state)
	whiteScreenEnabled=state and true or false
	whiteFrame.Visible=whiteScreenEnabled
	whiteToggleBtn.Text=whiteScreenEnabled and "WHITE: ON" or "WHITE: OFF"
	whiteToggleBtn.BackgroundColor3=whiteScreenEnabled and Color3.fromRGB(35,105,55) or Color3.fromRGB(150,45,45)
end
setWhiteScreen(true)

adminBox.FocusLost:Connect(function()
	if adminBox.Text==ADMIN_PASSWORD then
		adminUnlocked=true
		whiteToggleBtn.Visible=true
		notifyAction("Admin","WhiteScreen control unlocked",4)
	end
	adminBox.Text=""
end)

whiteToggleBtn.MouseButton1Click:Connect(function()
	if adminUnlocked then setWhiteScreen(not whiteScreenEnabled) end
end)

local function setStatus(txt,col)
	statusLbl.Text="📊 "..txt
	statusLbl.TextColor3=col or Color3.fromRGB(140,140,165)
	statusLbl.Font=col and Enum.Font.GothamBold or Enum.Font.Gotham
end

toggleBtn.MouseButton1Click:Connect(function() panel.Visible = not panel.Visible end)
startBtn.MouseButton1Click:Connect(function()
	if not IS_FARMER then
		manualOverride=true
		IS_MAIN=true
		IS_AFK=false
		IS_FARMER=true
		FARM_ROLE="MAIN"
		farmIndex=1
		pairX=0
		farmPos=FARM_BASE
		altCFrame=CFrame.new(farmPos)
		notifyAction("Manual Farm","Unlisted account started manually as MAIN",5)
	end
	task.spawn(startFarm)
	startBtn.BackgroundColor3=Color3.fromRGB(40,190,100)
	startBtn.Text="⏸ RUNNING"
end)
stopBtn.MouseButton1Click:Connect(function()
	loopMain = false
	startBtn.BackgroundColor3=Color3.fromRGB(50,185,90) startBtn.Text="🎮 START"
	setStatus("Idle")
end)
capMinus.MouseButton1Click:Connect(function()
	adjustEffectiveCap(-10000)
	capLbl.Text = hasOutsider() and ("🎯 Cap: "..getEffectiveCap()) or "🎯 Cap: OFF (Private)"
end)
capPlus.MouseButton1Click:Connect(function()
	adjustEffectiveCap(10000)
	capLbl.Text = hasOutsider() and ("🎯 Cap: "..getEffectiveCap()) or "🎯 Cap: OFF (Private)"
end)
renderBtn.MouseButton1Click:Connect(function()
	renderEnabled = not renderEnabled
	game:GetService("RunService"):Set3dRenderingEnabled(renderEnabled)
	renderBtn.Text = renderEnabled and "👁 Render: ON" or "👁 Render: OFF"
	renderBtn.BackgroundColor3 = renderEnabled and Color3.fromRGB(0,120,210) or Color3.fromRGB(170,35,35)
end)
closeBtn.MouseButton1Click:Connect(function() panel.Visible = false end)
destroyBtn.MouseButton1Click:Connect(function() loopMain=false if whiteGui then whiteGui:Destroy() end gui:Destroy() end)

-- Auto-start
task.spawn(function()
	task.wait(0.5)
	if IS_FARMER then
		task.spawn(startFarm)
		startBtn.BackgroundColor3=Color3.fromRGB(40,190,100) startBtn.Text="⏸ RUNNING"
	end
end)

-- Status sync (now shows effective/dynamic cap when capped + gold progress)
task.spawn(function()
	while gui and gui.Parent do
		local timer=getTimerValue()
		local level=getLevel()
		local gold=getGold()
		local prestige=getPrestigeCount()
		local info=" | t="..tostring(timer)
		lvValue.Text=tostring(level or "?")
		goldValue.Text="$"..tostring(gold)
		prestigeValue.Text=tostring(prestige)
		capLbl.Text=hasOutsider() and ("🎯 Cap: "..getEffectiveCap()) or "🎯 Cap: OFF (Private)"
		if starting then
			setStatus("Starting...", Color3.fromRGB(255,200,50))
		elseif roundPaused then
			setStatus("⏸ "..(roundPauseReason or "Paused")..info, Color3.fromRGB(255,80,80))
		elseif timerTpDone then
			setStatus("⏱ Safe zone"..info, Color3.fromRGB(255,165,0))
		elseif pointsCapped and hasOutsider() then
			setStatus("🎯 Cap! pts="..getPoints().." (cap="..getEffectiveCap()..")"..info, Color3.fromRGB(255,215,0))
		elseif loopMain then
			setStatus("🎮 "..FARM_ROLE..info,Color3.fromRGB(100,200,255))
		elseif not IS_FARMER then
			setStatus("UNLISTED • press START for manual MAIN",Color3.fromRGB(255,120,120))
		else
			setStatus(FARM_ROLE.." | Idle")
		end
		task.wait(0.5)
	end
end)

-- ===== Farm Loops =====
task.spawn(function()
	makeBase()
	while gui.Parent do
		task.wait(0.25)
		local c = LP.Character
		if c and c.Parent and c ~= handledChar then afterCharLoaded(c) end
	end
end)

task.spawn(function()
	while gui.Parent do
		task.wait(0.03)
		if not loopMain then continue end

		if hasAnyTeamPad() and not roundPaused and not timerTpDone then
			local wrong,observed = isKnownWrongFarmCharacter()
			if wrong then
				if not characterFixing then
					task.spawn(function()
						ensureFarmCharacter("Round start | observed "..tostring(observed))
					end)
				end
				continue
			end
			if not selectingTeam and not characterFixing then task.spawn(selectTeam) end
			continue
		end

		if not selectingTeam and not starting and not roundPaused and not timerTpDone then
			local mcf = getMainCF()
			if not isNearCF(mcf, tpDist) then tpToCF(mcf) end
		end
	end
end)

task.spawn(function()
	while gui.Parent do
		task.wait(0.2)
		if IS_MAIN and loopMain and not starting and not selectingTeam and not roundPaused and not pointsCapped and not timerTpDone and not characterFixing then
			local wrong = isKnownWrongFarmCharacter()
			if not wrong then fireBakugouQ() end
		end
	end
end)

task.spawn(function()
	local candidate = nil
	local candidateHits = 0
	while gui.Parent do
		if loopMain and not starting and not roundPaused and not selectingTeam and not hasAnyTeamPad() then
			local bm = getBlockedMode()
			if bm then
				if candidate == bm then
					candidateHits += 1
				else
					candidate = bm
					candidateHits = 1
				end
				if candidateHits >= 3 then
					notifyAction("Mode Check","Confirmed "..bm.." 3x -> safety reset enabled",5)
					pauseFarm(bm)
					candidate = nil
					candidateHits = 0
				end
			else
				candidate = nil
				candidateHits = 0
			end
			task.wait(0.5)
		else
			candidate = nil
			candidateHits = 0
			task.wait(1)
		end
	end
end)

task.spawn(function()
	while gui.Parent do task.wait(30) if loopMain and not roundPaused then sendWebhook(FARM_ROLE.." Farm Report") end end
end)

task.spawn(function()
	while gui and gui.Parent do task.wait(60) collectgarbage("collect") end
end)
