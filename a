-- Blade Ball Auto Parry - Matcha Rework v3
-- Based on actual BB example math + WabiSabi UI + attribute detection

print("=== BB AP v3 Loading ===")

local ws = game:GetService("Workspace")
local plrs = game:GetService("Players")
local rs = game:GetService("ReplicatedStorage")
local rs2 = game:GetService("RunService")

local lp = plrs.LocalPlayer
local mouse = lp:GetMouse()

local have_m1 = type(mouse1click) == "function" or type(mouse1press) == "function"
local have_draw = type(Drawing) == "table"
print("M1:", have_m1, "Draw:", have_draw)

-- ping
local pingStatsItem
local function getPing()
    if not pingStatsItem then
        pcall(function() pingStatsItem = game:GetService("Stats").Network:FindFirstChild("ServerStatsItem") and game:GetService("Stats").Network.ServerStatsItem:FindFirstChild("Data Ping") end)
    end
    if pingStatsItem and type(memory_read) == "function" then
        local ok, v = pcall(function() return memory_read("double", pingStatsItem.Address + 0xC8) end)
        if ok and v and v > 0 then return v end
    end
    local ok, v = pcall(function() return game:GetService("Stats").Ping end)
    return ok and v or 60
end

local pingHist = {}
local function smoothedPing()
    local raw = getPing()
    table.insert(pingHist, raw); if #pingHist > 50 then table.remove(pingHist, 1) end
    if #pingHist == 0 then return raw end
    local maxP, sumP = 0, 0
    for _, p in ipairs(pingHist) do maxP = math.max(maxP, p); sumP = sumP + p end
    local avgP = sumP / #pingHist
    if (maxP - avgP) > 20 then return maxP + 10 end
    return raw
end

local function dot(a, b) return a and b and a.X*b.X + a.Y*b.Y + a.Z*b.Z or 0 end
local function mag(v) return v and math.sqrt(v.X^2+v.Y^2+v.Z^2) or 0 end
local function norm(v)
    local m = mag(v); return m > 0 and Vector3.new(v.X/m, v.Y/m, v.Z/m) or Vector3.new()
end
local function dist(a, b) return a and b and mag(a - b) or 9999 end
local function sub(a, b) return a and b and a - b or Vector3.new() end

local function attr(c, k)
    local ok, v = pcall(function() return c:GetAttribute(k) end)
    return ok and v
end
local function isReal(c) return attr(c, "realBall") == true end
local function targetOf(c) return attr(c, "target") or "" end

local function bvel(obj)
    if not obj then return Vector3.new() end
    local ok, v = pcall(function() return obj.AssemblyLinearVelocity end)
    if ok and v and v.Magnitude > 0.001 then return v end
    return Vector3.new()
end
local function bpos(obj)
    if not obj then return Vector3.new() end
    local ok, p = pcall(function() return obj.Position end)
    return ok and p or Vector3.new()
end

-- locate balls
local function findBalls()
    local la = rawget(_G, "LuaApp")
    if not la then pcall(function() la = game:GetService("LuaApp") end) end
    if not la then la = game:FindFirstChild("LuaApp") end
    if la then
        local w = la:FindFirstChild("Workspace") or la
        return w:FindFirstChild("Balls")
    end
    return ws:FindFirstChild("Balls")
end
-- UI
loadstring(game:HttpGet("https://scripts.wabisabi.mom/wabi-sabi-ui-lib.lua"))()
local Library = WabiSabi
local Window = Library:CreateWindow({
    Title = "Blade Ball", SubTitle = "AP v3",
    Size = Vector2.new(580, 440), ConfigName = "BBAPv3",
    MinimizeKey = "8", Translucent = true, AutoStep = true,
})

local Main = Window:AddTab({ Title = "Main", Icon = "swords" })
local ms = Main:AddSection("Parry")
ms:AddToggle({ Id = "en", Title = "Auto Parry", Default = false, Keybind = "F1" })
ms:AddDropdown({ Id = "mode", Title = "Mode", Values = { "Mouse", "F Key" }, Default = "Mouse" })
ms:AddToggle({ Id = "auto_ab", Title = "Auto Ability", Default = false })
ms:AddToggle({ Id = "clash", Title = "Auto Clash", Default = false })
ms:AddSlider({ Id = "clash_dist", Title = "Clash Range", Min = 5, Max = 40, Default = 20 })
ms:AddSlider({ Id = "clash_spd", Title = "Clash Min Speed", Min = 10, Max = 100, Default = 40 })

