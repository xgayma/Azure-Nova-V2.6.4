-- =========================================================
-- AZURE NOVA | Made By Tony Montana
-- =========================================================

local CFG = {
    VERSION     = "3.1.2",
    SCRIPT_NAME = "Azure Nova",
    TABS        = { "Main Menu", "Gun Mods", "Settings", "Credits" },
    TARGET_PARTS = {
        { name = "R15", parts = {"Head","UpperTorso","LowerTorso","LeftUpperArm","LeftLowerArm","LeftHand","RightUpperArm","RightLowerArm","RightHand","LeftUpperLeg","LeftLowerLeg","LeftFoot","RightUpperLeg","RightLowerLeg","RightFoot"} },
        { name = "R6",  parts = {"Head","Torso","Left Arm","Right Arm","Left Leg","Right Leg"} },
        { name = "HumanoidRootPart", parts = {"HumanoidRootPart"} },
        { name = "Humanoid", parts = {"Torso","UpperTorso"} },
        { name = "Bypass", parts = {} }
    },
    WORKSPACE_MODES = { "Default", "Bypass" },
}

-- =========================================================
-- STATE
-- =========================================================
if not _G.AzureNovaUI then _G.AzureNovaUI = {} end
local S = _G.AzureNovaUI

if S.stateVersion ~= CFG.VERSION then
    S.activeTab       = 1
    S.guiOn           = false
    S.isLoading       = true
    S.loadProgress    = 0
    S.menuX           = 300
    S.menuY           = 50
    S.draggingUI      = false
    S.dragOffX        = 0
    S.dragOffY        = 0
    S.prevMouseDown   = false
    S.prevInsertDown  = false
    S.notifications   = {}
    S.drag            = {}

    -- Theme (Azure Blue)
    S.uiR = 0
    S.uiG = 150
    S.uiB = 255

    -- Features
    S.espMaster      = false
    S.espTeams       = false
    S.espIgnoreTeam  = false
    S.espWeapon      = false
    S.espRadar       = false
    S.radarX         = 20
    S.radarY         = 250
    S.radarSize      = 150
    S.radarZoom      = 0.5
    S.espHealthBar   = false
    S.espBoxes       = true
    S.espPartBoxes   = false
    S.espSkeleton    = false
    S.espName        = true
    S.espDistance    = true
    S.espMaxDistance = 2000
    S.univTargetIndex = 2  -- default R6
    S.workspaceModeIndex = 1 -- 1 = Default, 2 = Randomized

    -- Gun Mods
    S.recoilEnabled   = false
    S.recoilStrengthX = 0
    S.recoilStrengthY = 2
    S.recoilTickRate  = 60
    S.recoilLastTick  = 0
    S.fpsToggle       = false
    S.watermarkToggle = true
    S.crosshairToggle = false
    S.crosshairWidth  = 1
    S.crosshairLength = 8

    S.scanDepth    = 2  -- 1=fastest, 2=balanced, 3=deepest
    S.scanProgress = 100
    S.isScanning   = false
    S.lastScanTick = 0
    S.scanSource   = ""

    S.menuKeybind = 0x46  -- F key default
    S.changingKeybind = false

    S.stateVersion = CFG.VERSION
end

-- Type-safe coercion
S.uiR            = tonumber(S.uiR)            or 255
S.uiG            = tonumber(S.uiG)            or 65
S.uiB            = tonumber(S.uiB)            or 65
S.espMaxDistance = tonumber(S.espMaxDistance) or 2000
S.crosshairWidth = tonumber(S.crosshairWidth) or 1
S.crosshairLength= tonumber(S.crosshairLength)or 8
S.scanDepth      = tonumber(S.scanDepth)      or 2
S.workspaceModeIndex = tonumber(S.workspaceModeIndex) or 1
S.menuKeybind     = tonumber(S.menuKeybind)     or 0x46
S.drag           = S.drag           or {}
S.notifications  = S.notifications  or {}

if S.espIgnoreTeam == nil then S.espIgnoreTeam = false end
if S.espWeapon == nil then S.espWeapon = false end
if S.espRadar == nil then S.espRadar = false end
if S.recoilEnabled == nil then S.recoilEnabled = false end

S.recoilStrengthX = tonumber(S.recoilStrengthX) or 0
S.recoilStrengthY = tonumber(S.recoilStrengthY) or 2
S.recoilTickRate  = tonumber(S.recoilTickRate)  or 60

S.radarX = tonumber(S.radarX) or 20
S.radarY = tonumber(S.radarY) or 250
S.radarSize = tonumber(S.radarSize) or 150
S.radarZoom = tonumber(S.radarZoom) or 0.5

-- =========================================================
-- FPS COUNTER
-- =========================================================
if not _G.AzNovaFPS then _G.AzNovaFPS = {last=os.clock(), frames=0, fps=0} end
_G.AzNovaFPS.frames = _G.AzNovaFPS.frames + 1
if os.clock() - _G.AzNovaFPS.last >= 1 then
    _G.AzNovaFPS.fps    = _G.AzNovaFPS.frames
    _G.AzNovaFPS.frames = 0
    _G.AzNovaFPS.last   = os.clock()
end

-- ESP cache persisted across frames
S.espCache = S.espCache or {
    lastUpdate=0, lastHealthUpdate=0,
    entities={}, teamMap={}, charToPlayer={},
    healthMap={}, skeletonMap={}, weaponMap={}, worldSpace=nil
}

-- =========================================================
-- UTILITIES
-- =========================================================
local function clamp(v,mn,mx) return v<mn and mn or (v>mx and mx or v) end

local function ffc(inst, name)
    if not inst or inst==0 then return nil end
    local ok,r = pcall(dx9.FindFirstChild, inst, name)
    return (ok and r and r~=0) and r or nil
end

local function getChildrenSafe(obj)
    if not obj or obj==0 then return {} end
    local ok,c = pcall(dx9.GetChildren, obj)
    return (ok and type(c)=="table") and c or {}
end

local function getPos(obj)
    if not obj or obj==0 then return nil end
    local ok,p = pcall(dx9.GetPosition, obj)
    return (ok and p and type(p.x)=="number") and p or nil
end

local function w2s(pos)
    if not pos then return nil end
    local ok,sp = pcall(dx9.WorldToScreen, {pos.x, pos.y, pos.z})
    return (ok and sp and sp.x) and sp or nil
end

local function dist3(a,b)
    if not a or not b then return math.huge end
    return math.sqrt((a.x-b.x)^2+(a.y-b.y)^2+(a.z-b.z)^2)
end

