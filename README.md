--- V.8.2 Main Only + Juggernaut full lives reset + M1 only
repeat task.wait(0.1) until game:IsLoaded()

-- ===== CONFIG =====
_G.main  = {"Pevyacy37606", "Athadees29181", "Womeruzayr27750", "Aspenbaab40606", "Zerrtyran1579", "Rivyebbs9019"}
-- ==================



setfpscap(20)
local Players = game:GetService("Players")
local VIM     = game:GetService("VirtualInputManager")
local VUser   = game:GetService("VirtualUser")
local Http    = game:GetService("HttpService")
local UIS     = game:GetService("UserInputService")
local LP      = Players.LocalPlayer


-- เช็ค vip
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local PS = ReplicatedStorage:WaitForChild("PS")

local VIP_CODE = "JblH87"
local CHECK_INTERVAL = 60

local function IsABAVIP()
    local overrides = ReplicatedStorage:FindFirstChild("VipTeamOverrides")
    if not overrides then
        return false
    end

    local vipPlayers = overrides:FindFirstChild("Players")
    if not vipPlayers then
        return false
    end

    return vipPlayers:FindFirstChild(LocalPlayer.Name) ~= nil
end

local function CheckVIP()
    if not IsABAVIP() then
        PS:FireServer("join", VIP_CODE)
    end
end

CheckVIP()

task.spawn(function()
    while true do
        task.wait(CHECK_INTERVAL)
        CheckVIP()
    end
end)


local myName = LP.Name
local IS_MAIN = false
for _, n in ipairs(_G.main) do if n == myName then IS_MAIN = true end end
warn("[WWHub] Role: " .. (IS_MAIN and "MAIN" or "UNKNOWN"))

-- Random server hop disabled

-- ===== Vars =====
local altCFrame   = CFrame.new(20000, 2000, 20000)
local pauseCFrame = CFrame.new(156, 1, -43)
local baseName    = "WWHub_BasePlate"
local tpDist      = 18
local safeLimit   = 8

local m1Hit = CFrame.new(
	97.64178466796875, 497.5, -602.8313598632812,
	0.9989567399024963, 0.006808227859437466, -0.045158419758081436,
	4.656613428188905e-10, 0.9888255000114441, 0.14907847344875336,
	0.04566875472664833, -0.14892295002937317, 0.9877936840057373
)

local loopMain      = false
local starting      = false
local selectingTeam = false
local pointsCapped  = false
local roundPaused   = false
local roundPauseReason = nil
local roundResetting   = false
local handledChar   = nil
local pressedKChar  = nil
local timerTpDone   = false
local gui           = nil
local pointCapLimit = 200
local GOLD_THRESHOLD   = 30000
local LOW_GOLD_CAP     = 100
local GOLD_THRESHOLD_2 = 60000
local MID_GOLD_CAP     = 150

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
	local p = altCFrame.Position return CFrame.lookAt(p + Vector3.new(0,0,-1), p)
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

-- ===== M1 Only =====
local function fireM1()
	fireInput("M1", {air=false, skeyreal=false, skeydown=true, mousehit=m1Hit, md=Vector3.new(0,0,0)})
end
local function pressG()
	pcall(function() VIM:SendKeyEvent(true, Enum.KeyCode.G, false, game) task.wait(0.03) VIM:SendKeyEvent(false, Enum.KeyCode.G, false, game) end)
end
local function pressK()
	pcall(function() VIM:SendKeyEvent(true, Enum.KeyCode.K, false, game) end)
end
local function forceFieldOff() fireInput("ForceFieldOff") end

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

-- ===== Character =====
local function fireRespawnDone()
	local r = LP:WaitForChild("PlayerGui"):FindFirstChild("Respawning")
	if r and r:FindFirstChild("Done") then pcall(function() r.Done:FireServer() end) end
end
local function afterCharLoaded(char)
	if not char or handledChar == char then return end
	handledChar = char pressedKChar = nil
	char:WaitForChild("HumanoidRootPart", 10) char:WaitForChild("Humanoid", 10)
	task.wait(1) fireRespawnDone() task.wait(0.2) forceFieldOff()
end
local function resetChar()
	local c = getChar()
	if c then local h = c:FindFirstChildOfClass("Humanoid") if h then h.Health = 0 end end
	pcall(function() game:GetService("ReplicatedStorage"):WaitForChild("Loaded"):FireServer() end)
	task.wait(1) local nc = getChar() if nc then afterCharLoaded(nc) end