local Vis = Window:AddTab({ Title = "Visuals", Icon = "eye" })
local vs = Vis:AddSection("Display")
vs:AddToggle({ Id = "boxes", Title = "Boxes", Default = true })
vs:AddColorpicker({ Id = "box_col", Title = "Box Color", Default = Color3.fromRGB(100,200,255) })
vs:AddToggle({ Id = "traj", Title = "Trajectory", Default = true })
vs:AddToggle({ Id = "range", Title = "Range Ring", Default = true })
vs:AddToggle({ Id = "dbg", Title = "Debug", Default = true })

local Cfg = Window:AddTab({ Title = "Config", Icon = "settings" })
Window:BuildConfigSection(Cfg)
Window:BuildInterfaceSection(Cfg)

-- state
local State = {
    ball = nil, vel = Vector3.new(), spd = 0, pos = Vector3.new(),
    tgt = "", chasing = false, ping = 60,
    lastAb = 0,
    traj = {}, isCurving = false, lastDot = 1,
    prevVel = Vector3.new(), parryCount = 0,
    parriedBall = nil, parryTime = 0,
    -- auto-tuning TTI
    targetTTI = 0.18,  -- starts at 180ms, auto-adjusts
    parriedAt = 0,     -- TTI when we last parried
    checkBall = nil,   -- ball we're checking result for
    checkTime = 0,     -- when to check result
    checkTTI = 0,      -- TTI used for check parry
    successRate = {50, 0, 0}, -- {window, successes, total}
    errs = {}, loop = 0,
}


local function doParry()
    local m = Library.Options.mode and Library.Options.mode.Value or "Mouse"
    print("[PAR]", os.clock(), "mode:", m, "spd:", math.floor(State.spd), "tgt:", State.tgt)
    if m == "Mouse" then
        if type(mouse1click) == "function" then mouse1click()
        elseif type(mouse1press) == "function" then mouse1press(); task.wait(); mouse1release() end
    else
        pcall(function() keypress(0x46) task.wait() keyrelease(0x46) end)
    end
end

local function doAbility()
    local r = rs:FindFirstChild("Remotes")
    local ar = r and r:FindFirstChild("AbilityButtonPress")
    if ar then
        pcall(function() ar:FireServer() end)
    end
end

-- main loop
local ballContainer, lastScan = nil, 0