local function getMouse()
    local ok,m = pcall(dx9.GetMouse)
    if ok and type(m)=="table" then
        return tonumber(m.x or m[1]) or 0, tonumber(m.y or m[2]) or 0
    end
    return 0,0
end

local function pointInRect(px,py,x,y,w,h)
    return px>=x and px<=x+w and py>=y and py<=y+h
end

local function getScreenSize()
    local ok,sz = pcall(dx9.size)
    local sw,sh = 1920,1080
    if ok and type(sz)=="table" then
        sw = tonumber(sz.width  or sz[1]) or sw
        sh = tonumber(sz.height or sz[2]) or sh
    end
    return sw, sh
end

local function sendNotif(text, dur)
    table.insert(S.notifications, 1, {text=text, dur=dur or 2.5, time=os.clock()})
end

-- =========================================================
-- THEME
-- =========================================================
local function makeTheme()
    local r = clamp(S.uiR, 0, 255)
    local g = clamp(S.uiG, 0, 255)
    local b = clamp(S.uiB, 0, 255)
    return {
        BACKGROUND = {18,18,18},
        TOPBAR     = {24,24,24},
        ACCENT     = {r, g, b},
        BORDER     = {45,45,45},
        TEXT       = {240,240,250},
        TEXT_DIM   = {140,140,140},
        TAB_HOVER  = {35,35,35},
    }
end

-- =========================================================
-- INPUT
-- =========================================================
local mx, my = getMouse()

local mouseDown = false
pcall(function() mouseDown = dx9.isLeftClickHeld() and true or false end)
local mousePressed = mouseDown and not S.prevMouseDown
S.prevMouseDown = mouseDown

local insertDown = false
pcall(function() insertDown = dx9.is_key_down(S.menuKeybind) == true end)
local key = nil
pcall(function() key = dx9.GetKey() end)
if type(key)=="string" then
    local lk = string.lower(key):gsub("%[",""):gsub("%]",""):gsub("%s","")
    local keyNames = {
        [0x41]="a", [0x42]="b", [0x43]="c", [0x44]="d", [0x45]="e", [0x46]="f",
        [0x47]="g", [0x48]="h", [0x49]="i", [0x4A]="j", [0x4B]="k", [0x4C]="l",
        [0x4D]="m", [0x4E]="n", [0x4F]="o", [0x50]="p", [0x51]="q", [0x52]="r",
        [0x53]="s", [0x54]="t", [0x55]="u", [0x56]="v", [0x57]="w", [0x58]="x",
        [0x59]="y", [0x5A]="z", [0x2D]="insert", [0x1B]="escape", [0x20]="space",
        [0x0D]="enter", [0x09]="tab", [0x10]="shift", [0x11]="ctrl", [0x12]="alt"
    }
    if keyNames[S.menuKeybind] and lk == keyNames[S.menuKeybind] then
        insertDown = true
    end
end

if insertDown and not S.prevInsertDown then
    S.guiOn = not S.guiOn
end
S.prevInsertDown = insertDown

if not mouseDown then
    for k in pairs(S.drag) do S.drag[k] = false end
    S.draggingUI = false
end

local interactable = S.guiOn
local SW, SH = getScreenSize()

-- =========================================================
-- AZURE-STYLE UI COMPONENTS
-- =========================================================
local function drawToggle(x, y, w, title, value, T)
    local hov = pointInRect(mx, my, x, y, w, 22)
    if value then
        pcall(dx9.DrawFilledBox, {x, y+3}, {x+14, y+17}, T.ACCENT)
    else
        pcall(dx9.DrawFilledBox, {x, y+3}, {x+14, y+17}, T.BACKGROUND)
        pcall(dx9.DrawBox,       {x, y+3}, {x+14, y+17}, T.BORDER)
        if hov and interactable then
            pcall(dx9.DrawFilledBox, {x+2, y+5}, {x+12, y+15}, T.TAB_HOVER)
        end
    end
    pcall(dx9.DrawString, {x+22, y+5}, (hov and interactable) and T.TEXT or T.TEXT_DIM, title)
    return interactable and mousePressed and hov
end

local function drawSlider(x, y, w, title, valName, minV, maxV, T, storage)
    storage = storage or S
    local raw = storage[valName]
    local val = clamp(tonumber(raw) or minV, minV, maxV)
    local hov = pointInRect(mx, my, x, y, w, 30)
    pcall(dx9.DrawString, {x, y}, (hov and interactable) and T.TEXT or T.TEXT_DIM,
        title..": "..tostring(math.floor(val)))
    local ty = y+16
    pcall(dx9.DrawFilledBox, {x, ty}, {x+w, ty+6}, T.BACKGROUND)
    pcall(dx9.DrawBox,       {x, ty}, {x+w, ty+6}, T.BORDER)
    local pct = (maxV > minV) and (val-minV)/(maxV-minV) or 0
    pcall(dx9.DrawFilledBox, {x, ty}, {x+math.max(1, w*pct), ty+6}, T.ACCENT)

    local dn = "sl_"..valName
    if interactable and mousePressed and pointInRect(mx,my,x,ty-4,w,14) then
        S.drag[dn] = true
    end
    if S.drag[dn] then
        storage[valName] = minV + clamp((mx-x)/w, 0, 1)*(maxV-minV)
    end
end

local function drawButton(x, y, w, title, T)
    local hov = pointInRect(mx, my, x, y, w, 22)
    pcall(dx9.DrawFilledBox, {x, y+2}, {x+w, y+20}, T.BACKGROUND)
    pcall(dx9.DrawBox,       {x, y+2}, {x+w, y+20}, T.BORDER)
    if hov and interactable then
        pcall(dx9.DrawFilledBox, {x+2, y+4}, {x+w-2, y+18}, T.TAB_HOVER)
    end
    local tw = #title*7
    pcall(dx9.DrawString, {x+(w-tw)/2, y+5}, (hov and interactable) and T.TEXT or T.TEXT_DIM, title)
    return interactable and mousePressed and hov
end

local function drawSectionHeader(x, y, w, title, T)
    pcall(dx9.DrawString,    {x, y},       T.ACCENT, title)
    pcall(dx9.DrawFilledBox, {x, y+18}, {x+w, y+19}, T.BORDER)
end