end
local function pressKAfterTP()
	local c = LP.Character
	if not c or pressedKChar == c or not isNearCF(altCFrame, tpDist) then return end
	pressedKChar = c task.wait(0.35) if isNearCF(altCFrame, tpDist) then pressK() end
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
local function getLevel()
	local ok,v = pcall(function()
		local hud = LP.PlayerGui:FindFirstChild("HUD") if not hud then return "?" end
		local lo = hud:FindFirstChild("RightBotCorner") and hud.RightBotCorner:FindFirstChild("Line2") and hud.RightBotCorner.Line2:FindFirstChild("Lvl")
		return lo and tostring(tonumber(lo.Text:match("%d+")) or "?") or "?"
	end)
	return ok and v or "?"
end

-- ===== Gold / Dynamic Cap =====
local function getGold()
	local ok, v = pcall(function()
		return LP:WaitForChild("ReplicatedStats"):WaitForChild("Gold").Value
	end)
	return ok and v or 0
end
local function getEffectiveCap()
	local gold = getGold()
	if gold < GOLD_THRESHOLD then
		return LOW_GOLD_CAP      -- เงินไม่ถึง 30000 -> cap 200
	elseif gold < GOLD_THRESHOLD_2 then
		return MID_GOLD_CAP      -- เงินไม่ถึง 60000 -> cap 300
	end
	return pointCapLimit         -- เงินเกิน 60000 แล้ว -> ใช้ cap เดิม (ปรับได้จาก GUI)
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
				local hud = LP.PlayerGui:FindFirstChild("HUD")
				if hud then
					local lo = hud:FindFirstChild("RightBotCorner") and hud.RightBotCorner:FindFirstChild("Line2") and hud.RightBotCorner.Line2:FindFirstChild("Lvl")
					if lo then lvl = lo.Text end
				end
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
local function getBlockedMode()
	local pg = LP:FindFirstChild("PlayerGui") if not pg then return nil end
	local lb = pg:FindFirstChild("CustomLeaderboard") if not lb then return nil end
	local m = lb:FindFirstChild("Main") or lb
	if m:FindFirstChild("Juggernaut",true) then return "Juggernaut" end
	if m:FindFirstChild("Lives",true)      then return "Lives"      end
	for _, obj in ipairs(m:GetDescendants()) do
		local nm = (obj.Name or ""):lower()
		if nm:find("juggernaut") then return "Juggernaut" end
		if nm:find("lives")      then return "Lives"      end
		if obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton") then
			local tx = (obj.Text or ""):lower()
			if tx:find("juggernaut") then return "Juggernaut" end
			if tx:find("lives")      then return "Lives"      end
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
			pcall(function() hum.Health = 0 end)
			resetCount += 1

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
	pointsCapped = false sendWebhook("Farm Paused — "..roundPauseReason)
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
		if gui and gui.Parent and loopMain then sendWebhook("Farm Resumed — "..last) end
	end)
end

-- Timer watchdog (now uses dynamic Gold-based cap)
task.spawn(function()
	while true do
		task.wait(0.3)
		if loopMain then
			local pts = getPoints() local timer = getTimerValue()
			local capNow = getEffectiveCap()
			if not pointsCapped and pts >= capNow then pointsCapped = true end
			if pointsCapped and timer > 30 and pts < capNow then pointsCapped = false end
			if timer > 0 and timer <= 2 and not timerTpDone and not roundPaused then
				timerTpDone = true tpToSafeZone()
			end
			if timerTpDone then
				local pad = workspace:FindFirstChild("Red Team") or workspace:FindFirstChild("Blue Team") or workspace:FindFirstChild("Green Team")
				if timer > 30 or pad then timerTpDone = false end
			end
		else
			pointsCapped = false timerTpDone = false
		end
	end
end)

-- startFarm
local function startFarm()
	if starting then return end starting = true loopMain = false makeBase()
	fireInput("CharacterButton","Ichigo") task.wait(0.2) fireInput("ClickPlay")
	task.wait(2.5) resetChar() task.wait(2.5)
	fireInput("CharacterButton","Ichigo") task.wait(0.2) fireInput("ClickPlay")
	task.wait(2.5)
	roundPaused=false roundPauseReason=nil roundResetting=false timerTpDone=false pointsCapped=false
	loopMain = true
	sendWebhook("Main Farm Started")
	starting = false
