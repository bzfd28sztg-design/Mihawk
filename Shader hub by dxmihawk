--[[
    SHADERS HUB  -  by dxmihawk
    Follow dxmihawk on TikTok: https://www.tiktok.com/@dxmihawk

    Mobile-friendly | Black & White | Galaxy / Black Hole / Glowing Stars
    Roblox LocalScript (StarterPlayerScripts) or run client-side.

    KEY SYSTEM: asks for the key the first time only. The key is saved
    (writefile) so next time the hub opens directly.

    Features:
      * Floating draggable star button + diagonal slice open/close animation
      * 25+ shader presets + live sliders (touch friendly)
      * Real 3D rain in the map (amount + speed sliders), fully client-side
      * Master ON/OFF, Remove-all button, PC hotkey: RightShift
]]

local Players          = game:GetService("Players")
local Lighting         = game:GetService("Lighting")
local TweenService     = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService       = game:GetService("RunService")

local player = Players.LocalPlayer
if game:GetService("RunService"):IsServer() or not player then
    warn("[ShadersHub] Must run in a LocalScript (client-side only).")
    return
end

-- Clean up previous run
if _G.ShadersHubCleanup then pcall(_G.ShadersHubCleanup) end

local WHITE, BLACK = Color3.new(1, 1, 1), Color3.new(0, 0, 0)

-- Key system config
local KEY      = "dxmihawk67"
local KEY_FILE = "ShadersHub_dxmihawk_key.txt"
local TIKTOK   = "https://www.tiktok.com/@dxmihawk"

local function copyLink()
    local ok = pcall(function() (setclipboard or toclipboard or set_clipboard)(TIKTOK) end)
    return ok
end