-- =========================================================
-- AZURE NOVA ESP LOGIC
-- =========================================================
local function resolveMainWorld(game)
    if S.workspaceModeIndex == 1 then
        local ok,ws = pcall(function() return game:GetService("Workspace") end)
        if ok and ws and ws~=0 then return ws end
        return ffc(game, "Workspace")
    else
        local ok,ws = pcall(function() return dx9.FindFirstChildOfClass(game, "Workspace") end)
        if not ws or ws==0 then ws = ffc(game, "Workspace") end
        if ws and ws~=0 then
            local pfp = ffc(ws, "Players")
            if pfp and pfp~=0 then return pfp end
            return ws
        end
        return nil
    end
end

local function collectEntities(worldContainer)
    local results = {}
    if not worldContainer or worldContainer==0 then return results end
    local targetConfig = CFG.TARGET_PARTS[S.univTargetIndex] or {parts={"UpperTorso"}}
    local depth = clamp(S.scanDepth or 2, 1, 3)

    local function hasTargetPart(m)
        if targetConfig.name == "Bypass" then
            local kids = getChildrenSafe(m)
            local partCount = 0
            for _, k in ipairs(kids) do
                local cType = nil
                pcall(function() cType = dx9.GetClassName(k) end)
                if not cType then pcall(function() cType = dx9.GetType(k) end) end
                if cType == "Part" or cType == "MeshPart" then partCount = partCount + 1 end
            end
            return partCount > 3
        else
            for _,pName in ipairs(targetConfig.parts) do
                if ffc(m, pName) then return true end
            end
            return false
        end
    end

    local level1 = getChildrenSafe(worldContainer)
    for i=1,#level1 do
        local child = level1[i]
        if hasTargetPart(child) then
            table.insert(results, child)
        elseif depth >= 2 then
            local level2 = getChildrenSafe(child)
            for j=1,#level2 do
                local sub = level2[j]
                if hasTargetPart(sub) then
                    table.insert(results, sub)
                elseif depth >= 3 then
                    local level3 = getChildrenSafe(sub)
                    for k=1,#level3 do
                        if hasTargetPart(level3[k]) then
                            table.insert(results, level3[k])
                        end
                    end
                end
            end
        end
    end
    return results
end

local function scanDeepTeam(player)
    if not player or player==0 then return nil end
    local ok,t = pcall(dx9.GetTeam, player)
    if ok and type(t)=="string" and t~="" then return t end
    local attrs = {"Team","team","TeamId","Role","role","Faction","faction"}
    for _,an in ipairs(attrs) do
        local okA,attr = pcall(function() return player:GetAttribute(an) end)
        if okA and attr~=nil then return tostring(attr) end
    end
    return nil
end

local function buildCharacterTeamMap()
    local ok,game = pcall(dx9.GetDatamodel)
    if not ok or not game then return {} end
    local ps = ffc(game,"Players")
    if not ps then return {} end
    local players = getChildrenSafe(ps)

    local localPlayer = nil
    pcall(function() localPlayer = dx9.get_localplayer() end)
    local myTeam  = localPlayer and scanDeepTeam(localPlayer)
    local nameIdx = {}
    local count   = 0
    if myTeam then nameIdx[myTeam]=1; count=1 end
    for _,p in ipairs(players) do
        local t = scanDeepTeam(p)
        if t and not nameIdx[t] then count=count+1; nameIdx[t]=count end
    end
    local map = {}
    for _,p in ipairs(players) do
        local t = scanDeepTeam(p)
        if t and nameIdx[t] then
            local c=nil
            pcall(function() c=dx9.GetCharacter(p) end)
            if not c then pcall(function() c=p.Character end) end
            if c and c~=0 then map[c]=nameIdx[t] end
        end
    end
    return map
end

local function fetchVal(obj, prop, dx9Func)
    local vals = {}
    pcall(function() local v=obj[prop];           if type(v)=="number" then table.insert(vals,v) end end)
    pcall(function() local v=dx9.GetProperty(obj,prop); if type(v)=="number" then table.insert(vals,v) end end)
    pcall(function()
        if dx9Func then
            local v=dx9[dx9Func](obj)
            if type(v)=="number" then table.insert(vals,v) end
        end
    end)
    for _,v in ipairs(vals) do if v>0 then return v end end
    return vals[1]
end

local function scanDeepHealth(entity, playerObj)
    if not entity or entity==0 then return 100,100 end
    local h,mh = nil,nil
    local hum = ffc(entity,"Humanoid")
    if hum and hum~=0 then
        h  = fetchVal(hum,"Health","GetHealth")
        mh = fetchVal(hum,"MaxHealth","GetMaxHealth")
        if h and mh and mh>0 then return h,mh end
    end
    local attrH  = {"Health","HP","health","hp"}
    local attrMH = {"MaxHealth","MaxHP","maxhealth","maxhp"}
    for _,an in ipairs(attrH)  do local ok,v=pcall(function() return entity:GetAttribute(an) end); if ok and type(v)=="number" then h=v;  break end end
    for _,an in ipairs(attrMH) do local ok,v=pcall(function() return entity:GetAttribute(an) end); if ok and type(v)=="number" and v>0 then mh=v; break end end
    return h or 100, mh or math.max(100, h or 100)
end

local function drawSkeleton(pmap, col)
    if not pmap then return end
    local links = {
        {"Head","Torso"},{"Torso","Left Arm"},{"Torso","Right Arm"},{"Torso","Left Leg"},{"Torso","Right Leg"},
        {"Head","UpperTorso"},{"UpperTorso","LowerTorso"},
        {"UpperTorso","LeftUpperArm"},{"LeftUpperArm","LeftLowerArm"},{"LeftLowerArm","LeftHand"},
        {"UpperTorso","RightUpperArm"},{"RightUpperArm","RightLowerArm"},{"RightLowerArm","RightHand"},
        {"LowerTorso","LeftUpperLeg"},{"LeftUpperLeg","LeftLowerLeg"},{"LeftLowerLeg","LeftFoot"},
        {"LowerTorso","RightUpperLeg"},{"RightUpperLeg","RightLowerLeg"},{"RightLowerLeg","RightFoot"}
    }
    for _,link in ipairs(links) do
        local p1,p2 = pmap[link[1]], pmap[link[2]]
        if p1 and p2 then
            local pos1,pos2 = getPos(p1), getPos(p2)
            if pos1 and pos2 then
                local s1,s2 = w2s(pos1), w2s(pos2)
                if s1 and s2 then pcall(dx9.DrawLine,{s1.x,s1.y},{s2.x,s2.y},col) end
            end
        end
    end
end