end

-- Team selection — สุ่มสีให้ main แต่ละคนอยู่คนละทีม
-- ดึง index ของตัวเองใน _G.main เพื่อเลือก pad ต่างกัน
local myMainIndex = 0
for i, n in ipairs(_G.main) do if n == myName then myMainIndex = i break end end

local allTeamPads = {"Red Team", "Blue Team", "Green Team", "Yellow Team"}

local function getMyTeamPad()
	-- ถ้าเป็น FFA ไม่มี pad
	if not workspace:FindFirstChild("Red Team") and not workspace:FindFirstChild("Blue Team") then return nil end
	-- หา pad ที่มีอยู่จริงในเกม
	local availablePads = {}
	for _, name in ipairs(allTeamPads) do
		if workspace:FindFirstChild(name) then table.insert(availablePads, name) end
	end
	if #availablePads == 0 then return nil end
	-- เลือก pad ตาม index ของตัวเองใน main list (กระจายทีม)
	local padIndex = ((myMainIndex - 1) % #availablePads) + 1
	return workspace:FindFirstChild(availablePads[padIndex])
end

local function selectTeam()
	if selectingTeam then return end selectingTeam = true
	local pad = getMyTeamPad()
	if not pad then selectingTeam = false return end
	while pad and pad.Parent and not roundPaused do
		-- เช็คว่า pad ยังอยู่ไหม
		local currentPad = getMyTeamPad()
		if not currentPad then break end
		local hrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
		if hrp then
			local pp = currentPad:IsA("BasePart") and currentPad or currentPad:FindFirstChildWhichIsA("BasePart")
			if pp then hrp.CFrame = pp.CFrame + Vector3.new(0,3,0) end
		end
		task.wait(0.1)
	end
	task.wait(0.5) selectingTeam = false
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

-- G spam
task.spawn(function()
	while gui and gui.Parent do
		task.wait(0.05)
		if loopMain and not roundPaused and not timerTpDone and not starting then pressG() end
	end
end)

-- ===== GUI =====
gui = Instance.new("ScreenGui")
gui.Name = "WWHub_GUI_v7" gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling gui.DisplayOrder = 0 gui.Parent = game.CoreGui

local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(0,42,0,42) toggleBtn.Position = UDim2.new(0,10,0.5,-21)
toggleBtn.BackgroundColor3 = Color3.fromRGB(18,18,28) toggleBtn.Text = "⚡" toggleBtn.TextSize = 20
toggleBtn.TextColor3 = Color3.fromRGB(150,70,255) toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.ZIndex = 10 toggleBtn.Parent = gui
Instance.new("UICorner",toggleBtn).CornerRadius = UDim.new(0,10)
local tst = Instance.new("UIStroke",toggleBtn) tst.Color = Color3.fromRGB(110,40,200) tst.Thickness = 2

local panel = Instance.new("Frame")
panel.Size = UDim2.new(0,270,0,290) panel.Position = UDim2.new(0.5,-135,0.5,-145)
panel.BackgroundColor3 = Color3.fromRGB(14,14,22) panel.BorderSizePixel = 0 panel.Active = true panel.Parent = gui
Instance.new("UICorner",panel).CornerRadius = UDim.new(0,14)
local pst = Instance.new("UIStroke",panel) pst.Color = Color3.fromRGB(100,35,190) pst.Thickness = 2

local dragging,dragStart,dragPos
panel.InputBegan:Connect(function(i)
	if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
		dragging=true dragStart=i.Position dragPos=panel.Position end end)
panel.InputEnded:Connect(function(i)
	if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
		dragging=false end end)
if UIS then UIS.InputChanged:Connect(function(i)
	if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
		local d=i.Position-dragStart
		panel.Position=UDim2.new(dragPos.X.Scale,dragPos.X.Offset+d.X,dragPos.Y.Scale,dragPos.Y.Offset+d.Y) end end) end

local header = Instance.new("Frame")
header.Size = UDim2.new(1,0,0,48) header.BackgroundColor3 = Color3.fromRGB(20,20,32)
header.BorderSizePixel = 0 header.Parent = panel
Instance.new("UICorner",header).CornerRadius = UDim.new(0,14)

local titleLbl = Instance.new("TextLabel")
titleLbl.Size = UDim2.new(1,-50,1,0) titleLbl.Position = UDim2.new(0,12,0,0)
titleLbl.BackgroundTransparency = 1 titleLbl.Text = "⚡ WW Hub v7"
titleLbl.TextColor3 = Color3.fromRGB(155,80,255) titleLbl.TextSize = 18 titleLbl.Font = Enum.Font.GothamBold
titleLbl.TextXAlignment = Enum.TextXAlignment.Left titleLbl.Parent = header

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.new(0,32,0,32) closeBtn.Position = UDim2.new(1,-40,0,8)
closeBtn.BackgroundColor3 = Color3.fromRGB(190,35,55) closeBtn.Text = "✕"
closeBtn.TextColor3 = Color3.fromRGB(255,255,255) closeBtn.TextSize = 15 closeBtn.Font = Enum.Font.GothamBold
closeBtn.Parent = header Instance.new("UICorner",closeBtn).CornerRadius = UDim.new(0,8)

local statusLbl = Instance.new("TextLabel")
statusLbl.Size = UDim2.new(1,-20,0,20) statusLbl.Position = UDim2.new(0,10,0,54)
statusLbl.BackgroundTransparency = 1 statusLbl.Text = "📊 Idle"
statusLbl.TextColor3 = Color3.fromRGB(140,140,165) statusLbl.TextSize = 12 statusLbl.Font = Enum.Font.Gotham
statusLbl.TextXAlignment = Enum.TextXAlignment.Left statusLbl.Parent = panel

local div = Instance.new("Frame")
div.Size = UDim2.new(1,-20,0,1) div.Position = UDim2.new(0,10,0,78)
div.BackgroundColor3 = Color3.fromRGB(60,35,110) div.BorderSizePixel = 0 div.Parent = panel

local function mkBtn(txt,col,y,w,xOff)
	local b = Instance.new("TextButton")
	b.Size = UDim2.new(w or 1,-20,0,46) b.Position = UDim2.new(0,xOff or 10,0,y)
	b.BackgroundColor3 = col b.Text = txt b.TextColor3 = Color3.fromRGB(255,255,255)
	b.TextSize = 18 b.Font = Enum.Font.GothamBold b.Parent = panel
	Instance.new("UICorner",b).CornerRadius = UDim.new(0,10) return b
end

local startBtn = mkBtn("🎮 START MAIN", Color3.fromRGB(50,185,90), 88)
local stopBtn  = mkBtn("⏹ STOP",        Color3.fromRGB(190,50,50),  142)

local capLbl = Instance.new("TextLabel")
capLbl.Size = UDim2.new(1,-90,0,18) capLbl.Position = UDim2.new(0,10,0,200)
capLbl.BackgroundTransparency = 1 capLbl.Text = "🎯 Cap: "..pointCapLimit
capLbl.TextColor3 = Color3.fromRGB(255,200,60) capLbl.TextSize = 12 capLbl.Font = Enum.Font.GothamBold
capLbl.TextXAlignment = Enum.TextXAlignment.Left capLbl.Parent = panel

local function mkSBtn(txt,xOff)
	local b = Instance.new("TextButton")
	b.Size = UDim2.new(0,34,0,26) b.Position = UDim2.new(1,xOff,0,197)
	b.BackgroundColor3 = Color3.fromRGB(40,40,55) b.Text = txt
	b.TextColor3 = Color3.fromRGB(255,255,255) b.TextSize = 16 b.Font = Enum.Font.GothamBold
	b.Parent = panel Instance.new("UICorner",b).CornerRadius = UDim.new(0,7) return b
end
local capMinus = mkSBtn("−",-86) local capPlus = mkSBtn("+",-48)

local renderEnabled = true
local renderBtn = mkBtn("👁 Render: ON", Color3.fromRGB(0,120,210), 233)
renderBtn.TextSize = 14

local destroyBtn = Instance.new("TextButton")
destroyBtn.Size = UDim2.new(1,-20,0,26) destroyBtn.Position = UDim2.new(0,10,1,-34)
destroyBtn.BackgroundColor3 = Color3.fromRGB(35,35,50) destroyBtn.Text = "❌ Destroy GUI"
destroyBtn.TextColor3 = Color3.fromRGB(200,200,220) destroyBtn.TextSize = 12 destroyBtn.Font = Enum.Font.GothamBold
destroyBtn.Parent = panel Instance.new("UICorner",destroyBtn).CornerRadius = UDim.new(0,8)

local function setStatus(txt,col)
	statusLbl.Text = "📊 "..txt
	statusLbl.TextColor3 = col or Color3.fromRGB(140,140,165)
	statusLbl.Font = col and Enum.Font.GothamBold or Enum.Font.Gotham
end

toggleBtn.MouseButton1Click:Connect(function() panel.Visible = not panel.Visible end)
startBtn.MouseButton1Click:Connect(function()
	task.spawn(startFarm)
	startBtn.BackgroundColor3 = Color3.fromRGB(40,190,100) startBtn.Text = "⏸ RUNNING"
end)
stopBtn.MouseButton1Click:Connect(function()
	loopMain = false
	startBtn.BackgroundColor3 = Color3.fromRGB(50,185,90) startBtn.Text = "🎮 START MAIN"
	setStatus("Idle")
end)
capMinus.MouseButton1Click:Connect(function()
	pointCapLimit = math.max(10000, pointCapLimit-10000) capLbl.Text = "🎯 Cap: "..pointCapLimit end)
capPlus.MouseButton1Click:Connect(function()
	pointCapLimit = pointCapLimit + 10000 capLbl.Text = "🎯 Cap: "..pointCapLimit end)
renderBtn.MouseButton1Click:Connect(function()
	renderEnabled = not renderEnabled
	game:GetService("RunService"):Set3dRenderingEnabled(renderEnabled)
	renderBtn.Text = renderEnabled and "👁 Render: ON" or "👁 Render: OFF"
	renderBtn.BackgroundColor3 = renderEnabled and Color3.fromRGB(0,120,210) or Color3.fromRGB(170,35,35)
end)
closeBtn.MouseButton1Click:Connect(function() panel.Visible = false end)
destroyBtn.MouseButton1Click:Connect(function() loopMain = false gui:Destroy() end)

-- Auto-start
task.spawn(function()
	task.wait(0.5)
	if IS_MAIN then
		task.spawn(startFarm)
		startBtn.BackgroundColor3 = Color3.fromRGB(40,190,100) startBtn.Text = "⏸ RUNNING"
	end
end)

-- Status sync (now shows effective/dynamic cap when capped + gold progress)
task.spawn(function()
	while gui and gui.Parent do
		local timer = getTimerValue()
		local info = " | Lv:"..getLevel().." | t="..timer
		if starting then
			setStatus("Starting...", Color3.fromRGB(255,200,50))
		elseif roundPaused then
			setStatus("⏸ "..(roundPauseReason or "Paused")..info, Color3.fromRGB(255,80,80))
		elseif timerTpDone then
			setStatus("⏱ Safe zone"..info, Color3.fromRGB(255,165,0))
		elseif pointsCapped then
			setStatus("🎯 Cap! pts="..getPoints().." (cap="..getEffectiveCap()..")"..info, Color3.fromRGB(255,215,0))
		elseif loopMain then
			setStatus("🎮 MAIN"..info, Color3.fromRGB(100,200,255))
		else
			setStatus("Idle")
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
		if not roundPaused and not timerTpDone then pressKAfterTP() end
	end
end)

task.spawn(function()
	while gui.Parent do
		task.wait(0.08)
		if not loopMain then continue end
		local pad = getMyTeamPad()
		if pad and not selectingTeam and not roundPaused and not timerTpDone then task.spawn(selectTeam) end
		if not selectingTeam and not starting and not roundPaused and not timerTpDone then
			local mcf = getMainCF()
			if not isNearCF(mcf, tpDist) then tpToCF(mcf) end
		end
	end
end)

task.spawn(function()
	while gui.Parent do
		task.wait(0.12)
		if loopMain and not starting and not selectingTeam and not roundPaused and not pointsCapped and not timerTpDone then
			fireM1()
		end
	end
end)

task.spawn(function()
	while gui.Parent do
		if loopMain and not starting and not roundPaused then
			local bm = getBlockedMode() if bm then pauseFarm(bm) end
			task.wait(0.5)
		else task.wait(2) end
	end
end)

task.spawn(function()
	while gui.Parent do task.wait(30) if loopMain and not roundPaused then sendWebhook("Main Farm Report") end end
end)

task.spawn(function()
	while gui and gui.Parent do task.wait(60) collectgarbage("collect") end
end)