local function startHub()
    local conns, alive = {}, true

    ----------------------------------------------------------------
    -- Helpers
    ----------------------------------------------------------------
    local function new(class, props, parent)
        local o = Instance.new(class)
        for k, v in pairs(props or {}) do o[k] = v end
        if parent then o.Parent = parent end
        return o
    end

    local function tween(obj, time, props, style, dir)
        local t = TweenService:Create(obj, TweenInfo.new(time, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props)
        t:Play()
        return t
    end

    local function connect(signal, fn)
        local c = signal:Connect(fn)
        table.insert(conns, c)
        return c
    end

    local function corner(parent, r)
        return new("UICorner", {CornerRadius = UDim.new(0, r or 10)}, parent)
    end

    local function stroke(parent, thickness, transparency)
        return new("UIStroke", {
            Color = WHITE, Thickness = thickness or 1, Transparency = transparency or 0.5,
            ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
        }, parent)
    end

    local function label(parent, text, size, props)
        local l = new("TextLabel", {
            BackgroundTransparency = 1, Text = text, TextColor3 = WHITE,
            Font = Enum.Font.GothamMedium, TextSize = size or 14,
        }, parent)
        for k, v in pairs(props or {}) do l[k] = v end
        return l
    end

    ----------------------------------------------------------------
    -- Lighting effects
    ----------------------------------------------------------------
    local origClock    = Lighting.ClockTime
    local origExposure = Lighting.ExposureCompensation

    local bloom = new("BloomEffect", {Name = "ShadersHub_Bloom", Intensity = 0.4, Size = 40, Threshold = 0.9}, Lighting)
    local cc    = new("ColorCorrectionEffect", {Name = "ShadersHub_CC"}, Lighting)
    local sun   = new("SunRaysEffect", {Name = "ShadersHub_Rays", Intensity = 0, Spread = 0.8}, Lighting)
    local dof   = new("DepthOfFieldEffect", {Name = "ShadersHub_DOF", FarIntensity = 0, NearIntensity = 0, FocusDistance = 60, InFocusRadius = 40}, Lighting)

    local atm = Lighting:FindFirstChildOfClass("Atmosphere")
    local createdAtm = false
    if not atm then
        atm = new("Atmosphere", {Name = "ShadersHub_Atmosphere", Density = 0, Offset = 0.2, Color = Color3.fromRGB(190, 190, 200)}, Lighting)
        createdAtm = true
    end

    local effects = {bloom, cc, sun, dof}

    ----------------------------------------------------------------
    -- GUI root
    ----------------------------------------------------------------
    local parentGui = player:WaitForChild("PlayerGui")
    local okHui, hui = pcall(function() return gethui and gethui() end)
    if okHui and hui then parentGui = hui end

    local gui = new("ScreenGui", {
        Name = "ShadersHub", ResetOnSpawn = false, IgnoreGuiInset = true,
        DisplayOrder = 999, ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
    }, parentGui)

    ----------------------------------------------------------------
    -- Rain: real 3D rain in the map (client-side particles that follow your camera)
    ----------------------------------------------------------------
    local RAIN_HEIGHT   = 45     -- studs above the camera where drops spawn
    local RAIN_SQUASH   = -0.8   -- streak shape (if drops look wide instead of long, flip the sign)
    local RAIN_SOUND_ID = ""     -- optional looping rain sound, e.g. "rbxassetid://123456"
    local MAX_ALIVE     = 900    -- max rain particles alive at once (lower = faster on phones)

    local rainAmount, rainSpeed, rainOn, shelter = 0, 1, true, 1

    local rainPart = new("Part", {
        Name = "ShadersHubRain3D", Anchored = true, CanCollide = false, CanQuery = false, CanTouch = false,
        CastShadow = false, Transparency = 1, Size = Vector3.new(70, 1, 70), Locked = true,
    }, workspace)
    local rainEmitter = new("ParticleEmitter", {
        Enabled = false, Rate = 0, Lifetime = NumberRange.new(1), Speed = NumberRange.new(70),
        EmissionDirection = Enum.NormalId.Bottom, SpreadAngle = Vector2.new(2, 2),
        Orientation = Enum.ParticleOrientation.VelocityParallel,
        Size = NumberSequence.new(0.35), Squash = NumberSequence.new(RAIN_SQUASH),
        Transparency = NumberSequence.new(0.55), Color = ColorSequence.new(Color3.fromRGB(215, 228, 255)),
        LightEmission = 0.35, LightInfluence = 0.7, Acceleration = Vector3.new(-6, 0, 0),
        LockedToPart = false, Drag = 0,
    }, rainPart)

    local rainSound
    if RAIN_SOUND_ID ~= "" then
        rainSound = new("Sound", {SoundId = RAIN_SOUND_ID, Looped = true, Volume = 0, Playing = true}, rainPart)
    end

    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Exclude
    local rayTimer, shelterGoal = 0, 1

    local function setRainAmount(a) rainAmount = a end

    connect(RunService.RenderStepped, function(dt)
        local cam = workspace.CurrentCamera
        if not cam then return end
        local pos = cam.CFrame.Position
        rainPart.CFrame = CFrame.new(pos.X, pos.Y + RAIN_HEIGHT, pos.Z)

        -- lighter rain when you are under a roof
        rayTimer += dt
        if rayTimer > 0.3 then
            rayTimer = 0
            local char = player.Character
            rayParams.FilterDescendantsInstances = char and {char, rainPart} or {rainPart}
            local hit = workspace:Raycast(pos, Vector3.new(0, RAIN_HEIGHT, 0), rayParams)
            shelterGoal = hit and 0.06 or 1
        end
        shelter += (shelterGoal - shelter) * math.min(1, dt * 4)

        local sp = 70 * rainSpeed
        local life = (RAIN_HEIGHT + 15) / sp
        local on = rainOn and rainAmount > 0.01
        rainEmitter.Enabled = on
        if on then
            rainEmitter.Speed = NumberRange.new(sp * 0.9, sp * 1.1)
            rainEmitter.Lifetime = NumberRange.new(life)
            rainEmitter.Rate = rainAmount * MAX_ALIVE / life * shelter
        end
        if rainSound then
            rainSound.Volume = on and rainAmount * 0.6 * (0.3 + 0.7 * shelter) or 0
        end
    end)

    ----------------------------------------------------------------
    -- Draggable
    ----------------------------------------------------------------
    local function makeDraggable(handle, target)
        local dragging, startInput, startPos = false, nil, nil
        local state = {moved = false}

        connect(handle.InputBegan, function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging, state.moved = true, false
                startInput, startPos = input.Position, target.Position
                local c; c = input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragging = false
                        c:Disconnect()
                    end
                end)
            end
        end)

        connect(UserInputService.InputChanged, function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local d = input.Position - startInput
                if d.Magnitude > 6 then state.moved = true end
                if state.moved then
                    target.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
                end
            end
        end)
        return state
    end

    ----------------------------------------------------------------
    -- Main window (CanvasGroup => whole window fades)
    ----------------------------------------------------------------
    local main = new("CanvasGroup", {
        Name = "Main", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.58),
        Size = UDim2.fromScale(0.62, 0.62), BackgroundColor3 = BLACK, BorderSizePixel = 0,
        Visible = false, GroupTransparency = 1,
    }, gui)
    corner(main, 16)
    new("UISizeConstraint", {MinSize = Vector2.new(230, 190), MaxSize = Vector2.new(320, 390)}, main)
    local mainScale = new("UIScale", {Scale = 0.6}, main)

    -- NOTE: never put a UIGradient directly on a CanvasGroup (it tints every child).
    local bgFrame = new("Frame", {
        Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 0,
    }, main)
    new("UIGradient", {
        Rotation = 90,
        Color = ColorSequence.new(Color3.fromRGB(0, 0, 0), Color3.fromRGB(28, 28, 28)),
    }, bgFrame)

    -- Font-safe sparkle (built from frames, no special characters needed)
    local function sparkle(parent, size, pos, z)
        local holder = new("Frame", {
            BackgroundTransparency = 1, Size = UDim2.fromOffset(size, size),
            AnchorPoint = Vector2.new(0.5, 0.5), Position = pos, ZIndex = z,
        }, parent)
        local parts = {}
        local function line(w, h, rot, t)
            local f = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
                Size = UDim2.fromOffset(w, h), BackgroundColor3 = WHITE, BackgroundTransparency = t,
                BorderSizePixel = 0, Rotation = rot, ZIndex = z,
            }, holder)
            corner(f, 99)
            new("UIStroke", {Color = WHITE, Thickness = 1, Transparency = math.min(1, t + 0.4)}, f)
            table.insert(parts, f)
        end
        local th = math.max(1, size * 0.14)
        line(size, th, 0, 0)
        line(th, size, 0, 0)
        line(size * 0.6, math.max(1, size * 0.1), 45, 0.4)
        line(size * 0.6, math.max(1, size * 0.1), -45, 0.4)
        return holder, parts
    end

    local function fadeParts(parts, time, goal)
        for _, f in ipairs(parts) do
            tween(f, time, {BackgroundTransparency = goal})
            local s = f:FindFirstChildOfClass("UIStroke")
            if s then tween(s, time, {Transparency = goal}) end
        end
    end

    local mainStroke = stroke(main, 2, 0.3)
    tween(mainStroke, 1.8, {Transparency = 0.8}, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
    do -- pulse forever
        local info = TweenInfo.new(1.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
        TweenService:Create(mainStroke, info, {Transparency = 0.85}):Play()
    end

    -- Star field
    local stars = new("Frame", {
        Name = "Stars", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ZIndex = 1, ClipsDescendants = true,
    }, main)

    -- Galaxy + black hole background
    local galaxy = new("Frame", {
        Name = "Galaxy", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.78, 0.3),
        Size = UDim2.fromOffset(240, 240), BackgroundTransparency = 1, ZIndex = 1,
    }, stars)

    -- faint nebula rings that breathe
    for i, r in ipairs({230, 175, 125}) do
        local ring = new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
            Size = UDim2.fromOffset(r, r), BackgroundTransparency = 1, ZIndex = 1,
        }, galaxy)
        corner(ring, 999)
        local st = new("UIStroke", {Color = WHITE, Thickness = 1.5, Transparency = 0.95}, ring)
        TweenService:Create(st, TweenInfo.new(2.5 + i * 0.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true, i * 0.4), {Transparency = 0.75}):Play()
    end

    -- spiral arms (slowly rotating)
    local arms = new("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
        Size = UDim2.fromOffset(240, 240), BackgroundTransparency = 1, ZIndex = 1,
    }, galaxy)
    for arm = 0, 2 do
        for i = 1, 30 do
            local r = 14 + i * 3.4
            local th = i * 0.26 + arm * (math.pi * 2 / 3)
            local d = math.max(1, math.floor(3.2 - i / 11))
            local dot = new("Frame", {
                AnchorPoint = Vector2.new(0.5, 0.5),
                Position = UDim2.fromOffset(120 + math.cos(th) * r + math.random(-3, 3), 120 + math.sin(th) * r + math.random(-3, 3)),
                Size = UDim2.fromOffset(d, d), BackgroundColor3 = WHITE, BorderSizePixel = 0,
                BackgroundTransparency = math.clamp(0.25 + i / 45, 0, 0.92), ZIndex = 1,
            }, arms)
            corner(dot, 99)
            if i < 12 then new("UIStroke", {Color = WHITE, Thickness = 1, Transparency = 0.8}, dot) end
        end
    end
    TweenService:Create(arms, TweenInfo.new(60, Enum.EasingStyle.Linear, Enum.EasingDirection.In, -1), {Rotation = 360}):Play()

    -- accretion disk (two counter-rotating glowing ellipses)
    local function diskRing(w, h, secs, dir, tr)
        local holder = new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
            Size = UDim2.fromOffset(w, w), BackgroundTransparency = 1, ZIndex = 1,
        }, galaxy)
        local e = new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
            Size = UDim2.fromOffset(w, h), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 1,
        }, holder)
        corner(e, 999)
        new("UIGradient", {Transparency = NumberSequence.new({
            NumberSequenceKeypoint.new(0, 0.97), NumberSequenceKeypoint.new(0.5, tr), NumberSequenceKeypoint.new(1, 0.97),
        })}, e)
        new("UIStroke", {Color = WHITE, Thickness = 1.5, Transparency = 0.35}, e)
        TweenService:Create(holder, TweenInfo.new(secs, Enum.EasingStyle.Linear, Enum.EasingDirection.In, -1), {Rotation = 360 * dir}):Play()
    end
    diskRing(120, 22, 16, 1, 0.55)
    diskRing(92, 16, 10, -1, 0.4)

    -- event horizon + glowing photon ring
    local glowRing = new("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
        Size = UDim2.fromOffset(44, 44), BackgroundTransparency = 1, ZIndex = 1,
    }, galaxy)
    corner(glowRing, 999)
    local gst = new("UIStroke", {Color = WHITE, Thickness = 2, Transparency = 0.2}, glowRing)
    TweenService:Create(gst, TweenInfo.new(1.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {Transparency = 0.8, Thickness = 5}):Play()
    local horizon = new("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
        Size = UDim2.fromOffset(34, 34), BackgroundColor3 = BLACK, BorderSizePixel = 0, ZIndex = 1,
    }, galaxy)
    corner(horizon, 999)

    local function twinkle(parts)
        local info = TweenInfo.new(1 + math.random() * 2.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true, math.random() * 2)
        for _, f in ipairs(parts) do
            f.BackgroundTransparency = 0.2
            TweenService:Create(f, info, {BackgroundTransparency = 0.95}):Play()
            local s = f:FindFirstChildOfClass("UIStroke")
            if s then TweenService:Create(s, info, {Transparency = 1}):Play() end
        end
    end

    for _ = 1, 22 do
        local _, parts = sparkle(stars, math.random(7, 15), UDim2.fromScale(0.02 + math.random() * 0.96, 0.02 + math.random() * 0.96), 1)
        twinkle(parts)
    end
    for _ = 1, 24 do
        local d = math.random(2, 3)
        local dot = new("Frame", {
            Size = UDim2.fromOffset(d, d), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 1,
            Position = UDim2.fromScale(math.random(), math.random()),
        }, stars)
        corner(dot, 99)
        local st = new("UIStroke", {Color = WHITE, Thickness = 1.5, Transparency = 0.6}, dot)
        local info = TweenInfo.new(0.8 + math.random() * 2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true, math.random() * 2)
        TweenService:Create(dot, info, {BackgroundTransparency = 0.95}):Play()
        TweenService:Create(st, info, {Transparency = 1}):Play()
    end

    -- Shooting stars
    task.spawn(function()
        while alive do
            task.wait(2.5 + math.random() * 4)
            if alive and main.Visible and main.GroupTransparency < 0.5 then
                local y = math.random() * 0.5
                local ss = new("Frame", {
                    Size = UDim2.fromOffset(60, 2), BackgroundColor3 = WHITE, BorderSizePixel = 0,
                    Rotation = 25, AnchorPoint = Vector2.new(0.5, 0.5),
                    Position = UDim2.fromScale(-0.1, y), ZIndex = 1,
                }, stars)
                new("UIGradient", {Transparency = NumberSequence.new({
                    NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0),
                })}, ss)
                local tw = tween(ss, 0.9, {Position = UDim2.fromScale(1.1, y + 0.4)}, Enum.EasingStyle.Quad, Enum.EasingDirection.In)
                tw.Completed:Connect(function() ss:Destroy() end)
            end
        end
    end)

    -- Header
    local header = new("Frame", {Size = UDim2.new(1, 0, 0, 40), BackgroundTransparency = 1, ZIndex = 3}, main)
    label(header, "SHADERS HUB", 16, {
        Font = Enum.Font.GothamBold, Size = UDim2.new(1, -56, 0, 22),
        Position = UDim2.fromOffset(14, 3), TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 3,
    })
    label(header, "by dxmihawk", 11, {
        Size = UDim2.new(1, -56, 0, 13), Position = UDim2.fromOffset(15, 23),
        TextXAlignment = Enum.TextXAlignment.Left, TextColor3 = Color3.fromRGB(160, 160, 160), ZIndex = 3,
    })
    new("UIStroke", {Color = WHITE, Thickness = 1, Transparency = 0.8, ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual}, header:FindFirstChildOfClass("TextLabel"))

    local closeBtn = new("TextButton", {
        Text = "X", TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 16,
        Size = UDim2.fromOffset(28, 28), Position = UDim2.new(1, -36, 0, 6),
        BackgroundColor3 = Color3.fromRGB(14, 14, 14), AutoButtonColor = false, ZIndex = 4,
    }, header)
    corner(closeBtn, 14); stroke(closeBtn, 1, 0.4)

    new("Frame", {
        Size = UDim2.new(1, -24, 0, 1), Position = UDim2.new(0, 12, 0, 40),
        BackgroundColor3 = WHITE, BackgroundTransparency = 0.8, BorderSizePixel = 0, ZIndex = 3,
    }, main)

    makeDraggable(header, main)

    -- Scrolling content
    local scroll = new("ScrollingFrame", {
        Position = UDim2.fromOffset(0, 44), Size = UDim2.new(1, 0, 1, -44),
        BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3,
        ScrollBarImageColor3 = WHITE, ScrollBarImageTransparency = 0.4,
        CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y,
        ScrollingDirection = Enum.ScrollingDirection.Y, ZIndex = 2,
    }, main)
    new("UIListLayout", {Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder}, scroll)
    new("UIPadding", {PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14), PaddingTop = UDim.new(0, 8), PaddingBottom = UDim.new(0, 16)}, scroll)

    local order = 0
    local function nextOrder() order += 1 return order end

    local function section(text)
        label(scroll, "—  " .. text .. "  —", 12, {
            Size = UDim2.new(1, 0, 0, 20), TextColor3 = Color3.fromRGB(170, 170, 170),
            Font = Enum.Font.GothamBold, LayoutOrder = nextOrder(), ZIndex = 2,
        })
    end

    ----------------------------------------------------------------
    -- Slider system (touch + mouse)
    ----------------------------------------------------------------
    local activeUpdate = nil
    connect(UserInputService.InputChanged, function(input)
        if activeUpdate and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            activeUpdate(input.Position.X)
        end
    end)
    connect(UserInputService.InputEnded, function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            activeUpdate = nil
            scroll.ScrollingEnabled = true
        end
    end)

    local sliders = {}

    local function slider(key, text, min, max, default, decimals, callback)
        local holder = new("Frame", {Size = UDim2.new(1, 0, 0, 48), BackgroundTransparency = 1, LayoutOrder = nextOrder(), ZIndex = 2}, scroll)
        label(holder, text, 13, {Size = UDim2.new(0.65, 0, 0, 18), TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 2})
        local valLbl = label(holder, "", 13, {
            Size = UDim2.new(0.35, 0, 0, 18), Position = UDim2.fromScale(0.65, 0),
            TextXAlignment = Enum.TextXAlignment.Right, TextColor3 = Color3.fromRGB(190, 190, 190), ZIndex = 2,
        })

        local hit = new("TextButton", {
            Text = "", AutoButtonColor = false, BackgroundTransparency = 1,
            Position = UDim2.fromOffset(0, 20), Size = UDim2.new(1, 0, 0, 28), ZIndex = 2,
        }, holder)
        local track = new("Frame", {
            AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.fromScale(0, 0.5),
            Size = UDim2.new(1, 0, 0, 6), BackgroundColor3 = Color3.fromRGB(45, 45, 45), BorderSizePixel = 0, ZIndex = 2,
        }, hit)
        corner(track, 3)
        local fill = new("Frame", {Size = UDim2.fromScale(0, 1), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 2}, track)
        corner(fill, 3)
        local knob = new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(18, 18),
            Position = UDim2.fromScale(0, 0.5), BackgroundColor3 = WHITE, ZIndex = 3,
        }, track)
        corner(knob, 9)
        new("UIStroke", {Color = WHITE, Thickness = 3, Transparency = 0.75}, knob)

        local api = {}
        function api.set(v)
            v = math.clamp(v, min, max)
            local mult = 10 ^ decimals
            v = math.floor(v * mult + 0.5) / mult
            local a = (v - min) / (max - min)
            fill.Size = UDim2.fromScale(a, 1)
            knob.Position = UDim2.fromScale(a, 0.5)
            valLbl.Text = string.format("%." .. decimals .. "f", v)
            callback(v)
        end

        local function fromX(x)
            local a = math.clamp((x - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
            api.set(min + (max - min) * a)
        end

        connect(hit.InputBegan, function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                scroll.ScrollingEnabled = false
                activeUpdate = fromX
                fromX(input.Position.X)
                tween(knob, 0.15, {Size = UDim2.fromOffset(24, 24)}, Enum.EasingStyle.Back)
            end
        end)
        connect(hit.InputEnded, function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                tween(knob, 0.2, {Size = UDim2.fromOffset(18, 18)}, Enum.EasingStyle.Back)
            end
        end)

        api.set(default)
        sliders[key] = api
        return api
    end

    ----------------------------------------------------------------
    -- Toggle
    ----------------------------------------------------------------
    local function toggle(text, default, callback)
        local row = new("Frame", {Size = UDim2.new(1, 0, 0, 36), BackgroundTransparency = 1, LayoutOrder = nextOrder(), ZIndex = 2}, scroll)
        label(row, text, 14, {Size = UDim2.new(1, -60, 1, 0), TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 2})
        local pill = new("TextButton", {
            Text = "", AutoButtonColor = false, AnchorPoint = Vector2.new(1, 0.5),
            Position = UDim2.fromScale(1, 0.5), Size = UDim2.fromOffset(52, 26),
            BackgroundColor3 = Color3.fromRGB(40, 40, 40), ZIndex = 2,
        }, row)
        corner(pill, 13); stroke(pill, 1, 0.4)
        local dot = new("Frame", {
            AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 3, 0.5, 0),
            Size = UDim2.fromOffset(20, 20), BackgroundColor3 = WHITE, ZIndex = 3,
        }, pill)
        corner(dot, 10)

        local state = default
        local function render()
            tween(dot, 0.2, {Position = state and UDim2.new(1, -23, 0.5, 0) or UDim2.new(0, 3, 0.5, 0)}, Enum.EasingStyle.Back)
            tween(pill, 0.2, {BackgroundColor3 = state and Color3.fromRGB(235, 235, 235) or Color3.fromRGB(40, 40, 40)})
            tween(dot, 0.2, {BackgroundColor3 = state and BLACK or WHITE})
            callback(state)
        end
        connect(pill.Activated, function() state = not state render() end)
        render()
    end

    ----------------------------------------------------------------
    -- Presets
    ----------------------------------------------------------------
    local base = {
        bloom = 0.4, bright = 0, contrast = 0, sat = 0, exposure = origExposure,
        dof = 0, rays = 0, fog = 0, clock = origClock, rain = 0, rainSpeed = 1,
        tint = WHITE, fogColor = Color3.fromRGB(190, 190, 200),
    }

    local presets = {
        {"Default",   {}},
        {"Cinematic", {bloom = 0.7, contrast = 0.2, sat = -0.1, exposure = 0.1, fog = 0.25, dof = 0.2, tint = Color3.fromRGB(255, 240, 225)}},
        {"Noir",      {bloom = 0.5, contrast = 0.45, sat = -1, bright = -0.03, fog = 0.1}},
        {"Dreamy",    {bloom = 1.8, bright = 0.05, contrast = -0.05, sat = 0.25, rays = 0.25, fog = 0.25, tint = Color3.fromRGB(255, 230, 245), fogColor = Color3.fromRGB(255, 210, 235)}},
        {"Neon",      {bloom = 2.2, contrast = 0.3, sat = 0.6, bright = -0.05, tint = Color3.fromRGB(200, 220, 255)}},
        {"Sunset",    {bloom = 1.0, clock = 17.6, rays = 0.4, fog = 0.3, sat = 0.2, tint = Color3.fromRGB(255, 220, 190), fogColor = Color3.fromRGB(255, 170, 120)}},
        {"Space",     {bloom = 1.5, clock = 0, contrast = 0.3, sat = 0.2, exposure = 0.2, tint = Color3.fromRGB(190, 200, 255)}},
        {"Frost",     {bloom = 1.0, clock = 12, bright = 0.05, contrast = 0.15, sat = -0.3, fog = 0.3, tint = Color3.fromRGB(215, 235, 255), fogColor = Color3.fromRGB(235, 245, 255)}},
        {"Vivid",     {bloom = 0.8, contrast = 0.25, sat = 0.8, rays = 0.15}},

        -- Anime / B&W / Quality
        {"Anime",       {bloom = 1.2, bright = 0.06, contrast = 0.15, sat = 0.9, rays = 0.2, fog = 0.1, clock = 14, tint = Color3.fromRGB(255, 245, 250)}},
        {"Black&White", {bloom = 0.3, contrast = 0.25, sat = -1}},
        {"4K Ultra",    {bloom = 0.6, contrast = 0.3, sat = 0.25, exposure = 0.05, dof = 0.1, rays = 0.15, fog = 0.12, quality = 21}},
        {"Low Quality", {bloom = 0, contrast = -0.1, sat = -0.15, bright = -0.02, quality = 1}},

        -- Popular styles
        {"RTX",         {bloom = 0.9, contrast = 0.25, sat = 0.2, exposure = 0.15, rays = 0.3, fog = 0.2}},
        {"Realistic",   {bloom = 0.5, contrast = 0.15, sat = 0.05, dof = 0.15, fog = 0.18, tint = Color3.fromRGB(255, 250, 240)}},
        {"BSL",         {bloom = 1.1, contrast = 0.2, sat = 0.35, clock = 15, rays = 0.45, fog = 0.25, tint = Color3.fromRGB(255, 240, 220)}},
        {"Sildurs",     {bloom = 1.3, bright = 0.03, contrast = 0.15, sat = 0.5, rays = 0.35, fog = 0.2}},
        {"Cyberpunk",   {bloom = 2.0, contrast = 0.4, sat = 0.5, clock = 0, fog = 0.3, tint = Color3.fromRGB(230, 170, 255), fogColor = Color3.fromRGB(120, 60, 200)}},
        {"Vaporwave",   {bloom = 1.6, sat = 0.7, clock = 18, fog = 0.25, tint = Color3.fromRGB(255, 180, 240), fogColor = Color3.fromRGB(255, 150, 220)}},
        {"Horror",      {bloom = 0.2, bright = -0.1, contrast = 0.5, sat = -0.6, exposure = -0.5, clock = 0, fog = 0.5, tint = Color3.fromRGB(200, 220, 200), fogColor = Color3.fromRGB(30, 40, 30)}},
        {"Retro",       {bloom = 0.3, bright = 0.04, contrast = -0.1, sat = -0.2, tint = Color3.fromRGB(255, 235, 200)}},
        {"Golden Hour", {bloom = 1.2, clock = 17, rays = 0.5, fog = 0.25, tint = Color3.fromRGB(255, 225, 180), fogColor = Color3.fromRGB(255, 200, 140)}},
        {"Rainy",       {bloom = 0.5, bright = -0.05, contrast = 0.15, sat = -0.2, fog = 0.35, rain = 0.7, rainSpeed = 1.3, tint = Color3.fromRGB(200, 215, 235), fogColor = Color3.fromRGB(140, 150, 165)}},
        {"Storm",       {bloom = 0.3, bright = -0.1, contrast = 0.3, sat = -0.5, exposure = -0.3, fog = 0.5, rain = 1, rainSpeed = 2.4, clock = 20, tint = Color3.fromRGB(180, 195, 215), fogColor = Color3.fromRGB(70, 80, 95)}},
        {"Night",       {bloom = 0.8, bright = -0.05, contrast = 0.2, clock = 0, tint = Color3.fromRGB(170, 190, 255)}},
    }

    local function applyPreset(p)
        pcall(function()
            settings().Rendering.QualityLevel = p.quality or Enum.QualityLevel.Automatic
        end)
        for key, def in pairs(base) do
            local v = p[key]
            if v == nil then v = def end
            if key == "tint" then
                tween(cc, 0.4, {TintColor = v})
            elseif key == "fogColor" then
                tween(atm, 0.4, {Color = v})
            elseif sliders[key] then
                sliders[key].set(v)
            end
        end
    end

    ----------------------------------------------------------------
    -- Build the content
    ----------------------------------------------------------------
    local fogValue, masterOn = 0, true

    section("MASTER")
    toggle("Shaders Enabled", true, function(on)
        for _, e in ipairs(effects) do e.Enabled = on end
        masterOn = on
        rainOn = on
        atm.Density = on and fogValue or 0
    end)

    section("PRESETS")

    local removeBtn = new("TextButton", {
        Text = "REMOVE ALL SHADERS", TextColor3 = BLACK, Font = Enum.Font.GothamBold, TextSize = 14,
        Size = UDim2.new(1, 0, 0, 40), BackgroundColor3 = WHITE, AutoButtonColor = false,
        LayoutOrder = nextOrder(), ZIndex = 2,
    }, scroll)
    corner(removeBtn, 10)
    connect(removeBtn.Activated, function()
        applyPreset({bloom = 0})          -- everything back to neutral, bloom off
        tween(removeBtn, 0.1, {BackgroundColor3 = Color3.fromRGB(120, 120, 120)})
        task.delay(0.15, function() tween(removeBtn, 0.35, {BackgroundColor3 = WHITE}) end)
    end)

    local rows = math.ceil(#presets / 3)
    local grid = new("Frame", {Size = UDim2.new(1, 0, 0, rows * 36 + (rows - 1) * 8), BackgroundTransparency = 1, LayoutOrder = nextOrder(), ZIndex = 2}, scroll)
    new("UIGridLayout", {
        CellSize = UDim2.new(1 / 3, -6, 0, 36), CellPadding = UDim2.fromOffset(8, 8),
        SortOrder = Enum.SortOrder.LayoutOrder,
    }, grid)

    for i, data in ipairs(presets) do
        local name, vals = data[1], data[2]
        local b = new("TextButton", {
            Text = name, TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 12,
            BackgroundColor3 = Color3.fromRGB(12, 12, 12), AutoButtonColor = false,
            LayoutOrder = i, ZIndex = 2,
        }, grid)
        corner(b, 8)
        local st = stroke(b, 1, 0.5)
        connect(b.Activated, function()
            applyPreset(vals)
            b.BackgroundColor3, b.TextColor3 = WHITE, BLACK
            tween(b, 0.35, {BackgroundColor3 = Color3.fromRGB(12, 12, 12), TextColor3 = WHITE})
            st.Transparency = 0
            tween(st, 0.5, {Transparency = 0.5})
        end)
    end

    section("ADJUST")
    slider("bloom",    "Bloom Glow",   0,    3,   base.bloom,    2, function(v) bloom.Intensity = v end)
    slider("bright",   "Brightness",  -0.3,  0.3, base.bright,   2, function(v) cc.Brightness = v end)
    slider("contrast", "Contrast",    -0.5,  1,   base.contrast, 2, function(v) cc.Contrast = v end)
    slider("sat",      "Saturation",  -1,    1,   base.sat,      2, function(v) cc.Saturation = v end)
    slider("exposure", "Exposure",    -2,    2,   base.exposure, 2, function(v) Lighting.ExposureCompensation = v end)
    slider("rays",     "Sun Rays",     0,    1,   base.rays,     2, function(v) sun.Intensity = v end)
    slider("dof",      "Depth Blur",   0,    1,   base.dof,      2, function(v) dof.FarIntensity = v end)
    slider("fog",      "Atmosphere",   0,    0.6, base.fog,      2, function(v)
        fogValue = v
        atm.Density = masterOn and v or 0
        atm.Haze = math.clamp(v * 5, 0, 10)
    end)
    slider("clock",    "Time of Day",  0,    24,  base.clock,    1, function(v) Lighting.ClockTime = v end)

    section("RAIN")
    slider("rain",      "Rain Amount",              0,   1, base.rain,      2, setRainAmount)
    slider("rainSpeed", "Rain Speed  (slow - fast)", 0.2, 3, base.rainSpeed, 1, function(v) rainSpeed = v end)

    local tiktokBtn = new("TextButton", {
        Text = "FOLLOW @dxmihawk ON TIKTOK", TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 12,
        Size = UDim2.new(1, 0, 0, 36), BackgroundColor3 = Color3.fromRGB(12, 12, 12),
        AutoButtonColor = false, LayoutOrder = nextOrder(), ZIndex = 2,
    }, scroll)
    corner(tiktokBtn, 8); stroke(tiktokBtn, 1, 0.3)
    connect(tiktokBtn.Activated, function()
        tiktokBtn.Text = copyLink() and "LINK COPIED!" or "tiktok.com/@dxmihawk"
        task.delay(1.6, function() tiktokBtn.Text = "FOLLOW @dxmihawk ON TIKTOK" end)
    end)

    label(scroll, "drag the title bar to move", 11, {
        Size = UDim2.new(1, 0, 0, 22), TextColor3 = Color3.fromRGB(120, 120, 120),
        LayoutOrder = nextOrder(), ZIndex = 2,
    })

    ----------------------------------------------------------------
    -- Floating star button
    ----------------------------------------------------------------
    local openBtn = new("TextButton", {
        Name = "OpenButton", Text = "", AutoButtonColor = false,
        Size = UDim2.fromOffset(46, 46), Position = UDim2.new(1, -72, 0.3, 0),
        BackgroundColor3 = BLACK, ZIndex = 10,
    }, gui)
    corner(openBtn, 23)
    local btnStroke = stroke(openBtn, 2, 0.2)
    local btnIcon, btnParts = sparkle(openBtn, 24, UDim2.fromScale(0.5, 0.5), 11)
    btnIcon.Active = false
    do
        local info = TweenInfo.new(1.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
        TweenService:Create(btnStroke, info, {Transparency = 0.8, Thickness = 3.5}):Play()
        TweenService:Create(btnIcon, TweenInfo.new(6, Enum.EasingStyle.Linear, Enum.EasingDirection.In, -1), {Rotation = 360}):Play()
    end
    local btnDrag = makeDraggable(openBtn, openBtn)

    local function setBtnColors(bg, fg)
        tween(openBtn, 0.3, {BackgroundColor3 = bg})
        for _, f in ipairs(btnParts) do
            tween(f, 0.3, {BackgroundColor3 = fg})
            local st = f:FindFirstChildOfClass("UIStroke")
            if st then st.Color = fg end
        end
    end

    -- Star burst (positioned relative to the ScreenGui, centered on the button)
    local function burst()
        local c = openBtn.AbsolutePosition + openBtn.AbsoluteSize / 2 - gui.AbsolutePosition
        for i = 1, 10 do
            local ang = (i / 10) * math.pi * 2 + math.random() * 0.3
            local dist = 55 + math.random() * 35
            local holder, parts = sparkle(gui, math.random(10, 18), UDim2.fromOffset(c.X, c.Y), 12)
            tween(holder, 0.7, {
                Position = UDim2.fromOffset(c.X + math.cos(ang) * dist, c.Y + math.sin(ang) * dist),
                Rotation = math.random(-180, 180),
            })
            fadeParts(parts, 0.7, 1)
            task.delay(0.75, function() holder:Destroy() end)
        end
    end

    -- Slice animation (diagonal cut: two black halves split apart / close together)
    local SLICE_ANGLE = -22
    local sliceHolder = new("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
        Size = UDim2.fromOffset(0, 0), BackgroundTransparency = 1, Rotation = SLICE_ANGLE, ZIndex = 20,
    }, main)
    local halfA = new("Frame", {
        AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.fromOffset(0, 0),
        Size = UDim2.fromOffset(700, 900), BackgroundColor3 = BLACK, BorderSizePixel = 0, ZIndex = 20,
    }, sliceHolder)
    local halfB = new("Frame", {
        AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.fromOffset(0, 0),
        Size = UDim2.fromOffset(700, 900), BackgroundColor3 = BLACK, BorderSizePixel = 0, ZIndex = 20,
    }, sliceHolder)
    local seam = new("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(0, 0),
        Size = UDim2.fromOffset(2, 900), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 21,
    }, sliceHolder)
    local seamGlow = new("UIStroke", {Color = WHITE, Thickness = 3, Transparency = 0.5}, seam)

    local function setHalves(offset, time, style, dir)
        tween(halfA, time, {Position = UDim2.fromOffset(-offset, 0)}, style, dir)
        tween(halfB, time, {Position = UDim2.fromOffset(offset, 0)}, style, dir)
    end

    -- A bright slash that cuts across the screen through the window center
    local function slash()
        local l = new("Frame", {
            AnchorPoint = Vector2.new(0.5, 0.5), Position = main.Position, Size = UDim2.fromOffset(3, 0),
            BackgroundColor3 = WHITE, BorderSizePixel = 0, Rotation = SLICE_ANGLE, ZIndex = 15,
        }, gui)
        local g = new("UIStroke", {Color = WHITE, Thickness = 4, Transparency = 0.4}, l)
        tween(l, 0.16, {Size = UDim2.fromOffset(3, 1100)}, Enum.EasingStyle.Quart, Enum.EasingDirection.Out)
        task.delay(0.16, function()
            tween(l, 0.3, {BackgroundTransparency = 1})
            tween(g, 0.3, {Transparency = 1})
            task.delay(0.35, function() l:Destroy() end)
        end)
    end

    -- Open / close
    local isOpen, token = false, 0
    local function setOpen(state)
        if state == isOpen then return end
        isOpen = state
        token += 1
        local my = token
        burst()
        if state then
            -- start fully covered, cut, then split apart
            halfA.Position, halfB.Position = UDim2.fromOffset(0, 0), UDim2.fromOffset(0, 0)
            seam.BackgroundTransparency, seamGlow.Transparency = 0, 0.5
            main.GroupTransparency = 0
            mainScale.Scale = 0.85
            main.Visible = true
            tween(mainScale, 0.35, {Scale = 1}, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
            setBtnColors(WHITE, BLACK)
            slash()
            task.delay(0.12, function()
                if my ~= token then return end
                setHalves(700, 0.55, Enum.EasingStyle.Quart, Enum.EasingDirection.In)
                tween(seam, 0.3, {BackgroundTransparency = 1})
                tween(seamGlow, 0.3, {Transparency = 1})
            end)
        else
            -- close halves, cut, then vanish
            seam.BackgroundTransparency, seamGlow.Transparency = 1, 1
            setHalves(0, 0.35, Enum.EasingStyle.Quart, Enum.EasingDirection.Out)
            setBtnColors(BLACK, WHITE)
            task.delay(0.33, function()
                if my ~= token then return end
                seam.BackgroundTransparency, seamGlow.Transparency = 0, 0.5
                slash()
            end)
            task.delay(0.5, function()
                if my ~= token then return end
                tween(mainScale, 0.25, {Scale = 0.7}, Enum.EasingStyle.Back, Enum.EasingDirection.In)
                tween(main, 0.25, {GroupTransparency = 1})
                task.delay(0.28, function()
                    if my == token and not isOpen then main.Visible = false end
                end)
            end)
        end
    end

    connect(openBtn.Activated, function()
        if btnDrag.moved then return end
        setOpen(not isOpen)
    end)
    connect(closeBtn.Activated, function() setOpen(false) end)
    connect(UserInputService.InputBegan, function(input, gp)
        if not gp and input.KeyCode == Enum.KeyCode.RightShift then setOpen(not isOpen) end
    end)

    ----------------------------------------------------------------
    -- Cleanup (re-run safe)
    ----------------------------------------------------------------
    _G.ShadersHubCleanup = function()
        alive = false
        for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
        for _, e in ipairs(effects) do pcall(function() e:Destroy() end) end
        if createdAtm then pcall(function() atm:Destroy() end) end
        pcall(function() Lighting.ClockTime = origClock Lighting.ExposureCompensation = origExposure end)
        pcall(function() gui:Destroy() end)
        pcall(function() rainPart:Destroy() end)
    end

    -- Welcome: auto-open once
    task.delay(0.4, function() setOpen(true) end)
end -- startHub

----------------------------------------------------------------
-- KEY SYSTEM (asks only the first time; the key is saved)
----------------------------------------------------------------
local function clean(str)
    return (tostring(str or ""):gsub("%s+", "")):lower()
end

local function readSavedKey()
    local ok, res = pcall(function()
        if isfile and readfile and isfile(KEY_FILE) then return readfile(KEY_FILE) end
    end)
    if ok and res then return clean(res) end
    return clean(_G.ShadersHubKey)     -- fallback: remembered for this session
end

local function saveKey(k)
    _G.ShadersHubKey = k
    pcall(function() writefile(KEY_FILE, k) end)
end

local function showKeyGui()
    local function ui(class, props, parent)
        local o = Instance.new(class)
        for k, v in pairs(props or {}) do o[k] = v end
        if parent then o.Parent = parent end
        return o
    end
    local function tw(obj, t, props, style, dir)
        local x = TweenService:Create(obj, TweenInfo.new(t, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props)
        x:Play()
        return x
    end

    local parentGui = player:WaitForChild("PlayerGui")
    local okHui, hui = pcall(function() return gethui and gethui() end)
    if okHui and hui then parentGui = hui end

    local kg = ui("ScreenGui", {
        Name = "ShadersHubKey", ResetOnSpawn = false, IgnoreGuiInset = true,
        DisplayOrder = 1000, ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
    }, parentGui)
    _G.ShadersHubCleanup = function() pcall(function() kg:Destroy() end) end

    local win = ui("CanvasGroup", {
        AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
        Size = UDim2.fromOffset(300, 244), BackgroundColor3 = BLACK, BorderSizePixel = 0, GroupTransparency = 1,
    }, kg)
    ui("UICorner", {CornerRadius = UDim.new(0, 16)}, win)
    local scale = ui("UIScale", {Scale = 0.8}, win)
    local wst = ui("UIStroke", {Color = WHITE, Thickness = 2, Transparency = 0.3}, win)
    TweenService:Create(wst, TweenInfo.new(1.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {Transparency = 0.85}):Play()

    -- twinkling stars
    for _ = 1, 26 do
        local d = math.random(2, 4)
        local dot = ui("Frame", {
            Size = UDim2.fromOffset(d, d), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 1,
            Position = UDim2.fromScale(math.random(), math.random()),
        }, win)
        ui("UICorner", {CornerRadius = UDim.new(1, 0)}, dot)
        local st = ui("UIStroke", {Color = WHITE, Thickness = 1.5, Transparency = 0.6}, dot)
        local info = TweenInfo.new(0.8 + math.random() * 2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true, math.random() * 2)
        TweenService:Create(dot, info, {BackgroundTransparency = 0.95}):Play()
        TweenService:Create(st, info, {Transparency = 1}):Play()
    end

    local function lbl(text, size, y, color, font)
        return ui("TextLabel", {
            BackgroundTransparency = 1, Text = text, TextColor3 = color or WHITE, TextSize = size,
            Font = font or Enum.Font.GothamMedium, Size = UDim2.new(1, -24, 0, size + 4),
            Position = UDim2.new(0, 12, 0, y), ZIndex = 2, TextWrapped = true,
        }, win)
    end
    lbl("SHADERS HUB", 22, 14, WHITE, Enum.Font.GothamBold)
    lbl("by dxmihawk", 12, 42, Color3.fromRGB(160, 160, 160))
    lbl("Follow dxmihawk on TikTok to get the key", 12, 66, Color3.fromRGB(210, 210, 210))

    local box = ui("TextBox", {
        Text = "", PlaceholderText = "Enter key", PlaceholderColor3 = Color3.fromRGB(120, 120, 120),
        TextColor3 = WHITE, Font = Enum.Font.GothamMedium, TextSize = 15, ClearTextOnFocus = false,
        BackgroundColor3 = Color3.fromRGB(14, 14, 14), Size = UDim2.new(1, -40, 0, 38),
        Position = UDim2.new(0, 20, 0, 94), ZIndex = 2,
    }, win)
    ui("UICorner", {CornerRadius = UDim.new(0, 10)}, box)
    ui("UIStroke", {Color = WHITE, Thickness = 1, Transparency = 0.4}, box)

    local status = lbl("", 12, 138, Color3.fromRGB(200, 200, 200))
    status.Size = UDim2.new(1, -24, 0, 16)

    local verify = ui("TextButton", {
        Text = "VERIFY KEY", TextColor3 = BLACK, Font = Enum.Font.GothamBold, TextSize = 14,
        BackgroundColor3 = WHITE, AutoButtonColor = false, Size = UDim2.new(1, -40, 0, 36),
        Position = UDim2.new(0, 20, 0, 160), ZIndex = 2,
    }, win)
    ui("UICorner", {CornerRadius = UDim.new(0, 10)}, verify)

    local tt = ui("TextButton", {
        Text = "COPY TIKTOK LINK", TextColor3 = WHITE, Font = Enum.Font.GothamBold, TextSize = 12,
        BackgroundColor3 = Color3.fromRGB(12, 12, 12), AutoButtonColor = false,
        Size = UDim2.new(1, -40, 0, 30), Position = UDim2.new(0, 20, 0, 204), ZIndex = 2,
    }, win)
    ui("UICorner", {CornerRadius = UDim.new(0, 8)}, tt)
    ui("UIStroke", {Color = WHITE, Thickness = 1, Transparency = 0.4}, tt)

    tw(scale, 0.45, {Scale = 1}, Enum.EasingStyle.Back)
    tw(win, 0.35, {GroupTransparency = 0})

    local busy = false
    local function submit()
        if busy then return end
        if clean(box.Text) == clean(KEY) then
            busy = true
            saveKey(clean(KEY))
            status.Text = "Key accepted - welcome!"
            tw(verify, 0.2, {BackgroundColor3 = Color3.fromRGB(170, 170, 170)})
            task.delay(0.7, function()
                tw(scale, 0.3, {Scale = 1.1}, Enum.EasingStyle.Back, Enum.EasingDirection.In)
                tw(win, 0.3, {GroupTransparency = 1})
                task.delay(0.35, function()
                    kg:Destroy()
                    startHub()
                end)
            end)
        else
            status.Text = "Wrong key. Follow dxmihawk on TikTok to get it."
            busy = true
            task.spawn(function()
                for _, x in ipairs({12, -12, 8, -8, 0}) do
                    tw(win, 0.05, {Position = UDim2.new(0.5, x, 0.5, 0)})
                    task.wait(0.05)
                end
                busy = false
            end)
        end
    end

    verify.Activated:Connect(submit)
    box.FocusLost:Connect(function(enter) if enter then submit() end end)
    tt.Activated:Connect(function()
        status.Text = copyLink() and "TikTok link copied!" or "tiktok.com/@dxmihawk"
    end)
end

if readSavedKey() == clean(KEY) then
    startHub()          -- key already saved: skip the key screen
else
    showKeyGui()
end