local function drawCornerBox(x, y, w, h, col)
    local l  = math.max(4, math.floor(w * 0.25))
    local ex = x+w
    local ey = y+h
    pcall(dx9.DrawLine, {x,   y},   {x+l, y},   col)
    pcall(dx9.DrawLine, {x,   y},   {x,   y+l}, col)
    pcall(dx9.DrawLine, {ex-l,y},   {ex,  y},   col)
    pcall(dx9.DrawLine, {ex,  y},   {ex,  y+l}, col)
    pcall(dx9.DrawLine, {x,   ey},  {x+l, ey},  col)
    pcall(dx9.DrawLine, {x,   ey-l},{x,   ey},  col)
    pcall(dx9.DrawLine, {ex-l,ey},  {ex,  ey},  col)
    pcall(dx9.DrawLine, {ex,  ey-l},{ex,  ey},  col)
end

local PART_SIZES = {
    Head             = {0.50, 0.50, 0.50},
    Torso            = {1.00, 0.75, 0.50},
    UpperTorso       = {1.00, 0.65, 0.50},
    LowerTorso       = {0.80, 0.50, 0.50},
    HumanoidRootPart = {1.00, 1.50, 0.50},
    ["Left Arm"]     = {0.40, 0.75, 0.40},
    ["Right Arm"]    = {0.40, 0.75, 0.40},
    ["Left Leg"]     = {0.40, 0.75, 0.40},
    ["Right Leg"]    = {0.40, 0.75, 0.40},
    LeftUpperArm     = {0.38, 0.60, 0.38},
    LeftLowerArm     = {0.35, 0.55, 0.35},
    LeftHand         = {0.30, 0.30, 0.25},
    RightUpperArm    = {0.38, 0.60, 0.38},
    RightLowerArm    = {0.35, 0.55, 0.35},
    RightHand        = {0.30, 0.30, 0.25},
    LeftUpperLeg     = {0.40, 0.65, 0.40},
    LeftLowerLeg     = {0.38, 0.60, 0.38},
    LeftFoot         = {0.40, 0.25, 0.50},
    RightUpperLeg    = {0.40, 0.65, 0.40},
    RightLowerLeg    = {0.38, 0.60, 0.38},
    RightFoot        = {0.40, 0.25, 0.50},
}
local BOX_EDGES = {
    {1,2},{2,3},{3,4},{4,1},  -- top face
    {5,6},{6,7},{7,8},{8,5},  -- bottom face
    {1,5},{2,6},{3,7},{4,8},  -- verticals
}
local function draw3DPartBox(center, pName, col)
    if not center then return end
    local sz = PART_SIZES[pName] or {0.40, 0.50, 0.40}
    local hw,hh,hd = sz[1], sz[2], sz[3]
    local c = center
    local corners = {
        {x=c.x-hw, y=c.y+hh, z=c.z-hd},
        {x=c.x+hw, y=c.y+hh, z=c.z-hd},
        {x=c.x+hw, y=c.y+hh, z=c.z+hd},
        {x=c.x-hw, y=c.y+hh, z=c.z+hd},
        {x=c.x-hw, y=c.y-hh, z=c.z-hd},
        {x=c.x+hw, y=c.y-hh, z=c.z-hd},
        {x=c.x+hw, y=c.y-hh, z=c.z+hd},
        {x=c.x-hw, y=c.y-hh, z=c.z+hd},
    }
    local sc = {}
    for i=1,8 do sc[i] = w2s(corners[i]) end
    for _,e in ipairs(BOX_EDGES) do
        local s1,s2 = sc[e[1]], sc[e[2]]
        if s1 and s2 then pcall(dx9.DrawLine,{s1.x,s1.y},{s2.x,s2.y},col) end
    end
end

local function updateProgressiveScanner()
    if not S.isScanning then return end
    if os.clock()-S.lastScanTick >= 0.05 then
        S.scanProgress = S.scanProgress + math.random(15,25)
        S.lastScanTick = os.clock()
        if S.scanProgress >= 100 then
            S.scanProgress = 100
            S.isScanning   = false
            sendNotif("Scan Complete", 2)
        end
    end
end

