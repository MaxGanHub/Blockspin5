print("([[local BUILD_PROFILE = "Standard"
local env = (getgenv and getgenv()) or _G
if env.MaxGanHubCleanup then
    pcall(env.MaxGanHubCleanup)
end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Stats = game:GetService("Stats")
local UIS = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

local AIM_FOV = 150
local MAX_ESP_DIST = 2000
local ESPConfig = { Enabled=true, Box=true, Name=true, Distance=true, Health=true, Inventory=true }
local WeaponConfig = { Accuracy=true, Amount=1 }
env.MaxGanHubESPConfig = ESPConfig
env.MaxGanHubWeaponConfig = WeaponConfig
local AIM_LOCK_RESPONSE = 42
local AimConfig = {
    Mode = "Aim",
    LockKey = Enum.KeyCode.F,
    FOV = AIM_FOV,
    ShowFOV = false,
    TargetMarkerRadius = 8,
    StandingResponse = AIM_LOCK_RESPONSE,
    VehicleResponse = 60,
}
env.MaxGanHubAimConfig = AimConfig
local ignoredPlayers = type(env.MaxGanHubIgnoredPlayers) == "table" and env.MaxGanHubIgnoredPlayers or {}
env.MaxGanHubIgnoredPlayers = ignoredPlayers
local CharacterConfig = {
    WalkSpeed = true,
    SpeedMultiplier = 1.25,
    HighJump = true,
    JumpVelocity = 35,
    JumpHeight = 10.7,
    AntiKill = true,
    AntiVehicleRagdoll = true,
}
env.MaxGanHubCharacterConfig = CharacterConfig
local MenuConfig = { Key=Enum.KeyCode.G }
env.MaxGanHubMenuConfig=MenuConfig
local SnapConfig = { Key = Enum.KeyCode.V, Depth = 40 }
env.MaxGanHubSnapConfig = SnapConfig
local VehicleConfig = { Flight = false, Underground = false, Speed = 25, HoverHeight = 1.5, Depth = 30,
    BumpAura = false, BumpRadius = 25, BumpSpeed = 150 }
env.MaxGanHubVehicleConfig = VehicleConfig

local AutoHealConfig = { Enabled = false, Threshold = 50, AnimationSpeed = 2 }
env.MaxGanHubAutoHealConfig = AutoHealConfig

local isRunning = true
local Connections = {}
local DrawObjects = {}
local lockedTarget = nil
local cameraLockTarget = nil
local silentLockTarget = nil
local smoothedRotation = nil
local standingScopeMode = nil
local lockCamera = nil
local lockCharacter = nil
local activeLockSeat = nil
local vehicleRotation = nil
local vehicleFrameRotation = nil
local vehicleLockOffset = nil
local statusWindow = nil
local Features
local characterCleanup = nil
local weaponCleanup = nil
local collisionCleanup = nil
local vehicleCleanup = nil
local vehicleGuiCleanup = nil
local nativeGunModule = nil
local weaponOriginals = {}
local driverCameraState = nil
local status = {
    Brand = "MaxGan Hub",
    Build = "windui-combat-20261010-12-" .. BUILD_PROFILE,
    BuildProfile = BUILD_PROFILE,
    SilentAim = false,
    AimLock = "Ready",
    AimLockTarget = nil,
    FPS = nil,
    Ping = nil,
    PingJitter = nil,
    PingState = "Measuring",
    PingSamples = 0,
    Target = nil,
    TargetDistance = nil,
    State = "Starting",
}
env.MaxGanHubStatus = status

local xenoCompatibility = BUILD_PROFILE == "Xeno" or env.MaxGanHubXenoCompatibility == true
do
    local identify = identifyexecutor or getexecutorname
    if type(identify) == "function" then
        local ok, name = pcall(identify)
        if ok and type(name) == "string" then
            xenoCompatibility = xenoCompatibility or string.find(string.lower(name), "xeno", 1, true) ~= nil
        end
    end
end
status.ExecutorCompatibility = xenoCompatibility and "Xeno" or "Standard"
local networkAccessError
if type(getupvalues or (debug and debug.getupvalues)) ~= "function" then
    networkAccessError = "Unsupported: actual closure upvalues unavailable"
elseif type(filtergc) ~= "function" then
    networkAccessError = "Unsupported: active module lookup unavailable"
end
status.VehicleBumpCompatibility = networkAccessError or "Not checked"

local function requireGameModule(module)
    if not xenoCompatibility then return require(module) end
    status.GameModuleCompatibility = "Xeno: native game modules left untouched"
    error("Xeno: native module access disabled to preserve vehicle controls", 0)
end


local function track(conn)
    Connections[#Connections + 1] = conn
    return conn
end

local function resolveActiveNetwork()
    local read = getupvalues or (debug and debug.getupvalues)
    if networkAccessError then return nil, nil, networkAccessError end
    local modules = filtergc("table", { Keys = { "send", "get", "hook", "on_connect" } }, false)
    local best, sequence, score = nil, nil, -1
    for _, net in ipairs(modules) do
        local ok, values = pcall(read, net.get)
        if ok then
            for _, value in pairs(values) do
                if type(value) == "table" and type(rawget(value, "event")) == "number"
                    and type(rawget(value, "func")) == "number" then
                    local activity = value.event + value.func
                    if activity > score then best, sequence, score = net, value, activity end
                    break
                end
            end
        end
    end
    if score <= 0 then return nil, nil, "Active network counters unavailable" end
    return best, sequence
end
local function resolveGameGunModule(net)
    local read = getupvalues or (debug and debug.getupvalues)
    if not net or type(filtergc) ~= "function" or type(read) ~= "function" then return nil end
    local modules = filtergc("table", { Keys = { "class", "equipped_gun", "calculate_bullet_direction" } }, false)
    for _, guns in ipairs(modules) do
        if type(guns.loaded) == "function" then
            local ok, values = pcall(read, guns.loaded)
            if ok then for _, value in pairs(values) do if value == net then return guns end end end
        end
    end
end

local function restoreDriverCamera()
    local saved = driverCameraState
    driverCameraState = nil
    if not saved or not saved.camera.Parent then return end
    if saved.camera.CameraSubject ~= saved.hum then return end
    if saved.hum.Parent and saved.hum.SeatPart == saved.seat and saved.subject and saved.subject.Parent then
        saved.camera.CameraSubject = saved.subject
    elseif saved.hum.Parent then
        saved.camera.CameraSubject = saved.hum
    end
end

env.MaxGanHubCleanup = function()
    isRunning = false
    status.State = "Stopped"
    status.SilentAim = false
    status.AimLock = "Stopped"
    status.AimLockTarget = nil
    status.AimLockEnabled = false
    status.AntiKill = "Stopped"
    status.AntiKillReturnConfirmed = false
    status.Snap = "Stopped"
    status.PickupRecovery = "Stopped"
    cameraLockTarget, smoothedRotation, lockCamera, lockCharacter = nil, nil, nil, nil
    restoreDriverCamera()
    if vehicleCleanup then pcall(vehicleCleanup) end
    if characterCleanup then pcall(characterCleanup) end
    if env.MaxGanHubFeatures then pcall(function() env.MaxGanHubFeatures:RestoreReloads() end) end
    if weaponCleanup then pcall(weaponCleanup) end
    if collisionCleanup then pcall(collisionCleanup) end
    if vehicleGuiCleanup then pcall(vehicleGuiCleanup) end
    status.AntiKill = "Stopped"
    status.AntiKillReturnConfirmed = false
    if statusWindow then pcall(function() statusWindow:Unload() end) end
    pcall(function() RunService:UnbindFromRenderStep("MaxGanHubAimLock") end)
    pcall(function() RunService:UnbindFromRenderStep("MaxGanHubVehicleAimLock") end)
    pcall(function() RunService:UnbindFromRenderStep("MaxGanHubESP") end)
    for _, c in ipairs(Connections) do pcall(function() c:Disconnect() end) end
    for _, o in ipairs(DrawObjects) do pcall(function() o:Remove() end) end
    Connections, DrawObjects = {}, {}
end
if STATE and STATE.onCleanup then
    STATE.onCleanup(env.MaxGanHubCleanup)
end

task.spawn(function()
    local ok, guns = pcall(function()
        if type(filtergc) == "function" then
            local net = resolveActiveNetwork()
            return assert(resolveGameGunModule(net), "Active gun module unavailable")
        end
        return requireGameModule(ReplicatedStorage.Modules.Game.ItemTypes.Gun)
    end)
    if not ok or not guns.class or not guns.class.objects then
        status.WeaponCompatibility = ok and "Gun objects unavailable" or tostring(guns)
        warn("[MaxGan Hub] Weapon accuracy unavailable")
        return
    end
    if not isRunning then return end
    nativeGunModule = guns
    status.WeaponCompatibility = "Ready"
    local originals = weaponOriginals
    weaponCleanup = function()
        for object, values in pairs(originals) do
            pcall(function()
                if object.states.accuracy.get() == values.appliedAccuracy then object.states.accuracy.set(values.accuracy) end
                if object.states.range.get() == 500 then object.states.range.set(values.range) end
            end)
        end
        table.clear(originals)
    end
    while isRunning do
        local char = LocalPlayer.Character
        local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
        if not WeaponConfig.Accuracy then
            weaponCleanup()
        else
        for _, object in pairs(guns.class.objects) do
            local instance = object.instance
            if instance and instance.Parent and object.states and object.states.accuracy and object.states.range
                and ((char and instance:IsDescendantOf(char))
                    or (backpack and instance:IsDescendantOf(backpack))) then
                pcall(function()
                    local accuracy, range = object.states.accuracy.get(), object.states.range.get()
                    if type(accuracy) ~= "number" or type(range) ~= "number" then return end
                    local values = originals[object]
                    if not values then
                        values = { accuracy = accuracy, range = range, appliedAccuracy=accuracy }
                        originals[object] = values
                    else
                        if accuracy ~= values.appliedAccuracy then values.accuracy = accuracy end
                        if range ~= 500 then values.range = range end
                    end
                    local desired=math.clamp(WeaponConfig.Amount,0,1)
                    values.appliedAccuracy=desired
                    if accuracy ~= desired then object.states.accuracy.set(desired) end
                    if range ~= 500 then object.states.range.set(500) end
                end)
            elseif originals[object] then
                local values = originals[object]
                pcall(function()
                    if object.states.accuracy.get() == values.appliedAccuracy then object.states.accuracy.set(values.accuracy) end
                    if object.states.range.get() == 500 then object.states.range.set(values.range) end
                end)
                originals[object] = nil
            end
        end
        end
        task.wait(0.2)
    end
end)

local partCache = {}
local function getParts(player)
    local now = os.clock()
    local c = partCache[player]
    local currentChar = player.Character
    if c and c.char and c.char == currentChar and c.char.Parent
        and c.hum and c.hum.Parent == c.char
        and c.root and c.root.Parent == c.char
        and c.head and c.head.Parent == c.char and now - c.t < 1 then
        return c
    end
    local char = currentChar or Workspace:FindFirstChild(player.Name)
    if not char then
        partCache[player] = nil
        return nil
    end
    c = {
        char = char,
        hum = char:FindFirstChildOfClass("Humanoid"),
        root = char:FindFirstChild("HumanoidRootPart"),
        head = char:FindFirstChild("Head"),
        t = now,
    }
    partCache[player] = c
    return c
end
track(Players.PlayerRemoving:Connect(function(p) partCache[p] = nil end))

local function isValid(player)
    if not player or player == LocalPlayer or not player.Parent then return false end
    local c = getParts(player)
    if not c or not c.hum or not c.root or not c.root.Parent then return false end
    return c.hum.Health > 0
end

local function getAimPart(player)
    local c = getParts(player)
    if not c then return nil end
    if c.head and c.head.Parent then return c.head end
    return nil
end

local function getEquippedGun()
    local char = LocalPlayer.Character
    if not char then return nil end
    if nativeGunModule and nativeGunModule.equipped_gun then
        local ok, gun = pcall(nativeGunModule.equipped_gun.get)
        if ok and typeof(gun) == "Instance" and gun:IsDescendantOf(char) then return gun end
    end
    local tool = char and char:FindFirstChildOfClass("Tool")
    if tool and tool:GetAttribute("AmmoType") then return tool end
    for _, item in ipairs(char:GetChildren()) do
        if item:IsA("Model") and item:GetAttribute("AmmoType") then return item end
    end
end

local function getAimScreenPoint(cam)
    if getEquippedGun() then return cam.ViewportSize / 2 end
    local gui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
    local screen = gui and gui:FindFirstChild("CrosshairScreenGui")
    local frame = screen and screen:FindFirstChild("CrosshairFrame")
    if screen and screen:IsA("ScreenGui") and screen.Enabled and screen.IgnoreGuiInset
        and frame and frame:IsA("GuiObject") and frame.Visible then
        local screenPoint = frame.AbsolutePosition + frame.AbsoluteSize / 2
        local screenRay = cam:ScreenPointToRay(screenPoint.X, screenPoint.Y)
        local viewportPoint = cam:WorldToViewportPoint(screenRay.Origin + screenRay.Direction * 100)
        return Vector2.new(viewportPoint.X, viewportPoint.Y)
    end
    return cam.ViewportSize / 2
end

local function pickTarget()
    Camera = Workspace.CurrentCamera
    if not Camera then return nil end
    local center = getAimScreenPoint(Camera)
    local best, bestD = nil, math.huge
    for _, player in ipairs(Players:GetPlayers()) do
        if isValid(player) and not player:GetAttribute("SilentAimIgnore") and not ignoredPlayers[player.UserId] then
            local part = getAimPart(player)
            if part then
                local sp, on = Camera:WorldToViewportPoint(part.Position)
                if on then
                    local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                    if d <= AimConfig.FOV and d < bestD then
                        best, bestD = player, d
                    end
                end
            end
        end
    end
    return best
end

local aimLockEnabled = false
local capturingLockKey = false
local capturingSnapKey = false
local toggleSnap = nil
local requestManualReturn = nil
local refreshControls = nil
local function setAimLockEnabled(enabled)
    aimLockEnabled = enabled == true
    status.AimLockEnabled = aimLockEnabled
    status.SilentAim = aimLockEnabled and AimConfig.Mode ~= "Aim"
    if not aimLockEnabled then
        silentLockTarget = nil
        restoreDriverCamera()
        cameraLockTarget, smoothedRotation, lockCamera, lockCharacter = nil, nil, nil, nil
        vehicleRotation, vehicleFrameRotation = nil, nil
        vehicleLockOffset = nil
        lockedTarget = nil
        status.AimLock, status.AimLockTarget = "Ready", nil
        status.Target, status.TargetDistance = nil, nil
    end
    if refreshControls then refreshControls() end
end
track(UIS.InputBegan:Connect(function(input, processed)
    if not isRunning or UIS:GetFocusedTextBox() then return end
    if capturingLockKey or capturingSnapKey then
        if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
        if input.KeyCode ~= Enum.KeyCode.Escape and input.KeyCode ~= Enum.KeyCode.Unknown then
            if capturingSnapKey then
                if input.KeyCode ~= AimConfig.LockKey and input.KeyCode ~= MenuConfig.Key then SnapConfig.Key = input.KeyCode end
            elseif input.KeyCode ~= SnapConfig.Key and input.KeyCode ~= MenuConfig.Key then
                AimConfig.LockKey = input.KeyCode
            end
        end
        capturingLockKey = false
        capturingSnapKey = false
        if refreshControls then refreshControls() end
        return
    end
    if processed then return end
    if input.KeyCode == SnapConfig.Key then
        if toggleSnap then toggleSnap() end
        return
    end
    if input.KeyCode == AimConfig.LockKey then setAimLockEnabled(not aimLockEnabled) end
end))
status.AimLockEnabled = false

local function isAimLockHeld()
    return AimConfig.Mode ~= "Silent" and aimLockEnabled and not capturingLockKey and not capturingSnapKey and not UIS:GetFocusedTextBox()
end

local function validLockTarget(player)
    return isValid(player) and not player:GetAttribute("SilentAimIgnore") and not ignoredPlayers[player.UserId]
        and getAimPart(player) ~= nil
end

local function targetInFOV(player)
    if not player or not validLockTarget(player) then return false end
    local cam = Workspace.CurrentCamera
    local part = getAimPart(player)
    if not cam or not part then return false end
    local point, visible = cam:WorldToViewportPoint(part.Position)
    return visible and (Vector2.new(point.X, point.Y) - getAimScreenPoint(cam)).Magnitude <= AimConfig.FOV
end
local function selectedTarget()
    if isAimLockHeld() and cameraLockTarget and validLockTarget(cameraLockTarget) then
        return cameraLockTarget
    end
    if aimLockEnabled and AimConfig.Mode == "Silent" then
        if not targetInFOV(silentLockTarget) then silentLockTarget = pickTarget() end
        return silentLockTarget
    end
    return pickTarget()
end

local silentContext = { alive = true }
track({ Disconnect = function() silentContext.alive = false end })
local silentGuns, silentDirections, silentRayUtil
local function redirectSilentShot(args)
    if not isRunning or not aimLockEnabled or AimConfig.Mode == "Aim"
        or capturingLockKey or capturingSnapKey or UIS:GetFocusedTextBox() then return end
    if args[1] ~= "shoot_gun" or typeof(args[3]) ~= "CFrame" or type(args[4]) ~= "table" then return end
    status.SilentAimShotsObserved = (status.SilentAimShotsObserved or 0) + 1
    local target = selectedTarget()
    local part = target and getAimPart(target)
    if not part then status.SilentAimResult = "No target in FOV"; return end
    local origin = args[3].Position
    local delta = part.Position - origin
    if delta.Magnitude < 0.01 or delta.Magnitude > 500 then
        status.SilentAimResult = "Target outside bullet range"; return
    end
    local guns = nativeGunModule or silentGuns
    local gun
    if guns and guns.class and guns.class.objects then
        for _, object in pairs(guns.class.objects) do
            if object.instance == args[2] then gun = object; break end
        end
    end
    if not gun or not gun.states or not silentDirections or not silentRayUtil then
        status.SilentAimResult = "Native shot data unavailable"; return
    end
    if type(args[5]) ~= "number" then
        status.SilentAimResult = "Shot seed unavailable"; return
    end
    local saved = weaponOriginals[gun]
    local accuracy = saved and saved.accuracy or gun.states.accuracy.get()
    local seed = gun.states.seed.get()
    if type(accuracy) ~= "number" or type(seed) ~= "number" or seed <= 0 then
        status.SilentAimResult = "Gun seed unavailable"; return
    end
    local excluded = {}
    if LocalPlayer.Character then excluded[#excluded + 1] = LocalPlayer.Character end
    local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
    local seat = hum and hum.SeatPart
    local vehicle = seat and seat:FindFirstAncestorOfClass("Model")
    if vehicle then excluded[#excluded + 1] = vehicle end
    local pellets, targetHits = {}, 0
    for i in ipairs(args[4]) do
        local direction = silentDirections.direction_for(delta.Unit, accuracy, seed, args[5], i)
        local hits, remaining = {}, 2
        silentRayUtil.all_parts_on_ray(origin, direction * 500, excluded, function(hit)
            hits[#hits + 1] = { Instance = hit.Instance, Position = hit.Position, Normal = hit.Normal }
            if hit.Instance:IsDescendantOf(target.Character) then targetHits = targetHits + 1 end
            if hit.Instance.CanCollide then remaining = remaining - 1 end
            return remaining > 0
        end)
        pellets[i] = hits
    end
    if #pellets == 0 then status.SilentAimResult = "No bullet pellets"; return end
    args[3] = CFrame.lookAt(origin, part.Position)
    args[4] = pellets
    status.SilentAimTarget = target.Name
    status.SilentAimRequests = (status.SilentAimRequests or 0) + 1
    status.SilentAimPredictedHits = targetHits
    status.SilentAimResult = targetHits > 0 and "Shot sent - damage unconfirmed" or "Spread or cover missed target"
end
silentContext.redirect = redirectSilentShot
silentContext.enabled = function() return isRunning and aimLockEnabled and AimConfig.Mode ~= "Aim" end
status.SilentAimCompatibility = "Idle until Silent enabled"
task.spawn(function()
    repeat
        if not isRunning then return end
        task.wait(0.25)
    until silentContext.enabled()

    if networkAccessError then
        status.SilentAimCompatibility = networkAccessError; return
    end
    if type(hookfunction) ~= "function" then
        status.SilentAimCompatibility = "Executor hookfunction unavailable"; return
    end
    if type(filtergc) ~= "function" then
        status.SilentAimCompatibility = "Silent unsupported: filtergc unavailable"; return
    end
    local signatures = {
        net = { "send", "get", "hook", "on_connect" },
        guns = { "class", "equipped_gun", "calculate_bullet_direction" },
        directions = { "direction_for", "derive_seed", "calculate_direction" },
        rayUtil = { "all_parts_on_ray" },
    }
    local cache = env.MaxGanHubSilentModules
    local found = cache and cache.jobId == game.JobId and cache.modules or {}
    if nativeGunModule then found.guns = nativeGunModule end
    local function complete()
        return found.net and found.guns and found.directions and found.rayUtil
    end
    local started = os.clock()
    if not complete() and type(filtergc) == "function" then
        for name, keys in pairs(signatures) do
            if not isRunning then return end
            if not found[name] then
                task.wait()
                local success, value = pcall(filtergc, "table", { Keys = keys }, true)
                if success and type(value) == "table" then found[name] = value end
            end
        end
    end
    env.MaxGanHubSilentModules = { jobId = game.JobId, modules = found }
    status.SilentAimLookupMs = math.floor((os.clock() - started) * 1000)
    local net, sequence = resolveActiveNetwork()
    found.net = net
    silentGuns = resolveGameGunModule(net)
    found.guns = silentGuns
    status.SilentAimNetworkEvent = sequence and sequence.event
    status.SilentAimNetworkFunc = sequence and sequence.func
    silentDirections = found.directions
    silentRayUtil = found.rayUtil
    if not isRunning then return end
    if not net then status.SilentAimCompatibility = "Native network table unavailable"; return end
    if not silentGuns or not silentDirections or not silentRayUtil then
        status.SilentAimCompatibility = "Native shot tables unavailable"; return
    end
    local ok, err = pcall(function()
        local slot = env.MaxGanHubSilentHook
        if slot and slot.target == net.send and slot.revision == 3 then
            slot.context = silentContext
        else
            if slot then slot.context = nil end
            slot = { target = net.send, context = silentContext, revision = 3 }
            local original
            original = hookfunction(net.send, function(...)
                local context = slot.context
                if context and context.alive and select(1, ...) == "shoot_gun"
                    and context.enabled() then
                    local args = table.pack(...)
                    if args[1] == "shoot_gun" then
                        local success, errorMessage = pcall(context.redirect, args)
                        if success then return original(table.unpack(args, 1, args.n)) end
                        status.SilentAimError = tostring(errorMessage)
                    end
                end
                return original(...)
            end)
            slot.original = original
            env.MaxGanHubSilentHook = slot
        end
    end)
    status.SilentAimCompatibility = ok and "Ready" or tostring(err)
end)

local function resetCameraLock(state)
    restoreDriverCamera()
    cameraLockTarget, smoothedRotation, lockCamera, lockCharacter = nil, nil, nil, nil
    vehicleRotation, vehicleFrameRotation = nil, nil
    vehicleLockOffset = nil
    activeLockSeat = nil
    status.AimLockMode = nil
    status.AimLock = state
    status.AimLockTarget = nil
end

local function updateStandingLock(cam, head, dt)
    local position = cam.CFrame.Position
    local crosshair = getAimScreenPoint(cam)
    local ray = cam:ViewportPointToRay(crosshair.X, crosshair.Y)
    local localRay = cam.CFrame:VectorToObjectSpace(ray.Direction)
    local correction = CFrame.lookAt(Vector3.zero, localRay):Inverse()
    local desiredRotation = (CFrame.lookAt(position, head.Position) * correction).Rotation
    local tool = getEquippedGun()
    local sniper = tool ~= nil and tool:GetAttribute("SniperScope") ~= nil
        and tool:GetAttribute("SniperScope") ~= false
    if standingScopeMode ~= sniper then
        standingScopeMode = sniper
        smoothedRotation = nil
    end
    if sniper then
        smoothedRotation = smoothedRotation or desiredRotation:ToObjectSpace(cam.CFrame.Rotation)
        local alpha = 1 - math.exp(-AimConfig.StandingResponse * math.max(dt, 0))
        smoothedRotation = smoothedRotation:Lerp(CFrame.identity, alpha)
        local _, remainingAngle = smoothedRotation:ToAxisAngle()
        if math.abs(remainingAngle) < math.rad(0.025) then smoothedRotation = CFrame.identity end
        cam.CFrame = CFrame.new(position) * desiredRotation * smoothedRotation
        status.AimLockTracking = "Sniper"
        return
    end
    status.AimLockTracking = "Standard"
    smoothedRotation = smoothedRotation or cam.CFrame.Rotation
    local alpha = 1 - math.exp(-AimConfig.StandingResponse * math.max(dt, 0))
    smoothedRotation = smoothedRotation:Lerp(desiredRotation, alpha)
    local _, remainingAngle = smoothedRotation:ToObjectSpace(desiredRotation):ToAxisAngle()
    if math.abs(remainingAngle) < math.rad(0.025) then smoothedRotation = desiredRotation end
    cam.CFrame = CFrame.new(position) * smoothedRotation
end

local function vehicleDesiredRotation(cam, head)
    local position = cam.CFrame.Position
    if getEquippedGun() then
        return CFrame.lookAt(position, head.Position, Vector3.yAxis).Rotation
    end
    local crosshair = getAimScreenPoint(cam)
    local ray = cam:ViewportPointToRay(crosshair.X, crosshair.Y)
    local localRay = cam.CFrame:VectorToObjectSpace(ray.Direction)
    local correction = CFrame.lookAt(Vector3.zero, localRay):Inverse()
    return (CFrame.lookAt(position, head.Position, Vector3.yAxis) * correction).Rotation
end

local function updateVehicleLock(cam, head, dt)
    local position = cam.CFrame.Position
    local desiredRotation = vehicleDesiredRotation(cam, head)
    vehicleLockOffset = vehicleLockOffset or desiredRotation:ToObjectSpace(cam.CFrame.Rotation)
    vehicleLockOffset = vehicleLockOffset:Lerp(CFrame.identity,
        1 - math.exp(-AimConfig.VehicleResponse * math.max(dt, 0)))
    local _, angle = vehicleLockOffset:ToAxisAngle()
    if math.abs(angle) < math.rad(0.025) then vehicleLockOffset = CFrame.identity end
    vehicleRotation = desiredRotation * vehicleLockOffset
    vehicleFrameRotation = vehicleRotation
    cam.CFrame = CFrame.new(position) * vehicleFrameRotation
end

RunService:BindToRenderStep("MaxGanHubAimLock", Enum.RenderPriority.Last.Value + 3, function(dt)
    if not isRunning then return end
    if not isAimLockHeld() then resetCameraLock("Ready"); return end
    local cam = Workspace.CurrentCamera
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not cam or not hum or hum.Health <= 0 then
        resetCameraLock("Waiting for character")
        return
    end
    if cam.CameraType == Enum.CameraType.Scriptable then
        resetCameraLock("Waiting for camera")
        return
    end
    if lockCamera ~= cam or lockCharacter ~= char then
        resetCameraLock("Ready")
        lockCamera, lockCharacter = cam, char
    end
    if cameraLockTarget and not validLockTarget(cameraLockTarget) then
        cameraLockTarget, smoothedRotation = nil, nil
        vehicleRotation, vehicleFrameRotation = nil, nil
        vehicleLockOffset = nil
    end
    local seat = hum.SeatPart
    if seat ~= activeLockSeat then
        restoreDriverCamera()
        activeLockSeat = seat
        smoothedRotation = nil
        vehicleRotation, vehicleFrameRotation = nil, nil
        vehicleLockOffset = nil
    end
    if not cameraLockTarget then cameraLockTarget = pickTarget() end
    if not cameraLockTarget then resetCameraLock("No target"); return end
    local head = getAimPart(cameraLockTarget)
    if not head then resetCameraLock("No target"); return end
    local position = cam.CFrame.Position
    if (head.Position - position).Magnitude < 0.001 then return end

    if seat then
        if seat:IsA("VehicleSeat") and seat.Name == "DriverSeat" then
            if not driverCameraState and cam.CameraSubject == seat then
                driverCameraState = { camera = cam, hum = hum, seat = seat, subject = cam.CameraSubject }
                cam.CameraSubject = hum
                vehicleLockOffset = nil
            end
            status.AimLockMode = "Driver"
        else
            status.AimLockMode = "Passenger"
        end
    else
        updateStandingLock(cam, head, dt)
        status.AimLockMode = "On foot"
    end
    lockedTarget = cameraLockTarget
    status.AimLock = "Tracking"
    status.AimLockTarget = cameraLockTarget.Name
end)

local CrateConfig = { SkipAnimation = false }
env.MaxGanHubCrateConfig = CrateConfig
task.spawn(function()
    local controller
    local nextResolve = 0
    while isRunning do
        if CrateConfig.SkipAnimation then
            if not controller and os.clock() >= nextResolve then
                nextResolve = os.clock() + 5
                local ok, result = pcall(function()
                    return requireGameModule(ReplicatedStorage.Modules.Game.CrateSystem.Crate)
                end)
                if ok and type(result) == "table" and result.spinning and result.skipping
                    and type(result.spinning.get) == "function" and type(result.skipping.get) == "function"
                    and type(result.skipping.set) == "function" then
                    controller = result
                    status.CrateSkip = "Ready"
                    status.CrateSkipError = nil
                else
                    status.CrateSkip = "Unavailable"
                    status.CrateSkipError = ok and "Crate state API unavailable" or tostring(result)
                end
            end
            if controller then
                local ok, err = pcall(function()
                    if controller.spinning.get() and not controller.skipping.get() then
                        controller.skipping.set(true)
                        status.CrateSkip = "Skip requested"
                        status.CrateSkipRequests = (status.CrateSkipRequests or 0) + 1
                    elseif not controller.spinning.get() then
                        status.CrateSkip = "Waiting for crate spin"
                    end
                end)
                if not ok then
                    status.CrateSkip, status.CrateSkipError = "Unavailable", tostring(err)
                    controller = nil
                    nextResolve = os.clock() + 5
                end
            end
        else
            status.CrateSkip = "Off"
        end
        task.wait(CrateConfig.SkipAnimation and 0.05 or 0.25)
    end
    status.CrateSkip = "Stopped"
end)

local downedEscape = nil
local snapState = nil
local snapRecoveryOrigin = nil
local pendingReturn = env.MaxGanHubReturnAnchor
local lastSafeReturn = nil
local handledDownedCharacter = nil
local downedClearedSince = nil
local jumpHumanoid = nil
local jumpSettings = nil


local function currentBody()
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if hum and root and hum.Health > 0 then return char, hum, root end
end

local function isVehicleAttached(hum, root)
    if not hum then return false end
    if hum.SeatPart or hum.Sit or hum:GetState() == Enum.HumanoidStateType.Seated then return true end
    local char = hum.Parent
    root = root or (char and char:FindFirstChild("HumanoidRootPart"))
    local assembly = root and root.AssemblyRootPart
    return assembly ~= nil and char ~= nil and not assembly:IsDescendantOf(char)
end

if xenoCompatibility then
    local elapsed = 0
    local observedSeat = nil
    local hiddenSince = nil
    local speedometer = nil
    local speedLabel = nil
    local savedVisible = nil
    local savedText = nil
    local savedScale = nil
    local active = false
    local function releaseSpeedometer()
        if active and speedometer and speedometer.Parent then
            speedometer.Visible = savedVisible
            if speedLabel and speedLabel.Parent then speedLabel.Text = savedText end
            local scale = speedometer:FindFirstChildOfClass("UIScale")
            if scale and savedScale then scale.Scale = savedScale end
        end
        speedometer, speedLabel = nil, nil
        savedVisible, savedText, savedScale = nil, nil, nil
        hiddenSince, active = nil, false
    end
    vehicleGuiCleanup = releaseSpeedometer
    track(RunService.Heartbeat:Connect(function(dt)
        if not isRunning then return end
        elapsed = elapsed + dt
        if elapsed < 0.1 then return end
        elapsed = 0
        local ok, err = pcall(function()
            local _, hum = currentBody()
            local seat = hum and hum.SeatPart
            if not hum or hum.Health <= 0 or not seat or seat.Name ~= "DriverSeat"
                or not seat:IsA("VehicleSeat") or seat.Occupant ~= hum then
                releaseSpeedometer()
                observedSeat = nil
                status.VehicleSpeedRecovery = "Idle"
                return
            end
            if seat ~= observedSeat then
                releaseSpeedometer()
                observedSeat = seat
            end
            local vehicle = seat.Parent
            local chassis = vehicle and vehicle:FindFirstChild("Chassis")
            if not chassis or not chassis:IsA("BasePart") or vehicle:GetAttribute("DisableControls") then
                releaseSpeedometer()
                return
            end
            local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
            local gui = playerGui and playerGui:FindFirstChild("SpeedometerGui")
            local frame = gui and gui:FindFirstChild("Speedometer")
            local speedFrame = frame and frame:FindFirstChild("SpeedFrame")
            local label = speedFrame and speedFrame:FindFirstChild("SpeedLabel")
            if not gui or not gui.Enabled or not frame or not label or not label:IsA("TextLabel") then
                releaseSpeedometer()
                return
            end
            if frame ~= speedometer then
                releaseSpeedometer()
                speedometer, speedLabel = frame, label
            end
            if not active then
                if frame.Visible then hiddenSince = nil; return end
                hiddenSince = hiddenSince or os.clock()
                if os.clock() - hiddenSince < 0.5 then return end
                savedVisible, savedText = frame.Visible, label.Text
                local scale = frame:FindFirstChildOfClass("UIScale")
                savedScale = scale and scale.Scale
                active = true
            end
            local mph = math.round(chassis.AssemblyLinearVelocity.Magnitude / 5280 * 3600)
            label.Text = tostring(mph)
            frame.Visible = true
            local scale = frame:FindFirstChildOfClass("UIScale")
            if scale then
                scale.Scale = math.min(frame.AbsoluteSize.X, frame.AbsoluteSize.Y) < 500 and 0.8 or 1
            end
            status.VehicleSpeedMPH = mph
            status.VehicleSpeedRecovery = "Native speedometer fallback"
        end)
        if not ok then status.VehicleSpeedRecovery = tostring(err) end
    end))
end

local function downed(hum)
    return hum:GetAttribute("HasBeenDowned") == true or hum:GetAttribute("IsDead") == true
end

local healData, healNetSequence, healGetRemote
local healBusy, healElapsed, healNextAttempt = false, 0, 0
local healInventoryRetryAt = 0
local function findBloodBagTool(char, backpack)
    for _, container in ipairs({ char, backpack }) do
        if container then
            for _, item in ipairs(container:GetChildren()) do
                if item:IsA("Tool") and item.Name == "Blood Bag" then return item end
            end
        end
    end
end
local function resolveHealInventory()
    if healData and healNetSequence and healGetRemote then return true end
    if os.clock() < healInventoryRetryAt then return false end
    healInventoryRetryAt = os.clock() + 30
    if type(filtergc) ~= "function" then return false end
    local readUpvalues = getupvalues or (debug and debug.getupvalues)
    if type(readUpvalues) ~= "function" then return false end
    for _, data in ipairs(filtergc("table", { Keys = { "connected", "failed", "hook", "on_connect", "force_update" } }, false)) do
        local ok, items = pcall(function() return data.items end)
        if ok and type(items) == "table" then healData = data; break end
    end
    local net = filtergc("table", { Keys = { "send", "get", "hook", "on_connect" } }, true)
    if net then
        for _, value in pairs(readUpvalues(net.get)) do
            if type(value) == "table" and type(value.event) == "number" and type(value.func) == "number" then
                healNetSequence = value; break
            end
        end
    end
    local remotes = ReplicatedStorage:FindFirstChild("Remotes")
    healGetRemote = remotes and remotes:FindFirstChild("Get")
    return healData ~= nil and healNetSequence ~= nil and healGetRemote ~= nil
end
local function bloodBagStack()
    local items = healData and healData.items
    local stacks = items and items.consumable and items.consumable["Blood Bag"]
    for _, item in pairs(stacks or {}) do
        if item.location == "hand" and (item.amount or 0) > 0 and item.guid then return item end
    end
end
local function requestAutoHeal()
    if not isRunning or not AutoHealConfig.Enabled or healBusy or (Features and Features.consumeBusy) or os.clock() < healNextAttempt then return end
    local char, hum = currentBody()
    if not hum or hum.Health >= AutoHealConfig.Threshold or hum.Health >= hum.MaxHealth
        or downed(hum) or hum:GetAttribute("Stunned") then return end
    healBusy = true
    status.AutoHealTriggeredAt = os.clock()
    task.spawn(function()
        local attemptCompleted = false
        local spedTracks = {}
        local function restoreHealTracks()
            for animation, speed in pairs(spedTracks) do
                pcall(function() if animation.IsPlaying then animation:AdjustSpeed(speed) end end)
            end
        end
        local trackCleanup = { Disconnect = function()
            if restoreHealTracks then restoreHealTracks() end
        end }
        track(trackCleanup)
        local ok, err = pcall(function()
            local backpack = LocalPlayer:FindFirstChildOfClass("Backpack")
            local tool = findBloodBagTool(char, backpack)
            if not tool then
                if not resolveHealInventory() then status.AutoHeal = "Inventory access unavailable"; return end
                local stack = bloodBagStack()
                if not stack then status.AutoHeal = "No Blood Bag in carried inventory"; return end
                status.AutoHeal = "Equipping Blood Bag"
                healNetSequence.func += 1
                healGetRemote:InvokeServer(healNetSequence.func, "toggle_equip_item", stack.guid)
                local deadline = os.clock() + 4
                repeat
                    if not isRunning or not AutoHealConfig.Enabled or LocalPlayer.Character ~= char then return end
                    tool = findBloodBagTool(char, backpack)
                    if tool then break end
                    task.wait(0.1)
                until os.clock() >= deadline
                if not tool then status.AutoHeal = "Blood Bag equip not confirmed"; return end
            end
            if not isRunning or not AutoHealConfig.Enabled or LocalPlayer.Character ~= char or hum.Health <= 0
                or downed(hum) or hum.Health >= AutoHealConfig.Threshold then return end
            local previous = char:FindFirstChildOfClass("Tool")
            local healthBefore = hum.Health
            hum:EquipTool(tool)
            if tool.Parent ~= char then status.AutoHeal = "Blood Bag not held"; return end
            status.AutoHeal = "Using Blood Bag"
            status.AutoHealStartDelay = os.clock() - status.AutoHealTriggeredAt
            local stackBefore = healData and bloodBagStack()
            local amountBefore = stackBefore and stackBefore.amount
            local useAnimation = tool:FindFirstChild("UseAnimation", true)
            local useId = useAnimation and useAnimation.AnimationId:match("%d+") or "80735327245569"
            local animator = hum:FindFirstChildOfClass("Animator")
            tool:Activate()
            local deadline = os.clock() + 30
            local confirmed, consumed = false, false
            repeat
                task.wait(0.1)
                if not isRunning or LocalPlayer.Character ~= char or hum.Health <= 0 then return end
                if animator and tool.Parent == char then
                    for _, animation in ipairs(animator:GetPlayingAnimationTracks()) do
                        local id = animation.Animation and animation.Animation.AnimationId:match("%d+")
                        if id == useId then
                            if spedTracks[animation] == nil then spedTracks[animation] = animation.Speed end
                            animation:AdjustSpeed(math.clamp(AutoHealConfig.AnimationSpeed, 1, 3))
                        end
                    end
                end
                if hum.Health > healthBefore + 1 then confirmed = true; break end
                if stackBefore and amountBefore then
                    local stackNow = bloodBagStack()
                    if not stackNow or (stackNow.guid == stackBefore.guid and stackNow.amount < amountBefore) then
                        consumed = true
                        status.AutoHeal = "Blood Bag consumed - waiting for health"
                    end
                end
                if consumed then
                    task.wait(1)
                    confirmed = hum.Health > healthBefore + 1
                    break
                end
                if tool.Parent ~= char then break end
            until os.clock() >= deadline
            attemptCompleted = confirmed or consumed
            status.AutoHeal = confirmed and "Health restored" or (consumed and "Bag consumed - health unchanged" or "Healing not confirmed")
            if previous and previous ~= tool and previous.Parent == backpack and tool.Parent == char then
                if confirmed then hum:EquipTool(previous) end
            end
        end)
        restoreHealTracks()
        table.clear(spedTracks)
        local cleanupIndex = table.find(Connections, trackCleanup)
        if cleanupIndex then table.remove(Connections, cleanupIndex) end
        if not ok then status.AutoHeal = tostring(err) end
        healNextAttempt = os.clock() + (attemptCompleted and 0.3 or 3)
        healBusy = false
    end)
end
local healObservedHumanoid, healHealthConnection
track({ Disconnect = function()
    if healHealthConnection then healHealthConnection:Disconnect(); healHealthConnection = nil end
end })
track(RunService.Heartbeat:Connect(function(dt)
    if not isRunning then return end
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum ~= healObservedHumanoid then
        if healHealthConnection then healHealthConnection:Disconnect() end
        healObservedHumanoid = hum
        healHealthConnection = hum and hum.HealthChanged:Connect(requestAutoHeal) or nil
    end
    healElapsed += dt
    if healElapsed < 0.2 then return end
    healElapsed = 0
    requestAutoHeal()
end))


if xenoCompatibility then
    local elapsed = 0
    local detachedSince = nil
    local observedHumanoid = nil
    track(RunService.Heartbeat:Connect(function(dt)
        if not isRunning then return end
        elapsed = elapsed + dt
        if elapsed < 0.25 then return end
        elapsed = 0
        local _, hum, root = currentBody()
        if hum ~= observedHumanoid then
            observedHumanoid = hum
            detachedSince = nil
        end
        if not hum or not root or isVehicleAttached(hum, root) or downed(hum)
            or hum.PlatformStand or hum:GetState() == Enum.HumanoidStateType.Physics
            or hum:GetState() == Enum.HumanoidStateType.Ragdoll then
            detachedSince = nil
            return
        end
        detachedSince = detachedSince or os.clock()
        if os.clock() - detachedSince < 0.5 then return end
        local animator = hum:FindFirstChildOfClass("Animator")
        if not animator then return end
        local ok, err = pcall(function()
            local stopped = 0
            for _, animation in ipairs(animator:GetPlayingAnimationTracks()) do
                if animation.Name == "DriveAnimation" then
                    animation:Stop(0.15)
                    stopped = stopped + 1
                end
            end
            if stopped > 0 then
                status.VehicleExitRecovery = "Stopped stale drive animations"
                status.VehicleExitAnimationStops = (status.VehicleExitAnimationStops or 0) + stopped
            end
        end)
        if not ok then status.VehicleExitRecovery = tostring(err) end
    end))
end

local vehicleState = nil
local function driverVehicle()
    local char, hum = currentBody()
    local seat = hum and hum.SeatPart
    if not seat or not seat:IsA("VehicleSeat") or seat.Occupant ~= hum
        or downed(hum) then return end
    local vehicle = seat:FindFirstAncestorOfClass("Model")
    local chassis
    while vehicle and vehicle ~= Workspace do
        if vehicle:IsA("Model") then
            chassis = vehicle:FindFirstChild("Chassis")
            if not chassis then
                local body = vehicle:FindFirstChild("Body")
                chassis = body and body:FindFirstChild("Base")
            end
            if chassis and chassis:IsA("BasePart") then break end
        end
        vehicle = vehicle.Parent
    end
    if not vehicle or not vehicle:IsA("Model") or not chassis or not chassis:IsA("BasePart") or chassis.Anchored then return end
    local assembly = chassis.AssemblyRootPart
    if not assembly or not assembly:IsDescendantOf(vehicle) or assembly.Anchored then return end
    if vehicle:GetAttribute("DisableControls") then return end
    return vehicle, chassis, seat, char
end

local bumpRemote, bumpSequence
local bumpCooldowns = setmetatable({}, { __mode = "k" })
local bumpElapsed = 0
local bumpResolving = false
local function resolveBumpNetwork()
    if bumpResolving or not isRunning then return end
    bumpResolving = true
    local ok, err = pcall(function()
        local net, sequence, reason = resolveActiveNetwork()
        assert(net and sequence, reason or "Active network module unavailable")
        local remotes = ReplicatedStorage:FindFirstChild("Remotes")
        local remote = remotes and remotes:FindFirstChild("Send")
        assert(remote and remote:IsA("RemoteEvent"), "Vehicle hit channel unavailable")
        if not isRunning then return end
        bumpRemote, bumpSequence = remote, sequence
        status.VehicleBumpNetworkEvent = sequence.event
        status.VehicleBumpCompatibility = "Bound to active network - damage unconfirmed"
        status.VehicleBump = "Ready - server acceptance unconfirmed"
    end)
    bumpResolving = false
    if not ok and isRunning then
        status.VehicleBump = tostring(err)
        status.VehicleBumpCompatibility = tostring(err)
    end
end

track(RunService.Heartbeat:Connect(function(dt)
    if not isRunning or not VehicleConfig.BumpAura then return end
    bumpElapsed += dt
    if bumpElapsed < 0.2 then return end
    bumpElapsed = 0
    if not bumpRemote or not bumpSequence then return end
    local vehicle, chassis, _, char = driverVehicle()
    if not vehicle then status.VehicleBump = "Waiting for driver"; return end
    local now = os.clock()
    local target, distance = nil, VehicleConfig.BumpRadius
    for _, player in ipairs(Players:GetPlayers()) do
        local character = player.Character
        if player ~= LocalPlayer and not ignoredPlayers[player.UserId] and character and character ~= char then
            local hum = character:FindFirstChildOfClass("Humanoid")
            local root = character:FindFirstChild("HumanoidRootPart")
            if hum and root and hum.Health > 0 and not downed(hum)
                and not (hum.SeatPart and hum.SeatPart:IsDescendantOf(vehicle))
                and now >= (bumpCooldowns[character] or 0) then
                local currentDistance = (root.Position - chassis.Position).Magnitude
                if currentDistance <= distance then target, distance = character, currentDistance end
            end
        end
    end
    if not target then status.VehicleBump = "Ready - no eligible target"; return end
    local root = target:FindFirstChild("HumanoidRootPart")
    local direction = root.Position - chassis.Position
    direction = Vector3.new(direction.X, 0, direction.Z)
    if direction.Magnitude < 0.01 then
        direction = Vector3.new(chassis.CFrame.LookVector.X, 0, chassis.CFrame.LookVector.Z)
    end
    if direction.Magnitude < 0.01 then return end
    local impactVelocity = direction.Unit * math.clamp(VehicleConfig.BumpSpeed, 15, 1000)
    bumpCooldowns[target] = now + 2
    local ok, err = pcall(function()
        bumpSequence.event += 1
        bumpRemote:FireServer(bumpSequence.event, "run_over", vehicle, target, impactVelocity)
    end)
    if ok then
        status.VehicleBump = "Hit request sent: " .. target.Name
        status.VehicleBumpSent = (status.VehicleBumpSent or 0) + 1
    else
        status.VehicleBump = "Hit request failed: " .. tostring(err)
    end
end))

local function restoreVehicleCollision(state)
    for part, value in pairs(state.collisions) do
        if part.Parent then pcall(function() part.CanCollide = value end) end
    end
    table.clear(state.collisions)
    if state.savedEvaluateStateMachine ~= nil then
        local hum = state.char:FindFirstChildOfClass("Humanoid")
        if hum and hum.Parent and hum.EvaluateStateMachine == false then
            hum.EvaluateStateMachine = state.savedEvaluateStateMachine
        end
        state.savedEvaluateStateMachine = nil
    end
end

local function stopVehicle(reason, keepBumpArmed)
    local state = vehicleState
    vehicleState = nil
    VehicleConfig.Flight, VehicleConfig.Underground = false, false
    if keepBumpArmed == false then VehicleConfig.BumpAura = false end
    status.VehicleMode, status.VehicleBumpTarget = reason or "Off", nil
    if not state then return end
    for _, connection in ipairs(state.connections) do connection:Disconnect() end
    for _, object in ipairs(state.objects) do object:Destroy() end
    restoreVehicleCollision(state)
end
vehicleCleanup = function() stopVehicle("Stopped", false) end

local function setVehicleCollision(state, enabled)
    if enabled then restoreVehicleCollision(state); return end
    local hum = state.char:FindFirstChildOfClass("Humanoid")
    if hum and hum.SeatPart == state.seat then
        if state.savedEvaluateStateMachine == nil then state.savedEvaluateStateMachine = hum.EvaluateStateMachine end
        hum.EvaluateStateMachine = false
    end
    local function disable(part)
        if part:IsA("BasePart") then
            if state.collisions[part] == nil then state.collisions[part] = part.CanCollide end
            part.CanCollide = false
        end
    end
    for _, container in ipairs({ state.vehicle, state.char }) do
        for _, part in ipairs(container:GetDescendants()) do disable(part) end
    end
end

local function createVehicleState(vehicle, chassis, seat, char)
    local state = { vehicle = vehicle, chassis = chassis, seat = seat, char = char,
        origin = chassis.CFrame, collisions = {}, connections = {}, objects = {}, lastPosition = chassis.Position }
    vehicleState = state
    status.VehicleError = nil
    local function object(class, name)
        local value = Instance.new(class)
        value.Name = name
        state.objects[#state.objects + 1] = value
        return value
    end
    local attachment = object("Attachment", "MaxGanVehicleAttachment")
    attachment.Parent = chassis
    local velocity = object("LinearVelocity", "MaxGanVehicleVelocity")
    velocity.Attachment0 = attachment
    velocity.RelativeTo = Enum.ActuatorRelativeTo.World
    velocity.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
    velocity.ForceLimitsEnabled = true
    velocity.ForceLimitMode = Enum.ForceLimitMode.PerAxis
    velocity.Enabled = false
    velocity.Parent = chassis
    local orientation = object("AlignOrientation", "MaxGanVehicleOrientation")
    orientation.Mode = Enum.OrientationAlignmentMode.OneAttachment
    orientation.Attachment0 = attachment
    orientation.Responsiveness = 15
    orientation.MaxAngularVelocity = 4
    orientation.CFrame = chassis.CFrame.Rotation
    orientation.Enabled = false
    orientation.Parent = chassis
    state.velocity, state.orientation = velocity, orientation
    for _, container in ipairs({ vehicle, char }) do
        state.connections[#state.connections + 1] = container.DescendantAdded:Connect(function(part)
            if vehicleState == state and next(state.collisions) ~= nil and part:IsA("BasePart") then
                state.collisions[part] = part.CanCollide
                part.CanCollide = false
            end
        end)
    end
    return state
end

local function ensureVehicleState()
    local vehicle, chassis, seat, char = driverVehicle()
    if not vehicle then status.VehicleMode = "Sit in the driver seat"; return end
    if vehicleState and vehicleState.seat ~= seat then stopVehicle("Vehicle changed", true) end
    if vehicleState then return vehicleState end
    return createVehicleState(vehicle, chassis, seat, char)
end

local function returnVehicleUp()
    local state = vehicleState or ensureVehicleState()
    if not state then return end
    local chassis = state.chassis
    local position = chassis.Position
    local box, size = state.vehicle:GetBoundingBox()
    local bottomOffset = math.max(position.Y - box.Position.Y + size.Y / 2, size.Y / 2)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    local excluded = { state.vehicle, state.char }
    local vehicles = Workspace:FindFirstChild("Vehicles")
    if vehicles then excluded[#excluded + 1] = vehicles end
    for _, player in ipairs(Players:GetPlayers()) do
        if player.Character then excluded[#excluded + 1] = player.Character end
    end
    params.FilterDescendantsInstances = excluded
    params.RespectCanCollide = true
    local top = math.max(position.Y, 1000) + 512
    local hit = Workspace:Raycast(Vector3.new(position.X, top, position.Z),
        Vector3.new(0, -(top - Workspace.FallenPartsDestroyHeight + 100), 0), params)
    if not hit or hit.Normal.Y < 0.3 then
        status.VehicleMode = "Surface not found - keep holding"
        return
    end
    local targetY = math.max(position.Y, hit.Position.Y + bottomOffset + 2)
    local look = Vector3.new(chassis.CFrame.LookVector.X, 0, chassis.CFrame.LookVector.Z)
    if look.Magnitude < 0.01 then look = Vector3.new(0, 0, -1) end
    state.returnTarget = Vector3.new(position.X, targetY, position.Z)
    state.returnSurfaceY = hit.Position.Y
    setVehicleCollision(state, false)
    state.returnParts = {}
    for _, part in ipairs(state.vehicle:GetDescendants()) do
        if part:IsA("BasePart") then state.returnParts[#state.returnParts + 1] = part end
    end
    for part, collides in pairs(state.collisions) do
        if collides and part:IsDescendantOf(state.char) then state.returnParts[#state.returnParts + 1] = part end
    end
    state.returnCollisionSince = nil
    state.returnProgressY, state.returnProgressAt = position.Y, os.clock()
    state.returnLiftAttempts = 0
    state.returnRotation = CFrame.lookAt(Vector3.zero, look).Rotation
    state.returnDuration = math.max(12, (targetY - position.Y) / math.max(VehicleConfig.Speed, 10) * 2 + 5)
    VehicleConfig.Underground, VehicleConfig.Flight = false, false
    state.returning, state.stableSince = true, nil
    state.returnStarted = os.clock()
    setVehicleCollision(state, false)
    status.VehicleMode = "Returning vehicle"
end

local function setVehicleFlight(value)
    if not isRunning then return end
    if not value then
        VehicleConfig.Flight = false
        if vehicleState and not vehicleState.returning and not VehicleConfig.Underground then
            stopVehicle("Off")
        end
        return
    end
    if vehicleState and (vehicleState.undergroundOrigin or vehicleState.returning) then
        status.VehicleMode = "Return above ground first"; return
    end
    if not ensureVehicleState() then return end
    local state = vehicleState
    local box, size = state.vehicle:GetBoundingBox()
    state.hoverBottomOffset = math.max(state.chassis.Position.Y - box.Position.Y + size.Y / 2, size.Y / 2)
    state.hoverVelocity = Vector3.zero
    state.hoverY = state.chassis.Position.Y
    state.orientation.Responsiveness = 15
    state.orientation.MaxAngularVelocity = 4
    VehicleConfig.Flight = true
end

local function setVehicleUnderground(value)
    if not isRunning then return end
    if not value then
        if VehicleConfig.Underground then returnVehicleUp() end
        return
    end
    local state = ensureVehicleState()
    if not state then return end
    local _, size = state.vehicle:GetBoundingBox()
    state.height = size.Y
    local origin = state.undergroundOrigin or state.chassis.CFrame
    if origin.Position.Y - VehicleConfig.Depth - size.Y < Workspace.FallenPartsDestroyHeight + 10 then
        status.VehicleMode = "Depth below world limit"; return
    end
    state.undergroundOrigin = origin
    state.returning, state.stableSince = false, nil
    VehicleConfig.Underground, VehicleConfig.Flight = true, false
    setVehicleCollision(state, false)
end

local function vehicleMoveInput()
    if UIS:GetFocusedTextBox() or capturingLockKey or capturingSnapKey then return Vector3.zero, 0 end
    local cam = Workspace.CurrentCamera
    if not cam then return Vector3.zero, 0 end
    local look = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
    if look.Magnitude < 0.01 then return Vector3.zero, 0 end
    look = look.Unit
    local right = look:Cross(Vector3.yAxis)
    local forward = (UIS:IsKeyDown(Enum.KeyCode.W) and 1 or 0) - (UIS:IsKeyDown(Enum.KeyCode.S) and 1 or 0)
    local sideways = (UIS:IsKeyDown(Enum.KeyCode.D) and 1 or 0) - (UIS:IsKeyDown(Enum.KeyCode.A) and 1 or 0)
    local direction = look * forward + right * sideways
    if direction.Magnitude > 1 then direction = direction.Unit end
    return direction
end

local function updateLowHover(state, direction, dt)
    local chassis, position = state.chassis, state.chassis.Position
    local now = os.clock()
    if not state.hoverRayParams then
        local params = RaycastParams.new()
        params.FilterType = Enum.RaycastFilterType.Exclude
        params.RespectCanCollide = true
        local excluded = { state.vehicle, state.char }
        local vehicles = Workspace:FindFirstChild("Vehicles")
        if vehicles then excluded[#excluded + 1] = vehicles end
        for _, player in ipairs(Players:GetPlayers()) do
            if player.Character then excluded[#excluded + 1] = player.Character end
        end
        params.FilterDescendantsInstances = excluded
        state.hoverRayParams = params
    end
    if now >= (state.hoverNextRay or 0) then
        state.hoverNextRay = now + 0.06
        local hit = Workspace:Raycast(position + Vector3.new(0, 4, 0),
            Vector3.new(0, -(state.hoverBottomOffset + 20), 0), state.hoverRayParams)
        if hit and hit.Normal.Y > 0.4 then
            state.hoverGroundY = hit.Position.Y
            state.hoverGroundAt = now
        end
    end
    local targetY = state.hoverGroundY and state.hoverGroundY + state.hoverBottomOffset
        + math.clamp(VehicleConfig.HoverHeight, 0.5, 5) or state.hoverY
    state.hoverY += math.clamp(targetY - state.hoverY, -16 * dt, 20 * dt)
    state.hoverVelocity = direction * math.clamp(VehicleConfig.Speed, 5, 300)
    if direction.Magnitude > 0.05 then
        state.orientation.CFrame = CFrame.lookAt(Vector3.zero, direction).Rotation
    end
    status.VehicleHoverHeightError = state.hoverY - position.Y
    status.VehicleMode = now - (state.hoverGroundAt or now) < 0.5
        and "Surface glide - underground controls" or "Surface glide - holding height over gap"
    return state.hoverVelocity + Vector3.new(0, math.clamp((state.hoverY - position.Y) * 8, -16, 20), 0)
end

local function updateVehicleMovement(dt)
    local state = vehicleState
    if not isRunning then return end
    if not state then return end
    local vehicle, chassis, seat = driverVehicle()
    if vehicle ~= state.vehicle or seat ~= state.seat or LocalPlayer.Character ~= state.char then
        stopVehicle(VehicleConfig.BumpAura and "Bump aura waiting for driver" or "Driver control ended", true)
        return
    end
    if not state.velocity.Parent or not state.orientation.Parent then stopVehicle("Vehicle forces removed"); return end
    local now, position = os.clock(), chassis.Position
    local travelled = (position - state.lastPosition).Magnitude
    if travelled > math.max(30, chassis.AssemblyLinearVelocity.Magnitude * math.max(dt, 0.01) * 3 + 15) then
        state.corrections = (state.corrections or 0) + 1
        state.correctionStart = state.correctionStart or now
    end
    state.lastPosition = position
    if state.correctionStart and now - state.correctionStart > 5 then state.corrections, state.correctionStart = 0, nil end
    if (state.corrections or 0) >= 3 then stopVehicle("Repeated position correction"); return end
    local direction = vehicleMoveInput()
    local velocity = direction * VehicleConfig.Speed
    local active = VehicleConfig.Flight or VehicleConfig.Underground or state.returning == true
    local mass = chassis.AssemblyMass
    if mass == math.huge or mass ~= mass then stopVehicle("Vehicle assembly unavailable"); return end
    local force = math.max(mass * math.max(Workspace.Gravity, 50) * 8, 1000)
    state.velocity.MaxAxesForce = Vector3.new(force, active and force or 0, force)
    state.orientation.MaxTorque = force * 15
    state.orientation.Enabled = active
    if direction.Magnitude > 0.05 then state.orientation.CFrame = CFrame.lookAt(Vector3.zero, direction).Rotation end
    if (state.returning and not state.returnCollisionSince) or VehicleConfig.Underground then
        for part in pairs(state.collisions) do
            if part.Parent and part.CanCollide then part.CanCollide = false end
        end
    end
    status.VehicleBumpTarget = nil
    if state.returning then
        local target = state.returnTarget
        if not target then status.VehicleMode = "Return target unavailable"; return end
        local delta = target - position
        velocity = delta.Magnitude > 0.05 and delta.Unit * math.min(VehicleConfig.Speed, delta.Magnitude * 5) or Vector3.zero
        state.orientation.CFrame = state.returnRotation
        status.VehicleMode = "Returning vehicle"
        status.VehicleReturnHeightError = target.Y - position.Y
        if position.Y > state.returnProgressY + 0.5 then
            state.returnProgressY, state.returnProgressAt = position.Y, now
        end
        local clear = true
        local clearance = math.huge
        if delta.Magnitude < 2 then
            for _, part in ipairs(state.returnParts) do
                if part.Parent then
                    local cf, size = part.CFrame, part.Size
                    local halfY = (math.abs(cf.RightVector.Y) * size.X + math.abs(cf.UpVector.Y) * size.Y
                        + math.abs(cf.LookVector.Y) * size.Z) / 2
                    clearance = math.min(clearance, part.Position.Y - halfY - state.returnSurfaceY)
                end
            end
            clear = clearance >= 0.75
            status.VehicleReturnClearance = clearance
            if not clear and not state.returnCollisionSince then
                state.returnTarget += Vector3.new(0, 1.5 - clearance, 0)
                state.stableSince = nil
                return
            end
        end
        if state.returnCollisionSince and (delta.Magnitude > 2 or not clear) then
            setVehicleCollision(state, false)
            state.returnCollisionSince, state.stableSince = nil, nil
            state.returnProgressY, state.returnProgressAt = position.Y, now
            status.VehicleMode = "Return shifted - checking again"
            return
        end
        if delta.Magnitude < 1.5 and clear and math.abs(chassis.AssemblyLinearVelocity.Y) < 5 then
            state.stableSince = state.stableSince or now
            local confirmationTime = math.clamp((status.Ping or 150) / 1000 * 2 + 0.5, 0.75, 2)
            if now - state.stableSince >= confirmationTime then
                if not state.returnCollisionSince then
                    restoreVehicleCollision(state)
                    state.returnCollisionSince = now
                elseif now - state.returnCollisionSince >= math.max(1.5, confirmationTime) then
                    stopVehicle("Vehicle returned"); return
                end
                status.VehicleMode = "Checking vehicle clearance"
            end
        else state.stableSince = nil end
        if now - state.returnStarted > state.returnDuration then
            status.VehicleMode = "Vehicle return unconfirmed - holding"
        end
    elseif VehicleConfig.Underground then
        local targetY = state.undergroundOrigin.Position.Y - VehicleConfig.Depth
        if targetY - state.height < Workspace.FallenPartsDestroyHeight + 10 then returnVehicleUp(); return end
        velocity += Vector3.new(0, math.clamp((targetY - position.Y) * 6, -VehicleConfig.Speed, VehicleConfig.Speed), 0)
        status.VehicleMode = math.abs(targetY - position.Y) < 1.5 and "Underground" or "Moving underground"
    elseif VehicleConfig.Flight then
        velocity = updateLowHover(state, direction, math.clamp(dt, 0, 0.05))
    else
        state.velocity.Enabled = false
        status.VehicleMode = VehicleConfig.BumpAura and "Bump aura ready" or "Off"
    end
    state.velocity.VectorVelocity = velocity
    state.velocity.Enabled = active
end
track(RunService.PreSimulation:Connect(function(dt)
    local ok, err = pcall(updateVehicleMovement, dt)
    if not ok then stopVehicle("Vehicle error"); status.VehicleError = tostring(err) end
end))

local function recoveredFromDowned(hum)
    local state = hum:GetState()
    return hum.Health > 0 and not downed(hum) and not hum.PlatformStand
        and state ~= Enum.HumanoidStateType.Physics
        and state ~= Enum.HumanoidStateType.Ragdoll
        and state ~= Enum.HumanoidStateType.FallingDown
        and state ~= Enum.HumanoidStateType.GettingUp
        and state ~= Enum.HumanoidStateType.Dead
end

local function canReturnFromDowned(hum)
    return hum.Health > 0 and not downed(hum) and hum:GetState() ~= Enum.HumanoidStateType.Dead
end

local function moveEscapeBody(escape, destination)
    if LocalPlayer.Character ~= escape.char or not escape.root.Parent then return end
    local hum = escape.char:FindFirstChildOfClass("Humanoid")
    if isVehicleAttached(hum, escape.root) then
        status.CharacterPhysics = "Paused while attached to vehicle"
        return
    end
    local delta = destination * escape.root.CFrame:Inverse()
    escape.char:PivotTo(delta * escape.char:GetPivot())
    escape.observedPosition = escape.root.Position
    escape.observedVelocity = Vector3.zero
    escape.observedAt = os.clock()
    for _, part in ipairs(escape.char:GetDescendants()) do
        if part:IsA("BasePart") then
            part.AssemblyLinearVelocity = Vector3.zero
            part.AssemblyAngularVelocity = Vector3.zero
        end
    end
end

local function destroySnapHold(state)
    if not state then return end
    if state.anchorConnection then state.anchorConnection:Disconnect(); state.anchorConnection = nil end
    for _, name in ipairs({ "holdPosition", "holdRotation", "holdAttachment", "holdFolder" }) do
        if state[name] then state[name]:Destroy(); state[name] = nil end
    end
    if state.anchorParts then
        for part, anchored in pairs(state.anchorParts) do
            if part.Parent then part.Anchored = anchored end
        end
        state.anchorParts = nil
    end
    if state.hum and state.hum.Parent and state.autoRotate ~= nil then
        state.hum.AutoRotate = state.autoRotate
    end
    state.autoRotate = nil
end

local function releaseSnap()
    local state = snapState
    if not state or state.returning or not state.anchorParts then return end
    destroySnapHold(state)
    snapState = nil
    if LocalPlayer.Character == state.char and state.root.Parent then
        pendingReturn = state
        env.MaxGanHubReturnAnchor = state
    end
    status.Snap = "Released"
    status.SnapLastStopReason = "Release pressed"
    status.AntiKillReturnConfirmed = false
    if refreshControls then refreshControls() end
end

local function createSnapHold(state)
    local root = state.root
    assert(not isVehicleAttached(state.char:FindFirstChildOfClass("Humanoid"), root), "Exit vehicle before holding")
    state.anchorParts = { [root] = root.Anchored }
    state.hum = state.char:FindFirstChildOfClass("Humanoid")
    if state.hum then
        state.autoRotate = state.hum.AutoRotate
        state.hum.AutoRotate = false
    end
    root.AssemblyLinearVelocity = Vector3.zero
    root.AssemblyAngularVelocity = Vector3.zero
    local folder = Instance.new("Folder")
    folder.Name = "MaxGanSnapHold"
    folder.Parent = Workspace
    state.holdFolder = folder
    local attachment = Instance.new("Attachment")
    attachment.Name = "MaxGanSnapAttachment"
    attachment.Parent = root
    state.holdAttachment = attachment
    local mass = root.AssemblyMass
    if mass == math.huge or mass <= 0 then error("Character assembly is anchored") end
    local force = mass * math.max(Workspace.Gravity, 1) * 12
    local hold = Instance.new("AlignPosition")
    state.holdPosition = hold
    hold.Name = "MaxGanSnapPosition"
    hold.Mode = Enum.PositionAlignmentMode.OneAttachment
    hold.Attachment0 = attachment
    hold.ApplyAtCenterOfMass = true
    hold.RigidityEnabled = true
    hold.MaxForce = force
    hold.MaxVelocity = 80
    hold.Responsiveness = 60
    hold.Position = state.destination and state.destination.Position
        or state.origin.Position + Vector3.new(0, -SnapConfig.Depth, 0)
    hold.Parent = folder
    local rotation = Instance.new("AlignOrientation")
    state.holdRotation = rotation
    rotation.Name = "MaxGanSnapRotation"
    rotation.Mode = Enum.OrientationAlignmentMode.OneAttachment
    rotation.Attachment0 = attachment
    rotation.RigidityEnabled = false
    rotation.MaxTorque = force * 8
    rotation.MaxAngularVelocity = 40
    rotation.Responsiveness = 60
    rotation.CFrame = root.CFrame.Rotation
    rotation.Parent = folder
    if state.destination then rotation.Enabled = false end
end

local function maintainEscapeHold(escape)
    if not escape or not escape.destination or escape.returning then return end
    local hold = escape.holdPosition
    if not hold or not hold.Parent or not escape.holdAttachment
        or escape.holdAttachment.Parent ~= escape.root then
        local now = os.clock()
        if escape.nextHoldRetry and now < escape.nextHoldRetry then return end
        destroySnapHold(escape)
        local ok, err = pcall(createSnapHold, escape)
        if not ok then
            destroySnapHold(escape)
            escape.nextHoldRetry = now + 0.5
            status.AntiKillHold = tostring(err)
            return
        end
        hold = escape.holdPosition
    end
    hold.Position = escape.destination.Position
    hold.Enabled = true
    if escape.holdRotation then escape.holdRotation.Enabled = false end
    status.AntiKillHold = "Holding"
end

local function holdPositionRejected(state, label)
    local now = os.clock()
    local root = state.root
    local position, velocity = root.Position, root.AssemblyLinearVelocity
    if state.observedPosition and state.observedAt then
        local elapsed = now - state.observedAt
        local distance = (position - state.observedPosition).Magnitude
        local predicted = ((state.observedVelocity or Vector3.zero).Magnitude + velocity.Magnitude) * elapsed
        if elapsed > 0 and elapsed < 0.15 and distance > math.max(8, predicted * 2 + 2) then
            if not state.correctionWindow or now - state.correctionWindow > 8 then
                state.correctionWindow, state.correctionCount = now, 0
            end
            state.correctionCount = (state.correctionCount or 0) + 1
            status[label .. "Corrections"] = state.correctionCount
            status[label .. "CorrectionDistance"] = distance
            status[label .. "CorrectionAt"] = now
        end
    end
    state.observedPosition, state.observedVelocity, state.observedAt = position, velocity, now
    local target = state.destination and state.destination.Position
        or state.origin.Position + Vector3.new(0, -SnapConfig.Depth, 0)
    local error = (position - target).Magnitude
    status[label .. "HoldError"] = error
    if (state.correctionCount or 0) > 0 and error > 3 then
        state.stalledSince = state.stalledSince or now
    else
        state.stalledSince = nil
    end
    return (state.correctionCount or 0) >= 3
        or (state.stalledSince ~= nil and now - state.stalledSince >= 1)
end

local function stopRejectedHold(state, label)
    destroySnapHold(state)
    pendingReturn = state
    env.MaxGanHubReturnAnchor = state
    status[label] = "Position rejected - hold stopped"
    status[label .. "LastStopReason"] = "External position changes or hold stalled"
    status.AntiKillReturnConfirmed = false
    if label == "Snap" then
        snapState = nil
    else
        downedEscape = nil
        handledDownedCharacter = state.char
        downedClearedSince = nil
        status.AntiKillHold = "Stopped after corrections"
    end
    if refreshControls then refreshControls() end
end

local function safeSnapDepth(char, root, depth)
    local bounds, size = char:GetBoundingBox()
    local lowestY = bounds.Position.Y - size.Y / 2 - depth
    return root.Position.Y - depth > Workspace.FallenPartsDestroyHeight + 10
        and lowestY > Workspace.FallenPartsDestroyHeight + 5
end

toggleSnap = function()
    local char, hum, root = currentBody()
    if downedEscape and (downedEscape.char ~= char or downedEscape.root ~= root) then
        destroySnapHold(downedEscape)
        downedEscape = nil
        status.AntiKill = "Idle"
    end
    local saved = downedEscape or pendingReturn
    if not snapState and saved and saved.fromSnap and saved.char == char and saved.root == root then
        requestManualReturn()
        if refreshControls then refreshControls() end
        return
    end
    if not root or downed(hum)
        or (not snapState and not recoveredFromDowned(hum)) then
        status.Snap = "Wait for recovery"
        return
    end
    if isVehicleAttached(hum) then status.Snap = "Exit vehicle"; return end
    if root.Anchored and (not snapState or not snapState.anchorParts or snapState.anchorParts[root] == nil) then
        status.Snap = "Character anchored"
        return
    end
    if downedEscape then status.Snap = "Anti kill busy"; return end
    if snapState and snapState.returning then
        if snapState.landed then
            destroySnapHold(snapState)
            snapState = nil
        else
            status.Snap = "Returning"
            return
        end
    end
    if snapState and snapState.char == char then
        destroySnapHold(snapState)
        moveEscapeBody(snapState, snapState.origin)
        snapState.returning = true
        snapState.failed = nil
        snapState.started = os.clock()
        snapState.attempts = 1
        snapState.stableSince, snapState.landed = nil, nil
        snapState.nextReturn = os.clock() + 0.25
        status.Snap = "Returning"
    else
        local targetY = root.Position.Y - SnapConfig.Depth
        if not safeSnapDepth(char, root, SnapConfig.Depth) then
            status.Snap = "Depth below world limit"
            return
        end
        snapState = { char = char, root = root, origin = root.CFrame, targetY = targetY, downStarted = os.clock(), fromSnap = true }
        local bounds, size = char:GetBoundingBox()
        snapState.bottomOffset = root.Position.Y - (bounds.Position.Y - size.Y / 2)
        local ok, err = pcall(createSnapHold, snapState)
        if not ok then
            destroySnapHold(snapState)
            snapState = nil
            status.Snap = "Height hold unavailable"
            warn("[MaxGan Hub] Snap height hold unavailable: " .. tostring(err))
            return
        end
        moveEscapeBody(snapState, root.CFrame + Vector3.new(0, -SnapConfig.Depth, 0))
        snapState.lastY = root.Position.Y
        snapState.target = snapState.origin + Vector3.new(0, -SnapConfig.Depth, 0)
        status.Snap = "Checking down"
        status.SnapLastStopReason = nil
        status.SnapHoldRebuilds = 0
    end
    if refreshControls then refreshControls() end
end

track(RunService.Heartbeat:Connect(function(dt)
    if not isRunning or not snapState then return end
    local char, hum, root = currentBody()
    local state = snapState
    if char ~= state.char or not root or isVehicleAttached(hum) then
        destroySnapHold(state)
        snapState = nil
        status.Snap = "Idle"
        return
    end
    if downed(hum) or downedEscape then
        destroySnapHold(state)
        if CharacterConfig.AntiKill then
            snapRecoveryOrigin = state.origin
        else
            pendingReturn = state
            env.MaxGanHubReturnAnchor = state
        end
        snapState = nil
        status.Snap = "Anti kill takeover"
        status.SnapLastStopReason = "Downed or anti kill takeover"
        return
    end
    if state.failed then return end
    if not state.returning then
        if holdPositionRejected(state, "Snap") then
            stopRejectedHold(state, "Snap")
            return
        end
        local targetY = state.origin.Position.Y - SnapConfig.Depth
        if targetY <= Workspace.FallenPartsDestroyHeight + 10
            or targetY - state.bottomOffset <= Workspace.FallenPartsDestroyHeight + 5 then
            toggleSnap()
            return
        end
        local targetPosition = state.origin.Position + Vector3.new(0, -SnapConfig.Depth, 0)
        local camera = Workspace.CurrentCamera
        local look = camera and camera.CFrame.LookVector or root.CFrame.LookVector
        local horizontal = Vector3.new(look.X, 0, look.Z)
        local rotation = horizontal.Magnitude > 0.001
            and CFrame.lookAt(Vector3.zero, horizontal.Unit) or root.CFrame.Rotation
        local target = CFrame.new(targetPosition) * rotation
        state.targetY = targetY
        if not state.holdPosition or not state.holdPosition.Parent
            or not state.holdRotation or not state.holdRotation.Parent then
            state.rebuildAttempts = (state.rebuildAttempts or 0) + 1
            status.SnapHoldRebuilds = state.rebuildAttempts
            destroySnapHold(state)
            local ok, err = pcall(createSnapHold, state)
            if not ok or state.rebuildAttempts > 3 then
                destroySnapHold(state)
                moveEscapeBody(state, state.origin)
                state.returning = true
                state.started = os.clock()
                state.attempts = 1
                state.nextReturn = os.clock() + 0.25
                state.stableSince, state.landed = nil, nil
                env.MaxGanHubReturnAnchor = state
                status.Snap = "Hold failed - returning"
                status.SnapLastStopReason = tostring(err or "Hold repeatedly removed")
                if refreshControls then refreshControls() end
                return
            end
        end
        state.holdPosition.Enabled = true
        state.holdRotation.Enabled = true
        state.holdPosition.Position = targetPosition
        state.holdRotation.CFrame = rotation
        status.SnapHeightError = root.Position.Y - targetY
        status.SnapPositionError = (root.Position - targetPosition).Magnitude
        local velocity = root.AssemblyLinearVelocity.Magnitude
        if status.SnapPositionError <= 1 and velocity < 5 then
            state.holdStableSince = state.holdStableSince or os.clock()
            status.Snap = os.clock() - state.holdStableSince >= 0.5 and "Down" or "Checking down"
        else
            state.holdStableSince = nil
            status.Snap = "Holding down"
        end
        state.target = target
        return
    end
    local now = os.clock()
    local params = state.returnRaycastParams
    if not params then
        params = RaycastParams.new()
        params.FilterType = Enum.RaycastFilterType.Exclude
        params.FilterDescendantsInstances = { char }
        params.RespectCanCollide = true
        state.returnRaycastParams = params
    end
    local floor = Workspace:Raycast(root.Position, Vector3.new(0, -(hum.HipHeight + root.Size.Y / 2 + 2), 0), params)
    local grounded = hum.FloorMaterial ~= Enum.Material.Air or floor ~= nil
    local arrived = (root.Position - state.origin.Position).Magnitude < 6
        and math.abs(root.Position.Y - state.origin.Position.Y) < 3
    status.SnapReturnDistance = (root.Position - state.origin.Position).Magnitude
    status.SnapReturnHeightOffset = root.Position.Y - state.origin.Position.Y
    status.SnapReturnGrounded = grounded
    status.SnapReturnAttempts = state.attempts
    if arrived and grounded and recoveredFromDowned(hum) then state.landed = true end
    if state.landed and root.Position.Y < state.origin.Position.Y - 8 and not grounded then
        state.landed, state.stableSince = nil, nil
    end
    if state.landed and recoveredFromDowned(hum) then
        state.stableSince = state.stableSince or now
        status.Snap = "Checking return"
        if now - state.stableSince >= math.clamp((status.Ping or 150) / 1000 * 3 + 0.5, 1, 2) then
            snapState = nil
            status.Snap = "Idle"
            if refreshControls then refreshControls() end
        end
    elseif now >= state.nextReturn and state.attempts < 40 then
        moveEscapeBody(state, state.origin)
        state.nextReturn, state.attempts = now + 0.25, state.attempts + 1
    end
    if snapState == state and now - state.started >= 15 then
        state.returning = nil
        state.failed = true
        status.Snap = "Return unconfirmed - retry"
    end
end))

local function finishEscape(escape, result)
    destroySnapHold(escape)
    status.AntiKillHold = "Released"
    handledDownedCharacter = escape.char
    downedClearedSince = nil
    status.AntiKill = result
    status.AntiKillReturnConfirmed = result == "Returned"
    if result == "Returned" and escape.fromSnap then status.Snap = "Idle" end
    pendingReturn = result == "Return unconfirmed" and escape or nil
    if result == "Returned" or result == "Cancelled in vehicle" then
        env.MaxGanHubReturnAnchor = nil
    else
        env.MaxGanHubReturnAnchor = escape
    end
    downedEscape = nil
end

local function releaseEscape(force)
    local escape = downedEscape
    if not escape then return true end
    local char, hum, root = currentBody()
    if char ~= escape.char or root ~= escape.root or not hum then
        destroySnapHold(escape)
        downedEscape = nil
        if env.MaxGanHubReturnAnchor == escape then env.MaxGanHubReturnAnchor = nil end
        status.AntiKill = "Character changed"
        status.AntiKillReturnConfirmed = false
        return true
    end
    if isVehicleAttached(hum) then
        finishEscape(escape, "Cancelled in vehicle")
        return true
    end
    if force then
        destroySnapHold(escape)
        if not escape.landed then moveEscapeBody(escape, escape.origin) end
        finishEscape(escape, "Released")
        return true
    end
    local now = os.clock()
    if not escape.returning then
        escape.returning = true
        escape.returnStarted = now
        escape.returnAttempts = 0
        status.AntiKill = "Returning"
        status.AntiKillReturnConfirmed = false
    end
    if not canReturnFromDowned(hum) then
        escape.returnRecoverySince, escape.stableSince, escape.monitorUntil = nil, nil, nil
        escape.landed = nil
        escape.returnStarted = now
        status.AntiKillReturnConfirmed = false
        status.AntiKill = "Waiting for recovery"
        return false
    end
    escape.returnRecoverySince = escape.returnRecoverySince or escape.recoveredSince or now
    if now - escape.returnRecoverySince < 0.75 then
        escape.returnStarted = now
        status.AntiKill = "Confirming recovery"
        return false
    end
    destroySnapHold(escape)
    status.AntiKillHold = "Released for return"
    local offset = root.Position - escape.origin.Position
    local arrived = offset.Magnitude <= 6 and math.abs(offset.Y) <= 3
    local params = escape.returnRaycastParams
    if not params then
        params = RaycastParams.new()
        params.FilterType = Enum.RaycastFilterType.Exclude
        params.FilterDescendantsInstances = { char }
        params.RespectCanCollide = true
        escape.returnRaycastParams = params
    end
    local floor = Workspace:Raycast(root.Position, Vector3.new(0, -(hum.HipHeight + root.Size.Y / 2 + 2), 0), params)
    local grounded = hum.FloorMaterial ~= Enum.Material.Air or floor ~= nil
    local recovered = recoveredFromDowned(hum)
    local settled = math.abs(root.AssemblyLinearVelocity.Y) <= 8
    if arrived and grounded and recovered then escape.landed = true end
    if escape.landed and root.Position.Y < escape.origin.Position.Y - 8 then
        escape.fellSince = escape.fellSince or now
        if now - escape.fellSince >= 0.4 then escape.landed = nil end
    else
        escape.fellSince = nil
    end
    arrived = arrived or escape.landed == true
    status.AntiKillReturnDistance = offset.Magnitude
    status.AntiKillReturnGrounded = grounded
    status.AntiKillReturnRecovered = recovered
    status.AntiKillReturnAttempts = escape.returnAttempts
    if escape.monitorUntil then
        if recovered and escape.landed then
            status.AntiKill = "Verifying return"
            if now >= escape.monitorUntil then
                finishEscape(escape, "Returned")
                return true
            end
            return false
        end
        escape.monitorUntil = nil
        escape.stableSince = nil
        status.AntiKillReturnConfirmed = false
    end
    if arrived and grounded and recovered and settled then
        escape.stableSince = escape.stableSince or now
        local confirmationTime = math.clamp((status.Ping or 150) / 1000 * 4 + 0.5, 1, 2)
        if now - escape.stableSince >= confirmationTime then
            escape.monitorUntil = now + 3
            status.AntiKill = "Verifying return"
            return false
        end
    else
        escape.stableSince = nil
        if not escape.landed and (not arrived or not grounded)
            and escape.returnAttempts < 40 and (not escape.nextReturn or now >= escape.nextReturn) then
            moveEscapeBody(escape, escape.origin)
            escape.returnAttempts = escape.returnAttempts + 1
            escape.nextReturn = now + 0.25
            status.AntiKill = "Returning"
        end
    end
    if now - escape.returnStarted >= 15 then
        finishEscape(escape, "Return unconfirmed")
        return true
    end
    return false
end

requestManualReturn = function()
    local escape = downedEscape or pendingReturn
    local char, hum, root = currentBody()
    if not escape or char ~= escape.char or root ~= escape.root or not hum then
        status.AntiKill = "No saved return point"
        return
    end
    if isVehicleAttached(hum) then
        status.AntiKill = "Exit vehicle to return"
        return
    end
    downedEscape, pendingReturn = escape, nil
    env.MaxGanHubReturnAnchor = escape
    escape.manualReturn = true
    escape.waitingRecovery, escape.recoveredSince = nil, nil
    escape.returning, escape.returnStarted, escape.returnAttempts = true, os.clock(), 0
    escape.stableSince, escape.monitorUntil, escape.nextReturn = nil, nil, nil
    escape.landed, escape.fellSince = nil, nil
    escape.returnRecoverySince = nil
    status.AntiKillReturnConfirmed = false
    status.AntiKill = "Returning"
    releaseEscape()
end


local function restoreJumpSettings()
    if jumpHumanoid and jumpHumanoid.Parent and jumpSettings then
        if jumpSettings.power and math.abs(jumpHumanoid.JumpPower - jumpSettings.power) < 0.001 then
            jumpHumanoid.JumpPower = jumpSettings.originalPower
        end
        if jumpSettings.height and math.abs(jumpHumanoid.JumpHeight - jumpSettings.height) < 0.001 then
            jumpHumanoid.JumpHeight = jumpSettings.originalHeight
        end
    end
    jumpHumanoid, jumpSettings = nil, nil
end
characterCleanup = function()
    destroySnapHold(snapState)
    if snapState and not snapState.landed then moveEscapeBody(snapState, snapState.origin) end
    snapState, snapRecoveryOrigin = nil, nil
    releaseEscape(true)
    restoreJumpSettings()
end

track(LocalPlayer.CharacterAdded:Connect(function()
    destroySnapHold(downedEscape)
    destroySnapHold(snapState)
    snapState, snapRecoveryOrigin = nil, nil
    downedEscape = nil
    pendingReturn = nil
    env.MaxGanHubReturnAnchor = nil
    handledDownedCharacter, downedClearedSince = nil, nil
    status.AntiKill, status.Snap = "Idle", "Idle"
    status.SnapLastStopReason = "Character respawned"
    status.AntiKillReturnConfirmed = false
    restoreJumpSettings()
end))
track(RunService.PreSimulation:Connect(function(dt)
    if not isRunning then return end
    local _, hum, root = currentBody()
    if hum and isVehicleAttached(hum, root) then
        if snapState then
            local saved = snapState
            destroySnapHold(saved)
            pendingReturn = saved
            env.MaxGanHubReturnAnchor = saved
            snapState, snapRecoveryOrigin = nil, nil
            status.Snap = "Cancelled in vehicle"
        end
        if downedEscape then
            local saved = downedEscape
            destroySnapHold(saved)
            pendingReturn = saved
            env.MaxGanHubReturnAnchor = saved
            downedEscape = nil
            status.AntiKill = "Cancelled in vehicle"
            status.AntiKillReturnConfirmed = false
        end
        return
    end
    if not hum or downed(hum)
        or (snapState and snapState.anchorParts ~= nil)
        or (downedEscape and downedEscape.returning and not downedEscape.landed) then return end
    if jumpHumanoid ~= hum then
        restoreJumpSettings()
        jumpHumanoid = hum
        jumpSettings = { originalPower = hum.JumpPower, originalHeight = hum.JumpHeight }
    end
    if CharacterConfig.HighJump then
        if hum.UseJumpPower then
            jumpSettings.power = math.sqrt(2 * Workspace.Gravity * CharacterConfig.JumpHeight)
            if hum.JumpPower ~= jumpSettings.power then hum.JumpPower = jumpSettings.power end
        elseif Workspace.Gravity > 0 then
            jumpSettings.height = CharacterConfig.JumpHeight
            if hum.JumpHeight ~= jumpSettings.height then hum.JumpHeight = jumpSettings.height end
        end
    elseif jumpSettings then
        restoreJumpSettings()
    end
    if CharacterConfig.WalkSpeed and hum.MoveDirection.Magnitude > 0 then
        local extraSpeed = math.clamp(CharacterConfig.SpeedMultiplier, 1, 5) * 0.05 * 60
        root.CFrame = root.CFrame + hum.MoveDirection.Unit * extraSpeed * math.min(dt, 0.05)
    end
end))


track(RunService.Heartbeat:Connect(function()
    if not isRunning then return end
    local char, hum, root = currentBody()
    if not root then
        destroySnapHold(downedEscape)
        downedEscape, snapRecoveryOrigin, lastSafeReturn = nil, nil, nil
        status.AntiKill = "Character unavailable"
        status.AntiKillReturnConfirmed = false
        return
    end
    if downedEscape and (downedEscape.char ~= char or downedEscape.root ~= root) then
        destroySnapHold(downedEscape)
        downedEscape = nil
        status.AntiKill = "Character changed"
        status.AntiKillReturnConfirmed = false
    end
    local now = os.clock()
    if handledDownedCharacter and handledDownedCharacter ~= char then
        handledDownedCharacter, downedClearedSince = nil, nil
    end
    if not downed(hum) then
        downedClearedSince = downedClearedSince or now
        if now - downedClearedSince >= 0.75 then handledDownedCharacter = nil end
    else
        downedClearedSince = nil
    end
    if not downedEscape and not snapState and canReturnFromDowned(hum)
        and hum.FloorMaterial ~= Enum.Material.Air
        and (not lastSafeReturn or now - lastSafeReturn.time >= 0.2) then
        lastSafeReturn = { char = char, root = root, cframe = root.CFrame, time = now }
    end
    if downedEscape and not CharacterConfig.AntiKill and not downedEscape.manualReturn then releaseEscape(true); return end
    if downedEscape and downedEscape.returning then releaseEscape(); return end
    if downedEscape and holdPositionRejected(downedEscape, "AntiKill") then
        stopRejectedHold(downedEscape, "AntiKill")
        return
    end
    if downedEscape and downedEscape.waitingRecovery then
        maintainEscapeHold(downedEscape)
        local escape = downedEscape
        if isVehicleAttached(hum) then finishEscape(escape, "Cancelled in vehicle"); return end
        if canReturnFromDowned(hum) then
            escape.recoveredSince = escape.recoveredSince or now
            if now - escape.recoveredSince >= 0.75 then
                releaseEscape()
            elseif escape.destination and (root.Position - escape.destination.Position).Magnitude > 3
                and (not escape.nextEscape or now >= escape.nextEscape) then
                moveEscapeBody(escape, escape.destination)
                escape.nextEscape = now + 0.1
            end
        else
            escape.recoveredSince = nil
            if downed(hum) and escape.destination
                and (root.Position - escape.destination.Position).Magnitude > 3
                and (not escape.nextEscape or now >= escape.nextEscape) then
                moveEscapeBody(escape, escape.destination)
                escape.nextEscape = now + 0.1
            end
        end
        status.AntiKill = downedEscape and not downedEscape.returning and "Waiting for recovery" or status.AntiKill
        return
    end
    if CharacterConfig.AntiKill and downed(hum) and not isVehicleAttached(hum) then
        if handledDownedCharacter == char then return end
        if not downedEscape then
            local bounds, size = char:GetBoundingBox()
            local lowestY = bounds.Position.Y - size.Y / 2
            local depth = math.min(40, root.Position.Y - Workspace.FallenPartsDestroyHeight - 10,
                lowestY - Workspace.FallenPartsDestroyHeight - 5)
            if depth < 3 then
                status.AntiKill = "No safe escape depth"
                handledDownedCharacter = char
                return
            end
            pendingReturn = nil
            downedEscape = {
                char = char, root = root,
                origin = snapRecoveryOrigin or (lastSafeReturn and lastSafeReturn.char == char
                    and lastSafeReturn.root == root and now - lastSafeReturn.time < 2
                    and lastSafeReturn.cframe) or root.CFrame,
                fromSnap = snapRecoveryOrigin ~= nil,
                destination = root.CFrame + Vector3.new(0, -depth, 0), started = now,
            }
            env.MaxGanHubReturnAnchor = downedEscape
            snapRecoveryOrigin = nil
        end
        local escape = downedEscape
        maintainEscapeHold(escape)
        local offset = (root.Position - escape.destination.Position).Magnitude
        if now - escape.started >= 12 then
            escape.waitingRecovery = true
            status.AntiKill = "Waiting for recovery"
            return
        end
        if offset > 3 and (not escape.nextEscape or now >= escape.nextEscape) then
            moveEscapeBody(escape, escape.destination)
            escape.nextEscape = now + 0.1
            escape.escapeAttempts = (escape.escapeAttempts or 0) + 1
        end
        status.AntiKillEscapeDistance = offset
        status.AntiKillEscapeAttempts = escape.escapeAttempts or 0
        status.AntiKill = offset <= 3 and "Escape position reached" or "Escaping"
        status.AntiKillReturnConfirmed = false
        return
    elseif downedEscape then
        downedEscape.waitingRecovery = true
        downedEscape.recoveredSince = recoveredFromDowned(hum) and now or nil
        status.AntiKill = "Waiting for recovery"
        return
    end
end))

local vehicleImpactUntil = 0
local impactBody = nil
local impactVelocities = {}
local impactParts = nil
local impactPartNames = { "HumanoidRootPart", "LowerTorso", "UpperTorso" }
local vehicleCharacterConnections = {}
local vehiclePartCache = setmetatable({}, { __mode = "k" })
local function disconnectVehicleCharacter()
    for _, connection in ipairs(vehicleCharacterConnections) do connection:Disconnect() end
    table.clear(vehicleCharacterConnections)
    impactParts = nil
end
local function trackVehicleCharacter(connection)
    vehicleCharacterConnections[#vehicleCharacterConnections + 1] = connection
end
track({ Disconnect = disconnectVehicleCharacter })
local vehicleModels = setmetatable({}, { __mode = "k" })
local clearVehicleRagdoll = nil
local nativeRagdoll = nil
local impactLastContact = 0
local contactVehicle = nil
local vehicleContactUntil = 0
local vehicleEvents = {}
local vehicleDamageAt = 0
local damageHumanoid = nil
status.VehicleRagdollEvents = vehicleEvents

local function recordVehicleEvent(event, detail)
    local _, hum = currentBody()
    vehicleEvents[#vehicleEvents + 1] = {
        time = os.clock(), event = event, detail = detail,
        source = hum and hum:GetAttribute("LastDamageSource"),
        downed = hum and downed(hum), health = hum and hum.Health,
    }
    if #vehicleEvents > 24 then table.remove(vehicleEvents, 1) end
end

local function deathRagdoll(data)
    local name = type(data) == "table" and tostring(data.name) or tostring(data)
    name = string.lower(name)
    return string.find(name, "dead", 1, true) ~= nil
        or string.find(name, "down", 1, true) ~= nil or name == "in_vehicle"
end

local function vehicleImpactAllowed(hum)
    if not hum then local _, currentHum = currentBody(); hum = currentHum end
    return isRunning and CharacterConfig.AntiVehicleRagdoll and hum
        and not isVehicleAttached(hum) and not downed(hum)
end

local function recentVehicleDamage()
    local _, hum = currentBody()
    if not hum or hum ~= damageHumanoid or os.clock() - vehicleDamageAt > 0.75 then return false end
    local source = hum:GetAttribute("LastDamageSource")
    local vehicles = Workspace:FindFirstChild("Vehicles")
    local vehicle = type(source) == "string" and vehicles and vehicles:FindFirstChild(source)
    if not vehicle or not vehicle:IsA("Model") or vehicle:GetAttribute("VehicleType") == nil then return false end
    return true
end

local function vehicleFromPart(part)
    local now = os.clock()
    local cached = vehiclePartCache[part]
    if cached and cached.parent == part.Parent then
        if cached.vehicle and cached.vehicle:IsDescendantOf(Workspace)
            and part:IsDescendantOf(cached.vehicle) then return cached.vehicle end
        if not cached.vehicle and now < cached.expires then return nil end
    end
    local ancestor = part.Parent
    while ancestor and ancestor ~= Workspace do
        if ancestor:IsA("Model") then
            if vehicleModels[ancestor] then
                vehiclePartCache[part] = { parent = part.Parent, vehicle = ancestor }
                return ancestor
            end
            if ancestor:GetAttribute("VehicleType") ~= nil
                or ancestor:FindFirstChildWhichIsA("VehicleSeat", true) then
                vehicleModels[ancestor] = true
                vehiclePartCache[part] = { parent = part.Parent, vehicle = ancestor }
                return ancestor
            end
        end
        ancestor = ancestor.Parent
    end
    vehiclePartCache[part] = { parent = part.Parent, expires = now + 0.25 }
end

local function markVehicleImpact()
    vehicleImpactUntil = os.clock() + 0.75
    status.VehicleRagdollGuard = "Vehicle impact detected"
end

local function observeVehicleContact(char)
    disconnectVehicleCharacter()
    local function watch(part)
        if not part:IsA("BasePart") then return end
        trackVehicleCharacter(part.Touched:Connect(function(other)
            local root = char:FindFirstChild("HumanoidRootPart")
            if LocalPlayer.Character == char and vehicleImpactAllowed() and root
                and other:IsA("BasePart") then
                local vehicle = vehicleFromPart(other)
                if not vehicle then return end
                contactVehicle, vehicleContactUntil = vehicle, os.clock() + 1
                local speed = other.AssemblyLinearVelocity.Magnitude
                if os.clock() - impactLastContact > 0.08 then
                    impactLastContact = os.clock()
                    recordVehicleEvent("Contact", { vehicle = vehicle.Name, speed = speed })
                    if speed > 5 then markVehicleImpact() end
                end
            end
        end))
    end
    for _, part in ipairs(char:GetDescendants()) do watch(part) end
    trackVehicleCharacter(char.DescendantAdded:Connect(function(part)
        impactParts = nil
        watch(part)
    end))
    trackVehicleCharacter(char.DescendantRemoving:Connect(function() impactParts = nil end))
    local function observeHumanoid(hum)
        if not hum:IsA("Humanoid") then return end
        local previousHealth = hum.Health
        trackVehicleCharacter(hum.HealthChanged:Connect(function(health)
            local damaged = health < previousHealth
            previousHealth = health
            if not damaged then return end
            damageHumanoid, vehicleDamageAt = hum, os.clock()
            task.defer(function()
                if LocalPlayer.Character ~= char or not vehicleImpactAllowed() then return end
                if recentVehicleDamage() then
                    markVehicleImpact()
                    recordVehicleEvent("Vehicle damage", hum:GetAttribute("LastDamageSource"))
                end
            end)
        end))
    end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then observeHumanoid(hum) end
    trackVehicleCharacter(char.ChildAdded:Connect(observeHumanoid))
end
if LocalPlayer.Character then observeVehicleContact(LocalPlayer.Character) end
track(LocalPlayer.CharacterAdded:Connect(function(char)
    impactBody, impactVelocities, vehicleImpactUntil, impactLastContact = nil, {}, 0, 0
    contactVehicle, vehicleContactUntil = nil, 0
    damageHumanoid, vehicleDamageAt = nil, 0
    observeVehicleContact(char)
end))
track(RunService.PreSimulation:Connect(function()
    local char, hum, root = currentBody()
    if root ~= impactBody then
        impactBody, impactParts = root, nil
        table.clear(impactVelocities)
    end
    if not root or not vehicleImpactAllowed(hum) then
        table.clear(impactVelocities)
        return
    end
    if not impactParts then
        impactParts = {}
        table.clear(impactVelocities)
        for _, name in ipairs(impactPartNames) do
            local part = char:FindFirstChild(name)
            if part and part:IsA("BasePart") then impactParts[#impactParts + 1] = part end
        end
    end
    for _, part in ipairs(impactParts) do
        impactVelocities[part] = part.AssemblyLinearVelocity
    end
end))
track(RunService.PostSimulation:Connect(function()
    if not vehicleImpactAllowed() then return end
    if recentVehicleDamage() and os.clock() >= vehicleImpactUntil then markVehicleImpact() end
    if os.clock() >= vehicleImpactUntil then return end
    if nativeRagdoll and clearVehicleRagdoll and not deathRagdoll(status.LastRagdollType)
        and nativeRagdoll.is_ragdolling.get() then
        local ok, err = pcall(clearVehicleRagdoll)
        status.VehicleRagdollGuard = ok and "Vehicle ragdoll cleared" or tostring(err)
    end
    local char = LocalPlayer.Character
    for part, velocity in pairs(impactVelocities) do
        local assembly = part.Parent and part.AssemblyRootPart
        if char and part:IsDescendantOf(char) and assembly and assembly:IsDescendantOf(char) then
            if (part.AssemblyLinearVelocity - velocity).Magnitude > 12 then
                part.AssemblyLinearVelocity = velocity
                part.AssemblyAngularVelocity = Vector3.zero
            end
        end
    end
end))
task.spawn(function()
    local ok, err = pcall(function()
        local net = requireGameModule(ReplicatedStorage.Modules.Core.Net)
        nativeRagdoll = requireGameModule(ReplicatedStorage.Modules.Game.Ragdoll)
        local handlers
        for _, value in pairs(getupvalues(net.hook)) do
            if type(value) == "table" and type(value.apply_ragdoll) == "function"
                and type(value.apply_ragdoll_force) == "function" then handlers = value; break end
        end
        assert(handlers, "Native ragdoll handlers unavailable")
        if not isRunning then return end
        clearVehicleRagdoll = handlers.clear_ragdoll
        local originalRagdoll, originalForce = handlers.apply_ragdoll, handlers.apply_ragdoll_force
        local wrappedRagdoll, wrappedForce
        local function shouldBlockVehicleImpact()
            return vehicleImpactAllowed() and (os.clock() < vehicleImpactUntil or recentVehicleDamage())
        end
        wrappedRagdoll = function(data, ...)
            status.LastRagdollType = type(data) == "table" and tostring(data.name) or tostring(data)
            recordVehicleEvent("Ragdoll", status.LastRagdollType)
            if not deathRagdoll(data) and shouldBlockVehicleImpact() then
                status.VehicleRagdollBlocked = (status.VehicleRagdollBlocked or 0) + 1
                return
            end
            return originalRagdoll(data, ...)
        end
        wrappedForce = function(...)
            recordVehicleEvent("Force", os.clock() < vehicleImpactUntil)
            if not deathRagdoll(status.LastRagdollType) and shouldBlockVehicleImpact() then
                status.VehicleForceBlocked = (status.VehicleForceBlocked or 0) + 1
                return
            end
            return originalForce(...)
        end
        handlers.apply_ragdoll, handlers.apply_ragdoll_force = wrappedRagdoll, wrappedForce
        status.VehicleRagdollGuard = "Ready"
        collisionCleanup = function()
            clearVehicleRagdoll = nil
            if handlers.apply_ragdoll == wrappedRagdoll then handlers.apply_ragdoll = originalRagdoll end
            if handlers.apply_ragdoll_force == wrappedForce then handlers.apply_ragdoll_force = originalForce end
        end
    end)
    if not ok then
        status.VehicleRagdollGuard = tostring(err)
        warn("[MaxGan Hub] Vehicle impact protection unavailable: " .. tostring(err))
    end
end)

local pingSamples = {}
local elapsedFrames, frames, pingElapsed, targetElapsed = 0, 0, 0, 0
track(RunService.RenderStepped:Connect(function(dt)
    if not isRunning then return end
    elapsedFrames = elapsedFrames + dt
    frames = frames + 1
    pingElapsed = pingElapsed + dt
    targetElapsed = targetElapsed + dt
    if elapsedFrames >= 0.5 then
        status.FPS = math.floor(frames / elapsedFrames + 0.5)
        elapsedFrames, frames = 0, 0
    end
    if pingElapsed >= 0.5 then
        pingElapsed = 0
        local ok, ping = pcall(function()
            return Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
        end)
        if not ok or type(ping) ~= "number" or ping < 0 or ping ~= ping or ping == math.huge then
            status.Ping, status.PingJitter = nil, nil
            status.PingState = "Unavailable"
            table.clear(pingSamples)
            status.PingSamples = 0
        else
            status.Ping = math.floor(ping + 0.5)
            pingSamples[#pingSamples + 1] = ping
            if #pingSamples > 20 then table.remove(pingSamples, 1) end
            status.PingSamples = #pingSamples
            local delta, low, high = 0, ping, ping
            for i, sample in ipairs(pingSamples) do
                low, high = math.min(low, sample), math.max(high, sample)
                if i > 1 then delta = delta + math.abs(sample - pingSamples[i - 1]) end
            end
            local jitter = #pingSamples > 1 and delta / (#pingSamples - 1) or 0
            status.PingJitter = math.floor(jitter + 0.5)
            if #pingSamples < 8 then
                status.PingState = "Measuring"
            elseif jitter >= 20 or high - low >= 60 then
                status.PingState = "Unstable"
            elseif ping >= 150 then
                status.PingState = "High latency"
            else
                status.PingState = "Stable"
            end
        end
    end
    if targetElapsed >= 0.15 then
        targetElapsed = 0
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        local alive = root and hum and hum.Health > 0
        if alive and aimLockEnabled and AimConfig.Mode == "Silent" then
            lockedTarget = selectedTarget()
        else
            lockedTarget = alive and isAimLockHeld() and cameraLockTarget and validLockTarget(cameraLockTarget) and cameraLockTarget or nil
        end
        status.Target = lockedTarget and lockedTarget.Name or nil
        local head = lockedTarget and getAimPart(lockedTarget)
        status.TargetDistance = head and root and (head.Position - root.Position).Magnitude or nil
        if not alive then
            status.State = "Waiting for character"
        elseif lockedTarget then
            status.State = "Head target selected"
        else
            status.State = "No target"
        end
    end
end))

task.spawn(function()
    local ok, err = pcall(function()
        local WindUI = loadstring(game:HttpGet("https://raw.githubusercontent.com/Footagesus/WindUI/1.6.66/dist/main.lua"))()
        if not isRunning then return end
local thaiLabels={
["General"]="ตัวละคร",
["Combat"]="การต่อสู้",
["Weapon Mod"]="ปรับอาวุธ",
["Esp"]="การแสดงผล",
["Visualize"]="การแสดงผล",
["Vehicle"]="ยานพาหนะ",
["Buy"]="สุ่มลัง",
["Shop"]="ร้านค้าและสุ่มลัง",
["Cosmetic"]="สกิน",
["Misc"]="ฟื้นฟูและมุดดิน",
["Key Bind"]="ปุ่มลัด",
["Server"]="เซิร์ฟเวอร์",
["Config"]="การตั้งค่า",
["Movement Modifications"]="การเคลื่อนที่",
["Enable Speed"]="เพิ่มความเร็ว",
["Enable Super Jump"]="กระโดดสูง",
["Anti Kill"]="ป้องกันการตาย",
["Vehicle Ragdoll Guard"]="ป้องกันล้มจากรถ",
["Speed boost strength"]="ตัวคูณความเร็ว",
["Jump height (studs)"]="ความสูงกระโดด (สตัด)",
["Item Systems"]="การเก็บของ",
["PickUp Item (ดูดของอัตโนมัติ)"]="เก็บของอัตโนมัติ",
["Pickup range (studs)"]="ระยะเก็บของ (สตัด)",
["Aim System"]="ระบบช่วยเล็ง",
["Enable Aim"]="เปิดช่วยเล็ง",
["Aim mode"]="รูปแบบช่วยเล็ง",
["On-foot response"]="ความไวเมื่อเดิน",
["Vehicle response"]="ความไวเมื่ออยู่ในรถ",
["Aim status"]="สถานะช่วยเล็ง",
["Exclude players"]="ผู้เล่นที่ไม่เล็ง",
["Melee"]="อาวุธประชิด",
["Melee Aura"]="โจมตีประชิดอัตโนมัติ",
["Melee Range"]="ระยะโจมตีประชิด",
["Aura tools"]="การใช้อาวุธ",
["Throwable"]="อาวุธขว้าง",
["Throw Range"]="ระยะขว้าง",
["Throw Aura"]="ขว้างอัตโนมัติ",
["Gun Modifications"]="ปรับการยิง",
["Accuracy / Range Mod"]="ปรับความแม่นและระยะ",
["Weapon status"]="สถานะอาวุธ",
["Fast Reload"]="รีโหลดเร็ว",
["Reload Speed Multiplier"]="ตัวคูณความเร็วรีโหลด",
["Player ESP"]="แสดงข้อมูลผู้เล่น",
["Enable ESP"]="เปิดแสดงผู้เล่น",
["ESP Box"]="กรอบผู้เล่น",
["ESP Name"]="ชื่อผู้เล่น",
["ESP Distance"]="ระยะห่าง",
["ESP Health"]="เลือดผู้เล่น",
["ESP Inventory"]="ของที่ผู้เล่นถือ",
["Max distance (studs)"]="ระยะแสดงสูงสุด (สตัด)",
["Camera & FOV Effects"]="วงช่วยเล็ง",
["Show FOV"]="แสดงวงช่วยเล็ง",
["FOV radius (pixels)"]="รัศมีวงช่วยเล็ง (พิกเซล)",
["Vehicle Handling"]="การควบคุมรถ",
["Smooth Drive"]="รถลอยเหนือพื้น",
["Smooth Drive controls"]="วิธีควบคุมรถ",
["Vehicle Underground"]="รถมุดดิน",
["Vehicle speed (studs/s)"]="ความเร็วรถ (สตัด/วินาที)",
["Hover height (studs)"]="ความสูงเหนือพื้น (สตัด)",
["Underground depth"]="ความลึกใต้ดิน",
["Bump radius"]="ระยะชน",
["Impact speed"]="ความเร็วชน",
["Bump Aura"]="ชนเป้าอัตโนมัติ",
["Vehicle Return UP"]="นำรถขึ้นพื้น",
["Stop vehicle features"]="หยุดระบบรถ",
["Vehicle status"]="สถานะรถ",
["Smooth Drive Speed Multiplier"]="ตัวคูณความเร็วรถลอย",
["Pull Car Owner"]="เรียกรถของตัวเอง",
["Buy & Crate Skip"]="การสุ่มลัง",
["Skip crate animation"]="ข้ามภาพหมุนลัง",
["Crate animation status"]="สถานะการสุ่ม",
["Item Shop"]="ร้านขายของ",
["Open nearby Item Shop"]="เปิดร้านใกล้ตัว",
["Shop status"]="สถานะร้านค้า",
["Skins & Effects"]="สกินและเอฟเฟกต์",
["Item to customize"]="ของที่ต้องการใส่สกิน",
["Owned cosmetic"]="สกินที่มี",
["Refresh inventory"]="โหลดรายการของใหม่",
["Equip / Unequip cosmetic"]="ใส่ / ถอดสกิน",
["Cosmetic status"]="สถานะสกิน",
["Recovery & Healing"]="การฟื้นฟูเลือด",
["Auto Heal"]="เติมเลือดอัตโนมัติ",
["Heal below HP"]="เติมเมื่อเลือดต่ำกว่า",
["Heal animation speed"]="ความเร็วใช้ของเติมเลือด",
["Stamina Consumables"]="การเติมสเตมิน่า",
["Auto Consume (เพิ่มสเตมิน่า)"]="ใช้ของเติมสเตมิน่าอัตโนมัติ",
["Consume below stamina (%)"]="ใช้เมื่อสเตมิน่าต่ำกว่า (%)",
["Stamina consumable"]="ของเติมสเตมิน่าที่ใช้",
["Auto Consume status"]="สถานะเติมสเตมิน่า",
["Snap DOWN / UP"]="มุดดิน / กลับขึ้น",
["Return UP"]="กลับขึ้นพื้น",
["Release snap hold"]="ปล่อยการตรึงตัว",
["Snap depth (studs)"]="ความลึกมุดดิน (สตัด)",
["Instant Spin Controls"]="การข้ามสุ่มลัง",
["Instant Spin (ข้ามการหมุนลัง)"]="ข้ามการหมุนลังทันที",
["Force Spin 1x"]="สุ่มลังที่เลือก 1 ครั้ง",
["Feature status"]="สถานะการทำงาน",
["Keyboard Shortcuts"]="ตั้งค่าปุ่มลัด",
["Open / close menu"]="เปิด / ปิดเมนู",
["Key already assigned"]="ปุ่มนี้ถูกใช้อยู่",
["Server Control"]="จัดการเซิร์ฟเวอร์",
["Live status"]="สถานะเซิร์ฟเวอร์",
["Rejoin Server"]="เข้าเซิร์ฟเวอร์เดิมใหม่",
["Save / Load Configurations"]="จัดการการตั้งค่า",
["Config name"]="ชื่อชุดการตั้งค่า",
["Config unavailable"]="บันทึกการตั้งค่าไม่ได้",
["Config saved"]="บันทึกแล้ว",
["Config loaded"]="โหลดแล้ว",
["Config failed"]="ทำรายการไม่สำเร็จ",
["Save config"]="บันทึกการตั้งค่า",
["Load config"]="โหลดการตั้งค่า",
["Stop all / Unload"]="หยุดทั้งหมดและปิดสคริปต์",
["Off"]="ปิด",
["Ready"]="พร้อม",
["Starting..."]="กำลังเริ่ม...",
["Starting"]="กำลังเริ่ม",
["No target"]="ยังไม่มีเป้าหมาย",
["Go near a shop first"]="ต้องเข้าใกล้ร้านก่อน",
["Refresh inventory first"]="โหลดรายการของก่อน",
["Select a crate first"]="เลือกลังในเกมก่อน",
["Idle"]="รอ",
["Stable"]="เสถียร",
["Measuring"]="กำลังวัด",
["Waiting for low stamina"]="รอสเตมิน่าต่ำกว่าเกณฑ์",
["No stamina consumable in carried inventory"]="ไม่มีของเติมสเตมิน่าในกระเป๋า",
["Off; uses carried consumables"]="ปิด — ใช้ของที่อยู่ในกระเป๋า",
["Select item and compatible cosmetic"]="เลือกของและสกินที่ใช้ร่วมกันได้",
["No owned cosmetics"]="ยังไม่มีสกิน",
["Select an item and owned cosmetic first"]="เลือกของและสกินก่อน",
["Cosmetic is incompatible with this item"]="สกินนี้ใช้กับของที่เลือกไม่ได้",
["Executor file APIs are unavailable"]="ตัวรันสคริปต์ไม่รองรับการบันทึกไฟล์"
}
local function thai(value) return thaiLabels[tostring(value)] or tostring(value) end
local thaiKeys={}
for key in pairs(thaiLabels) do thaiKeys[#thaiKeys+1]=key end
table.sort(thaiKeys,function(a,b) return #a>#b end)
local function thaiStatus(value)
    local text=tostring(value)
    for _,key in ipairs(thaiKeys) do
        text=text:gsub(key:gsub("(%W)","%%%1"),function() return thaiLabels[key] end)
    end
    return text
end
local hubLogo="rbxassetid://131572376338692"
local logoOK,logoError=pcall(function()
    assert(type(writefile)=="function" and type(getcustomasset)=="function","Custom assets unavailable")
    local path="MaxGanCombat_BlockSpin/maxgan_logo.jpg"
    writefile(path,"\255\216\255\224\000\016\074\070\073\070\000\001\001\000\000\001\000\001\000\000\255\237\000\132\080\104\111\116\111\115\104\111\112\032\051\046\048\000\056\066\073\077\004\004\000\000\000\000\000\104\028\002\040\000\098\070\066\077\068\048\097\048\048\048\097\050\049\048\050\048\048\048\048\052\052\052\099\048\048\048\048\100\097\056\052\048\048\048\048\055\049\057\051\048\048\048\048\053\052\097\054\048\048\048\048\102\056\098\054\048\048\048\048\050\051\102\054\048\048\048\048\049\056\048\054\048\049\048\048\050\102\048\057\048\049\048\048\049\048\049\050\048\049\048\048\055\052\055\099\048\049\048\048\000\255\219\000\132\000\005\006\006\011\008\011\011\011\011\011\013\011\011\011\013\014\014\013\013\014\014\015\013\014\014\014\013\015\016\016\016\017\017\016\016\016\016\015\019\018\019\015\016\017\019\020\020\019\017\019\022\022\022\019\022\021\021\022\025\022\025\022\022\018\001\005\005\005\010\007\010\008\009\009\008\011\008\010\008\011\010\010\009\009\010\010\012\009\010\009\010\009\012\013\011\010\011\011\010\011\013\012\011\011\008\011\011\012\012\012\013\013\012\012\013\010\011\010\013\012\013\013\012\019\020\019\019\019\156\255\194\000\017\008\004\190\004\101\003\001\034\000\002\017\001\003\017\001\255\196\000\236\000\000\002\002\003\001\001\000\000\000\000\000\000\000\000\000\000\000\001\002\003\004\005\006\007\008\001\001\000\003\001\001\001\000\000\000\000\000\000\000\000\000\000\000\001\002\003\004\005\006\016\000\001\003\003\003\003\004\002\002\003\001\001\000\003\000\000\001\000\002\003\004\017\018\005\016\032\019\033\048\020\034\049\050\051\064\035\080\021\065\096\066\052\036\128\160\017\000\002\002\002\002\002\001\003\005\001\001\001\001\001\000\000\000\001\002\017\003\016\018\033\004\049\032\019\048\065\020\034\050\064\080\081\097\096\128\113\018\000\001\002\005\002\005\005\000\001\003\002\004\007\000\000\000\000\001\002\016\017\032\033\049\018\048\064\065\081\097\113\034\050\080\096\129\003\019\112\145\066\098\082\130\161\177\128\144\160\176\192\193\209\019\000\002\001\003\002\006\002\001\005\001\001\001\001\000\000\000\000\001\017\016\033\049\032\065\081\097\113\129\161\177\048\145\240\064\193\209\225\241\080\096\128\160\255\218\000\012\003\001\000\002\017\003\017\000\000\001\241\128\000\000\000\000\000\000\000\000\000\024\152\000\000\000\000\008\098\096\000\000\000\000\000\000\000\000\000\000\000\002\006\000\000\000\002\096\000\000\000\009\160\026\096\000\000\000\000\000\000\000\000\000\033\137\128\033\128\002\096\008\098\096\000\128\024\033\136\026\001\137\128\000\008\026\096\008\096\000\152\009\128\000\000\000\000\000\000\000\000\000\000\000\000\123\055\140\251\049\227\003\064\000\000\000\000\000\000\052\000\192\076\004\000\208\048\067\064\048\000\001\049\013\000\208\012\000\076\000\064\208\052\003\000\000\001\003\064\012\001\048\000\001\003\019\016\048\000\067\001\052\048\004\192\019\004\048\016\003\064\000\012\000\076\001\000\003\000\077\003\019\004\208\208\013\052\052\192\000\004\003\064\196\013\052\049\048\000\016\012\076\000\000\000\000\000\000\000\098\000\061\155\198\125\152\241\148\208\000\000\000\000\000\049\003\001\013\013\000\211\064\192\019\064\048\000\019\001\000\003\004\048\019\064\000\013\000\049\012\019\001\000\000\196\000\000\003\004\000\003\019\001\052\196\000\012\004\208\000\012\004\192\067\064\052\000\000\000\012\019\004\000\048\019\000\076\019\001\048\016\003\076\004\003\004\000\196\048\067\067\076\016\000\192\067\004\048\067\000\001\013\012\076\000\000\000\001\251\047\141\123\041\227\000\000\000\000\000\000\012\076\004\012\000\077\000\000\208\052\003\016\049\003\016\049\048\004\049\048\064\013\003\064\013\048\000\016\000\000\000\052\192\076\001\000\000\208\049\048\019\001\003\016\000\003\000\000\004\012\064\000\000\192\001\052\000\000\000\000\049\003\019\001\048\064\000\003\001\000\052\049\052\192\064\192\000\064\000\052\012\016\192\000\000\076\000\000\000\000\000\000\031\178\248\207\179\030\050\152\033\130\098\001\160\024\009\130\096\000\000\128\098\024\000\000\038\128\000\006\000\000\000\008\104\001\160\024\000\000\008\104\006\128\006\000\000\000\038\128\001\137\128\000\038\032\000\001\128\000\000\002\000\026\096\152\000\002\096\154\001\166\000\000\000\152\000\129\166\038\129\128\002\006\008\104\006\038\000\000\000\154\000\001\166\000\000\000\008\098\096\000\000\000\000\000\012\061\151\198\189\148\241\148\208\196\003\076\077\003\000\019\000\000\016\048\004\192\000\004\208\193\013\048\019\000\000\000\064\012\000\000\000\000\001\000\208\192\000\000\000\001\000\048\064\192\000\002\064\050\035\004\048\000\019\000\000\076\032\152\000\033\128\032\000\000\098\024\000\000\000\009\160\006\000\000\002\000\105\160\105\128\000\152\000\128\001\166\002\096\152\009\166\008\006\152\000\000\000\000\000\000\015\217\124\107\217\079\025\077\000\000\000\193\013\012\000\016\048\076\019\000\000\001\013\000\000\048\000\076\001\013\000\000\049\048\076\019\064\000\000\003\064\193\013\052\003\036\004\172\137\117\182\098\061\141\183\106\094\230\219\180\086\111\037\102\142\123\178\237\060\182\242\179\078\246\206\103\082\182\196\053\016\221\006\138\059\210\179\207\199\160\043\094\118\061\021\121\180\043\119\026\180\198\194\170\176\203\097\068\070\084\008\131\000\064\000\000\211\000\000\000\064\052\208\192\000\067\016\000\000\003\076\000\000\000\016\000\012\000\000\000\000\000\000\000\001\135\178\248\215\178\158\050\154\006\129\160\024\152\038\008\000\000\096\129\128\038\008\104\000\006\000\154\000\098\000\000\000\006\000\038\128\000\006\033\128\014\072\157\247\098\203\097\149\171\083\145\181\125\012\043\178\022\179\025\078\122\077\101\214\076\099\207\033\213\067\201\114\198\150\085\139\098\079\050\080\193\089\237\108\003\060\179\095\013\138\134\180\216\035\093\028\232\035\009\101\214\140\101\147\004\099\198\232\214\042\167\037\085\174\199\220\188\163\157\143\069\070\045\025\178\195\197\074\146\204\134\160\003\019\016\000\000\000\193\048\004\208\000\000\000\003\000\067\064\048\067\004\048\067\004\192\004\192\000\000\000\000\097\236\190\053\236\167\140\049\000\048\076\016\193\000\000\000\192\000\067\064\000\000\000\196\048\067\064\012\064\000\000\048\067\064\000\003\001\202\200\075\035\059\102\183\051\054\125\074\172\145\209\102\074\087\070\082\177\052\206\219\019\076\242\044\136\162\204\139\161\139\060\187\106\194\158\109\179\108\057\229\213\008\140\173\091\118\086\212\151\070\085\043\043\148\029\150\089\131\029\138\150\170\157\189\070\170\173\165\017\093\117\121\245\089\133\094\100\038\216\166\066\165\042\118\202\114\170\023\035\095\129\190\134\078\102\061\038\015\052\233\204\154\057\237\001\148\032\032\000\003\004\000\000\000\000\000\048\067\004\000\000\000\196\012\067\064\052\003\004\192\000\000\000\001\251\047\141\123\041\227\032\128\000\104\026\024\128\006\152\000\033\161\160\000\000\024\000\009\160\000\026\000\024\154\096\008\098\096\057\221\009\102\230\116\048\178\175\143\090\066\123\218\083\082\180\185\147\139\019\149\137\170\217\100\166\155\050\110\070\045\249\087\214\049\046\183\019\038\068\114\122\220\028\062\119\168\103\224\243\205\231\081\060\169\168\203\204\170\021\056\070\203\099\081\105\117\183\017\175\213\117\115\079\157\106\125\110\022\183\141\217\233\092\166\237\005\054\173\216\180\109\233\189\117\021\108\169\083\095\028\216\070\120\146\189\105\076\120\228\087\164\085\094\069\072\170\046\011\067\019\038\056\223\081\141\208\099\114\105\166\050\177\249\080\109\102\016\064\000\000\000\098\024\000\000\032\000\000\026\096\000\009\161\160\024\000\000\000\000\000\003\246\095\026\246\083\198\083\067\000\016\193\000\012\001\003\016\049\003\064\003\000\076\004\013\052\049\000\211\000\004\192\007\043\035\044\188\222\137\195\206\155\236\178\100\186\073\088\093\091\153\002\216\093\007\105\109\108\094\100\045\094\076\242\019\012\137\099\103\025\120\187\047\065\227\142\031\179\222\083\133\044\049\210\050\076\056\196\103\207\002\180\236\049\241\165\116\149\080\178\227\030\054\101\089\132\204\251\053\146\163\113\012\075\243\049\086\046\107\163\149\222\097\141\234\124\238\174\066\155\039\189\112\107\218\083\166\090\234\243\171\189\053\240\204\170\245\197\167\051\030\241\143\078\077\051\166\060\108\140\105\008\206\025\222\168\090\096\213\227\239\049\121\090\178\234\185\137\053\153\136\024\000\000\000\008\000\024\000\002\006\000\038\000\152\000\000\000\000\000\000\063\101\241\175\101\060\100\064\196\193\048\077\048\000\000\004\000\000\192\000\067\000\019\067\077\003\000\000\104\114\078\121\218\078\046\199\041\119\220\077\117\088\145\045\040\156\157\166\044\038\002\110\033\220\175\164\187\204\170\218\057\054\100\210\208\177\087\132\209\221\110\118\060\148\200\133\024\149\166\077\120\177\209\145\077\117\221\100\012\148\098\215\178\081\056\179\202\195\044\076\041\086\091\102\028\115\160\156\040\230\070\201\108\053\185\249\069\241\070\070\224\165\039\137\108\159\001\232\108\242\156\222\155\139\232\173\248\249\245\116\229\173\163\097\141\173\053\244\103\098\218\049\107\186\169\181\052\228\085\093\040\132\235\173\200\204\165\161\054\086\106\196\207\050\104\161\187\214\241\087\024\107\008\000\128\032\104\006\134\000\000\000\152\008\006\000\000\000\000\000\000\000\000\015\217\124\107\217\079\025\077\000\208\196\003\064\012\004\000\000\000\048\000\064\208\000\208\049\012\000\099\146\201\191\105\217\173\086\091\015\071\090\199\045\107\001\186\213\073\202\104\164\057\153\014\118\136\217\043\082\178\094\077\108\242\225\147\073\149\081\219\241\171\244\090\206\106\095\135\133\043\212\171\058\115\009\019\132\045\193\197\150\243\006\014\038\204\204\059\097\159\081\044\226\138\118\020\089\143\100\196\192\184\137\199\163\038\155\197\089\048\152\221\072\201\179\018\218\148\111\132\077\078\212\101\106\179\175\172\121\014\103\162\121\135\068\095\143\124\123\105\131\139\157\143\106\224\081\157\141\044\074\050\168\137\198\118\198\047\008\217\008\188\084\148\020\102\090\209\082\041\109\126\014\255\000\023\143\061\058\190\158\058\033\170\000\006\008\098\006\032\024\002\001\128\000\000\000\000\000\000\000\003\246\095\026\246\083\198\083\000\076\000\001\003\064\000\000\000\000\196\196\000\000\000\196\192\025\126\136\237\050\175\239\232\169\089\015\067\161\054\090\176\114\043\149\114\148\171\069\039\043\068\037\041\161\218\237\181\139\093\243\043\040\201\164\168\179\146\185\222\129\143\141\205\158\086\170\217\222\013\156\158\105\226\098\099\089\180\198\196\170\236\168\022\150\184\089\008\089\027\033\110\102\030\070\081\147\042\236\206\099\041\198\178\169\145\102\061\057\056\218\027\165\218\045\170\202\019\145\145\175\217\084\163\041\210\107\178\044\179\031\047\022\034\204\059\242\101\227\217\189\239\154\244\218\236\124\252\126\186\107\177\182\024\151\097\209\151\076\177\107\201\170\038\136\219\092\094\036\145\023\038\154\227\108\034\099\023\012\233\029\126\201\097\158\145\108\112\056\080\025\145\000\000\000\000\000\052\013\012\016\196\012\000\000\000\000\000\061\155\198\125\152\241\148\208\196\192\001\048\016\000\000\012\016\193\048\016\208\000\013\049\179\059\105\167\106\236\244\181\156\161\062\205\197\039\107\193\200\173\019\115\138\066\078\201\152\074\109\072\077\078\034\219\235\201\180\059\214\084\012\181\129\207\052\250\086\175\055\151\050\186\177\054\174\199\105\165\218\100\202\166\088\180\107\088\247\130\115\213\085\186\159\017\141\077\061\026\126\109\182\136\236\172\045\174\114\145\058\067\178\139\105\107\226\240\162\096\177\214\204\232\226\205\022\097\114\218\089\215\209\237\243\044\212\122\005\156\247\091\157\085\179\088\171\175\042\155\008\185\203\006\224\150\103\039\210\228\214\124\099\039\059\011\190\113\113\118\056\157\053\215\211\153\070\140\106\114\169\070\050\189\078\152\177\200\174\147\001\165\148\028\115\165\113\178\057\210\042\106\042\177\242\165\155\073\094\251\081\195\024\195\050\148\004\000\000\000\000\000\000\006\038\038\000\000\000\000\007\179\120\215\178\158\050\154\001\128\000\134\008\096\134\000\000\000\134\128\001\131\146\154\216\234\134\196\126\158\162\107\175\073\093\069\182\189\162\107\177\078\068\213\211\049\156\157\098\010\106\184\171\021\209\156\178\161\150\151\149\012\172\152\214\107\189\003\150\012\050\187\229\102\030\076\236\216\095\143\044\164\197\078\098\097\196\196\154\173\135\087\026\114\025\221\229\213\175\009\159\215\074\181\044\170\230\115\190\187\041\016\141\209\076\176\178\241\098\216\228\214\177\024\205\028\150\007\160\070\052\243\204\095\076\115\111\046\220\117\124\110\086\245\028\159\029\245\220\105\099\138\066\172\133\149\206\020\235\023\229\234\179\042\094\077\236\220\082\252\238\020\237\244\218\220\124\252\125\088\116\229\213\102\045\121\085\222\212\087\117\115\106\033\145\094\106\035\100\049\202\002\121\214\042\201\068\169\201\194\048\181\086\186\092\062\151\077\198\194\026\229\000\196\000\000\000\000\000\048\076\000\000\000\000\031\178\248\215\178\158\050\152\000\002\001\166\000\152\002\026\096\008\006\000\004\137\025\186\158\194\185\122\058\072\140\187\037\201\075\077\073\164\181\238\022\046\090\172\210\242\155\156\222\010\197\150\049\114\051\197\221\028\150\082\204\171\047\059\074\172\189\062\051\190\221\079\026\185\066\186\227\173\114\236\131\205\116\168\156\035\147\069\240\194\212\244\197\086\206\140\089\141\161\163\114\223\089\167\216\082\185\089\017\191\024\086\213\101\078\049\182\179\078\062\110\021\208\025\164\087\027\107\035\065\173\210\217\210\211\070\246\235\051\121\141\239\052\243\027\125\229\120\077\216\153\120\200\138\178\055\156\125\070\070\031\068\093\155\167\140\199\089\145\164\219\115\091\202\174\232\249\094\187\188\077\142\039\092\235\233\204\199\189\113\170\202\162\234\043\190\171\090\184\206\017\106\169\200\142\117\198\148\158\084\036\204\206\081\156\033\093\244\204\040\201\051\210\227\116\122\127\054\112\198\176\145\053\000\000\000\026\096\000\154\006\000\000\000\015\217\124\107\217\079\025\004\048\004\048\067\000\001\000\003\064\000\048\009\025\058\165\159\091\239\189\133\082\232\078\117\217\172\216\068\190\150\010\118\208\202\140\167\091\039\093\154\094\201\171\007\027\085\041\084\219\207\023\151\094\069\051\187\046\140\188\162\152\235\251\078\116\245\217\090\237\048\191\035\031\049\051\142\077\016\110\044\185\050\169\056\225\015\159\208\209\026\236\046\217\102\209\137\185\227\241\173\095\092\216\242\093\102\056\200\157\217\103\139\102\077\085\189\024\089\120\219\069\109\187\086\056\217\122\041\211\015\071\163\201\182\241\123\250\038\199\071\230\185\083\030\182\249\221\174\121\109\101\164\217\231\025\176\050\179\190\135\081\212\243\061\081\139\141\102\055\076\103\117\092\103\081\205\057\094\087\234\188\005\108\097\100\071\182\218\220\124\236\077\099\026\140\138\045\021\099\223\094\147\093\115\140\204\084\212\020\045\051\082\089\012\136\114\165\108\170\078\213\170\171\169\154\017\153\156\104\104\232\052\190\117\168\026\231\148\004\000\000\001\137\137\166\000\000\000\000\253\151\198\189\148\241\148\208\196\012\064\193\003\064\208\012\064\000\054\173\186\205\148\103\232\076\009\061\173\092\165\043\021\144\149\192\074\247\179\042\019\190\246\016\178\218\074\216\218\074\213\101\173\038\074\034\036\236\165\035\147\085\185\231\118\094\044\177\166\015\162\242\061\038\052\214\225\202\157\112\201\221\106\119\185\203\198\201\198\155\087\084\171\180\075\042\156\218\177\252\239\183\224\179\211\059\214\106\220\114\204\039\161\243\234\087\213\185\110\075\209\038\124\211\211\121\173\254\185\109\050\105\183\060\108\132\097\019\010\172\175\090\162\082\153\195\243\063\072\209\199\070\087\085\163\224\049\191\177\199\201\061\022\181\179\204\061\135\083\172\248\247\160\249\231\097\215\027\188\237\123\099\187\187\014\092\250\102\115\189\014\028\207\037\070\203\095\219\022\117\092\223\077\130\253\014\234\088\219\203\054\122\094\131\178\218\236\061\150\030\180\193\199\204\198\179\022\140\140\125\102\184\204\149\101\142\177\092\173\082\133\119\070\182\199\147\149\043\018\113\103\085\119\087\008\019\034\043\171\032\197\205\215\186\212\121\215\130\107\025\000\000\000\001\128\000\000\000\000\253\151\198\189\148\241\148\193\000\013\048\077\000\000\052\013\003\009\088\247\021\103\119\041\141\145\235\178\147\157\166\014\068\196\092\100\059\229\059\220\114\118\186\186\054\174\093\093\171\219\098\178\215\036\074\078\250\175\205\151\002\236\232\181\155\174\119\012\251\044\235\245\177\134\013\106\058\211\101\176\212\228\209\152\097\229\149\014\041\201\203\198\205\164\112\187\141\023\113\134\189\006\022\111\007\205\030\101\060\121\237\191\164\245\222\069\186\203\063\069\175\153\233\173\134\214\204\075\103\002\002\154\014\042\211\151\090\116\182\157\221\201\199\079\154\202\185\233\211\233\221\031\144\100\229\159\179\100\241\253\125\115\227\057\126\255\000\207\122\093\180\038\186\121\241\239\134\117\109\126\078\159\039\043\229\096\108\171\137\162\201\097\204\061\159\055\208\039\205\182\146\194\210\214\224\236\176\055\207\011\007\097\135\121\195\163\043\030\243\083\037\025\183\053\057\193\074\051\049\007\107\039\041\086\042\175\038\020\138\203\037\020\199\134\069\104\162\055\066\182\134\191\099\044\047\201\172\204\079\059\068\153\156\160\000\024\009\128\000\000\000\063\101\241\175\101\060\101\012\019\004\208\000\000\000\012\000\147\206\198\223\245\196\161\105\223\090\085\170\209\009\073\077\128\118\180\050\039\108\204\036\057\153\072\157\044\230\236\155\215\106\182\116\157\209\178\111\017\147\105\221\085\181\140\140\154\050\051\173\186\013\254\179\151\062\231\065\188\231\181\195\028\170\090\211\058\236\092\136\155\115\117\249\212\102\098\229\211\138\205\142\028\179\158\071\114\239\199\076\158\115\109\174\171\129\179\107\172\223\072\226\236\068\117\189\086\062\093\057\178\045\175\038\121\218\186\089\198\053\121\053\089\027\075\098\248\124\127\115\173\174\254\035\145\178\199\219\166\170\165\183\164\117\123\093\014\126\116\159\039\216\235\116\174\214\184\100\116\225\080\070\243\118\061\216\179\108\188\238\098\248\158\134\188\027\242\085\179\213\108\013\079\037\222\249\220\219\117\129\176\194\232\207\007\019\059\022\236\044\124\170\047\052\077\182\114\028\152\213\027\154\106\149\179\154\211\043\156\069\005\240\136\174\022\194\181\162\156\156\117\170\117\066\052\181\210\215\092\231\075\133\205\125\010\156\060\237\016\021\000\192\000\000\000\000\007\236\190\053\236\167\140\166\134\038\038\032\000\006\000\152\078\057\122\051\115\043\126\157\108\156\039\181\024\057\168\154\012\152\101\094\228\157\178\166\057\053\038\169\146\207\089\219\083\094\025\019\133\119\204\150\022\085\162\197\106\155\023\087\124\091\035\038\140\138\085\097\228\199\150\157\063\059\210\115\061\028\184\176\146\214\050\114\177\178\107\057\057\184\059\012\217\055\217\111\051\023\097\069\217\206\108\235\158\051\059\104\140\069\250\235\097\053\225\113\189\027\026\109\175\204\197\204\215\011\051\049\114\163\055\040\203\051\082\156\090\153\202\081\104\107\054\250\166\156\117\125\181\214\190\052\233\045\089\150\072\196\167\051\030\202\049\183\058\237\105\173\190\111\073\198\192\157\091\205\024\185\024\247\061\206\171\101\157\174\219\227\102\115\151\150\122\191\148\069\186\028\124\172\126\156\176\177\115\241\045\026\218\051\240\239\104\032\190\118\073\089\158\053\185\217\020\039\057\178\170\083\138\035\011\101\044\106\051\169\137\193\175\038\169\211\095\078\094\035\088\184\073\163\157\098\117\056\029\006\143\205\214\148\206\073\004\192\000\000\000\000\007\236\190\053\236\167\140\166\134\032\096\134\000\000\001\059\037\182\170\254\195\149\115\234\139\237\199\187\090\049\019\089\202\025\054\078\200\205\105\078\010\086\168\056\176\004\095\026\234\169\174\253\134\126\178\093\054\134\137\229\225\109\138\147\088\200\174\250\173\202\166\252\216\147\132\185\179\235\057\126\199\147\219\151\092\178\169\222\029\244\093\012\221\182\163\105\148\237\231\133\145\207\048\179\026\081\059\059\181\166\115\176\167\093\084\198\201\099\229\078\113\177\074\049\172\155\152\047\174\220\234\229\007\085\182\087\101\110\052\235\103\009\068\174\187\227\121\192\170\222\107\109\186\059\184\140\235\233\212\083\173\190\041\183\166\019\206\184\248\059\012\077\090\026\051\048\123\107\143\069\241\188\221\124\047\206\118\153\184\025\156\204\207\033\246\047\029\173\250\106\050\040\234\195\015\019\051\022\209\137\143\147\086\145\136\076\181\101\108\114\041\132\108\118\214\137\217\010\210\021\202\054\133\117\016\090\250\213\075\170\231\009\156\108\045\142\053\175\174\046\174\053\136\201\180\113\242\140\111\205\071\055\011\202\184\006\096\076\000\000\000\001\251\047\141\123\041\227\003\067\000\077\048\004\049\048\200\167\105\209\025\048\148\123\076\068\204\174\161\233\025\047\030\221\107\109\212\139\100\078\137\077\175\117\076\189\044\058\095\097\044\044\228\195\089\184\196\155\102\096\099\223\165\222\231\089\183\164\230\017\178\052\178\250\239\170\235\233\187\058\227\099\230\235\185\171\232\092\215\085\167\183\047\041\092\241\251\035\038\202\045\050\118\058\188\170\054\247\097\207\037\138\152\076\217\139\095\027\093\037\141\126\014\087\235\189\003\075\184\174\050\105\215\152\182\055\214\038\229\044\171\083\152\041\194\075\201\133\110\038\145\010\217\123\225\113\029\255\000\003\166\254\125\147\137\234\123\206\171\117\201\245\026\087\125\147\135\155\203\158\062\014\223\018\103\065\129\182\214\118\215\021\202\122\047\221\106\119\024\204\173\163\043\006\071\141\250\231\146\214\253\061\086\213\215\134\046\046\102\021\235\069\089\052\218\152\146\186\106\066\226\200\205\074\082\154\066\054\066\034\152\206\165\165\129\147\139\094\154\142\183\103\029\222\115\149\135\149\126\088\043\097\166\024\180\102\211\019\140\175\173\173\106\081\155\099\232\250\061\103\159\109\096\227\193\102\034\012\000\000\000\031\178\248\215\178\158\050\154\001\128\134\009\161\184\217\102\070\194\051\244\032\019\188\166\057\033\198\079\038\139\247\088\146\180\219\042\103\022\186\220\123\034\111\218\232\059\252\244\213\243\062\187\203\180\229\171\223\081\104\215\219\174\219\218\235\046\055\077\167\058\236\172\095\117\055\082\111\186\172\138\066\230\250\142\091\028\253\103\085\176\196\175\055\019\141\157\133\219\022\095\069\246\139\103\083\134\194\220\075\051\174\101\022\074\147\131\070\246\204\173\095\152\250\174\037\052\242\255\000\071\243\028\233\183\175\229\120\255\000\100\199\183\200\209\236\240\166\089\077\148\164\132\161\041\066\080\145\024\175\051\011\010\214\216\224\113\156\030\183\234\184\094\159\083\125\058\174\254\220\220\178\230\108\217\155\081\079\010\132\109\012\012\202\223\023\089\183\197\214\052\134\223\015\117\059\045\102\124\051\050\240\051\057\231\011\206\059\014\118\039\111\069\181\117\227\143\139\149\070\148\162\023\194\051\162\198\051\115\082\085\184\201\014\022\087\089\162\171\107\090\173\006\251\073\061\125\086\071\053\157\215\217\127\051\013\135\055\061\176\145\126\106\225\100\081\093\089\074\111\129\012\218\167\092\085\116\051\191\059\078\227\081\227\194\104\196\192\000\000\001\251\047\140\251\049\227\032\134\000\038\000\003\202\199\219\116\068\201\071\186\088\156\132\212\154\039\100\172\139\213\048\002\065\019\101\053\117\180\215\091\221\227\173\247\204\175\026\142\137\179\083\155\196\231\091\182\218\252\236\089\121\152\185\110\146\218\178\232\118\201\210\151\229\067\042\148\171\155\234\052\184\231\209\236\057\094\139\060\053\092\239\111\202\245\103\137\149\133\147\188\078\113\033\125\184\247\068\100\221\137\153\147\042\250\239\198\040\183\022\073\227\233\239\188\171\061\123\014\115\113\216\077\188\187\119\219\243\212\140\157\175\009\170\061\111\035\198\146\061\158\030\056\085\234\090\046\111\107\051\173\213\250\062\245\062\089\213\238\184\219\180\221\063\045\236\249\198\085\115\121\103\131\143\176\213\237\092\060\123\042\232\172\243\245\059\122\222\020\099\229\196\213\128\099\109\017\206\134\076\205\155\093\118\203\158\124\234\122\110\138\247\040\191\027\171\042\234\148\038\145\174\250\216\210\228\050\026\038\174\202\172\206\103\137\151\163\175\069\185\150\107\090\100\233\186\076\009\174\134\141\228\052\157\110\218\057\051\090\041\178\149\138\243\097\106\080\174\134\182\166\156\154\237\174\037\121\053\197\241\052\061\062\163\205\174\168\148\124\224\005\064\000\000\253\151\198\125\152\241\129\161\128\000\000\078\204\141\157\082\244\034\078\047\105\026\114\016\088\236\140\237\032\023\073\169\033\202\019\133\042\250\023\221\111\184\030\179\125\246\116\213\137\188\236\052\057\249\249\087\069\155\169\219\115\050\114\176\178\039\092\171\105\178\151\207\158\045\212\100\229\096\228\082\153\120\121\080\207\062\095\210\060\171\212\049\206\090\125\230\022\153\114\017\216\087\213\020\202\218\230\037\145\139\176\077\057\120\114\206\187\011\245\185\084\130\218\090\050\242\112\179\240\183\148\122\006\235\201\241\215\214\182\028\079\107\089\207\158\053\249\210\081\182\080\169\218\021\169\210\099\235\246\094\127\123\093\195\217\235\151\091\125\110\105\117\216\206\171\245\153\181\090\186\108\093\158\039\068\107\055\015\022\214\147\187\026\026\217\230\095\105\203\195\205\134\069\012\190\050\045\206\116\186\205\174\246\134\046\110\030\185\226\198\216\094\035\009\070\112\035\036\202\034\113\085\039\174\207\092\250\116\157\171\191\119\199\116\208\215\163\130\179\177\209\050\209\110\052\155\202\241\215\161\216\111\242\190\030\203\210\185\188\183\242\125\174\094\155\163\044\154\237\175\167\026\225\056\107\173\112\156\019\143\141\115\197\206\209\180\214\120\144\032\196\000\000\007\179\120\207\179\030\048\048\067\067\076\012\156\125\150\241\149\018\062\132\077\162\214\024\134\041\090\108\105\233\044\020\196\220\109\035\056\206\098\072\156\078\022\235\091\076\105\187\231\167\145\023\198\150\116\149\142\215\095\185\086\172\141\246\068\231\161\187\125\085\116\214\091\126\045\117\187\043\015\039\059\102\202\188\140\227\154\219\079\150\198\190\163\075\174\216\235\245\221\030\167\106\227\227\102\074\211\078\098\117\140\042\179\177\111\021\093\077\232\201\205\212\238\041\085\042\097\086\110\078\177\229\110\095\067\234\025\088\105\171\234\188\231\157\206\125\197\248\157\135\180\175\026\196\061\163\147\224\119\242\229\122\062\246\115\025\084\215\147\106\197\021\202\210\140\160\140\241\010\047\138\208\245\089\086\076\213\050\194\038\074\170\019\008\153\121\015\125\195\233\166\251\046\022\110\167\019\046\139\215\014\023\083\124\227\092\225\057\001\025\204\009\214\152\058\109\215\053\094\158\251\047\137\223\117\118\237\171\215\029\147\119\045\189\211\243\082\057\216\179\225\228\213\122\079\005\209\223\171\165\198\085\235\125\039\033\215\242\057\083\110\173\171\126\074\234\182\189\047\012\092\140\072\136\141\069\241\244\029\046\143\202\140\064\056\064\016\000\031\178\248\207\179\030\050\128\104\006\004\172\218\224\231\118\195\113\151\084\041\069\205\164\145\034\200\089\173\155\139\188\185\014\033\203\127\212\103\135\157\089\235\217\017\143\142\199\217\176\143\041\167\187\228\173\209\137\145\085\186\116\079\058\143\073\203\151\011\103\166\243\202\241\250\102\023\150\017\167\169\230\249\002\079\183\083\229\253\242\042\142\247\075\094\236\140\205\086\118\125\050\229\186\237\110\081\145\191\243\111\074\103\078\054\109\087\166\020\114\233\152\180\200\194\148\249\232\226\239\027\028\189\030\229\027\011\106\142\085\178\085\216\087\026\172\043\200\170\186\186\061\167\011\118\019\218\226\104\114\051\182\235\035\067\090\122\013\102\169\221\110\069\217\008\147\170\021\095\089\116\011\106\164\110\178\211\044\098\203\077\035\129\110\110\014\214\138\149\138\140\104\207\157\209\196\238\180\157\047\070\247\037\013\034\052\060\123\065\069\149\095\058\161\040\078\105\074\044\229\058\101\025\154\189\205\116\114\187\104\081\167\094\238\024\124\255\000\094\157\100\185\043\249\169\149\159\135\176\199\013\055\162\121\212\167\163\187\191\151\134\215\150\130\141\158\089\229\168\174\142\120\086\070\218\066\139\113\226\145\072\141\165\129\153\095\053\121\229\109\094\064\002\128\000\246\111\025\246\099\198\080\192\000\106\253\025\214\074\062\140\003\142\134\211\075\017\105\156\227\045\172\208\089\151\233\212\237\057\184\054\220\183\015\168\165\122\092\029\150\194\052\208\244\090\238\093\030\189\149\226\158\165\124\057\093\103\173\240\246\232\232\177\247\158\083\024\105\058\202\251\124\123\115\023\022\181\211\182\143\031\040\107\185\127\091\226\041\159\077\111\009\215\244\069\153\218\188\236\187\118\021\015\023\063\159\110\138\049\244\153\104\051\039\044\154\091\074\214\109\232\189\249\021\149\030\138\098\229\095\133\052\223\079\133\196\173\061\010\222\003\162\150\226\055\083\157\219\139\133\175\087\201\205\125\009\121\108\148\245\237\143\150\117\152\223\172\140\114\240\218\168\184\076\198\019\215\090\155\089\240\026\043\087\212\107\242\042\244\167\177\199\199\242\229\234\057\156\022\254\183\219\065\091\158\185\057\152\214\101\054\225\033\043\202\122\029\078\183\217\230\018\232\136\194\037\235\143\093\212\047\010\229\093\172\234\186\134\113\142\126\254\156\124\206\119\075\008\231\209\098\244\211\136\243\238\115\214\185\088\232\228\115\115\115\230\218\213\182\218\198\060\238\087\109\146\167\153\106\125\139\021\111\026\183\212\057\171\109\205\229\150\109\164\019\142\214\174\187\042\090\020\093\143\090\003\027\036\057\141\102\175\160\208\120\181\064\115\072\004\015\102\241\175\101\060\100\064\192\147\206\193\219\116\045\027\244\080\141\145\153\140\153\088\082\140\244\179\098\210\095\071\206\122\117\121\250\015\035\239\124\179\014\126\171\210\240\120\186\245\246\088\252\206\015\070\157\119\069\230\121\025\215\091\171\246\175\009\199\047\107\207\243\159\064\219\135\003\197\189\023\137\207\163\214\188\247\182\243\221\251\039\075\151\169\169\125\042\099\113\218\121\183\165\249\089\120\207\127\194\117\204\115\179\181\057\174\221\149\218\238\119\062\109\159\045\174\244\042\099\231\245\251\038\084\079\139\238\125\027\131\154\244\251\239\018\244\155\095\115\169\220\249\057\151\173\232\125\063\042\240\027\077\222\049\202\242\126\201\138\158\087\175\241\191\083\214\118\122\205\191\147\169\143\208\223\232\089\078\131\031\058\071\029\204\123\040\159\063\244\207\025\235\044\244\253\118\031\146\103\061\063\031\141\233\027\071\156\245\062\133\141\088\208\217\179\073\209\233\123\123\017\228\054\123\079\057\013\055\121\226\155\219\079\176\083\175\205\203\107\181\246\249\220\105\141\180\214\114\061\024\111\117\061\111\087\090\249\030\103\171\107\014\103\121\200\098\109\029\181\086\085\126\151\153\143\210\070\089\186\124\079\053\199\143\162\210\239\059\058\237\229\027\015\071\212\068\226\246\062\057\186\190\030\139\164\233\045\140\112\213\190\086\211\162\210\233\123\072\223\159\205\232\244\213\157\231\077\225\187\253\049\239\248\095\078\229\109\060\146\031\095\116\041\201\199\077\052\223\093\105\090\148\038\192\023\180\185\222\135\079\230\087\000\015\044\000\063\101\241\159\102\060\096\001\137\202\123\173\110\215\190\000\059\165\066\112\137\109\016\044\174\119\179\009\093\015\100\242\047\099\203\135\139\210\102\223\148\236\249\124\253\119\127\106\156\095\084\187\169\148\058\204\029\079\166\248\216\248\183\170\120\215\167\095\155\157\216\114\222\165\149\249\158\114\250\189\062\166\211\235\150\164\007\109\198\116\028\084\231\239\220\114\188\184\238\242\113\150\218\103\240\029\022\078\025\119\049\163\071\077\118\056\250\106\186\244\233\119\124\062\118\057\081\201\123\095\136\243\101\218\113\079\111\123\122\231\051\147\166\181\167\008\093\213\092\222\147\145\218\115\087\142\196\245\143\010\194\222\145\230\119\117\086\175\164\104\243\117\246\165\106\211\166\138\218\240\241\185\193\122\031\148\103\166\254\015\218\051\156\061\101\120\155\068\241\113\095\106\068\035\120\186\236\056\213\178\204\208\100\243\223\167\241\079\096\227\121\027\158\203\201\118\183\181\089\152\154\201\158\123\215\120\238\239\058\067\083\078\143\179\093\252\185\172\139\219\188\243\157\191\091\199\143\150\117\094\123\222\116\087\109\061\106\172\115\121\028\231\176\115\182\124\214\142\142\141\246\217\060\222\094\214\237\252\123\189\201\226\199\089\219\248\175\167\107\201\133\231\157\006\070\122\250\079\057\079\055\110\142\190\092\110\070\211\222\120\175\125\135\207\076\222\195\205\061\070\220\094\083\029\134\187\171\189\197\171\235\077\119\086\138\043\186\137\132\209\050\240\243\105\228\142\116\156\060\104\000\160\246\111\025\246\099\198\064\006\165\118\199\055\027\039\210\134\145\213\034\156\164\165\056\200\083\083\165\114\110\035\101\233\188\015\119\159\015\156\116\124\159\113\157\249\108\105\195\209\234\004\186\036\156\037\049\127\160\249\206\223\205\141\126\171\214\252\063\139\041\251\095\150\119\178\225\134\123\027\141\059\196\164\164\172\051\240\229\132\118\222\085\236\254\033\231\101\221\223\135\103\086\124\079\175\121\055\176\241\107\197\227\091\087\171\180\228\167\122\172\170\044\164\116\218\043\250\223\038\158\027\153\135\077\239\236\250\012\172\045\041\152\230\250\113\141\241\178\149\233\060\219\179\207\243\244\240\111\160\060\227\209\235\058\037\108\123\249\090\177\076\083\082\216\227\209\195\114\116\119\249\105\220\235\054\122\085\033\077\171\191\044\104\228\213\164\210\163\069\173\145\026\174\155\070\068\169\121\209\145\175\224\183\041\188\213\237\244\174\119\003\215\115\245\159\090\208\244\156\114\216\180\088\122\118\170\187\160\189\157\063\039\176\227\097\224\122\007\144\243\097\215\100\224\203\163\030\071\221\124\079\217\248\247\243\071\058\253\141\165\040\171\051\251\191\058\218\112\071\035\211\239\060\219\159\012\222\159\134\244\008\181\026\109\206\147\183\102\199\217\107\122\094\087\175\243\050\226\061\067\203\061\042\188\092\190\147\127\207\237\214\070\113\219\162\168\206\018\174\156\138\102\043\002\240\033\115\052\084\108\117\222\037\080\024\015\102\241\159\102\060\100\000\182\188\157\155\064\094\154\082\140\246\151\032\181\098\135\105\177\166\214\045\168\116\189\167\039\212\099\193\229\126\141\230\062\161\158\156\040\047\091\164\066\210\089\017\018\178\153\102\245\079\041\233\246\158\046\026\252\078\175\207\052\154\221\086\250\251\012\017\099\132\148\156\100\081\233\126\019\237\094\065\228\103\216\235\118\220\135\070\061\070\231\059\155\207\163\013\194\126\181\238\158\044\171\076\198\173\173\114\058\206\059\109\197\028\046\143\210\124\223\158\249\190\157\228\126\128\173\249\216\217\093\116\178\074\201\198\091\237\006\086\051\208\115\217\154\252\096\105\244\228\234\191\079\022\201\228\187\047\028\195\166\094\253\227\254\183\157\117\052\221\014\206\122\220\150\136\106\103\186\174\220\237\248\220\213\039\170\196\199\234\175\109\037\123\155\033\119\152\117\028\039\061\187\088\095\118\217\115\093\135\157\123\135\062\218\029\051\171\208\184\157\027\172\141\086\204\161\070\150\239\188\107\213\056\239\047\038\070\158\156\105\244\142\039\166\229\223\144\117\207\218\213\145\104\115\130\206\125\023\201\250\253\143\147\159\152\122\119\152\250\161\161\213\231\235\251\180\153\019\170\078\191\144\235\060\236\184\143\073\243\159\068\175\046\147\157\232\116\027\116\145\148\119\232\170\187\106\181\163\078\069\051\090\137\005\080\157\120\049\244\251\205\039\143\156\090\124\101\236\222\053\236\167\140\129\039\153\137\159\208\203\065\233\076\229\092\180\091\042\028\173\080\157\166\199\007\109\036\018\136\236\119\186\189\134\062\127\149\122\159\149\122\189\054\243\210\018\245\058\154\069\128\153\039\001\107\253\115\199\187\047\063\030\147\202\250\254\058\242\167\092\187\111\099\173\162\201\082\069\047\043\081\061\191\007\215\233\124\172\176\249\233\110\017\235\062\111\219\112\219\106\056\047\066\114\186\204\060\063\059\012\011\242\240\186\102\217\211\051\177\241\047\111\242\031\058\151\105\122\253\038\177\234\058\142\047\210\041\106\167\174\203\237\198\249\084\043\122\174\085\074\080\041\004\169\211\229\167\037\163\029\118\244\141\254\059\174\051\138\143\110\018\199\183\018\045\147\198\117\190\073\205\186\178\209\189\051\149\150\166\207\018\156\122\205\125\143\047\213\219\044\013\174\095\045\053\179\212\185\156\172\054\211\211\058\253\109\025\141\213\225\158\022\147\169\227\235\055\193\087\215\190\223\059\155\238\124\188\188\202\221\123\180\119\139\055\069\155\002\036\125\125\164\235\017\055\091\153\191\211\060\195\165\243\171\231\062\179\163\232\120\115\227\240\046\198\245\244\153\008\237\108\158\163\154\232\124\220\185\063\067\224\123\216\228\209\232\183\122\125\186\227\093\245\111\211\085\055\211\107\058\108\169\017\078\041\174\187\097\141\107\210\116\026\079\047\044\096\056\042\123\055\140\251\052\060\105\004\165\180\214\109\250\166\066\061\011\049\026\006\154\178\186\139\239\118\219\157\009\004\087\185\204\166\252\124\255\000\044\245\015\048\245\006\220\000\163\233\246\077\034\224\133\116\094\234\148\172\183\028\050\043\129\041\040\145\004\246\250\218\082\042\014\243\107\169\214\058\041\096\111\252\204\060\199\184\225\061\095\054\191\159\182\143\071\105\197\202\241\221\249\135\166\120\223\147\135\176\115\061\119\017\219\108\217\211\046\173\058\062\126\237\239\153\151\007\212\240\221\150\152\121\247\160\234\057\170\219\210\174\204\213\239\108\183\141\045\039\034\088\238\034\252\028\137\103\025\254\051\218\241\252\237\215\053\234\030\093\105\246\204\013\150\170\098\197\084\122\169\126\044\243\112\211\146\227\119\218\044\052\217\250\038\195\014\180\220\100\114\017\214\221\126\039\049\051\047\207\253\003\117\157\124\199\158\189\217\235\220\183\091\197\105\122\226\067\209\210\174\163\089\172\243\050\190\122\030\220\229\145\031\083\170\094\141\230\221\175\014\094\099\070\247\067\207\030\177\201\119\062\113\120\146\173\122\122\216\235\038\100\224\172\178\116\075\039\170\097\106\251\079\019\015\029\138\171\216\218\200\037\165\182\251\076\040\249\156\248\029\247\013\222\185\057\109\118\118\191\094\235\041\178\173\250\171\162\250\111\104\086\220\069\081\156\098\035\025\170\068\053\091\157\111\006\090\160\126\086\075\217\188\107\217\097\227\045\018\183\111\171\221\118\094\178\075\182\232\106\228\053\048\239\170\237\044\228\074\247\021\144\165\123\203\043\183\047\051\204\061\063\203\189\066\055\243\244\143\079\181\137\089\046\239\135\244\254\046\111\044\058\222\079\163\098\081\054\214\073\041\135\007\182\199\062\235\156\217\195\131\147\134\034\122\125\211\149\118\069\114\251\110\015\208\124\206\095\031\246\095\032\245\188\030\126\224\123\061\022\230\097\109\114\174\199\204\253\131\091\229\227\149\203\122\007\029\209\124\059\170\179\209\214\238\219\135\234\252\252\252\147\184\230\059\058\113\235\184\173\167\083\019\214\104\118\186\093\122\155\165\245\090\247\142\235\092\140\237\109\153\069\222\107\237\254\081\197\158\239\207\183\122\093\030\213\166\219\115\247\095\026\099\215\173\251\077\030\110\017\197\232\122\222\075\150\190\223\200\116\156\198\214\085\163\208\214\046\049\189\238\222\242\251\030\076\181\080\207\221\249\248\217\205\101\235\061\061\178\041\139\218\221\143\153\250\175\059\228\227\193\122\231\053\212\067\207\135\015\095\169\244\060\230\094\089\236\248\079\083\242\127\051\159\220\124\175\213\124\167\167\102\162\122\058\202\185\230\086\048\031\077\205\013\143\077\051\250\174\022\204\043\102\058\090\218\073\016\234\053\155\222\083\202\227\217\119\156\071\115\028\188\054\038\094\062\254\129\093\145\223\170\020\228\087\107\098\049\084\170\186\154\212\025\021\048\051\241\185\051\231\216\120\252\231\178\248\215\178\195\198\064\150\110\239\087\186\238\232\197\134\084\122\181\198\050\085\216\198\081\051\077\242\158\214\131\176\181\163\011\234\165\123\092\188\029\142\094\039\148\250\119\154\122\085\118\243\233\041\122\126\130\076\180\191\078\243\015\083\243\184\180\220\067\233\038\121\119\007\233\246\202\034\014\215\134\245\014\030\094\055\170\243\095\085\227\199\204\026\151\177\222\073\057\079\211\252\179\212\188\206\031\048\238\185\189\142\078\113\197\123\061\182\229\224\202\035\169\205\226\236\194\058\109\046\059\188\219\058\039\164\221\187\208\102\115\103\046\083\210\252\151\205\226\217\251\038\151\014\154\224\087\067\245\250\178\037\068\238\153\092\139\165\076\140\174\155\147\219\121\217\121\078\031\168\121\150\024\250\255\000\063\179\213\244\221\168\047\067\166\203\113\108\202\187\175\039\246\255\000\047\243\057\050\122\127\048\200\189\125\011\015\071\233\026\223\135\170\116\250\061\171\101\171\221\225\150\191\208\056\062\243\203\229\243\172\123\232\246\123\163\040\171\219\115\191\225\214\053\237\185\237\076\018\037\029\237\036\079\058\122\079\139\251\079\139\248\220\062\207\229\254\153\230\061\157\000\215\119\068\122\190\083\181\231\195\109\193\238\118\184\115\112\174\047\208\244\027\081\180\180\016\077\020\175\125\231\254\153\229\094\095\007\101\219\113\189\140\243\112\021\066\205\253\069\093\176\215\166\032\077\177\163\056\085\010\238\140\068\084\221\021\085\155\070\025\242\209\178\191\031\136\246\111\026\246\090\060\100\029\155\141\214\159\113\232\116\215\022\186\055\104\149\165\052\090\101\040\221\174\145\038\094\197\022\087\158\125\102\223\158\232\115\240\252\183\208\184\062\226\150\225\172\148\125\079\074\050\029\239\031\084\242\239\081\243\057\060\107\212\188\219\105\142\024\047\166\230\189\078\213\027\161\166\153\221\126\046\159\204\243\121\031\092\242\063\095\206\222\102\228\253\127\070\045\202\200\122\095\156\122\007\153\193\166\213\116\060\130\240\026\245\187\193\217\074\070\087\203\042\085\044\188\114\046\019\219\105\219\069\213\175\121\229\190\152\252\031\055\039\140\183\003\209\237\154\140\187\122\044\178\141\166\088\085\071\085\160\166\088\238\183\209\213\102\078\013\153\199\103\228\221\190\203\201\226\228\246\060\023\168\107\078\110\018\175\213\244\164\235\083\027\110\171\129\204\225\230\199\212\247\153\252\121\106\113\163\167\234\210\170\156\123\250\107\233\121\158\199\147\155\081\187\211\046\076\053\021\078\030\191\161\024\201\068\066\054\060\233\088\235\075\025\181\193\170\215\210\060\151\210\056\095\023\205\244\127\055\244\047\062\236\235\067\125\221\053\119\156\055\121\197\135\021\232\126\107\183\225\225\192\197\238\056\159\083\177\070\113\223\161\054\162\072\206\188\233\233\190\067\234\254\093\229\249\158\135\210\233\246\086\199\206\178\113\242\122\061\146\054\195\077\107\026\189\241\227\040\213\021\053\090\214\074\188\226\218\165\012\169\204\087\117\062\055\001\236\190\053\236\185\188\101\167\102\243\107\172\217\250\093\080\132\233\215\089\078\153\222\243\136\239\054\217\093\151\217\034\019\080\174\085\199\121\214\113\061\173\124\143\057\234\116\089\053\166\191\015\121\166\244\125\104\200\122\236\122\079\155\119\124\028\124\038\139\174\229\121\121\125\051\065\203\250\038\147\198\103\246\111\070\111\144\236\180\188\249\203\217\252\111\216\034\254\106\073\251\030\172\028\158\150\151\111\201\117\030\087\152\113\061\255\000\007\174\209\110\094\143\116\058\142\103\185\226\229\122\222\007\095\195\193\233\153\126\077\235\246\191\031\049\250\222\171\156\037\107\245\025\156\153\197\202\171\071\095\092\167\093\214\154\186\253\102\063\155\228\243\125\255\000\140\122\142\056\225\044\172\079\079\214\146\013\122\039\211\242\150\113\242\114\253\186\225\060\239\047\176\193\234\185\111\079\210\141\118\067\175\162\182\021\139\039\065\009\193\147\053\198\117\197\109\223\229\112\190\095\155\216\104\122\190\058\215\130\154\245\125\042\182\152\029\111\039\044\171\226\057\223\063\206\245\139\060\147\214\102\220\082\103\171\233\136\118\183\077\161\216\199\200\242\250\046\007\209\060\243\171\173\059\023\119\077\125\223\015\221\112\243\121\254\191\101\167\243\248\253\147\206\247\025\253\087\228\098\229\232\250\049\038\166\097\040\237\121\176\234\060\239\179\229\188\223\047\209\148\105\209\200\090\159\087\177\098\132\175\122\193\218\216\240\148\109\041\074\188\170\087\039\149\034\228\101\028\206\046\086\047\141\192\123\047\141\123\046\113\227\045\059\055\251\045\102\127\165\210\233\112\218\233\197\077\174\183\020\188\229\188\041\218\111\085\202\105\050\050\180\203\212\060\163\213\242\243\248\141\039\089\231\213\231\244\014\095\179\226\123\189\073\053\062\190\184\117\124\183\079\199\205\141\205\246\092\095\023\155\110\131\183\198\138\242\082\234\110\071\041\216\213\143\017\206\123\007\149\119\141\121\087\147\141\233\250\098\149\118\190\242\253\247\001\229\249\125\239\013\217\114\219\245\099\043\097\233\122\021\122\031\157\250\047\159\231\249\022\235\157\244\030\095\063\137\244\158\059\101\023\049\183\122\015\083\213\155\133\155\108\089\188\215\231\150\028\153\175\073\100\054\089\103\208\249\047\123\230\158\095\139\126\087\101\231\168\245\254\111\050\061\126\142\056\227\219\233\058\238\222\243\114\091\227\094\135\193\249\222\087\095\211\249\039\103\117\181\245\026\094\254\252\005\116\119\234\168\153\050\144\235\090\179\054\185\092\188\024\190\089\145\030\046\095\092\227\058\028\061\186\053\081\145\233\122\075\185\225\059\126\015\063\203\055\124\191\169\115\113\121\231\127\202\239\235\058\040\229\226\250\190\180\009\206\246\217\155\126\083\203\243\059\239\058\244\191\055\233\232\139\111\191\186\190\235\134\239\188\254\063\058\198\183\173\229\226\225\061\099\201\058\249\106\031\085\202\122\062\139\038\183\233\171\175\228\123\222\062\046\070\124\223\101\199\230\116\122\237\199\051\109\181\224\250\189\074\220\171\153\110\183\105\166\004\070\153\000\174\188\107\108\098\115\215\159\196\202\197\241\248\087\179\120\207\179\082\060\101\167\051\188\206\214\103\122\027\184\144\218\242\141\081\180\078\080\118\089\058\174\189\165\040\206\210\236\118\091\076\063\074\243\174\195\062\075\188\235\187\231\227\207\233\249\109\103\160\091\175\141\058\088\117\116\243\093\229\112\231\231\209\235\116\125\118\060\186\126\115\059\019\061\108\066\090\057\181\074\238\163\157\233\052\115\151\125\139\231\123\075\091\175\218\240\152\145\061\231\151\067\113\157\103\233\158\066\230\125\002\173\054\094\183\233\117\060\247\057\157\031\087\131\129\025\245\092\118\222\104\235\237\242\172\155\109\233\247\121\109\022\118\027\223\040\178\145\234\218\206\119\127\174\209\233\181\028\196\083\075\183\231\123\026\099\171\205\226\187\099\067\234\190\077\100\223\208\045\209\087\167\071\103\169\225\244\217\097\103\097\169\076\242\121\157\150\238\177\173\238\060\215\017\127\097\194\243\141\198\183\234\141\035\180\244\057\028\070\146\149\237\120\058\119\185\085\103\043\102\154\111\082\242\188\006\158\175\133\198\102\105\126\219\152\210\234\099\060\206\141\113\113\159\113\169\221\113\021\122\231\059\204\239\244\232\158\198\141\108\207\083\228\215\153\231\232\116\112\061\077\239\075\233\242\182\219\157\233\049\124\255\000\028\048\187\222\098\020\199\063\065\215\113\241\127\093\226\048\187\221\182\224\087\115\070\251\243\157\067\243\142\126\090\123\125\102\194\152\118\254\119\222\121\237\250\178\034\235\234\239\112\113\154\196\112\180\211\042\227\049\117\033\154\016\155\198\177\146\120\199\059\141\145\143\227\242\175\102\241\159\102\205\227\032\075\101\177\213\108\187\174\224\150\250\178\069\165\038\089\044\138\114\111\163\153\043\218\115\132\175\164\118\090\234\051\231\244\014\031\209\188\231\063\055\031\085\208\100\082\117\089\056\152\166\086\171\109\179\154\234\055\213\241\246\133\151\078\101\059\110\171\184\162\116\228\035\191\212\089\175\235\057\140\104\230\234\245\024\091\006\056\247\223\068\054\182\115\182\091\078\139\077\110\226\243\197\199\191\192\172\114\251\200\233\107\077\247\036\242\151\171\166\209\226\069\186\253\126\163\162\154\097\101\231\081\123\181\173\215\214\035\139\209\101\198\124\150\079\077\021\115\184\242\115\174\027\233\172\158\133\166\218\230\078\028\116\250\042\107\158\054\211\077\167\070\251\158\150\084\107\141\145\147\109\181\219\079\095\146\207\019\003\170\169\094\082\091\231\090\233\182\198\166\035\163\211\232\165\022\133\151\228\206\187\234\117\219\091\070\155\031\191\201\187\131\223\238\180\144\209\105\058\140\090\095\011\164\229\084\099\102\055\075\123\062\075\047\164\198\067\206\229\049\013\214\159\113\177\137\228\103\213\078\103\149\216\110\241\084\217\240\171\033\166\079\085\196\102\066\154\122\234\145\163\121\210\052\123\141\174\050\153\219\222\011\214\038\117\092\174\094\187\079\067\038\185\195\175\085\016\178\049\148\108\198\139\081\087\042\227\085\208\139\198\210\078\056\211\155\162\218\188\094\085\236\222\051\236\212\120\200\018\203\219\233\183\157\183\164\156\122\117\064\230\070\229\121\050\041\202\211\072\218\091\125\103\037\116\095\015\018\221\255\000\054\025\219\079\053\235\115\228\227\049\061\023\147\182\017\201\231\099\102\255\000\087\135\041\152\202\219\154\157\022\135\109\058\119\209\212\029\054\203\224\122\142\087\041\116\102\070\213\214\085\181\084\174\175\171\205\203\191\077\240\171\165\166\188\209\220\172\233\197\079\161\208\090\216\188\119\095\118\212\225\178\047\149\240\135\067\153\007\069\209\093\055\061\185\083\188\150\085\225\047\233\057\187\219\077\201\247\181\235\159\158\100\221\218\071\062\046\206\172\169\237\162\174\219\059\010\121\229\157\182\152\231\180\123\185\239\111\060\125\191\063\060\184\091\252\204\171\244\100\071\023\164\207\077\001\220\091\134\124\020\187\014\126\243\166\227\251\075\182\175\158\075\099\151\078\093\119\085\176\193\191\102\123\167\125\156\243\213\119\089\248\083\205\103\233\214\083\063\047\061\066\169\159\052\126\129\171\190\188\166\062\227\087\188\232\116\030\137\092\229\231\050\220\097\229\203\136\242\228\140\008\103\085\090\226\187\148\069\047\034\105\175\034\051\141\163\141\122\087\027\033\084\203\107\029\083\086\114\187\167\154\231\244\060\246\182\186\234\055\029\022\202\221\124\044\169\197\218\217\200\093\144\161\058\110\161\039\020\032\227\072\039\085\153\218\117\217\143\133\121\216\184\248\156\207\217\124\107\217\106\241\144\101\219\238\127\127\217\120\146\059\181\078\069\201\185\218\209\206\197\207\191\075\115\086\222\049\079\147\159\051\180\211\207\078\108\190\103\105\199\214\058\029\191\023\110\084\200\231\125\019\159\211\046\106\087\026\239\093\147\157\230\184\100\066\147\180\171\087\012\163\103\135\070\069\151\183\046\139\067\117\175\237\039\077\102\044\183\153\111\218\100\193\121\126\011\078\044\039\101\004\105\194\232\189\035\203\189\031\117\195\058\190\201\202\187\091\218\114\091\171\138\094\111\128\134\078\037\149\074\054\243\072\116\188\135\171\239\043\114\181\125\091\238\061\039\156\223\248\158\058\036\233\230\215\108\072\071\204\253\055\147\219\213\230\042\175\099\235\250\149\024\157\087\061\058\140\197\015\051\192\113\109\207\028\172\103\093\056\109\031\162\249\143\163\239\195\062\218\250\245\197\235\120\239\099\224\231\145\081\195\226\056\202\054\205\013\204\068\100\213\073\042\218\250\145\027\113\252\167\176\105\122\061\094\031\158\217\108\123\250\184\163\101\129\060\117\066\248\214\180\070\232\209\084\164\103\059\047\079\241\204\190\091\247\254\113\223\115\023\141\058\171\035\085\051\200\233\103\060\109\190\195\207\035\002\116\091\094\174\247\035\071\155\211\052\241\215\079\059\101\039\030\235\024\247\099\090\042\066\207\052\130\042\173\132\235\119\135\151\175\228\174\149\051\200\192\246\095\025\246\106\188\100\002\123\238\127\121\215\108\168\202\062\158\202\113\201\188\198\203\029\246\175\034\018\157\111\028\218\227\235\183\116\097\157\027\126\120\114\110\176\181\023\231\108\248\229\095\166\149\118\060\063\067\078\062\115\015\208\120\221\152\018\045\219\120\066\244\140\072\229\172\162\155\037\043\073\038\172\232\118\038\043\077\119\107\196\122\063\036\237\152\184\060\009\036\076\000\068\211\228\158\171\229\093\094\230\109\055\098\250\125\107\208\060\247\190\224\195\163\105\240\248\012\139\156\192\022\211\249\199\167\249\095\119\208\108\048\114\042\237\236\245\076\185\215\225\252\220\136\185\231\105\129\200\118\062\121\111\067\158\203\196\201\247\061\186\058\110\115\103\197\135\165\070\200\121\191\058\008\156\218\004\203\199\253\147\200\058\061\188\204\085\095\177\233\236\061\071\129\239\060\047\021\017\116\242\088\152\002\072\039\050\154\104\000\020\224\086\252\231\017\236\062\081\213\238\102\241\189\078\063\163\167\048\166\171\207\076\108\141\042\137\145\106\104\202\163\153\217\244\220\094\246\102\122\060\237\165\163\135\200\237\103\156\121\213\093\174\020\057\172\109\198\190\045\143\147\043\230\216\027\088\093\209\017\083\171\168\177\114\113\038\099\009\067\060\132\132\074\081\042\158\171\103\167\225\166\008\143\046\175\217\124\103\217\161\227\032\015\107\170\207\232\157\169\041\122\251\078\216\091\174\177\178\055\078\144\114\149\175\124\201\091\098\022\168\182\154\189\209\207\134\187\035\053\166\022\005\163\015\087\127\161\242\099\160\219\234\053\241\204\096\246\058\253\220\235\178\157\181\104\052\038\003\148\114\034\123\061\038\235\064\211\011\213\252\151\216\184\179\176\079\143\196\000\064\009\058\191\052\244\015\063\236\247\051\049\050\177\187\186\087\105\198\117\124\121\246\098\094\127\207\072\137\106\072\066\212\249\007\178\248\215\095\187\149\145\139\184\236\236\244\186\103\095\143\243\018\113\118\205\184\145\054\249\071\171\248\206\190\164\201\235\125\127\091\107\011\053\248\103\237\180\089\079\147\243\211\034\091\025\033\038\223\048\244\206\011\079\083\067\091\171\218\246\187\126\183\158\223\248\062\000\004\121\204\136\155\043\183\205\029\158\141\111\152\229\095\079\065\052\251\138\115\004\073\198\073\162\206\035\177\231\231\209\225\247\028\254\235\216\245\121\172\110\139\065\094\042\099\106\137\169\074\048\142\062\069\092\205\191\083\200\116\151\072\198\206\232\174\147\113\149\092\052\061\046\053\016\216\111\120\189\190\014\039\051\083\185\173\224\165\087\077\093\082\170\214\142\037\216\241\070\156\043\064\068\037\024\198\177\102\151\109\164\243\171\006\159\017\123\055\141\123\044\060\100\000\202\197\179\087\084\229\063\107\106\230\229\174\133\132\237\164\092\165\165\231\100\046\157\146\154\093\074\082\139\215\041\188\226\186\237\196\231\195\081\234\254\101\233\249\099\127\003\217\093\103\142\230\122\196\241\207\202\169\245\030\003\070\027\133\157\146\134\094\022\195\003\109\070\243\073\180\213\223\092\015\103\241\175\104\243\240\166\080\057\124\089\145\017\033\017\110\091\137\235\185\046\239\127\034\137\071\179\075\058\029\014\235\150\059\250\237\167\207\249\230\034\114\147\137\023\183\198\061\163\198\186\125\153\245\060\151\115\213\211\212\164\188\223\158\147\066\141\197\038\126\047\235\254\059\213\236\236\244\123\188\014\254\189\150\167\103\172\173\253\162\088\025\190\071\129\034\018\115\181\023\009\114\093\095\061\111\067\132\170\120\254\215\183\234\219\060\060\159\019\231\164\069\184\216\036\219\224\030\253\226\122\251\056\015\125\015\071\186\159\100\241\031\106\243\124\217\168\153\249\150\033\033\106\054\218\073\236\243\173\142\187\043\217\246\183\092\111\123\194\103\197\008\217\092\205\117\217\093\074\171\042\194\051\058\206\103\042\045\185\195\196\205\222\037\166\233\085\235\086\187\127\164\179\127\199\239\120\222\073\201\216\225\229\093\026\157\119\164\226\161\042\107\112\173\101\017\086\160\148\004\213\102\189\054\199\093\230\066\003\152\253\151\198\125\152\241\144\067\148\093\157\054\126\139\160\246\111\005\053\209\123\101\011\045\101\098\179\077\139\107\182\219\146\078\186\074\073\198\146\174\124\231\053\050\041\234\122\174\126\061\046\206\168\116\197\241\166\029\022\178\170\107\181\113\056\204\184\112\205\179\140\186\042\201\043\218\059\221\015\075\076\204\060\156\093\118\197\246\063\030\246\031\059\150\137\213\062\127\030\068\008\074\085\217\026\113\124\199\073\205\246\251\214\184\046\221\237\217\105\182\188\121\122\093\054\083\193\243\236\069\179\105\041\156\175\026\246\063\029\215\215\163\209\252\211\212\183\215\108\065\113\248\118\149\136\145\002\045\131\229\126\149\231\029\158\214\092\168\218\244\233\174\194\201\166\218\250\070\231\157\232\060\175\022\078\169\071\036\200\004\244\219\109\115\187\204\233\157\126\183\181\236\115\117\249\031\063\105\088\231\153\002\045\119\141\123\015\145\109\236\230\044\189\199\119\094\175\211\040\094\103\147\037\017\195\036\146\108\226\123\063\035\215\208\118\194\062\175\163\214\112\189\207\005\135\027\131\140\218\010\076\162\140\170\112\110\122\014\019\166\194\115\241\052\153\074\211\173\236\051\237\028\118\199\163\192\079\043\069\025\133\226\135\102\046\008\037\092\160\081\027\035\146\002\117\133\017\073\180\168\192\196\178\175\040\052\100\126\203\227\062\204\120\195\000\002\089\253\119\013\217\247\204\235\157\126\157\237\187\026\233\189\214\081\043\107\101\152\242\191\070\068\171\035\091\037\085\141\037\206\244\058\220\041\062\131\139\175\046\046\240\209\103\247\087\054\024\113\232\182\084\113\029\035\079\078\239\067\229\223\040\148\122\077\053\162\158\171\147\235\178\203\018\137\213\190\213\250\247\145\122\215\153\199\081\003\031\038\106\033\057\210\069\150\038\107\142\156\035\057\077\240\242\154\138\100\083\018\121\166\235\038\179\080\012\223\031\245\255\000\033\191\173\129\235\030\075\235\155\235\148\086\249\188\091\008\040\137\168\053\180\220\015\115\195\118\123\111\123\163\232\186\045\205\048\190\189\191\071\203\117\030\079\146\156\092\112\201\068\068\177\173\174\058\252\158\085\091\233\251\094\203\070\070\039\153\225\076\173\219\009\186\221\089\244\227\021\223\099\143\142\073\202\181\108\173\085\178\119\046\106\189\024\222\127\108\253\015\079\043\022\172\253\239\212\240\061\183\019\159\044\163\053\058\065\178\085\211\124\049\173\221\047\024\177\182\211\077\109\213\166\014\195\019\125\022\218\115\125\215\006\174\022\109\057\091\231\024\217\061\179\199\119\016\166\187\169\181\144\021\181\021\100\209\154\044\083\087\093\184\156\243\174\065\229\128\042\126\203\227\062\204\120\192\208\192\147\233\249\125\183\076\244\085\202\175\107\073\186\137\181\243\196\149\183\207\084\147\174\084\241\174\157\109\178\153\175\109\114\113\056\152\059\172\094\108\053\157\007\055\188\219\154\101\026\125\163\105\147\203\028\147\219\115\120\054\229\125\164\047\175\162\208\003\072\167\174\228\058\250\101\171\170\202\181\213\122\207\146\122\215\157\195\141\042\165\151\151\050\004\076\200\008\153\007\100\200\004\136\162\100\008\077\068\036\235\102\119\145\250\207\145\095\212\194\245\175\034\245\141\054\202\117\153\120\214\016\113\073\168\017\166\151\136\238\120\062\191\110\254\155\155\233\186\028\156\044\166\251\118\189\047\063\189\242\124\169\017\028\050\034\166\028\010\035\175\203\046\199\191\187\218\246\108\060\188\030\015\010\068\029\249\166\064\090\100\020\044\080\009\149\132\236\160\137\216\241\093\038\109\058\188\099\047\211\060\199\191\212\175\164\210\245\026\198\147\158\156\043\140\211\086\178\140\163\036\154\202\181\227\101\101\243\078\014\110\085\026\083\011\023\103\134\170\199\121\072\178\228\250\048\136\200\136\177\166\170\110\174\211\001\198\151\088\214\085\082\076\129\173\216\234\056\045\088\028\000\001\251\047\141\123\041\227\003\067\064\059\169\122\059\008\096\101\251\058\206\017\135\069\236\183\027\037\165\197\070\155\095\147\131\147\055\201\158\059\095\042\088\214\069\175\173\149\157\022\207\027\003\147\146\114\123\008\154\042\218\215\181\181\121\119\205\019\174\200\105\106\097\058\175\090\123\030\059\179\207\159\079\076\225\174\149\122\223\144\122\239\159\199\140\235\121\249\146\032\018\112\069\133\114\036\069\140\065\037\001\019\032\018\034\165\176\242\047\090\242\041\245\112\253\075\203\189\023\093\183\004\076\188\091\010\200\137\197\036\224\249\223\166\249\119\071\181\177\233\249\142\147\166\252\206\030\118\186\116\244\093\166\038\071\151\228\088\235\039\146\197\002\098\088\025\186\106\246\249\245\212\203\183\216\246\140\044\172\030\047\010\078\183\110\105\184\011\073\095\084\105\018\114\155\086\089\074\146\034\040\200\005\250\252\140\234\117\121\014\238\172\015\079\216\158\183\178\227\233\204\069\008\081\113\170\125\079\025\232\060\250\106\186\188\057\107\058\094\115\179\227\169\069\012\108\189\121\171\147\134\152\201\167\048\014\048\077\066\033\064\141\100\166\116\215\072\169\198\038\000\068\211\168\204\194\242\108\001\128\000\061\155\198\125\152\241\134\128\096\000\075\063\119\204\239\189\011\074\050\175\190\242\200\198\201\155\217\022\107\178\190\153\090\249\083\174\086\210\118\227\202\047\147\118\029\203\172\124\163\056\133\150\084\137\070\014\208\200\147\089\198\049\138\198\171\043\152\163\177\227\250\204\249\245\112\149\026\206\055\176\120\215\177\112\243\226\056\058\121\083\032\034\086\083\040\182\086\162\028\013\059\189\002\030\127\094\247\244\092\159\051\232\243\142\227\026\252\088\224\153\021\106\077\068\054\062\063\235\126\063\111\093\246\156\111\065\182\253\188\045\199\230\241\044\081\045\073\017\012\207\035\245\207\061\143\083\027\117\160\218\250\029\154\172\011\242\243\183\166\099\102\224\112\120\142\080\052\231\154\128\100\114\253\095\001\159\165\163\141\212\250\030\143\176\098\100\098\112\120\146\034\237\131\149\100\090\239\063\237\188\113\233\117\115\229\094\218\250\007\091\226\157\182\056\117\241\200\196\159\062\066\045\082\081\141\109\170\225\189\031\205\122\125\174\139\005\236\245\167\036\145\157\072\073\101\056\125\167\045\212\047\178\150\158\094\149\182\180\234\185\094\026\231\095\174\216\103\206\070\070\252\192\224\137\192\008\064\132\076\170\112\164\040\202\052\177\025\019\117\093\248\024\219\089\091\094\085\128\040\000\015\102\241\159\102\060\096\001\137\130\104\123\109\078\078\205\204\108\167\213\188\175\199\178\247\184\131\215\073\202\183\165\242\047\196\186\250\217\042\156\233\125\152\210\172\229\058\039\019\125\078\138\069\198\187\050\171\084\171\212\068\074\036\005\061\071\051\210\103\207\129\139\176\214\234\193\246\047\031\245\206\060\041\034\083\201\145\018\098\078\005\109\135\230\190\167\230\083\236\214\243\177\187\183\163\169\231\186\142\092\186\220\123\105\203\197\145\002\212\146\074\039\097\228\030\185\228\115\235\221\099\215\245\244\251\046\013\148\121\254\045\142\038\184\073\068\133\147\167\035\046\143\042\187\210\244\250\250\124\023\165\093\151\158\020\194\038\190\091\017\048\226\040\157\167\146\122\087\149\199\173\153\141\147\141\221\217\235\120\214\227\249\222\036\200\154\097\036\136\155\124\143\215\252\121\236\102\215\110\015\078\247\109\053\254\139\207\134\227\006\202\163\200\146\011\213\168\186\090\062\097\232\222\107\062\198\211\168\228\123\014\136\243\217\194\202\081\039\010\076\101\085\089\218\205\239\055\101\219\092\008\101\041\059\083\232\231\064\148\068\092\085\198\085\148\166\171\048\065\072\034\212\006\228\209\104\119\028\255\000\159\162\003\138\064\000\000\246\111\025\246\099\198\080\012\016\211\067\104\150\230\205\110\207\210\178\182\167\211\107\157\114\190\150\056\061\047\109\148\075\077\050\093\083\190\146\149\110\038\217\082\234\201\203\209\123\255\000\145\205\202\243\254\201\079\046\127\057\100\250\247\141\118\244\100\196\094\150\137\010\243\030\131\159\221\231\133\186\110\131\157\210\180\122\159\150\122\103\038\119\052\087\199\105\019\018\072\139\093\229\062\183\228\179\236\108\049\178\241\251\246\163\176\227\251\126\076\247\085\202\024\248\192\043\084\066\137\201\242\175\078\243\027\123\025\218\236\234\059\122\058\174\151\203\253\087\206\243\049\196\091\133\136\152\100\092\204\172\168\173\154\021\171\034\036\213\138\049\051\200\197\202\203\126\063\149\046\233\245\047\196\182\158\141\189\078\053\075\205\241\230\068\215\153\161\039\103\194\245\054\231\213\205\230\237\224\214\199\142\173\205\032\087\193\136\145\149\137\147\142\220\157\220\087\091\143\173\203\246\186\189\110\177\165\186\138\109\076\232\227\206\209\108\010\114\177\014\203\027\011\243\119\214\250\176\184\169\245\101\097\089\021\176\172\138\217\093\113\138\036\070\166\034\000\164\153\074\052\211\077\126\186\112\242\174\208\101\032\000\000\123\055\141\123\041\227\003\067\000\000\000\009\109\181\025\091\182\085\221\143\232\105\060\140\124\139\222\064\181\188\231\076\180\153\095\139\117\180\185\198\086\187\077\103\055\125\009\243\215\208\062\055\038\014\159\077\095\107\178\209\107\123\108\030\007\149\083\239\233\146\071\069\163\182\212\236\169\150\247\144\237\120\137\201\247\220\015\083\139\168\045\163\063\034\066\141\171\040\167\019\159\228\126\181\228\143\087\101\068\233\239\232\135\115\194\118\220\153\238\034\227\151\142\052\173\071\023\058\223\023\206\187\110\039\079\088\149\088\125\058\219\221\113\199\054\094\185\133\139\178\195\206\199\081\122\243\072\136\153\017\009\017\036\210\080\154\134\093\045\119\031\183\243\168\239\163\046\205\079\103\102\196\133\150\183\161\228\233\055\156\030\107\073\109\197\034\033\034\033\036\041\150\033\018\072\137\109\102\082\235\141\216\112\052\237\197\233\249\237\204\117\110\243\113\173\238\182\243\018\173\086\113\126\155\125\143\155\147\232\118\087\099\058\188\078\082\140\171\178\199\166\205\242\181\192\218\146\117\057\089\021\083\039\040\075\058\000\033\052\014\106\115\049\212\236\249\238\029\034\131\134\192\016\000\000\007\236\190\051\236\199\140\166\134\000\038\000\001\040\146\220\217\172\219\119\233\027\098\117\093\136\210\238\081\122\075\178\171\038\215\202\169\223\075\001\197\240\061\227\194\059\063\043\155\105\141\215\235\058\179\211\119\122\141\070\046\000\193\206\232\233\152\046\187\043\241\227\090\119\124\007\160\249\245\057\239\201\163\022\218\251\014\047\037\210\113\121\247\169\206\248\083\145\094\029\111\111\154\236\053\187\250\057\088\241\193\232\215\101\180\192\196\206\061\123\015\156\222\114\121\247\009\206\054\101\105\249\232\219\087\137\135\159\219\223\133\133\153\189\138\233\225\175\219\166\125\167\159\075\012\253\119\031\145\233\051\227\189\229\074\216\097\044\152\218\041\050\046\048\101\149\175\207\076\237\014\155\152\158\156\168\227\229\117\117\215\143\143\216\086\057\236\156\076\205\045\147\233\062\061\219\242\243\239\149\183\056\049\012\184\077\113\203\066\165\145\051\016\206\141\103\019\043\023\077\077\186\046\103\067\162\215\167\036\085\233\209\094\195\077\183\230\141\245\088\186\238\167\089\174\194\191\089\221\096\229\225\221\149\204\245\094\123\199\017\176\187\044\106\148\163\164\052\146\237\165\053\032\149\114\144\156\065\040\053\100\039\049\057\066\172\231\007\095\056\121\154\000\100\000\000\000\001\251\047\141\123\041\227\041\160\024\002\096\008\104\100\182\186\139\182\157\228\071\232\234\154\123\093\184\202\242\220\093\173\043\043\118\190\068\170\047\115\019\046\190\090\239\246\220\045\060\152\246\028\197\089\059\090\089\080\159\102\211\136\174\141\023\209\090\246\154\011\119\057\225\196\101\096\093\054\175\101\070\060\055\150\242\238\043\211\097\234\050\100\178\163\173\211\074\033\153\181\202\180\087\171\217\077\168\216\083\066\187\083\076\043\178\192\183\034\210\176\165\066\219\190\167\023\038\051\228\180\062\143\200\214\213\199\089\153\105\134\198\184\077\119\025\188\141\121\087\187\179\130\146\059\186\120\154\206\179\081\174\204\189\176\242\173\192\182\153\026\248\116\089\211\035\163\170\169\166\131\087\215\240\147\125\157\081\202\181\182\025\252\165\121\103\221\095\193\056\175\160\028\003\135\121\079\017\093\157\158\023\042\226\119\026\203\174\210\250\252\181\172\153\202\162\029\054\113\141\071\069\162\231\116\055\209\197\100\236\177\057\156\251\087\051\105\173\204\209\087\041\125\069\242\070\244\112\146\137\073\144\084\202\168\172\220\036\163\148\090\160\018\114\173\197\103\169\203\213\112\221\001\199\032\156\000\000\000\104\031\178\248\215\178\158\050\154\024\032\006\033\130\024\013\018\217\108\052\027\126\203\095\025\199\191\102\227\043\203\105\205\219\074\102\201\099\073\025\004\045\189\163\027\156\090\187\066\214\153\007\105\154\004\058\236\085\138\123\142\035\035\044\182\156\143\161\234\034\188\150\093\152\080\207\122\181\054\218\226\226\219\090\213\147\176\232\166\035\109\188\206\049\135\175\239\240\034\220\118\070\126\021\146\048\231\100\233\200\216\217\174\234\239\205\177\240\153\088\241\027\157\247\013\177\050\052\029\246\028\207\009\145\184\212\153\054\106\068\238\094\148\155\110\035\168\081\093\174\046\054\093\099\022\237\214\229\026\253\228\121\053\111\211\145\207\063\069\208\235\059\029\175\231\039\089\160\093\093\170\138\119\111\069\041\157\210\211\009\220\026\100\109\225\171\033\155\139\110\122\053\123\013\254\210\105\174\217\224\242\081\075\243\244\043\158\221\247\025\075\206\216\217\081\149\099\211\184\077\166\130\103\095\125\087\107\073\068\091\213\197\168\130\014\154\164\066\116\130\081\115\086\211\185\038\042\164\096\224\198\165\175\054\192\138\006\000\000\000\000\015\217\124\107\217\079\024\026\000\001\166\008\006\000\000\060\140\103\118\253\235\246\094\150\209\097\213\163\016\017\129\072\144\156\069\183\099\217\174\151\202\022\105\172\148\213\173\022\008\109\089\018\022\019\052\227\102\087\072\167\166\229\169\203\030\223\019\159\216\205\047\170\244\044\221\118\052\087\168\210\115\184\249\198\247\162\224\250\123\078\246\056\145\239\135\171\204\209\115\171\221\104\246\028\147\184\200\230\240\183\158\139\159\142\066\245\219\108\173\108\108\125\132\012\029\238\175\026\153\118\053\241\217\176\221\099\079\046\035\079\030\146\055\104\045\223\168\174\179\050\092\230\118\233\244\124\240\206\082\149\211\072\171\145\139\144\082\117\215\240\153\205\058\012\044\220\141\045\161\171\167\146\121\067\171\148\185\075\122\106\209\164\204\187\022\025\217\060\190\174\043\214\104\117\150\069\084\231\034\052\101\068\195\190\114\090\172\124\186\169\106\011\138\148\194\249\128\076\144\149\082\138\102\113\025\197\214\141\050\197\034\087\152\076\034\181\106\167\079\151\032\024\019\000\000\000\000\000\001\251\047\141\123\041\227\040\096\032\006\008\096\000\000\000\004\246\250\091\250\039\123\005\047\075\104\192\067\099\128\227\033\202\018\181\175\187\018\237\182\203\080\122\111\040\078\100\045\009\153\145\038\072\182\138\169\202\084\140\040\230\042\101\130\179\072\174\001\157\076\103\082\185\068\097\236\241\050\041\027\204\120\105\059\091\250\053\249\124\209\171\201\134\117\052\198\178\215\209\117\108\039\051\050\037\146\138\017\026\237\084\166\053\057\112\206\152\120\155\092\046\120\232\158\143\184\202\156\238\207\121\168\189\117\188\196\172\202\210\176\151\094\096\213\170\220\026\090\098\041\195\217\099\103\049\166\080\206\249\248\208\144\075\045\235\150\012\115\107\180\099\071\036\078\060\237\147\074\229\039\105\077\022\036\226\179\072\139\129\010\192\069\068\077\033\008\066\084\068\032\084\162\113\117\137\033\074\233\087\041\151\175\179\095\199\068\133\196\096\064\000\000\000\000\000\000\126\203\227\094\202\120\202\104\104\006\152\000\000\000\000\000\038\018\203\217\105\051\122\045\157\027\014\253\171\109\204\198\072\067\144\230\091\129\166\153\121\026\252\253\053\149\149\061\119\186\002\068\162\212\193\058\230\072\137\006\069\019\138\115\083\023\042\138\231\090\146\166\117\211\147\139\133\098\172\157\034\151\144\088\243\112\114\118\189\202\047\073\148\234\153\034\010\019\032\074\216\197\197\035\016\066\170\232\082\184\020\103\226\114\214\049\177\210\174\232\079\104\155\137\120\177\037\104\146\066\025\022\077\194\074\142\121\019\076\043\178\034\132\016\178\053\184\077\162\136\215\089\040\053\165\040\074\109\042\231\084\195\131\133\109\039\092\226\196\037\026\194\007\016\130\033\020\073\136\145\025\070\149\136\025\213\137\146\116\224\225\016\128\112\064\135\080\000\000\000\000\000\000\000\063\101\241\159\102\060\101\052\000\003\064\193\012\064\196\012\064\192\009\068\150\211\059\067\180\237\215\037\053\217\179\002\208\229\093\151\180\097\040\068\206\252\121\203\050\204\059\117\222\242\046\215\077\074\082\148\088\216\129\050\073\128\211\112\197\083\166\185\078\033\090\068\106\032\113\083\071\040\053\178\028\029\146\112\021\147\174\117\041\193\146\034\009\168\171\036\163\017\010\136\101\073\149\149\173\133\114\148\212\073\173\142\169\089\053\025\072\019\039\053\145\057\219\056\150\161\092\171\155\198\178\009\148\035\021\220\101\024\186\113\154\101\100\084\218\085\056\037\194\200\082\099\056\020\100\227\010\176\148\097\021\156\098\230\024\061\101\001\097\025\067\024\073\152\209\074\056\153\068\041\071\012\009\153\147\000\000\000\000\000\000\000\000\015\102\241\159\102\060\101\012\064\003\016\052\013\000\000\013\048\000\004\018\178\162\205\213\250\061\199\118\243\036\187\117\077\022\152\164\235\089\161\034\219\049\236\190\151\217\140\244\211\042\084\089\054\156\170\038\247\186\130\199\092\134\066\049\022\207\030\115\038\061\212\087\048\020\081\197\070\176\196\070\110\080\037\144\234\148\196\136\019\051\157\078\083\020\073\016\034\179\130\072\156\085\117\066\009\101\155\034\171\075\010\194\215\091\090\082\139\189\092\162\090\211\178\022\090\150\228\099\091\106\093\010\210\206\162\019\101\020\072\137\024\188\212\090\202\113\154\214\197\194\116\086\070\107\060\092\156\108\226\032\179\200\129\084\068\171\004\073\167\041\008\210\065\198\209\040\074\057\068\073\227\242\231\078\050\092\016\002\200\052\012\016\193\013\000\192\000\001\160\000\015\102\241\159\102\060\101\052\000\003\076\019\064\000\052\192\000\004\052\208\052\014\250\011\078\246\221\030\215\187\091\161\058\250\108\135\025\153\069\008\118\213\041\181\169\173\108\238\199\045\108\169\098\206\045\146\080\230\217\038\052\139\010\090\183\021\146\178\183\008\171\140\072\128\110\008\146\152\136\022\057\064\082\100\084\214\110\185\022\040\131\034\086\101\020\064\174\117\069\107\106\056\209\161\086\163\096\001\041\202\185\089\055\011\116\153\217\025\237\003\074\214\106\040\113\032\052\074\047\021\057\086\240\148\236\182\144\045\141\180\170\051\133\032\112\017\024\149\229\140\160\150\085\073\164\032\044\028\092\164\034\231\022\166\002\120\188\117\142\017\031\062\128\025\000\032\152\002\104\096\000\000\000\000\000\000\001\236\222\053\236\167\140\128\000\134\038\032\098\024\000\000\000\154\001\130\026\024\003\178\183\102\222\122\125\151\093\236\028\122\052\098\118\150\004\175\117\079\105\066\136\229\090\162\247\067\133\170\183\041\206\128\201\049\221\153\017\166\114\147\128\139\028\029\146\117\151\134\068\076\146\034\178\034\225\033\057\049\041\048\080\105\021\151\092\225\017\091\070\084\067\034\162\096\003\088\178\022\092\236\140\182\076\079\089\105\197\104\197\165\153\038\178\181\219\058\085\043\082\240\008\180\113\081\066\131\133\115\033\008\101\153\020\103\152\156\107\086\147\144\034\208\196\073\137\089\034\205\127\036\079\004\094\110\096\020\000\064\000\000\016\192\076\000\000\001\160\000\000\001\251\047\141\123\041\227\032\134\008\104\098\006\033\128\000\002\026\000\096\009\128\000\000\019\129\045\134\094\155\047\162\249\194\125\087\106\037\142\202\222\150\178\153\215\041\036\082\026\020\064\216\137\074\178\086\149\202\233\052\090\036\211\155\014\046\201\017\044\106\046\012\139\132\156\037\017\033\059\075\112\119\150\032\024\085\024\078\021\136\131\206\172\016\128\147\099\153\118\041\234\009\045\100\146\119\184\052\180\091\045\114\072\077\164\020\077\176\138\128\068\137\148\084\034\010\092\041\156\035\040\227\147\073\082\026\020\024\156\152\154\004\059\018\150\179\150\150\227\011\207\168\006\100\192\000\000\000\076\000\000\000\000\000\000\000\000\001\251\047\141\123\041\227\041\160\000\006\128\024\128\001\160\000\000\024\152\152\008\001\130\024\000\209\044\156\253\069\155\078\205\087\111\077\163\042\222\183\182\049\044\105\058\212\026\138\183\009\009\055\048\154\086\155\108\162\203\204\167\009\092\196\237\118\196\038\021\038\156\047\161\202\102\038\093\054\181\076\036\154\021\148\084\034\036\009\000\200\148\164\160\211\149\132\211\210\211\065\160\148\094\182\152\165\107\002\081\118\148\098\036\068\136\156\099\024\088\085\024\078\048\085\173\138\002\008\184\231\084\156\113\170\002\160\064\008\068\156\101\156\075\026\056\092\180\112\014\072\000\168\000\076\001\048\064\196\000\192\004\192\000\000\000\000\000\000\126\203\227\094\202\120\192\208\012\016\048\077\003\064\013\000\000\000\000\000\196\052\000\003\000\004\012\009\101\225\061\027\053\133\151\209\105\129\169\176\152\148\068\050\050\176\105\218\098\076\144\193\050\066\180\202\116\202\214\181\212\203\028\026\044\065\107\070\104\153\200\170\169\076\180\066\033\164\086\173\164\134\014\072\005\134\137\057\041\094\036\039\165\152\022\000\175\105\058\221\166\113\081\137\154\174\053\173\202\149\074\217\092\084\068\156\101\050\134\076\136\085\171\131\175\040\139\137\148\076\136\171\019\131\021\057\077\216\053\195\135\054\131\016\154\128\048\067\001\012\067\064\012\064\196\192\000\000\000\000\000\000\000\007\236\190\053\236\167\140\130\001\128\008\105\128\000\134\129\136\096\008\096\038\002\024\002\026\001\136\006\128\096\018\137\044\188\189\077\187\054\112\166\238\131\035\059\016\059\204\089\043\072\000\132\089\034\050\024\019\118\129\086\210\181\166\235\105\156\169\101\146\164\149\164\008\073\164\137\017\006\210\036\068\180\147\139\180\206\085\022\088\084\090\108\042\037\106\168\148\163\022\137\144\017\053\018\012\138\131\064\018\132\137\145\045\050\139\133\080\077\097\004\036\101\017\144\086\003\031\023\010\221\066\057\042\002\161\137\137\136\096\000\152\128\000\000\024\000\000\000\002\098\024\152\000\000\000\003\246\095\026\246\083\198\080\012\016\192\000\067\000\000\000\000\000\019\001\048\000\001\012\016\192\000\000\000\000\026\007\117\005\155\027\245\023\110\216\021\093\184\097\180\145\100\162\073\076\161\131\105\164\028\108\114\129\006\160\137\145\100\156\037\054\147\137\100\132\137\036\161\040\180\019\140\172\000\188\183\025\092\033\000\036\002\113\045\010\082\034\161\033\048\132\227\083\064\132\199\050\192\177\036\242\037\100\112\172\074\176\185\169\147\141\005\205\000\025\128\080\096\000\000\000\000\000\134\000\002\024\033\128\000\000\000\000\000\000\000\000\000\253\151\198\189\148\241\144\000\000\000\000\004\208\192\000\000\001\048\019\000\000\000\000\067\000\000\000\000\076\019\000\000\002\082\186\133\102\210\237\061\187\054\113\171\039\170\106\115\134\146\137\022\145\074\054\168\148\041\051\138\032\200\177\180\228\228\157\165\130\185\136\016\016\096\229\032\047\112\078\192\020\152\129\137\009\162\161\137\085\036\235\044\026\082\145\008\169\169\070\081\145\024\201\229\080\195\197\227\174\110\037\107\154\173\053\152\077\064\098\026\096\038\000\000\000\000\009\130\096\000\000\000\000\002\024\000\000\000\000\000\000\000\253\151\198\189\148\241\148\192\000\076\000\000\000\000\000\000\000\000\000\000\004\192\000\000\000\000\000\076\004\000\048\000\000\000\000\044\172\179\059\035\084\244\110\077\078\071\075\050\013\237\106\212\227\050\129\160\027\146\097\038\034\230\038\150\130\064\048\144\104\001\076\200\078\097\008\024\033\161\144\108\165\146\148\034\028\163\040\134\132\180\152\170\068\049\113\140\234\245\176\229\166\078\060\078\120\000\168\076\128\000\032\001\161\128\000\000\000\000\009\137\128\000\000\000\000\000\000\000\000\000\000\000\000\000\001\236\222\053\236\167\140\128\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\003\064\000\073\217\081\102\101\186\243\070\210\205\068\244\108\140\073\237\057\037\118\221\002\069\166\045\171\004\070\082\156\108\210\082\101\192\003\034\228\132\086\204\026\163\010\146\035\068\163\085\121\206\089\175\171\026\108\233\192\088\178\241\235\050\134\130\128\010\128\000\000\104\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\104\003\217\188\103\217\143\025\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\036\000\049\003\148\009\093\060\103\102\092\240\011\182\015\094\076\236\231\169\119\141\177\169\046\219\026\145\059\101\170\072\218\045\098\164\236\086\188\163\054\188\098\171\099\002\176\196\084\001\000\008\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\049\123\055\141\123\041\227\032\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\004\154\000\000\000\000\032\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\000\003\076\061\151\198\189\148\255\218\000\008\001\001\000\001\005\002\255\000\250\218\178\196\172\010\233\149\210\093\037\210\093\037\210\093\037\210\093\037\211\043\166\086\037\098\173\255\000\111\138\233\149\210\093\037\128\088\171\111\101\101\101\101\101\109\236\172\172\172\172\172\176\093\048\186\065\116\151\076\171\127\214\226\132\107\164\176\028\108\172\172\172\172\172\128\086\088\172\085\149\149\149\149\149\149\149\149\149\185\116\194\049\035\026\183\253\045\144\098\233\032\192\173\189\149\149\149\149\149\149\149\149\150\043\021\138\197\089\089\098\177\088\172\086\043\021\101\138\178\178\178\178\178\183\027\035\018\049\044\127\232\003\080\141\006\219\193\101\101\101\101\101\101\100\026\177\088\172\086\043\021\138\197\098\172\172\172\177\088\172\086\043\021\101\101\101\109\173\202\200\196\017\137\017\255\000\054\024\131\002\182\246\086\222\202\202\202\202\202\202\202\203\021\138\013\219\048\179\089\021\238\088\189\096\245\103\175\114\201\102\174\021\150\043\021\138\033\016\172\172\172\173\189\184\089\089\022\221\058\036\089\255\000\045\100\035\065\156\109\194\202\200\005\138\013\088\172\080\027\092\043\146\153\077\035\211\052\199\038\233\172\008\081\176\046\147\086\035\110\202\193\096\212\096\097\078\161\140\167\105\141\079\211\156\019\161\123\022\104\016\085\149\150\040\181\098\172\172\173\189\185\089\057\151\069\150\255\000\146\178\017\171\120\172\172\128\086\086\065\170\200\155\035\037\212\116\146\072\162\211\000\076\129\141\226\119\186\186\186\186\190\246\186\150\146\055\169\052\178\019\226\124\104\059\098\021\145\010\202\202\222\050\203\162\223\248\224\019\088\131\124\086\218\200\004\002\001\000\190\022\100\168\180\247\061\067\066\200\215\198\247\087\087\087\068\249\092\208\084\180\013\041\240\186\052\010\178\033\089\089\089\091\123\035\224\044\069\150\255\000\139\107\046\131\109\226\027\132\002\178\001\004\231\168\040\159\042\134\145\145\236\074\190\217\043\171\172\214\091\221\093\093\100\179\089\172\208\122\190\197\003\177\010\122\016\228\224\232\144\055\069\168\133\101\101\101\100\124\110\106\183\252\056\106\017\248\194\027\089\000\172\128\095\008\101\041\165\211\195\087\194\037\100\174\174\137\218\234\251\015\019\080\225\125\223\016\120\169\160\049\166\200\172\139\081\010\200\163\228\181\209\141\017\255\000\007\100\200\215\199\142\200\108\002\001\000\130\115\236\161\167\116\230\010\118\196\174\137\069\219\095\128\098\176\086\088\241\013\088\171\043\043\033\196\133\116\028\174\174\170\040\067\214\069\138\247\086\068\034\017\008\241\062\002\219\162\207\248\022\182\232\050\222\091\032\016\008\004\002\001\057\246\084\212\134\084\198\006\002\081\114\047\066\234\202\219\088\109\035\236\152\235\161\181\145\028\142\193\018\174\174\134\214\088\236\002\010\166\148\074\028\215\066\067\175\177\069\017\177\222\202\222\015\148\230\034\063\190\099\046\135\111\045\182\008\004\002\009\242\089\081\210\102\130\046\078\122\178\012\225\146\205\094\202\235\052\028\130\186\042\219\089\091\099\176\224\055\186\061\247\005\079\078\037\018\070\097\112\055\069\029\236\173\229\115\046\136\254\236\004\200\252\160\043\043\043\032\016\027\061\214\084\180\189\083\123\039\062\200\200\155\221\091\098\118\115\249\004\222\022\224\085\208\042\251\181\091\113\179\134\192\170\138\113\051\094\211\011\135\116\224\136\069\031\057\009\204\254\229\173\186\108\118\242\091\096\056\089\004\227\101\077\017\153\215\013\023\078\041\145\160\044\137\070\084\123\172\182\028\066\178\005\014\103\124\145\076\236\174\173\184\059\020\010\005\085\211\009\154\219\176\162\138\063\162\230\127\110\214\221\001\111\048\218\200\013\128\086\178\196\202\230\048\068\213\117\151\113\180\155\093\124\171\032\023\100\008\086\240\093\023\160\245\150\229\225\053\192\171\043\043\038\237\109\221\176\065\106\020\185\008\156\136\068\034\017\220\237\111\033\098\034\223\218\053\136\011\121\198\193\004\006\211\061\082\065\210\105\114\186\046\081\236\010\121\218\200\041\042\128\070\161\206\089\044\138\130\164\130\056\141\158\120\093\079\081\115\146\201\050\161\205\080\214\002\154\065\086\220\236\118\010\251\087\211\244\156\013\209\008\163\200\143\033\109\211\155\111\236\152\196\060\118\228\054\027\061\216\170\040\114\046\118\228\040\254\019\141\182\178\036\053\079\083\146\005\006\146\132\046\093\007\040\105\141\247\028\029\198\090\051\127\078\244\097\114\196\141\169\234\204\102\057\068\131\129\065\018\130\005\079\023\085\182\049\056\163\250\035\114\046\158\203\127\096\198\120\199\059\032\021\144\065\017\212\120\001\141\186\191\003\217\030\232\043\217\084\212\228\097\166\115\212\116\077\106\013\001\093\095\149\182\186\060\110\174\174\172\010\146\137\175\083\209\190\053\077\082\097\115\092\029\181\214\072\237\117\116\213\117\168\211\221\070\235\171\034\021\188\039\141\184\218\233\236\183\245\204\111\233\089\089\004\016\010\087\098\168\088\156\119\027\093\093\013\167\185\109\061\030\040\109\117\125\199\035\204\171\172\147\074\249\085\122\118\074\137\174\107\081\059\151\034\235\166\190\203\059\166\148\070\066\118\116\164\068\034\017\253\051\221\057\150\254\177\172\240\143\024\064\032\130\119\242\059\028\065\087\216\109\109\128\225\116\231\172\214\104\061\053\200\112\190\199\147\223\100\092\139\150\073\146\038\189\093\103\177\027\201\184\117\148\111\186\011\082\135\038\196\110\008\069\021\111\009\028\173\192\247\079\101\191\170\107\124\067\198\016\065\072\236\069\011\046\110\137\065\001\184\228\229\037\075\090\141\097\066\181\071\086\215\032\229\027\174\130\182\214\086\069\030\007\178\145\062\164\004\106\211\106\212\115\135\038\185\054\075\162\154\120\073\177\069\070\235\038\148\241\144\252\110\040\132\119\062\011\114\183\002\046\156\219\127\080\214\248\172\128\231\109\173\176\065\005\057\186\134\060\024\253\155\194\220\100\146\202\122\162\229\021\051\228\076\211\046\142\148\020\180\015\098\134\115\025\130\075\160\118\027\020\081\225\043\241\083\078\233\012\090\123\220\134\150\157\166\089\075\078\248\212\021\069\169\146\093\093\093\053\003\179\154\158\044\074\041\138\036\086\160\203\022\059\032\081\008\236\124\054\241\017\116\230\219\250\102\143\016\227\111\000\216\021\019\115\144\149\035\182\010\052\235\043\035\187\083\251\042\185\174\168\232\243\076\138\202\203\178\032\042\186\016\241\076\243\027\216\080\220\241\122\169\121\145\244\148\002\049\211\010\203\021\036\064\170\202\060\085\028\214\044\040\166\059\096\083\130\148\034\138\136\247\218\178\044\153\078\228\081\071\099\206\219\219\137\225\101\035\110\173\253\032\008\011\120\066\183\017\202\202\219\188\217\186\115\084\169\219\053\070\212\224\174\142\193\053\085\063\017\079\017\149\209\071\136\010\170\185\176\169\053\119\185\051\084\145\082\213\009\150\161\077\222\003\112\054\186\190\246\218\082\168\105\187\212\215\050\021\038\171\033\049\106\178\042\106\158\168\082\069\117\087\015\069\244\239\200\038\187\020\029\182\073\226\234\072\213\148\108\238\129\071\227\233\034\040\236\081\071\195\101\110\022\086\086\222\070\095\250\048\154\203\120\135\150\193\089\089\084\252\080\183\024\222\110\156\130\106\206\203\036\119\027\106\046\090\124\088\181\129\085\079\209\100\178\025\028\208\180\218\102\184\079\024\104\015\019\071\007\128\169\084\243\116\034\145\230\066\180\168\154\069\099\186\076\167\151\168\210\181\008\114\109\009\218\200\148\028\228\215\111\210\088\160\080\114\186\175\110\050\055\184\033\016\138\040\163\226\178\182\214\086\229\044\127\209\070\207\016\230\057\014\021\006\238\104\197\174\040\160\129\087\087\065\029\155\181\120\247\194\044\002\214\095\102\109\075\084\232\077\078\164\101\026\083\238\216\213\208\060\111\179\194\214\029\102\237\005\067\225\085\021\142\153\105\046\254\050\084\255\000\020\127\059\090\232\162\235\038\073\194\196\006\020\213\170\049\083\155\180\167\034\143\150\202\202\222\009\025\111\232\035\101\252\054\064\120\007\049\187\091\156\146\167\240\027\053\089\089\004\208\181\022\217\204\169\098\109\083\022\166\091\035\026\197\210\088\039\139\045\049\150\099\054\010\202\219\141\156\022\175\029\218\026\176\066\052\246\170\002\216\227\117\100\105\213\044\034\136\123\145\067\178\186\181\209\236\155\034\186\186\249\077\109\144\085\237\188\116\191\005\057\020\081\240\089\091\133\149\182\060\156\219\167\011\126\243\069\208\022\240\217\091\136\224\057\013\233\133\230\149\063\096\080\040\108\212\017\065\057\174\114\255\000\027\154\255\000\019\026\118\143\025\079\209\147\244\233\227\089\057\168\074\010\142\035\059\154\209\027\089\176\225\100\054\114\123\115\018\194\096\038\064\131\220\228\205\062\105\019\052\114\134\143\026\255\000\021\026\058\102\042\054\061\188\046\158\081\041\146\174\168\077\114\099\187\133\082\047\029\041\069\020\081\071\195\110\103\099\198\086\095\247\152\219\120\064\222\202\219\013\203\236\152\111\204\108\229\167\139\203\042\114\059\004\054\110\214\064\033\198\234\072\153\034\168\209\129\084\242\250\116\061\200\004\016\225\101\109\220\113\083\202\101\080\104\225\050\040\227\070\069\115\192\163\176\079\040\163\176\065\049\053\075\244\167\251\034\138\040\249\206\197\030\082\178\223\184\193\225\003\097\204\185\081\070\036\116\209\244\157\215\008\060\059\128\067\103\252\105\159\121\083\248\004\019\080\220\112\186\037\093\100\159\027\100\076\136\071\184\240\075\022\105\145\134\034\175\192\133\101\130\115\086\011\253\020\119\009\136\004\019\254\033\251\162\138\059\029\199\140\141\138\040\241\119\116\225\111\218\002\232\011\110\056\219\195\035\172\163\166\205\066\206\154\054\042\108\026\041\247\178\001\013\164\248\210\254\210\005\039\201\216\032\130\098\008\171\160\129\216\159\008\241\219\107\171\241\112\079\069\020\118\009\169\165\052\163\241\031\228\071\099\177\229\101\110\103\114\017\028\158\203\254\211\007\001\196\120\094\046\153\033\140\181\247\089\169\141\204\098\220\002\027\073\241\165\253\222\165\067\128\077\040\021\117\117\117\154\205\025\087\081\007\093\091\141\188\143\112\106\125\099\016\172\105\066\116\037\089\033\177\068\039\236\080\217\168\032\191\212\095\145\020\081\216\236\021\188\119\087\068\236\085\185\072\219\126\195\069\249\002\175\198\252\047\189\148\129\103\139\122\201\239\201\069\184\216\108\255\000\141\055\180\142\023\083\049\030\044\042\234\234\234\235\036\249\044\166\175\077\050\074\169\232\222\135\110\001\091\107\120\165\136\060\075\166\177\079\023\076\199\051\152\162\151\052\215\038\029\138\037\060\223\112\172\163\106\183\112\157\245\167\251\034\138\059\029\172\135\035\185\146\203\172\019\098\116\130\031\141\200\086\226\246\221\031\215\104\183\128\120\012\189\219\037\247\120\186\197\089\006\166\236\016\224\085\017\180\165\078\159\176\221\138\234\251\184\170\151\151\147\065\102\211\207\211\052\242\135\120\199\055\149\085\088\026\154\012\175\150\157\184\067\017\141\205\076\065\020\238\201\194\233\205\068\032\016\076\087\077\083\155\071\075\243\177\071\205\078\090\030\234\120\138\246\226\225\130\105\190\231\156\173\253\102\015\008\230\086\155\014\078\174\160\197\049\247\089\133\144\064\093\098\131\080\220\112\042\035\140\165\075\240\237\135\011\160\142\210\050\226\158\159\021\138\171\131\007\195\057\136\193\090\215\160\253\135\144\148\249\131\085\102\164\141\214\151\079\114\230\163\026\012\077\087\089\034\164\236\156\119\009\133\004\213\092\252\098\164\220\237\100\071\145\223\102\149\154\146\165\170\051\223\195\107\162\063\080\032\060\035\128\221\206\090\119\227\201\084\105\194\067\254\036\004\237\061\173\017\108\055\028\006\211\140\092\195\118\200\165\027\014\032\171\160\134\243\195\213\104\110\046\125\049\106\138\181\205\080\213\135\038\074\131\213\249\221\025\002\124\234\125\069\173\079\153\243\152\168\195\071\230\124\016\244\219\100\240\172\138\186\005\002\156\172\046\246\034\019\087\194\005\049\107\014\179\041\135\183\099\230\041\135\023\122\133\157\192\238\128\216\242\033\091\105\155\250\140\030\097\177\114\134\141\242\038\012\001\040\200\066\205\086\205\097\026\009\188\006\192\043\043\042\192\169\031\148\110\082\068\138\028\046\130\027\051\112\086\165\079\101\069\081\117\037\027\094\164\162\115\019\106\159\026\143\081\076\175\098\021\077\040\076\186\139\170\140\225\062\173\161\063\082\104\082\106\036\167\058\089\084\090\105\041\144\054\053\095\082\180\170\092\070\196\043\167\020\054\106\114\011\037\033\081\148\123\160\020\107\085\126\079\140\088\034\143\142\073\003\083\102\014\217\193\057\171\021\220\040\218\172\173\181\183\182\199\114\046\156\045\250\065\001\111\045\182\114\164\012\183\081\100\178\069\200\041\077\222\192\130\110\193\005\109\155\189\072\187\116\183\246\157\090\226\086\108\221\130\027\004\019\021\213\246\236\225\085\074\096\117\045\069\195\014\073\244\237\122\126\146\212\253\041\232\208\076\023\066\096\176\153\097\050\244\211\020\221\058\082\153\163\149\022\151\027\087\072\004\226\002\173\169\197\080\081\025\156\005\144\217\201\200\237\116\019\207\113\216\023\047\147\211\086\217\133\074\115\151\099\228\107\122\210\086\080\098\161\150\234\201\193\098\177\065\187\057\192\044\130\029\209\010\220\229\111\233\048\126\129\065\229\170\058\165\146\186\186\005\085\179\023\048\160\129\220\043\241\123\110\041\031\211\146\086\228\025\241\032\068\114\010\232\021\125\193\082\048\074\217\088\234\103\211\074\217\064\046\009\175\186\010\203\021\128\088\133\109\204\129\058\238\085\085\045\128\067\019\170\223\027\003\006\193\057\057\020\118\005\016\174\158\020\109\238\066\182\213\015\193\144\119\059\031\027\157\097\167\015\119\081\085\082\119\016\204\228\104\228\076\184\117\182\150\094\244\250\083\228\082\105\001\163\188\039\153\068\093\056\091\244\090\045\250\022\086\217\142\237\116\233\000\066\070\169\236\230\067\176\067\113\181\208\059\021\080\048\116\047\201\137\237\079\106\002\232\182\219\005\142\215\087\226\213\044\045\152\073\004\148\142\164\212\090\245\216\160\222\118\070\202\183\084\107\085\053\035\234\157\020\077\132\111\116\081\106\114\049\046\149\135\198\216\162\212\006\214\065\106\146\170\118\216\001\177\241\062\080\213\155\164\084\176\244\154\178\069\234\087\226\162\238\084\178\089\080\082\006\156\130\154\091\170\214\228\216\029\118\241\040\160\166\111\232\048\126\155\148\063\089\037\199\107\046\234\062\200\108\016\216\110\055\170\102\067\077\146\237\040\169\024\152\044\094\016\008\004\017\027\135\108\192\156\129\065\202\225\226\171\074\080\086\073\076\105\245\038\061\053\224\241\116\128\042\141\086\056\212\181\211\084\026\061\037\011\052\114\037\091\105\010\013\222\202\219\252\041\095\213\145\163\098\143\134\103\089\180\241\137\019\028\026\179\089\034\081\042\165\221\226\008\186\201\190\249\050\087\186\145\182\083\253\105\249\095\119\247\253\006\015\211\033\002\090\173\116\026\177\088\160\163\129\239\066\141\010\069\233\081\166\043\166\091\176\216\108\240\163\119\074\075\221\029\136\079\077\221\233\143\064\119\112\077\065\059\107\171\166\076\159\012\115\041\116\100\234\122\152\023\249\041\216\191\204\200\191\204\072\142\163\052\136\083\207\050\139\069\081\083\199\002\050\223\157\246\182\199\123\043\043\111\093\063\077\148\209\240\040\243\010\065\117\221\137\147\221\007\043\162\085\213\075\020\074\079\138\022\100\235\098\154\229\035\213\068\190\200\054\059\158\047\030\097\250\056\185\096\120\089\013\163\140\189\069\076\024\186\128\039\078\023\171\106\245\173\077\169\005\007\132\232\067\147\163\197\004\054\114\169\098\211\229\201\174\086\069\057\089\000\173\096\109\103\030\229\234\054\228\190\055\035\131\093\100\202\148\038\005\016\215\035\075\017\077\167\140\044\090\212\101\001\062\117\123\166\013\239\179\081\059\147\192\113\186\172\147\173\036\109\176\216\162\143\128\167\053\061\150\080\031\106\201\094\232\247\016\158\242\042\119\244\159\216\172\016\137\087\057\170\033\109\142\231\140\163\204\193\230\130\133\242\038\105\109\009\176\049\170\205\011\218\083\169\152\229\046\158\164\140\179\120\227\204\197\024\140\084\085\008\212\213\142\122\047\037\089\203\023\032\242\213\029\099\154\160\170\014\068\135\039\118\064\161\179\152\162\127\069\249\100\017\221\138\098\139\209\238\174\160\122\118\228\171\160\136\218\232\061\009\087\085\117\023\081\100\131\083\090\173\192\004\074\190\215\069\006\238\212\002\008\237\093\063\077\180\204\224\074\191\136\133\051\123\083\158\206\146\200\184\185\088\171\185\068\212\228\246\168\234\092\196\043\130\125\119\111\178\028\074\191\002\017\030\086\015\036\108\047\052\148\013\141\057\216\169\171\090\212\253\077\127\145\114\110\164\229\030\164\028\153\061\211\226\108\162\122\099\025\010\150\028\005\076\248\009\101\234\026\061\040\200\155\012\016\163\087\016\077\171\137\201\244\208\076\171\052\195\018\137\229\166\009\115\014\055\065\013\138\168\098\160\168\186\011\020\090\172\130\147\132\070\203\043\171\171\239\117\125\251\047\106\187\083\026\016\136\044\002\199\107\238\017\033\025\002\018\181\117\026\178\010\234\232\247\086\077\027\023\089\019\116\093\136\150\067\051\216\219\112\060\045\193\177\146\133\051\151\166\114\049\057\076\212\009\106\055\122\107\022\011\004\208\177\078\140\167\053\098\128\086\067\137\229\048\242\052\121\104\105\122\098\090\140\021\077\107\164\076\141\242\168\244\105\028\191\193\167\104\143\010\106\089\034\080\212\058\053\077\083\212\015\096\120\142\151\249\031\217\087\205\145\210\104\250\166\178\171\020\002\051\180\033\059\010\176\084\149\089\173\078\159\164\250\073\113\113\040\032\175\179\154\186\005\174\019\217\122\176\189\067\074\204\021\100\238\232\181\089\116\147\229\099\019\171\087\172\114\101\106\138\160\057\124\162\055\146\080\197\037\105\094\161\232\084\184\040\171\172\162\173\009\146\135\108\119\124\192\041\043\064\079\175\070\181\200\214\057\010\194\155\090\163\172\106\109\064\040\027\171\160\175\181\150\161\080\169\226\178\013\069\095\115\189\209\042\056\139\147\032\107\085\192\093\080\186\193\100\010\124\045\122\150\142\203\004\024\177\081\211\093\054\157\161\089\161\089\169\208\049\201\212\044\042\090\034\197\110\071\148\131\183\142\049\228\160\167\234\057\230\194\178\124\141\006\158\103\093\072\169\065\171\123\214\082\047\085\036\101\146\178\164\106\052\030\156\211\077\211\048\201\216\055\189\075\241\108\135\035\016\244\212\225\232\186\251\177\248\163\037\141\123\058\208\181\214\087\236\210\129\064\171\169\234\195\083\235\028\081\148\149\153\066\087\004\202\226\020\053\097\200\039\182\234\071\136\212\213\206\122\099\029\033\135\072\123\151\248\069\038\145\035\081\202\051\075\087\117\242\128\083\206\026\165\148\200\169\180\199\200\134\144\196\253\024\041\116\169\024\189\209\154\122\165\079\054\104\169\038\013\085\026\138\146\169\206\076\137\242\040\180\153\028\134\138\191\194\181\059\069\082\105\050\181\062\039\198\153\080\230\168\043\110\162\125\209\042\251\084\212\008\218\193\153\146\169\177\167\215\184\163\059\138\234\148\218\135\005\029\122\190\072\239\020\087\095\081\045\064\141\075\094\074\117\067\138\235\057\050\177\237\080\215\130\154\236\147\233\238\132\102\236\138\201\242\098\164\212\128\078\212\094\191\200\061\055\082\042\026\230\185\053\247\083\209\228\158\220\119\059\030\069\056\091\198\193\228\211\155\139\042\228\193\172\110\110\150\079\079\020\079\079\170\043\172\228\201\200\077\121\097\118\053\081\061\157\051\067\061\195\093\219\081\125\133\035\058\146\234\146\226\222\055\084\111\234\067\035\113\125\059\178\099\074\005\093\085\212\089\050\055\072\105\244\116\041\233\227\095\254\057\082\105\208\200\170\180\215\068\152\242\211\073\085\144\149\246\109\084\230\071\105\250\127\093\127\029\056\053\110\114\234\200\163\170\033\087\082\009\155\124\077\028\249\137\077\149\084\217\029\046\143\168\101\159\020\114\040\100\019\106\092\213\061\051\042\091\036\102\039\080\205\222\121\236\042\042\204\138\040\157\033\167\210\154\213\215\142\036\106\092\228\094\242\128\114\201\225\010\151\004\039\100\138\167\073\107\148\145\058\035\073\085\101\027\242\217\242\006\002\076\238\168\168\017\174\239\048\105\046\114\102\157\003\081\162\166\042\077\038\050\167\162\124\042\025\139\008\118\097\049\183\063\085\060\248\009\036\047\052\154\107\230\077\210\161\106\255\000\027\078\084\186\048\083\066\232\077\037\094\042\057\046\177\082\058\194\174\171\053\004\015\153\071\162\020\116\070\169\052\103\181\061\142\140\211\086\150\168\101\200\106\016\249\037\030\032\135\146\038\227\030\160\238\218\099\050\155\083\125\222\238\013\114\211\037\193\218\205\062\038\154\076\030\215\088\106\050\093\218\044\087\126\164\252\164\229\165\185\106\012\198\106\018\190\014\074\071\224\211\239\117\013\056\167\099\230\051\035\024\088\177\016\090\169\170\122\139\083\162\233\170\121\049\117\076\214\100\081\245\094\226\032\098\116\171\034\154\235\170\119\173\078\155\164\250\025\113\117\108\214\107\027\212\045\111\066\052\118\186\138\076\029\172\071\218\009\112\053\021\070\069\077\070\233\204\113\050\153\175\113\122\236\017\149\102\086\069\102\087\085\118\114\100\174\098\158\006\212\176\130\199\080\077\146\115\195\069\069\073\153\196\244\088\239\113\211\232\196\045\146\164\188\150\172\002\025\049\069\048\149\106\020\157\023\082\205\137\042\046\200\186\230\178\076\157\166\210\245\159\085\081\211\022\186\044\077\158\072\147\218\218\184\229\140\198\234\026\139\140\214\161\046\045\134\035\043\241\101\044\126\165\242\172\092\186\242\070\172\202\150\203\025\137\244\018\221\178\183\054\145\099\226\148\120\152\060\140\023\115\141\155\168\045\020\094\074\167\094\078\045\126\046\152\122\152\072\177\162\126\109\170\117\223\166\051\167\003\221\147\185\080\058\210\235\044\247\082\190\206\147\179\155\220\234\015\239\164\211\245\036\173\151\039\058\092\086\119\217\143\045\087\081\218\162\039\179\166\231\062\235\076\252\245\206\179\137\186\182\193\070\229\085\008\168\143\187\076\211\023\173\026\159\055\212\191\039\030\014\249\213\157\104\085\021\033\168\119\182\006\158\233\239\086\222\219\217\053\202\154\076\093\172\069\140\148\015\179\170\106\122\170\008\085\116\185\029\050\155\168\250\217\081\126\032\149\114\153\049\106\123\151\106\184\072\196\194\114\102\118\076\041\231\190\150\206\156\038\075\189\210\018\174\153\050\130\108\029\171\083\100\216\100\193\204\125\214\162\239\118\144\219\201\170\075\119\023\217\117\028\155\057\084\146\227\038\179\029\159\064\251\024\141\197\087\105\060\078\248\240\176\115\178\183\010\097\119\204\171\143\187\069\010\095\191\022\173\050\078\218\165\063\078\074\121\204\107\236\106\079\070\159\156\046\198\077\093\183\141\134\198\165\083\118\082\191\055\080\051\163\014\092\088\108\169\031\211\118\177\006\046\178\166\147\167\037\104\200\052\221\013\218\169\092\181\122\124\030\209\145\130\033\079\023\202\182\214\218\024\243\118\171\062\079\107\011\141\053\056\165\140\156\147\188\077\054\118\185\241\075\246\194\198\248\181\206\186\161\143\161\009\118\073\199\131\074\211\164\197\218\172\029\057\040\223\221\231\184\054\140\119\067\248\233\248\014\226\145\253\088\234\033\232\190\130\075\138\199\093\250\048\239\168\126\099\187\015\191\090\030\218\063\188\037\087\143\127\019\201\227\191\129\131\157\248\208\143\124\165\085\253\244\129\104\156\123\241\005\083\077\131\245\024\122\209\042\008\243\151\086\127\134\180\103\000\042\163\188\114\251\032\167\143\055\214\156\035\118\195\112\174\170\089\215\129\057\080\207\215\143\241\187\128\117\149\100\061\120\180\154\075\201\080\251\241\123\147\158\032\140\187\035\163\193\119\079\046\072\158\046\125\151\084\044\213\247\139\221\038\182\229\070\061\215\185\172\118\045\164\143\169\045\107\241\107\157\201\174\196\234\172\234\069\019\172\106\020\189\161\128\094\074\247\099\023\000\085\052\221\055\234\244\249\182\025\058\102\067\145\209\085\127\230\221\159\109\099\241\209\253\226\249\175\251\242\060\101\030\001\229\211\126\239\085\007\221\167\118\167\240\105\242\245\035\172\135\165\038\139\015\186\189\249\075\204\168\143\082\157\069\239\134\181\235\070\139\041\043\159\148\135\147\085\011\238\043\098\233\074\085\044\221\039\212\179\168\216\221\126\048\057\098\034\028\028\108\041\217\153\213\042\051\045\025\022\071\209\139\139\159\117\233\092\228\040\081\165\112\087\178\234\004\100\084\172\233\138\185\186\178\208\051\181\056\201\245\143\201\250\044\042\162\076\159\206\015\229\133\236\196\254\072\234\207\182\133\185\077\170\031\111\042\055\245\163\158\046\147\150\138\059\087\031\231\221\191\125\099\233\073\247\133\106\031\110\071\140\163\192\207\000\227\165\142\242\041\190\212\189\169\188\020\083\116\223\172\067\113\166\012\033\149\215\119\131\076\055\138\118\217\250\113\187\042\031\119\104\236\198\041\093\119\108\074\014\067\096\169\223\140\154\228\072\039\011\045\046\167\033\044\093\055\014\013\054\078\146\232\240\046\204\213\072\041\227\061\214\157\022\115\084\191\139\221\117\028\066\049\081\168\000\142\160\229\014\164\154\246\076\157\068\010\100\013\141\086\215\166\139\153\015\066\040\127\142\035\221\068\223\079\002\190\197\203\045\194\211\159\223\084\143\009\104\092\171\126\218\067\111\046\168\239\119\016\168\165\233\191\087\167\190\218\056\246\213\254\109\227\251\235\031\090\063\188\043\080\251\120\221\241\206\052\121\142\058\088\079\042\079\144\049\166\191\128\021\029\170\097\148\116\096\240\233\046\085\237\180\180\146\224\045\145\112\232\083\223\120\025\213\119\069\165\124\029\234\025\215\131\225\061\185\177\174\044\048\202\042\099\111\180\248\028\235\008\163\192\086\212\245\158\180\072\211\141\206\239\054\084\241\226\171\171\012\132\005\138\045\076\115\152\189\115\194\125\075\222\177\084\112\006\170\135\245\228\174\056\178\134\062\164\181\238\178\118\215\081\069\212\050\209\251\090\123\109\079\038\047\214\152\169\221\103\084\186\238\208\219\222\185\247\151\123\240\097\245\016\188\096\116\143\165\065\254\093\225\252\154\207\197\039\222\021\168\125\199\132\238\121\199\224\028\116\239\172\137\202\094\212\222\000\168\167\192\085\212\230\060\058\105\180\154\179\109\037\213\012\125\073\117\089\045\195\078\098\166\159\249\234\219\140\155\005\068\235\138\200\240\146\128\102\042\233\250\070\134\167\164\234\152\242\012\125\249\151\089\066\058\135\083\170\196\004\086\156\048\167\028\015\115\094\254\148\127\059\095\108\073\093\051\181\212\085\044\122\101\046\015\172\147\041\052\072\213\083\242\126\206\041\163\165\005\004\189\072\255\000\222\245\163\171\078\195\101\033\185\209\062\149\063\151\144\042\134\126\155\245\138\092\078\149\244\148\255\000\038\244\255\000\151\090\084\127\104\086\160\125\227\200\255\000\158\076\030\093\059\233\047\194\170\252\003\194\029\101\149\248\093\095\141\017\180\154\200\238\180\086\094\077\066\076\164\216\168\007\078\008\101\180\186\131\110\208\118\005\082\191\023\235\081\217\244\046\197\243\183\172\028\220\086\155\085\117\044\120\059\148\174\069\194\024\229\147\168\234\088\061\164\166\123\096\087\087\218\017\148\186\204\158\240\169\233\157\049\135\076\100\104\116\090\132\145\163\035\017\016\185\075\166\049\234\162\149\209\040\106\241\014\055\052\013\233\064\227\191\217\218\179\240\143\071\127\186\165\152\073\189\023\242\066\241\098\180\129\104\101\117\221\126\121\089\018\042\160\211\134\044\121\247\111\069\222\093\104\251\168\190\208\170\243\252\131\099\177\240\075\243\196\033\229\211\127\028\223\031\238\175\240\240\186\191\134\026\110\162\149\157\051\194\152\218\077\101\190\213\163\183\024\165\057\059\107\042\247\116\224\248\078\254\090\118\157\193\177\213\099\206\024\015\186\087\224\234\184\242\077\119\076\198\241\052\123\095\123\168\099\205\250\188\189\233\226\205\210\089\177\169\123\069\194\139\242\106\070\243\045\058\062\156\079\159\037\213\093\101\214\093\068\217\108\156\027\085\027\217\129\137\185\058\173\221\056\201\222\134\060\165\213\228\202\074\007\225\046\164\222\251\233\047\085\204\198\082\168\059\083\147\223\133\247\011\076\146\199\012\003\184\105\221\229\214\015\242\233\255\000\048\005\093\249\007\003\185\227\047\033\229\043\077\250\079\240\171\063\015\006\139\151\080\183\023\055\019\202\215\044\254\049\169\050\206\225\025\247\106\163\248\220\162\029\026\081\188\035\039\235\015\236\086\148\252\225\035\018\134\205\254\090\118\246\082\183\169\020\135\166\203\100\097\136\065\017\218\252\033\126\039\084\165\204\070\252\009\057\071\254\231\252\091\221\080\125\245\031\204\081\117\161\031\008\239\117\065\039\187\082\110\050\233\140\202\106\247\221\251\021\166\182\205\169\126\111\140\217\218\136\187\055\211\159\140\186\179\044\247\040\125\180\247\225\117\126\017\191\019\037\113\112\113\218\251\105\099\249\053\067\121\180\254\202\157\085\254\094\007\193\047\038\124\219\152\229\166\253\038\248\085\093\224\224\206\206\046\178\212\033\231\070\204\164\169\125\159\169\142\220\007\206\163\222\040\134\071\081\056\195\189\055\103\213\082\138\130\052\150\170\122\118\083\169\199\187\096\180\215\042\182\097\046\158\255\000\101\076\153\063\074\165\185\169\151\039\030\116\242\007\138\250\094\139\225\154\205\255\000\115\119\134\252\041\029\105\053\104\241\148\168\015\086\002\108\178\227\069\249\053\127\201\163\069\097\080\252\159\125\233\109\211\058\065\040\233\014\085\130\209\239\011\172\253\097\183\097\083\123\105\248\083\197\212\117\069\038\035\123\171\241\210\071\187\080\063\205\067\241\002\169\252\156\015\130\065\219\140\095\060\172\128\086\227\166\253\036\078\082\247\166\227\040\188\112\202\042\035\034\220\180\166\042\169\111\054\162\061\156\106\133\226\162\101\229\213\029\193\142\178\100\235\174\157\058\115\174\175\189\011\173\038\177\021\158\201\075\085\060\038\103\204\068\017\158\119\077\126\038\088\197\084\110\024\151\038\028\233\129\087\087\218\051\103\106\177\117\035\084\085\157\052\230\182\085\232\148\144\022\110\085\023\228\214\062\240\055\165\001\220\021\028\182\066\160\035\080\165\155\037\126\021\099\058\118\247\053\221\161\225\165\253\131\178\083\071\131\188\026\072\085\093\229\161\250\069\241\047\223\200\255\000\142\048\014\246\086\086\086\218\202\200\014\037\105\191\089\019\254\071\122\110\045\247\067\071\047\074\109\070\027\030\048\255\000\020\089\094\090\255\000\194\056\212\031\227\210\091\148\186\137\187\252\112\186\207\214\025\148\107\074\166\233\178\121\051\117\252\048\075\129\213\104\209\090\105\202\015\142\000\168\028\036\109\093\041\133\214\077\149\205\080\085\061\207\172\250\157\232\063\037\120\206\122\179\140\071\149\249\055\221\076\193\238\212\191\023\013\036\119\234\099\061\116\089\055\193\166\011\071\033\187\169\133\153\031\212\252\236\124\054\078\008\241\166\249\242\021\166\021\034\151\230\152\222\159\141\055\120\102\236\248\221\234\033\112\177\222\153\153\190\189\248\053\138\179\240\113\169\250\104\237\085\046\201\254\048\170\063\146\158\134\159\172\250\185\112\004\240\186\044\060\066\166\119\081\181\148\198\023\233\018\089\213\045\197\252\035\125\145\198\165\178\233\079\011\252\124\170\146\132\083\169\229\204\157\244\225\221\159\201\085\168\200\143\146\132\229\016\022\155\084\250\112\210\021\105\180\244\207\234\178\102\096\238\069\083\028\041\254\084\067\219\240\206\071\153\078\227\077\224\028\074\211\143\186\079\154\145\238\211\125\209\056\119\225\065\222\042\145\105\040\039\233\186\190\014\227\125\050\021\169\203\114\021\064\254\005\110\021\134\204\210\251\068\227\223\112\219\174\139\151\069\232\064\244\090\071\000\160\118\112\233\240\250\120\223\038\103\133\059\050\055\107\140\172\196\238\020\082\098\117\024\186\209\193\055\078\074\145\155\079\011\166\189\071\086\224\141\089\082\205\146\039\133\047\241\197\165\123\159\086\252\159\194\197\116\220\177\060\180\199\039\255\000\244\106\127\078\026\066\173\239\046\155\053\141\124\057\014\115\255\000\029\052\127\044\010\110\209\004\060\039\131\254\120\083\236\121\014\084\063\146\085\086\061\250\065\237\048\179\248\105\071\217\092\045\034\164\152\078\201\169\140\101\067\001\144\189\226\038\076\254\163\148\223\252\227\131\126\117\031\173\032\198\159\133\004\121\039\213\049\171\215\181\127\144\106\171\109\219\184\084\014\085\179\118\191\022\142\140\113\085\218\089\253\227\133\236\169\101\186\174\131\165\038\159\055\081\147\051\019\196\027\044\202\191\008\217\147\181\039\244\225\210\005\163\144\221\219\210\197\212\116\147\182\035\234\216\087\168\098\146\054\190\062\026\105\247\084\011\084\234\063\078\026\066\172\252\177\187\007\068\122\172\145\184\158\049\140\159\171\028\098\167\238\246\252\214\159\227\028\143\130\079\158\020\251\030\067\149\043\173\044\202\180\045\037\214\146\177\182\147\134\148\238\218\147\125\202\057\012\102\042\246\188\014\153\078\168\107\021\085\087\081\013\159\222\157\089\089\089\083\139\191\083\054\104\237\076\173\189\008\246\084\155\190\200\246\095\120\056\053\248\167\059\046\052\177\100\237\078\116\086\157\047\081\178\183\023\112\099\177\118\173\021\217\075\063\077\242\055\170\219\120\138\162\138\203\081\159\170\250\001\106\126\026\099\123\234\070\243\109\068\111\079\194\128\255\000\037\112\181\070\161\248\248\105\010\179\243\045\054\117\093\023\042\022\101\038\180\245\068\219\186\030\231\080\248\028\143\130\095\158\020\251\030\064\171\171\240\140\217\242\133\090\059\080\062\210\234\035\191\013\047\237\169\181\064\192\231\084\083\022\043\043\149\242\131\110\169\233\195\090\245\023\186\155\133\003\111\038\172\228\059\211\111\117\073\218\025\126\240\082\245\027\052\120\157\056\231\011\133\143\134\049\211\100\210\245\030\091\218\142\110\155\167\025\183\133\187\234\063\129\080\085\226\167\135\036\123\120\105\225\204\214\084\136\090\123\170\079\254\126\020\029\153\080\236\164\130\135\168\217\226\192\233\157\225\034\199\107\042\001\252\154\143\230\172\239\015\013\033\086\126\097\031\182\055\224\230\158\171\011\113\059\149\165\014\250\171\175\045\008\176\167\090\131\189\195\145\229\125\165\249\225\079\241\198\234\232\114\037\124\178\118\228\200\206\046\173\025\071\195\076\251\106\191\088\062\210\212\226\236\098\145\122\048\189\043\026\186\241\198\041\102\047\117\067\049\058\108\185\071\051\049\118\244\113\116\219\169\075\147\168\031\212\134\070\224\118\249\071\248\226\113\185\136\225\029\115\125\218\092\184\154\200\236\237\177\037\069\073\218\072\139\120\083\179\039\106\051\226\214\183\191\068\096\230\217\105\243\102\217\089\137\222\008\022\167\083\116\026\172\169\043\172\159\024\122\124\037\188\135\117\021\042\158\161\176\137\036\047\043\078\125\227\158\012\119\181\211\207\070\017\220\176\224\218\241\239\210\029\218\118\217\219\233\241\247\175\151\041\137\202\046\026\095\181\181\039\041\041\162\188\050\055\019\167\078\171\097\224\085\016\233\199\051\179\125\063\182\056\002\173\055\147\145\225\117\125\228\249\225\079\198\251\142\079\084\135\040\156\164\102\038\154\094\172\079\102\039\122\024\176\090\140\185\042\070\228\253\067\242\032\226\139\182\133\216\186\182\059\168\038\233\056\057\178\131\070\010\020\010\058\102\070\170\106\195\003\142\070\142\167\160\231\006\078\031\068\080\162\114\138\144\070\171\234\059\048\092\214\028\091\143\094\032\242\195\013\083\101\111\074\050\132\017\172\218\197\062\161\099\013\083\102\014\165\078\137\193\116\094\083\027\209\109\084\221\071\081\069\116\250\175\229\170\138\234\009\122\110\187\102\111\065\200\083\185\050\054\177\085\215\134\002\075\141\036\120\010\136\046\136\080\214\150\040\171\024\244\090\199\035\076\189\041\094\144\175\072\023\178\053\062\162\002\123\204\134\158\155\168\171\032\012\084\243\244\140\117\012\144\062\149\174\094\141\071\003\099\090\133\070\074\154\060\157\085\063\191\080\109\213\020\221\055\203\019\102\070\145\193\122\087\166\081\169\234\004\044\123\174\168\234\050\108\212\235\018\172\163\166\047\083\200\041\163\029\211\164\232\178\178\053\011\240\115\072\149\179\066\099\059\067\015\080\214\076\033\142\038\228\094\044\162\022\082\028\159\225\060\100\249\225\007\011\240\028\045\177\011\078\125\219\081\217\084\178\234\010\142\145\107\153\048\052\107\208\166\064\198\041\234\131\003\222\094\104\089\101\080\252\222\002\178\197\098\172\169\165\015\108\244\248\022\189\205\077\173\112\094\189\201\213\174\040\146\084\020\229\234\072\203\076\115\058\052\221\068\175\242\073\250\129\041\206\046\052\081\092\213\203\155\232\103\192\213\193\137\189\144\157\235\212\189\058\087\057\089\052\150\168\171\220\212\221\072\047\242\013\085\053\189\068\025\146\145\221\024\213\036\185\054\120\241\049\084\058\052\053\036\237\073\073\088\231\047\149\079\079\117\085\081\117\005\070\041\208\182\080\248\139\085\172\155\080\230\166\215\149\254\065\029\069\063\080\113\079\149\206\218\010\082\229\061\064\098\134\078\176\154\002\194\028\090\153\090\224\189\121\079\172\113\095\042\022\116\152\093\114\195\214\137\227\188\085\101\137\186\136\095\228\026\159\168\039\188\188\150\166\060\177\069\094\155\059\028\186\145\133\037\123\090\170\042\076\198\146\028\141\100\185\058\007\117\163\145\152\154\090\190\154\019\050\068\105\163\094\149\129\025\153\016\158\099\051\168\098\080\255\000\035\222\236\099\111\142\220\036\249\225\022\247\086\086\220\115\161\118\050\084\071\113\020\170\104\044\154\226\212\218\215\005\235\202\125\107\138\036\185\065\009\121\170\147\164\208\128\076\136\185\091\123\226\163\168\108\129\212\073\212\206\094\157\200\082\056\166\210\006\167\085\134\172\153\048\125\043\130\049\021\211\066\018\084\116\074\106\128\208\172\160\169\079\164\014\094\141\203\209\185\010\043\032\216\090\157\074\215\039\192\090\177\086\076\101\212\081\008\068\242\245\092\026\154\075\079\105\195\224\033\116\208\138\233\180\142\040\066\200\148\245\087\086\086\076\144\177\050\173\174\094\157\143\078\163\112\093\018\023\077\008\202\232\056\166\081\021\104\162\082\213\151\108\215\022\152\234\091\034\125\029\209\129\193\116\138\109\051\138\142\149\177\170\154\142\162\178\130\094\153\146\017\050\116\004\046\154\193\008\137\081\209\047\227\114\146\148\181\022\219\110\251\069\078\233\012\210\136\027\242\160\151\166\100\140\078\030\204\080\036\033\080\244\106\094\085\238\169\233\139\213\076\130\054\209\199\139\107\029\102\129\231\127\207\008\188\003\152\056\155\228\043\097\196\193\084\008\125\038\073\212\143\011\211\185\054\145\197\010\059\039\213\008\195\157\146\104\219\079\103\182\170\143\034\218\023\149\044\125\051\101\240\155\059\154\133\107\215\175\122\053\111\043\220\228\218\087\148\040\220\153\020\141\071\052\102\197\026\226\164\157\207\217\173\069\016\153\154\099\102\043\161\041\078\163\121\078\162\120\088\190\052\202\213\214\133\203\040\087\171\099\084\179\058\084\200\028\228\218\039\047\066\087\161\112\089\200\196\106\209\173\078\173\121\069\249\108\026\176\066\157\197\010\023\033\068\224\131\037\106\116\242\053\122\197\235\145\212\028\159\082\247\109\100\026\186\014\040\081\185\071\078\246\166\230\172\081\107\202\125\035\156\189\003\147\169\092\019\163\033\049\238\098\109\098\235\196\087\168\132\039\087\089\073\080\231\160\084\117\101\139\173\020\139\161\025\094\153\138\208\177\075\093\218\229\202\202\202\057\140\107\173\028\200\209\221\122\055\161\070\244\218\102\177\075\089\101\078\210\243\003\123\085\075\147\196\131\099\225\190\229\059\231\132\127\062\033\181\147\187\043\221\081\075\117\060\087\083\211\224\091\043\154\155\088\245\235\156\141\107\147\229\115\182\178\029\148\052\153\168\218\035\105\042\073\067\067\187\155\034\017\106\197\006\168\105\023\102\163\034\201\102\186\165\117\046\164\166\107\211\226\197\000\172\131\050\076\164\001\093\173\093\069\154\234\046\170\200\057\075\070\010\123\049\086\076\143\037\021\048\098\234\044\214\075\053\213\086\107\212\212\086\070\050\186\104\049\069\077\116\003\090\139\214\075\037\154\205\062\153\175\082\068\090\177\088\166\066\094\153\072\026\189\129\025\017\122\201\100\179\089\172\215\081\117\086\096\163\078\199\169\104\220\212\065\027\089\098\177\086\227\138\013\065\016\172\172\154\242\023\170\122\053\047\041\206\186\096\201\083\198\026\159\082\216\218\123\153\169\049\099\062\015\003\224\114\060\089\243\225\178\182\193\059\220\106\105\108\058\165\167\252\143\103\052\060\073\074\139\072\086\218\202\200\013\162\169\045\077\151\037\154\127\125\236\172\172\160\135\020\231\038\048\189\071\166\184\175\241\173\071\076\098\126\152\164\166\124\104\062\201\237\018\002\219\043\040\153\130\123\238\163\133\210\038\105\168\105\241\175\065\018\126\152\164\167\116\105\174\178\168\136\060\096\161\143\000\226\162\166\116\137\154\123\066\244\081\039\105\241\167\233\174\079\141\204\077\145\077\021\213\148\048\167\061\052\023\040\244\247\057\013\053\139\208\070\157\166\180\169\104\223\026\014\082\048\072\011\108\162\139\051\217\136\185\050\039\061\055\078\113\077\211\026\133\012\065\122\104\130\232\196\186\017\047\075\017\071\079\136\167\233\105\244\082\049\088\181\054\075\039\198\217\084\144\022\043\043\109\101\101\101\101\109\251\133\150\214\086\086\217\140\186\134\005\086\112\108\020\173\149\050\141\140\082\012\218\099\123\016\146\252\029\200\108\255\000\142\044\249\240\052\112\205\082\194\115\186\044\098\171\045\181\044\192\053\204\014\083\068\172\173\181\184\066\238\210\075\101\214\095\059\217\083\193\117\033\081\183\055\069\024\136\023\111\117\123\170\234\110\154\107\148\237\080\054\229\206\084\241\117\156\208\025\196\128\241\083\031\073\204\043\165\103\023\042\088\186\174\250\240\005\072\193\032\158\030\147\152\087\075\220\226\163\143\168\232\161\017\044\184\002\171\160\233\150\149\059\059\176\098\028\085\029\030\072\088\044\188\033\233\241\182\069\061\001\106\248\087\014\083\067\135\027\043\111\074\067\095\252\111\018\208\070\228\246\096\085\149\147\091\117\028\009\242\136\068\146\153\021\059\241\115\074\186\154\163\021\124\157\185\240\075\241\196\033\224\027\184\170\105\131\016\150\235\036\092\165\113\046\178\108\133\169\142\018\010\152\085\185\094\219\089\001\187\025\145\182\033\255\000\058\107\046\243\243\198\177\185\068\196\241\118\211\246\014\043\075\111\183\136\090\171\123\002\157\240\180\214\217\188\181\081\217\170\253\150\150\046\227\202\185\153\070\213\142\073\229\052\100\235\098\060\129\202\170\144\072\015\181\055\220\037\139\019\224\108\206\010\142\066\225\092\044\240\237\172\169\160\079\112\140\060\228\084\022\203\174\002\099\242\082\253\154\056\059\193\055\054\113\178\178\178\027\148\089\117\009\178\037\117\002\158\200\034\021\059\250\101\205\200\077\022\038\219\091\123\043\113\163\098\121\185\113\090\088\246\242\169\252\081\163\241\015\213\235\076\252\124\181\017\120\129\068\167\042\118\227\031\045\092\246\102\250\073\238\121\056\093\159\005\174\068\170\006\229\043\188\192\217\087\211\220\052\162\222\163\094\220\124\058\119\198\013\122\147\077\140\167\105\110\008\211\201\026\109\107\218\164\168\050\032\081\088\108\201\011\016\247\030\007\193\049\231\023\128\111\100\224\129\179\164\146\251\053\169\129\016\136\084\115\170\136\110\159\030\060\108\173\198\156\098\203\167\170\001\104\185\086\159\225\141\059\226\031\171\214\148\127\143\149\104\188\077\217\130\238\248\028\066\213\157\119\055\098\180\215\090\071\114\106\152\099\032\078\090\075\125\199\207\246\019\071\131\227\114\169\101\199\050\180\231\041\229\045\017\078\092\159\038\009\149\023\078\013\114\052\113\057\059\077\079\163\145\168\228\019\081\098\013\183\019\224\151\231\148\039\123\032\056\141\202\115\080\106\193\006\161\179\251\044\138\167\172\186\124\121\039\195\101\111\015\192\082\042\094\209\114\212\077\161\141\063\235\015\213\235\072\250\242\168\239\019\017\084\130\242\187\147\086\160\235\202\213\127\115\149\041\180\174\228\197\092\045\040\078\090\072\237\203\038\172\155\224\005\106\172\177\105\076\238\164\110\046\228\085\009\247\072\204\148\116\225\166\165\142\114\130\055\101\082\252\084\083\184\151\200\024\153\080\010\184\034\107\102\056\158\119\217\199\156\095\054\064\115\028\049\088\171\112\153\202\142\157\157\058\202\044\021\061\081\106\200\061\073\018\112\183\056\251\185\231\176\248\145\068\044\206\090\171\173\028\106\095\165\055\213\235\071\231\039\120\218\138\211\069\228\119\038\170\131\148\173\069\222\242\154\108\255\000\245\197\191\058\152\247\132\229\165\015\227\227\254\157\048\191\089\169\181\004\040\042\115\071\150\168\047\027\084\103\189\091\057\149\072\125\211\063\181\060\165\230\073\176\049\205\146\126\046\077\129\173\085\012\046\080\083\144\106\101\193\177\251\143\019\224\063\007\156\102\198\202\220\199\132\167\119\116\126\214\135\042\218\060\016\145\225\127\043\144\166\148\172\075\121\083\125\166\248\255\000\078\249\031\094\090\193\246\049\079\244\165\078\090\065\247\030\071\235\254\202\210\135\126\068\217\167\187\130\118\207\081\155\179\136\090\176\065\057\080\118\135\139\190\132\100\122\106\196\040\042\044\111\113\199\082\252\044\064\247\168\023\103\058\115\238\112\200\067\014\006\170\055\056\211\052\181\074\255\000\116\004\227\234\141\216\251\170\217\114\115\057\031\003\254\057\132\059\142\099\194\226\169\219\156\135\108\144\017\133\213\093\069\085\038\110\227\072\061\213\031\083\241\255\000\175\245\203\088\076\083\159\109\042\114\210\126\238\249\227\254\157\246\043\074\250\242\152\218\054\047\245\111\099\126\178\170\055\094\046\090\168\246\004\245\075\218\046\050\126\054\043\044\081\103\120\254\156\117\067\252\108\069\017\113\200\170\104\238\100\155\002\218\160\189\064\066\068\113\042\221\189\045\140\207\192\124\166\133\126\039\193\039\134\019\113\204\115\038\201\245\009\148\239\148\211\209\136\085\247\186\186\168\151\016\222\084\065\084\253\092\153\247\119\032\181\143\150\132\248\139\148\113\022\167\045\047\242\191\231\139\084\159\119\045\048\090\062\085\166\209\070\157\245\199\248\163\250\200\180\211\252\092\181\063\198\212\229\015\104\248\205\248\153\189\052\029\087\059\139\086\169\037\203\083\254\066\063\060\074\166\152\049\102\199\175\073\011\147\180\180\250\025\152\179\123\019\107\019\107\001\083\075\153\104\067\107\238\124\047\240\211\159\060\198\230\154\149\129\117\044\139\149\213\213\209\042\234\165\247\112\028\074\163\248\169\248\114\139\242\063\147\126\117\127\176\043\037\146\186\211\187\074\255\000\158\044\083\126\071\042\015\195\203\082\054\137\138\079\173\189\144\252\061\105\127\143\150\161\248\090\157\242\223\167\025\255\000\020\097\008\156\084\090\121\114\104\108\098\252\092\236\027\080\252\158\212\077\222\143\207\018\153\030\071\208\056\034\247\198\091\088\224\153\092\189\080\114\194\039\169\040\024\069\172\091\200\171\120\015\134\019\103\035\228\042\085\021\082\015\186\201\100\178\089\043\160\085\083\108\230\241\042\143\235\084\156\161\252\143\228\010\158\149\179\031\241\209\175\065\026\244\049\047\069\018\142\158\056\203\143\040\213\071\228\114\163\252\060\181\095\198\197\055\199\254\096\079\090\079\211\149\103\225\106\255\000\127\249\226\008\176\100\109\089\128\139\239\202\214\085\149\008\119\067\178\167\238\255\000\245\206\007\088\135\170\167\228\118\130\051\033\117\011\218\013\067\194\030\035\197\223\030\016\154\110\015\011\171\237\126\037\056\044\022\069\170\025\179\087\087\087\087\218\081\155\099\226\085\039\210\167\229\202\159\242\201\250\113\170\175\202\229\075\248\121\106\223\070\169\087\250\135\229\203\072\248\229\087\248\088\155\246\119\199\151\179\020\245\057\041\101\234\022\133\043\236\169\026\166\056\176\114\040\172\202\011\020\224\169\126\025\038\042\163\187\128\218\202\222\073\060\116\230\227\133\213\247\191\018\017\106\115\084\070\206\186\186\037\117\002\015\009\165\017\103\112\042\151\233\083\246\114\166\252\178\126\156\106\171\242\185\082\254\029\175\195\086\250\053\072\138\139\236\229\164\124\114\170\252\044\076\251\063\227\195\125\195\172\156\027\034\170\161\122\195\020\210\164\138\234\038\088\086\189\014\100\034\153\179\194\014\045\070\119\020\212\057\031\011\207\142\007\088\157\238\174\129\087\087\067\155\130\189\156\085\211\223\117\138\248\065\197\014\232\112\042\155\233\083\246\114\166\252\178\126\156\106\167\242\185\082\126\030\090\183\209\169\232\168\254\092\180\159\175\042\207\194\196\207\188\158\118\201\101\083\076\217\131\216\088\099\117\147\010\149\217\059\153\042\158\148\206\106\105\186\046\068\034\213\138\107\085\188\071\137\241\181\095\183\001\192\114\041\232\063\179\223\146\099\086\008\181\096\154\056\149\079\248\234\126\206\080\126\089\063\078\037\081\249\028\168\013\225\229\169\119\137\165\061\127\166\039\173\047\180\124\181\003\252\077\077\062\249\062\060\225\214\085\180\253\086\168\028\170\097\192\241\141\185\187\252\096\067\077\055\101\163\026\137\236\054\183\144\241\144\246\242\068\238\220\007\000\174\174\174\134\197\061\170\200\049\052\111\109\206\229\083\253\042\190\092\163\252\146\126\156\074\111\187\150\152\127\143\149\080\202\022\047\145\255\000\145\240\245\066\049\135\150\170\125\141\067\236\255\000\142\056\175\106\187\087\180\162\206\113\057\086\071\211\145\134\201\223\200\210\045\196\027\022\190\224\057\093\074\193\032\146\153\236\077\127\233\072\124\177\027\030\003\128\060\111\177\088\160\060\069\083\125\042\190\092\155\247\147\227\139\084\242\244\215\172\094\173\122\178\189\089\081\084\023\025\062\120\199\240\243\119\185\105\037\030\064\100\219\088\179\186\063\082\081\238\154\220\099\226\223\157\088\247\008\125\157\245\227\084\109\031\093\171\212\053\009\194\142\164\133\027\196\163\141\214\168\219\128\161\042\169\150\060\010\167\236\204\145\114\186\051\098\166\120\115\191\064\246\071\202\010\105\191\001\192\033\250\069\082\253\042\209\095\250\119\215\136\090\163\240\030\169\122\162\189\073\094\165\203\079\148\189\242\124\241\103\215\255\000\069\105\142\179\228\249\227\017\239\093\023\078\070\058\201\199\179\149\036\125\073\038\060\162\249\212\095\121\002\255\000\103\233\198\180\127\006\043\005\138\185\011\079\159\221\032\239\198\187\240\132\195\222\165\183\103\006\182\228\058\203\053\146\154\076\067\220\092\154\016\240\158\114\159\060\071\194\015\233\021\073\244\171\248\255\000\078\095\248\229\170\011\199\138\193\116\214\043\075\030\249\015\126\035\232\062\074\166\126\050\075\200\042\184\122\236\248\089\175\149\069\007\069\164\242\143\176\149\217\061\020\063\031\026\159\192\017\236\179\105\068\042\017\252\146\158\252\107\127\008\065\063\187\120\007\089\025\074\015\076\117\212\230\232\053\091\244\094\110\124\204\054\240\142\055\224\227\101\214\009\166\252\202\164\250\214\124\127\167\040\251\197\202\191\188\045\095\249\132\146\138\210\071\119\252\241\063\136\109\240\097\119\082\062\077\147\021\037\052\115\047\241\074\042\072\225\079\147\046\117\079\233\198\054\114\143\241\113\147\188\045\068\093\006\089\006\221\081\083\116\129\055\229\169\058\204\005\007\093\127\175\247\185\011\021\138\107\236\137\201\015\001\240\200\109\250\049\158\222\001\207\043\042\106\039\084\145\161\177\079\163\185\136\151\048\131\126\084\127\021\095\086\253\092\169\077\225\229\083\222\022\047\245\078\156\180\128\157\243\198\099\104\091\179\214\155\050\112\177\231\145\071\156\098\231\081\154\229\155\057\083\155\195\197\131\038\127\139\122\026\107\211\116\196\200\099\137\057\247\229\024\185\212\037\201\222\149\142\104\246\184\030\199\231\129\220\015\005\209\240\202\235\254\139\013\183\191\148\156\149\019\048\143\106\202\038\204\030\199\066\238\005\081\149\080\061\172\250\185\105\230\241\114\147\188\045\077\080\252\057\105\063\007\149\097\180\045\087\178\120\084\239\197\205\061\086\249\222\238\155\038\126\078\104\077\055\014\084\093\225\226\010\204\172\202\191\129\207\233\054\087\228\234\039\220\079\071\145\244\242\181\057\143\106\205\117\022\097\100\154\051\119\248\164\116\199\132\224\088\121\159\011\221\100\079\233\052\249\232\219\148\173\236\165\168\197\122\146\163\155\037\169\210\245\026\195\198\143\237\055\196\072\173\040\251\079\039\126\032\154\162\248\114\210\190\167\150\162\109\019\084\167\176\057\002\168\170\044\158\060\204\010\182\162\233\161\072\108\033\248\043\077\055\143\206\214\217\086\212\095\106\051\217\239\197\071\058\205\022\070\228\253\062\034\159\165\148\234\025\090\168\105\176\085\021\133\174\021\229\076\252\221\202\254\041\093\250\108\059\142\099\141\027\241\148\021\044\151\118\073\174\177\062\230\204\220\031\194\144\251\229\248\139\228\173\045\246\050\014\252\100\237\018\106\139\225\203\074\061\143\022\139\157\085\233\170\111\136\221\100\240\162\147\019\079\080\156\223\035\024\170\106\020\143\204\180\039\186\230\004\086\148\228\071\127\040\109\149\077\077\211\221\145\033\082\034\224\189\169\223\030\228\199\020\106\108\153\086\166\168\179\092\235\171\254\139\205\129\063\168\195\218\222\066\129\196\211\203\147\106\027\139\174\155\220\229\102\212\186\242\014\016\031\124\139\225\238\084\207\197\192\245\069\173\193\141\186\174\150\204\071\226\002\156\168\165\192\159\127\000\210\081\061\033\085\038\078\106\151\185\115\108\152\235\130\213\020\184\168\106\177\064\007\162\045\224\181\213\131\020\245\055\082\205\146\099\084\175\081\051\037\031\103\057\082\075\131\143\188\120\196\100\167\074\216\212\213\025\041\036\205\053\169\229\082\158\239\109\211\025\222\071\027\009\138\047\237\147\074\194\202\174\075\160\173\206\254\041\093\250\172\062\103\173\062\171\020\246\137\007\163\009\145\134\042\202\156\027\243\197\166\197\202\126\206\249\031\010\009\208\171\186\018\070\086\081\174\172\097\073\086\170\101\186\010\087\118\141\214\046\008\028\076\083\144\155\086\189\075\023\170\098\117\097\083\076\175\145\029\148\126\231\077\021\199\194\014\201\016\153\041\106\138\116\218\180\029\027\151\073\116\202\233\149\210\043\164\189\141\078\171\082\206\159\033\114\107\083\159\138\002\230\040\236\037\024\188\247\087\196\193\081\100\039\099\214\000\174\145\093\050\176\043\007\046\155\151\072\174\146\056\053\058\169\161\062\161\206\082\084\000\139\139\211\090\158\253\169\135\186\108\194\021\069\171\214\221\050\086\020\072\112\016\041\078\013\113\185\031\162\243\111\215\141\222\082\190\020\058\131\163\095\229\002\126\164\074\115\203\200\008\112\042\023\100\202\182\040\157\100\224\174\090\155\080\132\205\093\086\163\080\208\157\083\116\002\250\167\027\162\020\111\186\115\080\113\106\021\043\212\005\234\066\053\037\119\040\053\072\251\170\070\034\213\081\010\006\201\178\093\057\136\011\038\212\016\132\237\043\170\132\206\094\161\235\174\228\101\070\161\161\062\160\149\242\131\019\164\178\249\080\069\100\002\171\106\141\233\193\053\229\137\181\000\161\034\234\149\234\030\189\075\215\169\122\245\047\093\119\035\034\116\193\058\165\057\238\114\013\086\178\116\136\011\166\195\096\199\096\091\088\017\146\055\035\079\011\145\211\194\244\050\005\077\027\218\107\101\239\224\062\039\186\255\000\174\211\101\127\041\069\089\089\001\206\141\246\050\178\234\070\226\089\034\198\232\177\098\177\088\166\177\092\053\061\249\040\153\145\150\046\223\008\075\117\141\209\098\197\098\132\107\179\083\164\186\141\153\024\153\100\233\154\210\246\221\075\002\248\077\146\202\224\162\197\138\178\185\089\057\092\237\138\012\068\134\167\073\116\005\212\048\166\139\040\170\004\134\102\093\056\098\089\034\045\069\170\202\229\102\229\212\114\234\185\117\028\174\085\150\040\049\099\100\233\044\139\174\154\219\168\097\178\147\176\138\046\169\147\079\101\141\218\132\132\033\082\066\109\114\053\137\230\228\115\039\194\019\205\191\102\055\120\175\194\202\222\002\129\177\099\250\141\158\043\162\044\129\178\019\021\213\011\170\023\089\025\078\204\102\074\056\172\166\236\214\067\144\034\202\232\076\087\088\174\177\070\066\085\211\089\146\138\044\083\159\211\015\057\026\122\139\039\181\075\005\211\152\091\176\144\132\038\089\180\175\106\246\175\106\201\171\172\140\132\236\214\018\162\134\200\055\021\081\081\146\141\216\016\238\160\158\027\162\044\131\172\186\203\054\175\106\176\088\133\136\086\011\218\179\106\234\163\041\042\251\050\011\168\225\178\036\048\077\055\081\082\191\020\101\236\238\230\202\054\100\091\020\032\086\068\216\221\206\254\055\031\217\005\052\223\194\080\243\069\047\076\220\060\075\021\211\160\040\180\240\233\146\163\167\076\142\201\242\181\138\105\204\138\024\241\099\233\242\078\165\112\070\023\043\044\074\017\018\153\078\153\021\147\136\096\150\067\033\178\033\067\080\088\129\015\079\141\062\157\024\136\086\230\034\041\144\038\070\156\225\026\154\115\034\178\178\167\159\166\123\056\073\021\211\161\033\091\123\242\178\017\146\155\078\153\013\144\098\150\112\196\247\151\171\109\146\008\166\059\019\234\001\083\063\051\250\047\119\237\177\214\067\191\235\144\153\033\098\100\205\122\049\163\026\233\046\138\233\038\198\176\013\083\213\128\162\137\175\077\193\171\043\162\156\083\221\100\214\228\090\197\130\196\004\250\160\212\231\023\032\021\145\008\132\009\106\101\090\024\185\024\209\137\024\002\052\193\122\096\189\048\066\156\033\018\017\175\107\084\149\136\184\185\000\172\172\136\081\076\099\076\153\178\034\196\099\070\000\189\058\244\203\211\047\076\189\050\244\200\064\016\133\008\214\054\079\170\099\084\149\046\118\214\086\068\043\032\143\000\063\065\238\183\238\198\239\024\062\082\021\147\102\115\080\171\094\173\171\213\053\026\176\141\107\147\164\115\182\107\236\163\126\074\234\232\170\158\201\146\226\189\082\053\046\040\146\085\144\106\003\123\034\021\151\194\109\067\154\133\088\094\165\133\100\213\147\086\077\070\070\133\213\079\171\113\068\146\128\064\112\042\219\050\169\204\077\170\186\234\180\172\154\189\170\193\088\047\106\205\136\212\048\035\090\212\234\199\020\233\028\237\172\173\194\219\217\089\091\145\231\101\101\109\156\111\251\161\049\215\241\130\135\144\133\101\101\101\101\101\101\109\233\207\119\057\100\131\149\064\184\001\089\089\089\014\118\086\086\086\069\066\220\195\169\220\016\099\147\105\156\085\075\176\218\222\011\035\217\025\086\105\157\211\249\091\107\043\126\137\241\061\223\190\211\100\013\209\241\003\192\015\005\149\149\149\149\149\184\089\067\246\121\078\122\108\182\070\107\134\121\220\020\018\152\204\117\077\114\234\053\079\090\026\137\046\241\145\117\138\197\014\200\011\172\085\149\149\183\183\238\061\223\208\177\214\087\241\143\212\043\043\034\235\238\016\067\204\081\219\045\135\150\203\020\007\244\078\117\149\255\000\162\099\182\183\239\089\089\091\128\062\114\021\149\188\227\244\130\034\223\160\074\039\250\070\057\030\103\112\085\255\000\100\126\248\031\166\019\141\255\000\064\159\233\066\107\175\204\241\191\156\248\071\156\249\192\253\115\229\115\191\167\006\201\174\191\150\254\083\250\231\206\016\222\254\080\136\071\244\092\239\234\065\077\117\252\087\222\255\000\209\159\056\087\087\253\002\239\209\123\191\171\006\201\174\191\150\255\000\209\031\217\178\178\183\235\057\223\214\131\100\215\095\203\125\238\174\174\175\251\103\251\007\187\250\240\083\095\250\183\253\131\250\039\247\156\255\000\236\065\065\254\123\255\000\118\085\213\252\207\127\246\119\077\127\252\013\255\000\088\148\231\255\000\108\029\250\055\254\202\255\000\164\077\149\255\000\184\107\172\129\191\232\223\201\111\233\047\250\046\114\039\251\160\083\094\173\251\119\065\098\143\239\093\093\095\244\092\255\000\239\131\255\000\112\044\255\000\118\234\254\019\225\038\200\187\254\002\234\255\000\241\069\234\255\000\240\097\232\126\173\213\213\213\255\000\082\234\234\234\255\000\175\149\145\125\255\000\225\129\065\255\000\187\127\013\213\255\000\088\243\046\255\000\138\005\007\127\127\116\094\137\255\000\141\201\007\121\239\253\105\114\039\254\071\052\031\253\193\122\046\255\000\149\014\065\255\000\217\158\200\189\095\254\102\232\061\007\095\250\226\224\139\255\000\231\238\131\214\067\107\127\077\152\069\232\159\250\064\228\030\179\218\223\191\144\089\174\162\191\253\094\075\168\186\139\037\125\172\173\250\022\222\235\037\154\205\095\254\206\235\037\154\234\046\162\234\044\214\097\100\178\011\037\146\201\100\022\065\102\023\081\117\023\085\117\022\106\234\255\000\254\241\255\000\255\218\000\008\001\003\017\001\063\001\255\000\246\101\020\081\095\238\209\069\110\181\069\110\138\040\175\245\104\162\138\043\225\091\095\101\179\145\200\228\095\249\148\080\181\069\021\242\108\191\177\099\099\144\217\101\137\137\255\000\147\066\095\027\022\236\111\238\180\052\061\033\008\095\227\036\113\043\118\094\168\075\077\255\000\066\081\028\078\037\011\072\072\162\138\255\000\006\048\056\210\211\123\162\132\181\099\127\210\106\199\018\138\040\072\090\069\018\143\247\145\008\020\049\252\018\219\027\254\165\015\084\033\045\034\198\255\000\187\236\140\072\233\148\080\215\198\079\226\144\162\056\141\150\039\180\197\017\196\175\138\036\182\132\086\155\031\247\082\016\181\069\013\018\248\055\241\138\045\035\152\230\094\147\044\177\051\153\200\180\053\241\079\117\164\052\061\081\095\217\066\022\144\144\145\068\134\049\106\079\227\103\178\134\190\075\084\196\114\047\104\095\020\244\198\181\067\254\194\210\019\034\172\075\114\068\145\090\144\223\197\043\020\074\028\071\214\239\081\086\037\090\113\031\066\122\122\091\189\089\101\238\198\089\101\255\000\081\124\017\018\203\047\077\013\020\073\233\124\034\134\196\153\218\039\171\210\032\118\118\038\077\009\137\234\132\033\252\044\228\041\028\134\055\180\196\255\000\165\098\101\233\009\138\066\144\158\172\177\140\147\249\165\108\226\053\217\052\061\033\016\056\244\056\141\015\214\151\193\049\151\185\049\178\044\228\114\057\028\142\066\098\127\210\108\228\114\044\076\177\049\049\049\061\178\067\248\173\038\041\012\200\254\008\131\057\013\157\013\244\049\022\095\193\238\067\098\099\145\200\228\114\016\138\023\244\024\247\101\178\196\068\091\090\144\254\053\166\114\163\153\055\127\004\069\209\204\228\244\236\161\015\226\247\033\141\141\150\088\152\152\180\133\247\216\208\196\181\066\066\022\144\150\228\049\139\227\068\145\099\210\022\209\020\113\026\213\015\104\252\015\086\054\049\234\132\133\002\154\016\132\039\253\010\056\138\039\019\136\162\056\139\075\108\159\173\173\161\106\104\107\084\080\181\068\081\020\054\054\094\159\195\240\049\141\141\143\084\036\098\130\098\192\146\050\099\162\132\033\011\250\086\036\068\163\141\017\086\113\043\076\126\134\049\106\203\020\133\033\171\028\014\007\018\138\056\138\034\142\164\094\223\195\240\049\140\099\210\018\049\010\093\025\036\061\068\095\211\068\068\139\028\136\062\197\218\056\161\161\140\099\249\038\070\067\101\148\142\040\226\138\090\076\114\211\023\197\015\078\035\030\146\049\227\178\120\104\073\163\155\069\217\197\141\081\017\127\069\238\044\082\211\019\020\133\034\198\245\036\053\170\248\197\151\101\106\217\123\108\189\049\034\132\182\144\245\033\141\020\070\054\099\141\029\049\197\025\061\152\224\081\056\011\223\220\127\106\044\114\211\200\145\250\148\071\202\066\202\158\229\240\162\138\043\073\138\090\162\180\217\123\162\190\008\072\161\161\146\067\090\194\215\228\082\095\130\201\190\138\236\130\235\083\063\037\150\047\180\182\181\041\036\075\047\252\030\118\047\035\254\145\204\152\153\159\053\116\132\165\039\098\198\207\166\197\113\102\044\151\209\099\026\248\223\198\206\069\238\181\127\036\138\040\104\104\146\024\200\202\140\108\148\168\115\028\149\144\154\057\034\115\063\058\246\196\063\176\197\240\147\165\100\228\219\232\080\108\250\104\120\083\028\028\072\231\232\174\082\035\019\128\226\052\067\167\166\232\148\209\245\005\059\017\033\200\182\091\019\027\057\022\200\201\254\068\236\099\152\228\206\076\228\197\049\074\247\210\030\084\143\172\071\034\101\018\232\156\199\054\206\108\134\114\089\108\230\057\177\101\104\089\206\086\037\165\236\177\253\181\175\034\084\140\074\200\196\226\137\068\146\180\053\078\140\072\198\089\100\203\166\039\209\146\108\237\156\078\197\038\134\236\173\250\027\183\171\213\208\228\217\095\010\061\017\118\122\037\038\202\178\142\209\142\119\236\242\036\040\182\197\001\227\068\241\215\161\054\070\054\044\104\120\209\044\117\232\132\168\094\138\040\095\113\107\201\102\031\068\119\036\078\031\146\043\162\040\189\072\145\025\116\123\100\098\082\036\138\020\107\092\139\061\163\141\014\071\049\051\217\232\228\115\057\150\033\163\031\178\114\252\033\034\052\082\026\019\166\100\118\069\009\014\040\146\039\010\102\052\040\142\035\068\149\051\027\235\084\087\218\177\011\089\253\152\215\066\220\150\144\152\203\212\132\200\137\244\089\103\231\108\178\050\037\033\177\016\090\148\142\069\009\020\068\108\082\166\123\017\100\089\033\151\108\142\172\100\163\098\084\071\089\012\190\204\062\190\244\116\204\158\200\011\116\050\058\150\216\151\100\086\154\211\019\024\208\196\201\050\136\196\138\038\232\114\016\138\122\076\252\031\145\124\031\161\161\046\196\086\255\000\035\022\166\100\246\097\244\045\209\095\102\043\082\039\236\129\093\020\081\068\162\070\037\013\020\081\249\056\137\013\018\103\178\180\165\223\099\037\018\138\040\138\212\198\133\018\017\041\035\162\041\025\063\240\066\090\145\002\138\056\139\222\168\161\199\081\090\153\051\010\232\072\163\137\067\067\095\004\081\090\142\165\232\159\178\031\022\132\134\038\155\161\173\062\158\162\072\162\037\013\018\084\197\043\040\226\113\028\069\210\178\046\198\142\034\136\186\067\182\125\051\233\141\028\072\173\063\100\122\122\146\047\162\010\222\168\161\143\217\020\113\050\018\246\099\093\017\067\069\110\095\008\171\040\162\133\017\034\107\162\126\204\090\162\114\166\065\218\030\178\116\140\126\202\040\159\177\118\069\019\069\017\143\101\019\068\200\196\140\007\019\136\226\056\137\080\145\196\113\232\068\106\202\040\156\122\189\070\035\137\093\149\216\145\053\214\177\020\055\068\101\200\163\129\196\104\155\031\179\018\232\137\069\013\020\078\061\124\049\035\137\196\224\040\156\073\199\163\047\242\049\009\090\040\205\030\204\110\133\232\163\200\151\224\198\187\056\148\102\246\099\143\103\019\137\192\081\056\153\035\209\199\178\017\020\078\039\019\137\040\013\024\215\103\018\081\232\170\026\127\131\027\119\216\145\151\209\030\217\021\209\071\211\020\010\050\046\134\097\069\025\087\070\054\211\161\118\183\047\068\143\114\049\046\132\183\069\019\093\124\049\105\106\040\162\126\143\035\217\136\138\232\163\050\017\142\086\137\116\100\118\200\123\066\244\081\159\217\129\091\040\110\135\145\017\154\101\018\143\071\030\204\120\250\020\076\178\226\099\201\200\081\177\194\204\145\226\204\111\177\043\067\137\044\086\125\038\067\019\066\137\159\209\142\054\197\017\244\125\068\070\105\234\081\180\073\083\060\117\209\070\111\067\255\000\166\009\089\069\019\093\018\100\021\200\199\030\132\081\069\106\099\222\047\090\090\137\041\013\217\228\174\204\076\199\218\040\242\017\017\077\199\209\063\033\178\063\245\145\246\071\209\071\145\236\241\144\145\228\116\050\030\200\043\071\019\244\253\216\163\072\163\202\149\186\034\232\194\237\020\121\056\250\179\209\131\037\170\040\226\113\056\146\233\025\037\110\143\029\091\040\242\029\045\069\246\069\116\053\209\151\217\227\122\214\127\069\024\165\197\139\189\103\149\034\070\005\108\143\072\069\151\166\075\208\253\233\016\125\022\114\057\159\080\114\177\030\076\076\103\142\250\162\143\039\209\098\145\200\237\145\102\060\137\162\083\073\019\151\038\120\242\072\077\081\228\206\250\036\200\072\197\145\087\103\214\072\094\066\110\132\211\050\228\081\068\165\110\197\035\197\200\151\076\082\071\147\149\085\022\070\124\093\152\252\148\215\100\102\153\104\148\210\070\095\034\250\090\241\165\076\230\143\039\037\244\137\050\044\195\036\209\041\036\100\118\207\029\173\121\018\057\031\250\096\149\162\143\037\162\111\163\197\093\233\050\203\044\108\111\161\233\017\245\166\247\090\242\023\086\071\217\143\039\019\245\106\140\153\249\116\137\016\141\163\233\143\031\071\174\132\218\244\054\217\232\079\254\031\086\068\173\246\207\108\244\196\198\216\136\231\113\039\149\203\216\223\224\106\136\202\189\031\093\209\041\055\219\027\191\071\175\103\255\000\193\077\175\071\213\144\228\223\178\232\239\216\153\205\255\000\210\076\093\139\221\017\155\094\137\100\108\178\050\163\245\014\137\077\200\100\089\009\241\063\085\209\057\185\050\071\140\134\203\044\178\244\199\184\191\130\018\040\205\027\071\030\207\071\069\165\232\073\182\066\061\020\081\146\003\139\066\077\139\003\099\241\154\244\125\057\033\197\145\129\044\098\194\223\163\244\242\030\025\033\218\246\118\067\019\103\233\155\068\188\118\189\014\050\094\202\111\217\008\031\069\179\244\207\240\060\082\069\053\236\166\040\049\097\108\253\059\252\031\167\145\250\102\047\025\162\088\090\059\067\182\113\103\104\182\118\200\192\113\162\203\046\197\027\232\197\010\067\248\216\199\184\252\034\133\019\209\055\125\019\128\226\044\098\196\070\020\085\011\077\028\044\199\129\046\216\229\024\159\168\137\025\198\068\240\167\218\020\059\035\137\126\071\056\196\253\084\069\146\050\050\097\077\116\099\195\108\226\162\187\031\147\020\044\209\100\177\198\072\120\104\199\136\114\140\071\228\196\089\035\035\038\020\213\162\056\173\209\028\041\046\199\040\196\126\076\080\252\184\159\171\136\188\136\179\246\200\158\005\248\030\058\056\156\078\002\128\162\074\035\198\125\049\198\140\112\252\138\085\209\239\108\177\018\125\124\033\180\065\107\035\163\145\236\156\008\190\196\087\198\030\204\147\227\019\054\102\217\201\144\202\211\060\121\242\093\156\123\051\075\138\232\201\149\182\091\049\229\113\102\025\114\069\037\217\229\231\119\072\182\197\038\189\030\030\118\250\101\038\100\151\020\102\204\219\045\144\200\211\060\121\242\093\138\009\059\060\172\188\087\068\179\054\199\038\089\108\083\100\051\202\044\193\229\095\076\148\083\086\053\079\104\177\148\075\161\069\182\069\012\142\152\244\153\039\240\142\146\034\136\234\081\180\056\118\040\156\108\203\142\187\070\057\254\030\235\120\253\158\083\232\151\189\035\196\244\126\079\043\248\146\247\168\158\039\104\201\210\060\153\091\223\134\187\017\228\250\050\251\223\133\046\143\193\230\203\087\170\220\100\211\060\092\156\149\051\036\126\008\101\143\178\151\197\177\178\203\027\248\047\098\034\034\044\090\112\236\226\122\039\217\044\100\111\224\204\043\179\204\244\063\122\143\179\197\093\031\147\201\244\077\247\164\120\079\163\200\116\140\206\222\252\052\073\209\151\180\102\093\239\194\125\143\209\230\059\122\068\049\218\031\142\078\021\166\120\114\236\159\163\243\240\127\011\213\150\055\171\047\228\152\152\152\132\045\052\072\067\069\124\025\135\217\230\190\183\031\103\141\232\126\204\254\137\174\244\143\013\158\084\186\037\239\072\240\209\145\143\180\121\043\247\111\195\116\201\063\218\121\079\189\035\197\141\161\225\232\242\097\067\247\175\017\059\038\250\063\063\010\213\109\200\178\198\245\127\052\196\196\200\178\034\017\146\084\135\043\019\211\221\018\048\158\107\220\125\158\063\162\149\217\228\062\140\158\244\143\013\158\083\232\151\189\035\196\244\100\125\139\209\228\174\247\226\251\036\255\000\105\228\123\210\246\120\178\073\018\242\018\071\147\151\147\024\143\022\038\071\209\125\252\044\123\101\150\095\218\090\068\072\233\025\085\161\198\181\122\069\106\102\019\204\218\125\139\201\105\116\126\169\146\242\091\084\198\239\126\025\229\122\031\189\071\217\227\122\050\123\033\252\079\043\222\172\241\159\100\159\237\051\251\210\098\206\210\232\121\155\044\246\099\199\108\197\010\050\072\252\238\203\067\144\134\057\022\088\159\218\079\072\136\139\019\025\056\146\215\022\045\228\048\158\095\194\203\248\248\103\151\232\126\245\031\103\143\232\200\099\244\121\126\198\035\198\246\079\248\153\253\252\147\049\229\167\217\139\034\104\156\074\221\028\073\198\133\049\200\108\178\196\196\203\251\043\081\098\101\137\137\141\089\056\118\070\003\137\090\102\067\015\163\203\030\248\179\139\056\191\135\136\121\062\137\123\210\246\120\207\162\107\178\061\035\202\118\199\175\025\118\079\248\153\253\233\031\077\159\077\156\026\223\143\149\166\116\208\253\233\011\082\086\074\052\095\193\050\254\210\101\145\098\098\122\066\026\219\211\050\024\143\047\208\245\031\100\113\166\143\160\075\021\034\126\247\226\030\074\232\159\189\089\226\207\162\074\217\039\072\207\043\101\136\241\145\063\226\103\247\168\251\049\194\209\192\201\143\162\074\158\163\211\048\059\068\253\150\033\106\232\201\033\233\124\023\219\076\076\076\139\044\082\044\108\077\012\107\089\076\103\149\218\030\163\236\241\233\163\164\101\244\079\222\145\226\250\050\246\140\202\158\240\228\226\200\231\084\103\242\019\093\014\086\244\145\227\199\162\094\143\035\222\163\236\241\189\012\203\036\145\055\111\081\246\120\253\035\038\147\057\009\147\144\221\252\215\218\177\008\068\095\195\044\136\201\145\158\242\024\217\228\046\135\239\072\241\159\067\246\101\244\075\222\146\236\241\189\019\060\136\015\124\152\237\239\028\109\152\213\033\187\071\144\187\222\060\252\080\252\171\039\149\189\227\139\108\198\186\036\142\035\076\138\056\147\137\067\136\215\197\011\236\161\008\090\082\044\100\196\143\076\139\181\172\132\012\235\162\075\189\035\198\244\063\102\095\068\151\122\138\236\193\232\111\179\036\109\025\049\215\175\148\099\102\056\030\144\189\030\066\239\231\024\217\143\029\011\161\189\209\206\137\059\040\104\104\162\180\144\151\219\066\022\144\181\052\089\236\128\201\174\136\251\037\218\050\097\062\139\035\137\152\163\072\126\201\171\068\241\031\068\134\058\032\169\013\139\178\080\039\136\120\217\193\156\025\028\118\067\021\009\036\054\071\209\150\022\060\108\224\206\044\226\197\006\071\007\253\033\141\033\036\181\098\101\150\062\246\198\138\040\161\047\184\152\152\180\132\196\201\080\226\040\137\106\072\098\099\071\003\129\232\178\199\019\136\162\134\247\099\137\192\250\104\250\104\140\017\210\027\027\019\031\099\129\244\209\244\145\244\133\140\081\211\098\118\052\081\067\248\063\133\011\238\197\237\022\114\061\156\068\182\201\071\073\150\089\122\189\094\210\061\106\245\099\122\099\099\100\089\098\122\178\203\044\099\145\023\067\144\180\223\193\252\023\223\139\189\222\162\196\086\168\226\056\142\059\189\061\045\083\020\068\134\134\139\044\114\057\028\186\028\182\152\164\114\044\179\145\200\228\055\240\078\135\045\081\090\111\084\087\244\019\161\063\130\032\196\080\150\218\028\071\001\196\161\148\036\036\087\193\140\162\072\072\099\248\038\114\027\028\206\066\098\218\213\021\241\162\139\254\148\100\086\208\186\032\254\116\073\021\240\069\151\171\211\211\057\023\170\219\027\045\233\033\045\086\145\095\039\183\253\056\203\084\036\052\039\068\101\241\173\113\178\081\040\173\034\254\047\079\227\090\108\162\138\018\040\072\163\137\091\127\007\253\120\200\178\198\245\025\028\132\203\219\103\034\079\079\227\101\151\183\242\101\020\081\066\040\081\040\072\122\191\139\254\186\098\145\122\076\076\178\197\035\145\200\228\114\028\139\044\178\198\203\044\189\191\147\018\018\040\072\072\226\070\039\029\088\216\254\044\127\217\076\095\011\044\228\114\057\028\139\044\178\254\195\248\086\232\072\081\056\145\137\196\073\012\099\099\248\216\223\246\211\047\072\122\178\247\101\252\151\193\148\081\090\162\132\132\132\138\044\179\144\228\074\095\054\255\000\187\098\098\099\101\238\254\194\069\125\181\171\044\228\088\228\088\223\198\198\255\000\192\178\254\107\224\132\054\050\254\194\213\141\150\089\127\054\095\249\182\089\101\237\124\249\022\094\215\198\203\255\000\074\254\109\151\246\172\191\254\018\198\203\255\000\066\254\251\248\089\127\234\223\221\178\203\047\253\187\044\178\203\044\178\203\255\000\233\191\255\218\000\008\001\002\017\001\063\001\255\000\244\029\020\086\168\175\247\040\081\177\098\108\088\088\176\011\000\176\159\065\031\071\255\000\015\163\255\000\135\208\071\208\030\001\224\030\017\226\099\133\021\254\154\086\071\027\100\112\145\192\071\018\066\130\071\090\162\190\020\081\069\020\113\030\043\030\027\037\130\137\098\161\170\255\000\057\070\200\097\178\024\040\088\210\040\173\061\216\242\080\242\163\235\031\084\089\081\245\016\164\094\144\245\067\133\147\193\100\176\081\040\052\087\249\049\141\144\193\100\048\209\024\081\095\022\075\037\018\204\075\051\099\200\216\155\101\020\052\114\104\142\090\035\152\142\075\044\229\164\037\174\036\241\089\060\031\240\150\058\043\252\085\027\033\130\200\097\161\068\162\135\240\156\232\201\152\228\228\112\056\028\005\010\056\156\071\001\227\028\040\161\058\035\144\140\172\161\033\011\076\104\158\059\039\132\148\043\252\056\227\179\030\026\035\010\018\022\154\026\211\116\079\045\018\151\033\064\081\040\186\057\035\146\044\179\145\100\214\232\077\196\199\150\200\177\009\150\054\049\141\019\197\102\076\084\053\253\244\172\197\130\200\097\162\190\076\151\070\073\158\197\026\047\084\080\226\122\019\028\132\197\034\202\026\044\177\245\232\197\146\200\177\063\133\021\169\070\204\152\073\066\191\187\024\089\135\199\035\010\211\127\041\058\050\100\056\217\084\073\217\024\028\074\220\144\250\047\081\026\212\151\193\245\218\049\101\177\050\202\040\173\178\068\241\217\056\087\246\241\227\228\098\195\066\085\166\081\067\091\108\201\050\185\009\018\066\136\134\246\216\216\209\024\138\037\009\013\018\149\022\039\187\166\098\157\145\223\019\137\067\030\167\142\201\227\162\191\177\143\031\035\022\026\020\074\210\069\012\122\108\148\137\062\068\099\067\061\149\069\150\049\012\122\138\017\071\161\178\078\198\171\073\009\018\137\138\116\099\149\137\233\020\050\067\220\163\102\076\085\253\124\120\236\197\010\034\045\197\013\018\221\147\145\024\252\025\066\136\209\040\140\122\091\138\179\137\147\024\208\132\066\036\161\100\163\070\025\137\136\079\077\014\036\145\090\162\112\179\046\042\254\172\033\102\060\116\068\076\076\177\009\150\049\140\155\161\187\098\027\219\044\230\125\065\078\201\170\028\168\229\100\075\017\245\040\250\167\043\028\009\064\186\049\074\196\140\168\196\232\132\139\019\019\044\099\137\040\233\034\137\227\179\046\046\063\211\140\108\199\142\182\132\034\206\069\141\140\108\155\034\081\050\058\099\100\230\040\217\124\078\086\072\072\069\146\149\014\216\250\049\204\139\178\081\178\080\049\186\035\043\050\118\046\136\050\196\196\203\044\177\161\173\179\036\108\201\010\254\138\141\152\113\080\149\020\080\132\089\122\067\036\050\108\138\040\113\178\168\178\076\147\040\093\018\141\145\067\066\091\246\036\056\156\104\198\196\201\014\034\116\046\201\035\027\044\095\038\061\179\036\044\156\107\250\024\097\098\141\105\020\036\061\089\098\101\140\145\054\067\076\099\145\038\057\138\072\140\211\037\066\036\206\066\100\132\203\028\206\068\038\069\233\043\037\003\137\035\027\016\152\183\101\156\139\047\124\076\216\137\042\251\216\227\102\040\208\182\139\024\245\098\101\150\073\147\033\166\072\147\036\198\113\020\078\209\031\068\181\099\144\153\236\080\028\072\244\070\103\034\040\104\146\036\064\066\016\198\203\044\191\147\086\103\199\095\117\043\049\066\136\177\124\215\177\064\146\173\050\076\200\099\144\152\201\014\035\136\241\138\005\015\077\252\081\030\206\022\060\116\122\032\200\061\072\153\007\216\132\034\198\063\138\043\072\203\142\201\199\143\220\197\011\018\161\017\144\187\210\024\216\157\233\076\125\234\076\100\204\108\136\201\068\162\135\208\216\209\067\099\101\143\225\004\038\050\068\100\066\069\153\025\035\031\177\011\087\241\179\139\035\215\194\172\242\049\013\087\218\138\179\028\107\073\106\043\109\146\236\095\180\078\197\182\050\104\198\069\233\141\012\108\122\104\146\040\162\138\020\005\017\244\041\156\172\227\099\141\016\149\017\149\146\037\209\143\177\008\127\037\236\077\015\183\190\037\025\033\104\203\026\127\107\020\044\075\084\036\037\191\102\060\071\210\070\072\113\213\012\100\137\008\199\045\180\072\161\196\186\028\134\206\072\232\232\180\142\104\076\156\181\006\034\072\072\143\068\164\076\196\171\075\224\217\098\150\253\010\103\033\105\158\078\033\170\251\009\024\227\091\068\086\232\170\049\076\115\163\036\185\109\140\100\181\007\066\145\099\144\251\026\212\209\200\125\140\178\203\034\139\040\104\128\164\093\137\142\071\033\246\065\105\109\140\140\108\250\054\060\092\079\068\228\065\057\017\232\131\213\025\097\201\025\099\079\236\098\133\145\085\180\037\170\211\020\184\142\092\181\232\178\199\216\208\198\068\140\139\027\034\198\050\172\150\059\037\022\134\081\196\170\042\202\161\012\075\075\161\179\150\160\132\047\132\132\098\138\098\129\157\080\217\142\054\112\163\242\068\076\108\187\060\152\015\228\140\049\022\145\020\080\199\051\153\206\196\202\027\025\209\099\212\145\036\033\049\049\043\020\073\034\138\040\113\178\120\044\250\007\209\022\036\135\208\209\232\074\202\042\181\039\066\236\138\178\049\210\248\053\172\114\166\075\055\018\121\121\013\017\149\015\037\137\088\180\196\121\016\178\106\190\080\141\152\227\069\105\017\031\068\165\098\232\250\136\250\135\177\076\177\201\201\210\023\140\223\177\248\237\017\109\058\098\086\073\018\066\136\224\202\100\029\009\143\178\145\072\150\184\148\100\135\252\061\105\070\207\163\103\210\062\145\044\067\084\073\088\145\008\145\067\136\150\153\200\187\219\118\093\014\069\136\138\018\248\056\217\228\067\139\249\097\141\177\033\234\034\036\199\209\028\079\033\031\019\255\000\012\158\053\122\049\075\240\056\146\232\241\049\223\108\081\028\015\042\028\090\102\049\196\113\037\042\062\175\254\017\157\156\108\078\135\147\254\031\185\138\109\123\027\017\041\055\232\227\033\073\175\100\163\201\016\129\202\139\114\244\142\050\042\072\250\159\244\171\028\104\081\178\061\014\116\044\182\070\073\233\140\121\104\089\015\103\173\057\208\178\035\217\009\081\022\050\132\181\229\195\226\140\017\161\033\162\136\173\051\041\227\066\144\145\040\089\154\060\036\038\100\060\104\084\081\101\158\090\180\097\144\209\146\085\102\060\060\187\063\078\100\195\094\136\075\186\050\051\006\059\236\224\101\199\100\037\248\037\042\048\198\251\018\050\196\129\057\254\012\088\175\217\028\116\112\030\051\038\018\063\181\209\037\100\099\067\237\209\015\028\126\057\053\193\145\118\143\102\071\070\015\030\209\047\028\254\014\143\100\250\048\225\229\236\126\040\227\197\209\232\131\218\215\146\173\018\085\240\138\049\068\091\161\235\033\133\116\180\207\047\029\219\049\178\185\073\034\010\181\102\117\105\152\088\217\047\220\209\138\020\134\201\043\050\174\050\036\173\024\101\069\146\086\074\060\088\255\000\115\072\132\104\108\200\199\042\070\028\119\219\035\208\164\089\100\204\190\209\008\217\145\209\227\066\251\023\067\060\136\093\152\164\053\070\078\218\070\040\210\067\086\121\056\191\038\057\025\015\027\210\044\242\023\238\067\068\004\142\034\069\025\099\102\088\211\248\099\086\099\066\210\024\198\075\240\098\084\150\242\198\209\037\193\179\199\092\167\127\012\138\209\030\155\057\116\120\235\148\196\168\108\103\147\140\131\180\127\018\018\177\246\101\199\102\060\085\166\199\217\086\210\035\026\018\037\062\039\212\177\078\137\100\042\197\209\145\217\130\020\180\140\145\179\248\200\111\162\042\230\136\174\181\053\104\146\225\035\033\227\174\150\188\143\228\181\140\095\012\135\146\190\024\145\031\091\069\013\012\107\180\064\178\245\228\224\114\244\120\184\120\123\248\072\146\169\050\125\035\193\135\228\147\163\038\094\204\114\229\070\088\217\005\077\162\081\049\202\132\134\134\134\073\148\098\086\202\031\067\118\036\054\050\052\050\043\148\136\042\070\108\156\076\057\121\030\204\241\169\023\209\133\092\133\191\047\031\228\114\179\199\143\237\067\051\247\037\168\017\030\155\038\121\043\225\132\091\090\108\098\246\136\250\213\238\135\208\165\123\202\170\068\221\158\060\120\164\101\149\013\221\158\044\172\146\050\174\050\061\146\135\228\197\043\026\040\200\248\139\247\018\232\193\249\214\103\072\066\253\199\208\062\139\036\156\071\035\198\133\187\061\025\221\179\020\169\145\060\200\137\244\120\202\219\022\242\071\146\039\143\139\163\007\241\068\140\157\200\108\128\181\101\140\242\098\061\035\010\034\045\068\162\067\023\180\067\214\178\206\153\142\124\138\214\087\072\132\255\000\112\181\157\118\123\104\132\122\051\162\168\241\149\050\143\046\031\146\062\140\147\060\120\117\101\013\025\113\218\033\046\046\137\246\120\232\226\103\143\076\076\192\136\164\113\070\120\018\060\104\082\038\079\217\030\218\049\158\076\110\044\076\241\023\189\074\084\070\086\050\088\084\157\145\141\019\037\252\137\016\066\211\211\103\144\173\015\072\192\182\136\173\072\098\246\136\013\025\253\158\054\074\044\179\201\145\007\077\017\244\052\121\030\204\043\148\144\137\070\199\128\142\058\040\242\033\105\144\149\034\043\156\140\112\163\137\196\226\121\056\235\180\070\086\143\024\226\101\133\162\169\138\084\067\047\116\068\206\250\101\091\070\053\073\013\089\044\022\071\002\069\081\150\054\135\213\158\038\179\186\070\044\180\069\222\172\144\221\201\141\016\022\152\198\102\093\050\094\244\140\043\161\139\081\211\040\170\102\063\067\060\149\068\122\102\041\242\036\232\203\059\100\125\162\062\138\060\133\217\226\070\222\164\232\121\072\228\189\077\090\100\215\022\209\227\097\174\196\140\146\226\067\045\244\081\060\124\145\146\031\078\095\248\097\116\197\232\106\204\222\061\246\135\142\095\240\195\130\187\098\084\121\044\195\027\144\162\055\067\200\071\032\187\037\232\203\026\108\241\017\071\149\233\145\116\097\157\234\140\157\038\127\209\144\093\008\067\067\025\147\211\039\249\210\246\097\245\165\164\088\245\052\097\125\045\121\081\040\134\078\036\179\216\251\061\052\065\244\089\228\062\207\015\243\175\042\084\041\179\030\078\200\187\067\068\188\107\149\137\082\025\159\037\186\033\042\102\057\114\040\242\049\088\153\131\037\233\163\129\068\153\158\086\120\138\245\228\207\138\062\160\178\118\099\125\033\153\215\103\139\235\094\095\166\036\096\157\009\222\179\202\145\021\168\045\068\099\025\063\076\201\239\072\193\232\177\050\244\181\100\143\030\087\175\033\116\197\240\174\209\142\093\018\157\035\044\172\241\039\071\035\202\149\145\031\076\195\059\067\146\066\205\098\118\101\159\017\187\099\070\012\148\089\155\039\224\081\035\046\038\060\214\041\166\114\037\052\140\153\198\236\241\229\196\082\071\147\046\066\069\118\097\149\162\082\050\246\207\025\209\103\148\250\034\136\186\102\041\217\103\145\217\021\066\246\045\045\052\072\201\233\147\210\048\189\038\033\109\162\070\060\156\079\174\100\201\200\076\122\173\044\148\060\173\158\197\251\079\174\201\079\145\232\246\071\037\015\045\138\077\017\205\068\242\114\018\034\197\042\099\204\074\092\143\069\217\116\071\045\031\092\150\091\061\151\248\210\203\067\118\055\068\136\101\162\089\108\125\144\201\196\253\065\147\039\035\208\204\121\104\126\071\068\242\094\160\138\016\150\164\198\201\250\100\253\189\035\014\208\132\034\100\165\186\030\146\019\025\122\161\068\113\058\061\141\052\071\191\193\199\073\089\071\177\068\107\076\074\206\007\018\142\135\209\236\224\070\039\017\161\197\158\138\071\069\234\142\143\101\252\044\132\232\078\197\166\049\153\061\050\127\157\035\014\210\018\040\114\162\083\177\200\076\179\177\070\196\168\081\084\074\052\113\177\192\142\058\039\149\068\253\090\035\153\072\225\098\133\018\072\150\104\196\126\082\033\153\072\112\191\068\097\100\164\160\075\202\161\121\009\145\106\067\196\070\041\019\204\162\063\041\017\242\019\056\095\103\001\069\034\089\148\073\121\095\240\253\081\250\175\252\035\228\166\071\036\100\056\127\195\129\196\104\173\115\163\149\150\088\187\036\040\051\030\074\034\239\077\141\140\202\233\050\091\192\070\034\136\145\020\072\201\035\030\043\050\099\161\070\136\171\020\005\002\112\057\180\071\178\042\145\070\121\113\071\242\040\163\197\201\125\013\030\068\248\161\020\085\030\036\185\034\107\138\179\052\249\061\081\130\124\088\163\102\121\112\069\242\211\071\137\062\074\153\196\242\114\087\072\171\018\071\090\162\171\209\143\055\031\098\169\028\071\017\196\113\037\018\049\161\059\026\215\161\077\009\089\015\068\139\024\207\034\093\015\126\059\034\132\138\018\026\050\099\033\147\137\041\114\020\015\226\066\086\033\156\005\017\145\060\198\045\248\099\103\150\045\248\047\217\153\210\101\219\123\196\174\072\135\072\242\151\066\223\134\233\146\093\025\221\203\230\213\158\046\079\195\219\067\068\209\008\242\062\136\241\180\113\104\098\086\099\141\017\100\153\101\150\121\082\248\097\116\071\208\180\145\067\141\143\016\177\208\250\039\043\020\232\142\065\072\090\145\003\205\126\190\030\034\036\121\075\175\135\132\251\103\146\233\049\105\158\060\109\141\213\025\213\166\045\248\238\166\073\244\077\220\158\226\185\014\003\235\112\234\072\094\181\067\038\140\026\163\133\153\033\070\056\136\076\108\177\177\179\200\099\222\054\098\237\034\196\045\177\147\153\005\200\158\049\099\098\077\016\122\145\003\204\246\138\211\060\069\208\207\039\248\177\122\213\030\031\179\204\125\020\086\188\085\217\038\100\093\021\236\162\140\125\073\018\127\180\126\222\252\088\242\108\120\058\051\070\158\227\237\011\208\180\201\152\221\010\098\104\108\201\043\033\026\219\101\150\073\153\229\127\008\179\199\149\173\033\061\061\101\149\028\028\140\120\168\080\056\164\057\091\035\169\024\207\047\218\213\018\071\136\250\045\030\076\186\162\040\162\143\027\249\030\099\248\120\136\153\046\208\215\108\162\133\211\068\223\236\063\238\252\073\241\109\147\242\149\019\151\055\123\197\027\099\084\143\069\235\032\164\114\020\168\121\025\143\247\009\109\151\172\158\140\142\254\009\158\036\244\180\158\242\194\200\075\137\138\092\142\036\146\036\184\200\131\211\034\121\061\201\021\166\133\107\211\046\095\244\118\255\000\034\091\241\255\000\145\229\254\010\211\060\052\073\009\116\073\126\231\186\237\019\127\176\069\020\112\020\010\210\086\096\133\018\149\252\038\133\208\161\103\018\080\049\070\139\022\154\040\070\119\067\127\031\030\084\197\218\018\248\181\102\072\030\044\168\203\159\136\188\130\079\147\033\169\008\242\063\146\030\232\162\138\040\163\002\253\199\149\237\106\138\232\240\253\019\023\162\095\201\148\080\215\104\159\240\033\186\040\162\138\113\236\193\151\151\068\149\011\114\154\039\035\019\179\038\050\048\018\161\136\108\122\179\200\159\202\014\153\134\124\139\019\213\238\074\198\248\138\046\066\196\125\049\042\212\136\030\066\253\200\162\138\058\058\213\020\081\129\126\227\201\090\162\186\060\063\201\052\062\145\249\101\020\123\104\200\191\105\004\081\066\044\178\138\043\143\104\198\249\196\170\044\147\035\014\066\241\199\030\004\101\200\075\084\080\199\169\116\140\210\191\146\060\124\133\137\137\252\101\011\035\026\248\200\198\121\030\209\090\145\141\089\244\137\199\137\018\181\129\126\227\201\137\029\081\134\092\100\123\051\078\145\014\202\212\059\145\151\248\152\209\067\070\040\216\241\147\141\016\236\162\075\163\196\125\019\212\217\227\171\101\035\060\140\107\086\088\216\222\145\154\116\077\223\207\020\169\144\149\150\038\038\047\132\178\080\179\038\041\039\185\016\060\133\217\069\013\030\050\187\056\158\068\104\130\043\094\063\182\103\143\068\010\214\072\254\081\015\034\186\037\055\049\070\145\068\140\017\236\202\186\049\254\074\026\232\241\253\178\074\140\146\032\181\035\004\232\114\178\201\043\023\237\022\102\132\249\049\042\030\172\108\122\110\143\034\127\097\024\101\098\098\098\098\098\099\036\233\030\070\082\051\102\028\196\093\234\068\015\040\143\162\138\060\111\108\103\148\071\214\164\120\203\179\047\099\092\068\086\184\033\082\220\217\134\052\100\118\136\123\101\020\040\184\190\134\228\255\000\034\199\170\050\012\140\154\020\197\144\156\206\070\039\101\151\166\203\044\070\073\082\050\074\254\206\041\209\025\090\019\019\019\019\214\067\034\182\098\241\211\070\088\112\125\024\037\107\082\034\207\037\089\013\051\198\093\177\158\073\029\072\241\081\041\025\035\100\093\124\104\147\162\017\177\244\123\066\233\189\164\086\168\148\232\074\197\030\197\137\052\063\030\199\227\018\197\196\227\100\035\069\233\141\238\232\207\144\111\236\166\097\157\136\136\180\137\043\050\198\153\139\045\025\101\201\152\053\033\244\077\114\069\052\115\037\043\048\071\136\228\101\092\142\209\201\255\000\194\156\140\075\138\036\203\178\112\063\136\178\028\199\144\115\177\070\200\046\035\149\145\100\227\216\166\115\057\156\199\144\183\033\099\042\134\251\022\114\057\083\057\153\231\100\034\086\172\108\098\214\073\082\050\074\254\222\057\209\142\086\034\203\019\214\092\092\133\142\136\226\178\016\227\169\030\196\199\027\062\152\160\093\018\098\157\156\108\224\040\208\228\055\164\236\113\177\227\062\153\244\200\193\033\202\135\043\019\046\135\251\135\003\129\244\207\166\125\050\048\161\186\031\102\078\133\049\100\022\079\253\019\229\182\095\193\244\102\201\247\019\048\228\035\222\249\028\207\103\027\018\173\180\062\143\101\209\245\007\059\019\050\076\130\103\055\018\057\044\078\244\221\137\234\232\228\115\028\142\086\080\229\071\033\074\207\071\033\076\228\114\057\022\050\201\171\056\216\225\067\070\053\166\055\241\203\146\137\074\254\236\029\024\167\098\027\161\189\066\066\248\201\029\163\150\173\018\102\044\124\133\132\207\014\036\024\138\020\071\017\150\114\161\052\199\208\231\071\043\213\015\161\078\133\036\202\071\071\069\161\200\178\142\034\137\040\217\192\093\110\066\016\137\186\050\206\254\254\044\148\067\037\146\150\226\200\189\173\052\052\056\028\014\036\160\070\092\089\250\154\050\229\230\099\128\145\069\105\196\113\050\068\132\232\150\075\035\027\035\026\040\161\162\113\178\154\057\017\069\020\080\145\090\099\213\110\075\072\114\163\046\091\031\223\070\044\148\071\178\180\144\186\035\047\157\020\052\078\002\129\244\200\117\187\044\189\056\217\147\025\024\017\085\242\225\103\004\086\168\162\180\244\246\145\196\150\242\206\134\255\000\162\153\139\040\157\156\068\137\008\079\226\181\099\026\043\075\236\075\189\175\130\248\086\208\198\088\217\098\022\167\172\153\056\146\157\255\000\077\074\140\057\108\076\178\236\178\044\076\178\203\044\189\054\094\211\249\178\076\178\203\022\145\069\021\240\075\108\123\066\212\204\147\226\078\124\191\171\025\081\139\045\137\150\088\152\153\103\035\145\200\228\114\027\047\084\086\236\189\177\252\017\017\047\141\009\124\036\054\045\161\186\050\229\163\036\249\127\094\050\163\030\081\074\203\019\019\027\057\028\142\071\035\145\122\079\086\050\197\167\166\063\132\081\020\047\133\109\234\076\122\066\213\153\115\081\057\217\127\216\078\140\121\005\043\044\140\134\247\098\022\210\211\101\150\047\131\043\105\017\136\162\113\018\040\161\162\183\038\072\189\166\057\081\147\061\019\159\047\237\167\068\050\081\025\088\153\200\108\189\088\152\152\153\101\141\237\045\094\235\073\017\136\151\194\198\203\047\086\073\141\252\037\058\050\103\037\043\254\244\103\068\114\088\165\246\044\189\081\066\099\101\150\089\122\138\016\152\153\200\114\044\114\044\189\050\076\189\089\060\180\079\047\033\191\240\044\134\074\020\239\224\190\011\086\114\047\226\132\034\203\044\228\089\101\150\094\155\036\198\057\209\147\055\252\028\239\252\069\042\033\144\140\236\078\254\053\171\210\040\173\164\037\242\178\198\203\213\142\068\178\081\060\196\167\101\255\000\141\098\153\012\180\067\034\098\119\182\203\218\248\036\036\087\194\198\203\221\156\168\158\100\140\153\201\100\177\191\242\212\168\142\090\035\156\089\044\191\141\150\123\022\239\111\118\060\137\019\207\068\188\139\037\059\047\252\251\020\232\089\069\152\142\097\100\076\083\057\022\071\118\089\200\115\030\084\075\057\047\036\150\123\030\075\047\253\091\044\228\041\159\085\139\051\023\144\126\164\253\081\250\145\249\044\121\216\242\182\057\182\114\047\255\000\164\255\218\000\008\001\001\002\006\063\002\255\000\217\160\193\131\218\167\181\079\106\152\049\253\130\177\130\240\193\131\020\096\193\130\197\139\161\127\189\090\030\162\201\192\096\244\169\116\251\180\144\189\140\112\183\045\111\186\077\108\156\095\167\037\254\223\038\147\117\248\219\147\105\127\182\118\045\240\026\155\146\075\246\169\175\180\146\124\023\114\075\246\141\078\199\195\075\151\217\181\059\004\190\030\075\246\078\196\147\226\059\154\087\236\082\066\073\241\090\155\146\095\096\146\026\151\043\183\099\164\111\141\217\036\114\073\214\220\212\152\095\176\106\094\091\087\045\136\224\193\125\217\161\131\005\225\216\154\109\072\210\191\094\145\165\054\164\134\011\152\224\240\090\199\099\177\109\173\105\245\237\107\181\036\038\238\038\108\201\039\109\073\126\187\034\095\033\169\050\159\092\087\240\024\248\021\066\095\090\144\137\187\036\133\204\150\185\037\220\085\082\072\094\022\088\073\075\110\234\251\060\144\154\226\155\100\208\237\189\013\047\154\117\052\150\244\190\178\174\224\059\150\177\123\157\004\254\068\229\182\231\175\088\090\200\098\103\069\141\132\221\095\172\249\220\145\062\176\087\026\150\026\150\230\172\072\158\220\205\075\009\243\053\038\080\071\065\123\018\222\078\042\127\005\062\045\016\077\198\137\006\167\090\052\202\066\166\219\027\031\073\037\139\133\218\088\162\253\097\060\238\183\201\147\220\036\150\114\166\123\109\119\074\110\167\184\091\142\219\151\214\147\115\210\178\038\231\170\156\204\170\030\151\175\233\212\245\036\161\046\070\148\219\084\082\248\135\165\167\067\212\243\042\115\061\047\145\234\089\238\056\095\180\221\009\177\077\047\108\187\238\233\099\117\119\039\252\139\248\089\056\039\120\023\233\138\170\118\217\081\120\075\161\036\220\183\008\162\253\046\106\176\185\237\217\095\150\095\063\075\237\184\191\007\115\039\187\128\252\023\207\217\164\216\077\093\193\073\075\076\201\124\111\175\129\120\069\119\212\116\180\159\062\022\073\147\200\141\232\075\150\243\188\112\126\163\006\148\232\099\228\166\166\191\227\253\077\244\243\189\053\204\059\044\051\126\003\036\154\077\077\123\061\235\119\006\146\143\081\126\078\109\177\239\039\171\114\098\111\170\115\052\169\054\146\093\238\176\212\226\072\035\120\004\078\255\000\071\232\134\148\163\074\110\034\137\192\127\081\191\166\149\135\164\255\000\244\185\147\221\071\035\220\117\044\115\061\069\144\210\134\181\224\081\058\125\027\189\075\186\230\240\018\092\019\079\111\034\240\186\030\155\022\048\115\063\212\127\168\194\194\238\058\198\073\147\083\177\240\050\053\051\244\151\202\216\189\058\184\233\041\216\182\119\122\184\159\035\074\112\042\079\125\086\026\255\000\142\025\052\198\072\077\246\044\235\146\095\142\072\100\206\244\196\094\006\078\045\142\164\157\101\219\211\252\119\113\169\127\202\154\091\193\105\223\146\037\083\132\141\079\140\254\063\191\002\173\224\164\228\053\127\031\248\036\239\250\157\020\205\055\082\222\165\036\223\240\134\175\228\255\000\004\155\193\076\095\059\211\113\100\166\081\074\023\229\049\187\062\015\212\136\122\020\182\165\240\095\254\167\040\127\248\095\081\235\082\200\156\036\185\169\061\235\023\166\113\090\023\226\176\123\086\185\081\147\059\051\052\244\218\182\230\016\246\161\237\067\009\195\246\078\021\080\149\090\083\225\250\023\048\135\035\145\131\210\094\050\143\072\097\079\106\194\251\051\248\233\115\053\047\003\045\142\208\244\151\248\089\033\055\123\163\131\016\233\011\199\188\117\062\205\057\024\135\047\194\109\186\066\117\233\094\027\038\119\051\181\051\183\019\143\137\212\185\088\091\007\165\179\061\074\136\123\203\057\015\083\127\075\019\162\070\167\123\080\208\204\147\114\204\196\038\204\154\031\146\105\135\018\174\104\095\103\039\167\102\198\097\115\053\251\139\083\157\141\009\195\222\171\095\225\103\201\033\046\070\167\089\166\148\044\217\030\226\247\036\077\061\171\025\138\176\252\021\220\233\071\167\037\039\251\004\090\186\025\141\225\052\132\212\146\089\011\036\207\087\164\247\022\089\146\194\154\086\153\173\144\201\103\022\185\210\153\033\146\200\170\094\199\184\247\169\103\150\185\234\106\165\125\201\169\222\025\050\100\245\019\074\174\088\201\238\050\094\022\162\209\188\102\153\036\191\007\062\162\136\157\084\068\076\154\157\117\045\029\077\037\216\086\175\034\093\034\212\238\035\107\144\228\238\120\162\072\073\046\164\255\000\145\078\071\034\214\038\151\066\199\120\118\038\190\210\077\066\200\123\143\081\169\051\078\181\246\161\165\167\184\247\023\059\154\086\061\139\033\171\249\009\053\011\030\227\222\123\142\164\156\079\248\236\073\077\043\025\169\216\210\211\170\147\114\233\047\115\145\232\082\255\000\230\019\074\110\077\125\045\046\176\244\056\147\144\146\209\036\193\038\161\234\113\103\022\093\068\149\036\073\097\171\224\219\004\017\189\042\210\191\234\193\173\057\230\050\028\238\135\138\220\209\123\138\145\153\228\212\236\150\178\070\109\083\075\141\109\199\056\121\017\162\053\009\173\026\073\242\088\162\117\017\059\023\163\176\143\066\100\185\022\199\053\135\109\142\168\127\184\084\092\164\038\164\147\004\249\158\077\078\201\038\255\000\146\235\011\041\165\196\211\218\164\185\066\112\151\066\254\214\154\091\146\234\100\234\135\115\074\242\037\210\018\017\189\079\007\068\061\199\081\122\138\213\229\025\124\011\124\137\007\047\068\029\082\059\161\249\004\255\000\104\227\087\091\142\094\245\249\026\234\017\189\009\174\026\104\078\068\146\141\072\042\047\065\091\210\009\084\207\251\029\210\026\249\052\149\077\072\075\253\060\205\045\038\237\169\114\082\127\241\067\074\066\093\009\242\105\161\041\155\114\119\037\210\042\162\154\135\057\127\040\255\000\106\137\252\137\200\069\140\196\103\034\209\236\163\093\212\148\023\224\091\031\228\081\222\107\086\147\228\227\201\229\073\126\108\053\073\136\034\138\238\130\169\171\168\174\235\084\186\136\254\176\106\247\017\213\072\212\152\112\137\214\019\231\079\097\027\201\164\147\153\223\153\053\219\067\248\226\171\009\245\184\174\171\079\082\124\156\075\172\028\162\031\149\043\029\224\115\073\010\041\249\067\124\140\081\062\021\071\040\239\053\162\147\078\087\131\070\179\097\060\159\144\098\157\212\106\119\017\181\207\161\249\029\043\148\037\090\185\127\210\075\149\074\189\133\094\167\245\023\008\075\113\006\054\050\026\130\052\149\077\094\230\174\130\019\234\121\025\228\149\072\191\228\254\162\114\038\078\011\226\134\249\026\039\194\184\093\157\043\200\114\126\142\120\189\182\127\032\158\068\111\065\093\208\241\176\173\028\159\176\069\053\182\171\139\042\181\041\165\057\008\157\068\078\219\025\133\225\097\094\162\168\231\010\040\191\200\191\130\246\216\144\169\208\078\195\080\104\212\175\074\138\216\175\138\027\228\104\144\079\129\088\056\252\217\236\227\090\114\038\057\118\085\007\039\113\083\160\225\093\212\114\236\039\113\175\143\244\220\118\219\211\022\246\185\042\100\132\212\146\066\240\178\151\052\182\008\156\212\212\075\170\159\132\246\021\167\146\081\106\087\217\194\127\034\069\244\055\200\216\167\192\184\095\002\249\063\054\165\216\211\180\244\028\060\242\164\187\081\033\091\208\149\031\144\212\077\057\029\201\046\214\165\063\218\152\131\222\042\211\173\073\037\022\050\102\026\220\034\032\198\013\255\000\034\054\142\198\166\208\209\174\139\148\240\155\018\236\042\047\040\062\134\249\025\020\248\021\023\196\063\056\085\240\078\013\079\209\173\161\206\021\015\052\043\071\032\173\135\099\091\118\102\105\072\249\166\068\169\246\169\237\088\233\117\132\114\096\081\207\023\181\019\230\042\010\148\079\180\092\059\098\092\148\254\162\115\204\031\230\134\140\138\124\010\142\241\004\225\080\106\193\206\232\120\161\084\071\119\026\234\060\136\238\176\084\230\144\254\155\182\036\076\087\014\122\193\158\041\065\027\009\052\245\220\193\136\096\244\041\234\255\000\036\148\085\063\040\068\238\035\069\104\180\042\118\021\033\049\222\118\039\208\239\040\059\205\013\025\240\174\063\068\223\153\042\090\034\193\206\238\042\197\009\067\242\132\081\029\208\065\028\107\065\021\009\236\104\036\043\082\012\165\194\194\100\233\177\113\080\106\119\017\180\120\037\208\111\113\022\135\052\116\063\005\243\178\173\094\099\133\243\066\009\224\095\131\081\222\015\209\041\068\044\074\185\013\065\041\111\152\254\127\222\134\160\141\130\180\084\161\083\177\225\068\084\017\171\208\146\115\037\092\207\234\052\152\229\019\200\202\093\224\116\019\197\114\130\118\037\210\135\056\114\247\026\189\198\173\030\068\088\127\203\181\050\084\175\131\240\116\023\224\084\119\136\037\040\053\077\105\206\182\141\078\227\105\079\048\106\119\026\148\034\147\153\238\022\074\042\208\230\142\065\103\015\234\047\044\029\147\099\074\157\151\003\145\096\202\083\185\062\176\074\208\079\003\158\057\104\210\123\140\136\157\040\106\247\017\208\252\170\105\180\225\194\193\126\005\069\130\085\248\043\057\161\042\156\241\060\141\090\191\006\141\110\231\145\029\212\177\033\026\155\051\059\138\213\202\065\041\069\053\039\040\105\038\134\105\065\167\229\089\051\087\224\158\068\165\195\154\075\101\195\252\197\126\005\208\083\242\175\195\178\172\141\073\133\174\125\196\171\240\094\219\173\017\221\033\253\071\101\073\237\118\063\170\223\216\043\122\085\161\069\233\202\040\209\041\098\018\221\252\019\200\148\184\114\026\211\101\084\119\152\175\192\186\011\090\014\242\119\145\042\016\145\250\039\138\191\007\184\093\223\193\019\146\100\208\155\138\199\018\229\200\115\106\153\037\061\055\061\166\183\230\149\083\192\141\221\084\037\254\239\254\196\165\194\169\033\083\097\124\065\162\248\248\037\130\142\065\106\119\147\178\154\146\141\074\074\009\226\175\193\202\047\154\112\096\193\122\085\005\114\229\069\090\124\026\106\153\169\006\175\224\142\074\250\214\231\040\247\139\078\012\024\169\200\127\204\054\151\015\052\154\147\099\240\065\007\124\043\208\117\042\044\052\187\037\177\025\019\135\229\041\228\252\169\085\073\069\029\074\161\165\042\238\076\212\149\105\081\073\046\236\141\061\069\081\104\145\040\234\068\165\079\209\180\184\113\050\093\080\084\169\019\184\212\018\030\126\009\034\169\216\090\092\135\236\038\133\204\033\105\022\138\120\165\190\096\148\185\071\069\190\042\189\062\013\041\009\041\042\080\107\206\196\211\115\082\157\155\083\148\093\150\137\075\135\195\073\170\164\236\053\164\197\026\159\004\223\048\152\209\022\149\063\230\036\165\177\012\194\194\171\179\040\039\138\210\167\121\038\072\151\066\091\083\021\073\157\148\157\041\029\042\106\077\174\198\148\204\063\041\114\142\082\115\021\005\065\105\101\078\030\106\039\009\082\171\005\088\034\124\018\013\094\194\141\094\226\045\095\243\008\073\078\135\184\187\143\078\075\243\020\086\083\062\100\141\036\163\035\240\081\162\047\081\090\078\054\047\086\148\134\147\193\165\105\154\154\018\050\118\204\214\019\088\105\039\071\228\024\144\114\011\066\184\240\126\082\229\031\230\050\053\037\042\163\151\185\230\011\240\104\042\066\084\234\081\169\222\098\071\049\069\017\228\225\101\046\227\172\038\118\038\090\019\116\090\211\187\073\167\034\244\088\145\104\222\026\148\236\107\066\116\201\050\077\077\078\053\055\016\188\115\012\151\088\122\114\077\084\237\212\244\195\169\101\145\238\046\105\017\006\167\252\035\093\212\242\119\143\169\075\019\231\009\182\056\036\135\145\136\106\234\076\149\050\231\006\048\152\171\240\106\157\009\154\146\057\051\025\154\197\090\180\058\022\162\240\145\098\240\177\053\053\114\072\105\092\041\052\194\195\038\075\194\197\225\146\073\009\067\066\215\053\193\165\184\059\019\108\114\094\054\046\176\154\217\013\012\052\172\045\074\189\115\200\153\046\105\077\144\154\194\197\225\202\051\228\135\131\079\052\036\073\113\076\205\075\200\115\133\248\079\034\154\029\011\045\090\018\054\162\102\151\030\149\048\096\193\055\041\038\023\201\107\209\055\088\210\200\233\118\009\182\055\083\169\232\090\117\058\143\247\083\055\041\038\226\054\036\244\061\043\078\011\157\084\146\089\033\052\036\243\211\115\006\012\026\158\075\148\117\055\037\232\155\172\134\147\170\085\165\185\140\219\152\088\201\152\232\108\037\215\225\017\123\136\166\164\052\188\155\076\024\048\122\148\147\009\172\102\077\176\148\115\028\209\147\218\138\123\011\036\139\173\054\153\147\220\123\161\205\015\082\076\186\067\210\133\204\067\037\139\164\203\180\179\082\023\163\016\201\146\237\061\167\180\177\154\049\015\113\117\057\025\061\209\193\099\212\147\061\167\180\244\162\023\088\117\066\246\061\199\184\186\204\147\044\094\054\061\086\061\043\031\082\146\097\053\135\143\133\210\177\205\025\162\106\073\035\058\166\226\212\220\177\122\038\226\201\077\201\182\139\215\054\209\122\236\094\054\046\091\106\232\116\045\126\007\038\076\209\046\103\145\031\254\126\009\016\069\111\034\124\203\165\201\150\224\102\176\177\117\050\166\084\244\184\186\067\188\103\015\074\030\167\071\210\226\240\154\102\029\227\234\088\250\084\186\066\105\009\172\044\094\197\213\097\101\058\164\059\198\073\011\033\117\046\232\123\079\105\237\049\015\075\142\165\225\209\120\094\228\213\234\102\102\159\129\177\169\082\030\210\082\145\165\120\025\172\037\085\205\073\136\078\137\037\050\082\080\156\059\018\074\100\164\168\149\115\076\044\039\029\078\044\155\119\066\109\132\148\237\179\115\008\090\194\165\093\201\173\051\227\238\102\055\162\123\210\130\175\074\214\149\090\218\180\043\171\106\208\171\182\212\238\034\111\077\050\073\073\046\214\106\153\050\127\018\138\145\237\189\056\170\214\232\046\242\086\148\058\181\023\207\007\173\055\020\245\033\101\145\101\153\237\049\092\254\010\213\105\093\223\034\236\186\011\021\173\098\212\238\037\104\155\142\078\241\114\240\018\021\033\171\101\075\081\118\161\208\244\184\196\252\023\069\248\205\047\220\065\096\218\214\011\023\086\232\183\113\054\022\046\090\242\103\101\029\025\109\078\101\139\211\116\065\116\252\082\077\013\077\193\037\218\074\027\178\177\117\110\219\117\013\242\039\000\165\212\201\101\036\185\174\113\069\221\185\052\044\095\227\027\224\185\169\184\044\097\076\023\173\068\019\200\218\219\021\131\171\112\190\096\229\173\194\249\132\251\197\181\181\118\093\224\253\138\076\069\216\110\212\137\204\176\179\133\227\046\095\020\145\146\224\246\150\077\164\027\228\074\219\004\022\011\090\142\243\005\090\221\025\137\006\214\145\109\078\241\066\013\170\080\104\190\054\039\028\195\004\190\042\229\139\026\151\221\186\167\236\027\228\074\219\184\163\188\236\175\000\218\157\226\142\196\170\068\131\118\100\093\016\232\122\094\117\240\094\127\025\055\092\181\114\173\068\243\006\249\216\110\235\188\238\126\011\005\217\065\190\042\119\136\089\167\170\196\155\084\226\155\050\044\226\075\030\074\093\167\165\126\042\074\102\185\214\162\121\131\124\236\077\105\154\108\058\009\090\080\232\045\107\004\242\055\197\082\083\005\146\185\172\039\021\222\155\031\050\075\241\151\222\108\027\231\133\116\027\091\096\144\116\029\090\193\190\068\222\154\157\163\040\046\250\241\114\227\219\006\249\225\093\006\214\216\054\014\131\171\088\055\200\155\246\141\160\141\221\183\206\054\013\243\194\186\009\091\096\216\058\014\217\111\145\056\009\166\073\044\038\042\236\246\045\143\159\108\027\231\132\081\222\118\210\014\130\236\183\200\158\056\029\073\152\072\237\084\143\113\147\074\009\243\233\006\249\019\131\081\222\096\181\172\127\005\217\068\131\124\137\095\184\201\157\137\011\185\037\045\114\095\058\144\111\145\043\193\131\017\149\106\059\204\021\043\114\010\148\181\043\068\130\121\027\094\076\153\044\167\122\218\232\206\164\166\105\243\169\004\242\054\180\083\006\054\156\047\157\175\052\033\045\148\242\055\132\077\171\124\250\009\006\214\148\173\110\138\008\181\219\220\132\150\058\151\043\090\168\177\109\075\074\108\183\233\011\067\119\029\091\163\061\153\225\079\113\053\186\238\054\167\120\167\083\179\090\036\027\243\221\027\212\201\232\089\146\114\086\234\018\181\130\139\007\086\180\075\130\149\009\082\182\062\167\022\216\148\037\222\011\243\114\234\034\071\253\221\077\043\083\133\218\116\086\014\220\159\062\003\186\240\221\227\054\172\046\218\100\123\140\146\095\147\098\119\167\087\052\173\098\228\173\209\088\058\180\170\123\211\090\149\056\009\173\119\106\022\177\233\083\218\043\156\073\022\019\249\054\121\130\199\240\114\086\176\084\173\098\176\084\173\027\084\148\237\185\053\193\108\080\176\084\223\154\157\162\187\019\249\100\081\015\052\057\105\074\123\213\166\169\165\061\233\148\036\165\238\132\219\181\055\029\137\066\084\234\077\222\170\095\110\100\190\095\074\151\061\208\239\082\083\099\213\012\195\210\074\018\162\197\208\193\130\197\201\236\089\079\082\025\051\070\075\172\207\074\023\093\155\023\177\103\081\136\228\186\150\072\117\088\201\035\138\051\243\119\185\130\196\214\180\039\027\023\050\102\022\166\081\186\070\197\227\056\078\023\141\225\147\220\123\143\113\152\090\054\162\116\228\201\147\038\076\158\227\220\100\177\152\094\171\161\208\244\184\178\158\162\095\067\211\214\139\109\223\106\092\034\164\111\179\154\046\090\158\199\165\223\073\153\051\189\024\048\098\165\093\169\147\036\234\174\144\201\147\048\178\083\114\073\130\100\232\188\050\100\201\147\038\097\138\175\066\193\016\150\147\211\143\162\246\038\155\023\140\143\059\055\163\177\109\235\157\163\216\154\112\061\084\189\083\250\079\122\239\098\077\038\229\044\221\139\158\146\244\205\015\081\101\218\186\146\105\122\186\046\245\174\118\250\150\075\161\136\096\177\125\156\110\221\035\152\217\055\046\144\201\152\100\202\025\133\144\177\117\251\012\248\060\023\052\253\226\092\022\100\100\177\053\255\000\224\100\255\000\255\218\000\008\001\001\003\001\063\033\255\000\245\179\004\142\081\202\164\157\147\226\117\211\215\224\235\246\117\141\235\028\162\092\063\246\232\019\049\048\079\185\020\046\017\026\096\141\124\065\026\129\030\003\078\021\166\195\064\219\255\000\089\002\103\176\214\036\089\032\018\032\130\062\080\089\191\130\194\008\172\078\195\102\212\024\139\063\244\183\015\098\040\108\033\026\066\214\253\126\142\096\092\069\096\136\183\129\232\108\178\191\244\012\116\211\084\104\138\016\084\087\205\194\109\210\117\108\050\203\161\020\065\026\026\061\135\007\044\013\068\127\230\096\152\088\138\197\034\164\009\084\065\020\081\069\020\047\033\044\134\130\252\047\004\184\240\018\125\255\000\076\159\248\015\241\025\023\240\049\195\127\209\061\252\008\009\197\251\208\195\169\089\101\208\130\040\138\186\008\028\017\139\255\000\044\165\065\008\182\148\016\064\130\208\066\042\134\200\106\054\217\236\063\201\117\176\230\112\103\046\127\085\018\246\253\016\236\250\033\112\240\071\007\129\240\060\018\108\250\050\011\245\064\049\148\023\183\147\032\125\072\225\205\080\178\233\029\023\066\008\162\008\025\021\036\060\071\254\069\074\154\085\169\033\016\064\130\165\069\082\032\129\132\118\088\069\253\165\248\048\047\209\138\177\137\039\064\084\144\198\153\041\236\100\096\254\139\148\227\024\254\035\142\068\209\122\111\130\008\163\024\244\040\070\071\254\054\067\138\033\086\008\034\169\009\008\036\045\032\027\100\054\050\246\047\246\142\036\125\005\192\078\146\233\038\146\073\053\078\172\070\053\061\139\205\239\006\125\009\101\228\099\174\097\162\006\016\099\208\199\113\111\175\254\046\036\052\043\225\069\009\009\009\065\082\090\108\172\143\087\242\166\005\075\226\244\105\208\220\011\201\027\208\085\074\230\160\171\074\049\110\205\074\046\118\248\015\161\108\047\023\081\101\134\029\006\049\142\175\068\227\117\255\000\135\123\034\049\069\240\032\130\066\160\084\173\171\216\208\015\004\069\247\014\002\139\011\020\058\110\036\101\184\155\116\075\036\099\100\209\162\008\129\181\089\164\089\103\218\025\103\094\141\172\139\234\012\037\013\014\141\012\099\172\081\135\012\131\255\000\006\164\073\145\070\031\020\080\130\090\013\081\075\204\252\169\064\149\036\191\017\195\074\050\049\092\066\205\040\152\143\036\016\052\072\088\058\054\153\008\153\038\173\164\148\222\184\139\048\120\177\191\014\004\171\097\233\000\107\090\052\033\065\203\255\000\002\214\020\035\226\130\005\168\166\038\197\145\150\063\039\208\084\076\037\246\200\232\185\147\100\081\084\100\069\099\100\134\100\064\067\190\188\009\193\146\040\106\069\138\010\035\152\198\105\182\144\142\112\119\031\242\070\065\008\049\068\160\208\131\029\072\214\213\012\031\247\081\186\021\009\031\197\002\009\009\160\056\100\200\255\000\145\005\101\100\190\136\135\177\053\011\034\070\056\018\108\077\066\002\052\140\010\099\141\020\192\131\171\137\153\020\116\140\061\231\142\227\243\008\243\081\141\012\058\026\249\004\127\246\231\034\187\034\040\170\168\133\164\065\005\080\036\065\230\059\000\190\072\065\048\169\028\216\129\082\178\096\144\076\106\068\237\160\247\035\073\212\234\076\145\166\132\198\164\176\074\244\113\078\001\191\238\054\098\240\089\154\130\080\208\209\003\248\229\032\035\254\210\207\076\082\004\180\018\161\002\066\066\008\071\037\013\089\245\002\216\165\075\157\132\110\044\066\137\137\139\084\019\052\064\181\035\147\070\077\026\022\172\093\067\017\072\129\208\202\018\042\108\017\135\145\155\007\122\084\065\161\161\161\161\145\165\209\215\038\250\026\255\000\174\187\225\124\048\065\021\033\081\037\073\193\034\086\075\031\221\036\150\010\065\137\146\197\068\032\042\102\211\127\035\127\232\196\009\008\129\025\048\056\145\209\186\142\005\158\087\217\187\047\181\066\045\190\002\229\165\154\179\209\037\217\209\082\016\049\006\135\084\104\117\154\079\129\255\000\245\019\212\045\049\072\164\105\042\018\130\026\218\189\034\191\160\145\056\218\130\077\197\078\122\033\098\159\233\022\187\159\131\052\224\073\238\254\200\183\127\098\042\191\200\105\184\137\034\104\198\105\053\069\075\092\200\072\108\255\000\210\075\013\159\201\007\191\099\102\204\005\232\129\009\076\006\132\129\012\168\119\254\011\059\251\076\139\073\068\026\026\026\248\071\165\011\081\191\244\073\018\095\065\011\074\018\016\138\069\016\130\066\137\017\185\143\126\063\163\100\191\152\036\153\037\018\005\169\019\154\022\079\250\028\240\208\158\070\152\133\061\135\255\000\165\051\178\181\033\096\066\161\161\106\068\146\087\083\251\164\079\250\015\001\224\146\004\255\000\168\168\223\234\177\035\030\006\156\208\106\073\099\239\061\152\220\236\122\056\197\026\024\208\208\232\200\029\029\029\025\002\104\204\255\000\159\074\073\050\099\066\022\164\105\066\162\065\004\024\165\020\182\051\212\114\161\092\072\157\168\053\013\199\131\131\107\201\192\093\108\139\173\239\193\142\240\044\016\146\004\132\170\136\029\008\208\131\061\038\027\060\088\250\054\009\011\060\086\228\164\074\155\078\060\214\197\227\084\041\049\160\066\036\055\089\232\068\129\208\048\208\208\208\200\164\013\008\069\098\136\032\138\056\161\255\000\206\040\221\021\022\149\084\132\136\172\016\032\130\137\074\009\026\097\057\036\211\035\046\042\040\068\230\224\189\199\242\089\130\069\065\084\132\042\166\152\163\210\133\145\066\067\254\078\028\240\035\253\215\145\211\120\164\104\115\025\005\201\162\166\096\100\146\217\250\102\107\072\049\143\075\029\096\141\012\146\041\031\243\100\196\085\106\043\137\017\088\034\169\009\073\004\129\218\212\130\151\104\251\222\150\021\216\145\068\030\146\010\140\132\195\209\056\164\010\165\067\212\103\026\038\032\200\167\083\052\218\166\068\141\214\103\063\016\017\108\164\162\012\052\053\170\036\132\075\082\008\162\096\073\229\255\000\043\120\152\208\181\213\022\164\132\161\075\249\056\109\142\163\156\189\059\099\018\069\016\236\052\238\059\140\191\006\201\015\220\057\135\111\070\242\100\049\041\129\078\139\052\032\112\219\183\228\177\231\208\237\145\014\080\255\000\048\071\117\065\017\112\076\144\064\178\058\090\041\166\068\137\185\019\243\191\161\105\032\199\067\029\031\194\067\016\058\070\025\255\000\033\033\138\161\105\066\010\161\081\042\032\065\033\004\167\022\115\136\176\116\116\058\153\024\068\209\216\082\182\240\141\181\094\089\178\194\227\196\223\013\161\225\001\177\015\066\032\214\025\032\180\164\034\169\144\178\242\109\052\224\037\228\075\203\099\238\006\184\183\017\134\085\232\142\155\088\191\068\196\138\208\018\075\129\158\029\016\200\068\137\073\006\132\248\094\163\208\179\035\254\052\165\146\210\168\234\086\170\016\072\072\068\016\064\168\076\138\228\095\095\066\151\111\216\186\049\206\054\051\096\178\164\036\143\001\237\181\184\230\075\250\228\090\036\150\004\168\143\194\031\026\154\194\114\201\130\017\216\153\082\170\100\017\075\017\248\250\068\006\147\208\036\017\026\008\028\171\023\248\120\013\154\243\225\147\209\053\089\008\157\104\084\224\204\162\082\054\154\194\136\058\024\244\069\017\160\180\032\130\040\142\056\127\196\153\138\170\008\032\138\164\138\021\066\161\011\080\145\004\193\035\252\174\088\139\196\033\032\164\237\069\194\016\072\022\222\228\167\109\197\037\037\136\045\019\221\207\006\227\044\083\200\179\179\247\049\062\036\182\106\025\014\244\083\205\016\061\132\163\169\020\065\251\132\106\242\209\209\009\110\239\195\141\053\017\112\251\042\226\100\135\192\106\046\066\098\186\007\073\131\127\087\039\084\144\038\004\228\065\033\155\085\038\141\236\158\242\098\115\126\198\133\020\107\080\130\052\049\132\034\143\080\204\068\127\194\073\034\015\068\017\084\132\170\016\132\033\104\072\180\042\111\010\139\135\037\037\149\147\224\092\196\146\069\092\048\032\055\024\004\054\231\221\140\013\153\157\240\033\162\040\073\126\196\055\125\246\180\198\204\231\167\180\088\163\135\237\086\234\132\032\130\063\037\216\227\117\185\232\147\016\250\234\125\005\050\212\027\101\072\181\155\199\217\120\158\056\062\004\080\196\041\154\025\026\146\018\065\096\176\070\197\034\025\241\046\095\013\004\017\166\042\058\176\052\069\097\080\255\000\133\005\195\170\035\068\008\036\065\002\021\075\081\008\071\000\063\177\043\046\008\221\047\026\146\175\081\041\161\019\167\152\165\184\064\231\004\089\214\076\174\184\101\016\084\027\141\109\133\062\075\027\093\106\093\100\085\092\208\253\170\210\091\182\194\045\159\012\197\123\055\065\039\164\253\030\123\022\006\039\113\165\064\208\247\145\185\032\129\132\153\062\166\037\163\133\208\020\099\162\034\142\183\164\050\084\033\003\085\106\141\077\137\060\191\224\072\228\068\013\016\071\196\016\145\020\065\017\084\042\016\136\023\039\178\203\080\196\045\005\229\172\090\010\103\240\133\082\152\224\100\184\164\115\008\100\043\211\060\085\046\137\232\095\134\110\077\091\018\018\172\207\060\028\036\229\085\146\051\102\217\020\048\082\173\236\009\070\048\036\007\038\185\015\046\181\001\162\004\136\032\104\129\080\130\008\171\002\014\179\068\169\042\063\093\040\137\166\008\032\066\165\020\090\009\009\009\165\033\068\096\239\001\242\056\234\012\111\067\085\093\029\076\073\123\194\082\093\044\126\102\204\007\112\054\063\116\070\045\037\228\097\253\106\018\058\043\050\249\010\095\131\022\139\072\026\024\065\082\171\237\063\012\142\165\216\124\152\138\146\207\039\098\038\084\077\211\209\096\095\044\238\056\115\246\038\075\188\145\100\071\100\072\235\097\061\005\172\137\194\092\146\053\061\144\068\210\133\208\030\168\018\162\008\032\138\061\034\036\147\104\053\250\196\164\137\240\069\052\069\088\161\081\037\056\137\016\069\016\149\008\193\146\060\164\221\089\105\073\014\218\074\042\161\192\178\045\246\049\073\248\027\225\011\062\228\110\078\087\082\010\019\067\157\008\104\085\207\003\223\168\089\119\036\094\120\033\045\191\171\141\120\244\076\092\109\241\208\049\052\011\030\229\162\069\232\089\234\011\123\195\165\005\211\068\085\034\008\032\130\008\022\138\032\244\108\100\047\214\078\201\162\214\045\007\161\009\049\000\218\227\026\214\098\252\209\136\035\075\005\128\178\222\078\140\134\033\014\095\066\202\032\136\165\107\170\232\155\216\187\103\174\004\045\005\161\152\243\008\095\011\030\216\242\048\132\053\081\008\031\054\027\034\034\161\177\138\060\210\081\103\164\253\022\247\061\143\064\106\134\042\160\129\082\041\036\017\065\170\023\074\068\192\149\250\169\213\075\081\072\129\042\200\234\244\176\239\220\069\163\072\123\041\061\133\199\073\147\125\105\002\160\090\103\060\026\146\017\168\169\067\208\078\132\146\077\090\210\066\214\208\203\176\140\071\122\073\020\018\218\046\226\150\150\104\044\183\071\232\243\030\235\116\186\032\072\066\018\161\105\073\052\052\053\090\141\049\164\107\245\049\173\041\211\145\050\073\213\149\148\200\137\035\017\139\086\037\022\092\241\072\220\073\102\055\162\036\120\168\187\070\022\138\126\046\047\119\145\133\164\168\133\173\139\037\160\072\176\181\020\130\144\243\075\201\004\129\186\016\086\022\106\197\253\094\139\186\175\102\213\148\104\065\161\004\032\072\084\117\100\178\070\164\040\054\094\061\073\031\168\148\044\104\154\137\017\036\136\073\034\028\005\070\047\166\040\092\073\183\113\017\140\163\013\005\086\097\164\238\093\016\015\020\066\038\141\131\173\102\027\109\140\108\049\076\156\155\093\196\130\090\016\066\040\138\045\014\140\217\064\250\114\151\082\205\176\139\105\127\161\105\042\149\130\036\086\134\033\185\142\195\161\072\088\226\020\104\235\189\023\117\159\177\099\080\064\130\016\036\069\029\081\034\195\252\145\132\233\079\113\183\014\142\162\052\068\018\063\076\136\090\230\133\161\081\024\137\079\097\001\017\052\010\154\203\069\081\008\145\108\197\039\189\009\109\018\016\215\029\245\054\070\100\224\225\029\019\192\134\083\242\077\082\018\022\150\208\199\068\164\045\109\159\081\033\059\182\250\067\080\048\091\094\070\188\158\149\181\023\156\161\205\129\203\067\154\011\221\018\158\097\037\159\083\097\232\154\032\072\129\033\085\140\099\028\082\204\253\012\102\207\068\043\216\082\251\024\205\097\047\097\106\058\160\106\142\187\159\166\153\252\153\021\016\240\132\177\114\175\208\226\005\244\011\075\217\255\000\007\056\147\113\024\010\129\104\168\066\016\146\073\223\007\146\245\034\206\132\168\156\049\204\106\050\049\169\102\156\112\015\115\011\113\219\246\021\236\250\008\098\106\133\162\053\050\017\019\110\029\201\253\231\125\251\185\126\172\099\153\045\040\134\138\215\150\004\052\018\123\139\072\058\202\023\216\153\163\165\170\136\018\035\075\029\025\113\024\151\208\160\046\154\106\093\011\132\066\098\184\199\070\134\135\087\019\034\127\165\068\190\005\172\069\161\034\124\075\058\120\102\250\255\000\067\200\187\176\195\118\062\134\135\019\109\066\016\132\164\071\050\037\028\145\129\003\169\085\080\067\042\088\132\118\197\001\185\214\026\236\175\179\119\242\072\037\138\091\146\073\058\045\022\016\150\124\150\148\229\214\197\189\227\007\005\062\200\105\196\218\075\146\156\139\093\180\119\116\194\043\075\009\017\147\151\008\147\059\016\153\213\169\204\052\240\070\174\142\135\161\106\099\025\136\153\154\145\221\200\074\214\185\151\155\163\141\038\053\069\133\099\115\244\146\124\043\082\081\037\145\114\107\175\105\066\019\133\121\099\068\089\027\054\113\121\150\141\160\144\130\166\160\049\100\057\185\065\130\157\201\036\177\142\068\025\162\160\212\073\032\146\022\076\130\207\233\034\083\251\077\142\011\157\203\201\104\115\216\209\178\191\078\125\013\240\024\004\251\072\159\253\071\230\232\252\093\009\101\167\116\102\211\238\076\019\125\140\193\191\147\009\041\228\210\069\209\227\144\191\007\178\075\189\075\103\119\142\043\153\021\173\038\170\218\018\227\133\027\229\202\010\112\063\216\143\090\072\035\082\048\100\070\086\254\008\146\205\059\199\133\098\130\044\197\016\069\013\008\065\004\104\036\071\232\209\190\008\212\192\226\242\137\230\030\207\242\134\036\046\127\099\016\123\022\052\009\168\178\100\144\013\009\114\028\118\093\034\006\146\099\066\229\195\036\059\029\050\067\106\150\244\110\113\244\033\149\207\200\128\154\022\251\092\118\247\062\208\134\083\236\194\183\217\195\125\200\068\225\127\124\051\246\067\133\117\104\071\178\142\199\080\035\188\143\070\214\015\236\082\036\172\151\236\053\020\074\038\041\123\033\180\009\134\193\057\109\032\150\077\190\111\233\009\126\213\186\189\073\140\229\181\244\198\175\015\014\029\005\094\179\067\176\071\026\096\200\006\110\194\118\226\042\147\163\067\068\012\221\253\020\191\162\194\244\126\024\165\113\211\133\162\192\195\246\076\170\019\019\024\083\162\098\099\193\096\044\187\059\125\152\131\108\158\072\197\097\024\036\084\170\027\211\164\211\110\095\244\201\119\244\104\078\111\031\184\217\036\087\032\200\129\007\183\131\145\225\028\175\004\072\028\033\099\162\070\073\200\203\146\122\234\123\036\037\236\249\116\079\072\181\045\146\185\046\181\006\228\088\044\018\145\046\178\236\033\210\254\004\073\230\009\058\242\074\219\095\184\245\246\023\051\107\104\221\249\014\211\094\008\033\161\143\045\138\038\187\236\246\065\097\201\255\000\115\041\049\142\140\100\012\096\019\191\064\148\145\190\005\240\153\199\131\197\023\140\224\060\196\229\215\246\096\042\136\084\038\042\009\162\021\014\151\122\048\072\068\235\131\066\051\019\002\008\170\043\137\065\105\245\059\138\100\246\092\140\053\204\140\119\049\225\139\102\231\048\073\234\104\064\167\009\118\037\119\011\236\187\151\250\150\248\035\145\009\099\143\054\043\081\008\180\155\132\091\129\204\121\004\202\009\130\115\052\177\214\166\053\059\189\007\174\244\115\014\227\111\009\253\033\151\050\198\201\033\220\072\142\254\119\018\074\049\027\139\151\155\136\228\053\161\024\028\254\190\200\001\143\081\153\019\159\208\072\254\021\173\081\139\098\245\228\041\033\007\119\046\142\170\013\160\085\039\066\026\059\033\056\229\122\169\061\018\122\205\168\123\114\065\138\073\205\028\200\198\033\223\034\102\108\031\105\228\065\167\029\083\246\039\187\238\024\009\119\036\146\073\048\226\044\063\176\146\039\052\158\195\195\226\198\182\176\125\167\153\102\161\033\075\218\140\078\136\200\092\196\089\176\233\150\053\082\016\065\131\224\036\223\132\153\014\133\252\018\013\023\075\027\113\018\194\151\178\252\151\085\100\160\100\015\113\137\026\089\044\040\037\108\210\200\031\052\099\024\199\042\172\007\111\158\037\035\162\248\149\085\101\019\066\098\102\151\154\010\146\065\193\011\160\205\216\158\038\022\098\198\241\015\052\039\076\019\040\046\092\254\209\109\038\255\000\189\013\166\174\037\050\209\018\034\137\010\194\048\201\030\112\033\096\118\056\194\116\035\003\143\240\089\232\041\023\152\187\216\107\129\212\208\184\095\068\058\009\118\240\090\148\246\014\226\199\116\143\245\046\044\204\227\105\036\066\021\025\034\100\014\035\069\131\047\022\008\017\194\166\143\247\204\027\209\042\058\143\090\051\069\201\192\080\033\171\057\209\088\148\185\022\048\169\123\123\036\220\132\192\145\028\227\143\065\209\058\035\127\050\200\172\190\005\165\018\133\124\047\018\046\055\233\156\095\160\252\197\032\065\040\246\011\190\208\101\148\177\217\137\111\029\198\020\108\011\158\227\153\047\008\121\121\028\076\189\019\216\092\153\244\018\040\065\068\108\011\032\136\037\025\070\224\132\019\038\089\009\035\163\119\027\078\149\147\103\100\207\224\036\097\254\164\112\215\101\064\147\004\233\085\076\068\133\146\100\064\217\006\007\145\042\053\006\075\046\240\134\067\144\138\093\042\195\080\235\036\138\139\036\166\085\119\080\147\112\146\068\076\018\204\133\111\015\061\198\151\060\078\161\079\044\065\028\010\158\180\203\230\185\242\164\222\020\151\072\129\123\121\240\140\120\112\031\066\044\071\210\140\222\113\180\036\009\008\116\042\109\021\223\242\054\181\158\205\237\182\046\053\217\143\250\067\012\227\236\113\116\161\045\157\197\034\206\155\216\139\019\074\029\054\223\177\005\038\025\002\012\066\017\012\227\066\210\197\009\146\072\049\121\005\027\042\244\248\147\226\062\098\076\145\209\180\145\177\092\144\066\028\169\026\055\135\102\059\136\088\016\122\072\166\095\101\254\036\138\147\065\134\199\072\209\036\083\128\095\008\163\222\066\226\145\110\064\228\090\086\135\113\191\068\176\065\203\072\088\036\108\154\054\148\202\008\031\200\139\031\002\038\169\137\150\200\091\222\009\139\078\030\009\104\114\250\238\051\098\222\018\216\181\055\062\120\016\178\175\222\072\202\120\186\057\040\091\137\150\046\036\155\194\242\199\076\247\238\065\179\149\134\196\241\019\092\097\179\031\046\198\073\037\219\250\016\241\116\038\077\126\250\145\042\086\019\069\024\113\058\039\082\178\064\158\251\009\035\014\044\205\010\217\036\012\157\114\059\198\130\231\038\150\010\098\162\182\226\104\054\006\232\124\042\049\076\146\044\137\111\228\227\124\156\069\036\152\126\070\045\084\017\132\145\052\069\177\182\038\127\132\064\138\049\212\233\120\212\087\014\134\200\197\006\069\199\054\023\033\021\070\033\187\219\193\254\001\203\142\197\159\224\129\020\138\077\029\015\076\046\126\073\025\141\043\082\082\210\226\035\250\192\092\229\224\121\179\216\119\014\111\011\171\063\158\019\236\232\102\167\099\132\092\024\026\221\040\082\017\223\114\227\048\246\030\048\089\046\203\216\143\237\150\143\193\033\008\089\078\166\040\102\023\129\228\163\172\159\206\244\101\152\245\038\051\129\101\230\048\226\025\060\194\149\252\139\132\175\212\092\191\186\068\091\046\034\071\220\137\138\103\020\153\096\181\133\054\172\163\123\035\097\008\145\016\040\150\199\057\073\006\232\032\031\226\070\058\168\129\194\205\127\093\087\113\186\061\216\157\215\145\202\001\120\061\013\185\016\221\110\242\068\151\189\037\075\127\036\056\249\163\084\055\160\108\054\050\151\116\173\027\215\136\057\143\192\080\180\080\077\178\036\078\020\046\019\192\221\254\027\005\050\228\144\242\059\184\016\058\049\235\079\242\119\052\198\187\239\251\073\200\194\241\003\099\059\060\179\100\031\182\052\137\083\193\093\182\109\208\144\112\135\210\029\094\254\208\220\255\000\137\143\087\013\251\146\039\006\032\123\193\246\049\252\223\219\108\090\120\221\206\090\030\229\119\126\227\050\171\088\221\162\059\166\238\038\030\082\094\004\132\247\183\129\207\141\141\004\090\096\220\191\070\227\003\113\028\215\228\121\103\230\153\223\103\150\234\131\040\083\180\060\254\192\185\055\232\216\142\090\028\034\227\072\179\248\046\145\023\219\022\074\081\216\182\203\236\131\014\068\046\070\211\183\001\042\189\125\137\043\249\012\237\125\049\019\195\228\192\114\203\141\164\108\169\113\226\072\066\083\035\110\235\240\100\127\096\251\198\051\133\254\069\110\254\063\131\242\023\240\055\250\017\249\003\062\248\098\223\102\243\043\236\084\037\192\156\238\071\138\100\115\189\216\234\055\118\098\152\202\024\187\035\251\009\249\217\135\051\089\004\183\154\019\165\224\109\064\133\151\127\035\075\108\112\249\206\028\080\053\139\031\129\075\063\224\169\052\044\130\010\149\185\192\226\047\026\224\142\103\131\104\055\216\124\196\119\253\049\083\178\024\250\003\214\073\080\072\248\224\090\085\086\149\045\003\143\170\246\042\218\130\229\120\033\110\233\060\224\099\128\110\220\201\221\008\046\142\060\208\173\229\061\080\198\101\012\252\110\050\118\125\009\196\106\226\119\253\217\017\110\240\187\013\237\166\229\005\225\178\107\193\202\150\242\201\149\189\076\134\095\114\046\201\251\137\075\234\027\079\131\019\179\031\084\033\222\228\207\229\025\146\011\026\017\156\038\068\066\030\031\067\031\026\246\146\098\075\114\082\205\170\166\177\094\063\174\223\040\079\194\107\237\052\033\008\090\182\116\191\224\133\229\150\203\063\033\099\120\223\099\239\126\204\070\081\098\047\221\035\182\069\232\108\016\150\182\225\033\219\083\030\196\025\205\191\165\220\143\100\190\019\100\126\208\035\013\132\112\129\196\062\199\197\076\154\022\059\237\127\036\235\165\195\102\068\120\107\233\141\121\127\162\005\028\191\208\158\191\225\002\189\239\076\074\002\082\119\108\074\161\056\111\006\215\110\165\178\019\186\019\225\125\108\098\082\190\132\170\108\040\038\065\034\201\002\222\223\110\163\035\105\147\251\064\098\092\207\168\240\034\122\141\038\252\210\139\055\061\158\204\099\167\219\209\023\169\020\228\091\030\018\028\210\089\228\105\246\048\190\198\047\023\113\254\047\081\092\168\073\147\116\113\246\049\150\085\185\192\169\008\045\022\042\199\071\166\023\063\018\075\018\035\224\085\106\123\145\110\074\150\232\088\150\112\040\101\162\011\113\026\215\091\200\070\171\131\249\016\156\025\222\035\201\019\177\123\039\117\100\081\213\204\150\125\189\234\170\149\067\139\156\100\059\149\061\159\017\251\051\119\187\255\000\071\245\198\069\242\252\136\017\123\098\056\159\100\237\255\000\210\070\045\174\070\043\240\199\238\077\175\173\203\096\217\013\245\152\205\133\074\230\103\254\019\138\191\148\181\236\197\181\113\024\147\044\151\219\020\151\048\251\021\239\147\025\178\088\182\177\139\083\119\177\123\230\253\200\113\108\088\047\195\164\180\114\017\146\207\150\199\217\128\205\187\034\067\156\115\196\235\036\245\204\203\006\054\176\155\239\037\144\024\252\016\210\123\012\086\018\063\216\001\018\003\057\183\246\100\119\191\192\248\171\125\005\113\123\146\239\228\204\155\184\137\170\239\143\236\063\250\137\151\038\066\094\086\043\002\123\111\177\123\216\119\098\149\157\224\108\179\127\169\192\242\047\019\238\129\210\223\091\166\061\100\242\091\050\244\207\236\047\069\147\184\181\238\083\201\110\200\174\142\173\155\023\238\104\072\199\153\188\176\139\052\164\158\215\076\221\025\246\182\029\004\227\209\046\185\063\067\153\193\036\213\162\007\166\117\063\021\239\146\001\201\236\136\114\067\224\252\072\146\109\195\083\129\165\156\092\186\056\028\228\242\167\179\069\253\119\145\062\230\252\019\205\153\246\065\206\150\250\157\072\179\008\067\003\204\080\154\210\228\088\061\125\075\101\131\164\144\236\182\093\086\195\208\027\100\051\112\039\108\168\108\165\047\184\030\219\115\250\216\098\164\222\007\203\204\114\123\015\232\016\130\194\006\184\061\012\043\120\234\076\073\164\242\244\193\023\056\094\073\070\092\014\018\070\109\069\204\130\004\140\093\081\196\007\030\017\176\131\177\225\192\113\209\017\202\239\155\046\077\060\184\012\099\144\145\020\069\076\086\155\126\212\248\133\173\146\125\180\137\046\003\027\107\216\164\229\231\209\105\227\232\094\235\208\216\249\093\222\133\169\013\123\143\136\090\030\009\035\016\105\148\240\095\077\033\174\220\205\125\049\042\226\139\016\079\202\076\149\157\073\198\222\089\052\183\125\016\110\004\248\140\086\100\059\235\240\150\119\134\230\184\156\008\253\136\041\172\050\088\116\038\120\006\245\096\235\044\192\145\191\147\013\185\067\227\255\000\081\043\054\125\178\103\136\243\162\105\045\015\077\204\063\135\123\084\084\138\035\186\145\138\236\075\002\052\055\180\070\155\172\078\206\211\244\079\034\244\073\042\150\037\246\041\090\221\030\089\008\155\165\246\087\034\218\250\132\070\112\065\050\230\140\056\208\065\025\177\248\056\194\217\062\217\073\020\166\203\063\036\205\032\131\016\228\047\114\020\198\118\221\127\180\126\078\135\098\051\034\008\065\020\067\038\205\122\019\026\146\238\008\091\091\196\190\173\073\023\054\094\168\138\026\046\175\015\104\138\127\098\069\005\203\130\251\057\212\187\155\026\228\210\052\065\004\016\065\004\086\089\095\152\047\236\098\071\031\111\251\028\220\103\219\023\038\205\027\113\185\033\219\207\209\033\004\016\071\098\081\184\250\073\106\175\228\035\155\136\129\142\106\073\113\056\181\236\232\063\146\050\190\252\189\024\223\146\042\242\106\122\150\204\115\029\166\057\173\137\007\202\193\208\080\094\143\074\184\034\111\039\147\185\151\163\222\167\012\248\214\126\020\013\240\192\180\162\070\036\116\068\234\072\199\144\234\051\240\077\206\109\040\129\156\029\112\159\065\108\188\236\116\164\031\131\159\163\181\075\244\151\193\049\046\009\236\186\242\250\049\118\044\129\173\127\131\025\199\210\123\016\207\047\163\106\233\165\144\225\011\152\120\018\200\187\187\156\164\075\086\225\063\101\151\252\145\063\201\068\057\052\084\067\160\214\209\220\067\022\241\110\077\014\198\121\114\248\147\098\030\140\073\101\155\140\151\216\247\188\068\183\032\236\076\134\043\064\208\236\035\042\009\088\166\072\160\182\112\022\222\242\240\076\252\144\203\011\189\193\195\057\151\209\011\083\189\186\034\005\169\026\229\039\145\043\076\193\146\110\104\182\057\005\178\017\142\066\039\138\007\104\237\162\049\219\035\132\022\133\222\029\211\029\007\241\019\156\066\221\247\060\103\182\058\248\111\101\221\085\232\111\056\215\117\049\086\042\232\157\023\190\005\151\242\164\179\030\255\000\100\157\226\243\212\221\245\122\164\153\092\201\236\150\118\101\157\179\109\058\050\109\130\193\219\165\240\166\151\058\227\116\053\018\184\079\129\122\017\055\010\004\179\254\203\038\222\195\075\016\169\002\047\148\076\251\122\103\012\027\135\070\032\229\155\231\132\008\064\066\190\194\170\036\149\128\159\044\155\247\027\203\123\232\080\112\145\125\088\186\246\200\140\055\036\033\013\172\251\104\074\023\082\042\201\016\133\151\106\028\109\201\061\145\066\091\013\208\141\250\062\136\251\041\062\217\122\068\243\101\062\009\098\114\236\057\027\216\119\213\146\237\238\225\163\137\172\252\029\196\015\163\144\030\004\092\208\222\156\168\220\075\239\184\107\196\013\109\179\113\210\151\015\169\245\239\111\071\138\246\134\251\215\163\222\051\239\076\168\245\157\036\083\240\037\254\085\204\107\253\141\061\070\125\182\242\127\122\209\019\210\239\177\027\254\006\075\241\203\251\036\077\219\017\058\153\201\169\242\114\057\188\177\031\140\177\036\235\224\217\219\190\142\102\183\088\078\088\211\070\131\147\006\011\167\222\012\032\133\244\041\155\030\071\090\203\217\126\149\022\227\077\094\011\072\133\172\185\251\099\182\109\229\201\022\219\181\040\184\151\241\015\067\223\120\221\249\113\073\036\022\043\054\047\129\112\097\081\254\105\014\177\151\237\156\149\122\138\153\189\207\091\147\126\047\035\098\080\191\013\161\189\219\207\154\036\136\141\024\208\070\214\247\251\038\218\199\177\036\184\214\114\046\124\019\038\225\167\083\017\235\145\220\129\176\207\052\199\075\123\186\126\244\146\075\186\111\104\123\124\215\163\025\151\118\049\045\078\174\151\151\175\045\073\214\134\103\152\046\234\008\024\224\046\164\234\220\221\071\130\224\203\131\234\145\013\186\195\223\184\169\138\073\036\141\144\126\057\071\095\029\104\188\141\162\223\216\196\163\145\246\209\135\082\073\057\105\014\008\174\018\109\125\166\169\197\247\080\117\063\100\140\163\175\129\041\076\171\062\194\155\161\195\188\019\111\237\040\118\239\205\008\154\073\034\173\163\113\078\255\000\049\142\002\177\020\220\151\212\233\082\000\164\098\240\248\068\181\141\037\188\136\075\118\002\103\215\217\192\031\224\118\001\111\162\056\238\067\182\079\192\132\159\200\054\082\194\118\112\023\054\096\176\226\054\073\058\236\079\219\250\100\065\048\065\211\019\107\240\102\052\036\066\191\104\125\213\197\057\243\046\169\164\051\034\102\188\152\137\159\204\154\073\119\069\236\243\063\098\212\051\238\120\021\058\177\233\053\054\022\031\192\122\036\157\108\222\108\104\093\098\246\250\191\099\065\208\047\129\160\123\018\127\232\248\158\222\235\036\234\136\113\066\035\136\192\229\247\161\025\123\231\162\067\164\145\166\115\244\072\243\179\146\245\176\154\094\119\211\233\155\123\121\236\216\166\099\032\218\227\087\057\071\068\069\255\000\161\019\153\036\210\073\038\129\146\048\190\153\196\247\158\074\164\043\230\099\091\238\178\060\092\103\053\063\050\038\238\098\044\038\118\240\197\141\244\063\224\123\191\067\254\006\163\104\236\208\163\015\128\165\043\246\010\096\159\177\044\217\088\149\178\158\058\068\137\199\201\050\106\205\204\112\205\214\098\031\008\240\089\251\035\250\185\049\050\248\037\206\126\135\157\108\100\067\191\076\138\136\183\212\248\150\175\127\068\141\230\209\127\085\013\135\095\069\234\103\221\210\016\129\209\140\117\116\076\205\174\199\192\180\249\076\242\131\087\239\251\150\246\204\062\023\135\164\033\036\215\174\200\255\000\038\041\194\004\242\201\117\248\100\205\049\036\076\185\244\115\193\126\155\130\035\228\073\072\216\181\126\036\234\019\249\059\161\206\059\184\199\135\146\063\209\208\185\172\050\073\036\146\073\059\166\078\086\113\220\123\207\196\078\135\019\031\068\211\223\201\110\228\019\197\073\058\040\147\129\129\015\047\054\046\207\046\150\019\010\253\038\108\252\014\015\193\157\079\162\251\003\240\055\178\060\024\230\220\057\192\067\060\054\207\237\018\182\222\242\077\018\231\232\033\110\240\067\197\035\170\073\163\046\091\195\047\012\228\227\099\193\061\197\047\177\035\230\162\073\036\154\072\153\146\102\015\232\121\226\110\169\165\003\094\158\201\248\068\140\230\172\136\043\023\039\251\030\178\242\211\166\168\199\087\084\214\088\190\005\167\039\115\192\100\254\028\207\091\075\163\058\230\006\164\177\236\120\231\068\167\154\036\230\016\206\112\055\210\183\236\115\253\186\220\250\151\146\037\050\210\242\132\236\124\033\253\057\026\055\193\031\213\009\167\038\154\246\069\057\139\236\236\066\206\135\008\220\079\049\112\192\198\236\168\238\033\041\129\058\036\154\051\155\065\102\155\103\161\030\216\056\112\095\098\082\215\054\188\184\022\047\079\068\254\196\146\073\023\208\058\078\005\097\114\114\229\151\157\094\040\176\188\072\101\236\028\018\094\253\083\030\247\105\250\034\221\208\136\185\034\097\177\178\001\237\236\066\237\028\181\037\246\069\199\082\105\120\160\252\092\253\150\038\044\095\137\130\070\115\123\164\147\084\146\049\176\108\059\141\097\112\126\133\191\090\147\068\232\145\058\120\055\152\022\105\087\246\143\071\240\081\069\213\123\035\090\022\151\148\207\032\044\175\198\079\092\085\103\063\135\184\158\198\049\242\168\180\201\073\188\121\098\046\139\246\072\120\214\105\003\249\061\146\075\161\113\246\158\087\066\196\073\039\081\151\131\234\094\012\004\058\218\087\129\252\017\177\058\021\206\243\046\232\105\245\060\049\183\131\080\039\044\127\070\073\074\150\221\114\045\182\090\239\044\114\104\146\073\062\164\091\075\171\190\104\141\029\060\239\129\120\030\208\254\031\068\146\077\072\145\216\049\250\246\117\186\122\044\135\093\009\020\072\095\107\039\057\159\116\083\073\036\107\018\089\212\119\210\177\201\229\246\118\231\209\053\133\235\094\014\114\080\191\155\116\110\239\236\157\060\210\071\041\114\143\033\115\038\109\147\084\206\224\038\121\033\144\052\142\216\042\058\037\090\211\142\164\148\248\069\161\158\123\046\234\007\103\223\247\047\125\005\141\013\039\154\246\061\118\045\228\141\172\057\117\162\211\209\151\125\017\210\033\120\233\161\031\081\061\139\055\068\115\081\008\055\120\093\140\088\085\178\035\066\007\242\016\220\072\136\155\209\081\042\119\063\039\092\231\236\075\086\201\240\077\112\082\079\166\222\141\203\066\007\036\146\073\036\067\015\251\161\236\133\153\240\035\137\163\009\213\123\046\093\061\010\194\073\036\135\109\033\220\008\185\052\033\005\148\189\019\161\236\052\051\086\057\036\051\152\157\087\248\142\242\081\052\102\105\195\115\228\096\218\202\068\218\183\111\005\231\150\139\198\142\133\060\179\182\094\079\226\055\215\245\166\200\043\131\102\147\084\201\157\023\014\132\188\184\240\041\169\118\153\008\209\004\124\048\129\162\008\032\141\009\004\081\141\246\179\014\134\088\223\087\236\250\084\045\027\167\205\016\175\241\003\141\191\212\115\182\180\072\180\054\101\058\125\016\085\178\019\244\154\102\060\031\144\224\065\056\054\254\137\095\010\013\232\168\162\223\201\014\062\069\126\050\080\132\146\091\249\136\059\139\192\181\173\194\138\239\158\133\132\092\013\029\201\036\146\073\169\208\118\030\087\246\077\013\204\006\157\019\158\017\224\177\082\078\068\180\068\243\189\120\153\191\065\151\157\134\120\027\038\238\188\140\158\040\221\074\229\113\254\081\003\199\247\027\249\205\096\058\078\226\159\216\225\249\015\177\191\052\077\038\033\243\094\025\016\249\061\031\064\246\060\046\072\122\018\123\158\132\059\057\253\143\103\226\042\181\103\126\088\105\124\229\140\100\101\237\212\245\064\146\195\211\053\007\081\026\026\008\032\138\047\071\081\046\250\050\222\163\246\090\133\163\098\199\241\131\041\144\221\217\006\191\070\151\124\011\014\205\161\246\079\178\238\146\210\127\193\249\124\137\135\007\182\118\090\162\068\077\085\102\156\209\018\102\031\068\216\234\065\205\033\206\108\044\014\066\039\068\209\050\062\050\250\066\157\188\199\247\047\056\241\034\036\215\009\027\209\056\254\050\033\030\083\045\161\140\183\179\183\115\141\166\190\132\073\091\218\134\049\036\117\247\213\200\244\228\169\117\146\073\113\213\125\213\224\181\242\248\101\157\141\062\112\228\219\123\035\182\179\240\247\239\209\040\226\196\096\176\253\024\243\215\122\143\068\009\081\097\153\189\067\214\181\046\033\147\238\091\214\100\250\224\152\191\157\045\216\126\194\091\108\023\022\223\036\049\205\149\162\049\181\139\123\129\146\124\158\198\158\131\214\159\232\252\221\008\255\000\044\019\063\203\104\084\145\058\077\109\105\240\253\140\207\196\013\225\246\028\008\206\095\100\180\232\033\118\039\073\036\113\055\150\123\161\235\153\217\192\158\119\135\252\147\092\024\201\036\076\115\193\192\187\207\204\142\092\161\249\185\059\129\118\039\208\158\123\023\014\157\045\039\107\063\185\046\032\105\211\058\251\235\232\140\224\038\241\232\116\158\171\104\246\038\070\232\111\009\159\173\120\050\097\205\232\203\171\126\203\119\066\123\240\174\223\122\186\045\008\209\139\050\122\114\099\248\017\165\018\185\049\125\136\020\116\180\144\047\155\036\085\247\200\008\183\183\251\050\065\010\204\106\060\014\083\136\181\153\108\118\246\075\210\122\021\017\072\186\236\126\035\129\213\210\072\238\111\122\024\208\174\115\194\227\126\168\047\097\035\071\242\056\138\164\078\238\058\242\026\199\121\236\134\201\175\079\013\150\226\089\126\070\134\034\211\253\071\028\213\251\031\112\186\073\211\022\132\193\105\193\112\091\003\203\016\059\119\157\011\023\148\153\213\199\073\104\104\159\119\211\057\143\162\028\183\209\049\183\131\053\101\161\072\126\159\216\111\022\134\126\253\010\065\223\078\132\007\123\091\225\208\123\060\083\200\178\190\132\048\232\056\020\077\047\089\150\054\149\207\198\200\216\174\052\012\200\219\177\056\226\028\222\153\085\193\179\190\203\204\175\198\141\185\060\143\141\063\130\050\034\168\182\228\011\178\036\063\016\037\013\118\246\096\232\244\097\088\047\087\020\246\088\232\073\190\050\127\122\020\219\033\219\097\168\138\059\121\022\151\120\243\161\133\041\048\200\211\016\234\067\224\093\030\067\165\187\050\041\004\077\080\070\227\073\136\089\221\117\017\187\047\069\143\106\178\068\052\073\184\229\071\071\021\042\092\193\118\220\230\180\147\062\110\142\157\012\077\041\091\236\092\039\130\126\012\102\226\133\138\178\053\029\241\005\241\250\208\207\221\060\177\232\093\163\232\075\054\121\032\229\228\234\231\146\029\133\020\108\187\096\134\060\159\028\244\101\211\139\211\154\069\014\142\136\234\017\111\244\070\211\226\115\001\190\206\160\208\233\008\165\205\251\083\121\094\209\026\140\219\185\156\008\176\014\120\107\010\055\030\015\170\122\018\208\033\220\158\009\206\194\105\008\201\222\068\196\119\116\051\185\060\033\045\014\121\067\090\068\213\023\039\129\023\225\098\203\156\154\057\083\075\152\230\190\136\163\111\076\114\158\232\018\005\182\122\035\090\003\252\145\042\151\003\171\069\207\147\029\219\239\165\157\027\002\076\186\220\155\238\047\231\068\066\097\198\005\151\244\164\085\061\232\025\005\207\030\081\177\192\174\065\020\116\018\241\116\209\124\044\233\004\129\252\007\072\034\153\116\035\023\083\164\137\147\065\233\165\156\158\201\018\124\145\058\245\018\110\036\060\102\136\030\018\111\126\044\142\034\011\165\222\132\197\198\125\178\094\076\196\073\236\093\018\075\102\013\253\095\134\055\228\236\037\086\066\062\004\106\015\169\032\138\049\246\164\191\170\046\151\254\210\151\222\100\076\117\082\104\158\165\072\180\231\161\040\098\092\114\067\249\030\008\251\224\035\215\053\154\041\043\154\054\184\064\246\037\075\106\053\048\127\003\099\102\104\111\029\224\148\155\230\099\092\196\093\247\164\018\071\172\157\078\102\209\028\153\062\212\064\057\191\122\030\249\111\107\216\176\114\130\052\098\124\193\173\246\144\221\063\066\218\179\043\204\014\115\109\034\208\091\225\004\189\045\151\154\109\105\055\001\067\248\017\073\024\220\203\161\025\009\027\163\165\131\045\170\039\212\189\147\211\030\142\145\147\147\137\236\134\249\011\066\094\126\014\136\118\138\213\074\056\137\049\055\009\246\102\019\108\150\067\143\048\129\011\097\149\063\067\028\182\021\046\240\095\217\120\044\109\164\071\030\135\181\189\100\104\130\027\243\022\139\241\098\087\197\179\174\224\131\096\056\088\198\069\195\036\145\088\164\144\203\063\067\158\068\016\065\211\145\199\135\098\100\092\096\135\142\212\243\030\239\153\248\019\188\080\090\162\092\006\202\036\042\103\226\024\236\137\167\106\109\060\163\152\162\008\032\138\043\192\115\123\036\085\103\100\057\255\000\066\084\072\152\239\099\034\146\181\009\131\240\218\022\247\023\236\231\084\121\034\126\103\226\119\025\215\063\117\130\022\045\251\144\029\036\146\252\088\118\127\116\129\139\041\188\147\046\111\004\146\227\036\130\123\027\156\175\070\046\240\180\039\028\185\250\031\206\094\196\187\196\140\029\142\154\021\091\036\157\083\171\062\150\179\163\038\166\045\037\068\177\118\236\070\120\178\076\046\126\133\077\202\251\058\064\156\141\146\066\226\122\059\165\217\073\069\065\009\254\072\134\239\182\049\229\207\122\126\125\098\192\195\074\070\165\049\251\019\231\015\216\242\096\253\169\023\012\140\248\107\095\113\188\222\120\033\115\115\036\220\063\184\250\105\098\083\164\242\071\046\036\117\149\004\018\215\244\008\054\031\148\008\091\183\226\116\135\225\099\078\208\132\110\009\163\062\081\061\222\077\144\088\168\091\094\067\030\246\082\074\216\067\020\182\088\141\178\226\072\253\132\003\089\026\048\061\155\023\134\229\159\228\144\207\145\138\096\163\114\222\008\153\107\185\011\115\216\229\164\152\028\210\082\145\186\037\128\066\213\222\003\007\245\135\063\005\251\016\239\241\204\233\239\193\216\110\195\041\251\131\043\147\075\049\216\254\015\195\044\037\041\193\224\237\055\161\087\220\040\008\037\056\246\029\113\068\219\014\081\097\123\032\153\109\197\177\142\223\038\216\165\167\148\078\243\017\101\071\106\034\240\068\228\188\121\100\154\119\111\044\227\021\167\163\047\165\086\048\123\009\170\242\048\090\223\177\036\140\079\002\003\202\209\191\082\193\187\253\201\009\218\236\133\045\191\099\158\077\213\233\129\177\180\231\210\217\027\248\132\033\021\135\059\115\005\003\167\217\147\132\212\139\110\222\071\225\038\114\236\110\185\035\239\225\023\173\133\058\122\037\069\200\157\202\047\044\101\169\127\166\049\155\137\030\003\212\063\179\045\176\149\074\078\009\253\139\171\249\073\136\194\063\164\019\122\188\068\239\086\016\254\199\226\102\073\067\073\104\056\136\127\062\139\056\013\224\125\143\104\072\071\024\254\238\054\219\145\046\221\026\229\249\004\098\155\011\139\156\052\096\236\134\155\075\191\177\155\049\203\065\150\187\137\215\126\003\104\106\014\003\056\208\083\001\047\003\146\096\017\220\254\076\034\122\251\063\141\009\066\204\017\191\017\027\241\000\106\222\204\154\012\149\251\156\129\029\172\093\165\222\126\197\090\159\096\246\239\159\220\093\220\197\224\134\024\143\101\189\227\201\188\026\214\073\248\222\137\083\127\008\177\049\253\143\228\073\192\103\026\160\033\177\120\097\012\091\126\197\140\030\045\250\017\035\149\216\099\003\191\017\010\111\012\122\232\119\197\190\213\190\217\048\190\157\011\187\003\151\085\145\218\037\223\156\070\061\044\154\069\017\092\154\095\052\157\033\208\136\016\168\203\086\211\251\035\165\231\204\051\215\244\049\192\100\047\220\229\249\048\054\028\075\114\066\018\017\035\125\196\154\050\112\152\044\113\073\038\208\201\184\146\091\200\188\137\127\074\170\120\180\091\203\047\039\010\030\026\126\194\089\094\009\061\188\024\228\254\135\043\176\253\141\175\113\034\089\096\182\178\250\023\037\064\213\177\062\198\066\209\105\108\095\251\067\171\162\204\210\219\082\240\110\215\128\233\054\069\130\064\133\189\132\242\091\218\027\126\033\223\224\229\038\113\062\004\028\066\234\095\203\136\126\192\239\099\048\151\067\219\120\039\195\193\192\120\022\227\244\049\123\023\209\184\181\206\111\079\012\138\086\005\049\223\137\152\240\242\103\124\009\127\161\253\056\065\038\054\156\143\120\217\124\208\229\180\046\151\157\113\099\072\065\132\045\010\253\135\127\065\038\089\199\007\204\195\007\013\251\016\214\239\236\158\039\246\067\022\161\091\142\199\247\089\035\155\158\254\198\165\170\114\049\154\102\124\215\113\034\170\003\108\156\140\049\099\125\111\247\035\184\239\213\145\165\024\244\189\018\061\025\180\228\254\034\162\163\071\043\019\232\124\208\144\223\202\084\048\155\212\251\209\028\255\000\213\059\157\002\092\224\205\183\196\099\073\045\208\193\125\199\232\103\017\201\150\112\135\200\114\049\113\128\101\222\194\091\207\099\161\244\169\103\207\102\192\047\188\120\165\032\207\096\135\127\072\174\030\195\040\125\194\034\104\032\193\145\255\000\049\187\016\178\060\205\208\251\208\155\092\014\176\049\040\250\151\022\208\032\026\182\164\097\119\109\140\055\128\194\189\169\019\114\196\040\250\082\065\252\036\024\029\139\102\060\122\029\117\221\196\164\140\076\240\140\055\128\239\035\029\014\231\027\121\055\071\177\029\252\075\048\191\070\212\151\099\034\254\139\177\023\189\188\031\224\012\236\099\225\220\139\003\236\143\196\145\136\085\244\103\231\220\104\220\217\140\144\121\119\147\111\232\092\222\120\027\079\001\024\110\200\201\159\112\168\217\152\060\004\190\141\135\153\196\243\051\254\196\152\092\126\134\196\180\177\088\088\054\145\018\018\095\066\110\153\117\026\182\026\218\011\158\034\052\112\184\238\101\037\215\057\034\252\096\158\140\082\179\164\244\078\146\246\214\070\152\021\021\009\080\193\012\013\229\232\074\180\213\134\058\048\096\033\220\091\121\057\003\248\177\149\242\162\042\027\006\066\117\068\176\023\124\094\052\005\135\183\109\206\088\045\160\194\079\114\072\128\073\128\091\054\177\175\009\069\007\052\034\242\008\019\118\075\137\046\034\110\038\219\064\027\036\213\232\123\194\046\215\178\203\037\003\102\117\121\039\196\079\184\243\148\014\220\034\064\157\082\204\177\024\053\061\134\236\117\029\068\184\151\089\220\074\221\140\115\021\056\231\244\046\188\075\008\224\033\188\078\189\051\214\072\226\003\027\113\117\176\103\053\068\023\037\153\020\136\004\209\152\127\033\034\128\054\230\228\123\096\072\144\253\041\100\247\038\232\187\111\099\087\101\199\064\212\029\108\208\232\169\098\102\122\113\124\008\074\149\066\010\214\241\094\089\006\033\198\121\137\034\089\035\193\023\029\183\035\121\147\240\068\050\020\069\008\192\050\202\066\077\012\060\025\188\081\208\188\090\197\114\081\212\036\250\046\017\246\184\151\151\119\095\192\199\242\033\255\000\201\115\118\023\029\135\117\082\073\048\197\033\043\196\252\012\252\100\109\228\216\156\017\203\111\184\221\197\119\016\088\056\193\254\140\009\155\107\113\115\238\037\049\118\092\096\207\013\245\132\114\060\141\097\181\220\090\242\246\026\196\093\172\069\208\090\199\074\018\013\161\204\036\151\055\135\145\076\142\240\062\111\217\152\215\211\047\158\011\142\092\142\083\249\039\069\002\206\001\236\254\042\177\183\158\196\179\051\153\119\017\156\167\209\200\012\220\100\174\226\063\146\230\216\094\002\059\064\207\196\155\048\191\186\183\077\170\069\014\185\004\140\243\026\200\159\213\025\024\145\034\025\190\047\096\066\210\203\237\008\174\092\056\014\101\072\041\184\019\182\148\053\161\168\208\255\000\001\225\135\167\006\136\032\138\042\017\076\028\023\050\026\018\032\230\112\102\195\232\113\145\027\165\233\229\068\175\236\183\100\056\209\020\065\021\179\172\094\076\167\067\149\149\054\161\119\244\039\036\174\049\147\068\200\181\009\034\027\113\244\136\153\048\188\112\047\017\139\101\145\012\099\115\084\224\098\014\074\120\231\070\028\017\049\010\182\228\132\184\210\050\069\024\136\114\050\070\054\230\168\238\108\109\033\234\081\021\149\252\146\116\129\088\150\204\076\039\236\050\022\091\038\231\095\116\080\155\123\012\152\131\177\038\054\078\169\099\023\050\205\224\055\197\121\046\208\236\196\168\202\149\145\020\130\052\138\143\031\018\074\082\238\200\070\227\035\150\190\069\082\084\033\246\103\030\240\015\255\000\020\142\224\052\097\008\075\226\021\070\212\181\163\029\010\169\104\192\172\056\123\158\241\082\218\026\066\130\048\238\194\139\089\033\112\067\016\036\065\003\068\176\034\069\064\169\000\136\041\006\154\008\095\169\211\095\177\114\036\057\023\046\148\027\183\036\207\192\012\193\187\028\244\154\197\034\137\065\107\082\145\033\094\244\167\004\155\225\020\184\215\035\192\115\245\062\164\179\118\130\104\254\038\044\152\182\158\068\111\129\161\144\012\229\082\008\172\082\076\015\145\125\185\101\211\141\020\041\022\165\145\165\191\024\199\063\248\136\050\025\135\034\177\230\060\199\129\021\100\125\042\175\109\072\185\104\072\095\000\227\090\027\033\057\194\058\202\018\085\030\186\023\198\211\084\013\084\072\074\178\054\251\018\134\078\228\124\070\191\017\152\087\056\025\251\139\085\211\132\082\180\123\065\023\229\236\090\081\026\204\009\176\203\049\150\168\175\038\043\030\013\236\137\082\137\228\103\243\196\196\223\114\001\080\110\072\105\213\214\006\136\028\113\210\253\214\237\103\087\195\185\097\126\016\057\151\110\091\010\098\013\213\200\102\024\152\146\209\158\148\079\192\034\245\084\042\045\004\169\133\016\205\141\110\002\027\036\208\186\038\126\140\094\035\202\034\145\082\052\066\007\113\197\247\035\057\235\131\160\204\079\028\099\137\055\081\139\092\156\103\067\003\080\185\045\087\052\114\092\180\092\041\147\092\076\181\094\154\234\117\073\228\106\038\038\127\058\073\143\185\194\155\144\150\230\087\195\129\161\034\013\097\238\089\095\200\157\210\100\167\107\143\048\081\248\163\128\188\211\038\215\118\100\024\129\052\058\054\053\029\100\154\052\235\108\083\034\172\074\139\065\108\073\069\008\043\112\178\064\222\002\208\247\022\178\134\176\217\107\070\092\116\018\133\193\126\194\118\110\108\123\011\240\000\097\083\225\246\061\174\108\122\151\236\031\203\225\051\153\216\244\007\211\013\186\178\058\178\183\222\007\157\040\139\034\067\019\135\038\053\068\200\027\114\241\149\050\115\241\020\141\072\152\066\195\034\012\005\168\164\176\107\212\143\091\100\091\092\069\146\204\107\019\169\139\154\082\062\200\164\074\177\140\078\180\216\153\252\080\090\034\167\086\021\024\018\025\035\129\166\076\220\141\223\249\035\112\023\161\096\008\122\218\018\015\052\068\221\253\030\121\031\029\087\178\007\201\107\137\092\096\192\176\046\099\015\177\236\163\206\171\071\038\088\223\087\068\171\224\134\190\166\137\036\026\019\038\228\246\055\044\233\172\141\095\026\092\133\223\025\212\179\114\126\137\046\102\231\225\102\242\247\063\058\145\035\084\039\004\123\027\004\112\226\098\107\026\080\022\041\162\032\197\192\035\017\027\030\227\226\143\088\019\219\005\062\050\204\199\071\087\250\024\129\161\080\133\240\146\209\019\212\246\090\184\033\037\176\126\152\199\202\254\013\225\244\070\063\160\187\054\068\184\082\107\020\073\094\227\125\078\149\253\023\178\222\146\244\045\086\014\104\198\167\252\203\065\158\171\250\076\247\169\156\143\084\131\147\244\052\190\098\209\047\249\100\119\072\181\167\208\152\114\090\154\229\206\133\085\131\158\171\025\205\232\115\058\137\011\020\102\249\097\115\114\090\089\147\183\179\004\070\082\220\162\172\117\100\075\220\093\244\027\162\074\082\069\140\144\198\075\097\086\127\132\136\161\003\209\072\200\088\125\022\155\013\141\147\071\248\027\224\102\090\082\052\198\180\016\049\081\146\005\207\194\045\183\079\020\073\016\200\191\164\250\026\172\011\177\120\150\198\194\211\051\116\099\067\118\060\084\103\164\246\059\116\245\044\143\101\232\096\088\007\185\223\240\057\095\164\203\122\193\197\215\167\156\153\115\111\169\134\047\043\168\059\025\106\069\227\165\056\125\031\068\181\121\015\066\075\238\233\098\040\056\162\201\184\042\045\016\243\163\217\129\107\235\094\200\151\049\199\189\024\244\094\087\065\003\089\019\049\228\033\238\046\080\145\193\100\038\150\101\029\084\206\072\134\027\171\127\009\254\004\118\202\175\128\168\168\148\145\176\045\099\115\201\193\120\122\056\014\068\146\048\196\159\022\111\157\197\161\153\035\248\012\023\068\036\244\222\204\058\005\167\036\098\210\082\163\097\204\205\045\012\181\152\244\050\222\184\220\078\189\125\166\204\071\134\160\242\112\036\233\214\147\049\143\226\036\014\072\154\200\134\142\163\209\252\233\035\023\107\050\089\009\180\120\209\034\075\059\038\041\143\208\193\244\126\140\221\094\136\026\020\093\035\118\187\015\021\182\228\220\018\226\125\076\018\135\052\155\042\117\078\007\172\153\075\084\171\009\162\104\223\011\203\248\111\180\033\136\066\018\035\066\172\192\200\197\042\240\130\069\139\165\201\255\000\047\081\208\169\049\147\176\181\235\075\196\031\196\187\162\246\108\213\128\098\180\079\143\154\028\199\214\027\186\050\222\168\193\139\009\069\167\188\193\137\104\198\191\022\049\204\007\148\086\073\170\075\024\162\238\162\246\036\116\030\169\036\213\254\231\161\220\056\236\100\207\162\009\216\242\070\143\020\073\053\083\155\039\102\166\005\194\005\135\209\141\119\087\174\200\195\094\123\050\098\131\071\048\037\228\097\063\160\219\029\160\108\113\177\206\092\004\248\008\214\242\254\024\205\005\069\085\161\024\019\153\091\014\080\027\041\105\022\097\103\041\050\245\161\208\132\197\252\079\028\109\237\170\038\035\183\028\197\197\242\071\254\159\132\159\148\144\014\228\142\117\110\045\235\179\018\221\107\120\140\012\029\136\183\167\236\109\117\048\030\200\245\036\134\034\191\076\059\116\030\181\063\182\049\075\244\127\080\029\161\036\139\128\047\100\169\240\188\023\131\065\179\040\118\127\038\101\253\233\099\058\152\215\121\242\117\050\009\033\073\199\051\180\000\148\206\187\019\110\094\226\039\075\248\047\031\017\225\145\174\164\077\089\021\019\069\068\227\027\038\068\140\181\023\084\067\236\070\151\185\158\117\062\040\203\235\092\146\077\036\157\018\077\055\247\060\147\026\009\022\159\064\192\253\162\045\252\216\246\140\070\243\014\178\072\207\100\192\240\135\142\189\105\146\070\077\038\170\229\137\078\145\059\072\238\074\163\018\005\185\063\023\190\178\018\088\057\162\205\017\022\178\068\087\047\143\114\015\045\253\211\138\208\064\244\053\165\190\037\076\058\205\008\073\034\019\069\164\008\001\196\101\041\075\057\227\155\136\123\218\226\153\002\210\253\218\076\207\012\102\137\039\092\147\073\036\146\073\038\153\179\201\049\168\154\038\190\161\129\159\096\150\061\145\172\123\099\201\036\146\073\035\121\013\135\134\060\085\234\178\073\036\146\073\052\073\036\193\044\060\145\047\149\251\019\104\106\234\148\244\196\114\048\029\254\016\144\065\224\084\090\200\182\151\231\227\029\135\162\103\242\099\077\039\080\013\069\168\176\056\139\114\063\028\081\112\158\226\182\016\107\071\031\115\200\051\060\017\154\253\030\108\243\076\071\157\079\244\037\238\134\007\236\024\083\224\045\238\163\127\190\167\134\238\096\120\001\173\208\190\119\216\238\136\067\031\086\111\009\114\137\160\075\092\131\122\231\208\181\072\148\088\182\092\189\136\047\244\098\038\039\172\065\233\104\116\122\078\200\105\127\027\067\066\154\019\070\077\082\039\066\100\146\077\107\012\070\108\118\024\214\150\199\090\217\222\179\103\068\061\214\169\036\154\205\038\146\077\036\253\194\227\098\206\201\173\097\124\032\180\096\221\007\233\251\011\019\230\096\036\174\164\234\140\098\125\064\240\004\210\107\036\147\073\211\035\222\069\108\111\216\086\037\187\012\187\172\133\165\104\119\018\060\168\173\134\063\097\127\002\201\026\053\168\196\070\183\071\173\007\200\068\208\164\147\161\036\143\106\136\053\082\013\233\066\065\020\042\152\232\195\087\183\162\246\126\209\058\039\084\086\073\036\146\079\067\026\122\227\113\042\056\104\154\195\056\073\128\253\036\217\209\232\183\152\217\129\009\207\247\022\168\104\158\048\127\173\122\211\036\082\150\225\019\073\165\128\177\098\235\093\201\225\254\228\099\103\251\147\164\069\030\255\000\185\062\056\104\100\107\225\004\115\025\080\107\205\159\041\027\171\058\061\110\175\084\143\230\071\089\248\082\098\018\044\232\051\162\070\049\209\114\247\045\234\155\070\134\114\123\027\232\090\100\201\010\142\082\093\180\159\007\209\202\095\072\229\175\161\077\163\232\179\081\254\150\119\128\220\089\059\031\222\174\134\191\071\071\054\037\200\120\016\040\234\092\075\140\029\012\090\175\074\251\103\068\060\244\021\102\146\034\094\131\070\124\168\054\126\102\248\247\153\056\121\007\089\162\135\138\098\196\054\032\056\181\050\025\016\148\069\121\010\217\124\011\225\100\143\068\022\134\159\150\038\070\029\092\146\106\106\073\034\210\254\007\071\238\158\243\020\099\162\246\095\208\090\178\070\227\224\158\193\143\150\114\008\075\067\093\170\206\147\029\250\170\108\133\143\170\040\241\038\030\194\097\041\097\201\108\163\161\113\038\218\150\122\075\023\010\061\002\238\130\211\036\140\093\081\096\178\068\068\093\084\146\105\034\204\184\069\017\243\011\199\001\085\145\066\082\056\012\160\222\040\109\118\067\240\027\163\107\133\071\207\178\058\166\077\083\211\034\073\249\089\186\179\204\024\088\215\085\236\203\185\047\090\147\039\221\005\067\160\116\165\198\171\058\108\201\213\212\075\215\018\053\052\057\019\055\242\008\111\184\187\096\144\219\230\111\236\254\247\018\036\122\091\163\230\092\221\050\093\081\119\077\106\187\162\044\140\182\084\111\113\107\150\140\076\250\133\151\185\253\248\122\036\202\026\220\106\119\100\011\024\221\009\133\031\129\235\072\199\232\050\080\234\137\021\088\078\105\036\136\073\058\016\155\135\087\160\224\158\164\202\159\192\111\161\173\059\127\177\137\134\125\073\217\103\243\047\067\093\169\161\188\153\159\223\177\142\207\004\016\188\080\181\060\127\066\036\119\112\015\063\122\134\118\026\236\156\075\220\212\211\218\209\052\251\151\161\136\208\037\135\060\043\255\000\100\182\071\218\039\078\149\252\082\216\201\157\072\203\244\102\238\175\222\137\042\059\029\027\036\234\073\058\145\024\127\160\150\165\169\181\039\066\125\252\174\130\170\239\061\011\244\062\024\100\170\033\088\105\116\139\045\216\185\062\008\024\191\110\133\239\018\212\178\076\157\254\008\146\022\013\206\246\039\150\180\206\096\218\250\114\253\139\067\008\074\186\070\053\115\106\124\137\027\161\188\180\037\006\003\044\102\148\157\049\010\141\180\046\194\176\043\199\105\019\162\244\022\242\217\158\067\247\162\040\109\113\033\050\017\018\073\036\232\055\192\137\145\250\025\195\189\083\169\061\018\073\049\046\228\133\193\064\198\049\090\054\096\251\143\073\254\026\226\137\149\165\136\076\047\035\002\069\106\098\253\138\083\037\157\218\086\249\155\213\224\104\220\232\073\066\090\023\015\142\107\052\095\228\092\146\045\073\060\204\007\242\234\098\057\140\230\121\027\189\252\147\174\069\238\030\199\215\236\202\142\223\222\199\135\114\085\179\232\154\202\240\037\098\006\163\084\155\248\046\086\191\162\048\200\064\056\107\200\153\058\036\145\137\248\032\146\126\133\018\170\072\180\205\019\208\201\115\240\169\098\238\041\100\069\172\068\037\251\232\216\208\204\203\144\147\210\102\045\117\063\145\041\044\111\190\175\009\250\051\125\089\188\246\104\246\140\181\071\056\197\016\030\218\088\228\237\153\013\184\126\062\089\018\184\075\201\113\224\220\044\252\076\168\196\229\164\254\129\009\141\047\039\183\247\075\193\238\075\185\114\186\149\208\253\172\035\249\080\226\031\007\016\116\105\142\100\225\237\099\101\079\103\083\224\191\069\089\038\169\159\129\216\145\199\232\227\102\071\242\013\018\097\098\078\154\046\033\026\102\231\127\216\117\017\232\100\008\100\238\052\039\173\060\254\032\114\116\183\086\133\150\054\071\242\081\247\226\214\214\152\196\042\021\180\080\240\136\157\027\050\019\167\009\227\039\224\142\234\236\079\197\036\155\041\228\076\069\137\228\103\044\130\239\106\118\197\061\214\072\025\117\249\146\184\215\179\099\100\029\120\129\015\002\116\049\075\025\232\112\007\012\033\069\044\157\004\238\005\051\171\236\158\219\222\140\235\108\090\144\139\161\039\233\037\160\181\161\104\114\025\175\100\186\118\067\031\192\035\037\036\040\028\151\163\171\143\193\133\089\016\226\201\152\202\150\187\056\013\178\208\215\021\019\144\134\135\102\099\121\249\155\010\144\221\171\178\137\115\097\133\113\138\049\238\095\153\034\063\025\097\200\151\177\002\041\006\100\171\036\146\077\019\224\070\147\176\206\069\244\050\214\009\008\044\037\116\047\215\090\123\071\148\069\095\035\120\194\048\064\251\154\016\220\049\238\148\236\188\038\013\082\145\075\012\120\032\155\160\075\096\149\194\189\212\141\076\073\036\147\165\018\184\253\044\108\201\004\235\078\136\098\030\150\126\112\209\192\024\044\150\068\238\199\027\153\072\180\075\184\053\236\121\083\198\005\190\039\120\119\039\192\220\120\127\184\171\016\221\034\135\173\236\050\033\035\192\204\211\052\197\065\166\047\133\193\236\037\151\225\130\012\122\140\217\033\017\203\123\146\145\185\102\089\103\076\187\138\094\126\233\231\110\137\047\019\250\054\228\242\098\097\224\187\010\251\143\175\220\229\136\121\209\119\027\033\018\136\141\251\157\070\188\189\211\082\194\207\161\220\234\180\089\148\040\012\114\205\203\135\003\140\006\216\251\031\083\185\201\057\116\156\163\241\147\157\023\116\113\079\115\113\140\227\132\099\061\131\137\127\209\249\236\070\025\062\224\152\089\246\051\074\059\052\036\075\163\146\144\104\125\136\110\222\232\019\207\242\053\144\217\063\004\144\250\141\207\233\081\034\138\049\107\076\085\223\112\162\045\121\067\053\185\096\158\050\245\199\057\042\041\119\195\055\196\196\208\154\010\144\025\128\227\014\201\100\177\136\131\039\113\136\107\102\070\028\179\146\061\132\015\043\073\021\217\176\197\020\136\114\176\102\002\173\202\131\047\076\182\036\156\160\092\030\098\088\242\160\227\137\243\230\111\018\097\016\052\242\062\161\086\155\245\025\207\062\136\075\232\071\102\070\062\177\150\179\021\183\144\151\143\049\104\178\086\204\142\047\144\159\021\022\066\142\055\137\236\177\065\120\035\095\083\001\220\195\190\012\015\144\237\197\238\095\191\228\191\182\039\069\055\022\172\013\240\204\019\063\079\040\082\095\018\208\074\021\050\210\201\086\255\000\068\006\137\100\067\103\143\067\039\034\162\012\055\055\099\186\014\152\075\037\177\033\118\007\181\113\171\042\187\014\043\029\134\011\017\017\126\092\072\125\252\017\039\143\146\055\040\128\251\050\046\068\096\196\145\014\254\078\103\145\191\119\246\067\123\249\017\123\216\230\108\127\064\204\008\110\242\037\101\224\225\087\201\021\175\201\031\001\178\037\186\189\025\161\047\118\115\142\119\131\153\224\230\013\226\079\127\034\014\123\016\200\039\007\242\059\033\143\106\008\112\142\086\158\137\167\113\115\200\172\106\077\250\157\196\020\214\075\243\223\244\034\226\055\063\212\237\063\138\210\103\074\144\181\156\174\001\008\076\254\228\121\015\120\027\129\212\010\149\114\007\247\130\070\057\022\032\191\025\194\252\201\042\226\117\129\029\197\192\057\006\225\067\152\082\115\022\214\206\220\100\107\155\127\007\031\046\060\004\053\042\232\217\082\036\222\009\001\114\069\226\041\209\237\035\006\249\076\013\018\146\090\016\253\143\176\212\175\177\080\255\000\198\119\067\242\025\131\254\004\219\036\078\228\038\213\121\200\082\219\253\006\158\100\086\041\039\035\035\243\018\103\159\077\216\246\224\133\201\113\020\178\242\060\142\180\077\184\112\076\233\183\009\022\247\188\019\125\111\073\105\152\185\035\253\076\004\111\130\070\024\079\227\129\161\243\110\055\135\218\016\080\192\150\218\022\002\036\011\218\101\092\178\192\086\032\211\087\010\234\057\128\134\196\213\137\182\054\003\138\040\074\190\155\178\069\183\237\091\097\119\244\023\075\018\229\028\011\083\164\181\037\035\180\210\182\023\075\118\220\178\099\128\065\150\068\242\030\232\094\069\156\096\221\101\082\104\157\050\102\208\057\228\094\193\072\179\175\224\031\203\127\020\039\004\232\066\007\128\200\146\233\192\090\155\024\170\133\167\103\245\129\242\248\144\168\190\008\026\164\246\083\177\097\118\250\031\150\078\064\215\135\130\028\060\009\022\222\040\192\150\135\088\017\187\175\142\198\224\120\031\211\170\032\164\185\081\095\152\050\089\046\234\075\117\207\142\195\025\111\054\210\133\236\145\232\150\203\220\217\255\000\112\207\207\036\155\120\026\218\160\068\150\194\248\081\123\034\239\115\105\238\056\150\151\164\006\086\199\014\039\166\054\063\060\147\229\120\030\216\107\079\081\212\126\110\037\056\033\092\014\073\011\157\187\163\136\167\035\250\088\200\180\001\034\135\240\018\055\240\170\023\250\205\157\047\224\022\168\034\140\106\187\029\252\007\241\014\096\229\177\060\022\048\131\034\158\246\026\030\228\014\098\128\203\072\210\071\011\034\125\142\058\198\085\207\113\104\131\067\165\102\248\028\201\083\011\170\039\222\137\113\040\209\105\188\030\071\112\172\140\139\158\245\072\034\136\049\130\204\156\175\177\009\136\091\226\065\060\015\179\154\190\206\075\236\143\236\013\027\077\202\123\008\110\156\057\236\221\014\226\066\010\168\169\143\072\095\009\023\084\197\201\095\173\104\162\141\106\138\162\116\033\045\049\065\235\001\003\068\049\047\104\136\154\186\041\165\212\232\051\107\048\121\046\137\024\003\028\129\011\012\145\113\020\171\053\129\155\132\187\081\193\246\090\227\126\162\111\139\251\047\199\203\047\248\233\020\069\072\208\244\207\196\154\161\086\127\215\202\248\241\080\079\065\004\233\141\071\102\041\003\022\136\219\099\027\176\241\062\200\066\053\078\151\072\167\040\095\136\071\184\111\067\236\080\213\239\172\195\026\201\044\066\209\058\029\113\137\226\049\229\233\120\035\228\079\232\096\176\255\000\224\072\047\248\147\026\171\074\248\024\244\020\146\060\042\173\026\170\146\073\053\122\020\066\078\239\178\104\090\228\157\023\209\021\024\199\240\058\205\095\232\032\142\095\240\204\209\031\002\098\117\159\149\210\006\135\065\008\035\081\036\210\073\164\232\116\029\056\018\249\016\132\082\070\201\030\182\061\009\063\161\080\019\255\000\195\159\130\069\010\178\214\095\003\067\036\067\253\001\210\104\198\244\078\137\214\180\142\175\074\037\013\141\232\120\122\052\147\173\232\118\039\255\000\138\208\098\104\154\078\132\137\208\157\031\200\159\042\106\171\058\024\223\050\169\052\110\143\082\021\025\004\107\079\192\235\130\079\249\089\070\135\161\009\147\161\036\214\107\036\147\092\147\173\019\162\107\036\142\172\061\082\078\165\168\054\058\078\148\033\209\072\116\164\127\011\164\027\003\255\000\145\024\132\164\107\078\137\036\085\073\036\214\073\036\076\127\018\100\146\073\063\011\105\159\129\011\064\145\141\235\129\045\009\198\153\191\252\017\146\172\136\073\036\147\073\039\084\232\143\213\021\080\133\162\117\070\129\026\093\036\159\152\110\127\230\232\003\209\053\090\146\078\144\146\073\038\179\240\079\234\008\090\221\032\130\008\034\143\075\100\252\171\153\178\135\255\000\058\018\079\129\105\146\073\172\146\077\019\170\126\105\249\075\227\036\036\065\031\004\143\230\218\255\000\163\001\049\003\249\039\074\073\036\159\211\063\137\105\084\117\129\010\178\078\147\249\164\148\075\036\199\253\053\010\025\209\053\122\228\159\134\126\025\248\094\184\170\248\035\074\073\210\216\217\031\042\081\039\253\104\245\072\254\025\162\106\191\093\031\036\146\073\052\073\035\099\029\103\226\066\014\127\246\026\022\029\137\036\146\117\078\148\254\060\010\151\159\130\126\036\181\079\193\036\147\068\146\079\203\004\024\036\255\000\181\009\038\071\092\147\250\104\038\095\066\071\252\113\036\252\024\054\087\253\232\178\039\038\041\063\060\210\073\213\096\195\115\240\199\203\058\239\230\038\138\137\012\127\248\005\017\043\253\003\209\036\147\162\007\250\041\209\036\147\073\249\036\122\240\112\135\063\252\028\089\025\063\213\004\147\250\009\210\019\089\253\036\239\252\050\018\108\153\253\066\164\213\053\146\107\063\165\036\157\072\032\148\142\024\223\254\037\232\067\051\250\041\215\036\255\000\193\130\008\026\035\128\073\255\000\141\080\016\255\000\064\159\248\242\073\004\014\194\214\009\063\242\041\133\051\063\052\255\000\200\148\142\024\199\255\000\149\128\075\020\050\008\253\067\209\063\163\087\025\110\164\229\255\000\153\078\169\163\254\082\107\109\196\142\008\223\254\118\068\235\114\017\052\137\210\095\169\143\129\216\104\037\024\255\000\244\140\071\020\072\203\058\035\245\145\070\129\170\192\216\110\255\000\245\082\038\066\106\018\144\099\085\098\169\017\241\069\017\072\013\105\108\095\073\255\000\216\216\072\078\074\180\004\017\165\014\036\120\145\227\162\168\086\108\054\039\255\000\220\143\255\218\000\012\003\001\002\002\016\003\017\000\000\016\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\202\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\056\242\143\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\143\060\243\207\060\243\207\060\243\207\060\243\207\060\243\205\060\243\207\060\243\205\060\243\207\060\211\207\060\243\207\060\243\207\060\003\207\060\243\207\060\241\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\028\243\207\060\243\207\060\243\207\008\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\128\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\195\056\243\199\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\240\015\060\241\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\208\205\050\250\249\077\164\028\073\165\174\186\243\143\060\243\207\060\243\199\060\243\207\060\115\207\060\243\207\060\243\207\056\003\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\011\161\215\046\183\020\193\178\072\187\239\210\034\217\071\236\190\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\206\000\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\010\129\059\166\187\126\180\212\021\026\017\135\027\121\012\054\118\031\244\034\195\143\060\243\207\060\243\207\060\243\207\060\243\207\060\243\192\060\243\207\060\243\207\060\243\199\060\243\207\060\243\195\059\093\236\230\014\172\255\000\039\082\219\085\167\175\090\255\000\209\067\148\144\193\163\159\039\013\060\243\207\060\241\207\060\243\207\060\243\207\060\240\015\060\243\207\060\243\207\060\243\207\060\243\207\056\255\000\047\213\138\053\164\150\114\141\245\201\167\092\122\243\031\121\219\068\226\044\229\046\098\188\219\060\243\207\060\115\207\060\243\207\060\243\207\060\003\207\060\243\199\060\243\207\060\211\207\060\115\125\154\005\076\088\011\246\208\010\121\239\216\177\211\142\183\171\253\150\223\102\001\054\185\106\158\123\077\188\243\207\060\243\207\060\243\207\060\243\207\000\243\207\060\243\207\060\243\207\060\243\205\054\125\155\123\153\215\081\063\015\131\094\058\131\111\000\143\027\184\227\093\098\151\255\000\025\247\253\236\143\240\037\124\243\207\060\243\207\060\243\207\060\243\192\060\243\207\060\243\207\060\243\207\060\240\077\162\246\141\118\084\216\056\031\161\138\059\171\011\143\195\215\015\185\147\051\201\243\008\012\134\072\045\099\250\105\108\243\207\060\243\207\060\243\207\060\242\143\060\243\207\060\243\207\060\243\193\042\209\251\044\128\019\005\198\186\133\217\046\091\050\002\246\096\010\135\219\168\234\205\021\111\082\228\007\149\182\150\235\092\119\227\207\060\243\207\060\179\207\060\131\207\060\243\207\060\243\207\008\171\169\076\020\162\226\069\092\034\090\093\094\047\250\187\092\137\032\040\111\163\137\110\228\126\160\167\179\167\232\182\084\142\216\107\082\243\207\060\243\207\060\243\207\000\243\207\060\243\207\060\243\204\058\086\137\139\053\079\006\005\062\068\052\171\104\214\117\102\131\206\215\130\228\205\157\084\207\169\058\241\122\037\007\021\210\148\138\130\243\243\207\060\243\207\060\243\192\060\243\207\060\243\207\060\147\128\221\086\203\209\213\081\163\012\160\080\070\217\045\066\174\072\195\199\013\088\228\196\157\135\229\001\050\211\158\164\077\101\213\240\233\248\208\147\207\060\243\207\060\240\015\060\243\207\060\243\207\045\142\050\247\032\185\227\054\091\145\249\121\195\156\215\170\031\251\220\164\064\099\108\120\125\171\099\153\046\054\000\184\037\056\003\066\181\219\253\152\243\207\060\243\207\060\003\207\060\243\207\060\227\156\134\184\173\229\002\238\113\191\045\070\024\028\095\197\125\207\185\027\088\250\218\238\186\084\250\170\060\199\109\175\066\001\145\174\134\059\029\199\139\060\115\207\060\243\207\000\243\207\060\243\195\011\207\248\202\152\193\187\120\227\135\176\151\023\040\221\239\207\228\058\136\198\085\215\053\142\211\061\150\212\020\231\012\084\090\237\090\069\176\032\229\038\098\052\243\207\060\243\192\060\243\207\060\242\143\254\012\250\243\004\169\214\205\239\164\253\081\130\133\244\246\113\140\057\246\030\061\018\064\060\083\100\137\231\193\255\000\190\144\152\219\119\237\146\146\240\136\123\188\243\207\060\240\015\060\243\207\056\133\179\251\179\145\085\122\060\058\122\187\075\231\158\225\234\059\163\202\012\000\086\043\195\036\242\167\032\052\124\210\125\234\002\149\049\024\183\167\145\056\024\030\183\060\243\207\060\003\207\060\243\200\093\122\118\231\026\033\163\160\129\187\206\074\199\052\239\097\007\196\163\021\005\255\000\003\108\238\186\231\058\186\016\172\126\048\026\141\009\216\188\067\250\098\109\091\141\045\060\243\207\000\243\207\060\034\154\230\028\227\054\057\157\059\250\099\154\211\024\240\073\037\081\156\171\131\208\083\111\064\160\210\063\114\186\254\018\086\077\032\042\012\226\164\165\011\105\235\012\155\025\207\060\243\194\060\243\207\051\216\237\147\171\096\178\139\029\012\125\186\244\108\044\108\047\216\211\152\114\143\014\119\215\253\044\218\096\110\255\000\119\129\218\200\068\235\033\055\200\155\158\016\193\075\164\215\079\060\240\143\060\243\136\108\164\170\148\227\239\148\045\192\204\156\185\009\138\127\167\000\252\200\250\114\128\115\015\048\159\205\032\074\048\217\222\121\017\199\037\175\030\008\175\193\218\045\108\019\056\081\207\060\163\207\060\160\093\093\121\100\026\199\052\255\000\013\109\108\159\022\074\252\216\135\038\154\131\155\223\185\117\079\204\089\143\059\251\235\129\026\214\179\163\126\161\103\161\113\086\037\119\144\080\059\115\207\008\243\207\052\111\163\238\095\097\148\213\203\000\095\007\162\103\217\246\237\044\144\005\081\113\190\164\183\025\059\150\243\108\215\142\190\245\036\214\021\192\089\084\022\167\028\031\172\222\069\131\252\243\202\060\243\129\111\245\174\240\010\113\162\204\119\238\246\222\025\053\087\234\185\177\208\131\064\130\123\180\210\199\172\131\078\044\208\079\034\048\147\122\167\095\147\187\063\135\219\091\210\143\169\069\252\242\015\040\248\175\167\027\089\030\229\224\216\136\170\186\102\105\022\196\182\179\121\165\195\190\234\199\078\124\141\210\182\239\012\230\130\228\008\008\202\019\201\222\191\085\082\118\206\115\072\092\034\031\060\035\207\033\080\039\142\172\085\200\187\139\112\148\197\084\194\135\252\215\167\075\081\150\027\050\243\178\172\000\174\185\232\188\152\109\106\203\004\134\210\255\000\202\171\247\189\093\008\079\221\201\044\139\079\040\243\131\210\240\015\003\054\210\166\130\042\076\140\027\228\128\069\168\123\221\175\099\190\050\173\057\162\029\046\023\143\226\240\003\071\024\201\235\045\064\123\246\054\138\050\167\243\168\067\060\032\027\202\060\194\120\136\240\216\251\042\127\004\191\004\221\221\126\232\102\093\216\068\213\130\129\007\151\079\187\198\046\127\097\195\193\157\210\076\053\112\192\116\239\153\229\131\000\129\123\002\066\208\149\091\162\015\054\251\151\058\166\213\080\204\062\248\123\198\001\023\120\071\091\247\125\087\057\126\149\065\034\159\131\141\103\016\218\210\151\036\141\115\247\127\049\086\093\019\187\098\038\118\255\000\024\140\091\212\124\163\195\206\051\181\048\155\113\006\000\167\056\186\189\209\042\159\127\174\009\194\247\064\064\010\104\021\146\036\126\102\038\035\168\074\083\151\217\142\206\155\129\214\053\235\223\251\128\114\038\048\035\078\032\176\114\199\059\088\019\085\197\162\171\251\134\076\030\041\108\069\225\062\098\226\213\080\144\024\107\006\071\235\054\076\062\164\087\133\213\215\106\233\203\208\223\056\158\166\158\236\144\215\241\132\030\128\060\185\044\054\189\057\169\115\164\201\186\053\210\167\127\239\215\238\140\201\249\200\005\171\154\219\098\238\010\164\068\068\229\245\026\057\016\157\233\006\006\149\194\137\234\141\185\184\227\061\183\163\114\015\041\163\162\058\065\247\072\188\124\185\193\255\000\213\077\056\186\003\123\055\236\060\068\234\036\003\102\070\023\255\000\031\063\221\031\055\126\131\243\093\085\193\162\202\071\212\032\006\057\051\038\175\193\157\003\196\086\173\212\034\191\203\081\185\189\249\062\075\086\240\170\233\104\219\058\061\218\204\189\213\239\052\217\004\170\141\174\138\122\105\055\083\074\119\236\048\058\166\038\051\127\223\007\202\043\045\162\000\241\019\046\232\192\156\212\155\041\147\069\171\215\140\131\132\125\007\165\001\210\085\016\057\082\229\081\220\147\026\175\110\099\177\167\212\170\152\067\076\067\245\182\009\068\213\203\101\214\053\070\248\058\124\083\019\067\125\065\196\197\135\062\133\128\121\107\081\120\018\026\027\133\086\072\250\094\060\100\041\076\213\077\241\023\200\092\212\076\099\023\038\045\142\018\112\228\242\115\196\090\039\183\134\195\214\143\025\103\158\194\232\183\224\210\207\085\074\217\195\177\106\081\198\203\201\036\147\080\016\154\187\059\133\218\183\121\116\199\062\085\069\064\116\151\023\152\101\065\040\049\249\219\226\001\170\096\100\253\163\202\237\239\101\171\015\203\177\066\145\181\227\096\231\170\190\022\089\222\110\194\104\155\093\067\031\093\215\242\235\162\247\174\003\067\083\165\138\233\174\199\145\073\073\099\188\114\008\098\012\019\253\000\225\126\071\045\249\001\245\244\029\088\146\225\053\214\051\214\095\172\150\189\142\033\180\035\010\046\136\254\019\057\096\162\055\152\160\070\168\047\201\015\056\147\086\183\061\146\032\124\041\030\073\203\130\060\060\051\248\172\247\123\128\033\010\018\005\196\132\159\179\099\120\076\050\147\096\112\043\028\067\079\063\239\218\048\053\054\105\004\141\125\235\156\096\012\006\101\116\156\121\177\121\124\215\215\017\240\143\061\140\204\235\026\068\227\236\129\023\132\031\192\150\090\128\005\023\024\065\221\029\184\060\102\142\165\247\246\026\229\147\029\101\026\199\084\179\146\073\227\185\085\208\137\223\181\203\086\192\227\232\131\207\001\050\096\126\063\127\233\227\136\093\166\125\021\226\111\117\091\062\137\060\198\034\183\247\144\237\144\010\238\103\000\190\085\147\221\196\073\166\080\212\070\241\005\229\118\247\198\004\246\203\239\008\242\199\149\037\213\170\055\074\058\194\058\105\041\046\086\172\059\114\193\162\242\083\199\070\026\233\091\134\150\243\147\022\015\197\076\254\042\185\237\188\129\056\198\077\121\162\066\024\202\143\162\082\194\060\241\060\145\218\129\133\146\197\143\105\003\185\189\155\038\057\192\084\212\195\158\223\147\000\244\114\161\106\176\193\179\067\002\195\009\065\020\114\014\097\002\229\114\078\018\007\063\190\203\232\124\240\143\056\252\028\074\087\068\127\022\011\190\064\250\134\053\190\002\043\140\048\227\131\031\035\002\201\142\096\118\192\020\090\003\152\052\176\206\051\010\225\072\243\095\114\168\145\074\060\018\199\244\159\060\003\207\060\009\206\013\147\037\067\192\073\207\044\085\153\117\012\036\227\239\172\051\203\097\003\196\063\224\084\196\064\243\164\246\015\112\116\014\039\166\148\105\164\086\114\058\226\178\248\056\205\091\207\040\243\206\009\027\188\053\206\008\106\171\188\007\119\232\016\224\001\075\101\093\204\227\199\157\115\073\068\048\136\016\012\003\240\172\093\233\228\017\211\038\172\147\136\107\138\024\175\153\122\058\252\243\202\060\243\207\167\185\072\037\056\139\026\184\139\064\090\066\003\002\062\121\253\159\055\087\014\161\123\035\251\115\143\124\242\236\002\001\136\053\196\152\255\000\200\224\087\043\023\030\059\064\073\219\111\060\242\143\060\243\203\086\031\227\205\184\179\246\162\122\058\003\177\223\046\154\095\251\058\247\005\198\223\172\040\192\220\106\200\062\180\251\104\105\235\209\191\108\234\070\246\165\255\000\041\214\017\046\241\207\060\131\207\060\242\241\134\252\186\255\000\013\017\229\062\059\188\077\033\218\000\150\214\157\102\224\141\190\205\054\022\207\158\131\002\028\243\247\207\248\161\119\201\227\081\242\002\035\253\124\211\107\188\243\207\008\243\207\060\176\200\047\014\161\235\141\126\210\094\237\224\043\233\231\177\100\203\190\014\181\248\040\190\168\251\104\217\204\055\042\228\154\056\133\197\107\003\167\234\050\168\175\187\085\091\079\060\243\192\060\243\207\060\255\000\128\219\088\125\222\030\147\211\029\236\250\172\115\050\106\189\117\139\002\138\230\121\199\127\202\215\124\151\208\059\193\251\076\177\073\050\147\030\206\049\144\220\051\238\243\207\060\176\015\060\243\207\060\173\250\064\216\232\034\060\109\211\105\036\222\014\126\097\230\144\226\190\049\171\166\040\215\191\243\179\252\178\194\219\111\191\140\193\079\085\229\159\140\232\170\255\000\149\060\243\207\060\003\207\060\243\207\044\146\014\047\127\108\235\120\116\239\161\228\187\108\125\045\199\054\158\187\051\131\143\242\110\083\100\132\189\062\203\239\188\190\202\063\248\238\053\162\015\042\163\051\207\060\243\207\000\243\207\060\243\207\059\013\116\243\045\225\178\217\189\242\012\172\231\150\042\227\169\162\089\096\191\074\049\162\150\170\098\065\114\049\219\169\032\240\121\099\199\188\104\136\025\163\092\243\207\060\243\192\060\243\207\060\243\207\035\118\063\195\067\255\000\053\086\144\079\121\241\005\029\254\178\168\223\025\242\228\004\249\041\154\219\054\226\008\144\159\173\033\113\252\048\255\000\061\239\125\165\143\060\243\207\060\240\143\060\243\207\060\243\203\039\019\055\186\074\225\117\006\095\189\128\255\000\190\167\105\005\030\223\113\186\174\138\076\190\044\221\246\125\178\050\085\180\056\231\222\248\067\027\252\076\243\207\060\243\207\060\163\207\060\243\207\060\243\203\028\137\171\083\091\049\228\020\151\140\095\191\169\012\187\148\019\028\118\091\157\106\222\193\141\019\142\060\038\214\052\088\123\029\237\216\194\230\079\060\243\207\060\242\207\040\243\207\060\243\207\060\243\203\028\204\016\107\090\219\213\102\127\210\050\035\178\156\086\073\200\011\198\158\255\000\203\106\136\041\246\023\024\022\054\167\137\242\231\244\148\004\243\207\060\243\207\060\243\200\060\243\207\060\243\207\060\243\192\063\018\149\073\252\082\239\140\247\207\012\179\027\172\240\207\196\000\191\175\099\156\212\123\226\138\089\186\124\034\219\102\048\205\051\207\060\243\207\060\243\199\060\240\015\060\243\207\060\243\207\060\243\203\183\066\205\127\070\072\163\061\056\234\000\162\025\206\144\097\161\235\190\157\028\064\146\223\073\071\157\110\252\081\185\045\059\124\243\207\060\243\207\060\243\207\060\003\207\060\243\207\060\243\207\060\243\207\058\046\002\188\133\191\087\170\182\015\138\184\219\227\176\229\193\054\212\250\198\008\199\215\100\040\203\095\008\165\226\231\207\060\243\207\060\243\207\060\243\207\000\243\207\060\243\207\060\243\207\060\243\207\014\175\222\085\063\220\090\106\215\231\059\191\235\027\252\075\047\114\049\147\092\028\230\150\120\117\119\205\221\199\252\211\207\060\243\207\060\243\207\060\243\192\060\243\207\060\243\207\060\243\207\060\243\207\044\225\147\135\228\191\073\075\055\207\254\049\243\140\111\235\015\250\065\100\211\116\227\147\073\221\255\000\013\243\207\060\243\207\060\243\207\060\243\207\060\240\015\060\243\207\060\243\207\060\243\207\060\243\207\044\178\174\089\167\095\064\090\035\241\046\033\195\012\248\073\158\182\182\053\197\015\243\153\194\118\079\060\243\207\060\243\207\060\243\207\060\243\207\060\003\207\060\243\207\052\243\207\060\243\207\060\243\205\060\241\076\145\140\032\247\004\191\126\012\063\227\003\129\018\155\224\055\097\236\042\249\191\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\000\243\207\060\243\207\060\243\207\060\243\207\052\243\207\060\243\207\026\084\220\049\015\095\127\126\053\092\254\220\231\173\106\223\190\001\236\243\207\060\243\207\060\243\207\060\243\205\060\243\207\060\243\200\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\199\060\240\232\181\039\083\215\035\184\249\255\000\208\255\000\099\159\221\243\207\060\179\207\060\243\207\060\243\207\060\243\207\060\243\207\060\114\143\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\195\044\112\207\187\126\017\239\249\227\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\056\131\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\207\060\243\202\000\255\218\000\008\001\003\018\001\063\016\255\000\209\134\226\032\000\016\116\000\172\000\016\000\131\176\000\016\001\192\000\013\000\002\000\000\066\064\016\152\001\000\016\000\004\007\251\056\000\000\001\032\032\000\000\000\000\177\128\148\000\002\001\078\128\195\130\021\128\012\119\128\044\019\142\032\000\099\243\226\037\000\045\120\034\110\044\032\230\151\217\014\095\022\140\016\000\012\126\198\048\042\000\246\000\048\144\178\000\025\127\099\071\104\013\192\130\082\080\225\064\128\166\064\000\097\000\001\048\224\064\203\032\005\120\113\005\120\000\000\023\251\024\000\012\108\040\160\020\009\032\001\000\002\016\152\210\203\176\191\217\217\048\016\000\069\022\002\016\001\172\134\128\128\193\040\024\186\200\168\000\000\000\040\066\065\105\082\160\128\036\231\033\028\122\006\000\000\000\000\003\152\099\112\032\010\193\146\204\037\250\060\054\150\071\096\067\078\020\021\004\180\100\185\065\120\112\068\005\251\193\013\252\130\000\004\008\024\087\081\016\065\080\228\002\014\002\255\000\106\005\007\128\140\003\106\224\045\233\168\000\128\160\064\154\160\056\017\000\213\072\129\073\001\144\036\102\036\065\107\130\197\246\130\228\147\128\230\001\034\018\113\122\164\018\074\008\073\207\164\149\002\119\143\249\056\008\016\155\137\048\148\129\033\040\024\189\228\208\004\044\193\120\025\094\039\069\146\029\097\066\073\016\001\037\186\074\032\114\248\042\096\036\034\005\000\020\084\017\152\094\062\058\132\020\192\066\073\006\052\132\008\071\011\155\160\002\088\198\089\032\136\216\220\060\100\051\016\016\086\068\022\229\016\037\052\047\005\004\144\072\200\005\140\013\233\004\066\159\004\034\195\148\161\009\111\128\032\000\161\104\012\024\000\102\128\140\132\099\058\145\129\115\200\039\175\163\064\136\050\000\016\192\008\152\043\020\006\005\016\028\072\002\076\161\128\128\066\090\192\202\137\241\003\040\133\064\176\085\144\072\234\224\140\007\020\215\216\181\047\129\008\049\000\028\077\022\048\004\169\032\032\020\021\202\004\020\160\136\144\080\041\057\006\073\101\002\008\128\000\125\012\020\024\011\024\141\133\147\037\051\048\196\036\160\079\232\192\064\010\129\042\022\098\132\040\132\120\233\014\095\219\080\032\004\084\135\136\202\050\005\226\068\016\002\000\096\014\144\114\020\140\041\034\009\160\003\080\007\000\120\006\095\071\168\129\000\008\006\104\000\008\004\013\206\160\140\255\000\180\141\065\099\016\000\033\064\000\089\085\072\000\066\070\162\012\064\016\004\025\097\148\084\016\161\022\116\065\138\253\064\128\000\168\097\001\074\007\000\009\136\134\034\000\128\159\039\007\188\008\056\021\012\136\020\167\194\179\016\002\064\004\206\168\142\127\104\024\028\128\204\216\013\162\000\001\042\161\033\065\027\064\122\173\130\050\137\076\014\116\082\253\024\000\000\016\144\048\064\053\128\006\170\001\055\246\229\034\000\128\000\032\000\001\016\000\000\017\002\024\000\048\048\084\041\192\136\001\039\254\060\148\000\003\136\068\000\024\006\136\001\006\060\036\000\034\072\096\030\064\144\066\193\007\128\011\008\104\000\032\010\000\000\127\228\167\004\112\008\004\002\005\027\143\159\255\196\000\034\016\000\002\002\003\001\001\001\001\001\001\001\001\000\000\000\000\000\001\016\017\032\033\049\048\065\064\081\080\096\097\128\255\218\000\008\001\002\016\001\063\016\255\000\236\080\025\032\077\133\048\084\214\037\033\104\216\076\255\000\088\000\223\162\125\080\000\161\072\032\038\009\159\231\147\003\066\122\128\166\097\016\020\043\040\070\096\128\144\255\000\242\200\096\111\066\124\000\070\033\000\044\088\100\006\193\071\128\004\195\051\252\097\193\225\033\025\132\143\227\033\098\216\128\100\131\007\244\017\224\000\064\254\099\063\195\053\138\089\128\021\032\055\000\041\224\000\043\000\125\002\060\000\020\006\255\000\128\044\015\217\138\020\205\162\061\192\082\045\035\126\064\000\054\088\224\241\001\008\127\049\159\180\193\246\098\060\064\249\065\065\240\120\128\022\001\146\189\000\000\037\011\031\174\192\159\032\001\160\217\033\179\192\001\179\200\000\022\176\161\228\002\003\255\000\072\230\244\128\000\214\144\235\208\000\054\024\035\032\001\141\069\015\032\008\067\187\243\216\019\248\192\016\009\204\023\010\149\152\000\177\233\000\001\008\053\181\249\088\040\126\064\000\000\002\130\052\046\100\001\065\066\152\000\224\144\177\232\000\000\132\252\130\193\067\213\000\093\073\198\032\096\176\209\136\040\064\182\032\002\192\132\139\103\176\004\001\107\240\176\125\031\144\004\000\000\209\071\070\129\141\136\014\026\006\089\179\016\044\241\000\044\194\016\099\126\029\067\212\000\054\103\001\005\136\045\026\202\006\194\148\035\048\006\217\132\191\020\052\156\030\214\004\039\136\045\144\014\124\000\025\224\000\108\125\004\020\144\209\128\123\246\000\080\046\095\230\016\056\017\069\140\007\088\193\178\006\072\166\003\200\076\144\209\020\252\032\000\168\092\255\000\014\000\000\000\128\167\152\000\048\216\114\117\006\137\028\123\128\010\015\178\243\022\010\158\032\022\240\032\117\228\000\118\141\203\066\195\032\208\108\053\102\000\006\239\033\081\145\066\245\066\046\120\000\216\254\167\244\021\199\225\000\000\017\236\064\000\018\060\224\004\111\048\062\222\051\097\083\196\013\219\016\021\198\097\244\177\230\000\002\216\007\241\035\168\041\003\068\054\121\000\077\129\159\006\069\154\189\064\101\130\135\136\044\012\104\208\169\092\001\215\152\002\000\164\148\031\195\241\128\000\001\051\004\244\127\012\128\055\095\176\002\130\254\076\089\078\199\177\098\167\254\013\253\200\001\099\032\008\232\071\005\139\253\128\167\227\000\000\001\031\048\217\018\161\099\017\080\177\149\194\134\072\126\136\167\080\209\240\255\000\192\218\163\250\136\123\097\182\088\128\000\201\037\136\041\146\138\140\027\235\001\087\210\000\000\024\184\217\120\129\178\200\016\149\148\047\024\005\056\017\244\189\044\051\163\236\106\016\036\008\162\130\056\144\076\052\138\108\134\013\181\009\165\029\007\035\074\144\132\088\143\234\052\195\180\095\129\204\005\016\072\078\000\208\064\073\198\160\149\236\068\006\131\218\158\064\106\191\020\011\201\214\009\040\027\148\107\057\198\032\232\144\093\188\169\088\059\105\160\082\088\161\080\187\098\161\208\016\034\142\079\128\102\193\002\130\004\150\183\048\094\137\172\093\012\152\021\005\216\197\209\245\013\013\005\107\144\039\244\106\193\089\064\216\106\200\062\156\097\106\069\114\129\208\172\019\208\104\052\098\032\040\027\067\096\086\169\013\137\097\168\086\000\055\002\152\129\241\148\055\002\229\224\098\177\213\076\161\129\166\053\208\177\028\067\094\009\160\080\198\044\026\050\014\242\070\012\079\024\104\013\028\029\194\128\136\104\184\142\138\016\080\028\214\240\006\026\132\130\193\189\026\077\150\104\044\086\026\160\124\032\080\100\104\013\003\057\139\080\089\206\070\236\055\102\007\080\060\224\109\011\082\240\241\033\154\034\031\193\080\208\153\203\021\144\003\131\099\013\005\193\002\010\072\100\161\046\083\134\226\029\198\015\132\173\132\165\156\201\126\035\150\035\175\002\026\004\181\096\021\042\039\080\176\104\081\160\216\028\068\169\034\133\137\003\153\135\037\154\145\120\153\108\209\112\030\013\076\203\069\172\136\250\008\031\046\011\115\144\106\243\032\117\131\031\164\169\130\092\056\155\199\001\213\168\157\128\006\129\210\006\006\227\006\169\030\129\087\032\085\028\068\240\125\089\195\056\136\232\163\166\040\094\024\194\018\142\036\106\204\007\168\232\209\226\003\196\149\138\114\124\114\024\057\226\002\069\241\130\096\218\010\021\144\212\119\082\014\038\088\027\050\236\140\006\128\150\134\209\003\194\113\051\114\138\018\226\066\115\226\006\117\225\065\226\000\116\034\134\096\088\027\025\009\045\072\088\021\123\065\082\231\008\006\007\099\128\049\098\139\218\033\162\035\068\004\008\192\039\171\007\021\048\026\233\227\192\000\122\242\136\125\192\061\080\099\104\102\135\203\051\021\169\168\012\136\104\054\147\065\013\068\216\020\046\010\075\062\136\173\078\074\020\130\237\022\046\057\048\110\097\219\232\046\086\034\079\166\137\087\032\157\120\000\123\066\164\207\024\057\032\046\089\200\199\000\199\068\138\033\170\005\009\003\146\058\032\049\172\074\111\000\002\248\160\092\168\092\017\060\000\045\074\056\057\077\016\020\033\170\065\198\096\122\142\141\024\130\062\097\053\078\099\065\099\114\103\007\074\022\104\217\209\196\025\192\192\192\104\108\131\096\054\134\117\034\193\162\227\082\086\250\048\050\064\208\209\244\129\122\025\128\169\147\221\228\004\111\071\062\128\020\169\062\056\134\072\168\066\066\220\134\195\036\164\133\190\224\027\022\025\128\027\235\048\055\144\080\161\152\062\166\231\048\006\049\179\016\022\055\008\134\052\065\208\206\180\173\064\129\244\096\009\168\015\162\017\048\110\000\140\064\147\081\041\013\131\134\072\188\125\104\004\011\076\180\084\080\088\177\097\252\066\226\101\012\129\062\188\096\006\093\020\172\064\102\000\026\010\084\091\169\006\196\127\078\052\042\090\007\104\063\166\082\014\232\128\058\153\058\146\133\155\145\244\113\252\039\003\152\169\136\063\064\246\163\208\083\000\054\157\002\002\035\162\129\099\048\047\083\184\209\136\015\161\074\134\177\160\123\008\145\075\131\136\106\163\140\071\018\102\165\129\187\031\024\139\208\249\009\027\114\130\173\141\131\224\062\236\128\106\194\230\096\077\176\052\012\097\144\209\046\035\195\044\212\117\021\120\204\233\161\229\138\012\111\013\242\037\046\112\010\244\000\004\217\226\032\080\166\179\249\150\070\027\052\102\215\025\197\031\028\105\217\049\123\120\075\062\135\230\035\162\160\136\026\014\066\166\065\160\185\134\207\009\007\014\169\197\003\067\224\187\064\168\220\163\070\007\240\198\239\051\097\143\179\067\040\103\072\220\196\117\046\006\065\144\086\109\248\001\099\019\225\224\035\116\144\161\128\029\156\204\170\004\137\233\211\025\117\021\028\026\141\158\102\072\132\074\196\107\020\125\157\016\088\058\168\040\102\005\067\103\172\128\105\017\208\180\208\195\104\113\029\126\013\097\009\192\245\045\235\000\156\102\028\001\105\099\179\016\254\049\027\118\134\100\001\112\249\102\021\188\000\049\180\048\216\156\035\175\100\128\081\114\071\010\108\146\245\014\056\058\049\010\072\217\162\005\003\065\096\254\163\027\196\001\216\092\205\242\246\000\029\102\102\169\088\231\004\213\010\134\144\254\133\014\139\133\120\192\016\218\048\001\220\104\059\048\043\158\048\010\133\143\008\161\224\002\058\130\224\079\071\121\066\154\021\013\024\188\254\036\208\165\097\229\035\065\014\014\220\074\028\035\002\168\209\028\010\232\087\018\022\226\055\097\248\064\004\219\042\087\128\000\007\124\128\219\217\066\058\058\200\023\163\140\007\097\160\184\113\045\152\015\129\083\017\037\194\009\250\057\127\007\240\132\007\245\016\127\050\222\080\000\088\241\177\152\000\115\001\180\226\137\108\040\028\096\158\142\078\035\131\232\108\043\028\014\055\136\092\179\071\138\097\192\135\236\228\161\035\248\157\076\216\092\204\052\122\065\082\178\010\016\043\074\098\180\097\125\032\044\022\024\059\013\133\076\066\003\097\240\135\112\040\032\111\003\122\016\133\131\069\014\217\031\208\161\066\130\036\116\072\041\170\134\169\200\106\223\136\001\107\208\082\200\011\013\097\162\134\007\193\200\032\162\104\250\050\161\072\016\062\018\125\002\018\010\020\128\064\099\065\176\166\007\169\065\016\022\031\200\108\142\230\007\136\248\122\159\044\193\076\192\022\022\024\025\244\119\017\070\226\052\051\016\044\047\054\195\255\000\080\045\101\008\091\023\090\075\016\088\021\090\241\130\144\177\235\096\173\236\000\006\160\112\182\216\129\073\050\216\129\088\005\002\069\191\064\032\000\000\129\064\185\239\115\098\147\196\005\120\000\001\084\022\152\193\066\032\117\090\027\209\064\099\008\192\031\020\011\012\252\000\003\064\228\071\095\128\230\052\121\128\088\128\223\190\040\033\049\000\191\049\000\021\134\126\035\228\253\000\003\255\000\028\000\000\020\006\126\064\094\140\071\250\000\000\000\000\032\148\025\249\086\044\226\255\000\060\000\000\000\000\004\134\127\194\002\000\000\000\000\014\005\134\055\233\055\252\178\004\000\000\000\000\136\103\055\237\001\031\136\000\000\023\249\064\000\008\031\196\177\251\153\224\066\253\192\000\000\000\010\255\000\133\000\000\064\000\009\134\103\248\135\015\234\032\035\246\000\000\008\130\223\242\003\015\170\017\031\156\000\005\032\031\196\103\249\129\129\176\083\034\247\002\144\009\013\097\159\232\134\006\194\152\068\020\240\002\032\038\022\134\127\176\022\027\050\105\114\197\241\109\130\223\244\175\255\196\000\042\017\001\001\001\000\002\002\002\002\002\003\000\003\000\003\001\000\001\000\017\033\049\016\065\081\097\032\113\048\129\064\145\161\080\096\177\112\193\225\209\255\218\000\008\001\001\001\001\063\016\255\000\003\127\139\127\131\063\151\127\153\241\191\150\254\089\227\127\013\183\201\231\109\241\182\254\091\227\127\013\179\240\223\207\124\111\241\231\254\134\126\079\248\239\254\166\126\079\226\254\071\242\031\147\252\143\243\031\201\158\095\200\255\000\192\191\204\127\033\229\255\000\017\255\000\019\063\131\060\063\158\254\015\254\012\242\249\099\248\095\253\129\255\000\198\191\201\159\192\126\025\248\103\225\150\089\103\227\158\079\057\111\240\229\159\134\126\121\101\191\150\120\127\135\045\252\118\207\057\103\140\252\054\207\061\121\203\060\062\054\207\039\140\178\203\044\179\206\127\022\127\038\091\231\044\252\118\223\027\111\157\182\223\203\124\237\190\079\059\252\025\231\127\013\243\191\142\254\007\157\182\219\127\131\127\143\109\183\249\054\223\057\111\225\190\055\241\219\127\241\143\241\159\251\169\254\049\252\121\252\004\255\000\022\255\000\001\254\009\248\147\249\063\134\255\000\133\191\198\248\099\201\224\243\182\219\252\059\111\248\039\242\063\227\191\129\248\111\241\062\095\059\254\070\249\127\148\240\255\000\239\207\243\191\142\254\123\111\157\183\243\223\007\157\254\013\241\191\193\159\134\219\111\156\179\248\155\063\044\241\159\201\159\142\219\109\191\134\249\223\203\109\241\182\219\248\190\055\198\248\223\059\227\127\141\252\179\198\254\089\231\063\029\254\045\252\115\240\223\203\127\151\046\094\161\189\049\243\033\167\213\123\160\190\164\087\237\112\141\067\244\065\246\076\033\234\126\100\143\180\163\211\038\122\254\029\183\249\095\025\248\237\182\254\057\249\109\158\114\207\027\252\143\254\087\044\141\222\137\189\070\076\237\145\236\236\047\180\023\065\000\244\070\061\070\252\014\087\043\039\142\108\252\089\238\229\117\203\055\024\207\161\150\244\145\233\110\117\204\167\184\091\205\234\097\122\178\207\192\255\000\044\255\000\024\240\249\063\023\242\063\145\254\054\127\131\063\019\119\076\155\235\239\118\099\210\132\232\240\060\125\160\198\108\069\023\145\250\095\165\139\031\016\007\086\095\192\177\086\166\034\073\054\073\232\248\134\189\055\223\202\244\179\252\055\248\050\063\153\143\224\217\254\099\198\079\151\242\127\145\252\073\242\066\244\189\030\089\102\183\203\152\003\224\240\245\224\067\193\041\143\001\049\184\030\002\099\207\202\212\253\060\016\201\156\250\159\105\114\137\225\053\250\076\080\088\229\145\023\122\025\063\091\091\057\187\178\075\063\144\242\255\000\033\252\103\147\241\063\192\063\206\203\047\075\063\182\244\017\199\080\218\096\179\234\207\168\044\120\171\010\072\188\034\099\227\028\226\241\141\151\235\108\245\113\036\252\079\062\047\166\075\007\082\114\071\199\131\031\034\253\101\202\252\035\134\222\046\216\090\190\174\245\183\106\100\226\207\252\129\227\044\179\242\207\057\249\229\150\089\231\063\028\178\207\204\220\191\072\206\121\099\160\129\186\092\173\195\240\026\139\175\082\073\228\021\146\028\201\205\239\225\149\249\147\216\226\029\147\153\208\088\064\054\015\198\033\118\113\252\196\158\066\117\233\151\177\203\030\142\101\239\118\014\037\151\169\058\011\011\232\146\089\115\139\012\094\050\022\064\147\142\046\248\139\093\101\235\220\147\063\060\179\248\114\203\044\252\179\198\089\227\063\149\252\118\223\199\063\029\252\178\223\203\127\045\183\240\201\061\039\115\124\025\131\130\194\008\035\081\152\135\139\244\072\157\066\245\038\243\032\216\061\095\076\157\174\066\110\199\097\253\039\004\023\204\117\056\247\151\101\127\123\015\015\019\161\177\232\048\031\088\251\004\071\102\063\099\135\255\000\199\052\058\126\174\237\157\056\124\094\234\159\016\015\130\125\161\056\103\071\024\144\190\173\125\088\250\241\056\245\111\234\227\102\014\108\153\204\209\197\202\064\217\011\201\178\206\112\219\124\105\056\146\207\231\203\127\060\241\215\141\254\087\255\000\010\065\063\222\109\054\235\067\103\155\131\193\207\132\131\010\060\068\153\190\159\033\159\113\092\219\050\087\234\060\229\242\049\226\253\193\013\003\253\204\058\127\200\072\149\129\053\051\247\154\179\169\061\195\125\203\238\121\237\138\194\253\160\174\063\033\054\048\030\113\180\004\062\097\189\049\145\211\199\166\216\076\085\250\075\039\237\042\072\067\061\195\238\116\181\248\149\120\203\212\100\254\039\252\135\249\223\007\224\127\133\190\015\225\203\047\129\182\208\190\172\199\108\242\240\130\059\060\028\191\137\239\215\047\197\169\241\057\114\012\137\003\174\012\152\110\167\204\104\224\253\177\028\000\023\062\173\206\088\129\128\144\100\178\187\109\223\121\048\166\160\142\057\185\161\211\142\100\119\125\073\030\037\175\044\010\008\126\102\159\051\234\057\202\158\146\237\092\101\254\178\250\120\024\223\166\198\124\192\071\226\073\240\242\029\047\163\137\233\038\089\248\031\158\219\254\075\224\255\000\192\022\183\033\005\214\182\125\100\022\089\023\234\203\035\080\216\124\043\111\116\092\022\059\025\159\049\156\059\244\132\135\183\152\001\198\083\135\190\186\149\226\210\228\231\220\227\221\151\220\096\238\030\005\174\023\051\192\198\232\247\096\242\206\124\079\102\197\164\187\221\196\211\153\206\060\048\083\234\227\007\179\097\086\247\036\208\204\113\176\252\027\247\097\113\117\049\112\091\192\093\035\098\029\195\153\051\201\229\182\221\049\054\245\203\214\201\159\226\031\229\111\248\225\034\113\220\089\222\238\152\019\153\215\134\065\007\205\145\150\089\207\224\111\209\110\245\097\234\216\234\047\131\126\045\085\121\206\039\218\148\112\000\000\014\130\049\111\185\244\219\087\153\196\039\187\094\217\075\031\018\092\218\236\108\059\099\119\210\228\153\155\213\133\182\215\105\159\114\066\028\057\149\222\058\156\029\249\146\029\195\141\094\035\235\139\029\117\198\194\004\123\112\128\225\096\225\217\004\079\118\071\205\195\033\103\031\171\104\073\114\066\217\047\009\107\184\224\140\115\063\127\233\039\166\089\255\000\133\060\103\225\182\219\248\231\141\241\182\219\248\030\054\223\004\158\151\107\164\096\234\247\060\247\061\068\065\036\150\049\162\060\139\123\075\138\199\155\230\094\151\082\064\242\193\182\115\205\021\221\193\199\119\023\045\170\105\104\103\083\007\119\096\188\035\247\155\008\112\074\117\032\089\012\198\078\014\046\069\203\204\038\230\135\120\150\241\212\145\204\094\004\176\176\050\224\099\028\191\081\028\144\140\094\208\179\136\015\211\125\051\065\207\148\033\091\005\098\049\021\237\180\077\205\109\015\018\100\147\046\100\088\038\102\093\073\178\206\047\223\094\167\137\060\239\241\237\159\134\249\223\231\219\127\157\254\028\130\033\199\023\057\145\137\049\243\155\003\004\069\145\014\118\158\075\215\226\206\227\109\131\254\073\216\000\015\137\025\115\120\178\251\131\115\136\047\043\186\127\171\015\044\130\206\161\125\198\122\037\121\179\029\195\033\237\033\050\055\089\030\241\098\055\046\096\138\172\140\195\009\079\022\187\014\253\222\196\030\145\178\131\061\216\222\122\179\200\141\056\119\153\030\140\153\155\040\192\003\131\016\043\055\045\049\212\062\045\031\014\126\167\240\024\239\019\207\155\051\137\039\198\079\128\224\150\139\150\191\243\246\127\202\060\016\219\233\011\136\014\189\207\059\233\060\217\048\065\007\128\136\240\049\153\241\159\100\031\136\138\057\089\065\003\116\217\197\051\164\016\121\200\097\203\035\054\032\214\206\016\201\106\029\196\013\120\139\164\230\115\118\078\236\157\212\022\200\000\201\147\211\073\091\140\004\037\078\011\224\101\219\156\036\071\155\019\196\235\176\087\153\238\092\036\151\102\107\198\094\144\035\145\163\205\134\019\098\012\241\090\034\142\152\048\060\070\057\021\241\051\100\201\207\139\038\124\101\146\089\225\198\013\252\146\239\224\147\255\000\000\126\025\225\254\023\248\130\071\214\221\002\015\066\207\234\212\201\116\152\108\108\152\060\134\087\004\252\092\188\089\103\017\183\060\189\069\065\126\155\012\236\011\000\137\097\113\039\103\090\156\217\002\195\135\060\068\123\184\007\118\103\155\197\131\178\045\046\134\242\065\178\155\151\003\134\099\011\147\200\118\030\009\054\084\112\079\114\073\035\125\083\135\087\054\040\109\151\184\056\018\195\114\012\134\005\101\174\174\013\045\193\040\079\016\030\172\004\019\093\194\000\228\135\108\044\214\051\062\053\066\073\044\179\198\073\225\114\215\184\139\247\122\051\252\162\124\063\129\252\047\147\252\108\131\102\121\001\251\136\126\164\147\155\060\045\130\124\064\130\229\228\012\091\245\227\035\169\072\221\233\133\043\109\088\000\024\024\101\141\222\228\112\119\118\135\048\198\196\203\035\073\116\055\009\070\221\216\110\093\089\056\132\140\248\128\220\141\044\204\182\045\142\075\166\247\061\056\033\046\051\145\055\160\129\056\133\024\003\018\000\226\090\185\096\178\088\028\190\190\172\094\109\218\104\004\029\029\089\163\015\070\002\041\142\115\103\153\212\060\028\094\065\124\067\195\036\130\207\004\149\182\200\030\109\039\015\220\153\254\009\252\079\231\159\225\004\234\017\199\236\201\249\182\217\230\072\057\130\011\035\148\114\137\153\008\218\120\053\183\131\095\086\200\038\243\145\020\117\203\015\165\149\103\100\000\078\160\079\082\092\230\206\087\164\185\213\143\126\165\049\027\061\193\176\094\163\157\155\196\034\240\129\229\014\049\098\059\106\234\215\212\252\145\143\041\003\206\129\247\042\044\003\175\189\065\238\114\090\191\016\186\109\230\050\107\179\190\099\071\035\078\072\168\036\041\220\059\151\114\244\223\125\007\032\235\136\113\205\155\245\248\000\195\228\056\244\147\230\206\124\153\109\093\075\224\117\220\031\116\226\054\127\133\191\192\249\223\059\111\229\159\134\219\249\004\221\248\035\056\017\228\005\150\047\142\071\029\130\056\065\228\070\060\091\093\081\009\128\250\008\096\113\117\005\183\098\109\159\011\039\086\147\194\005\214\055\221\157\201\156\072\068\128\123\097\199\139\248\159\234\016\149\110\241\232\107\025\037\056\153\078\131\201\016\142\080\134\217\143\128\001\207\171\007\032\121\130\209\024\029\245\204\188\056\012\131\170\193\124\027\011\054\124\067\066\072\020\088\062\075\039\036\173\240\065\226\004\238\059\006\014\086\238\108\035\022\030\034\007\050\215\032\129\031\086\157\121\046\075\155\192\150\122\128\100\201\153\102\018\078\030\000\122\130\074\126\111\210\255\000\129\182\255\000\011\252\155\252\069\181\217\028\022\066\091\089\254\019\192\217\060\005\079\000\143\137\238\027\035\032\127\145\025\091\204\012\050\062\143\136\215\013\219\100\039\010\140\097\105\214\003\043\108\003\099\246\043\162\041\187\119\083\220\191\185\137\012\023\096\142\003\227\137\115\227\140\187\155\080\225\109\118\092\141\205\181\044\135\158\204\187\132\055\038\004\247\239\234\119\073\159\048\043\240\017\132\093\242\048\010\034\116\071\006\233\177\201\033\140\227\009\249\102\048\194\029\045\094\047\085\236\070\160\114\206\096\019\224\222\207\016\214\230\240\011\029\223\006\009\151\133\230\106\192\253\217\028\017\156\206\239\189\031\242\054\124\031\145\231\044\254\016\241\191\088\064\028\039\059\191\081\112\240\216\234\032\135\049\066\201\044\135\131\162\227\142\219\151\213\233\133\198\204\001\028\227\165\108\022\032\233\000\001\238\008\209\226\221\144\201\001\076\028\173\180\154\157\145\205\016\125\195\010\032\006\070\064\014\178\214\052\100\039\153\188\001\196\235\182\052\234\208\142\201\108\245\100\112\192\114\188\217\117\031\048\216\113\063\184\126\152\122\158\231\243\145\238\187\235\097\205\080\046\034\192\014\109\025\122\077\154\072\220\109\224\254\160\226\247\023\247\100\125\003\055\223\041\213\190\150\095\129\161\016\152\147\175\043\169\050\206\046\112\035\199\198\096\104\119\242\089\231\063\139\063\137\254\003\242\063\012\242\054\211\151\169\001\133\187\248\002\033\100\027\225\003\152\137\146\062\163\202\053\211\112\090\097\243\108\227\203\025\078\094\182\228\189\107\049\229\061\035\013\160\003\038\007\126\238\215\030\224\027\169\203\003\153\216\071\208\096\122\037\006\048\007\045\135\185\143\079\016\061\140\022\040\090\123\128\150\111\076\139\196\129\061\187\103\196\146\092\182\204\057\132\178\123\182\112\109\059\067\002\035\199\051\026\162\114\140\055\131\131\097\195\139\046\174\135\220\238\109\165\140\177\028\201\024\157\130\001\238\211\057\130\141\012\075\142\113\052\131\015\211\096\243\007\169\161\008\073\036\146\113\008\101\187\006\197\032\089\062\002\032\164\078\250\211\252\119\242\063\136\252\003\102\229\000\060\030\025\005\156\091\146\114\244\088\185\118\130\039\201\139\224\230\046\011\170\126\095\028\221\135\238\079\003\241\168\088\103\137\241\072\039\094\038\057\138\250\144\238\227\044\248\141\014\110\234\222\145\151\120\189\221\132\176\247\204\249\172\112\226\229\018\199\035\166\177\227\139\123\101\146\088\238\122\131\068\211\228\071\123\217\209\174\044\243\221\172\221\226\008\230\198\031\159\006\000\088\230\077\025\056\046\041\057\012\007\061\220\120\217\197\242\051\035\219\171\030\044\102\223\054\018\073\140\140\201\103\183\137\134\109\141\183\059\182\202\073\091\224\152\216\180\110\206\222\215\156\255\000\013\255\000\012\054\246\163\046\014\045\045\216\101\243\016\219\197\194\065\216\013\240\060\002\013\146\012\131\079\052\110\150\223\249\113\096\125\037\129\059\109\220\128\241\007\025\034\122\180\072\118\089\092\129\156\072\064\207\185\131\088\053\184\036\005\203\226\051\227\136\028\215\096\063\115\147\018\124\150\072\246\088\024\251\140\078\014\046\212\096\198\204\124\177\205\146\018\204\243\203\050\184\131\219\056\227\175\168\215\215\247\063\191\250\143\110\167\166\214\039\022\066\029\174\053\121\128\255\000\083\162\112\245\115\210\092\231\187\091\221\187\139\096\239\109\134\253\068\099\166\092\155\163\040\008\125\222\171\154\062\041\179\151\127\028\025\044\009\198\117\205\153\018\108\010\216\079\018\015\019\096\077\238\065\226\127\196\127\012\178\203\063\028\252\142\124\112\235\014\094\229\118\240\054\199\054\238\150\014\032\230\060\165\161\063\009\204\215\041\122\188\067\112\159\009\219\151\022\098\053\138\114\249\092\207\152\222\049\178\097\242\151\120\122\176\035\157\192\204\142\229\141\147\188\014\100\150\251\072\198\198\158\243\141\190\251\200\061\195\247\105\035\014\120\185\130\111\017\155\142\015\167\082\024\015\168\195\162\089\213\206\228\156\065\198\207\007\048\024\157\113\208\171\112\011\041\228\122\133\028\219\065\027\235\109\098\223\134\044\143\202\192\219\023\115\194\143\082\148\230\088\114\193\004\140\077\251\035\073\059\037\114\067\029\144\131\211\052\227\213\155\120\123\131\228\005\205\207\171\155\197\197\226\078\036\151\156\179\153\176\248\184\130\107\050\234\035\234\119\197\229\140\156\088\206\078\125\077\146\126\167\248\119\193\252\015\141\182\219\127\013\243\182\254\005\188\147\017\220\242\120\221\139\183\150\229\206\036\029\130\016\104\249\067\133\137\220\098\092\158\004\119\009\215\162\192\246\027\096\031\024\200\014\036\037\136\097\019\163\137\157\193\103\148\192\150\211\238\085\157\196\185\221\160\056\122\216\188\236\016\116\128\112\093\185\204\009\208\236\209\051\250\141\160\180\135\000\165\192\206\123\230\215\185\101\185\045\241\067\238\069\156\079\072\150\200\056\228\048\030\213\152\156\115\000\117\047\212\084\074\184\091\123\038\215\044\124\012\038\057\158\034\072\114\078\004\147\131\114\237\092\254\119\112\121\177\221\253\154\004\067\251\146\147\190\118\180\222\137\013\223\171\146\202\034\113\019\033\196\206\172\214\076\187\234\215\129\142\164\082\113\220\194\009\015\196\020\059\234\228\189\147\180\207\240\223\240\113\136\194\114\193\185\126\160\197\049\227\191\000\006\067\016\143\048\033\008\044\178\203\204\069\142\004\127\051\045\240\228\224\141\039\253\203\091\037\135\191\153\201\189\074\193\212\231\140\156\015\187\105\121\200\221\057\001\099\117\180\231\000\016\203\151\171\087\006\104\007\175\011\224\095\068\049\052\046\104\101\078\145\032\145\242\040\136\240\049\230\091\212\050\062\086\067\250\152\009\188\204\119\239\212\205\118\061\081\206\247\168\036\198\119\065\015\060\237\176\103\035\018\206\221\210\007\222\230\051\201\125\166\206\208\247\063\038\012\197\230\238\185\136\168\226\094\230\115\026\087\212\012\002\023\130\031\022\008\055\120\056\161\038\252\243\114\065\204\065\060\226\092\067\152\044\223\014\214\097\204\248\027\114\120\056\073\144\217\080\165\023\036\139\117\140\103\047\234\239\248\015\225\127\029\143\226\212\031\048\109\219\011\032\130\039\015\027\111\212\004\028\065\008\066\061\070\016\068\027\111\036\122\216\048\143\098\127\112\217\204\236\174\180\162\249\036\111\060\101\152\025\124\207\049\114\039\150\100\178\198\194\134\030\229\027\119\022\007\178\001\151\105\065\130\031\066\107\060\007\197\004\112\049\241\227\128\117\002\014\072\033\231\142\019\141\226\027\146\093\181\151\060\217\073\183\137\157\251\184\175\156\224\135\161\106\006\000\033\238\020\128\251\133\024\197\196\029\061\157\067\175\169\043\053\055\109\085\233\140\159\185\198\173\213\044\071\194\071\055\187\167\081\232\072\054\247\103\209\141\155\195\126\239\099\238\226\173\234\029\108\207\144\044\055\196\120\186\120\141\179\032\143\002\073\101\155\126\190\014\033\234\055\056\147\004\206\167\147\173\241\014\127\130\255\000\056\092\097\223\139\135\247\036\247\013\141\088\150\124\201\241\027\177\062\237\057\098\016\066\015\003\004\017\023\043\060\029\199\171\208\203\239\036\227\059\056\174\248\180\190\062\011\130\048\207\078\110\093\047\089\230\226\226\117\013\226\248\161\203\024\250\002\060\050\078\193\038\015\155\156\109\210\175\116\228\032\239\040\252\152\139\154\247\056\192\203\217\015\142\015\141\118\117\190\054\122\206\182\055\045\249\199\186\231\129\241\033\157\156\226\184\185\176\157\237\014\015\197\105\030\000\051\162\033\126\174\024\184\193\023\050\099\173\034\045\221\234\100\061\149\142\253\229\247\125\201\062\133\210\023\088\035\197\222\099\019\051\193\177\088\032\048\147\015\202\079\153\126\004\145\150\004\097\042\235\036\094\057\127\130\248\207\228\011\001\073\131\245\105\106\225\103\224\105\148\128\136\033\224\233\007\054\017\216\137\145\190\047\139\214\018\225\250\190\197\135\009\211\012\151\031\191\002\247\062\045\054\099\099\075\060\238\070\112\078\240\196\241\104\049\205\048\184\240\079\076\063\112\134\071\025\224\254\206\100\157\088\223\212\157\054\224\171\103\207\144\163\171\056\230\016\130\217\022\205\133\223\169\254\252\127\088\014\252\054\124\107\174\193\039\133\046\229\030\158\097\065\048\245\013\217\018\219\249\158\235\079\136\212\076\135\006\003\089\068\062\076\137\113\198\236\007\022\255\000\231\183\214\023\078\039\141\206\049\226\109\151\178\066\181\248\048\167\214\250\219\190\243\137\240\015\196\184\203\118\135\118\099\189\157\072\159\009\255\000\043\109\137\112\140\075\247\054\089\248\033\173\206\090\184\064\194\008\113\101\175\055\162\195\195\004\030\046\072\065\133\250\191\094\220\207\075\092\009\183\005\177\226\021\240\056\069\030\037\193\157\216\000\218\028\213\190\160\054\249\126\235\027\104\248\187\031\160\064\197\013\238\060\146\143\152\115\206\039\114\038\026\198\023\001\155\016\055\230\007\113\128\056\141\207\007\122\147\223\128\067\227\187\018\129\132\058\098\150\114\049\043\185\253\077\049\055\205\147\048\211\189\136\133\129\199\115\146\241\233\141\048\199\073\002\060\178\192\156\146\153\112\089\135\122\201\028\170\126\225\179\217\003\061\207\094\098\014\110\095\254\067\099\034\088\242\199\191\030\214\044\025\196\070\229\155\228\016\241\174\084\236\163\039\205\205\153\115\112\141\088\052\242\119\097\254\106\239\014\088\097\099\239\198\022\022\108\114\128\129\241\031\073\221\161\180\095\104\193\179\206\242\158\136\093\254\210\101\251\090\129\143\136\239\080\158\126\171\099\241\154\005\235\110\046\221\036\152\077\201\170\064\119\116\038\205\200\189\046\126\034\025\225\193\010\188\193\205\136\248\247\107\166\125\205\222\119\185\061\214\184\080\198\025\185\018\000\226\060\220\092\201\016\108\016\204\034\123\045\078\031\148\152\201\225\080\004\114\028\230\008\131\207\100\057\135\009\163\021\148\251\110\095\041\215\184\139\191\017\013\131\186\090\230\199\192\051\091\145\009\147\241\105\119\217\105\115\195\115\230\192\124\199\176\246\047\132\044\025\045\146\015\045\153\097\177\223\012\130\063\118\108\248\049\029\201\110\073\036\039\060\006\223\006\043\109\089\112\207\243\062\079\196\252\059\177\126\011\208\248\135\196\035\198\094\237\179\110\004\054\050\077\234\209\234\205\131\096\078\110\094\231\045\179\012\023\102\117\246\099\030\044\230\015\014\246\137\234\250\240\028\116\073\230\039\196\186\153\001\037\066\062\137\057\204\005\178\159\016\219\134\216\061\200\198\044\156\011\100\113\251\140\180\211\184\199\034\057\204\248\183\152\252\001\008\198\223\176\035\161\199\108\056\102\220\073\132\245\181\056\033\205\132\178\186\075\148\028\068\234\074\101\227\122\134\113\187\044\045\054\092\229\131\015\127\080\038\122\139\213\146\249\041\197\125\192\221\099\097\159\113\059\028\120\148\183\195\035\076\083\193\097\224\241\028\178\116\113\012\238\229\099\035\167\131\139\015\049\227\132\035\093\091\199\250\255\000\036\240\024\015\155\126\037\046\096\240\120\219\024\051\192\025\012\078\086\095\023\048\115\028\098\019\162\054\058\048\160\079\195\017\096\178\015\170\060\011\102\124\209\092\150\117\044\253\050\018\187\044\049\207\221\195\033\044\143\089\122\140\192\200\131\073\086\173\193\112\248\095\140\181\181\027\106\240\218\061\065\144\241\062\060\045\050\087\101\153\117\046\147\158\228\030\139\114\230\118\027\239\136\135\184\099\134\096\226\016\121\145\134\072\127\118\157\206\228\245\006\092\066\205\225\018\059\241\013\022\026\109\051\196\163\202\065\163\008\242\248\046\035\060\060\002\027\115\071\107\136\226\204\151\153\121\180\221\110\179\169\143\187\075\135\193\156\230\058\241\146\090\126\229\135\245\228\255\000\008\242\027\114\233\203\047\129\201\115\019\110\072\126\096\030\070\124\006\215\199\196\103\046\084\139\023\168\245\001\239\153\157\233\128\022\113\176\048\069\154\199\037\253\025\112\210\057\120\192\011\188\187\113\099\145\112\027\173\178\247\063\183\132\142\099\004\156\225\200\055\184\223\104\031\115\024\010\253\077\013\024\005\167\128\032\036\000\207\044\071\086\249\079\012\247\055\033\249\103\250\255\000\086\012\063\185\084\199\079\146\200\027\113\153\014\206\150\206\217\114\093\169\133\162\205\219\071\146\224\234\213\212\112\207\114\119\046\207\196\022\169\029\160\197\217\018\224\184\191\008\130\120\128\094\188\031\014\061\081\075\218\031\184\110\160\033\070\000\187\036\178\203\033\227\047\238\185\254\126\252\231\224\217\048\124\022\009\220\156\067\196\049\181\238\039\043\239\117\143\016\215\130\210\232\079\120\179\196\175\216\131\018\056\234\056\102\022\096\059\090\206\247\015\025\029\195\099\158\024\221\063\171\037\242\011\069\117\245\047\024\125\203\110\080\228\064\134\141\145\037\050\130\098\044\208\062\102\092\025\238\236\156\250\154\102\061\017\053\055\012\219\183\198\219\064\088\240\051\006\092\188\239\130\153\013\102\131\049\005\233\001\206\152\058\143\066\194\047\251\145\226\223\082\140\047\054\198\027\107\028\044\044\103\155\110\108\125\193\013\057\183\051\171\038\210\252\089\111\178\156\161\151\089\028\134\206\223\015\063\049\174\188\004\119\195\181\184\200\012\250\102\228\078\119\039\049\002\136\170\022\187\144\146\030\038\056\146\098\253\141\175\062\039\198\127\011\248\239\226\054\200\249\123\155\166\223\081\224\114\042\219\129\012\068\162\225\101\208\059\200\183\060\093\108\127\091\161\241\103\218\177\183\168\206\132\098\132\097\226\077\191\084\109\185\168\130\111\238\052\067\155\120\151\138\177\129\140\103\100\041\221\233\187\081\134\117\222\075\059\019\082\210\104\028\096\066\116\240\009\127\223\030\119\051\044\242\185\005\190\067\146\198\051\121\156\235\198\068\131\164\200\239\060\188\038\079\008\097\202\240\232\193\027\076\079\056\180\050\000\059\054\160\196\199\050\188\216\033\000\064\187\143\197\203\097\188\223\055\021\254\239\065\225\235\114\147\108\160\120\015\079\000\207\025\176\143\059\041\057\009\147\215\097\190\071\124\076\009\194\017\060\189\169\174\029\075\001\041\151\079\012\164\240\220\230\082\231\201\251\159\227\127\132\185\204\224\135\027\103\193\228\213\226\225\227\099\036\183\168\100\072\175\188\226\199\073\140\102\228\222\193\153\142\039\024\204\186\154\189\046\089\108\031\022\086\049\225\162\077\187\151\017\156\062\046\182\006\147\001\242\182\031\130\246\126\255\000\000\075\156\176\015\187\096\230\224\113\034\130\237\212\113\103\199\091\203\103\030\219\195\100\135\077\018\008\166\059\032\173\250\012\119\234\221\195\197\150\028\089\157\092\217\224\003\000\121\220\103\135\044\028\150\048\088\249\156\151\221\225\048\247\055\153\177\189\136\227\185\189\046\077\195\102\090\195\080\116\194\222\230\147\103\134\119\101\187\093\016\119\056\145\067\089\235\199\101\169\146\055\130\070\228\092\236\200\070\007\015\030\060\139\155\195\072\203\207\128\195\004\065\097\031\023\034\054\207\219\121\203\020\190\160\246\231\035\201\134\097\027\186\092\018\199\212\066\228\089\107\103\107\108\156\221\203\033\222\150\145\240\255\000\027\224\252\051\201\212\131\175\152\055\159\015\080\065\037\245\024\101\190\054\124\067\057\142\013\161\158\086\140\230\105\171\132\128\106\030\072\067\121\200\219\068\054\105\242\007\008\119\155\156\017\248\174\146\098\222\183\214\143\229\235\043\185\237\247\030\077\135\169\233\006\219\094\230\066\237\147\130\064\125\217\208\193\206\216\030\071\022\203\082\185\226\000\043\193\204\055\140\039\166\025\023\051\234\056\199\033\233\003\243\182\091\016\219\247\041\243\035\146\218\034\221\120\007\182\108\124\063\035\057\172\094\008\208\025\141\070\035\023\025\032\004\186\233\132\114\251\109\028\022\058\123\098\239\156\148\120\097\237\001\226\071\020\220\176\092\254\224\118\025\073\060\073\122\123\100\201\151\247\115\190\044\129\238\031\030\254\003\100\146\079\002\035\223\135\169\167\119\040\101\189\027\146\175\174\219\225\226\153\045\066\014\019\117\128\131\063\087\041\152\251\129\241\011\224\025\011\111\068\201\254\039\248\011\189\241\102\219\142\126\029\091\102\195\035\203\112\176\057\189\165\094\178\201\206\188\176\046\252\028\177\236\216\142\145\253\119\088\141\055\157\072\231\238\119\137\043\102\016\250\151\187\228\135\054\145\205\185\178\244\007\027\134\116\134\064\017\175\103\093\092\227\227\139\071\061\079\103\101\202\024\055\250\155\048\184\090\199\005\178\086\172\043\146\156\180\122\017\191\204\232\089\186\186\061\036\158\207\213\018\230\231\009\007\040\144\249\129\099\154\126\110\242\236\049\118\037\031\172\057\056\200\046\072\085\031\209\006\063\215\071\164\119\230\098\100\253\008\025\028\135\041\128\229\131\150\105\253\076\217\027\199\011\014\092\179\195\035\203\061\012\173\201\186\141\162\227\016\241\099\119\221\182\152\082\103\184\135\143\100\157\190\102\240\216\030\047\106\195\244\158\203\179\151\018\206\166\036\030\054\030\099\179\208\132\153\167\143\220\207\222\215\132\163\178\059\023\053\209\122\178\057\059\230\225\034\075\172\199\156\076\048\250\241\049\155\083\086\103\240\111\230\120\027\024\243\221\213\158\255\000\006\008\132\067\013\202\079\150\204\029\113\001\207\225\154\112\112\248\038\126\113\139\189\136\079\219\204\066\085\089\205\148\118\006\081\056\151\117\150\230\064\008\057\150\119\105\147\002\048\251\057\045\147\179\153\159\242\190\193\054\097\039\017\134\089\051\108\023\220\165\143\048\248\112\057\151\192\132\111\119\007\190\225\102\201\020\028\118\034\215\109\068\118\025\016\255\000\255\000\082\205\020\110\163\240\071\092\159\184\203\209\246\071\215\127\083\227\195\215\138\118\239\204\197\239\119\182\022\217\240\093\137\028\139\116\158\062\039\081\204\233\080\117\049\114\109\086\031\120\112\009\177\205\154\241\112\077\023\091\181\176\176\127\084\060\110\108\212\125\199\251\109\129\238\208\033\132\054\061\193\129\009\078\048\217\027\228\163\012\062\162\119\104\151\145\036\025\225\142\228\036\057\102\246\236\024\003\200\242\024\003\224\078\219\107\137\253\139\030\157\037\061\091\118\224\070\115\206\233\231\080\150\012\162\001\036\080\076\057\114\103\159\153\254\115\198\177\248\180\195\139\118\222\060\183\173\130\011\097\143\002\201\176\210\201\086\048\131\189\247\012\158\141\241\078\153\063\106\081\015\075\049\172\022\215\009\107\100\236\016\118\098\208\238\229\136\155\228\101\150\220\077\064\255\000\104\150\109\216\103\140\071\039\164\150\029\070\174\151\014\027\172\039\028\180\047\139\093\135\153\033\195\212\008\058\226\053\001\229\196\049\101\246\017\007\164\247\124\178\251\154\122\101\057\201\237\009\167\164\001\208\038\078\228\003\213\131\092\181\112\215\234\093\007\217\050\224\135\020\103\068\214\216\003\000\056\164\098\019\103\238\121\220\244\177\185\020\229\034\014\101\119\078\173\143\234\067\061\093\201\227\050\197\227\220\193\233\027\136\106\226\031\019\181\138\097\109\179\121\134\159\184\103\128\189\219\062\027\102\210\228\013\207\136\017\250\155\241\192\141\181\056\215\132\137\003\149\227\158\162\123\217\029\038\069\071\092\022\089\193\127\204\142\106\188\097\003\073\026\019\150\028\052\216\070\079\124\044\208\222\118\223\139\114\129\034\202\206\252\173\147\252\017\251\025\229\143\027\108\199\196\032\207\001\145\049\028\147\179\044\251\131\158\144\059\185\073\233\059\181\142\094\200\135\019\159\119\004\092\054\053\095\080\099\146\226\124\079\025\153\096\237\232\189\080\051\155\172\188\056\093\143\107\087\156\056\097\096\089\055\078\032\124\022\173\234\211\146\115\245\009\008\108\010\120\075\054\125\217\057\163\220\174\237\181\129\039\237\023\237\053\001\115\028\027\019\223\016\002\122\010\132\002\194\244\145\039\029\035\102\188\093\117\022\150\147\212\206\237\174\029\133\154\043\168\154\118\198\237\128\064\012\231\027\237\043\146\051\036\056\245\096\182\197\052\079\097\220\033\237\000\007\048\006\122\185\159\057\158\114\192\093\101\186\101\202\205\180\247\206\082\048\115\145\204\161\030\033\036\216\236\191\128\230\012\228\153\117\023\196\208\062\141\131\029\219\091\119\125\078\236\114\065\216\245\197\178\094\189\017\182\118\178\242\089\253\143\034\008\082\220\238\096\007\018\175\194\054\233\239\153\135\148\179\212\089\025\147\101\155\115\134\013\184\017\196\255\000\062\124\241\250\183\152\124\115\228\060\139\136\145\096\240\031\152\009\100\231\243\021\105\094\086\155\145\099\118\103\055\159\022\192\022\134\070\207\139\165\221\151\016\178\108\243\153\105\105\204\186\103\206\212\056\252\016\228\135\134\115\189\017\114\070\119\122\180\056\054\214\092\068\195\086\211\115\171\123\091\113\190\032\104\252\220\120\216\167\084\149\003\132\236\206\159\150\033\182\061\245\000\206\099\211\015\018\015\196\229\212\040\142\097\084\031\187\097\126\012\133\072\172\128\056\195\237\168\017\128\024\100\182\223\237\002\119\043\014\027\121\138\195\129\147\163\221\131\057\139\039\105\202\052\132\192\146\154\023\044\002\193\155\040\183\001\217\005\228\224\067\008\007\068\028\115\111\096\190\015\135\184\153\129\217\026\099\147\202\093\081\034\113\103\085\142\205\141\236\025\143\251\068\016\005\122\044\247\078\219\027\248\004\016\208\044\066\059\176\092\126\164\049\115\112\149\187\004\222\220\089\243\060\035\059\199\089\241\252\025\225\252\002\193\138\063\023\184\136\252\007\151\187\178\201\110\017\104\013\093\218\122\190\139\245\183\102\107\232\044\204\095\051\026\028\081\138\099\017\209\054\039\102\111\099\037\116\079\066\228\090\248\225\037\030\096\115\140\243\144\105\030\133\159\094\173\072\069\039\028\123\129\227\220\115\184\208\245\003\119\144\167\207\134\029\183\152\142\049\152\245\010\233\096\068\195\147\213\207\057\138\181\041\195\007\043\019\026\032\039\134\000\140\160\226\067\159\255\000\169\047\003\102\211\149\245\003\007\095\108\054\229\115\190\035\208\067\151\027\221\029\069\221\142\165\140\205\016\143\123\002\019\046\218\026\203\013\182\142\108\071\069\180\124\082\195\245\112\101\193\167\118\200\125\097\028\221\155\215\131\213\222\029\195\037\196\188\075\205\178\197\135\196\129\207\121\117\102\243\031\146\005\196\135\034\119\106\223\054\227\141\143\150\007\232\216\095\113\196\123\085\212\067\188\074\056\230\110\030\153\117\251\089\113\103\193\221\037\240\155\196\238\230\011\252\218\242\226\031\019\227\099\175\039\144\241\015\158\093\182\187\207\234\029\211\192\061\015\185\153\188\039\238\007\029\072\094\160\061\088\016\255\000\076\091\157\188\211\063\003\246\198\240\006\081\227\143\152\103\173\163\051\197\209\241\194\177\088\087\133\146\231\146\213\199\004\143\023\008\093\066\224\118\157\219\139\188\006\205\243\195\102\217\184\227\072\067\109\088\066\158\239\080\244\021\185\063\083\142\063\220\220\185\048\204\206\110\011\165\117\110\153\055\116\142\053\144\017\113\234\195\131\171\128\109\239\238\096\238\174\192\024\082\014\146\062\160\142\051\250\181\100\064\188\195\155\000\037\078\103\093\199\158\058\110\065\033\025\226\036\049\063\068\132\231\185\115\123\151\007\016\097\151\097\234\071\244\145\190\016\221\179\030\070\025\098\028\113\007\078\101\242\074\235\044\182\142\173\050\141\122\144\100\058\226\045\014\045\181\236\101\115\137\001\136\025\174\187\018\131\051\187\227\193\205\182\015\155\219\175\041\041\076\228\100\251\156\031\031\130\067\125\182\073\247\012\249\148\183\086\201\182\216\181\059\059\147\063\151\109\122\189\248\223\003\013\234\013\179\060\016\093\228\190\044\118\190\086\200\078\125\071\014\096\249\142\182\017\213\170\108\113\055\226\195\126\198\050\023\162\224\251\136\177\224\112\121\096\128\027\156\178\039\187\162\114\143\047\019\219\103\197\242\087\204\011\177\126\227\038\139\238\026\253\148\123\184\114\074\007\187\166\234\077\180\212\135\149\151\228\033\124\185\115\030\130\027\056\226\093\143\012\080\216\019\150\244\153\187\203\037\173\123\133\214\097\115\205\197\023\182\225\130\059\098\214\231\203\211\176\118\179\058\103\247\185\206\248\155\156\096\123\091\009\237\051\187\054\231\057\046\203\001\234\197\120\096\228\135\187\131\150\001\017\135\171\039\016\135\114\028\076\093\036\121\068\182\112\137\066\050\042\243\017\195\061\241\212\110\056\114\035\220\031\104\018\225\062\022\176\100\246\079\004\201\134\061\096\061\194\145\053\167\189\186\095\190\183\162\147\071\104\006\107\025\248\220\183\003\177\034\229\139\184\117\178\078\077\153\048\175\185\107\149\093\135\025\019\110\231\092\074\115\038\078\101\242\015\141\219\096\254\019\200\176\235\204\240\231\141\182\057\241\234\030\252\180\011\069\135\171\181\023\166\029\176\004\235\071\232\142\094\095\123\122\017\034\105\079\121\097\064\014\169\167\178\030\001\094\152\048\225\119\195\108\076\213\115\033\207\003\186\218\251\208\066\199\093\112\065\210\110\074\005\253\178\014\016\145\241\127\234\085\103\142\111\112\012\148\121\072\010\119\015\036\048\188\134\049\246\245\100\241\110\005\161\204\125\220\087\199\187\071\129\123\192\240\077\248\180\224\047\076\134\050\023\092\109\135\226\116\231\220\226\060\092\151\015\185\081\219\044\060\057\125\070\194\119\039\187\098\065\204\167\051\096\189\048\061\255\000\114\221\134\195\237\183\105\231\238\058\197\146\056\206\253\078\231\158\094\231\004\111\064\131\227\036\007\025\151\244\189\053\139\184\115\028\097\183\218\229\197\138\030\145\025\239\194\096\240\060\202\116\161\192\132\030\133\215\050\219\167\139\117\179\101\116\186\017\207\012\247\135\063\171\183\239\088\023\212\226\213\019\096\176\099\108\226\028\048\177\000\113\101\058\200\211\213\202\100\095\118\019\218\099\186\132\121\184\156\104\079\081\002\092\152\185\046\110\178\215\198\079\022\089\195\167\249\048\044\111\189\240\247\248\013\216\235\206\001\219\194\044\060\090\108\009\117\055\101\122\075\226\025\246\051\001\218\011\206\019\120\240\128\023\232\088\175\033\168\003\080\061\055\211\073\153\000\119\166\065\173\195\183\233\070\023\239\051\020\150\175\027\014\208\029\056\142\125\118\243\056\077\238\032\201\219\033\201\147\092\119\061\163\023\055\055\016\021\175\092\128\109\181\145\022\072\008\085\230\030\032\012\051\000\059\140\227\105\246\013\250\184\189\207\220\225\240\178\099\080\156\214\090\099\221\130\233\037\061\018\137\194\123\101\062\049\245\127\244\178\020\192\063\112\006\207\102\216\227\155\099\094\046\163\190\045\076\103\162\207\213\252\094\202\217\060\143\150\201\254\133\139\163\153\008\092\092\189\195\149\001\000\226\113\237\183\187\125\022\190\255\000\180\244\001\000\236\189\121\113\182\062\201\159\239\240\216\159\126\226\184\152\026\013\192\144\028\091\121\148\068\143\253\099\046\055\034\200\215\171\130\201\031\143\179\007\164\127\083\110\231\082\180\045\075\137\222\065\044\116\094\219\254\164\186\038\030\008\124\132\114\178\101\208\173\180\233\051\026\178\038\113\164\193\192\172\111\105\062\153\128\244\079\070\111\166\125\079\063\168\197\057\026\059\027\167\228\188\158\094\226\216\123\103\241\207\196\187\190\023\112\073\146\108\064\178\015\195\012\119\149\137\208\004\062\231\028\195\162\040\234\069\140\028\080\111\104\157\045\241\123\031\219\084\140\051\073\136\199\052\023\195\077\027\205\195\007\147\163\163\105\099\156\190\146\049\016\187\115\032\225\204\123\064\218\074\251\155\218\089\213\131\032\239\168\248\102\198\064\194\026\017\035\122\099\024\071\056\213\174\062\015\190\015\191\022\142\095\028\083\079\160\090\220\191\168\248\152\035\087\038\095\096\136\058\223\120\202\107\187\217\008\125\253\146\181\250\018\053\199\217\048\077\254\032\048\169\071\118\127\048\180\222\163\012\005\222\073\255\000\112\077\142\057\153\198\102\112\117\059\210\235\003\125\030\166\008\023\126\103\106\098\021\051\221\196\068\003\221\067\154\072\131\017\121\190\252\142\117\056\059\181\142\092\077\109\141\244\076\102\167\178\049\059\095\049\071\252\241\251\008\231\109\191\012\105\113\039\196\103\001\040\235\223\219\022\008\222\054\202\080\030\146\195\157\097\015\049\151\101\012\097\148\187\085\053\091\011\064\102\090\093\239\139\073\094\179\092\230\125\204\231\092\245\113\126\134\145\224\019\234\009\227\140\249\190\015\067\000\012\227\038\233\027\209\017\224\124\153\142\255\000\164\022\112\201\027\129\243\000\253\180\063\161\216\024\131\155\057\122\251\139\056\213\249\178\200\059\051\206\079\205\213\037\234\007\179\003\152\033\236\184\207\212\152\134\027\012\098\123\066\074\024\143\059\046\242\117\103\016\227\195\148\248\123\155\114\211\249\022\137\240\255\000\017\127\115\101\146\089\225\159\050\059\248\094\175\143\046\084\174\083\038\249\160\156\216\132\100\013\143\032\108\047\156\086\231\202\017\222\223\054\047\155\164\142\023\040\037\134\028\198\195\222\168\091\111\062\231\212\225\226\088\001\193\164\123\141\196\152\103\080\158\226\201\062\100\047\210\211\230\100\106\029\230\143\114\192\063\212\038\115\124\114\244\011\180\137\049\029\193\192\204\219\237\023\238\061\199\132\168\019\166\180\209\030\092\128\009\167\146\234\204\118\079\059\222\201\002\114\230\076\242\191\182\067\129\070\113\006\231\007\210\202\116\015\171\035\048\184\036\084\179\209\032\175\105\198\003\007\040\099\033\120\002\215\199\068\066\127\005\128\188\177\235\072\087\070\253\078\145\115\211\049\050\111\115\100\002\104\145\160\162\241\177\218\117\032\076\040\074\129\224\130\029\153\164\011\047\191\112\069\151\142\002\158\255\000\111\118\121\063\016\057\229\152\089\193\243\118\055\216\140\199\207\168\097\170\115\152\218\030\163\007\081\203\194\204\158\127\080\062\230\059\000\226\122\090\120\033\008\014\113\072\030\231\244\067\088\071\116\060\049\123\089\255\000\052\035\245\242\200\121\212\250\016\078\207\185\023\195\030\201\111\238\098\035\040\238\096\245\232\196\105\167\004\008\090\123\099\000\069\243\059\033\250\078\080\254\134\027\069\088\145\054\240\176\199\158\007\119\070\238\072\056\057\100\078\027\169\169\073\188\209\196\145\236\032\156\228\049\044\122\148\007\123\096\012\198\112\176\043\091\165\171\188\123\046\028\172\146\220\031\139\031\100\045\223\025\114\077\246\191\128\241\145\127\162\178\201\152\038\059\140\156\201\062\062\216\094\131\070\109\241\237\187\056\024\199\030\013\073\063\164\095\028\075\151\053\250\113\115\182\052\051\165\238\132\216\239\038\055\220\117\120\018\186\180\103\006\086\246\079\060\217\199\146\109\193\150\015\247\005\208\122\105\102\231\137\088\126\238\001\144\162\237\224\190\120\048\032\118\024\105\102\198\144\226\061\142\251\217\179\176\126\075\064\208\245\052\140\058\163\056\091\159\129\008\151\014\018\200\215\035\136\102\229\102\000\129\005\132\026\166\242\115\057\195\197\194\091\014\062\175\113\113\196\236\027\222\038\118\178\078\143\034\218\020\102\071\007\000\035\194\220\219\123\254\032\125\200\251\006\239\105\200\178\187\099\139\234\061\194\234\135\112\035\005\229\128\099\048\119\006\093\142\172\029\138\123\137\199\247\094\211\004\192\029\117\117\009\039\040\122\037\234\131\172\142\144\004\024\048\144\204\213\028\241\156\000\027\205\204\128\184\017\255\000\096\192\144\243\083\168\000\100\244\089\128\033\194\160\222\084\247\031\115\251\137\055\131\211\007\022\088\144\120\117\052\097\069\057\057\136\247\028\217\015\250\092\102\247\098\014\241\070\018\212\081\146\032\096\019\064\052\107\027\222\063\036\196\024\251\035\068\028\056\098\225\181\056\096\077\223\128\153\023\219\197\162\188\158\111\073\169\192\113\192\198\152\042\226\110\161\223\187\028\155\014\114\223\075\007\097\156\043\014\224\047\039\003\234\209\213\236\060\147\139\073\017\179\193\102\223\167\127\009\104\126\167\213\151\068\204\078\193\243\014\090\176\111\012\184\061\203\234\130\181\206\243\063\161\194\114\055\108\187\251\128\182\117\142\061\242\071\062\223\004\244\003\133\005\123\152\067\149\048\101\206\094\041\212\205\072\216\061\136\227\212\069\150\100\131\044\175\065\113\247\102\055\209\248\090\155\192\243\000\109\214\022\206\003\140\210\217\184\217\226\147\159\048\233\007\234\236\139\007\229\177\057\103\196\167\067\153\099\072\080\036\029\152\152\135\132\157\006\014\055\015\011\225\153\247\010\223\156\138\030\188\077\181\203\230\096\177\231\080\224\075\059\028\201\129\153\052\011\171\113\128\190\214\216\103\068\123\205\021\231\212\071\159\158\040\080\121\049\003\008\028\046\178\051\131\044\002\003\089\045\116\115\133\056\045\154\172\033\132\206\243\246\147\238\225\238\230\218\093\236\062\071\006\197\121\201\228\024\133\192\109\190\080\240\164\038\154\059\152\041\159\004\203\213\204\195\164\001\235\011\229\251\155\121\064\233\092\193\170\080\172\229\210\047\198\168\213\237\145\202\156\060\193\077\117\049\071\135\095\124\179\040\098\052\182\065\210\212\123\071\245\007\206\236\222\083\238\077\220\076\048\207\058\142\176\251\213\137\013\118\242\023\232\167\011\234\014\091\091\228\044\246\192\116\189\132\136\239\031\048\191\171\144\074\195\053\035\232\149\156\124\098\200\116\060\207\213\178\207\131\115\229\112\132\142\047\226\249\034\007\229\028\248\238\206\032\231\230\062\151\011\229\098\072\093\063\170\091\135\169\133\232\023\222\156\067\238\252\014\120\221\139\149\250\094\233\240\012\226\075\144\099\208\084\048\118\097\163\165\200\132\015\236\096\240\065\100\153\115\023\198\249\063\075\235\170\001\239\009\243\175\027\019\232\221\156\051\106\125\142\107\153\203\172\018\044\176\175\113\179\119\000\023\233\204\014\183\217\122\168\048\195\177\188\094\171\155\131\030\033\016\136\243\159\051\020\232\234\199\201\223\144\056\247\001\014\012\158\012\187\078\195\013\137\022\003\188\072\093\238\236\003\189\014\114\052\080\128\032\015\186\160\181\212\122\045\022\146\108\206\095\022\059\245\112\128\247\113\030\246\031\111\102\179\244\237\124\224\187\028\179\020\004\221\188\148\025\030\084\199\098\052\136\205\177\146\036\182\207\139\015\014\054\054\055\114\030\236\222\110\004\194\239\077\200\089\192\048\247\168\162\244\250\096\232\123\085\005\234\048\236\176\191\030\073\194\003\065\128\009\236\156\054\014\242\222\235\188\023\092\231\101\207\224\075\147\026\247\096\228\191\082\016\126\096\081\239\216\076\184\244\060\221\027\164\117\050\165\182\027\039\054\089\050\105\127\100\159\204\152\124\111\130\224\236\064\142\030\025\225\250\063\054\135\238\249\102\095\037\054\119\065\116\203\109\182\068\251\077\215\073\031\024\245\135\190\049\028\075\134\244\213\057\073\232\132\055\124\145\225\052\157\125\173\154\231\158\105\135\247\159\036\102\088\093\007\054\005\166\136\048\248\106\129\200\140\248\136\025\116\179\099\008\096\185\019\153\070\207\064\096\052\190\208\099\113\209\221\038\150\210\206\096\255\000\032\188\108\157\125\218\030\096\216\243\042\220\214\166\050\059\220\012\158\092\006\014\116\225\220\057\227\213\142\193\179\129\204\103\111\135\039\011\220\198\039\252\165\098\031\070\224\109\113\121\153\204\224\061\079\039\062\079\193\118\195\109\189\221\092\003\073\196\214\067\007\215\088\076\251\080\157\098\112\079\123\153\115\094\114\009\242\246\083\180\017\160\140\048\103\019\220\179\204\142\199\024\221\100\212\132\102\128\050\251\019\138\224\055\162\151\166\086\183\220\053\244\116\068\254\129\015\159\071\139\082\243\195\065\026\028\150\245\245\201\056\043\055\173\187\253\197\191\085\011\095\001\060\172\127\074\153\083\245\023\143\233\135\118\102\037\144\217\227\051\101\183\108\063\131\145\000\007\235\200\109\153\031\114\147\154\093\089\101\156\113\063\198\023\052\159\173\048\105\241\187\151\155\229\167\102\219\109\184\198\223\244\180\142\150\134\023\130\060\199\002\104\099\129\125\166\004\108\178\008\200\228\131\155\158\151\181\082\131\079\181\009\242\215\038\250\098\172\041\193\225\154\027\209\235\194\121\066\008\131\153\007\229\045\085\190\131\028\071\147\067\079\153\150\193\097\134\195\092\222\036\195\183\011\156\004\016\052\179\043\151\004\055\026\244\136\171\213\236\059\100\019\005\109\078\014\184\205\248\148\151\183\200\080\235\147\032\173\174\188\182\080\217\099\100\057\234\212\061\078\046\069\227\238\250\091\104\188\103\197\195\036\008\203\168\234\241\107\014\155\205\190\116\028\049\138\143\066\247\010\236\099\015\088\159\222\130\124\187\136\139\123\247\039\050\088\073\140\229\152\075\145\176\163\160\097\081\031\109\005\143\182\026\046\060\237\183\222\023\085\062\035\238\219\097\181\065\246\149\224\027\176\022\231\107\029\203\063\189\063\169\147\015\045\140\226\168\055\235\139\021\031\233\141\139\224\207\118\071\100\250\179\195\113\190\197\063\145\109\094\002\024\135\045\159\212\220\159\193\114\211\226\022\143\244\188\002\230\252\192\240\205\187\150\223\043\016\187\126\114\163\022\115\217\173\247\140\004\075\217\243\226\115\012\048\195\030\022\231\061\209\156\089\132\197\023\058\217\204\061\035\038\252\107\077\103\213\008\142\088\225\178\247\107\154\191\115\001\140\187\229\216\048\177\186\095\189\164\013\074\127\129\228\076\023\076\022\105\071\039\113\100\200\126\161\135\155\023\014\072\099\235\194\027\100\026\245\098\107\203\220\107\171\198\066\150\170\150\227\173\231\048\117\157\249\143\144\061\073\204\241\111\182\237\130\125\092\012\217\174\202\058\051\226\119\013\011\166\119\243\021\212\175\196\215\000\184\248\231\058\202\125\207\010\067\180\030\044\140\211\142\097\017\102\024\230\022\023\056\136\113\064\121\048\227\123\026\156\025\215\048\179\091\068\006\021\240\222\251\182\231\028\027\000\150\014\205\009\225\228\046\059\208\226\219\239\110\077\224\044\187\087\238\219\098\220\156\111\074\182\150\007\210\153\135\095\171\038\251\093\191\092\004\223\007\194\051\041\065\191\174\092\235\111\160\158\147\050\113\061\204\057\146\206\033\060\092\130\057\224\252\008\108\081\012\048\216\150\236\195\108\018\109\245\007\139\067\235\133\201\123\240\077\008\125\199\089\225\247\108\049\225\076\246\081\058\005\138\057\087\000\039\219\189\103\022\033\139\177\224\062\022\215\246\013\035\226\227\210\000\046\116\192\135\211\142\134\057\065\201\247\219\099\030\175\181\253\233\108\050\203\187\217\160\066\075\133\136\247\081\106\113\163\068\029\095\115\015\134\106\150\066\023\016\015\128\072\053\156\143\017\237\130\024\196\232\151\017\027\208\095\097\028\069\033\153\188\194\141\213\193\127\214\224\046\028\054\063\110\100\228\188\203\108\250\226\244\009\201\179\150\030\242\109\111\059\237\178\090\122\230\002\211\061\076\078\076\106\144\190\046\195\088\204\235\116\051\174\163\140\155\207\016\220\220\047\187\211\118\118\114\076\240\248\037\144\057\183\159\117\131\160\198\216\104\240\225\180\180\094\175\023\081\028\248\016\198\251\211\111\059\226\220\249\211\013\250\120\142\015\238\219\099\019\169\067\135\203\008\119\249\044\128\113\035\065\143\253\023\213\089\240\027\132\049\217\241\083\253\178\228\169\201\142\036\184\079\055\182\232\093\231\192\149\234\192\226\208\252\136\102\195\152\250\182\024\132\219\088\248\222\098\251\184\252\202\211\067\253\186\103\122\224\159\086\203\109\182\229\182\231\054\139\017\135\124\206\191\030\145\120\241\210\231\022\176\176\203\224\041\245\099\037\157\178\248\149\234\047\082\084\051\175\069\164\227\147\152\131\114\071\166\012\023\008\224\054\199\175\118\236\117\245\105\238\230\013\134\206\199\020\007\126\025\042\053\056\098\244\138\018\059\111\198\185\004\126\029\145\168\139\016\156\058\178\235\035\046\120\204\127\094\096\255\000\107\028\031\171\097\049\161\235\237\086\157\204\227\136\181\185\086\113\112\035\115\153\109\031\115\071\165\063\168\238\013\061\051\070\125\191\104\117\141\237\150\236\029\019\223\000\088\218\155\196\179\103\029\022\200\120\224\079\086\110\059\219\015\027\091\109\025\164\237\058\215\210\149\157\125\195\126\244\167\024\106\076\122\081\045\175\108\186\095\046\118\107\062\056\149\109\183\198\194\090\049\247\197\241\032\199\233\057\064\238\250\143\185\035\109\201\099\159\238\185\254\229\015\233\023\109\105\245\150\132\226\050\092\025\248\182\228\246\126\108\055\097\159\129\190\011\013\079\225\176\195\030\063\118\065\004\245\008\184\193\183\239\092\023\213\024\008\101\182\210\221\183\192\227\176\168\061\201\219\153\117\219\101\184\119\107\231\027\111\210\217\113\023\184\244\079\240\134\061\226\028\060\012\181\251\061\093\081\021\057\019\048\065\253\128\067\206\027\002\248\203\140\226\172\008\217\128\110\007\032\233\015\027\024\158\107\045\116\220\049\177\048\121\227\108\094\081\074\181\009\120\060\244\004\041\112\218\068\003\183\130\116\248\065\103\059\000\249\088\064\142\045\206\164\039\163\173\134\116\019\096\022\000\192\237\192\066\251\070\050\142\028\061\032\173\236\229\225\249\255\000\052\052\222\177\152\190\199\002\027\039\134\027\055\204\118\215\121\038\019\178\053\049\120\189\231\078\213\000\203\098\125\246\079\163\153\041\153\233\133\142\124\195\143\085\038\185\175\162\251\097\064\176\067\062\012\076\027\131\119\203\003\056\216\064\098\059\190\063\049\235\189\173\222\165\159\133\250\095\055\105\247\078\127\165\058\030\157\180\059\210\122\240\082\120\207\159\005\151\193\145\203\031\219\242\026\145\007\242\234\013\186\248\008\037\193\253\075\127\108\089\119\033\015\063\093\029\045\182\210\032\142\093\199\024\125\219\224\109\134\064\216\014\195\235\034\156\113\234\049\012\054\223\160\171\232\170\089\254\184\139\161\091\176\123\106\071\022\145\153\242\083\164\129\071\220\028\000\007\150\007\076\083\156\185\139\180\204\177\188\180\062\059\217\233\163\150\011\209\231\046\216\158\167\050\008\119\027\191\223\132\135\140\128\191\169\106\218\075\169\221\214\124\057\222\147\005\240\161\130\116\002\063\252\071\140\222\049\249\066\242\119\160\044\107\245\023\028\004\109\076\232\160\137\250\153\054\235\101\247\151\098\126\068\067\016\098\029\175\177\020\198\243\032\224\234\097\010\062\140\240\113\094\253\013\156\243\163\124\125\150\125\022\200\125\204\031\251\115\183\236\046\016\227\020\113\222\244\243\099\238\040\110\092\157\178\117\105\099\201\147\108\111\105\140\120\006\159\236\136\083\171\094\229\062\047\236\013\206\092\072\214\124\023\032\253\207\063\171\135\018\248\036\046\103\195\143\114\120\099\117\190\015\192\097\024\008\044\178\203\044\186\120\004\016\067\136\213\167\227\129\158\049\215\030\054\250\218\087\079\027\139\115\059\179\131\004\203\121\199\192\226\091\109\142\232\161\029\034\224\100\224\030\007\119\079\213\165\245\134\248\226\088\095\005\200\206\126\162\248\021\160\072\251\222\099\086\094\050\248\199\102\103\179\128\131\179\152\167\185\021\254\177\224\075\205\194\235\049\142\094\036\119\060\032\200\062\190\002\246\204\018\058\130\080\142\005\250\240\199\095\155\231\016\251\120\156\175\167\056\097\061\133\195\028\241\158\146\032\145\121\248\002\101\247\016\241\254\188\090\215\191\127\128\221\031\184\198\158\150\197\143\176\169\101\234\012\230\093\157\231\187\158\126\011\245\182\221\123\131\008\119\210\231\051\036\223\211\033\189\141\160\067\229\104\135\130\233\065\115\045\185\140\005\120\010\078\013\224\242\203\119\247\197\129\188\060\163\003\246\084\053\108\188\078\024\215\238\027\108\041\197\198\189\061\166\086\065\239\195\166\091\227\147\019\176\095\209\241\062\052\012\130\115\014\151\121\063\030\120\109\242\063\009\228\015\252\201\159\144\216\097\004\016\067\187\133\066\066\254\138\249\183\021\232\182\216\190\162\167\237\113\049\247\023\088\027\204\244\102\207\014\123\186\038\142\078\126\245\101\183\193\139\176\195\204\247\137\231\216\039\247\042\246\190\107\173\091\182\227\233\044\097\230\099\222\174\051\038\025\119\075\110\216\060\140\165\056\123\004\119\234\057\247\109\047\161\056\035\007\040\253\014\120\051\187\242\098\248\102\131\028\183\191\004\105\028\038\020\241\188\254\236\138\112\192\099\089\055\068\105\039\142\005\159\086\139\216\129\026\006\107\113\156\182\064\240\006\226\012\247\199\233\058\152\141\199\025\015\021\153\116\054\048\044\254\231\174\252\112\143\184\032\176\037\224\097\183\180\208\066\187\073\167\204\108\193\212\033\252\148\170\110\042\113\176\019\031\159\126\023\057\248\190\180\163\022\057\024\123\159\185\133\235\138\039\007\205\182\219\243\109\019\113\230\236\091\215\024\122\129\241\104\067\217\003\185\126\027\047\187\253\109\127\071\038\045\157\194\027\235\047\208\217\196\097\057\190\052\156\182\204\154\248\112\130\005\172\249\060\014\031\185\129\248\198\189\220\145\008\076\226\056\055\005\092\139\223\142\162\209\071\139\036\142\032\253\102\185\019\228\049\082\211\016\079\211\016\195\187\109\150\012\178\062\038\038\132\226\205\241\134\161\209\099\168\142\225\163\247\055\142\247\115\138\228\124\069\163\046\168\103\204\008\231\168\071\120\113\187\139\220\085\251\156\179\227\033\156\078\236\045\241\131\100\014\000\099\253\135\123\065\149\107\130\152\185\002\229\186\175\038\055\238\235\196\241\241\057\091\012\234\024\136\222\010\032\209\128\136\193\150\215\123\247\183\186\145\025\009\229\226\234\231\140\055\088\195\094\012\128\250\132\157\066\223\027\024\066\031\241\153\142\242\092\075\115\203\099\253\115\249\209\144\028\124\091\098\209\123\074\076\226\091\102\188\199\239\136\001\203\152\088\236\176\109\140\228\253\120\111\204\112\222\154\011\189\068\159\216\132\245\174\218\224\253\121\119\213\168\237\008\230\163\113\145\081\185\188\077\242\122\159\025\035\150\167\219\031\234\235\027\061\178\000\124\108\254\043\097\210\078\039\134\025\228\135\050\018\065\204\150\109\163\192\142\108\126\028\092\227\118\210\060\242\233\128\242\144\134\127\064\204\007\230\143\232\083\237\050\062\060\164\155\219\175\007\119\172\001\143\189\007\032\160\115\111\062\028\200\188\057\047\080\238\236\091\186\081\210\016\100\016\115\029\015\220\201\124\214\130\089\158\122\049\206\178\243\197\162\042\231\136\111\184\223\152\047\180\037\191\023\060\252\215\105\017\076\155\126\034\082\192\198\193\223\197\195\059\086\097\023\215\086\071\119\107\098\235\253\207\244\138\070\087\075\137\007\083\217\115\006\160\203\236\042\114\091\107\118\012\224\009\153\140\026\157\148\142\065\220\099\090\191\009\175\156\023\017\031\220\212\051\079\005\241\103\191\005\236\221\167\072\028\142\014\120\241\112\120\171\230\126\068\243\219\228\011\000\033\147\188\041\253\094\135\151\234\230\133\150\079\023\051\250\071\042\196\169\129\119\056\072\119\152\124\109\190\054\094\249\180\111\098\217\243\051\238\141\221\181\251\175\222\040\050\123\132\016\065\219\036\179\159\041\213\195\125\067\063\100\126\003\151\212\015\115\101\150\071\144\089\005\198\093\239\220\121\158\023\124\065\135\230\189\199\136\184\111\029\063\129\056\191\213\051\119\193\212\183\188\115\048\011\114\099\003\150\122\158\174\045\208\234\254\174\151\238\244\159\037\128\112\062\008\057\187\207\182\044\205\108\195\167\054\115\222\227\147\051\136\057\241\218\018\034\067\102\024\118\095\112\216\190\118\198\114\214\084\014\063\192\114\012\119\204\253\067\013\134\115\186\224\064\229\131\107\191\086\243\196\067\017\102\014\089\205\192\027\025\187\154\148\116\188\033\003\246\153\251\179\114\247\032\241\027\212\059\176\161\199\133\133\161\244\140\015\226\223\118\224\140\224\141\177\167\162\122\223\159\007\221\191\174\057\071\119\052\121\067\157\033\203\106\217\118\109\181\148\054\182\241\037\146\109\234\246\022\003\039\233\102\113\083\056\044\179\192\247\251\189\175\181\056\168\150\228\056\088\132\249\060\114\173\031\143\116\063\106\168\135\176\154\157\175\136\228\059\214\027\150\120\037\145\187\051\039\144\219\037\253\023\253\223\129\224\114\038\088\242\183\040\246\146\206\037\206\236\188\223\026\181\236\253\047\128\182\228\028\199\213\021\143\012\158\099\169\222\077\079\220\201\231\120\236\033\113\029\082\219\025\013\230\102\190\125\088\083\135\091\024\113\006\095\237\240\074\029\011\151\049\136\050\052\031\115\217\215\220\211\190\233\131\205\140\014\243\096\221\062\188\026\044\191\123\144\001\007\054\048\197\119\236\134\107\194\050\008\067\071\048\196\242\177\011\189\233\030\145\046\028\089\075\243\044\012\206\018\058\103\060\065\091\155\082\194\246\152\121\130\034\019\212\078\151\028\099\188\014\046\045\061\217\182\077\008\072\102\140\246\068\107\001\243\116\047\233\123\002\125\206\007\025\126\039\153\224\252\221\065\023\230\030\220\092\079\206\248\018\202\019\143\017\005\243\031\212\044\113\047\068\145\211\149\147\228\117\099\227\134\090\252\107\025\240\211\079\178\101\144\066\028\066\236\123\203\214\019\187\153\059\175\088\022\198\203\044\130\123\246\089\240\036\147\239\070\109\014\130\126\224\097\112\031\114\225\039\018\073\097\177\118\201\044\134\027\014\054\193\119\236\143\246\063\002\215\095\115\112\102\024\120\241\220\045\201\103\022\132\110\121\233\046\039\247\109\115\135\087\235\013\241\132\016\069\156\120\060\111\254\107\128\152\142\145\179\056\177\024\246\093\241\145\175\163\164\075\065\117\182\157\003\053\151\236\120\159\220\035\148\065\057\058\185\247\224\126\136\206\153\214\047\072\114\079\059\246\207\118\079\017\073\161\179\224\071\047\255\000\222\158\163\117\016\005\196\038\111\006\071\054\074\113\243\106\046\078\163\224\063\075\003\234\121\134\086\215\029\188\023\196\166\178\012\013\151\115\056\181\220\242\241\185\115\190\034\126\112\003\102\017\135\150\135\222\060\099\166\049\114\091\140\183\038\084\220\186\198\120\017\120\078\228\044\222\033\088\239\182\097\240\240\142\059\236\182\246\191\084\027\009\056\210\192\093\029\101\004\191\041\241\095\184\234\208\189\192\048\014\009\116\120\210\058\023\223\229\159\198\106\182\079\006\089\014\046\177\202\126\245\026\157\180\208\066\176\102\161\205\156\217\103\142\018\134\097\198\167\044\167\161\217\011\061\056\095\102\080\104\007\162\020\114\029\147\199\166\236\248\122\150\091\123\241\156\199\063\000\191\239\241\235\110\091\023\184\188\124\022\086\250\158\093\195\133\147\107\251\095\059\139\146\062\036\235\156\114\131\122\179\234\200\233\151\217\055\110\159\206\017\063\220\081\083\218\001\022\096\070\122\213\239\153\174\144\248\150\179\231\125\013\163\234\125\133\055\060\053\019\246\249\062\208\130\253\192\177\206\124\003\032\231\155\239\239\119\097\120\186\099\173\139\167\181\063\058\209\197\212\031\118\125\198\022\211\198\211\090\223\011\032\230\017\024\235\107\183\102\197\056\201\180\116\078\098\117\020\102\051\166\051\209\185\216\146\075\119\231\020\032\092\240\099\218\208\036\081\181\205\037\062\132\239\102\073\053\140\089\227\057\146\076\144\028\060\196\051\062\022\125\098\048\204\232\173\045\239\107\019\057\187\178\234\061\024\088\029\192\014\044\029\178\147\200\226\249\042\006\042\039\195\050\204\189\095\221\151\216\203\244\020\077\136\146\112\220\095\180\025\251\033\154\124\216\094\119\088\235\061\093\207\012\120\018\025\125\059\149\194\157\206\091\165\112\118\039\218\228\249\110\213\226\008\101\227\207\118\059\153\220\183\123\132\242\220\063\109\191\128\127\190\208\039\190\011\143\135\053\128\230\062\118\015\153\163\190\030\175\067\228\062\228\107\141\183\038\176\231\091\134\122\064\106\250\240\007\135\216\069\135\239\136\048\026\009\148\020\246\068\183\211\243\122\099\144\247\028\246\079\203\126\176\000\023\007\028\056\113\234\142\193\173\097\187\233\240\146\194\248\108\237\131\242\097\243\083\198\068\178\136\143\221\035\141\008\153\014\137\168\157\066\052\228\193\151\207\195\061\221\165\203\119\207\017\150\188\103\107\151\107\067\105\159\104\045\195\056\023\221\136\238\008\018\046\110\057\210\224\195\226\028\240\102\182\192\245\200\156\191\165\137\241\028\194\059\196\228\037\224\114\124\217\182\030\018\003\124\047\171\001\247\107\016\030\219\067\024\097\147\222\011\165\153\202\140\159\186\037\162\192\253\067\211\029\108\090\092\100\074\093\158\139\147\053\029\203\150\233\143\240\052\079\137\039\133\160\248\201\204\253\214\223\216\113\128\098\075\056\097\205\247\057\150\014\014\054\076\230\058\137\156\233\192\141\138\030\073\014\056\148\199\035\006\084\180\205\156\069\209\221\145\191\045\244\251\110\146\230\015\153\219\109\159\054\203\115\179\036\022\144\240\252\187\254\217\236\090\079\145\182\090\023\125\187\016\228\044\124\219\111\134\152\249\008\125\203\105\033\014\207\096\166\184\150\065\133\190\172\127\092\056\253\195\238\119\045\189\073\204\232\067\014\252\143\080\196\032\248\034\067\243\069\239\080\002\244\190\059\124\100\162\048\132\078\147\210\092\179\016\106\250\045\067\206\094\103\044\105\028\193\195\140\055\166\060\066\028\078\053\178\065\202\194\248\074\107\194\004\027\156\212\198\014\013\219\071\125\165\146\023\216\090\215\230\073\154\200\137\079\081\111\242\232\139\196\095\086\249\195\190\086\086\158\219\121\048\048\076\089\170\135\067\168\037\152\138\041\078\184\012\237\199\011\031\171\037\120\026\176\189\031\036\069\021\238\055\130\185\234\079\082\036\216\110\058\027\077\130\154\037\197\099\242\018\014\050\127\210\231\100\158\062\167\048\237\214\161\206\076\241\031\007\047\058\174\144\051\159\214\071\147\200\102\088\115\083\072\143\029\220\066\213\102\124\006\006\078\236\131\170\121\167\076\224\092\255\000\250\154\040\134\208\071\169\118\090\075\192\193\134\206\055\211\027\014\110\174\201\200\058\068\100\031\062\061\114\233\064\149\130\242\118\064\116\053\169\003\114\178\108\087\147\008\120\046\206\178\140\201\038\099\100\172\005\062\082\004\248\230\034\205\057\196\192\126\151\018\114\091\131\196\237\043\032\226\092\234\112\243\107\193\031\112\242\100\247\240\011\015\219\061\178\152\178\046\219\117\230\246\143\152\240\054\014\037\086\119\114\166\188\110\108\240\064\092\077\137\178\036\199\028\072\236\099\202\005\097\201\059\115\029\102\126\004\110\077\158\205\068\062\047\027\176\119\160\203\246\127\083\215\177\217\163\238\047\023\213\047\048\135\047\142\205\143\055\018\030\007\098\197\151\162\108\240\167\213\203\220\062\100\161\201\224\011\173\135\184\192\121\215\195\026\000\195\068\130\177\071\171\183\004\008\037\200\092\225\184\005\193\058\171\078\096\036\039\153\069\057\146\092\199\020\092\097\097\205\206\095\016\126\226\124\145\238\002\060\029\164\085\077\028\236\006\128\030\139\219\220\156\204\070\106\001\154\182\236\215\005\240\147\164\107\124\154\011\000\248\209\046\207\027\205\189\141\154\051\156\055\238\237\101\227\016\079\020\081\128\070\067\121\198\049\132\251\144\125\225\008\159\018\117\101\232\112\163\026\157\227\058\019\241\108\226\201\078\140\122\130\207\091\232\071\046\004\249\149\044\184\050\040\037\125\066\123\242\020\196\206\038\070\039\121\185\129\186\009\168\195\144\245\005\245\203\047\047\119\191\135\019\232\039\088\214\240\130\016\136\026\013\070\103\018\089\103\232\102\124\015\144\189\105\162\016\236\034\070\000\097\134\220\235\036\207\131\017\157\003\015\041\009\199\068\126\171\033\201\236\089\161\014\050\027\155\019\154\241\104\199\020\147\222\038\043\179\140\199\000\035\007\179\134\039\175\030\057\066\234\197\177\199\020\194\198\172\219\057\140\111\169\027\061\255\000\194\110\051\037\201\079\133\238\247\003\123\128\133\151\096\046\070\051\143\198\150\023\155\137\086\089\155\176\113\110\074\058\136\109\201\225\176\220\165\202\208\138\142\135\155\016\110\154\228\211\147\196\145\064\081\204\187\014\097\156\227\048\028\151\054\032\056\048\009\223\119\000\128\235\071\023\198\187\197\185\197\166\021\185\212\160\227\212\169\122\225\110\159\175\171\149\103\213\219\149\032\253\058\196\033\163\238\110\165\063\055\094\064\119\048\049\207\017\115\095\164\065\200\125\193\061\185\024\229\231\165\180\201\051\104\115\155\089\203\028\152\090\105\188\017\092\123\066\067\145\171\234\023\191\212\014\056\021\071\226\231\255\000\246\210\036\124\092\122\126\215\113\255\000\113\209\195\250\129\089\215\108\058\167\094\096\123\195\204\132\094\219\187\056\027\140\022\086\250\120\156\224\204\113\135\066\211\189\253\218\250\219\226\080\139\125\177\035\128\231\091\181\103\002\146\098\252\131\034\001\239\092\047\016\185\048\158\203\173\088\244\216\226\044\047\104\067\134\221\200\159\153\171\166\124\016\248\035\208\039\042\192\143\011\177\142\151\009\194\200\185\078\152\233\183\062\139\028\226\034\094\251\102\015\063\118\111\021\218\003\122\025\168\216\117\038\034\120\008\136\216\211\198\101\178\152\236\219\191\119\000\075\041\068\125\022\109\139\174\008\226\236\031\051\085\061\242\092\083\150\039\147\189\236\115\061\061\200\118\113\208\090\205\250\136\221\246\043\027\251\245\126\035\044\080\230\199\071\028\207\154\062\140\004\051\075\175\005\248\176\116\050\044\062\129\008\017\155\133\156\147\194\221\097\239\026\119\187\241\177\203\251\124\010\093\153\150\092\179\123\157\079\131\136\247\061\252\108\159\214\093\089\204\234\195\096\252\001\196\065\146\101\136\159\016\233\120\196\226\030\205\034\120\174\065\178\104\180\121\155\166\095\005\195\013\124\199\138\058\116\212\036\250\135\040\055\150\062\077\152\203\067\101\018\178\118\203\156\140\071\155\019\120\226\120\117\002\240\142\178\056\039\028\214\230\025\249\201\173\203\097\243\089\142\195\247\112\140\206\115\002\008\236\102\055\012\251\183\048\122\018\052\234\067\218\071\065\112\012\003\216\198\092\104\196\220\231\181\118\080\030\178\250\038\107\057\083\092\229\242\031\212\124\007\238\062\007\177\055\061\177\041\085\231\118\025\207\187\226\092\253\079\011\088\193\003\026\057\199\086\192\137\215\113\046\016\108\014\000\155\122\207\146\023\166\116\067\048\056\067\243\113\228\057\186\223\209\071\246\124\207\252\193\018\115\201\008\018\188\038\222\014\189\054\134\062\226\220\215\017\123\079\149\114\101\127\092\199\079\115\176\212\231\208\033\176\126\144\039\123\243\247\050\026\044\072\244\047\048\169\200\019\115\094\039\204\204\204\064\192\065\030\069\050\047\174\120\086\014\030\018\058\001\027\102\216\009\097\243\182\047\189\184\161\215\196\225\032\028\243\011\198\129\197\155\113\240\164\186\233\015\077\062\167\220\147\126\249\253\219\134\111\151\053\160\179\020\135\167\034\181\157\089\239\037\170\192\156\144\085\137\195\123\214\248\156\088\255\000\185\004\115\110\158\198\035\200\060\171\024\077\227\082\096\253\185\166\113\194\160\254\222\010\018\204\248\062\230\216\131\096\198\064\184\206\047\251\191\030\166\246\090\243\108\107\023\169\179\111\140\004\012\178\028\093\241\224\080\123\100\199\157\213\096\014\087\148\134\000\206\134\043\151\115\147\228\159\171\022\216\092\143\221\212\220\200\098\007\075\090\169\154\237\131\058\054\015\148\250\097\083\007\166\232\016\238\198\199\242\078\137\034\117\207\205\135\156\122\142\252\254\208\074\113\220\055\226\229\234\109\208\176\251\177\157\041\210\108\061\054\095\239\008\237\073\158\231\000\241\059\034\062\136\031\110\065\151\030\070\206\032\218\031\068\023\254\235\180\077\157\151\237\077\078\054\022\159\161\024\139\247\143\112\089\231\224\201\190\189\110\106\115\209\157\205\144\086\136\191\164\043\158\124\092\082\039\204\031\068\111\182\016\215\088\126\035\150\023\247\025\158\023\014\156\221\161\147\224\117\243\045\255\000\052\031\130\079\076\232\033\243\062\035\102\122\184\205\062\206\104\113\254\008\169\188\171\237\145\247\178\028\039\103\205\079\131\092\179\125\097\200\253\180\015\238\131\127\167\155\251\107\230\039\184\111\146\254\174\212\051\221\168\067\153\175\023\228\176\226\114\126\167\200\211\230\009\159\010\199\166\103\162\010\115\210\251\035\143\214\054\007\033\220\245\163\096\061\057\061\209\061\017\210\033\198\145\155\062\070\212\078\150\236\211\217\102\153\125\206\058\122\167\200\101\058\015\155\137\254\217\106\250\158\096\150\072\202\193\114\152\119\008\203\211\001\013\231\169\120\125\195\039\212\245\055\070\094\101\182\080\194\097\180\144\011\245\063\236\126\060\199\221\154\164\211\143\005\155\016\243\103\004\033\175\049\092\224\010\193\195\171\149\061\006\206\070\130\072\211\189\044\157\163\181\192\182\244\219\181\068\053\145\245\051\061\237\167\054\003\230\047\177\065\024\099\015\118\047\115\006\046\028\147\216\123\112\070\239\106\237\143\168\080\053\089\196\038\047\203\036\120\060\113\178\122\067\245\034\221\071\027\093\143\177\112\198\146\188\241\220\154\224\203\022\199\212\116\117\135\009\250\186\099\126\111\170\062\167\216\160\142\208\222\204\131\005\219\097\121\121\201\012\000\207\043\031\196\133\195\212\003\136\125\093\186\202\061\161\243\160\124\191\118\232\105\246\092\145\239\226\046\051\019\142\110\239\044\061\108\158\249\243\026\100\143\115\224\002\215\181\146\143\104\023\180\117\160\098\076\248\229\036\152\241\114\143\246\157\013\143\108\036\130\157\145\060\042\066\116\150\255\000\073\075\138\118\015\203\031\188\007\219\032\227\091\127\034\052\239\103\167\007\199\077\079\232\046\016\035\241\006\124\198\185\137\194\074\253\092\058\183\228\216\123\100\126\210\044\241\004\045\009\134\110\222\063\080\140\223\247\119\139\037\149\039\230\017\188\027\038\006\252\217\064\008\201\243\136\197\083\120\137\160\215\153\012\012\038\103\141\183\121\189\219\040\103\182\092\151\212\181\126\227\240\121\001\050\088\193\196\068\057\180\200\068\167\049\197\164\141\114\148\113\128\036\000\169\128\167\235\011\049\003\060\218\173\062\153\084\029\090\191\072\118\157\248\003\129\032\134\104\113\007\010\217\031\118\015\228\186\094\195\038\158\067\168\124\220\245\046\065\149\125\117\150\185\182\015\136\252\017\157\177\013\172\040\025\217\245\031\235\067\061\206\006\091\167\003\102\064\228\157\006\114\195\214\037\238\126\134\098\145\062\115\006\225\242\017\228\078\145\072\241\083\125\013\169\202\122\073\083\159\234\064\088\014\102\037\224\228\059\026\030\238\087\060\217\185\249\092\016\249\118\067\239\206\222\223\137\178\018\116\051\069\017\238\021\135\107\178\226\131\228\132\161\156\216\175\143\068\143\014\002\212\087\235\044\160\252\083\185\200\249\084\007\053\247\028\028\195\217\012\247\194\069\161\132\209\070\036\161\232\059\096\008\139\057\101\211\103\141\093\246\198\054\007\217\003\052\190\172\093\021\242\207\064\127\112\005\148\191\065\063\087\255\000\037\199\106\135\193\027\142\071\212\199\043\251\142\217\019\225\133\060\061\009\061\228\223\012\111\227\136\159\172\159\143\014\203\126\028\126\163\120\002\190\011\164\127\184\044\249\179\018\231\087\213\063\004\055\171\211\117\212\216\094\210\149\158\194\005\236\123\161\206\047\155\168\068\226\115\148\046\099\016\104\239\169\095\026\025\028\095\114\207\155\223\137\061\223\197\100\028\146\065\224\120\002\028\218\030\060\250\151\015\019\145\005\253\018\025\011\186\203\199\037\233\130\242\226\163\157\100\026\078\189\179\028\083\234\114\057\018\215\153\143\220\029\138\083\210\233\146\033\100\181\234\067\222\252\229\199\238\010\227\006\061\089\179\190\033\089\111\084\011\135\025\115\168\174\096\210\227\205\195\238\031\018\179\187\126\219\035\150\066\016\060\088\023\159\152\206\125\069\130\119\220\090\185\057\091\175\130\109\078\106\124\006\018\173\091\060\055\070\032\199\076\180\080\157\152\071\011\195\198\067\025\249\064\159\130\223\013\077\161\160\112\056\147\178\243\106\196\247\059\163\038\002\189\051\015\216\198\028\061\060\089\056\057\114\199\158\131\130\231\007\230\063\143\030\088\245\156\181\125\173\195\168\043\135\044\233\136\241\204\015\145\236\099\099\016\021\046\184\015\108\052\055\177\108\158\007\028\071\216\180\231\103\093\248\207\031\167\060\004\232\183\101\195\238\011\006\190\230\130\101\206\077\121\167\233\177\112\157\011\050\134\244\100\230\201\030\008\154\001\212\027\009\067\233\176\024\222\061\066\071\194\228\206\251\230\193\164\241\159\173\198\155\011\067\250\187\209\233\168\127\160\134\000\120\140\206\123\148\059\154\026\062\178\224\159\049\239\238\073\039\150\203\047\142\238\005\182\075\238\127\023\037\166\079\192\065\183\033\048\060\105\050\251\100\026\013\187\194\229\132\126\036\135\118\239\228\234\248\081\125\101\202\171\240\098\076\113\201\035\096\253\090\007\168\229\248\062\229\210\092\074\228\188\201\097\144\130\099\249\214\015\089\135\037\179\111\134\227\009\169\241\031\137\105\244\216\224\036\198\015\124\092\074\065\237\073\236\251\243\207\175\011\017\234\215\221\028\091\171\220\105\108\214\220\089\207\117\171\203\239\198\044\240\028\120\189\196\220\180\019\222\079\039\108\065\215\082\087\125\248\060\062\007\006\115\180\244\126\160\251\110\111\071\161\151\022\233\178\001\192\001\058\092\249\101\150\089\231\046\071\139\136\194\074\192\022\153\018\146\172\184\242\239\185\159\026\186\100\206\038\049\039\195\037\018\028\129\145\223\017\238\211\207\003\118\015\158\229\190\225\233\101\135\147\198\196\129\161\129\019\213\087\168\010\124\067\153\192\159\120\135\119\207\109\182\247\162\198\023\046\075\051\078\046\165\155\114\094\237\241\134\039\241\124\146\198\073\129\101\095\009\076\075\056\241\221\240\249\131\089\195\005\226\029\022\110\250\130\236\115\042\007\046\079\019\169\213\139\191\002\055\007\000\154\095\160\125\070\141\130\035\225\176\219\009\105\204\024\065\150\076\224\241\126\130\090\106\112\191\043\184\117\087\230\063\006\089\114\223\237\062\015\169\098\253\205\167\127\127\224\101\156\089\201\100\099\180\200\045\073\046\131\216\092\067\139\182\125\045\252\057\161\124\139\178\083\243\150\187\076\230\207\108\065\031\127\131\102\241\112\238\189\087\232\034\050\031\037\221\247\117\109\047\063\214\045\159\025\231\127\006\253\203\211\175\113\128\003\214\087\038\093\013\025\251\086\193\164\144\178\193\158\030\055\023\025\059\007\124\177\014\210\117\115\004\125\092\160\206\091\051\201\199\166\055\174\062\051\066\004\227\221\160\020\150\062\238\069\243\039\055\151\187\034\076\102\079\019\063\126\022\094\027\166\219\036\249\207\029\165\253\114\204\119\155\059\113\156\121\118\139\105\009\207\048\112\028\197\208\139\050\254\209\243\057\147\182\193\135\212\001\100\252\067\225\183\190\235\218\250\079\020\215\136\197\147\113\034\009\000\120\119\063\087\216\249\091\191\222\075\062\105\247\073\209\191\134\221\223\183\081\195\112\169\127\188\031\245\176\095\021\195\198\229\182\248\253\043\155\124\250\150\151\177\054\056\071\170\223\202\063\177\144\199\019\215\080\205\203\214\048\099\251\241\182\219\227\247\032\095\184\009\197\045\047\160\038\047\192\219\225\235\198\219\227\109\183\193\016\252\014\036\112\192\226\217\195\112\143\201\057\036\150\065\100\144\206\166\152\171\098\049\158\095\149\205\113\030\178\002\025\250\054\134\003\013\177\039\226\073\161\191\012\215\119\232\096\224\226\253\210\207\231\220\199\247\117\211\209\041\108\005\180\150\094\072\193\226\208\252\251\226\225\004\035\221\255\000\008\087\137\124\111\097\166\193\146\113\105\010\025\199\055\020\222\144\201\052\178\059\049\230\209\225\044\164\058\045\156\004\078\025\227\146\152\153\248\054\017\224\024\124\145\224\213\183\206\164\031\203\197\138\249\047\086\121\200\234\192\111\119\084\243\244\069\084\014\255\000\019\095\143\092\027\124\022\248\249\215\019\102\253\049\097\101\190\057\159\033\240\103\157\182\219\219\105\031\008\243\125\248\117\012\127\171\127\185\168\176\168\246\091\248\051\228\124\221\050\046\009\241\101\243\152\181\111\157\182\228\055\064\070\025\210\015\074\058\185\182\219\109\240\128\239\187\228\099\150\225\064\028\163\169\242\184\029\034\076\207\012\183\188\131\249\046\123\230\250\133\169\248\054\097\208\231\044\186\142\224\222\177\050\245\209\064\247\071\221\149\183\125\074\000\142\096\006\013\120\133\209\191\018\100\178\226\112\099\250\149\106\243\050\219\110\018\110\026\153\127\034\231\143\153\240\006\059\015\016\059\196\010\071\206\233\226\218\030\072\117\105\241\126\139\129\113\240\060\178\126\016\180\026\219\112\021\229\008\199\125\056\230\003\168\253\066\110\132\011\148\109\223\203\237\109\175\215\008\088\095\146\181\126\197\002\075\120\182\223\005\234\251\017\087\009\126\196\178\123\033\171\245\049\251\049\229\251\182\219\097\240\102\131\244\169\252\220\105\123\168\182\223\057\027\232\189\227\142\071\144\178\028\241\182\148\251\047\129\200\125\254\120\109\182\207\083\192\223\077\165\112\063\171\236\180\223\118\219\109\176\206\150\080\221\162\136\039\006\025\200\102\248\135\214\061\034\044\240\059\101\150\231\077\243\179\142\009\131\025\096\239\133\189\073\234\098\079\131\197\253\185\105\188\199\060\039\000\244\251\128\078\121\099\161\072\100\121\245\006\049\028\227\117\204\115\150\228\192\194\010\243\177\041\114\093\148\179\137\124\044\207\018\248\226\167\171\224\179\240\254\209\116\253\216\093\120\024\080\142\236\222\102\225\114\199\140\211\196\140\028\095\242\064\229\186\231\058\137\218\115\161\025\075\120\081\134\165\218\114\112\124\063\083\141\175\238\117\038\249\158\030\039\168\009\113\248\007\127\070\172\034\028\253\091\012\095\050\024\254\040\184\182\223\004\110\135\221\158\158\107\136\150\015\182\232\061\100\116\102\031\062\224\143\187\109\182\217\097\159\230\135\003\226\021\244\017\155\049\087\238\028\182\223\015\197\245\173\062\195\143\155\209\035\241\016\245\250\201\253\122\172\125\211\196\054\243\225\120\178\015\221\151\200\143\137\152\254\163\244\123\068\248\230\233\149\203\188\242\015\129\136\228\153\115\115\014\027\239\196\218\090\091\044\175\129\008\090\070\117\174\095\222\066\005\025\046\030\012\252\093\096\253\041\190\246\110\219\120\012\194\049\154\018\182\078\049\158\188\242\226\064\148\236\133\158\067\112\154\142\108\035\155\225\158\227\220\060\165\180\157\075\115\203\182\205\179\204\248\199\019\231\103\202\193\244\200\095\212\023\120\008\044\130\053\014\032\178\200\032\240\049\029\056\139\200\023\180\206\108\013\124\000\019\126\039\189\087\171\146\061\249\135\096\127\081\125\156\247\103\190\056\241\115\049\184\176\245\227\174\237\007\197\127\164\161\133\089\151\230\195\007\197\046\109\109\109\150\143\221\247\213\156\077\251\197\145\124\145\196\075\015\228\142\098\231\212\111\157\184\154\025\126\032\121\250\044\254\083\248\172\177\075\063\179\084\058\223\068\232\199\185\110\248\050\228\090\047\138\220\134\088\095\011\018\223\219\035\224\151\047\184\048\252\208\182\190\054\247\112\181\171\065\227\035\076\109\135\099\040\190\225\174\225\205\164\165\218\088\222\203\132\224\126\165\009\238\077\142\012\125\137\243\051\217\032\047\096\005\177\171\227\023\255\000\191\039\051\132\181\114\095\146\077\202\025\012\238\118\062\252\206\022\170\249\214\037\001\197\172\191\050\248\022\247\109\182\203\105\044\055\065\179\249\240\190\194\004\248\030\008\226\080\241\111\016\179\143\017\201\226\102\068\169\103\217\020\231\157\176\068\019\140\204\117\187\194\239\139\166\207\143\031\106\192\194\219\165\170\214\016\089\231\201\241\153\038\062\194\121\225\137\158\085\067\227\109\142\135\221\202\204\128\115\111\032\251\074\016\116\245\012\219\236\034\238\134\025\101\151\139\151\217\092\068\044\063\075\236\070\219\109\182\082\082\251\115\064\191\103\075\171\245\183\245\208\213\250\177\207\210\221\127\086\199\133\241\151\240\109\129\182\225\242\248\077\097\182\092\152\185\240\048\051\159\184\051\236\216\063\047\093\159\080\001\058\182\217\023\215\103\054\040\244\240\046\047\220\179\236\199\181\022\076\179\201\051\224\054\093\077\118\204\254\242\063\016\057\207\197\019\071\080\247\005\111\007\001\188\127\213\211\181\113\158\142\172\062\252\014\214\099\169\124\075\173\188\219\108\219\108\113\207\133\124\030\055\201\126\202\069\072\120\168\218\068\012\099\169\235\195\120\137\224\126\035\251\205\099\071\094\065\129\072\047\068\003\223\127\115\031\051\049\059\155\230\201\154\166\039\086\001\241\001\029\216\120\112\215\242\228\240\062\105\116\125\067\057\219\245\135\155\075\109\061\218\064\127\109\199\252\027\098\049\203\088\214\242\201\239\100\251\025\234\140\203\109\150\219\152\124\084\051\079\171\246\091\101\109\182\219\205\192\065\227\062\015\155\151\168\195\223\136\197\127\165\244\099\110\113\241\111\147\101\191\069\149\112\095\052\202\192\120\241\102\195\201\101\243\161\056\092\151\210\102\233\216\210\243\236\153\169\190\210\041\117\231\201\179\196\160\098\234\126\221\249\067\112\251\135\190\135\253\117\180\203\199\140\219\056\147\152\113\196\092\121\092\216\008\183\186\084\190\131\098\110\047\155\136\106\189\198\001\253\211\215\111\204\101\022\055\027\220\170\205\044\063\191\012\204\220\184\153\129\146\208\151\202\240\218\031\195\245\024\079\249\002\059\148\060\078\024\103\008\248\021\241\201\070\125\202\254\128\236\064\072\055\003\187\095\185\068\167\139\078\236\215\238\202\207\014\236\249\164\203\060\112\157\146\122\091\068\252\199\207\245\014\220\177\031\083\109\182\219\098\021\244\221\081\221\066\251\079\156\179\031\076\062\054\101\000\122\097\205\107\013\178\219\114\127\088\078\078\191\080\254\189\176\219\108\182\217\007\203\079\024\224\252\199\049\044\250\152\169\063\106\236\249\219\109\191\170\203\135\234\031\183\092\007\129\253\248\219\101\067\171\179\194\118\110\034\119\079\031\004\124\195\109\218\112\196\022\192\234\124\104\224\205\085\218\204\239\065\105\183\184\207\235\041\228\253\171\056\146\203\215\129\226\210\119\061\025\019\126\033\150\116\057\109\028\202\035\249\167\117\123\216\106\237\008\156\078\077\014\102\117\219\082\103\112\182\103\142\222\018\024\241\027\179\018\064\147\249\111\148\039\226\200\251\206\101\182\203\152\134\044\100\107\169\006\030\007\192\103\116\200\083\115\071\098\050\111\068\247\108\235\230\092\175\153\095\048\172\107\199\031\016\003\236\051\031\143\015\062\035\031\221\115\138\231\250\150\216\176\254\160\237\182\219\109\190\207\029\248\108\207\133\191\030\055\038\114\182\124\254\179\133\123\063\083\207\213\224\173\182\219\127\251\044\047\075\120\155\157\063\232\065\159\182\118\092\031\220\092\167\141\242\046\037\191\170\046\031\168\104\215\009\139\013\182\248\017\120\047\030\027\043\005\225\220\153\081\236\046\119\062\148\097\183\147\137\067\136\197\187\238\097\103\183\134\217\123\076\131\194\089\225\046\008\246\178\204\193\100\188\247\086\215\197\170\090\028\095\114\171\014\004\095\180\147\032\135\153\011\230\196\130\067\028\234\121\243\183\245\060\062\070\227\207\226\086\175\194\224\067\012\066\238\221\047\189\197\020\118\082\233\003\225\250\172\167\090\219\048\146\217\025\115\100\245\073\055\130\128\222\133\173\200\226\078\124\152\097\251\098\255\000\100\179\250\092\164\193\250\160\111\134\219\109\182\219\228\223\028\249\017\019\101\254\143\026\141\126\165\191\170\027\062\024\180\075\121\231\230\227\251\098\225\013\253\147\003\244\092\063\100\231\126\225\254\232\185\062\024\179\098\103\250\025\209\124\220\170\121\019\109\044\088\177\031\128\235\199\091\114\025\111\092\198\022\243\151\232\155\236\059\006\049\228\230\030\117\175\036\056\054\058\254\204\055\198\073\050\089\182\237\187\188\151\123\168\016\123\135\038\067\058\205\034\246\107\190\239\089\050\102\121\178\231\037\144\120\001\187\062\127\160\254\003\193\113\219\131\100\227\130\223\014\016\164\216\138\189\048\202\024\124\060\223\076\104\116\013\167\062\068\131\168\195\153\220\232\132\250\108\246\055\008\034\218\189\189\216\153\038\201\158\023\023\253\254\085\044\179\254\104\238\219\109\182\219\109\182\219\187\027\143\143\027\045\238\094\039\254\153\108\023\047\213\250\149\176\220\092\091\150\187\191\115\111\138\213\198\230\254\208\255\000\204\121\126\231\202\203\231\051\145\131\105\113\063\082\223\182\145\112\159\142\059\238\223\027\109\182\219\109\150\219\044\057\207\087\253\198\152\152\079\080\148\017\184\088\042\062\014\238\108\000\237\243\045\004\222\121\089\230\208\249\111\244\136\229\060\064\049\123\033\144\055\150\001\227\015\238\198\195\203\011\036\185\056\132\075\058\150\217\147\100\182\109\143\227\217\117\204\127\169\055\070\064\071\013\158\195\225\155\046\060\004\060\003\176\179\083\230\024\067\067\171\129\088\045\078\103\133\244\198\029\203\031\080\113\096\179\041\158\012\126\216\191\219\062\127\114\169\254\166\030\109\251\149\245\043\234\212\252\022\219\225\175\134\190\077\207\005\207\233\020\077\140\009\182\195\108\178\233\038\118\234\006\110\095\017\157\045\126\227\086\043\236\070\114\109\134\219\103\155\012\220\089\228\034\069\225\059\225\182\254\003\124\007\124\237\194\201\063\178\083\193\206\228\074\079\100\091\094\028\176\062\150\094\165\182\089\112\177\087\093\201\225\043\243\024\085\223\057\015\136\007\108\227\190\058\088\243\206\182\090\076\023\185\243\156\195\038\019\196\166\201\044\139\047\108\255\000\039\178\114\093\074\058\148\199\054\000\183\196\240\036\060\013\033\216\233\050\217\204\187\063\025\249\201\145\012\186\185\027\027\173\213\159\251\239\250\225\201\250\156\231\255\000\009\202\217\121\183\226\251\091\107\189\091\245\115\240\192\190\155\095\023\063\012\168\227\248\119\192\187\209\072\060\126\158\013\013\182\248\044\183\204\194\046\159\023\247\072\225\062\096\011\037\202\053\117\157\139\057\182\219\109\135\080\249\111\154\024\097\119\124\019\006\219\109\156\126\228\061\247\185\039\165\175\164\167\004\038\254\193\028\113\255\000\045\150\223\027\006\185\007\057\156\041\156\164\201\013\240\088\225\136\175\181\150\219\045\214\230\156\212\048\223\086\076\020\140\239\061\223\003\058\109\047\235\096\154\011\240\049\199\129\030\050\124\055\101\150\082\172\228\200\193\151\244\159\196\120\044\207\134\123\215\185\089\091\225\111\198\090\151\007\212\067\013\149\201\097\238\014\025\099\122\177\131\039\133\165\182\248\045\036\245\116\186\050\253\038\097\191\053\156\143\139\233\235\108\191\052\057\205\178\195\051\046\221\109\193\002\030\237\244\198\109\111\099\135\190\015\025\123\200\053\206\011\109\150\219\108\018\180\047\184\028\126\150\191\032\185\135\195\054\219\109\240\127\047\071\138\226\107\039\202\089\167\193\150\066\251\069\237\116\047\069\238\214\250\182\219\109\143\247\109\117\116\067\137\254\140\108\126\107\121\181\223\013\181\009\234\029\174\128\217\159\172\190\193\152\124\031\134\010\016\059\065\108\081\249\183\101\231\195\109\163\240\171\038\057\056\110\097\251\157\115\245\176\116\129\132\101\109\147\099\171\007\216\032\061\192\120\180\055\122\181\056\231\195\020\032\187\200\220\039\096\118\235\198\203\044\187\032\153\202\036\144\091\051\213\125\091\023\231\249\112\159\139\063\193\118\032\015\012\050\236\178\208\134\051\026\148\059\108\018\241\059\091\124\044\178\207\062\029\039\188\061\051\172\175\245\195\167\226\075\085\013\182\248\217\114\063\048\228\116\028\067\158\150\071\161\063\014\011\210\224\032\034\202\219\109\182\089\041\167\251\176\056\181\247\131\028\159\046\195\109\182\248\029\157\012\184\191\147\070\015\165\184\193\213\185\059\176\018\170\103\056\112\115\198\219\109\182\039\199\043\099\122\065\128\120\133\174\183\045\241\179\062\234\016\177\238\061\044\254\172\221\175\159\250\109\175\237\015\147\110\011\125\177\070\097\124\095\025\023\200\229\044\183\194\203\004\245\188\194\061\070\092\188\054\239\049\005\100\209\108\182\206\167\197\220\025\204\187\109\179\114\075\190\005\223\024\088\048\064\095\184\255\000\049\108\124\228\155\117\013\197\224\045\190\064\097\134\113\008\060\202\164\184\218\094\231\073\101\150\089\230\050\223\166\015\245\085\175\245\047\211\105\181\065\226\219\109\134\195\063\119\214\112\208\200\111\168\089\020\182\110\056\045\032\109\182\089\120\150\170\228\255\000\052\116\180\251\130\195\039\161\052\245\241\107\109\182\187\032\123\028\216\152\199\072\170\008\248\144\250\089\176\005\024\004\213\014\045\054\027\202\247\109\182\219\096\108\244\056\051\134\230\189\160\194\225\082\209\067\190\054\223\007\245\111\019\247\016\111\002\232\230\054\060\157\094\158\029\196\143\144\182\219\109\150\090\253\220\226\011\227\190\160\226\253\035\192\207\092\201\255\000\217\189\080\223\139\085\161\118\047\087\027\092\100\224\120\001\133\147\111\055\022\203\039\155\153\111\115\048\036\143\022\114\250\182\062\015\231\201\248\178\226\114\036\182\207\143\005\205\188\195\040\216\033\143\002\108\083\149\176\192\092\121\134\228\088\238\219\124\030\173\226\233\110\095\015\054\223\163\008\199\095\185\168\239\136\060\091\109\182\219\199\247\115\191\029\141\071\234\116\046\201\186\216\009\240\228\211\226\218\126\203\109\182\217\111\086\087\053\251\129\199\238\095\116\159\061\057\110\037\191\054\219\109\175\169\142\058\123\047\094\126\210\004\217\123\128\046\182\011\244\029\022\126\011\046\153\223\055\198\211\204\182\223\109\234\093\125\054\215\224\131\150\219\224\186\049\199\241\014\175\220\011\190\238\029\204\088\095\076\038\010\194\224\134\101\109\182\203\005\195\229\156\233\203\217\001\200\000\118\081\192\173\254\191\017\036\073\230\189\207\111\139\186\097\034\054\024\022\241\229\093\152\114\210\126\230\034\095\011\224\223\020\094\123\185\127\127\192\254\068\092\023\105\038\119\238\030\101\197\176\248\035\171\209\108\054\195\013\155\035\075\201\045\228\030\209\200\206\123\038\134\099\157\198\192\103\099\114\145\125\203\178\199\087\089\233\122\027\097\240\197\250\023\189\242\076\135\215\221\182\219\108\158\114\249\022\017\240\125\071\067\234\096\210\077\241\015\127\100\223\015\133\180\223\037\200\254\237\140\176\045\131\166\144\158\131\178\223\238\219\124\046\093\216\114\122\184\051\082\047\126\054\219\109\182\245\190\198\107\039\166\004\039\171\165\250\018\014\158\022\217\096\214\004\131\098\110\099\186\055\088\203\157\012\248\046\165\249\217\126\092\061\007\141\182\219\210\207\249\053\128\176\137\033\078\238\244\140\069\193\071\220\105\013\067\175\013\202\193\218\030\024\070\015\137\187\227\248\077\219\100\022\146\243\108\248\088\035\171\189\245\163\198\127\062\111\195\045\105\007\049\142\044\236\048\195\012\054\240\219\013\218\113\047\109\147\021\022\027\111\230\027\221\195\076\180\160\170\012\114\001\129\217\159\029\118\205\249\077\191\163\091\007\197\202\125\113\103\047\238\075\060\060\095\075\003\244\174\058\125\219\031\114\031\216\157\044\062\196\181\062\231\206\231\050\237\161\246\207\172\224\047\145\017\208\151\227\246\093\012\060\201\134\195\191\134\195\109\182\231\142\219\048\085\226\116\244\017\057\155\141\134\156\001\125\072\202\127\073\006\031\015\132\120\083\245\115\101\252\078\118\201\130\134\027\109\150\095\094\216\151\232\226\216\091\202\216\226\046\156\230\116\003\090\140\218\198\067\212\176\158\121\135\174\254\224\029\100\187\211\142\125\094\071\063\006\057\024\229\027\147\029\138\107\051\097\151\063\003\122\075\178\249\077\130\196\145\095\118\213\249\254\108\252\025\127\036\153\043\103\204\117\017\013\240\135\155\066\223\004\248\111\094\057\048\007\014\131\008\181\217\116\093\179\135\213\235\006\232\232\209\140\053\250\180\122\155\056\226\124\075\062\117\205\210\223\189\058\228\250\226\215\236\077\226\219\101\143\101\206\103\016\165\159\214\127\238\157\089\255\000\182\046\092\251\182\223\013\187\125\134\156\186\248\177\127\123\096\059\204\209\030\050\224\115\235\072\065\203\205\196\223\137\109\243\190\054\217\109\157\036\124\198\188\001\166\206\067\120\232\219\058\237\230\251\020\123\046\107\192\192\188\253\049\109\190\119\212\101\176\219\111\145\222\061\200\120\006\155\057\088\224\112\039\046\239\107\182\120\246\048\220\212\126\037\199\035\126\111\086\079\153\171\185\126\035\185\045\248\135\220\241\208\207\102\062\216\026\153\240\088\139\129\219\035\206\097\083\183\162\089\108\185\110\211\108\199\108\241\178\091\158\119\032\141\249\187\175\083\252\047\227\190\118\046\110\165\199\030\226\236\058\189\017\224\109\186\101\014\248\038\203\109\247\139\119\174\205\185\147\162\194\007\041\056\000\121\145\159\046\009\244\109\015\135\111\181\251\145\144\193\249\033\125\172\246\137\087\137\241\193\116\190\023\086\203\034\242\092\143\137\224\223\118\007\235\003\247\049\240\144\062\194\195\185\216\249\217\121\151\180\230\083\155\217\112\047\176\141\179\084\127\073\228\176\199\151\004\023\211\217\191\235\009\026\182\219\109\182\219\103\194\142\155\183\118\057\136\077\134\104\189\030\161\029\068\209\248\056\150\253\132\120\075\045\253\017\051\031\048\120\243\182\219\227\124\109\182\252\064\174\029\252\091\164\003\066\114\135\232\018\135\161\213\207\245\183\025\247\176\193\236\240\227\026\121\058\182\188\132\231\082\094\003\251\176\242\200\014\056\196\094\068\168\103\168\229\203\117\015\018\218\195\109\163\013\180\180\182\124\024\030\237\015\244\182\047\207\140\254\007\242\223\035\151\245\087\039\152\004\156\121\045\241\189\149\166\199\049\113\185\155\023\066\231\074\086\212\114\104\202\025\062\227\094\108\130\203\056\233\201\155\199\132\186\079\247\156\135\007\212\127\107\016\127\124\220\238\230\195\022\103\204\198\012\075\126\237\132\186\182\030\219\072\245\203\046\124\253\223\101\022\161\249\154\035\230\102\231\057\214\003\228\249\032\188\076\109\183\030\057\152\225\126\231\168\089\193\108\151\151\148\134\143\220\178\062\184\034\088\253\064\095\165\034\120\201\081\204\061\049\162\215\168\200\014\123\037\057\012\180\191\127\131\229\046\119\055\133\091\130\011\117\065\245\050\035\174\031\101\198\255\000\237\250\095\121\043\057\199\041\122\017\068\113\073\055\028\212\119\067\142\103\071\031\199\109\182\219\234\053\227\183\226\231\007\216\198\188\002\057\206\244\075\103\001\009\229\056\046\024\232\176\251\072\009\220\096\218\167\196\027\222\038\104\204\067\155\041\020\054\008\248\101\002\251\024\052\046\176\139\109\226\221\182\222\044\015\007\201\022\219\097\078\101\254\164\125\112\079\241\063\150\126\031\166\096\238\120\060\033\219\124\251\134\198\029\060\030\206\115\221\139\076\222\236\072\002\060\137\025\124\031\013\242\175\102\220\192\056\097\022\223\059\111\002\230\198\111\184\084\003\236\005\106\127\104\015\220\238\073\189\169\000\202\118\078\056\191\100\023\179\226\067\214\125\165\196\112\191\108\064\228\114\096\152\022\087\183\187\055\225\110\035\243\101\250\123\177\117\252\032\240\023\230\223\186\236\242\253\206\004\039\196\252\215\208\135\059\020\133\248\184\139\236\027\183\054\112\135\017\242\068\238\239\124\099\155\196\206\071\246\049\147\016\049\120\210\116\179\248\099\216\084\035\192\125\137\246\164\122\207\220\099\147\103\191\071\162\020\226\061\221\250\190\017\208\057\078\250\182\062\179\152\024\057\079\048\007\028\228\140\253\236\241\254\233\197\244\243\030\109\123\172\012\235\029\155\251\130\233\063\072\135\188\252\251\238\066\123\195\246\007\238\024\152\029\034\248\038\153\187\210\219\159\093\105\142\026\197\207\137\171\240\124\204\122\117\219\027\215\155\152\124\046\080\044\236\157\167\143\152\004\013\253\072\028\143\238\009\164\130\119\191\168\092\159\023\220\012\043\196\028\073\054\248\229\001\196\234\033\242\060\120\024\229\158\163\251\230\237\254\055\248\031\010\228\047\055\081\242\060\155\188\090\201\212\237\032\063\114\048\136\207\136\151\193\144\143\097\247\025\133\252\179\004\051\221\153\015\087\082\203\028\220\187\038\142\055\001\157\078\237\067\224\068\112\228\125\218\005\062\096\224\024\158\203\246\011\224\047\105\175\213\192\101\188\043\034\175\037\117\102\047\123\209\039\108\143\038\049\254\188\102\094\046\105\126\174\007\127\201\015\237\037\210\129\066\251\090\117\040\143\080\119\106\206\142\139\069\103\124\017\230\156\202\163\207\185\078\088\193\255\000\081\144\058\211\234\091\080\221\090\124\222\211\083\113\198\078\208\226\125\137\110\204\189\207\136\067\221\241\111\153\062\100\245\238\217\026\241\242\134\247\123\097\095\148\217\130\213\016\245\197\194\014\061\218\125\076\043\123\030\173\007\051\226\246\056\087\056\147\063\120\047\156\124\027\232\204\119\116\149\238\254\231\103\165\167\094\126\110\210\199\209\107\153\174\193\244\254\167\155\096\028\173\182\141\114\063\133\098\115\017\151\097\118\115\033\079\196\234\071\192\220\216\065\195\250\022\154\048\229\142\097\207\011\196\079\141\156\144\236\119\030\024\135\060\063\254\203\244\199\241\030\031\203\127\002\192\108\167\231\240\045\240\060\067\111\138\228\034\115\029\181\009\123\108\120\070\091\241\109\245\036\123\251\046\216\003\072\078\157\241\063\194\188\088\059\254\208\057\033\031\082\027\184\191\072\030\239\130\067\212\143\089\240\018\155\253\009\180\120\023\019\116\132\196\225\128\056\222\182\030\209\055\100\079\046\163\235\063\056\195\238\066\210\083\209\027\156\043\098\223\187\013\157\070\068\024\032\224\131\137\173\199\245\041\103\076\094\184\125\193\240\233\190\022\253\150\063\083\214\088\247\019\238\120\018\247\147\251\138\229\241\195\229\189\160\030\139\079\215\198\068\010\183\095\223\132\201\192\156\177\015\003\163\013\222\163\014\188\241\097\241\245\183\162\134\197\235\027\126\235\224\124\058\118\126\038\095\220\189\187\047\158\127\115\251\155\164\158\156\007\162\027\063\083\027\229\221\146\233\245\014\028\230\211\223\079\113\041\186\053\097\211\005\188\198\181\040\216\156\184\187\214\234\022\072\243\075\213\180\218\174\227\205\182\219\225\110\050\203\169\118\034\037\182\219\097\171\140\222\082\255\000\035\252\099\013\057\227\097\242\068\188\070\017\091\234\205\044\179\134\198\001\109\178\219\119\024\008\226\246\245\226\196\132\195\129\060\067\178\090\178\041\200\009\061\196\034\014\155\225\079\130\117\202\199\076\226\050\255\000\100\069\241\240\093\124\058\195\096\113\217\062\073\124\211\247\116\048\207\164\023\199\063\083\167\151\099\156\033\024\227\151\118\124\112\227\011\095\060\167\001\184\123\206\236\198\236\028\184\054\167\028\017\251\233\116\255\000\237\122\099\247\033\200\193\248\158\002\031\156\245\084\199\164\253\223\254\142\074\190\228\120\056\194\078\057\247\097\096\123\182\025\030\226\235\211\212\009\237\057\008\003\167\089\220\057\141\223\144\231\012\236\197\095\014\253\202\244\045\061\108\175\031\073\123\055\250\157\032\199\173\031\187\220\097\240\073\217\129\234\045\135\212\178\176\145\048\039\084\125\233\232\181\230\062\226\153\062\248\096\141\111\102\013\068\023\112\043\028\208\246\140\054\140\128\057\182\091\109\230\118\200\200\118\098\046\211\107\109\247\004\179\095\157\231\252\060\243\158\011\065\126\226\013\159\136\060\153\113\055\078\188\058\067\025\182\107\227\124\115\022\089\058\135\121\190\090\049\039\096\195\220\009\214\063\050\047\107\177\069\215\164\191\220\011\243\055\201\054\049\017\192\034\076\190\002\071\086\158\002\126\094\181\027\124\235\213\158\224\005\220\230\121\163\152\232\043\011\144\089\099\254\177\029\090\104\227\170\115\138\007\130\026\191\174\247\029\243\007\234\095\081\252\200\157\206\175\089\161\241\061\225\046\153\121\203\118\214\094\123\145\240\059\122\222\033\051\077\176\250\054\055\111\198\055\133\014\143\132\190\034\154\188\057\032\040\144\103\078\126\109\254\197\220\133\191\234\193\106\215\171\089\101\181\251\142\136\222\204\126\239\137\193\031\244\222\131\003\220\034\135\163\038\198\125\017\121\035\150\059\114\090\088\019\124\210\113\113\185\213\222\052\233\097\078\047\169\223\061\216\079\121\252\078\243\227\054\005\235\213\221\175\228\063\131\124\031\128\229\162\033\202\078\110\188\019\146\228\183\113\196\249\185\091\107\100\185\142\045\097\219\096\226\249\090\221\156\090\228\107\146\056\065\166\167\028\159\036\215\210\248\153\207\211\195\129\094\008\200\161\243\028\161\028\053\204\117\115\078\056\105\237\142\219\255\000\036\061\095\033\189\151\187\101\157\048\103\086\222\161\193\063\220\052\013\234\166\236\085\220\142\111\136\030\016\058\183\161\019\143\183\214\101\255\000\022\193\015\132\250\133\220\036\248\196\167\217\115\112\183\218\111\144\196\232\150\088\228\199\114\141\200\195\126\134\237\150\079\155\072\225\055\178\017\183\228\230\001\162\251\204\061\105\245\014\175\134\248\104\249\154\028\015\204\012\111\180\031\083\034\115\218\071\184\014\246\000\068\253\022\206\059\122\032\063\219\219\099\224\240\151\060\067\112\185\221\070\069\056\241\109\190\080\119\105\036\120\203\157\135\027\109\151\048\104\030\089\110\243\223\243\063\192\126\037\139\241\100\059\245\231\101\106\249\033\199\192\060\221\228\226\075\044\130\003\096\044\245\028\141\144\045\028\251\249\044\157\016\244\193\127\242\066\246\050\019\218\075\247\016\135\247\183\119\250\070\135\220\107\190\034\182\044\246\071\243\038\073\133\233\177\252\208\122\011\208\015\212\158\183\237\063\124\004\036\006\192\185\015\113\103\086\026\135\048\038\164\126\172\227\112\244\202\205\143\213\153\202\047\077\128\122\048\143\072\222\151\218\126\174\021\049\186\197\140\125\087\113\126\211\002\024\132\079\014\220\072\244\225\249\178\253\067\024\111\149\242\093\017\031\184\245\150\210\026\193\122\102\020\249\220\189\193\124\019\185\023\221\161\128\253\068\053\035\093\111\191\193\153\100\134\009\204\241\013\240\126\208\032\252\027\046\096\217\241\182\225\248\001\174\110\175\195\169\145\183\252\099\242\208\124\196\001\238\103\091\044\190\188\030\071\055\006\228\201\243\183\079\012\238\193\124\061\092\076\147\163\195\244\205\068\150\115\019\129\058\032\217\135\210\193\207\205\183\079\113\142\122\128\145\152\152\216\192\121\115\193\054\078\120\090\178\205\178\061\092\037\206\020\250\154\229\105\096\184\062\227\185\239\135\033\043\238\222\003\213\145\099\175\015\051\021\152\060\117\048\015\075\091\215\016\204\060\244\158\211\044\157\251\066\124\216\154\253\205\237\023\156\102\049\229\149\190\054\217\215\141\155\101\182\026\237\185\047\129\061\074\082\241\014\027\213\211\058\252\115\252\157\176\024\065\151\007\143\118\248\013\179\011\097\135\033\144\120\180\034\057\226\210\225\112\150\220\135\155\139\148\198\230\124\034\147\134\194\069\172\051\245\002\015\082\093\113\099\112\113\031\034\061\055\036\028\176\060\013\182\248\108\179\212\194\073\244\141\228\180\067\158\201\254\176\025\012\255\000\070\114\165\240\036\012\092\175\184\074\037\151\060\004\182\062\109\101\170\089\234\091\180\249\250\189\134\129\057\152\199\206\012\243\153\108\182\219\225\152\182\219\229\124\050\079\150\056\097\141\070\110\055\185\127\132\255\000\041\123\078\035\135\225\178\116\109\183\193\115\027\110\088\206\145\156\068\178\089\182\093\151\225\150\078\089\014\033\222\248\086\092\073\159\187\223\247\007\062\236\123\203\083\085\161\045\178\183\240\027\111\130\248\075\073\135\103\031\255\000\035\239\091\095\127\220\183\212\184\134\027\121\241\182\204\032\217\197\133\193\126\151\180\067\207\095\128\109\148\140\110\060\061\075\037\183\192\101\223\200\182\239\150\127\002\216\135\230\103\182\255\000\128\254\057\248\101\150\126\025\028\089\119\213\166\062\231\132\124\236\184\035\152\238\203\146\221\176\137\006\216\127\003\117\029\079\228\156\070\075\087\005\171\017\114\197\198\234\226\060\142\120\253\045\242\013\137\124\158\045\173\152\196\120\153\150\195\014\203\109\177\108\068\118\196\234\111\082\011\089\229\043\185\099\109\139\185\075\195\059\033\251\157\115\238\076\179\060\050\203\248\191\031\129\101\217\031\127\194\127\011\254\009\155\092\031\003\171\032\044\008\008\225\012\060\013\148\203\072\109\182\215\230\214\063\049\103\204\089\212\049\241\067\234\028\245\033\019\103\203\225\243\146\241\015\022\190\055\156\039\168\239\193\107\045\249\240\217\092\210\155\105\104\120\107\109\177\054\217\109\097\230\032\185\123\163\070\119\101\101\043\041\045\183\137\133\182\236\108\124\016\124\064\200\122\153\140\022\033\199\169\118\092\100\077\095\015\131\187\187\191\147\226\000\213\191\171\254\027\252\199\225\168\140\158\242\099\158\058\240\216\170\024\099\030\050\239\228\030\007\215\225\159\130\105\100\204\054\207\141\182\217\099\176\241\182\203\224\177\201\225\182\231\086\153\182\217\113\226\219\155\091\097\182\219\109\182\029\240\016\131\033\003\193\124\011\204\181\153\124\062\012\036\207\029\045\162\073\189\091\111\050\158\017\205\145\062\014\227\187\191\135\171\123\061\094\191\082\249\060\111\141\254\071\248\031\226\217\023\016\131\211\103\147\205\215\050\216\240\078\194\051\019\118\031\019\149\188\203\225\178\249\142\022\025\124\139\045\182\030\007\037\199\130\219\119\025\050\219\124\139\139\109\184\039\188\219\109\182\219\228\221\243\185\016\216\067\045\226\089\225\225\086\206\045\219\073\127\005\217\176\174\089\046\030\174\249\241\212\197\190\014\045\135\193\030\065\179\130\224\229\045\151\201\231\124\111\131\252\212\099\018\249\147\153\240\094\037\139\109\206\225\035\194\063\022\182\196\035\006\217\242\052\054\127\019\225\114\089\109\252\056\063\017\025\101\182\217\092\182\219\209\047\133\109\183\013\176\195\197\182\219\008\075\060\252\201\120\022\255\000\146\157\091\109\155\010\077\189\100\247\007\129\169\046\221\075\100\074\249\032\252\055\027\100\230\192\054\227\195\252\188\179\241\203\063\012\252\093\012\067\238\075\169\124\236\187\028\075\198\176\206\237\125\199\172\066\114\135\193\155\108\060\075\178\204\093\153\141\022\121\056\182\219\109\134\217\109\182\101\182\082\243\108\190\008\242\184\242\118\030\058\237\178\248\042\219\041\109\155\225\218\041\116\076\195\047\019\230\089\138\181\155\060\158\089\109\150\245\219\060\229\159\150\089\231\127\129\252\183\241\219\109\243\191\129\225\086\145\128\247\224\188\218\075\207\134\239\133\204\120\203\115\168\249\071\024\199\151\030\007\224\203\047\062\001\100\032\075\109\182\248\111\141\134\217\109\182\093\252\008\173\143\130\058\252\011\020\130\012\036\180\182\092\182\095\000\222\227\200\019\011\108\203\172\147\108\177\019\022\095\047\030\075\109\183\193\003\122\070\074\095\059\227\127\029\182\219\127\137\255\000\029\169\030\123\149\122\046\174\060\030\007\226\092\079\086\203\111\049\074\234\094\109\184\198\175\223\195\247\149\182\216\109\240\089\109\183\198\219\108\048\219\109\191\062\027\108\173\241\150\126\067\136\057\131\152\226\223\027\046\109\240\155\229\067\004\178\009\126\101\037\054\101\201\139\124\108\182\219\047\149\182\039\151\045\203\110\050\239\249\015\226\255\000\011\252\004\205\056\163\032\230\030\015\001\206\195\105\225\109\183\155\114\034\087\037\179\091\146\017\226\120\246\216\109\182\219\124\239\131\240\095\043\159\135\063\002\206\124\016\108\032\136\242\123\242\053\240\004\011\011\073\019\022\103\185\100\083\187\119\248\215\057\189\097\104\206\178\239\249\047\134\063\131\127\156\122\155\072\004\103\185\207\001\135\076\158\018\207\141\182\235\155\091\016\124\101\144\199\129\231\196\109\150\219\109\134\216\197\215\048\239\063\131\100\202\025\060\122\143\007\024\056\130\054\113\007\131\195\006\248\004\028\220\071\041\140\117\230\215\221\178\231\135\105\022\101\178\207\231\182\192\201\199\212\187\249\239\227\159\142\219\252\015\135\252\129\203\222\132\113\057\039\231\198\198\068\075\054\219\109\219\227\011\110\060\070\163\094\050\094\056\252\027\171\109\133\248\140\140\183\139\114\219\109\241\185\044\228\030\087\199\187\152\054\049\030\114\060\101\215\141\203\108\249\031\192\184\165\041\158\188\159\027\110\249\096\185\187\203\215\136\187\254\091\254\006\254\123\248\013\250\011\169\185\249\242\006\198\091\231\045\203\018\207\141\203\006\006\229\177\022\219\101\146\121\118\184\027\009\206\248\207\027\018\228\197\252\243\157\131\122\240\101\213\176\199\062\090\182\228\182\248\217\197\153\051\202\111\057\139\051\061\126\091\109\155\113\013\234\046\049\059\187\249\229\159\145\227\063\028\255\000\043\063\136\147\171\144\113\236\201\223\171\050\018\060\133\146\088\126\015\131\151\097\203\072\124\016\147\248\144\048\185\057\158\062\022\248\120\182\095\059\113\113\227\032\130\012\133\183\198\194\067\231\113\150\088\109\183\212\225\239\102\169\180\223\035\050\132\137\013\190\054\223\056\175\022\231\159\086\187\209\046\249\063\022\060\231\249\007\248\251\014\094\243\079\005\061\175\191\035\207\118\103\156\178\060\174\066\097\098\099\198\068\241\106\204\118\222\006\075\146\219\028\254\044\177\060\016\076\126\025\015\134\219\045\182\229\218\214\093\124\109\182\219\111\133\123\243\190\015\134\036\007\124\219\191\022\255\000\224\079\057\252\239\240\103\145\201\126\150\111\205\156\065\204\158\225\230\091\097\151\241\060\184\092\226\023\033\241\039\104\152\203\015\062\003\146\203\248\111\130\092\182\216\230\060\158\029\203\111\129\195\110\114\204\095\185\126\252\059\205\176\199\148\240\226\091\224\114\219\109\131\103\228\101\143\202\106\248\109\240\121\207\047\156\240\126\025\225\143\224\223\225\063\139\124\063\145\062\055\193\224\082\244\091\114\147\248\113\248\030\022\219\101\139\109\182\216\111\133\168\157\045\059\177\234\102\249\127\014\251\178\079\136\031\113\223\141\203\100\150\036\079\009\125\095\180\059\238\220\240\217\109\150\219\097\241\183\124\075\146\239\133\155\097\178\201\235\091\245\031\134\254\015\243\109\191\200\255\000\049\254\025\225\061\224\133\015\017\187\205\158\114\014\032\159\036\199\227\143\145\182\089\037\185\019\124\057\091\109\130\229\108\057\051\132\184\095\118\203\109\179\227\127\021\183\195\224\242\012\183\195\055\040\225\183\107\109\226\054\255\000\225\183\249\119\249\051\243\037\149\061\223\072\194\014\231\134\120\240\076\254\007\224\054\243\113\047\130\195\125\249\080\252\013\181\149\098\057\179\201\227\101\223\027\205\187\062\054\219\124\039\226\121\089\119\195\091\114\254\175\103\227\047\228\203\063\061\254\093\254\087\240\223\241\015\059\252\035\063\233\226\135\100\011\175\025\249\030\024\251\144\078\173\181\124\157\120\124\239\141\051\192\065\158\118\030\045\242\248\234\219\109\134\239\195\123\243\145\225\033\098\055\015\230\094\221\070\252\167\175\228\127\139\191\204\255\000\033\254\080\228\076\248\187\038\211\163\187\037\207\126\051\193\103\018\187\108\202\079\054\216\248\143\205\252\067\143\225\124\111\225\158\114\203\060\146\207\054\028\108\251\215\250\215\177\159\252\075\225\143\225\219\127\131\109\252\015\059\248\236\120\127\013\135\061\207\159\118\102\146\229\025\231\196\171\044\182\101\149\183\231\193\100\031\134\219\108\182\249\063\051\109\150\219\187\015\192\240\069\159\131\196\031\124\138\233\165\175\082\123\121\223\242\079\059\252\175\254\021\254\015\097\057\223\055\210\088\119\182\079\030\249\148\249\150\223\057\100\113\252\013\144\115\227\127\045\183\241\079\037\213\220\092\120\156\247\115\220\112\229\145\153\186\204\252\036\125\237\191\147\252\047\241\191\131\249\031\248\125\255\000\000\114\213\239\019\118\230\247\184\216\138\147\044\131\195\036\223\224\088\128\254\007\206\089\224\245\228\152\121\226\005\234\199\102\035\142\230\233\197\237\127\240\027\255\000\161\142\094\198\083\233\033\015\038\193\125\202\056\179\230\207\011\157\091\109\182\237\204\115\197\215\141\241\159\134\248\210\210\223\139\108\103\172\129\222\044\092\030\239\099\116\137\189\112\094\195\185\127\246\125\134\051\238\246\080\189\108\111\178\127\187\023\119\193\027\149\126\247\039\169\114\101\223\044\178\199\198\062\115\124\223\089\195\220\135\184\083\158\137\082\189\165\159\253\193\176\189\163\231\096\196\031\075\015\168\123\101\247\145\151\100\124\075\095\073\024\097\220\252\214\019\057\248\149\232\151\047\220\183\185\079\187\127\247\179\206\219\109\182\219\109\191\250\009\255\000\224\211\255\000\193\255\000\255\217")
    hubLogo=getcustomasset(path)
end)
status.Logo=logoOK and "Embedded" or tostring(logoError)
        local window = WindUI:CreateWindow({
            Title = "MaxGanHub | Block Spin", Icon = hubLogo, IconThemed=false, IconSize=36, IconRadius=18,
            Author = "Combat Edition", Folder = "MaxGanCombat_BlockSpin",
            Size = UDim2.fromOffset(650, 450), MinSize = Vector2.new(560, 380),
            Theme = "Dark", Transparent = true, Resizable = true,
            SideBarWidth = 170, ScrollBarEnabled = true, HideSearchBar = false,
        })
        window:SetToggleKey(Enum.KeyCode.G)
        statusWindow = { unloaded = false }
        statusWindow.Unload = function(self)
            if self.unloaded then return end
            self.unloaded = true
            capturingLockKey, capturingSnapKey = false, false
            refreshControls = nil
            env.MaxGanHubWindow = nil
            env.MaxGanHubControls = nil
            window:Destroy()
        end
        window:OnDestroy(function() if isRunning then env.MaxGanHubCleanup() end end)
        window:EditOpenButton({
            Title = "MaxGanHub | Block Spin", Icon = hubLogo, IconThemed=false, IconSize=36, IconRadius=18,
            CornerRadius = UDim.new(0, 16), StrokeThickness = 2,
            Color = ColorSequence.new(Color3.fromHex("FF0F7B"), Color3.fromHex("F89B29")),
            OnlyMobile = false, Enabled = true, Draggable = true,
        })
        local tabs, controls, sync = {}, {}, false
        for _, spec in ipairs({
            {"General","user"}, {"Combat","swords"}, {"Weapon Mod","crosshair"},
            {"Esp","scan"}, {"Vehicle","car"},
            {"Shop","shopping-bag"}, {"Cosmetic","sparkles"},
            {"Misc","ellipsis"}, {"Key Bind","keyboard"}, {"Server","server"}, {"Config","settings"},
        }) do tabs[spec[1]] = window:Tab({Title=thai(spec[1]), Icon=spec[2]}) end
        tabs.Visualize=tabs.Esp
        tabs.Buy=tabs.Shop
        local bindings = {}
        local function toggle(tab, title, key, getter, setter)
            local control = tab:Toggle({Title=thai(title), Value=getter(), Flag=key, Callback=function(value)
                if isRunning and not sync then setter(value) end
            end})
            controls[key] = control
            bindings[#bindings+1] = {control=control, get=getter, last=getter()}
            return control
        end
        local function slider(tab, title, key, value, minimum, maximum, step, setter)
            local control = tab:Slider({Title=thai(title), Flag=key, Step=step,
                Value={Min=minimum, Max=maximum, Default=value}, Callback=function(v)
                    v=tonumber(v)
                    if isRunning and not sync and v and v==v then setter(math.clamp(v,minimum,maximum)) end
                end})
            controls[key]=control
            return control
        end
        local function action(key, callback)
            task.spawn(function()
                local ok, accepted, reason=pcall(callback)
                status[key]=ok and (accepted and (reason or "Accepted") or (reason or "Rejected")) or tostring(accepted)
            end)
        end
        local function featureToggle(tab,title,key)
            toggle(tab,title,key,function() return Features.config[key] end,function(v) Features.config[key]=v end)
        end
        tabs.General:Section({Title="การเคลื่อนที่"})
        for _, item in ipairs({{"WalkSpeed","Enable Speed"},{"HighJump","Enable Super Jump"},
            {"AntiKill","Anti Kill"},{"AntiVehicleRagdoll","Vehicle Ragdoll Guard"}}) do
            local key=item[1]
            toggle(tabs.General,item[2],key,function() return CharacterConfig[key] end,function(value)
                CharacterConfig[key]=value
                if key=="HighJump" and not value then restoreJumpSettings() end
            end)
        end
        slider(tabs.General,"Speed boost strength","SpeedMultiplier",CharacterConfig.SpeedMultiplier,1,5,0.05,
            function(v) CharacterConfig.SpeedMultiplier=v end)
        slider(tabs.General,"Jump height (studs)","JumpHeight",CharacterConfig.JumpHeight,1,60,0.1,
            function(v) CharacterConfig.JumpHeight=v end)
        tabs.General:Section({Title="การเก็บของ"})
        featureToggle(tabs.General,"PickUp Item (ดูดของอัตโนมัติ)","Pickup")
        slider(tabs.General,"Pickup range (studs)","PickupRange",20,1,20,1,function(v) Features.config.PickupRange=v end)
        tabs.Combat:Section({Title="ระบบช่วยเล็ง"})
        toggle(tabs.Combat,"Enable Aim","AimEnabled",function() return aimLockEnabled end,setAimLockEnabled)
        local aimModes={["เล็งด้วยกล้อง"]="Aim",["เล็งเงียบ"]="Silent",["ผสม"]="Mixed"}
        local aimModeNames={Aim="เล็งด้วยกล้อง",Silent="เล็งเงียบ",Mixed="ผสม"}
        local mode=tabs.Combat:Dropdown({Title="รูปแบบช่วยเล็ง",Values={"เล็งด้วยกล้อง","เล็งเงียบ","ผสม"},Value=aimModeNames[AimConfig.Mode],
            Flag="AimMode",Callback=function(value)
                value=aimModes[value] or value
                if not isRunning or sync then return end
                if value~="Aim" and value~="Silent" and value~="Mixed" then return end
                AimConfig.Mode=value; silentLockTarget=nil
                if value~="Aim" then AimConfig.ShowFOV=true; if AimConfig.FOV==AIM_FOV then AimConfig.FOV=150 end end
                resetCameraLock("Ready"); status.SilentAim=aimLockEnabled and value~="Aim"
                if refreshControls then refreshControls() end
            end})
        controls.AimMode=mode
        slider(tabs.Combat,"On-foot response","StandingResponse",AimConfig.StandingResponse,10,120,1,function(v) AimConfig.StandingResponse=v end)
        slider(tabs.Combat,"Vehicle response","VehicleResponse",AimConfig.VehicleResponse,10,120,1,function(v) AimConfig.VehicleResponse=v end)
        local aimStatus=tabs.Combat:Paragraph({Title="สถานะช่วยเล็ง",Desc="กำลังเริ่ม..."})
        local names, updatingPlayers={},false
        local excluded=tabs.Combat:Dropdown({Title="ผู้เล่นที่ไม่เล็ง",Values={},Value={},Multi=true,
            AllowNone=true,SearchBarEnabled=true,Callback=function(selected)
                if not isRunning or updatingPlayers then return end
                local selectedNames={}
                for _,name in ipairs(selected) do selectedNames[name]=true end
                for name,player in pairs(names) do ignoredPlayers[player.UserId]=selectedNames[name] or nil end
                if cameraLockTarget and ignoredPlayers[cameraLockTarget.UserId] then resetCameraLock("Ready") end
                if silentLockTarget and ignoredPlayers[silentLockTarget.UserId] then silentLockTarget=nil end
            end})
        local function rebuildPlayers()
            if not isRunning or statusWindow.unloaded then return end
            updatingPlayers=true; table.clear(names)
            local values, selected={},{}
            for _,player in ipairs(Players:GetPlayers()) do
                if player~=LocalPlayer then
                    local name=player.DisplayName.." (@"..player.Name..")"
                    names[name]=player; values[#values+1]=name
                    if ignoredPlayers[player.UserId] then selected[#selected+1]=name end
                end
            end
            table.sort(values); excluded:Refresh(values); excluded:Select(selected); updatingPlayers=false
        end
        track(Players.PlayerAdded:Connect(rebuildPlayers))
        track(Players.PlayerRemoving:Connect(function() task.defer(rebuildPlayers) end))
        rebuildPlayers()
        tabs.Combat:Section({Title="อาวุธประชิด"})
        featureToggle(tabs.Combat,"Melee Aura","Melee")
        slider(tabs.Combat,"Melee Range","MeleeRange",15,1,30,1,function(v) Features.config.MeleeRange=v end)
        tabs.Combat:Section({Title="อาวุธขว้าง"})
        featureToggle(tabs.Combat,"Throw Aura","Throw")
        slider(tabs.Combat,"Throw Range","ThrowRange",39,1,100,1,function(v) Features.config.ThrowRange=v end)
        tabs.WeaponMod=tabs["Weapon Mod"]
        tabs.WeaponMod:Section({Title="ปรับการยิง"})
        toggle(tabs.WeaponMod,"Accuracy / Range Mod","WeaponAccuracy",function() return WeaponConfig.Accuracy end,
            function(v) WeaponConfig.Accuracy=v; if not v and weaponCleanup then weaponCleanup() end end)
        slider(tabs.WeaponMod,"ความแม่นยำ (%)","AccuracyAmount",100,0,100,1,function(v) WeaponConfig.Amount=v/100 end)
        featureToggle(tabs.WeaponMod,"Fast Reload","FastReload")
        slider(tabs.WeaponMod,"Reload Speed Multiplier","ReloadMultiplier",2.5,1,5,0.1,function(v) Features.config.ReloadMultiplier=v end)
        tabs.Esp:Section({Title="แสดงข้อมูลผู้เล่น"})
        toggle(tabs.Esp,"Enable ESP","ESPEnabled",function() return ESPConfig.Enabled end,function(v) ESPConfig.Enabled=v end)
        for _,item in ipairs({{"Box","ESP Box"},{"Name","ESP Name"},{"Distance","ESP Distance"},
            {"Health","ESP Health"},{"Inventory","ESP Inventory"}}) do
            local key=item[1]
            toggle(tabs.Esp,item[2],"ESP"..key,function() return ESPConfig[key] end,function(v) ESPConfig[key]=v end)
        end
        slider(tabs.Esp,"Max distance (studs)","ESPDistance",MAX_ESP_DIST,100,5000,100,function(v) MAX_ESP_DIST=v end)
        tabs.Visualize:Section({Title="วงช่วยเล็ง"})
        toggle(tabs.Visualize,"Show FOV","ShowFOV",function() return AimConfig.ShowFOV end,function(v)
            AimConfig.ShowFOV=v
            if v and AimConfig.FOV==AIM_FOV then AimConfig.FOV=150 end
        end)
        local fov=slider(tabs.Visualize,"FOV radius (pixels)","AimFOV",AimConfig.FOV,25,400,1,function(v) AimConfig.FOV=math.floor(v) end)
        slider(tabs.Visualize,"ขนาดวงเหลืองบนเป้า (รัศมีพิกเซล)","TargetMarkerSize",AimConfig.TargetMarkerRadius,4,24,1,function(v) AimConfig.TargetMarkerRadius=math.floor(v) end)
        tabs.Vehicle:Section({Title="การควบคุมรถ"})
        toggle(tabs.Vehicle,"Smooth Drive","VehicleHover",function() return VehicleConfig.Flight end,setVehicleFlight)
        toggle(tabs.Vehicle,"Vehicle Underground","VehicleUnderground",function() return VehicleConfig.Underground end,setVehicleUnderground)
        for _,item in ipairs({{"Speed","Vehicle speed (studs/s)",5,300,5},
            {"HoverHeight","Hover height (studs)",0.5,5,0.5},{"Depth","Underground depth",5,100,5},
            }) do
            local key=item[1]
            slider(tabs.Vehicle,item[2],"Vehicle"..key,VehicleConfig[key],item[3],item[4],item[5],function(v)
                if key=="Depth" and vehicleState and vehicleState.undergroundOrigin then
                    local _,size=vehicleState.vehicle:GetBoundingBox()
                    if vehicleState.undergroundOrigin.Position.Y-v-size.Y<Workspace.FallenPartsDestroyHeight+10 then
                        status.VehicleMode="Depth below world limit"; return
                    end
                end
                VehicleConfig[key]=v
            end)
        end
        tabs.Vehicle:Section({Title="ชนเป้า"})
        toggle(tabs.Vehicle,"Bump Aura","BumpAura",function() return VehicleConfig.BumpAura end,function(v)
            if networkAccessError then status.VehicleBump=networkAccessError; return end
            VehicleConfig.BumpAura=v; status.VehicleBump=v and "Waiting for target" or "Off"
            if v then task.spawn(resolveBumpNetwork) end
        end)
        slider(tabs.Vehicle,"Bump radius","VehicleBumpRadius",VehicleConfig.BumpRadius,5,60,5,function(v) VehicleConfig.BumpRadius=v end)
        slider(tabs.Vehicle,"Impact speed","VehicleBumpSpeed",VehicleConfig.BumpSpeed,15,1000,25,function(v) VehicleConfig.BumpSpeed=v end)
        tabs.Vehicle:Section({Title="จัดการรถ"})
        tabs.Vehicle:Button({Title="นำรถขึ้นพื้น",Callback=returnVehicleUp})
        tabs.Vehicle:Button({Title="หยุดระบบรถ",Callback=function() stopVehicle("Off",false); refreshControls() end})
        local vehicleStatus=tabs.Vehicle:Paragraph({Title="สถานะรถ",Desc="ปิด"})
        tabs.Vehicle:Button({Title="เรียกรถของตัวเอง",Desc="เรียกรถของตัวเองที่เกิดแล้ว หาที่จอดว่างรอบตัว",Callback=function()
            action("CarRecall",function() return Features:PullCar() end)
        end})
        tabs.Buy:Section({Title="การสุ่มลัง"})
        toggle(tabs.Buy,"Skip crate animation","SkipCrateAnimation",function() return CrateConfig.SkipAnimation end,
            function(v) CrateConfig.SkipAnimation=v end)
        local crateStatus=tabs.Buy:Paragraph({Title="สถานะการสุ่ม",Desc="ปิด"})
        tabs.Buy:Button({Title="สุ่มลังที่เลือก 1 ครั้ง",Desc="เปิดลังที่เลือกไว้ในเกม 1 ครั้ง ใช้เงินหรือโทเคนในเกม",Callback=function()
            action("ForceSpin",function() return Features:ForceSpin() end)
        end})
        tabs.Shop:Section({Title="ร้านขายของ"})
        tabs.Shop:Button({Title="เปิดร้านใกล้ตัว",Desc="อยู่ใกล้ร้านแล้วเปิดรายการสินค้าของเกม",Callback=function()
            action("Shop",function() return Features:OpenShop() end)
        end})
        local shopStatus=tabs.Shop:Paragraph({Title="สถานะร้านค้า",Desc="ต้องเข้าใกล้ร้านก่อน"})
        tabs.Cosmetic:Section({Title="สกินและเอฟเฟกต์"})
        local owned,chosenItem,chosenCosmetic={},nil,nil
        local itemDrop=tabs.Cosmetic:Dropdown({Title="ของที่ต้องการใส่สกิน",Values={},Callback=function(v) chosenItem=owned[v] end})
        local cosmeticDrop=tabs.Cosmetic:Dropdown({Title="สกินที่มี",Values={},Callback=function(v) chosenCosmetic=owned[v] end})
        local function refreshCosmetics()
            local ok,items=pcall(function() return Features:OwnedItems() end)
            if not ok then status.Cosmetic=tostring(items);return end
            owned=items;chosenItem=nil;chosenCosmetic=nil
            local targets,cosmetics={},{}
            for label,item in pairs(owned) do
                if item.item_type=="cosmetic" then cosmetics[#cosmetics+1]=label
                elseif item.item_type=="gun" or item.item_type=="melee" or item.item_type=="car" or item.item_type=="bike" then targets[#targets+1]=label end
            end
            table.sort(targets);table.sort(cosmetics);itemDrop:Refresh(targets);cosmeticDrop:Refresh(cosmetics)
            status.Cosmetic=#cosmetics>0 and "Select item and compatible cosmetic" or "No owned cosmetics"
        end
        tabs.Cosmetic:Button({Title="โหลดรายการของใหม่",Callback=refreshCosmetics})
        tabs.Cosmetic:Button({Title="ใส่ / ถอดสกิน",Callback=function()
            action("Cosmetic",function()
                if not chosenItem or not chosenCosmetic then return false,"Select an item and owned cosmetic first" end
                local valid=Features:Module("Game.ItemCustomization.ValidCosmeticTypes")
                if not valid.can_apply_cosmetic(chosenItem.name,chosenCosmetic.name) then return false,"Cosmetic is incompatible with this item" end
                return Features:ApplyCosmetic(chosenItem.guid,chosenCosmetic.guid)
            end)
        end})
        local cosmeticStatus=tabs.Cosmetic:Paragraph({Title="สถานะสกิน",Desc="โหลดรายการของก่อน"})
        tabs.Misc:Section({Title="การฟื้นฟูเลือด"})
        toggle(tabs.Misc,"Auto Heal","AutoHeal",function() return AutoHealConfig.Enabled end,function(v)
            AutoHealConfig.Enabled=v
            if v then healNextAttempt=0; task.spawn(function() pcall(resolveHealInventory); requestAutoHeal() end) end
        end)
        slider(tabs.Misc,"Heal below HP","HealThreshold",AutoHealConfig.Threshold,5,100,5,function(v) AutoHealConfig.Threshold=v end)
        slider(tabs.Misc,"Heal animation speed","HealAnimation",AutoHealConfig.AnimationSpeed,1,3,0.5,function(v) AutoHealConfig.AnimationSpeed=v end)
        tabs.Misc:Section({Title="การเติมสเตมิน่า"})
        featureToggle(tabs.Misc,"Auto Consume (เพิ่มสเตมิน่า)","AutoConsume")
        slider(tabs.Misc,"Consume below stamina (%)","ConsumeThreshold",30,5,95,5,function(v) Features.config.ConsumeThreshold=v end)
        local staminaChoices={"เลือกอัตโนมัติ"}
        for _,item in ipairs(ReplicatedStorage.Items.consumable:GetChildren()) do
            if (item:GetAttribute("StaminaRestoreAmount") or 0)>0 then staminaChoices[#staminaChoices+1]=item.Name end
        end
        table.sort(staminaChoices,function(a,b) if a=="เลือกอัตโนมัติ" then return b~="เลือกอัตโนมัติ" end; if b=="เลือกอัตโนมัติ" then return false end; return a<b end)
        controls.ConsumeItem=tabs.Misc:Dropdown({Title="ของเติมสเตมิน่าที่ใช้",Values=staminaChoices,Value="เลือกอัตโนมัติ",Flag="ConsumeItem",Callback=function(v)
            if isRunning and not sync and table.find(staminaChoices,v) then Features.config.ConsumeItem=v=="เลือกอัตโนมัติ" and "Smart Select" or v end
        end})
        local consumeStatus=tabs.Misc:Paragraph({Title="สถานะเติมสเตมิน่า",Desc="ปิด — ใช้ของที่อยู่ในกระเป๋า"})
        tabs.Misc:Section({Title="มุดดินและกลับขึ้นพื้น"})
        tabs.Misc:Button({Title="มุดดิน / กลับขึ้น",Callback=function() if isRunning then toggleSnap() end end})
        tabs.Misc:Button({Title="กลับขึ้นพื้น",Callback=function() if isRunning then requestManualReturn() end end})
        tabs.Misc:Button({Title="ปล่อยการตรึงตัว",Callback=function() if isRunning then releaseSnap() end end})
        slider(tabs.Misc,"Snap depth (studs)","SnapDepth",SnapConfig.Depth,1,100,1,function(v)
            local origin=snapState and snapState.origin
            local _,_,root=currentBody()
            local height=origin and origin.Position.Y or (root and root.Position.Y)
            if not height or (height-v>Workspace.FallenPartsDestroyHeight+10 and
                (not snapState or height-v-snapState.bottomOffset>Workspace.FallenPartsDestroyHeight+5)) then
                SnapConfig.Depth=v
            else status.Snap="Depth below world limit" end
        end)
        tabs.KeyBind=tabs["Key Bind"]
        tabs.KeyBind:Section({Title="ตั้งค่าปุ่มลัด"})
        local menuKey=Enum.KeyCode.G
        local menuKeys={}
        for _,key in ipairs(Enum.KeyCode:GetEnumItems()) do
            if #key.Name==1 or key.Name:match("^F%d+$") or key.Name=="Insert" or key.Name=="Home" or key.Name=="End" or key.Name=="RightControl" then menuKeys[#menuKeys+1]=key.Name end
        end
        table.sort(menuKeys)
        local function setMenuKey(name)
            local key=Enum.KeyCode[name]
            if not key or key==AimConfig.LockKey or key==SnapConfig.Key then return false end
            menuKey=key;MenuConfig.Key=key;window:SetToggleKey(key);return true
        end
        controls.MenuKey=tabs.KeyBind:Dropdown({Title="เปิด / ปิดเมนู",Values=menuKeys,Value="G",Flag="MenuKey",Callback=function(v)
            if isRunning and not sync and not setMenuKey(v) then
                WindUI:Notify({Title="ปุ่มนี้ถูกใช้อยู่",Content="เลือกปุ่มที่ไม่ซ้ำ Aim หรือ Snap",Duration=3})
                sync=true;controls.MenuKey:Select(menuKey.Name);sync=false
            end
        end})
        local aimKey=tabs.KeyBind:Button({Title="ปุ่มช่วยเล็ง: F",Callback=function()
            capturingLockKey=not capturingLockKey; capturingSnapKey=false; refreshControls()
        end})
        local snapKey=tabs.KeyBind:Button({Title="ปุ่มมุดดิน: V",Callback=function()
            capturingSnapKey=not capturingSnapKey; capturingLockKey=false; refreshControls()
        end})
        tabs.Server:Section({Title="จัดการเซิร์ฟเวอร์"})
        local serverStatus=tabs.Server:Paragraph({Title="สถานะเซิร์ฟเวอร์",Desc="กำลังเริ่ม..."})
        tabs.Server:Button({Title="เข้าเซิร์ฟเวอร์เดิมใหม่",Callback=function()
            game:GetService("TeleportService"):TeleportToPlaceInstance(game.PlaceId,game.JobId,LocalPlayer)
        end})
        tabs.Config:Section({Title="จัดการการตั้งค่า"})
        local configName="default"
        tabs.Config:Input({Title="ชื่อชุดการตั้งค่า",Value=configName,Placeholder="default",Callback=function(v)
            local name=tostring(v):match("^[%w_%-]+$")
            if name and #name<=64 then configName=name end
        end})
        local function configAction(save)
            if type(readfile)~="function" or type(writefile)~="function" or type(isfile)~="function" then
                WindUI:Notify({Title="บันทึกการตั้งค่าไม่ได้",Content="Executor file APIs are unavailable",Duration=4}); return
            end
            local success,result,message=pcall(function()
                local config=window.ConfigManager:CreateConfig(configName)
                if save then
                    config:Set("AimKey",AimConfig.LockKey.Name); config:Set("SnapKey",SnapConfig.Key.Name);config:Set("MenuKey",menuKey.Name)
                    config:Save(); return true
                end
                local data,reason=config:Load()
                if data==false then return false,reason end
                local aim=data.AimKey and Enum.KeyCode[data.AimKey]
                local snap=data.SnapKey and Enum.KeyCode[data.SnapKey]
                if aim and snap and aim~=snap and aim~=Enum.KeyCode.Unknown and snap~=Enum.KeyCode.Unknown
                    and aim~=menuKey and snap~=menuKey then
                    AimConfig.LockKey=aim; SnapConfig.Key=snap
                end
                return true
            end)
            WindUI:Notify({Title=success and result and (save and "Config saved" or "Config loaded") or "Config failed",
                Content=success and result and configName or tostring(success and message or result),Duration=3})
        end
        tabs.Config:Button({Title="บันทึกการตั้งค่า",Callback=function() configAction(true) end})
        tabs.Config:Button({Title="โหลดการตั้งค่า",Callback=function() configAction(false) end})
        tabs.Config:Button({Title="หยุดทั้งหมดและปิดสคริปต์",Icon="power",Callback=function() env.MaxGanHubCleanup() end})
        local lastText,lastFOV={},AimConfig.FOV
        local function desc(control,key,text)
            if lastText[key]~=text then lastText[key]=text; control:SetDesc(thaiStatus(text)) end
        end
        refreshControls=function()
            if not isRunning or statusWindow.unloaded then return end
            sync=true
            for _,binding in ipairs(bindings) do
                local v=binding.get()
                if binding.control.Value~=v then binding.last=v; binding.control:Set(v) end
            end
            if lastFOV~=AimConfig.FOV then lastFOV=AimConfig.FOV; fov:Set(lastFOV) end
            sync=false
            local a=capturingLockKey and "กดปุ่มที่ต้องการ (Esc ยกเลิก)" or "ปุ่มช่วยเล็ง: "..AimConfig.LockKey.Name
            local s=capturingSnapKey and "กดปุ่มที่ต้องการ (Esc ยกเลิก)" or "ปุ่มมุดดิน: "..SnapConfig.Key.Name
            if lastText.aimKey~=a then lastText.aimKey=a; aimKey:SetTitle(a) end
            if lastText.snapKey~=s then lastText.snapKey=s; snapKey:SetTitle(s) end
        end
        local function update()
            refreshControls()
            desc(aimStatus,"aim",AimConfig.Mode.." | "..tostring(status.AimLock or "Ready").."\n"..
                tostring(status.Target or "No target").."\nเล็งเงียบ: "..tostring(status.SilentAimError or status.SilentAimResult or
                    (status.SilentAimCompatibility=="Ready" and "พร้อมดักคำสั่งยิง — ยังไม่ยืนยันดาเมจ" or status.SilentAimCompatibility) or "Starting"))
            desc(vehicleStatus,"vehicle",tostring(status.VehicleMode or "Off").."\nชนเป้า: "..tostring(status.VehicleBump or "Off"))
            desc(crateStatus,"crate",tostring(status.CrateSkipError or status.CrateSkip or "Off").."\n"..tostring(status.ForceSpin or "Select a crate first"))
            desc(shopStatus,"shop",tostring(status.Shop or "Go near a shop first"))
            desc(consumeStatus,"consume",tostring(status.AutoConsume or "Off"))
            desc(cosmeticStatus,"cosmetic",tostring(status.Cosmetic or "Refresh inventory first"))
            desc(serverStatus,"server","FPS "..tostring(status.FPS or "--").." | Ping "..tostring(status.Ping or "--")..
                " ms | "..tostring(status.PingState or "Measuring"))
        end
        update()
        local elapsed=0
        track(RunService.Heartbeat:Connect(function(dt)
            if not isRunning or statusWindow.unloaded then return end
            elapsed+=dt
            if elapsed>=0.5 then elapsed=0; update() end
        end))
        for _,node in ipairs(window.UIElements.Main:GetDescendants()) do
            if node:IsA("TextBox") and (node.PlaceholderText=="Search..." or node.PlaceholderText=="Search") then node.PlaceholderText="ค้นหา..." end
            if node:IsA("TextLabel") and node.Text=="Search" then node.Text="ค้นหา" end
        end
        local function fitLogo(node)
            if node:IsA("ImageLabel") and node.Image==hubLogo then node.ScaleType=Enum.ScaleType.Fit end
        end
        local rootGui=game:GetService("CoreGui"):FindFirstChild("RobloxGui")
        local windRoot=rootGui and rootGui:FindFirstChild("WindUI")
        if windRoot then
            for _,node in ipairs(windRoot:GetDescendants()) do fitLogo(node) end
            track(windRoot.DescendantAdded:Connect(fitLogo))
        end
        window:SelectTab(1)
        env.MaxGanHubWindow,env.MaxGanHubControls=window,controls
        status.UI="WindUI"; status.PendingFeatures={}
    end)
    if not ok then
        status.UI,status.UIError="Unavailable",tostring(err)
        if statusWindow then pcall(function() statusWindow:Unload() end) end
        warn("[MaxGan Hub] WindUI unavailable: "..tostring(err))
    end
end)

Features = {config={Pickup=false,PickupRange=20,Melee=false,MeleeRange=15,Throw=false,ThrowRange=39,
    FastReload=false,ReloadMultiplier=2.5,AutoConsume=false,ConsumeThreshold=30,ConsumeItem="Smart Select"},modules={},reloads={},pickupTimes=setmetatable({},{__mode="k"})}
env.MaxGanHubFeatures=Features
VehicleConfig.SpeedMultiplier=1
function Features:Module(path)
    if self.modules[path] then return self.modules[path] end
    local instance=ReplicatedStorage.Modules
    for segment in path:gmatch("[^.]+") do instance=assert(instance:FindFirstChild(segment),"Module missing: "..path) end
    local value=requireGameModule(instance)
    self.modules[path]=value
    return value
end
function Features:Get(action,...)
    if not isRunning then return false,"Stopped" end
    local net,sequence,reason=resolveActiveNetwork()
    if not sequence then return false,reason end
    local remote=ReplicatedStorage.Remotes:FindFirstChild("Get")
    if not remote or not remote:IsA("RemoteFunction") then return false,"Get channel unavailable" end
    sequence.func+=1
    return remote:InvokeServer(sequence.func,action,...)
end
function Features:Stamina()
    local value=self:Module("Game.Sprint").sprint_bar.get()
    return type(value)=="number" and math.clamp(value,0,1) or nil
end
function Features:StaminaItems()
    local result={}
    local folder=ReplicatedStorage.Items:FindFirstChild("consumable")
    for name,stacks in pairs(self:Module("Core.Data").items.consumable or {}) do
        local template=folder and folder:FindFirstChild(name)
        local restore=template and template:GetAttribute("StaminaRestoreAmount")
        if type(restore)=="number" and restore>0 then
            for _,stack in pairs(stacks) do
                if stack.location=="hand" and (stack.amount or 0)>0 and stack.guid then
                    result[#result+1]={name=name,guid=stack.guid,amount=stack.amount,restore=restore,
                        price=template:GetAttribute("RarityPrice") or 0}
                end
            end
        end
    end
    return result
end
function Features:ConsumeStamina()
    if not self.config.AutoConsume then status.AutoConsume="Off"; return end
    if self.consumeBusy or healBusy or os.clock()<(self.consumeNextAttempt or 0) then return end
    local char,hum=currentBody()
    if not char or not hum or hum.Health<=0 or downed(hum) or hum:GetAttribute("Stunned") then return end
    if AutoHealConfig.Enabled and hum.Health<AutoHealConfig.Threshold then return end
    local stamina=self:Stamina()
    if not stamina then status.AutoConsume="Stamina unavailable"; return end
    if stamina*100>self.config.ConsumeThreshold then status.AutoConsume="Waiting for low stamina"; return end
    local needed=(1-stamina)*(LocalPlayer:GetAttribute("MaxStamina") or 100)
    local candidates=self:StaminaItems()
    table.sort(candidates,function(a,b)
        local ar=math.min(a.restore,needed)/math.max(a.price,1)
        local br=math.min(b.restore,needed)/math.max(b.price,1)
        if ar==br then return a.name<b.name end
        return ar>br
    end)
    local chosen
    for _,item in ipairs(candidates) do
        if self.config.ConsumeItem=="Smart Select" or self.config.ConsumeItem==item.name then chosen=item;break end
    end
    if not chosen then status.AutoConsume="No stamina consumable in carried inventory"; self.consumeNextAttempt=os.clock()+3; return end
    self.consumeBusy=true
    task.spawn(function()
        local consumed=false
        local previous=char:FindFirstChildOfClass("Tool")
        local tool
        local restored=false
        local function restoreTool()
            if restored then return end
            restored=true
            if LocalPlayer.Character~=char or hum.Health<=0 or not tool then return end
            local held=char:FindFirstChildOfClass("Tool")
            if held==tool or (consumed and not held) then
                if previous and previous~=tool and previous.Parent==LocalPlayer:FindFirstChildOfClass("Backpack") then
                    hum:EquipTool(previous)
                elseif held==tool then hum:UnequipTools() end
            end
        end
        self.consumeCancel=restoreTool
        local function valid()
            return isRunning and self.config.AutoConsume and LocalPlayer.Character==char and hum.Health>0
                and not downed(hum) and not hum:GetAttribute("Stunned")
                and not (AutoHealConfig.Enabled and hum.Health<AutoHealConfig.Threshold)
        end
        local function remaining()
            for _,item in ipairs(self:StaminaItems()) do if item.guid==chosen.guid then return item.amount end end
            return 0
        end
        local ok,err=pcall(function()
            local backpack=LocalPlayer:FindFirstChildOfClass("Backpack")
            local function find()
                for _,container in ipairs({char,backpack}) do
                    if container then for _,item in ipairs(container:GetChildren()) do
                        if item:IsA("Tool") and item.Name==chosen.name and item:GetAttribute("StaminaRestoreAmount") then return item end
                    end end
                end
            end
            tool=find()
            if not tool then
                status.AutoConsume="Equipping "..chosen.name
                self:Get("toggle_equip_item",chosen.guid)
                local deadline=os.clock()+4
                repeat
                    if not valid() then return end
                    tool=find()
                    if tool then break end
                    task.wait(0.1)
                until os.clock()>=deadline
            end
            if not tool then status.AutoConsume="Equip not confirmed"; return end
            if not valid() or healBusy then return end
            hum:EquipTool(tool)
            if tool.Parent~=char then status.AutoConsume="Consumable not held"; return end
            status.AutoConsume="Consuming "..chosen.name
            local before=self:Stamina()
            tool:Activate()
            local deadline=os.clock()+30
            repeat
                task.wait(0.1)
                if not valid() then return end
                if remaining()<chosen.amount then consumed=true;break end
                if tool.Parent~=char then status.AutoConsume="Consumption interrupted";return end
            until os.clock()>=deadline
            status.AutoConsume=consumed and "Consumed "..chosen.name or "Consumption not confirmed"
            status.ConsumeLast={item=chosen.name,confirmed=consumed,staminaBefore=before,staminaAfter=self:Stamina()}
        end)
        restoreTool()
        if self.consumeCancel==restoreTool then self.consumeCancel=nil end
        if not ok then status.AutoConsume=tostring(err) end
        self.consumeNextAttempt=os.clock()+(consumed and 5 or 3)
        self.consumeBusy=false
    end)
end
track({Disconnect=function() if Features.consumeCancel then Features.consumeCancel() end end})
task.spawn(function()
    while isRunning do
        local ok,err=pcall(function() Features:ConsumeStamina() end)
        if not ok then status.AutoConsume=tostring(err);Features.consumeNextAttempt=os.clock()+3 end
        task.wait(0.25)
    end
end)
function Features:Nearest(radius)
    local _,hum,root=currentBody()
    if not root or not hum or hum.Health<=0 or downed(hum) or hum.SeatPart or LocalPlayer:GetAttribute("IsInPassiveMode") then return end
    local best,distance=nil,radius
    for _,player in ipairs(Players:GetPlayers()) do
        if isValid(player) and not ignoredPlayers[player.UserId] and not player:GetAttribute("IsInPassiveMode")
            and not player:GetAttribute("IsSafeZoneProtected") and not player:GetAttribute("IsInSafeZone") then
            local parts=getParts(player)
            if parts.root and parts.hum and parts.hum.Health>0 and not downed(parts.hum) then
                local d=(parts.root.Position-root.Position).Magnitude
                local params=RaycastParams.new()
                params.FilterType=Enum.RaycastFilterType.Exclude; params.FilterDescendantsInstances={LocalPlayer.Character,player.Character}
                params.RespectCanCollide=true
                if d>0.1 and d<distance and not Workspace:Raycast(root.Position,parts.root.Position-root.Position,params) then
                    best,distance=player,d
                end
            end
        end
    end
    return best
end
function Features:AuraAllowed()
    local _,hum=currentBody()
    return isRunning and hum and hum.Health>0 and not downed(hum) and not hum.SeatPart
        and not hum:GetAttribute("Stunned") and not healBusy and not self.consumeBusy
        and not UIS:GetFocusedTextBox() and not LocalPlayer:GetAttribute("IsInPassiveMode")
        and not LocalPlayer:GetAttribute("IsInSafeZone") and not LocalPlayer:GetAttribute("IsSafeZoneProtected")
end
function Features:InstallAuraHooks()
    if self.auraHooks then return end
    local melee=self:Module("Game.ItemTypes.Melee")
    local throwable=self:Module("Game.ItemTypes.Throwable")
    local net=resolveActiveNetwork()
    assert(net and type(net.send)=="function","Native network unavailable")
    local features=self
    self.auraStatus=status
    local originalHits=melee.get_hit_players
    local hits=function(object)
        if features.config.Melee and features:AuraAllowed() and melee.equipped_melee.get()==object.instance then
            local target=features:Nearest(math.clamp(features.config.MeleeRange,1,30))
            status.MeleeTarget=target and target.Name or nil
            return target and {target} or {}
        end
        return originalHits(object)
    end
    melee.get_hit_players=hits
    local originalVisual=throwable.thrown_item
    local visual=function(player,name,position,direction,...)
        local pending=features.throwPending
        if pending and player==LocalPlayer and name==pending.tool.Name then direction=pending.direction or direction end
        return originalVisual(player,name,position,direction,...)
    end
    throwable.thrown_item=visual
    self.auraHooks={melee=melee,hits=hits,originalHits=originalHits,throwable=throwable,
        visual=visual,originalVisual=originalVisual}
    local slot=env.MaxGanHubAuraHook
    if not slot or slot.target~=net.send then
        if slot then slot.context=nil end
        slot={target=net.send}
        local original
        original=hookfunction(net.send,function(...)
            local context=slot.context
            if context then
                local auraStatus=context.auraStatus
                local args=table.pack(...)
                if args[1]=="throw_item" and context.throwPending and args[2]==context.throwPending.tool then
                    local pending=context.throwPending
                    local parts=getParts(pending.target)
                    if parts.root and parts.hum and parts.hum.Health>0 and typeof(args[3])=="Vector3" then
                        local delta=parts.root.Position-args[3]
                        if delta.Magnitude>0.1 then
                            pending.direction=delta.Unit
                            args[4]=pending.direction
                            auraStatus.ThrowRequests=(auraStatus.ThrowRequests or 0)+1
                            auraStatus.ThrowTarget=pending.target.Name
                        end
                    end
                    return original(table.unpack(args,1,args.n))
                elseif args[1]=="melee_attack" and context.config.Melee then
                    auraStatus.MeleeRequests=(auraStatus.MeleeRequests or 0)+1
                end
            end
            return original(...)
        end)
        slot.original=original
        env.MaxGanHubAuraHook=slot
    end
    slot.context=self
    self.auraSlot=slot
end
function Features:RestoreAuras()
    if self.auraSlot and self.auraSlot.context==self then self.auraSlot.context=nil end
    local hooks=self.auraHooks
    if hooks then
        if hooks.melee.get_hit_players==hooks.hits then hooks.melee.get_hit_players=hooks.originalHits end
        if hooks.throwable.thrown_item==hooks.visual then hooks.throwable.thrown_item=hooks.originalVisual end
        self.auraHooks=nil
    end
    self.throwPending=nil
end
function Features:ThrowAction(tool)
    self.throwActions=self.throwActions or setmetatable({},{__mode="k"})
    if self.throwActions[tool] then return self.throwActions[tool] end
    local context=getsenv(ReplicatedStorage.Modules.Game.Keybinds)
    local actions=context and context.hooked_functions
    if not actions then return end
    local visited={}
    local function find(fn,depth)
        if type(fn)~="function" or visited[fn] or depth>3 then return end
        visited[fn]=true
        local ok,values=pcall(getupvalues,fn)
        if not ok then return end
        local hasTool,hasSprint,hasThrowable=false,false,false
        for _,value in pairs(values) do
            if value==tool then hasTool=true end
            if type(value)=="table" then
                if type(rawget(value,"consume_stamina"))=="function" then hasSprint=true end
                if rawget(value,"equipped_throwable") then hasThrowable=true end
            end
        end
        if hasTool and hasSprint and hasThrowable then return fn end
        for _,value in pairs(values) do
            if type(value)=="function" then local found=find(value,depth+1);if found then return found end end
        end
    end
    for _,name in ipairs({"aim","aim_toggle","shoot"}) do
        for _,handler in ipairs(actions[name] or {}) do
            local found=find(handler.func,0)
            if found then self.throwActions[tool]=found;return found end
        end
    end
end
function Features:ActivateAt(target,kind,tool)
    self:InstallAuraHooks()
    if kind=="Throw" then
        local action=self:ThrowAction(tool)
        if not action then return false,"Native throw action unavailable" end
        local before=status.ThrowRequests or 0
        self.throwPending={target=target,tool=tool}
        local ok,err=pcall(action,true)
        self.throwPending=nil
        if not ok then return false,tostring(err) end
        local requested=(status.ThrowRequests or 0)>before
        return requested,requested and "Native throw requested; damage unconfirmed" or "Waiting for native cooldown, ammunition or stamina"
    end
    local melee=self:Module("Game.ItemTypes.Melee")
    local object=melee.class.get(tool)
    if not object then return false,"Melee object unavailable" end
    melee.melee_attack(object)
    return true,"Native melee requested; damage unconfirmed"
end
function Features:RestoreReloads()
    for object,saved in pairs(self.reloads) do
        pcall(function()
            if object.states.reload_time.get()==saved.applied then object.states.reload_time.set(saved.original) end
        end)
    end
    table.clear(self.reloads)
end
function Features:UpdateReloads()
    if not self.config.FastReload then self:RestoreReloads(); status.FastReload="Off"; return end
    local guns=nativeGunModule or self:Module("Game.ItemTypes.Gun")
    local char,backpack=LocalPlayer.Character,LocalPlayer:FindFirstChildOfClass("Backpack")
    local count=0
    for _,object in pairs(guns.class.objects) do
        local tool=object.instance
        if tool and object.states and object.states.reload_time and
            ((char and tool:IsDescendantOf(char)) or (backpack and tool:IsDescendantOf(backpack))) then
            local current=object.states.reload_time.get()
            local saved=self.reloads[object]
            if not saved then saved={original=current}; self.reloads[object]=saved
            elseif current~=saved.applied then saved.original=current end
            if type(saved.original)=="number" and saved.original>0 then
                saved.applied=math.max(0.1,saved.original/math.clamp(self.config.ReloadMultiplier,1,5))
                object.states.reload_time.set(saved.applied);count+=1
                local reloading=object.reload and object.reload.running
                local mag=object.states.mag.get()
                local shot=object.states.last_shot.get()
                if reloading and not saved.reloading then
                    saved.magBefore=mag;saved.shotBefore=shot
                elseif saved.reloading and not reloading and saved.magBefore==mag and
                    mag<(tool:GetAttribute("MagSize") or 0) and shot==saved.shotBefore then
                    saved.retryAt=os.clock()+0.5
                end
                saved.reloading=reloading
                if saved.retryAt and os.clock()>=saved.retryAt then
                    saved.retryAt=nil
                    if tool.Parent==char and not reloading and mag==saved.magBefore and shot==saved.shotBefore and
                        (saved.retries or 0)<2 then
                        saved.retries=(saved.retries or 0)+1
                        object.reload()
                    end
                end
                if mag~=saved.magBefore or shot~=saved.shotBefore then saved.retries=0;saved.retryAt=nil end
            end
        elseif self.reloads[object] then
            local saved=self.reloads[object]
            if object.states.reload_time.get()==saved.applied then object.states.reload_time.set(saved.original) end
            self.reloads[object]=nil
        end
    end
    status.FastReload=count>0 and "Fast reload active; native retry enabled" or "Waiting for a gun in your backpack"
end
function Features:Pickup()
    if not self.config.Pickup then status.Pickup="Off";return end
    local _,hum,root=currentBody()
    if not root or not hum or hum.Health<=0 or LocalPlayer:GetAttribute("IsInPassiveMode") then status.Pickup="Character unavailable or passive";return end
    local folder=Workspace:FindFirstChild("DroppedItems")
    if not folder then status.Pickup="Waiting for dropped items";return end
    for _,item in ipairs(folder:GetChildren()) do
        if item:IsA("Model") and item.PrimaryPart and not item:GetAttribute("Locked") and
            (item:GetPivot().Position-root.Position).Magnitude<=self.config.PickupRange and
            os.clock()-(self.pickupTimes[item] or -math.huge)>=2 then
            self.pickupTimes[item]=os.clock()
            local accepted,reason=self:Get("pickup_dropped_item",item)
            status.Pickup=accepted and "Server accepted pickup" or tostring(reason or "Pickup rejected")
            if not accepted and reason=="NoSpace" then self.pickupBlockedUntil=os.clock()+5 end
            return
        end
    end
    status.Pickup="Waiting for nearby drops"
end
function Features:PullCar()
    local _,_,root=currentBody()
    if not root then return false,"Character unavailable" end
    local folder=Workspace:FindFirstChild("Vehicles")
    if not folder then return false,"No spawned vehicles" end
    for _,car in ipairs(folder:GetChildren()) do
        if car:IsA("Model") and car:GetAttribute("OwnerUserId")==LocalPlayer.UserId then
            for _,seat in ipairs(car:GetDescendants()) do
                if (seat:IsA("Seat") or seat:IsA("VehicleSeat")) and seat.Occupant then return false,"All occupants must leave the car first" end
            end
            local params=OverlapParams.new();params.FilterType=Enum.RaycastFilterType.Exclude
            params.FilterDescendantsInstances={car,LocalPlayer.Character}
            local box,size=car:GetBoundingBox()
            local boxOffset=car:GetPivot():ToObjectSpace(box)
            local ray=RaycastParams.new();ray.FilterType=Enum.RaycastFilterType.Exclude
            ray.FilterDescendantsInstances={folder,LocalPlayer.Character};ray.RespectCanCollide=true
            for i=0,7 do
                local around=root.CFrame*CFrame.Angles(0,i*math.pi/4,0)*CFrame.new(0,0,-math.max(12,size.Z))
                local hit=Workspace:Raycast(around.Position+Vector3.new(0,8,0),Vector3.new(0,-30,0),ray)
                if hit and hit.Normal.Y>0.8 then
                    local center=Vector3.new(around.Position.X,hit.Position.Y+size.Y/2+0.5,around.Position.Z)
                    local destination=CFrame.new(center)*around.Rotation
                    local blocked=false
                    for _,part in ipairs(Workspace:GetPartBoundsInBox(destination,size,params)) do
                        if part.CanCollide then blocked=true;break end
                    end
                    if not blocked then
                        car:PivotTo(destination*boxOffset:Inverse())
                        return true,"Local recall requested; server position must be checked"
                    end
                end
            end
            return false,"No clear parking space nearby"
        end
    end
    return false,"Spawn your owned car first"
end
function Features:ForceSpin()
    local ui=self:Module("Game.CrateSystem.CrateUI")
    local crate=self:Module("Game.CrateSystem.Crate")
    if self.spinningRequest or crate.spinning.get() or ui.awaiting_server_response.get() then return false,"Spin already active" end
    local chosen=ui.chosen_crate.get()
    if not chosen or not chosen.Parent then return false,"Select a crate in the game first" end
    local method=ui.purchase_method.get()
    if method~="money" and method~="token" then return false,"Only in-game money/token purchases are supported" end
    local data=self:Module("Core.Data")
    if method=="money" and data.money.hand<(chosen:GetAttribute("price") or math.huge) then return false,"Not enough cash in hand" end
    self.spinningRequest=true
    local called,accepted,reward=pcall(self.Get,self,"open_crate",chosen,method)
    self.spinningRequest=false
    if not called then return false,tostring(accepted) end
    if not accepted then return false,tostring(reward or "Server rejected crate purchase") end
    local parent=chosen.Parent.Parent
    local _,hum=currentBody()
    if hum then hum:UnequipTools() end
    ui.close()
    crate.spin(crate.class.get(parent),accepted,chosen)
    return true,"Server accepted crate purchase"
end
function Features:OpenShop()
    local shops=self:Module("Game.ConsumableShop")
    local ui=self:Module("Game.UI.ConsumableBuyUI")
    local _,_,root=currentBody()
    if not root then return false,"Character unavailable" end
    local nearest,distance=nil,15
    for _,object in pairs(shops.class.objects) do
        local instance=object.instance
        if instance and instance.Parent then
            local d=(instance:GetPivot().Position-root.Position).Magnitude
            if d<distance then nearest,distance=object,d end
        end
    end
    if not nearest then return false,"Go within 15 studs of a shop first" end
    local level=nearest.states.unlock_level.get()
    if level>0 then
        local xp=self:Module("Game.Skills.XPFormula").xp_to_level(self:Module("Core.Data").xp.total_level or 0)
        if xp<level then return false,"Shop level locked: "..level end
    end
    ui.shop_name.set(nearest.states.shop_name.get());ui.shop_instance.set(nearest.instance)
    ui.shop_type.set(nearest.states.shop_type.get());ui.enabled.set(true)
    return true,"Native shop opened; select items there"
end
function Features:OwnedItems()
    local results={}
    for category,names in pairs(self:Module("Core.Data").items or {}) do
        for name,list in pairs(names) do for _,item in pairs(list) do
            if type(item)=="table" and item.guid and category~="ammo" and category~="fish" then
                local entry=table.clone(item)
                entry.item_type=entry.item_type or category
                entry.name=entry.name or name
                results[name.." ["..item.guid:sub(2,9).."]"]=entry
            end
        end end
    end
    return results
end
function Features:ApplyCosmetic(itemGuid,cosmeticGuid)
    if not itemGuid or not cosmeticGuid then return false,"Select an item and cosmetic first" end
    return self:Get("toggle_equip_cosmetic",itemGuid,cosmeticGuid)
end
task.spawn(function()
    while isRunning do
        local ok,err=pcall(function()
            Features:UpdateReloads()
            if os.clock()>=(Features.pickupBlockedUntil or 0) then Features:Pickup() end
        end)
        status.FeatureError=not ok and tostring(err) or nil
        task.wait(0.2)
    end
    Features:RestoreReloads()
end)
track({Disconnect=function() Features:RestoreAuras() end})
task.spawn(function()
    local nextAttack={Melee=0,Throw=0}
    local busy={Melee=false,Throw=false}
    while isRunning do
        local ok,err=pcall(function()
            for _,kind in ipairs({"Melee","Throw"}) do
                if Features.config[kind] and not busy[kind] and os.clock()>=nextAttack[kind] and Features:AuraAllowed() then
                    local module=Features:Module(kind=="Melee" and "Game.ItemTypes.Melee" or "Game.ItemTypes.Throwable")
                    local tool=(kind=="Melee" and module.equipped_melee or module.equipped_throwable).get()
                    if not tool then status[kind.."Aura"]="Equip a "..kind.." tool"
                    else
                        local range=Features.config[kind.."Range"]
                        local target=Features:Nearest(range)
                        if target then
                            nextAttack[kind]=os.clock()+math.max(kind=="Melee" and 0.1 or (tool:GetAttribute("ThrowDebounceTime") or 0.3),0.1)
                            busy[kind]=true
                            task.spawn(function()
                                local success,accepted,reason=pcall(function()
                                    if not Features.config[kind] or not Features:AuraAllowed() or tool.Parent~=LocalPlayer.Character then return false,"Action cancelled" end
                                    return Features:ActivateAt(target,kind,tool)
                                end)
                                status[kind.."Aura"]=success and (accepted and "Native action requested; damage unconfirmed" or reason) or tostring(accepted)
                                busy[kind]=false
                            end)
                        else status[kind.."Aura"]="Waiting for target in configured range" end
                    end
                elseif not Features.config[kind] then status[kind.."Aura"]="Off"
                elseif not Features:AuraAllowed() then status[kind.."Aura"]="Paused: safe zone, character state or consumption" end
            end
        end)
        status.AuraError=not ok and tostring(err) or nil
        task.wait(0.05)
    end
    Features:RestoreAuras()
end)

local function finalizeVehicleLock(dt)
    status.VehicleCameraFrames = (status.VehicleCameraFrames or 0) + 1
    if not isRunning or not isAimLockHeld() or not cameraLockTarget then return end
    local cam = Workspace.CurrentCamera
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not cam or not hum or hum.Health <= 0 or not isVehicleAttached(hum)
        or cam.CameraType == Enum.CameraType.Scriptable or not validLockTarget(cameraLockTarget) then return end
    local head = getAimPart(cameraLockTarget)
    if head and (head.Position - cam.CFrame.Position).Magnitude > 0.001 then
        updateVehicleLock(cam, head, dt)
        local offset = head.Position - cam.CFrame.Position
        local forward = offset:Dot(cam.CFrame.LookVector)
        status.AimLockRayMiss = (offset - cam.CFrame.LookVector * forward).Magnitude
    end
end

RunService:BindToRenderStep("MaxGanHubVehicleAimLock", Enum.RenderPriority.Last.Value + 4, function(dt)
    local ok, err = pcall(finalizeVehicleLock, dt)
    status.VehicleCameraError = not ok and tostring(err) or nil
end)

local overlayGui=Instance.new("ScreenGui")
overlayGui.Name="MaxGanCombatOverlay"
overlayGui.IgnoreGuiInset=true
overlayGui.ScreenInsets=Enum.ScreenInsets.None
overlayGui.ClipToDeviceSafeArea=false
overlayGui.ResetOnSpawn=false
overlayGui.DisplayOrder=20
overlayGui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling
overlayGui.Parent=LocalPlayer:WaitForChild("PlayerGui")
track({Disconnect=function() overlayGui:Destroy() end})
local Overlay={}
function Overlay.new(kind)
    local node=Instance.new(kind=="Text" and "TextLabel" or "Frame")
    node.Name=kind
    node.BackgroundTransparency=1
    node.BorderSizePixel=0
    node.Visible=false
    node.Parent=overlayGui
    local stroke
    if kind=="Text" then
        node.Font=Enum.Font.SourceSansSemibold
        node.TextSize=13
        node.TextStrokeTransparency=0
        node.TextStrokeColor3=Color3.new(0,0,0)
        node.TextXAlignment=Enum.TextXAlignment.Center
        node.TextYAlignment=Enum.TextYAlignment.Top
        node.Size=UDim2.fromOffset(600,24)
        node.AnchorPoint=Vector2.new(.5,0)
    else
        stroke=Instance.new("UIStroke")
        stroke.ApplyStrokeMode=Enum.ApplyStrokeMode.Border
        stroke.Parent=node
        if kind=="Circle" then
            local corner=Instance.new("UICorner")
            corner.CornerRadius=UDim.new(1,0)
            corner.Parent=node
            node.AnchorPoint=Vector2.new(.5,.5)
        end
    end
    local values={Visible=false,Position=Vector2.zero,Radius=0,Transparency=1,Thickness=1}
    local removed=false
    local methods={Remove=function() if not removed then removed=true;node:Destroy() end end}
    return setmetatable({}, {
        __index=function(_,key) return methods[key] or values[key] end,
        __newindex=function(_,key,value)
            if removed or values[key]==value then return end
            values[key]=value
            if key=="Visible" then node.Visible=value
            elseif key=="Position" then
                local origin=overlayGui.AbsolutePosition
                node.Position=UDim2.fromOffset(value.X-origin.X,value.Y-origin.Y)
            elseif key=="Radius" then node.Size=UDim2.fromOffset(value*2,value*2)
            elseif key=="Size" then
                if kind=="Text" then node.TextSize=value;node.Size=UDim2.fromOffset(600,value+7)
                else node.Size=UDim2.fromOffset(value.X,value.Y) end
            elseif key=="Color" then
                if kind=="Text" then node.TextColor3=value else stroke.Color=value;node.BackgroundColor3=value end
            elseif key=="Text" then node.Text=value
            elseif key=="Thickness" and stroke then stroke.Thickness=value
            elseif key=="Outline" and kind=="Text" then node.TextStrokeTransparency=value and 0 or 1
            elseif key=="Center" and kind=="Text" then node.AnchorPoint=Vector2.new(value and .5 or 0,0)
            elseif key=="Transparency" then
                if kind=="Text" then node.TextTransparency=1-value else stroke.Transparency=1-value end
            elseif key=="Filled" and kind~="Text" then node.BackgroundTransparency=value and 0 or 1 end
        end,
    })
end
status.OverlayRenderer="Roblox GUI"

local hasDrawing=true

if hasDrawing then
    local fovCircle = Overlay.new("Circle")
    fovCircle.Visible = false
    fovCircle.Filled = false
    fovCircle.Thickness = 1.5
    fovCircle.Color = Color3.fromRGB(180, 125, 245)
    fovCircle.Transparency = 1
    DrawObjects[#DrawObjects + 1] = fovCircle
    local targetRing = Overlay.new("Circle")
    targetRing.Visible = false
    targetRing.Filled = false
    targetRing.Radius = AimConfig.TargetMarkerRadius
    targetRing.Thickness = 2
    targetRing.Color = Color3.fromRGB(255, 230, 0)
    targetRing.Transparency = 1
    DrawObjects[#DrawObjects + 1] = targetRing
    local espCache = {}
    local rarityColors = {
        Common = Color3.fromRGB(220, 220, 220),
        Uncommon = Color3.fromRGB(90, 220, 125),
        Rare = Color3.fromRGB(51, 170, 255),
        Epic = Color3.fromRGB(210, 100, 255),
        Legendary = Color3.fromRGB(255, 200, 65),
        Omega = Color3.fromRGB(255, 90, 90),
    }
    local itemDefinitions = {}
    local meshDefinitions = {}
    local function handleMesh(item)
        local handle = item:FindFirstChild("Handle", true)
        if not handle then return nil end
        if handle:IsA("MeshPart") then return handle.MeshId end
        local mesh = handle:FindFirstChildOfClass("SpecialMesh")
        return mesh and mesh.MeshId or nil
    end
    local itemsFolder = ReplicatedStorage:FindFirstChild("Items")
    if itemsFolder then
        for _, item in ipairs(itemsFolder:GetDescendants()) do
            if item:IsA("Tool") or item:IsA("Model") then
                itemDefinitions[item.Name] = item
                local mesh = handleMesh(item)
                if mesh and mesh ~= "" then
                    local previous = meshDefinitions[mesh]
                    if previous == nil then
                        meshDefinitions[mesh] = item
                    elseif previous and previous.Name ~= item.Name then
                        meshDefinitions[mesh] = false
                    end
                end
            end
        end
    end

    local function newText(size, color)
        local t = Overlay.new("Text")
        t.Size = size
        t.Center = true
        t.Outline = true
        t.Color = color
        t.Visible = false
        DrawObjects[#DrawObjects + 1] = t
        return t
    end

    local function createESP(player)
        if player == LocalPlayer or espCache[player] then return end
        local box = Overlay.new("Square")
        box.Thickness = 1.5
        box.Filled = false
        box.Color = Color3.fromRGB(255, 55, 55)
        box.Visible = false
        DrawObjects[#DrawObjects + 1] = box
        espCache[player] = {
            box = box,
            name = newText(13, Color3.fromRGB(255, 255, 255)),
            dist = newText(11, Color3.fromRGB(200, 200, 200)),
            hp = newText(11, Color3.fromRGB(80, 255, 80)),
            inventory = {},
            nextInventoryUpdate = 0,
        }
    end

    local function removeESP(player)
        local e = espCache[player]
        if e then
            for _, key in ipairs({"box", "name", "dist", "hp"}) do
                pcall(function() e[key]:Remove() end)
            end
            for _, row in ipairs(e.inventory) do pcall(function() row:Remove() end) end
            espCache[player] = nil
        end
    end

    local function hide(e)
        e.box.Visible = false
        e.name.Visible = false
        e.dist.Visible = false
        e.hp.Visible = false
        for _, row in ipairs(e.inventory) do row.Visible = false end
    end

    local function updateInventory(player, e, now)
        if now < e.nextInventoryUpdate then return end
        e.nextInventoryUpdate = now + 0.5
        local entries, grouped, seen = {}, {}, {}
        local function readContainer(container)
            if not container then return end
            for _, tool in ipairs(container:GetChildren()) do
                if tool:IsA("Tool") and not seen[tool] then
                    seen[tool] = true
                    local mesh = handleMesh(tool)
                    local definition = itemDefinitions[tool.Name] or (mesh and meshDefinitions[mesh])
                    local display = tool:GetAttribute("DisplayName")
                        or (definition and (definition:GetAttribute("DisplayName") or definition.Name))
                        or tool.Name
                    local name = tool.Name:lower()
                    local displayName = tostring(display):lower()
                    local isFist = name == "fist" or name == "fists" or displayName == "fist"
                        or displayName == "fists" or (tool:GetAttribute("Locked") == true
                            and tool:GetAttribute("cone_angle") == 60
                            and tool:GetAttribute("range") == 5
                            and tool:GetAttribute("AmmoType") == nil
                            and tool:GetAttribute("RarityName") == nil)
                    if not isFist then
                        local rarity = tool:GetAttribute("RarityName")
                            or (definition and definition:GetAttribute("RarityName")) or "Unknown"
                        local key = tostring(rarity) .. ":" .. tostring(display)
                        local entry = grouped[key]
                        if entry then
                            entry.count = entry.count + 1
                        else
                            entry = { name = tostring(display), rarity = tostring(rarity), count = 1 }
                            grouped[key] = entry
                            entries[#entries + 1] = entry
                        end
                    end
                end
            end
        end
        readContainer(player.Character)
        readContainer(player:FindFirstChildOfClass("Backpack"))
        table.sort(entries, function(a, b)
            if a.rarity ~= b.rarity then return a.rarity < b.rarity end
            return a.name < b.name
        end)
        for i, entry in ipairs(entries) do
            local row = e.inventory[i]
            if not row then
                row = newText(16, Color3.new(1, 1, 1))
                e.inventory[i] = row
            end
            row.Text = entry.name .. " [" .. entry.rarity .. "]"
                .. (entry.count > 1 and (" x" .. entry.count) or "")
            row.Color = rarityColors[entry.rarity] or Color3.fromRGB(220, 220, 220)
        end
        for i = #e.inventory, #entries + 1, -1 do
            pcall(function() e.inventory[i]:Remove() end)
            e.inventory[i] = nil
        end
    end

    for _, p in ipairs(Players:GetPlayers()) do createESP(p) end
    track(Players.PlayerAdded:Connect(createESP))
    track(Players.PlayerRemoving:Connect(removeESP))

    RunService:BindToRenderStep("MaxGanHubESP", Enum.RenderPriority.Last.Value + 5, function(dt)
        if not isRunning then return end
        local cam = Workspace.CurrentCamera
        if not cam then return end
        fovCircle.Position = getAimScreenPoint(cam)
        fovCircle.Radius = AimConfig.FOV
        fovCircle.Visible = AimConfig.ShowFOV
        status.FOVDrawing = {visible=fovCircle.Visible, radius=fovCircle.Radius, position=tostring(fovCircle.Position)}
        targetRing.Radius = AimConfig.TargetMarkerRadius
        targetRing.Visible = false
        if aimLockEnabled and AimConfig.Mode ~= "Aim" and lockedTarget and targetInFOV(lockedTarget) then
            local part = getAimPart(lockedTarget)
            local point, visible = cam:WorldToViewportPoint(part.Position)
            targetRing.Position = Vector2.new(point.X, point.Y)
            targetRing.Visible = visible and AimConfig.ShowFOV
        end
        local myChar = LocalPlayer.Character
        local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")

        for player, e in pairs(espCache) do
            if isValid(player) then
                local c = getParts(player)
                local root, head, hum = c.root, c.head, c.hum
                if root and head then
                    local rootPos, on = cam:WorldToViewportPoint(root.Position)
                    local dist = myRoot and (myRoot.Position - root.Position).Magnitude or 0
                    if ESPConfig.Enabled and on and dist <= MAX_ESP_DIST then
                        local top = cam:WorldToViewportPoint(head.Position + Vector3.new(0, 0.6, 0))
                        local bot = cam:WorldToViewportPoint(root.Position - Vector3.new(0, 2.9, 0))
                        local topY = math.min(top.Y, bot.Y)
                        local botY = math.max(top.Y, bot.Y)
                        local h = math.max(botY - topY, 4)
                        local w = h * 0.65
                        local locked = (lockedTarget == player)

                        e.box.Size = Vector2.new(w, h)
                        e.box.Position = Vector2.new(rootPos.X - w / 2, topY)
                        e.box.Color = locked and Color3.fromRGB(255, 230, 0) or Color3.fromRGB(255, 55, 55)
                        e.box.Thickness = locked and 2.5 or 1.5
                        e.box.Visible = ESPConfig.Box

                        e.name.Text = player.Name
                        e.name.Position = Vector2.new(rootPos.X, topY - 16)
                        e.name.Color = locked and Color3.fromRGB(255, 230, 0) or Color3.fromRGB(255, 255, 255)
                        e.name.Visible = ESPConfig.Name
                        updateInventory(player, e, os.clock())
                        for i, row in ipairs(e.inventory) do
                            row.Position = Vector2.new(rootPos.X, topY - 18 - (#e.inventory - i + 1) * 20)
                            row.Visible = ESPConfig.Inventory and row.Position.Y >= 0
                        end

                        e.dist.Text = "[" .. math.floor(dist) .. " studs]"
                        e.dist.Position = Vector2.new(rootPos.X, botY + 2)
                        e.dist.Visible = ESPConfig.Distance

                        local hp = math.floor(hum and hum.Health or 0)
                        e.hp.Text = hp .. " HP"
                        e.hp.Position = Vector2.new(rootPos.X, botY + 15)
                        e.hp.Color = hp > 50 and Color3.fromRGB(80, 255, 80)
                            or (hp > 25 and Color3.fromRGB(255, 170, 0) or Color3.fromRGB(255, 60, 60))
                        e.hp.Visible = ESPConfig.Health
                    else
                        hide(e)
                    end
                else
                    hide(e)
                end
            else
                hide(e)
            end
        end
    end)
end

task.spawn(function()
    local ok, store = pcall(function()
        return requireGameModule(ReplicatedStorage.Modules.Game.Store.StoreUI)
    end)
    if not ok then
        status.PickupRecovery = "Game module unavailable"
        status.PickupRecoveryError = tostring(store)
        return
    end
    local coreGui = game:GetService("CoreGui")
    local purchaseRootNames = { "PurchasePromptApp", "BulkPurchaseApp", "CommercePurchaseApp" }
    local purchaseTextCache = setmetatable({}, { __mode = "k" })
    local function purchaseTexts(root)
        local cached = purchaseTextCache[root]
        if not cached then
            cached = { dirty = true }
            purchaseTextCache[root] = cached
            track(root.DescendantAdded:Connect(function() cached.dirty = true end))
            track(root.DescendantRemoving:Connect(function() cached.dirty = true end))
        end
        if cached.dirty then
            cached.texts = {}
            for _, object in ipairs(root:GetDescendants()) do
                if object:IsA("TextLabel") or object:IsA("TextButton") then
                    cached.texts[#cached.texts + 1] = object
                end
            end
            cached.dirty = false
        end
        return cached.texts
    end
    local function purchasePromptVisible()
        for _, name in ipairs(purchaseRootNames) do
            local root = coreGui:FindFirstChild(name)
            if root and (not root:IsA("ScreenGui") or root.Enabled) then
                for _, object in ipairs(purchaseTexts(root)) do
                    if object.Parent and object.Text ~= "" then
                        local current, visible = object, true
                        while current and current ~= root do
                            if (current:IsA("GuiObject") and not current.Visible)
                                or (current:IsA("ScreenGui") and not current.Enabled) then
                                visible = false
                                break
                            end
                            current = current.Parent
                        end
                        if visible then return true end
                    end
                end
            end
        end
        return false
    end
    local absentSince, pendingProduct, pendingSku
    while isRunning do
        local checked, err = pcall(function()
            local blocked = store.prompt_open.get()
            status.PickupGateBlocked = blocked
            if not blocked then
                absentSince, pendingProduct, pendingSku = nil, nil, nil
                status.PickupRecovery = "Pickup gate open"
                return
            end
            if store.enabled.get() or store.is_gift_mode.get()
                or LocalPlayer:GetAttribute("is_server_purchasing") == true
                or purchasePromptVisible() then
                absentSince, pendingProduct, pendingSku = nil, nil, nil
                status.PickupRecovery = "Purchase active"
                return
            end
            local product = store.prompt_open_product_id.get()
            local sku = store.prompt_open_purchase_sku.get()
            local now = os.clock()
            if not absentSince or product ~= pendingProduct or sku ~= pendingSku then
                absentSince, pendingProduct, pendingSku = now, product, sku
            end
            local knownEmpty = product == -1 and sku == ""
            if not knownEmpty and now - absentSince < 0.75 then
                status.PickupRecovery = "Checking closed purchase"
                return
            end
            if not isRunning or store.enabled.get() or store.is_gift_mode.get()
                or LocalPlayer:GetAttribute("is_server_purchasing") == true
                or store.prompt_open_product_id.get() ~= product
                or store.prompt_open_purchase_sku.get() ~= sku then
                absentSince = nil
                return
            end
            store.prompt_open.set(false)
            local confirmed = not store.prompt_open.get()
            status.PickupRecoveryConfirmed = confirmed
            status.PickupGateBlocked = not confirmed
            if confirmed then
                status.PickupRecovery = "Pickup gate restored"
                status.PickupRecoveryCount = (status.PickupRecoveryCount or 0) + 1
                status.PickupRecoveryLastAt = now
                status.PickupRecoveryLastSku = sku
                absentSince, pendingProduct, pendingSku = nil, nil, nil
            else
                status.PickupRecovery = "Reset not confirmed"
            end
        end)
        if not checked then
            absentSince, pendingProduct, pendingSku = nil, nil, nil
            status.PickupRecovery = "Check unavailable"
            status.PickupRecoveryError = tostring(err)
        end
        task.wait(0.1)
    end
end)

print("[MaxGan Hub] " .. status.Build .. " | " .. status.ExecutorCompatibility .. " | Block Spin loaded - Head Aim Lock (F toggle), ESP active.")
]]):gsub('.+', (function(a) _UiIKV_Q_owqC = a; end)); return(function(a,...)local d;local r;local h;local o;local f;local l;local e=24915;local n=0;local t={};while n<262 do n=n+1;while n<0x28c and e%0x3600<0x1b00 do n=n+1 e=(e*655)%21720 local c=n+e if(e%0x1b78)>0xdbc then e=(e*0x332)%0x2af1 while n<0x327 and e%0x268e<0x1347 do n=n+1 e=(e*544)%17770 local d=n+e if(e%0x4b68)<=0x25b4 then e=(e-0xbb)%0xa8ee local e=46756 if not t[e]then t[e]=0x1 r=string;end elseif e%2~=0 then e=(e*0x38e)%0x215d local e=55375 if not t[e]then t[e]=0x1 end else e=(e-0x322)%0x51b0 n=n+1 local e=46092 if not t[e]then t[e]=0x1 o={};end end end elseif e%2~=0 then e=(e-0x23a)%0xbdcb while n<0xc7 and e%0x8de<0x46f do n=n+1 e=(e+809)%9416 local l=n+e if(e%0x25c6)>0x12e3 then e=(e-0x2aa)%0x8fa4 local e=14991 if not t[e]then t[e]=0x1 d=getfenv and getfenv();end elseif e%2~=0 then e=(e-0x101)%0x8a93 local e=20389 if not t[e]then t[e]=0x1 end else e=(e-0x2ff)%0x8d36 n=n+1 local e=99457 if not t[e]then t[e]=0x1 f="\4\8\116\111\110\117\109\98\101\114\109\122\115\106\118\84\108\83\0\6\115\116\114\105\110\103\4\99\104\97\114\86\72\82\69\89\99\82\74\0\6\115\116\114\105\110\103\3\115\117\98\81\113\90\68\100\106\88\82\0\6\115\116\114\105\110\103\4\98\121\116\101\77\109\76\76\104\120\122\73\0\5\116\97\98\108\101\6\99\111\110\99\97\116\87\104\111\70\88\122\75\111\0\5\116\97\98\108\101\6\105\110\115\101\114\116\99\110\118\81\88\105\70\108\5";end end end else e=(e-0x91)%0x46de n=n+1 while n<0x2e0 and e%0x2fbc<0x17de do n=n+1 e=(e-463)%21734 local f=n+e if(e%0x2efa)<0x177d then e=(e*0x7a)%0x4ccf local e=76410 if not t[e]then t[e]=0x1 d=(not d)and _ENV or d;end elseif e%2~=0 then e=(e-0xe8)%0x2194 local e=76947 if not t[e]then t[e]=0x1 h=tonumber;end else e=(e+0x2d1)%0x98d3 n=n+1 local e=17436 if not t[e]then t[e]=0x1 l=function(l)local e=0x01 local function t(n)e=e+n return l:sub(e-n,e-0x01)end while true do local n=t(0x01)if(n=="\5")then break end local e=r.byte(t(0x01))local e=t(e)if n=="\2"then e=o.mzsjvTlS(e)elseif n=="\3"then e=e~="\0"elseif n=="\6"then d[e]=function(n,e)return a(8,nil,a,e,n)end elseif n=="\4"then e=d[e]elseif n=="\0"then e=d[e][t(r.byte(t(0x01)))];end local n=t(0x08)o[n]=e end end end end end end end e=(e*341)%16017 end l(f);local n={};for e=0x0,0xff do local t=o.VHREYcRJ(e);n[e]=t;n[t]=e;end local function c(e)return n[e];end local r=(function(a,l)local f,t=0x01,0x10 local n={{},{},{}}local d=-0x01 local e=0x01 local r=a while true do n[0x03][o.QqZDdjXR(l,e,(function()e=f+e return e-0x01 end)())]=(function()d=d+0x01 return d end)()if d==(0x0f)then d=""t=0x000 break end end local d=#l while e<d+0x01 do n[0x02][t]=o.QqZDdjXR(l,e,(function()e=f+e return e-0x01 end)())t=t+0x01 if t%0x02==0x00 then t=0x00 o.cnvQXiFl(n[0x01],(c((((n[0x03][n[0x02][0x00]]or 0x00)*0x10)+(n[0x03][n[0x02][0x01]]or 0x00)+r)%0x100)));r=a+r;end end return o.WhoFXzKo(n[0x01])end);l(r(12,".;rat<JF^7PKAHI227"));l(r(72,"H)dtR%LiuQ?vJ=h;vuiLQv%R9?vQiLdiiJQR%=Z=Ju%t#Q=uQhLt^=v2ii=Ru=vs%v))=RuJd;-?QvRtd=v%Q?dLQQ%%iutdh)Qh%JhtQQ%uRd;%Qht;h?LE)hRt;JvdL))hJ;d%R%;)vJLtd<vRLJdvhRv%%;Gt=;tshJo=vQuRt/u%?R%Rs==uuud?hn?R%i;L?=LiRdQR%Rit)=hRQ%R%XJv)iJd)=?QJR=))J%L)t;h;utRi;?LRdARR;ivtLJdd=)uht;;??hRi))*****;RQ;1%oO=tR=QQd%Rl%%hu%tL;WQQ%t)tJ)ihtQhd?=Lt;i=?dR=hc%vLi1)Q=tQtdLC?JuRJOQhd?5tQ?R%=uCt?hQ?LLRhv=Lu%Rt;Ru?RJ;uLJdLRv;%vRL%)vhRid%)=vJ%%ddQ=)iQhu?RJ%i)tJhut=%L)RJ;uiRd;;?uRh7QvRLL%-h%R="));local e=(-4731+(function()local l,n=0,1;(function(t,d,n,e)t(d(n,n,e,n),e(e,d and e,t,e),t(n,e,t and n,n),d(n,e,t,n)and e(e,e,n,d))end)(function(t,e,o,d)if l>348 then return o end l=l+1 n=(n+714)%14261 if(n%420)>=210 then n=(n+321)%19461 return e else return t(e(t,o and t,d,e),t(o,e,o,d),d(e,e,e,e and t)and e(o,d,d,t),d(t,d,o,t))end return e(d(t and d,e,t and o,o and e),t(o and d,t,t,e),t(d and e,o,t,o),o(e,d,e and d,d))end,function(o,t,d,e)if l>159 then return e end l=l+1 n=(n+813)%37701 if(n%662)<=331 then n=(n-993)%12456 return e(t(e,e,t,d),o(o,e,t,e),e(d and d,t,e,e),d(e,t,o,t))else return o end return d end,function(d,t,e,o)if l>342 then return e end l=l+1 n=(n+14)%28517 if(n%1412)>706 then return e else return e(e(o,t,o,e)and t(e,o,o,d),d(d,t,e,o),t(d,e,t,e),d(e,e,d,t))end return e(e(e and d,e,e and d,e),d(t,t,e,e),d(d,d,t and d,e),o(t,t,t and t,o))end,function(e,d,o,t)if l>269 then return e end l=l+1 n=(n+332)%31038 if(n%1216)>=608 then n=(n+807)%19940 return d else return t(o(d,e,e,t),o(o,t,e,o and e),d(d,t and e,o,o),t(t,d,t and e,e))end return t(e(t and e,e,e,t),d(d,o,o,d),e(t and t,d and d,e,d)and t(o and o,e,e,d),d(t,o,t and d,e))end)return n;end)())local g=(getfenv)or(function()return _ENV end);local _=o.FrxDIjpQ or o.afiZmkGr;local t=3;local l=2;local ee=1;local c=4;local function p(b,...)local u=r(e,"89s(Jypnb_oQ51{,n_{by51>Jn5ss5oj9J,yp9_Ey9Q_Jypf9pb1+Q9(_1,bp5{fo9(noQ9{nn,1p,1,Jn15J{Q_(s_,9,pb,1p5{&yy1s9_5d(JopoQs__&UJnbJQQy(1_9:nbpw(p_{Qp,{Qy__,1(pQQbs5o-pb,jpy{(SJQ{(yo59_Jb9(n9,n9b15ys5((5o99(_b,1nGoUpQ19ywQb{Jby9Qb,usyQ{9y(bJs{n,5((_ob95_#nJ{oyy5{JbysQ5(loJ9bb1SDnJ{by11&JJQb({Qy9Jbb,5ns{Jyb55ys17sb_59Vbn,bp5{Ano5Q(5QEsJ_{j5b9,Jp515yJ5,(bo5s+_oGbn1,0pJ{sJ55q(Jo_95_MdJbo{5pN1JJb59(=oJ9b_{VsnJ{_y519JJQbs5o6sbbb,5n&{pyb51JFQJ(9_59LbJ,_p5{LyJ5b(5Q!sJ_b");local n=0;o.WzSCuErF(function()n=n+1 end)local function e(e,t)if t then return n end;n=e+n;end local d,n,f=a(0,a,e,u,o.MmLLhxzI);local function r()local n,t=o.MmLLhxzI(u,e(1,3),e(5,6)+2);e(2);return(t*256)+n;end;local s=true;local s=0 local function j()local e=n();local n=n();local l=1;local t=(d(n,1,20)*(2^32))+e;local e=d(n,21,31);local n=((-1)^d(n,32));if(e==0)then if(t==s)then return n*0;else e=1;l=0;end;elseif(e==2047)then return(t==0)and(n*(1/0))or(n*(0/0));end;return o.ZXd_FEGj(n,e-1023)*(l+(t/(2^52)));end;local p=n;local function m(n)local t;if(not n)then n=p();if(n==0)then return'';end;end;t=o.QqZDdjXR(u,e(1,3),e(5,6)+n-1);e(n)local e=""for n=(1+s),#t do e=e..o.QqZDdjXR(t,n,n)end return e;end;local p=#o.jfGYxWiX(h('\49.\48'))~=1 local e=n;local function y(...)return{...},o.JqCuJiYI('#',...)end local function k()local s={};local e={};local u={};local h={s,u,nil,e};local e=n()local a={}for d=1,e do local t=f();local e;if(t==2)then e=(f()~=#{});elseif(t==0)then local n=j();if p and o.DDKLeWCg(o.jfGYxWiX(n),'.(\48+)$')then n=o.whVYLFxU(n);end e=n;elseif(t==3)then e=m();end;a[d]=e;end;for u=1,n()do local e=f();if(d(e,1,1)==0)then local o=d(e,2,3);local f=d(e,4,6);local e={r(),r(),nil,nil};if(o==0)then e[t]=r();e[c]=r();elseif(o==#{1})then e[t]=n();elseif(o==b[2])then e[t]=n()-(2^16)elseif(o==b[3])then e[t]=n()-(2^16)e[c]=r();end;if(d(f,1,1)==1)then e[l]=a[e[l]]end if(d(f,2,2)==1)then e[t]=a[e[t]]end if(d(f,3,3)==1)then e[c]=a[e[c]]end s[u]=e;end end;for e=1,n()do u[e-(#{1})]=k();end;h[3]=f();return h;end;local function p(t,e,n)local d=e;local d=n;return h(o.DDKLeWCg(o.DDKLeWCg(({o.WzSCuErF(t)})[2],e),n))end local function z(h,e,f)local function g(...)local r,g,u,p,k,n,m,j,b,s,_,d;local e=0;while-1<e do if 2>=e then if 1<=e then if e>-1 then repeat if 2>e then u=a(6,97,3,52,h);k=y p=0;break;end;n=-41;m=-1;until true;else u=a(6,97,3,52,h);k=y p=0;end else r=a(6,91,1,22,h);g=a(6,15,2,41,h);end else if 5<=e then if 3~=e then repeat if 6~=e then d=a(7);break;end;e=-2;until true;else e=-2;end else if e~=1 then for n=44,64 do if e~=4 then j={};b={...};break;end;s=o.JqCuJiYI('#',...)-1;_={};break;end;else s=o.JqCuJiYI('#',...)-1;_={};end end end e=e+1;end;for e=0,s do if(e>=u)then j[e-u]=b[e+1];else d[e]=b[e+1];end;end;local e=s-u+1 local e;local o;local function a(...)while true do end end while true do if n<-40 then n=n+42 end e=r[n];o=e[ee];if 8>=o then if 3<o then if 6>o then if o~=3 then for r=46,84 do if o~=4 then f[e[t]]=d[e[l]];break;end;if(d[e[l]]~=e[c])then n=n+1;else n=e[t];end;break;end;else f[e[t]]=d[e[l]];end else if 7<=o then if 4~=o then for n=25,72 do if o~=7 then f[e[t]]=d[e[l]];break;end;d(e[l],e[t]);break;end;else d(e[l],e[t]);end else local e=e[l]d[e](d[e+1])end end else if o<=1 then if-1<o then repeat if o~=0 then for o=0,3 do if o<2 then if o~=-2 then for a=22,70 do if 0<o then f[e[t]]=d[e[l]];n=n+1;e=r[n];break;end;d[e[l]]=(e[t]~=0);n=n+1;e=r[n];break;end;else f[e[t]]=d[e[l]];n=n+1;e=r[n];end else if 2~=o then if(d[e[l]]~=e[c])then n=n+1;else n=e[t];end;else d[e[l]]=f[e[t]];n=n+1;e=r[n];end end end break;end;d[e[l]]=f[e[t]];until true;else for o=0,3 do if o<2 then if o~=-2 then for a=22,70 do if 0<o then f[e[t]]=d[e[l]];n=n+1;e=r[n];break;end;d[e[l]]=(e[t]~=0);n=n+1;e=r[n];break;end;else f[e[t]]=d[e[l]];n=n+1;e=r[n];end else if 2~=o then if(d[e[l]]~=e[c])then n=n+1;else n=e[t];end;else d[e[l]]=f[e[t]];n=n+1;e=r[n];end end end end else if o~=-1 then repeat if o~=3 then local o;d(e[l],e[t]);n=n+1;e=r[n];o=e[l]d[o](d[o+1])n=n+1;e=r[n];do return end;break;end;if(d[e[l]]~=e[c])then n=n+1;else n=e[t];end;until true;else if(d[e[l]]~=e[c])then n=n+1;else n=e[t];end;end end end else if 12<o then if o>14 then if o>15 then if o<17 then local e=e[l]d[e](d[e+1])else n=e[t];end else d[e[l]]=(e[t]~=0);end else if 14~=o then local o,r,a,c,f;local n=0;while n>-1 do if 2<n then if n>=5 then if 6>n then d(f,c);else n=-2;end else if 1<=n then repeat if 4~=n then c=o[a];break;end;f=o[r];until true;else f=o[r];end end else if 1<=n then if n~=1 then a=t;else r=l;end else o=e;end end n=n+1 end else do return end;end end else if o>=11 then if o>7 then for r=21,57 do if o~=12 then n=e[t];break;end;d[e[l]]=(e[t]~=0);break;end;else d[e[l]]=(e[t]~=0);end else if 9~=o then do return end;else d[e[l]]=f[e[t]];end end end end n=1+n;end;end;return g end;local d=0xff;local a={};local o=(1);local l='';(function(n)local t=n local r=0x00 local e=0x00 t={(function(d)if r>0x1f then return d end r=r+1 e=(e+0xc08-d)%0x23 return(e%0x03==0x0 and(function(t)if not n[t]then e=e+0x01 n[t]=(0x68);end return true end)'tAUcF'and t[0x3](0x35d+d))or(e%0x03==0x1 and(function(t)if not n[t]then e=e+0x01 n[t]=(0xc);end return true end)'ynzMZ'and t[0x2](d+0x92))or(e%0x03==0x2 and(function(t)if not n[t]then e=e+0x01 n[t]=(0xe5);end return true end)'AGaFZ'and t[0x1](d+0x9a))or d end),(function(f)if r>0x20 then return f end r=r+1 e=(e+0x1369-f)%0x45 return(e%0x03==0x1 and(function(t)if not n[t]then e=e+0x01 n[t]=(0x2f);end return true end)'vYRUC'and t[0x3](0x28c+f))or(e%0x03==0x0 and(function(t)if not n[t]then e=e+0x01 n[t]=(0x71);d[2]=(d[2]*(p(function()a()end,_(l))-p(d[1],_(l))))+1;a[o]={};d=d[2];o=o+d;end return true end)'boBmw'and t[0x2](f+0x1bc))or(e%0x03==0x2 and(function(t)if not n[t]then e=e+0x01 n[t]=(0xb5);a[o]=g();o=o+d;end return true end)'_mRpU'and t[0x1](f+0x2fc))or f end),(function(f)if r>0x2e then return f end r=r+1 e=(e+0x112f-f)%0x2c return(e%0x03==0x2 and(function(t)if not n[t]then e=e+0x01 n[t]=(0x51);end return true end)'aIbAO'and t[0x1](0x3ac+f))or(e%0x03==0x0 and(function(t)if not n[t]then e=e+0x01 n[t]=(0xca);l={l..'\58 a',l};a[o]=k();o=o+(1);l[1]='\58'..l[1];d[2]=0xff;end return true end)'WWclx'and t[0x2](f+0x367))or(e%0x03==0x1 and(function(t)if not n[t]then e=e+0x01 n[t]=(0xf8);l='\37';d={function()d()end};l=l..'\100\43';end return true end)'WLKda'and t[0x3](f+0xd0))or f end)}t[0x3](0xbf)end){};local e=z(_(a));return e(...);end return p((function()local n={}local e=0x01;local t;if o.CAXfseP_ then t=o.CAXfseP_(p)else t=''end if o.DDKLeWCg(t,o.CAHzufOB)then e=e+0;else e=e+1;end n[e]=0x02;n[n[e]+0x01]=0x03;return n;end)(),...)end)((function(t,e,n,d,l,o)local o;if 3<t then if t<6 then if t>0 then for o=21,61 do if t<5 then local t=d;local d,l,o=l(2);do return function()local e,n,r,f=e(n,t(t,t),t(t,t)+3);t(4);return(f*d)+(r*l)+(n*o)+e;end;end;break;end;local t=d;do return function()local e=e(n,t(t,t),t(t,t));t(1);return e;end;end;break;end;else local t=d;do return function()local e=e(n,t(t,t),t(t,t));t(1);return e;end;end;end else if t>=7 then if t>=5 then for e=18,68 do if 8>t then do return setmetatable({},{['__\99\97\108\108']=function(e,l,t,d,n)if n then return e[n]elseif d then return e else e[l]=t end end})end break;end;do return n(t,nil,n);end break;end;else do return setmetatable({},{['__\99\97\108\108']=function(e,d,t,l,n)if n then return e[n]elseif l then return e else e[d]=t end end})end end else do return l[n]end;end end else if 2>t then if 0~=t then do return function(n,e,t)if t then local e=(n/2^(e-1))%2^((t-1)-(e-1)+1);return e-e%1;else local e=2^(e-1);return(n%(e+e)>=e)and 1 or 0;end;end;end;else do return e(1),e(4,l,d,n,e),e(5,l,d,n)end;end else if t>=0 then repeat if t~=3 then do return 16777216,65536,256 end;break;end;do return e(1),e(4,l,d,n,e),e(5,l,d,n)end;until true;else do return e(1),e(4,l,d,n,e),e(5,l,d,n)end;end end end end),...).")# Blockspin5