rs2.Heartbeat:Connect(function()
    local ok, e = pcall(function()
        local now = tick()
        State.loop = State.loop + 1
        local opt = Library.Options
        if not opt.en or not opt.en.Value then return end

        State.ping = smoothedPing()

        -- auto-tuning result check
        if State.checkBall and now > State.checkTime then
            if State.checkBall.Parent then
                local cv = bvel(State.checkBall)
                local cp = bpos(State.checkBall)
                local chr2 = lp.Character
                if chr2 and cv.Magnitude > 1 then
                    local hrp2 = chr2:FindFirstChild("HumanoidRootPart")
                    if hrp2 then
                        local toP = norm(hrp2.Position - cp)
                        local toward = dot(norm(cv), toP) > 0
                        if toward then
                            State.targetTTI = math.min(State.targetTTI + 0.008, 0.5)
                        else
                            State.targetTTI = math.max(State.targetTTI - 0.004, 0.05)
                        end
                        State.successRate[2] = State.successRate[2] + (toward and 0 or 1)
                        State.successRate[3] = State.successRate[3] + 1
                    end
                end
            end
            State.checkBall = nil
        end

        -- find balls
        if not ballContainer or not ballContainer.Parent or now - lastScan > 0.5 then
            ballContainer = ballContainer or findBalls()
            lastScan = now
        end
        local bc = ballContainer

        -- find real ball
        local ballObj
        if bc then
            for _, c in ipairs(bc:GetChildren()) do
                if isReal(c) then ballObj = c; break end
            end
        end
        if not ballObj then
            for _, c in ipairs(ws:GetChildren()) do
                if isReal(c) then ballObj = c; break end
            end
        end

        State.ball = ballObj
        if not ballObj or not ballObj.Parent or not isReal(ballObj) then
            State.vel = Vector3.new(); State.spd = 0; State.chasing = false
            State.traj = {}; State.parryCount = 0
            return
        end

        State.pos = bpos(ballObj)
        State.vel = bvel(ballObj)
        State.spd = mag(State.vel)
        local tgt = targetOf(ballObj)
        State.tgt = tgt
        State.chasing = string.lower(tgt or "") == string.lower(lp.Name or "")

        -- trajectory tracking for curve detection
        if not State.traj[ballObj] then State.traj[ballObj] = { samples = {}, isCurve = false, lastDot = 1 } end
        local td = State.traj[ballObj]
        table.insert(td.samples, { vel = State.vel })
        if #td.samples > 25 then table.remove(td.samples, 1) end

        -- curve detection
        if #td.samples >= 3 then
            local angleSum = 0
            for i = 2, #td.samples do
                local pv = td.samples[i-1].vel; local cv = td.samples[i].vel
                local pm2 = mag(pv); local cm2 = mag(cv)
                if pm2 > 0 and cm2 > 0 then
                    local d = math.max(-1, math.min(1, dot(norm(pv), norm(cv))))
                    local ang = math.deg(math.acos(d))
                    if ang > 4 then angleSum = angleSum + ang/4 end
                end
            end
            td.isCurve = angleSum > 6
        end

        -- dot to player
        local tpVec = sub(State.pos, (lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") and lp.Character.HumanoidRootPart.Position) or State.pos + Vector3.new(0,0,10))
        local dirToPlayer = norm(tpVec)
        local velDir = norm(State.vel)
        td.lastDot = dot(dirToPlayer, velDir)
        State.isCurving = td.isCurve

        -- TTI (time-to-impact) + distance
        local hrpPos = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") and lp.Character.HumanoidRootPart.Position
        local d = hrpPos and dist(State.pos, hrpPos) or 999
        local tti = State.spd > 0.5 and d / State.spd or 99
        local parryDist = State.spd * (State.targetTTI + (State.ping / 800)) + 2
        if td.isCurve then parryDist = parryDist * 1.1 end
        parryDist = math.min(parryDist, 32)
        parryDist = math.max(parryDist, 6)

        -- dedup: same ball within 0.5s = already handled
        local alreadyParried = State.parriedBall == ballObj and (now - State.parryTime) < 0.5

        -- parry when distance crosses threshold (ONLY if ball targets us)
        local chr = lp.Character
        if chr and State.chasing and State.spd >= 10 and not alreadyParried then
            local hrp = chr:FindFirstChild("HumanoidRootPart")
            if hrp then
                local toward = dot(State.vel, norm(hrp.Position - State.pos)) > 0
                if toward and d > 0 and d < parryDist then
                    doParry()
                    State.parriedBall = ballObj
                    State.parryTime = now
                    State.parriedAt = tti
                    State.parryCount = State.parryCount + 1
                    State.checkBall = ballObj
                    State.checkTime = now + 0.25

                    if opt.auto_ab and opt.auto_ab.Value and now - State.lastAb > 1.2 then
                        doAbility(); State.lastAb = now
                    end
                end
            end
        end

        -- clash: only if this ball was NOT already parried in this frame
        local clashOn = opt.clash and opt.clash.Value
        if clashOn and chr and State.spd >= (opt.clash_spd and opt.clash_spd.Value or 40) and not (State.parriedBall == ballObj and (now - State.parryTime) < 0.15) then
            local hrp = chr:FindFirstChild("HumanoidRootPart")
            if hrp then
                local toward = dot(State.vel, norm(hrp.Position - State.pos)) > 0
                local cd = opt.clash_dist and opt.clash_dist.Value or 20
                if toward and d > 0 and d < cd then
                    doParry()
                    State.parriedBall = ballObj
                    State.parryTime = now
                    State.parryCount = State.parryCount + 1
                    State.checkBall = ballObj
                    State.checkTime = now + 0.25
                end
            end
        end

        State.prevVel = State.vel
    end)
    if not ok then
        local m = tostring(e):sub(1, 80)
        table.insert(State.errs, 1, m); if #State.errs > 5 then table.remove(State.errs) end
    end
end)

-- render
task.spawn(function()
    local ogLines, dotObj, ringLines = {}, {}, {}
    for i = 1, 16 do
        local l = Drawing.new("Line")
        if l then l.Visible = false; l.Thickness = 3; l.Transparency = 0 end
        ringLines[i] = l
    end
    local cursorTxt = Drawing.new("Text")
    if cursorTxt then cursorTxt.Font = Drawing.Fonts.UI; cursorTxt.Size = 14; cursorTxt.Outline = true; cursorTxt.Center = true; cursorTxt.Visible = false end
    for i = 1, 12 do
        local t = Drawing.new("Text")
        if t then t.Font = Drawing.Fonts.UI; t.Size = 12; t.Outline = true; t.Visible = false end
        ogLines[i] = t
    end
    local bg = Drawing.new("Square"); local bdr = Drawing.new("Square")
    if bg then bg.Thickness = 0; bg.Visible = false end
    if bdr then bdr.Filled = false; bdr.Thickness = 1; bdr.Color = Color3.fromRGB(60,60,60); bdr.Visible = false end

    local dbgDrag = { active = false, offX = 0, offY = 0, posX = 10, posY = 10 }
    local prevClick = false

    while true do
        task.wait()
        if Library.Unloaded == true then break end
        if not have_draw then task.wait(1) end

        -- drag handling
        local mx, my
        pcall(function() mx = mouse.X; my = mouse.Y end)
        local _, clicked = pcall(ismouse1pressed)

        if clicked and not prevClick and mx and my then
            if bg and bg.Visible and mx >= dbgDrag.posX and mx <= dbgDrag.posX + 340 and my >= dbgDrag.posY and my <= dbgDrag.posY + 20 then
                dbgDrag.active = true
                dbgDrag.offX = mx - dbgDrag.posX
                dbgDrag.offY = my - dbgDrag.posY
            end
        end
        if dbgDrag.active and clicked and mx and my then
            dbgDrag.posX = mx - dbgDrag.offX
            dbgDrag.posY = my - dbgDrag.offY
        end
        if not clicked then dbgDrag.active = false end
        prevClick = clicked

        local ok, e = pcall(function()
            local opt = Library.Options
            local dOn = opt.dbg and opt.dbg.Value

            if bg then bg.Visible = dOn end
            if bdr then bdr.Visible = dOn end
            for i = 1, 12 do if ogLines[i] then ogLines[i].Visible = dOn end end

            if dOn then
                if bg then bg.Color = Color3.fromRGB(10,10,10); bg.Transparency = 0.15; bg.Size = Vector2.new(340,320); bg.Position = Vector2.new(dbgDrag.posX, dbgDrag.posY) end
                if bdr then bdr.Position = Vector2.new(dbgDrag.posX, dbgDrag.posY); bdr.Size = Vector2.new(340,320) end
                local chr = lp.Character
                local hrp = chr and chr:FindFirstChild("HumanoidRootPart")
                local d = hrp and State.ball and dist(State.pos, hrp.Position) or 0
                local pdist = State.spd > 0.5 and math.min(State.spd * (State.targetTTI + (State.ping / 800)) + 2, 32) or 0
                if State.isCurving then pdist = pdist * 1.1 end
                pdist = math.max(pdist, 6)

                local tti = State.spd > 0.5 and d / State.spd or 0
                local sr = State.successRate[3] > 0 and math.floor(State.successRate[2]/State.successRate[3]*100) or 0

                local lines = {
                    "=== BB AP v3 ===",
                    "Ball:", tostring(State.ball ~= nil),
                    "Speed:", math.floor(State.spd),
                    "Target:", State.tgt, "Chasing:", tostring(State.chasing),
                    "Ping:", State.ping .. "ms",
                    "Dist:", string.format("%.1f", d),
                    "TTI:", string.format("%.2fs", tti),
                    "TTItarget:", string.format("%.2fs", State.targetTTI),
                    "ParryDist:", string.format("%.1f", pdist),
                    "Curve:", tostring(State.isCurving),
                    "Parries:", State.parryCount,
                    "Success:", sr .. "%",
                    "Dedup:", State.parriedBall == State.ball and "yes" or "no",
                    "Clash:", (opt.clash and opt.clash.Value) and "ON" or "OFF",
                    "Err:", #State.errs > 0 and State.errs[1] or "none",
                }
                for i = 1, #lines, 2 do
                    local t = ogLines[(i+1)//2]
                    if t then
                        t.Text = lines[i] .. " " .. tostring(lines[i+1])
                        t.Position = Vector2.new(dbgDrag.posX + 8, dbgDrag.posY + 6 + ((i-1)//2)*18)
                        t.Color = i == 1 and Color3.fromRGB(255,200,100) or Color3.fromRGB(200,230,255)
                    end
                end
            end

    -- range ring (matches heartbeat formula)
    local calcParryDist
    if State.ball and State.spd > 0.5 then
        calcParryDist = State.spd * (State.targetTTI + (State.ping / 800)) + 2
        if State.isCurving then calcParryDist = calcParryDist * 1.1 end
        calcParryDist = math.min(calcParryDist, 32)
        calcParryDist = math.max(calcParryDist, 6)
    else
        calcParryDist = 12
    end
    for i = 1, 16 do if ringLines[i] then ringLines[i].Visible = false end end
    if opt.range and opt.range.Value and lp.Character and State.ball and calcParryDist > 3 then
        local hrp = lp.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            local ppos = hrp.Position; local seg = 16
            for i = 1, seg do
                local a1, a2 = (i-1)/seg*math.pi*2, i/seg*math.pi*2
                local py = ppos.Y - 2
                local p1 = Vector3.new(ppos.X + math.cos(a1)*calcParryDist, py, ppos.Z + math.sin(a1)*calcParryDist)
                local p2 = Vector3.new(ppos.X + math.cos(a2)*calcParryDist, py, ppos.Z + math.sin(a2)*calcParryDist)
                local ok1, s1, v1 = pcall(WorldToScreen, p1)
                local ok2, s2, v2 = pcall(WorldToScreen, p2)
                if ok1 and ok2 and v1 and v2 and ringLines[i] then
                    ringLines[i].Visible = true
                    ringLines[i].From = s1; ringLines[i].To = s2
                    ringLines[i].Color = State.chasing and Color3.fromRGB(100,255,100) or Color3.fromRGB(255,200,100)
                    ringLines[i].Transparency = 0
                end
            end
        end
    end

    -- cursor overlay
    if cursorTxt then
        cursorTxt.Visible = true
        if mx and my then
            cursorTxt.Position = Vector2.new(mx, my + 20)
            cursorTxt.Text = "target=" .. tostring(State.chasing) .. " spd=" .. math.floor(State.spd)
            cursorTxt.Color = State.chasing and Color3.fromRGB(100,255,100) or Color3.fromRGB(255,100,100)
        else
            cursorTxt.Visible = false
        end
    end

    -- trajectory line
    local function wts2(p)
        local ok, s, v = pcall(WorldToScreen, p)
        return ok and s or Vector2.new(), ok and v or false
    end
    for _, l in pairs(dotObj) do if l then l.Visible = false end end
    if opt.traj and opt.traj.Value and State.ball and State.spd > 0.5 then
        local prev
        for i = 0, 5 do
            local p = State.pos + norm(State.vel) * State.spd * (i * 0.35)
            local sv, on = wts2(p)
            if on then
                if prev then
                    if not dotObj[i] then dotObj[i] = Drawing.new("Line") end
                    if dotObj[i] then dotObj[i].Visible = true; dotObj[i].From = prev; dotObj[i].To = sv; dotObj[i].Color = Color3.fromRGB(255,255,0); dotObj[i].Thickness = 3; dotObj[i].Transparency = 0 end
                end
                prev = sv
            end
        end
    end
        end)
        if not ok then
            local m = tostring(e):sub(1,80)
            print("[AP Render]", m)
        end
    end
end)

print("=== BB AP v3 Loaded ===")