local function runESP(T)
    if not S.espMaster or S.isScanning then return end
    local ok,game = pcall(dx9.GetDatamodel)
    if not ok or not game then return end

    local now   = os.clock()
    local cache = S.espCache

    if now - cache.lastUpdate > 0.5 then
        cache.worldSpace = resolveMainWorld(game)
        cache.entities   = cache.worldSpace and collectEntities(cache.worldSpace) or {}
        cache.teamMap    = (S.espTeams or S.espIgnoreTeam) and buildCharacterTeamMap() or {}

        local sParts = {
            "Head","Torso","Left Arm","Right Arm","Left Leg","Right Leg",
            "UpperTorso","LowerTorso","LeftUpperArm","LeftLowerArm","LeftHand",
            "RightUpperArm","RightLowerArm","RightHand",
            "LeftUpperLeg","LeftLowerLeg","LeftFoot",
            "RightUpperLeg","RightLowerLeg","RightFoot"
        }
        cache.skeletonMap = {}
        for i=1,#cache.entities do
            local e=cache.entities[i]
            local pmap={}
            for _,pn in ipairs(sParts) do pmap[pn]=ffc(e,pn) end
            cache.skeletonMap[e]=pmap
        end

        cache.weaponMap = {}
        if S.espWeapon then
            for i=1,#cache.entities do
                local e = cache.entities[i]
                local kids = getChildrenSafe(e)
                for j=1,#kids do
                    local k = kids[j]
                    local cType = nil
                    pcall(function() cType = dx9.GetClassName(k) end)
                    if not cType then pcall(function() cType = dx9.GetType(k) end) end
                    if cType == "Tool" then
                        pcall(function() cache.weaponMap[e] = dx9.GetName(k) end)
                        break
                    end
                end
            end
        end

        local gamePlayers={}
        pcall(function() gamePlayers=getChildrenSafe(game:GetService("Players")) end)
        cache.charToPlayer={}
        for _,p in ipairs(gamePlayers) do
            local c=nil
            pcall(function() c=dx9.GetCharacter(p) end)
            if not c then pcall(function() c=p.Character end) end
            if c and c~=0 then cache.charToPlayer[c]=p end
        end
        cache.lastUpdate = now
    end

    if not S.espMaster then return end

    local updateHealth = (now - cache.lastHealthUpdate > 0.1)
    if updateHealth then cache.lastHealthUpdate = now end

    local localPlayer = nil
    pcall(function() localPlayer = dx9.get_localplayer() end)
    local localChar = nil
    if localPlayer then pcall(function() localChar = localPlayer.Character end) end

    local pPos = {x=0,y=0,z=0}
    pcall(function()
        if localPlayer then
            local ok2,pos = pcall(function() return localPlayer.Position end)
            if ok2 and type(pos)=="table" and pos.x then pPos=pos end
        end
    end)
    if pPos.x==0 and pPos.y==0 and localChar then
        local r = ffc(localChar,"HumanoidRootPart") or ffc(localChar,"Torso")
        if r then pPos = getPos(r) or pPos end
    end

    local maxH = SH * 0.9
    local targetConfig = CFG.TARGET_PARTS[S.univTargetIndex] or {parts={"UpperTorso"}}

    -- Draw Minimap Radar Background
    if S.espRadar then
        pcall(dx9.DrawFilledBox, {S.radarX, S.radarY}, {S.radarX+S.radarSize, S.radarY+S.radarSize}, {15, 15, 15})
        pcall(dx9.DrawBox, {S.radarX, S.radarY}, {S.radarX+S.radarSize, S.radarY+S.radarSize}, T.BORDER)
        -- Draw LocalPlayer at Center
        local cX = S.radarX + S.radarSize/2
        local cY = S.radarY + S.radarSize/2
        pcall(dx9.DrawFilledBox, {cX-1, cY-1}, {cX+1, cY+1}, {255, 255, 255})
    end

    for i=1,#cache.entities do
        pcall(function()
            local entity = cache.entities[i]
            if entity==localChar then return end
            if S.espIgnoreTeam and cache.teamMap[entity] == 1 then return end

            local minX,minY,maxX,maxY = 99999,99999,-99999,-99999
            local centerPos = nil
            local foundAny  = false

            if targetConfig.name == "Bypass" then
                local kids = getChildrenSafe(entity)
                for _, part in ipairs(kids) do
                    local pt = nil
                    pcall(function() pt = dx9.GetClassName(part) end)
                    if not pt then pcall(function() pt = dx9.GetType(part) end) end
                    if pt == "Part" or pt == "MeshPart" then
                        local p3d = getPos(part)
                        if p3d then
                            if not centerPos then centerPos = p3d end
                            local hh = 1.2
                            local pts = {
                                w2s({x=p3d.x, y=p3d.y+hh, z=p3d.z}),
                                w2s({x=p3d.x, y=p3d.y-hh, z=p3d.z}),
                                w2s({x=p3d.x+hh, y=p3d.y, z=p3d.z}),
                                w2s({x=p3d.x-hh, y=p3d.y, z=p3d.z}),
                            }
                            for _,sp in ipairs(pts) do
                                if sp then
                                    foundAny = true
                                    if sp.x < minX then minX=sp.x end
                                    if sp.x > maxX then maxX=sp.x end
                                    if sp.y < minY then minY=sp.y end
                                    if sp.y > maxY then maxY=sp.y end
                                end
                            end
                        end
                    end
                end
            else
                for _,pName in ipairs(targetConfig.parts) do
                    local part = ffc(entity, pName)
                    if part then
                        local p3d = getPos(part)
                        if p3d then
                            if not centerPos then centerPos=p3d end
                            local hh = (pName=="Head") and 0.6
                                    or (pName=="HumanoidRootPart") and 2.5
                                    or (pName=="UpperTorso" or pName=="Torso" or pName=="LowerTorso") and 1.2
                                    or 0.8
                            local pts = {
                                w2s({x=p3d.x,     y=p3d.y+hh, z=p3d.z}),
                                w2s({x=p3d.x,     y=p3d.y-hh, z=p3d.z}),
                                w2s({x=p3d.x+hh,  y=p3d.y,    z=p3d.z}),
                                w2s({x=p3d.x-hh,  y=p3d.y,    z=p3d.z}),
                            }
                            for _,sp in ipairs(pts) do
                                if sp then
                                    foundAny = true
                                    if sp.x < minX then minX=sp.x end
                                    if sp.x > maxX then maxX=sp.x end
                                    if sp.y < minY then minY=sp.y end
                                    if sp.y > maxY then maxY=sp.y end
                                end
                            end
                        end
                    end
                end
            end

            if not foundAny or not centerPos then return end
            local d = dist3(centerPos, pPos)
            if d > S.espMaxDistance then return end

            local bx,by = minX,minY
            local bw,bh = maxX-minX, maxY-minY

            local col = T.ACCENT
            if S.espTeams then
                local idx = cache.teamMap[entity]
                if     idx==1             then col={50,255,50}
                elseif idx and idx>1      then col={255,50,50} end
            end

            -- Minimap Radar drawing
            if S.espRadar then
                local rx = centerPos.x - pPos.x
                local rz = centerPos.z - pPos.z
                local mx = rx * S.radarZoom
                local my = rz * S.radarZoom
                local radHalf = S.radarSize / 2
                
                -- Circle clamping
                local distMap = math.sqrt(mx*mx + my*my)
                if distMap > radHalf - 2 then
                    mx = (mx / distMap) * (radHalf - 2)
                    my = (my / distMap) * (radHalf - 2)
                end
                
                local sx = S.radarX + radHalf + mx
                local sy = S.radarY + radHalf + my
                pcall(dx9.DrawFilledBox, {sx-2, sy-2}, {sx+2, sy+2}, col)
            end

            if bh>maxH or bw>maxH*1.5 or bw<2 or bh<2 then return end

            if S.espBoxes    then drawCornerBox(bx,by,bw,bh,col) end
            if S.espSkeleton then drawSkeleton(cache.skeletonMap[entity], col) end

            if S.espPartBoxes then
                for _,pName in ipairs(targetConfig.parts) do
                    local part=ffc(entity,pName)
                    if part then
                        local p3d=getPos(part)
                        if p3d then draw3DPartBox(p3d, pName, col) end
                    end
                end
            end

            if S.espHealthBar then
                if updateHealth or not cache.healthMap[entity] then
                    local pObj=cache.charToPlayer[entity]
                    local hp,mhp=scanDeepHealth(entity,pObj)
                    cache.healthMap[entity]={hp=hp,maxHp=mhp}
                end
                local c2   = cache.healthMap[entity]
                local hpct = clamp(c2.hp/math.max(1,c2.maxHp),0,1)
                local barX = bx-7
                local barH = bh*hpct
                pcall(dx9.DrawFilledBox,{barX-1,by-1},{barX+3,by+bh+1},{0,0,0})
                pcall(dx9.DrawFilledBox,{barX,by+bh-barH},{barX+2,by+bh},
                    {math.floor((1-hpct)*255), math.floor(hpct*255), 0})
            end

            local name=""
            pcall(function()
                local p=cache.charToPlayer[entity]
                if p then
                    local ok2,n=pcall(function() return p.Name end)
                    if ok2 and type(n)=="string" then name=n end
                end
            end)
            if S.espName then
                if name=="" then pcall(function() name=dx9.GetName(entity) or "" end) end
                if name~="" then
                    local tw=#name*3.5
                    pcall(dx9.DrawString,{bx+bw/2-tw, by-14},col, name)
                end
            end

            if S.espDistance then
                local dText=math.floor(d).."m"
                pcall(dx9.DrawString,{bx+bw/2-#dText*3.5, by+bh+4},{200,200,200},dText)
            end

            if S.espWeapon then
                local wName = cache.weaponMap[entity]
                if wName and wName ~= "" then
                    local yOff = S.espDistance and 16 or 4
                    pcall(dx9.DrawString,{bx+bw/2-#wName*3.5, by+bh+yOff},{220,200,100}, wName)
                end
            end
        end)
    end
end



-- =========================================================
-- RENDER LOOP
-- =========================================================
local T = makeTheme()

if S.isLoading then
    S.loadProgress = math.min(1, S.loadProgress + 0.005)
    if S.loadProgress >= 1 then S.isLoading = false end

    local lw,lh = 300,120
    local lx=(SW-lw)/2
    local ly=(SH-lh)/2

    pcall(dx9.DrawFilledBox,{lx-2,ly-2},{lx+lw+2,ly+lh+2},{5,5,8})
    pcall(dx9.DrawFilledBox,{lx,  ly},  {lx+lw,  ly+lh},  T.BACKGROUND)
    pcall(dx9.DrawBox,      {lx,  ly},  {lx+lw,  ly+lh},  T.BORDER)
    pcall(dx9.DrawFilledBox,{lx,  ly},  {lx+lw,  ly+2},   T.ACCENT)

    local t1=CFG.SCRIPT_NAME
    pcall(dx9.DrawString,{lx+(lw-#t1*7)/2, ly+22},T.ACCENT,   t1)
    local t2="Made By Tony Montana"
    pcall(dx9.DrawString,{lx+(lw-#t2*7)/2, ly+42},T.TEXT_DIM, t2)

    local barW,barH = 260,8
    local barX=lx+(lw-barW)/2
    local barY=ly+82
    pcall(dx9.DrawFilledBox,{barX-1,barY-1},{barX+barW+1,barY+barH+1},T.BORDER)
    pcall(dx9.DrawFilledBox,{barX,barY},{barX+barW*S.loadProgress,barY+barH},T.ACCENT)
    local pct=tostring(math.floor(S.loadProgress*100)).."%"
    pcall(dx9.DrawString,{lx+lw/2-#pct*3.5, barY-18},T.TEXT, pct)
else
    updateProgressiveScanner()
    runESP(T)

    pcall(function()
        if S.recoilEnabled and mouseDown then
            local rInt = 1 / math.max(1, math.floor(S.recoilTickRate))
            local rNow = os.clock()
            if rNow - S.recoilLastTick >= rInt then
                S.recoilLastTick = rNow
                local dx = (math.random() * S.recoilStrengthX * 2 - S.recoilStrengthX)
                local dy = S.recoilStrengthY
                
                -- Attempt multiple mouse movement methods for compatibility
                if type(mousemoverel) == "function" then
                    pcall(mousemoverel, dx, dy)
                elseif type(dx9.mouse_move) == "function" then
                    pcall(dx9.mouse_move, dx, dy)
                else
                    local cx, cy = getScreenSize()
                    cx, cy = math.floor(cx/2), math.floor(cy/2)
                    pcall(dx9.FirstPersonAim, {cx + dx, cy + dy}, 1, 1)
                end
            end
        end
    end)

    if S.watermarkToggle then
        local wm = CFG.SCRIPT_NAME.." v"..CFG.VERSION
        if S.fpsToggle then wm = wm.." | FPS: ".._G.AzNovaFPS.fps end
        local wmW = #wm*7+20
        pcall(dx9.DrawFilledBox,{18,18},{18+wmW+4,47},{5,5,8})
        pcall(dx9.DrawFilledBox,{20,20},{20+wmW,  45},T.TOPBAR)
        pcall(dx9.DrawBox,      {20,20},{20+wmW,  45},T.BORDER)
        pcall(dx9.DrawFilledBox,{20,20},{20+wmW,  22},T.ACCENT)
        pcall(dx9.DrawString,   {30,25},T.TEXT, wm)
    end

    if S.crosshairToggle then
        local cX,cY = math.floor(mx), math.floor(my)
        local s = S.crosshairLength
        local g = 3
        local t = math.max(1, S.crosshairWidth)
        local function dRect(x,y,w,h)
            pcall(dx9.DrawFilledBox,{x-1,y-1},{x+w+1,y+h+1},{0,0,0})
            pcall(dx9.DrawFilledBox,{x,  y},  {x+w,  y+h},  T.ACCENT)
        end
        dRect(cX-t/2, cY-g-s, t, s)
        dRect(cX-t/2, cY+g,   t, s)
        dRect(cX-g-s, cY-t/2, s, t)
        dRect(cX+g,   cY-t/2, s, t)
    end

    local hudFeatures = {
        {S.espMaster,      "ESP Player"},
        {S.espTeams,       "Team Checker"},
        {S.espRadar,       "Minimap Radar"},
        {S.espHealthBar,   "Health Bars"},
        {S.espBoxes,       "Corner Boxes"},
        {S.espSkeleton,    "Skeleton"},
        {S.espPartBoxes,   "Part Dots"},
        {S.watermarkToggle,"Watermark"},
        {S.crosshairToggle,"Crosshair"},
        {S.fpsToggle,      "FPS Counter"},
    }
    local listY = SH-45
    for _,f in ipairs(hudFeatures) do
        if f[1] then
            local tw=#f[2]*7
            pcall(dx9.DrawFilledBox,{15,listY-2},{15+tw+14,listY+17},T.BACKGROUND)
            pcall(dx9.DrawFilledBox,{15,listY-2},{17,       listY+17},T.ACCENT)
            pcall(dx9.DrawString,   {24,listY+1},{0,0,0},    f[2])
            pcall(dx9.DrawString,   {23,listY},  T.TEXT, f[2])
            listY = listY-22
        end
    end

    local curT = os.clock()
    local nY   = 60
    for i=#S.notifications,1,-1 do
        local n=S.notifications[i]
        if curT-n.time > n.dur then
            table.remove(S.notifications,i)
        else
            local tw=#n.text*7
            local bW=math.max(180,tw+24)
            local bH=40
            local bX=SW-bW-20
            pcall(dx9.DrawFilledBox,{bX+2,nY+2},{bX+bW+2,nY+bH+2},{10,10,10})
            pcall(dx9.DrawFilledBox,{bX,  nY},  {bX+bW,  nY+bH},  T.BACKGROUND)
            pcall(dx9.DrawBox,      {bX,  nY},  {bX+bW,  nY+bH},  T.BORDER)
            local pb=math.max(0,1-((curT-n.time)/n.dur))
            pcall(dx9.DrawFilledBox,{bX,   nY},     {bX+3,          nY+bH},T.ACCENT)
            pcall(dx9.DrawFilledBox,{bX+3, nY+bH-2},{bX+3+(bW-3)*pb,nY+bH},T.ACCENT)
            pcall(dx9.DrawString,   {bX+12,nY+12},  T.TEXT, n.text)
            nY=nY+bH+8
        end
    end

    if S.guiOn then
        local mw,mh = 520,550

        if mousePressed and pointInRect(mx,my,S.menuX,S.menuY,mw,30) then
            S.draggingUI = true
            S.dragOffX   = mx-S.menuX
            S.dragOffY   = my-S.menuY
        end
        if S.draggingUI then
            S.menuX = mx-S.dragOffX
            S.menuY = my-S.dragOffY
        end

        local cx,cy = S.menuX, S.menuY

        pcall(dx9.DrawFilledBox,{cx-2,cy-2},{cx+mw+2,cy+mh+2},{5,5,8})
        pcall(dx9.DrawFilledBox,{cx,  cy},  {cx+mw,  cy+mh},  T.BACKGROUND)
        pcall(dx9.DrawBox,      {cx,  cy},  {cx+mw,  cy+mh},  T.BORDER)

        pcall(dx9.DrawFilledBox,{cx,cy},{cx+mw,cy+30},T.TOPBAR)
        pcall(dx9.DrawFilledBox,{cx,cy},{cx+mw,cy+2}, T.ACCENT)
        pcall(dx9.DrawBox,      {cx,cy},{cx+mw,cy+30},T.BORDER)

        local t1w = #CFG.SCRIPT_NAME*7
        pcall(dx9.DrawString,{cx+15,        cy+8}, T.ACCENT,   CFG.SCRIPT_NAME)
        pcall(dx9.DrawString,{cx+15+t1w+5,  cy+8}, T.TEXT_DIM, "v"..CFG.VERSION)
        local aut="Made By Tony Montana"
        pcall(dx9.DrawString,{cx+mw-#aut*7-15, cy+8}, T.TEXT_DIM, aut)

        local tabY = cy+30
        local tabW = mw/#CFG.TABS
        pcall(dx9.DrawFilledBox,{cx,tabY+29},{cx+mw,tabY+30},T.BORDER)

        for i,tab in ipairs(CFG.TABS) do
            local tx  = cx+(i-1)*tabW
            local hov = pointInRect(mx,my,tx,tabY,tabW,30)
            local act = (S.activeTab==i)
            if hov and not act then pcall(dx9.DrawFilledBox,{tx,tabY},{tx+tabW,tabY+29},T.TAB_HOVER) end
            if act             then pcall(dx9.DrawFilledBox,{tx,tabY+28},{tx+tabW,tabY+30},T.ACCENT) end
            if i<#CFG.TABS     then pcall(dx9.DrawFilledBox,{tx+tabW,tabY},{tx+tabW+1,tabY+30},T.BORDER) end
            local tw=#tab*7
            pcall(dx9.DrawString,{tx+(tabW-tw)/2, tabY+8}, act and T.TEXT or T.TEXT_DIM, tab)
            if mousePressed and hov then S.activeTab=i end
        end

        local pX   = cx+20
        local pY   = cy+72
        local rowW = mw-40

        if S.activeTab==1 then
            drawSectionHeader(pX, pY, rowW, "ESP Controls", T)

            local espLabel = S.isScanning
                and ("ESP Player  [Scanning "..math.min(S.scanProgress,100).."%]")
                or "ESP Player"

            local toggles = {
                {espLabel,     "espMaster"},
                {"Team Checker","espTeams"},
                {"Don't Show Team", "espIgnoreTeam"},
                {"Minimap Radar","espRadar"},
                {"Health Bars", "espHealthBar"},
                {"Corner Boxes",    "espBoxes"},
                {"Part Boxes (3D)",  "espPartBoxes"},
                {"Skeleton",         "espSkeleton"},
                {"Nametags",         "espName"},
                {"Distance",         "espDistance"},
                {"Held Weapon",      "espWeapon"},
            }

            for i,tog in ipairs(toggles) do
                local iy = pY+25+(i-1)*26
                if drawToggle(pX, iy, rowW, tog[1], S[tog[2]], T) then
                    S[tog[2]] = not S[tog[2]]
                    if tog[2]=="espMaster" and S.espMaster then
                        S.scanProgress=0; S.isScanning=true; S.lastScanTick=os.clock()
                    end
                    if tog[2]=="espBoxes"    and S.espBoxes    then S.espSkeleton=false end
                    if tog[2]=="espSkeleton" and S.espSkeleton then S.espBoxes=false    end
                    sendNotif(tog[1].." "..(S[tog[2]] and "On" or "Off"))
                end
            end

            local modeY = pY+25+#toggles*26+6
            local curMode = (CFG.TARGET_PARTS[S.univTargetIndex] or {name="?"}).name
            if drawButton(pX, modeY, rowW, "Target Mode: "..curMode, T) then
                S.univTargetIndex = S.univTargetIndex+1
                if S.univTargetIndex>#CFG.TARGET_PARTS then S.univTargetIndex=1 end
                sendNotif("Mode: "..(CFG.TARGET_PARTS[S.univTargetIndex] or {name="?"}).name)
            end

            local scanLabels = {"Lv1  Fast (shallow)", "Lv2  Balanced", "Lv3  Deep (slow)"}
            local depthLabel = scanLabels[S.scanDepth] or "Lv2  Balanced"
            if drawButton(pX, modeY+28, rowW, "Scan Depth: "..depthLabel, T) then
                S.scanDepth = (S.scanDepth % 3) + 1
                S.espCache.lastUpdate = 0
                sendNotif("Scan Depth -> "..scanLabels[S.scanDepth])
            end
            
            local wsModeLabel = CFG.WORKSPACE_MODES[S.workspaceModeIndex] or "Default"
            if drawButton(pX, modeY+56, rowW, "Target: "..wsModeLabel, T) then
                S.workspaceModeIndex = S.workspaceModeIndex + 1
                if S.workspaceModeIndex > #CFG.WORKSPACE_MODES then S.workspaceModeIndex = 1 end
                S.espCache.lastUpdate = 0
                sendNotif("Target -> "..CFG.WORKSPACE_MODES[S.workspaceModeIndex])
            end

            drawSlider(pX, modeY+84, rowW, "ESP Distance", "espMaxDistance", 0, 10000, T, S)

        elseif S.activeTab==2 then
            drawSectionHeader(pX, pY, rowW, "Recoil Assist", T)

            if drawToggle(pX, pY+25, rowW, "Enable No-Recoil", S.recoilEnabled, T) then
                S.recoilEnabled = not S.recoilEnabled
                sendNotif("No-Recoil "..(S.recoilEnabled and "On" or "Off"))
            end

            drawSlider(pX, pY+65, rowW, "Pull Down Strength", "recoilStrengthY", -10, 10, T, S)
            drawSlider(pX, pY+100, rowW, "Horizontal Shake", "recoilStrengthX", 0, 10, T, S)
            drawSlider(pX, pY+135, rowW, "Update Rate", "recoilTickRate", 10, 120, T, S)

            local infoY = pY+175
            pcall(dx9.DrawString, {pX, infoY}, {150,150,150}, "Note: This simulates moving your mouse")
            pcall(dx9.DrawString, {pX, infoY+15}, {150,150,150}, "down/left/right while you hold left click.")

        elseif S.activeTab==3 then
            drawSectionHeader(pX, pY, rowW, "Display", T)

            local dispToggles = {
                {"Watermark",   "watermarkToggle"},
                {"FPS Counter", "fpsToggle"},
                {"Crosshair",   "crosshairToggle"},
            }
            for i,d in ipairs(dispToggles) do
                local iy=pY+25+(i-1)*26
                if drawToggle(pX,iy,rowW,d[1],S[d[2]],T) then
                    S[d[2]]=not S[d[2]]
                    sendNotif(d[1].." "..(S[d[2]] and "On" or "Off"))
                end
            end

            local divY = pY+25+#dispToggles*26+10
            drawSectionHeader(pX, divY, rowW, "Keybinds", T)

            local keyNames = {
                [0x41]="A", [0x42]="B", [0x43]="C", [0x44]="D", [0x45]="E", [0x46]="F",
                [0x47]="G", [0x48]="H", [0x49]="I", [0x4A]="J", [0x4B]="K", [0x4C]="L",
                [0x4D]="M", [0x4E]="N", [0x4F]="O", [0x50]="P", [0x51]="Q", [0x52]="R",
                [0x53]="S", [0x54]="T", [0x55]="U", [0x56]="V", [0x57]="W", [0x58]="X",
                [0x59]="Y", [0x5A]="Z", [0x2D]="Insert", [0x1B]="Escape", [0x20]="Space",
                [0x0D]="Enter", [0x09]="Tab", [0x10]="Shift", [0x11]="Ctrl", [0x12]="Alt"
            }
            local currentKeyName = keyNames[S.menuKeybind] or "0x"..string.format("%X", S.menuKeybind)

            local keybindLabel = S.changingKeybind and "Press any key..." or "Menu Keybind: "..currentKeyName
            
            if drawButton(pX, divY+25, rowW, keybindLabel, T) then
                if not S.changingKeybind then
                    S.changingKeybind = true
                end
            end

            if S.changingKeybind then
                local newKey = nil
                pcall(function() newKey = dx9.GetKey() end)
                if type(newKey)=="string" and newKey~="" then
                    local keyMap = {
                        ["A"]=0x41,["B"]=0x42,["C"]=0x43,["D"]=0x44,["E"]=0x45,["F"]=0x46,
                        ["G"]=0x47,["H"]=0x48,["I"]=0x49,["J"]=0x4A,["K"]=0x4B,["L"]=0x4C,
                        ["M"]=0x4D,["N"]=0x4E,["O"]=0x4F,["P"]=0x50,["Q"]=0x51,["R"]=0x52,
                        ["S"]=0x53,["T"]=0x54,["U"]=0x55,["V"]=0x56,["W"]=0x57,["X"]=0x58,
                        ["Y"]=0x59,["Z"]=0x5A,["INSERT"]=0x2D,["INS"]=0x2D,["ESC"]=0x1B,["ESCAPE"]=0x1B,
                        ["SPACE"]=0x20,["ENTER"]=0x0D,["RETURN"]=0x0D,["TAB"]=0x09,
                        ["SHIFT"]=0x10,["CTRL"]=0x11,["CONTROL"]=0x11,["ALT"]=0x12
                    }
                    local lk = string.upper(newKey):gsub("%[",""):gsub("%]",""):gsub("%s","")
                    if keyMap[lk] then
                        S.menuKeybind = keyMap[lk]
                        S.changingKeybind = false
                        sendNotif("Keybind set to "..lk)
                    end
                end
            end

            local div2 = divY+65
            drawSectionHeader(pX, div2, rowW, "Crosshair", T)
            drawSlider(pX, div2+25, rowW, "Length", "crosshairLength", 1, 50, T, S)
            drawSlider(pX, div2+60, rowW, "Width",  "crosshairWidth",  1, 10, T, S)

            local div3 = div2+100
            drawSectionHeader(pX, div3, rowW, "Accent Color", T)
            drawSlider(pX, div3+25,  rowW, "Red",   "uiR", 0, 255, T, S)
            drawSlider(pX, div3+60,  rowW, "Green", "uiG", 0, 255, T, S)
            drawSlider(pX, div3+95,  rowW, "Blue",  "uiB", 0, 255, T, S)

            local div4 = div3+135
            if drawButton(pX, div4, rowW, "[ Force Rescan ]", T) then
                if S.espMaster or S.espTeams then
                    S.scanProgress=0; S.isScanning=true; S.lastScanTick=os.clock()
                    sendNotif("Rescanning...", 2)
                else
                    sendNotif("Enable ESP Player first", 2)
                end
            end

        elseif S.activeTab==4 then
            local credits = {
                {CFG.SCRIPT_NAME.." "..CFG.VERSION, T.ACCENT},
                {"design and tuned with ai help",    T.TEXT},
                {"UI Built with DXForge",            T.TEXT},
                {"Made By Tony Montana",             T.TEXT_DIM},
            }
            for i,l in ipairs(credits) do
                pcall(dx9.DrawString,{pX, pY+(i-1)*22}, l[2], l[1])
            end
        end
    end
end
