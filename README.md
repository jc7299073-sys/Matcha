--[[
    Matcha · Universal · Pro
    Skin Changer + full Spoofer
    UI custom 560x670 · yabujin borders · snow/blur on open
]]

if game.GameId ~= 6035872082 then
    warn("[Vnix.lol] Aviso: pensado para Rivals (6035872082). Actual: " .. tostring(game.GameId))
end
print("[Vnix.lol] loading... GameId=" .. tostring(game.GameId))
if not game:IsLoaded() then game.Loaded:Wait() end

function GS(n)
    local s = game:GetService(n)
    return cloneref and cloneref(s) or s
end

local Players            = GS("Players")
local ReplicatedStorage  = GS("ReplicatedStorage")
local ReplicatedFirst    = GS("ReplicatedFirst")
local ScriptContext      = GS("ScriptContext")
local CoreGui            = GS("CoreGui")
local MarketplaceService = GS("MarketplaceService")
local RunService         = GS("RunService")
local GuiService         = GS("GuiService")
local UserInputService   = GS("UserInputService")
local Workspace          = GS("Workspace")
local Lighting           = GS("Lighting")
local SoundService       = GS("SoundService")

local LocalPlayer = Players.LocalPlayer
local LocalChar

function GetCharacter()
    local c = LocalPlayer.Character
    if not c then
        local ok, ch = pcall(function()
            return LocalPlayer.CharacterAdded:Wait()
        end)
        c = ok and ch or LocalPlayer.Character
    end
    if c then
        pcall(function() c:WaitForChild("Humanoid", 3) end)
        pcall(function() c:WaitForChild("HumanoidRootPart", 3) end)
    end
    return c
end
LocalChar = LocalPlayer.Character or nil
task.spawn(function()
    LocalChar = GetCharacter()
end)
LocalPlayer.CharacterAdded:Connect(function(c)
    LocalChar = c
    pcall(function() c:WaitForChild("Humanoid", 5) end)
    pcall(function() c:WaitForChild("HumanoidRootPart", 5) end)
end)

pcall(function()
    local bypassed = false
    local kKickNames = {"Kick", "kick"}
    local kProtectedProperties = {Enabled = true, Disabled = false}
    local kSlotMap = {[69]=2,[138]=3,[207]=4,[276]=5,[345]=6,[414]=7}
    local kFilledSub = {1,2,3,4,5}

    local ac_script = ReplicatedFirst:WaitForChild("LocalScript3", 10)
    local ac_event  = ReplicatedStorage:WaitForChild("Remotes", 10):WaitForChild("RemoteEvent", 10)
    if not ac_script or not ac_event then return end

    local last, first_seen, hijack_ready, client_id
    local expected_interval, min_interval, ema_alpha, samples = 0.6, 0.25, 0.5, 0
    local hidden_fn, max_stack_depth = {}, 128

    setstackhidden = setstackhidden or function(fn_or_level, hidden)
        local ok, fn = pcall(function()
            if typeof(fn_or_level) == "number" then return debug.info(fn_or_level + 2, "f") end
            return fn_or_level
        end)
        if ok and fn then hidden_fn[fn] = not hidden end
    end

    local TrustedFunctions = setmetatable({}, {__mode = "k"})
    local HookReferences = {}
    local function TrustFunction(fn)
        if type(fn) == "function" then TrustedFunctions[fn] = true end
        return fn
    end

    local function SafeHook(hookfn, ...)
        local args = {...}
        local func, inst, metamethod, detour
        if hookfn == hookmetamethod then
            inst, metamethod, detour = args[1], args[2], args[3]
        else
            func, detour = args[1], args[2]
        end
        if hookfn == hookfunction and iscclosure(func) then detour = newcclosure(detour) end
        if not iscclosure(detour) then detour = newcclosure(detour) end
        pcall(setstackhidden, detour, true)
        HookReferences[#HookReferences + 1] = detour
        local original
        pcall(function()
            TrustFunction(detour)
            if hookfn == hookmetamethod then
                original = hookfn(inst, metamethod, detour)
            else
                original = hookfn(func, detour)
            end
        end)
        if original then HookReferences[#HookReferences + 1] = original end
        return original
    end

    local function SafeCall(func, ...)
        if checkcaller() then return func(...) end
        local old = getthreadidentity()
        if old ~= 2 then setthreadidentity(2) end
        local r = {func(...)}
        if old ~= 2 then setthreadidentity(old) end
        return table.unpack(r)
    end

    local oldindex
    oldindex = SafeHook(hookmetamethod, ac_script, "__index", function(t, k)
        if t == ac_script and not checkcaller() and kProtectedProperties[k] ~= nil and not bypassed then
            return kProtectedProperties[k]
        end
        if checkcaller() then return oldindex(t, k) end
        return SafeCall(oldindex, t, k)
    end)

    local oldnewindex
    oldnewindex = SafeHook(hookmetamethod, ac_script, "__newindex", function(t, k, v)
        if t == ac_script and not checkcaller() and kProtectedProperties[k] ~= nil and not bypassed then
            kProtectedProperties[k] = v
            if k == "Enabled" then kProtectedProperties.Disabled = not v end
            if k == "Disabled" then kProtectedProperties.Enabled = not v end
            return
        end
        if checkcaller() then return oldnewindex(t, k, v) end
        return SafeCall(oldnewindex, t, k, v)
    end)

    client_id, last = "", tick()
    local oldfireserver
    oldfireserver = SafeHook(hookfunction, ac_event.FireServer, function(self, ...)
        local now, args = tick(), {...}
        if not first_seen then
            first_seen = true
            local a = args[1]
            if type(a) == "table" and #a >= 1 and (type(a[1]) == "string" or type(a[1]) == "number") then
                client_id = tostring(a[1])
            end
            last, samples, hijack_ready = tick(), 1, true
            return SafeCall(oldfireserver, self, ...)
        end
        local interval = now - (last or now)
        if interval > 0 then
            expected_interval = samples == 0 and interval or (ema_alpha * interval + (1 - ema_alpha) * expected_interval)
            samples += 1
            if expected_interval < min_interval then expected_interval = min_interval end
        end
        local res = SafeCall(oldfireserver, self, ...)
        last = tick()
        return res
    end)

    local function BuildSubTable()
        local num_empty = math.random(1, 5)
        local empty_map, empty_slots = {[7]=true}, {7}
        while #empty_slots < num_empty do
            local s = math.random(1, 6)
            if not empty_map[s] then empty_map[s] = true; table.insert(empty_slots, s) end
        end
        table.sort(empty_slots)
        local r = {}
        for i = 1, 7 do r[i] = empty_map[i] and {} or kFilledSub end
        return r, empty_slots
    end

    local function ApplyTransforms(t, mask, empty_slots)
        local payload = t[1]
        local outer_index = #payload
        local inner_index = empty_slots[math.random(1, #empty_slots)]
        local derived, outer_val = nil, payload[outer_index]
        if type(outer_val) == "table" and type(inner_index) == "number" then
            derived = outer_val[inner_index]
        else
            for i = outer_index, 1, -1 do
                if type(payload[i]) == "table" then
                    local c = payload[i]
                    derived = (type(inner_index) == "number" and c[inner_index] ~= nil) and c[inner_index] or c
                    break
                end
            end
            derived = derived or {}
        end
        local written = {}
        for _, v in ipairs(mask) do
            local slot = kSlotMap[v]
            if slot and not written[slot] then t[slot] = derived; written[slot] = true end
        end
        return t
    end

    local function BuildPayload(challenge, mask)
        local sub, empty = BuildSubTable()
        local total = math.random(1, 8)
        local payload = {client_id, buffer.tostring(challenge)}
        for _ = 1, math.random(0, 2) do payload[#payload+1] = "" end
        while #payload < (total - 1) do payload[#payload+1] = math.random(5, 100000) end
        payload[#payload+1] = sub
        return ApplyTransforms({payload, {}, nil, nil, nil, nil, nil}, mask, empty)
    end

    task.spawn(function()
        getfenv().script = ac_script
        while not hijack_ready do task.wait() end
        ac_script.Enabled = false
        ac_event.OnClientEvent:Connect(function(...)
            last = tick()
            local remote = Instance.new("RemoteEvent")
            remote:FireServer()
            local t = {...}
            local challenge, index, mask = t[1], t[2], t[3]
            if typeof(challenge) ~= "buffer" or type(index) ~= "number" or type(mask) ~= "table" then
                LocalPlayer:Kick("[Vnix.lol-changer.gg]: Bypass failed!")
                return
            end
            local payload = BuildPayload(challenge, mask)
            task.defer(function()
                local wait_t = expected_interval - (tick() - (last or 0))
                if wait_t > 0 then task.wait(wait_t) end
                ac_event:FireServer(table.unpack(payload, 1, 5))
                last = tick()
                remote:Destroy()
            end)
        end)
        bypassed = true
    end)

    for _, name in ipairs(kKickNames) do
        local f = LocalPlayer[name]
        if type(f) == "function" then
            local old
            old = SafeHook(hookfunction, f, function(self, ...)
                if self == LocalPlayer and not checkcaller() then return end
                return old(self, ...)
            end)
        end
    end

    local bypass_tries = 0
    while not bypassed and bypass_tries < 20 do
        bypass_tries += 1
        task.wait(0.5)
    end
    if not bypassed then
        warn("[Vnix.lol-changer.gg]: Bypass timed out after 10s, continuing anyway")
    else
        task.wait(0.5)
    end
end)

pcall(function()
    local AuditState = {
        active = false,
        violations = {},
        executor_name = "",
        checks_passed = 0,
        checks_failed = 0,
    }

    local APIs = {}
    local function CacheAPIs()
        pcall(function() APIs.islclosure = islclosure end)
        pcall(function() APIs.iscclosure = iscclosure end)
        pcall(function() APIs.isourclosure = isourclosure end)
        pcall(function() APIs.isexecutorclosure = isexecutorclosure end)
        pcall(function() APIs.isnewcclosure = isnewcclosure end)
        pcall(function() APIs.isfunctionhooked = isfunctionhooked end)
        pcall(function() APIs.clonefunction = clonefunction end)
        pcall(function() APIs.hookfunction = hookfunction end)
        pcall(function() APIs.setrawmetatable = setrawmetatable end)
        pcall(function() APIs.getrawmetatable = getrawmetatable end)
        pcall(function() APIs.sethiddenproperty = sethiddenproperty end)
        pcall(function() APIs.identifyexecutor = identifyexecutor end)
        pcall(function() APIs.gethui = gethui end)
        if APIs.identifyexecutor then
            local ok, name = pcall(APIs.identifyexecutor)
            if ok and name then AuditState.executor_name = tostring(name) end
        end
    end

    local function ReportViolation(tag, details)
        AuditState.checks_failed += 1
        AuditState.violations[#AuditState.violations + 1] = {tag = tag, details = details or "", timestamp = tick()}
    end

    local function ReportSuccess(tag)
        AuditState.checks_passed += 1
    end

    local function VerifyClosureAPIs()
        if not APIs.islclosure or not APIs.iscclosure then return end
        local lua_closure = function() return 1 end
        local c_closure = print
        if APIs.islclosure(lua_closure) ~= true then ReportViolation("audit:islclosure.lua_mismatch") else ReportSuccess("audit:islclosure.lua") end
        if APIs.islclosure(c_closure) ~= false then ReportViolation("audit:islclosure.c_mismatch") else ReportSuccess("audit:islclosure.c") end
        if APIs.iscclosure(c_closure) ~= true then ReportViolation("audit:iscclosure.c_mismatch") else ReportSuccess("audit:iscclosure.c") end
        if APIs.iscclosure(lua_closure) ~= false then ReportViolation("audit:iscclosure.lua_mismatch") else ReportSuccess("audit:iscclosure.lua") end
    end

    local function VerifyMetatableAPIs()
        if not APIs.setrawmetatable or not APIs.getrawmetatable then return end
        local info = debug.info
        if info then
            local ok, source = pcall(info, APIs.setrawmetatable, "s")
            if ok and source ~= "[C]" then ReportViolation("pre:setrawmetatable.src_not_C") else ReportSuccess("pre:setrawmetatable.src") end
            ok, source = pcall(info, APIs.getrawmetatable, "s")
            if ok and source ~= "[C]" then ReportViolation("pre:getrawmetatable.src_not_C") else ReportSuccess("pre:getrawmetatable.src") end
        end
    end

    local function VerifyFireServerIntegrity()
        local ok, remote = pcall(function() return Instance.new("RemoteEvent") end)
        if not ok or not remote then return end
        local fireserver = remote.FireServer
        if debug.info and fireserver then
            local src_ok, source = pcall(debug.info, fireserver, "s")
            if src_ok and source ~= "[C]" then ReportViolation("fs:src_not_C") else ReportSuccess("fs:src") end
        end
        pcall(function() remote:Destroy() end)
    end

    local function VerifyKnownTruths()
        if APIs.isexecutorclosure and APIs.isfunctionhooked then
            local ok, hooked = pcall(APIs.isfunctionhooked, APIs.isexecutorclosure)
            if ok and hooked == true then ReportViolation("known-truth:iec_hooked") else ReportSuccess("known-truth:iec") end
        end
    end

    CacheAPIs()
    VerifyClosureAPIs()
    VerifyMetatableAPIs()
    VerifyFireServerIntegrity()
    VerifyKnownTruths()
    AuditState.active = true

    if AuditState.checks_failed > 0 then
        warn("[Audit System]: " .. AuditState.checks_failed .. " violations detected!")
        for _, v in ipairs(AuditState.violations) do warn("  - " .. v.tag) end
    end
end)

local modules = {}
pcall(function()
    local PS = LocalPlayer:WaitForChild("PlayerScripts", 10)
    local Controllers = PS and PS:FindFirstChild("Controllers")
    local RSModules = ReplicatedStorage:FindFirstChild("Modules")

    if Controllers then
        pcall(function() modules.PlayerDataController = require(Controllers:WaitForChild("PlayerDataController", 5)) end)
        pcall(function() modules.LeaderboardController = require(Controllers:WaitForChild("LeaderboardController", 5)) end)
        pcall(function() modules.ControlsController = require(Controllers:WaitForChild("ControlsController", 5)) end)
        pcall(function() modules.MonetizationController = require(Controllers:WaitForChild("MonetizationController", 5)) end)
    end
    if RSModules then
        pcall(function() modules.SeasonLibrary = require(RSModules:WaitForChild("SeasonLibrary", 5)) end)
        pcall(function() modules.MonetizationLibrary = require(RSModules:WaitForChild("MonetizationLibrary", 5)) end)
        pcall(function() modules.CosmeticLibrary = require(RSModules:WaitForChild("CosmeticLibrary", 5)) end)
        pcall(function() modules.ShopLibrary = require(RSModules:WaitForChild("ShopLibrary", 5)) end)
        pcall(function() modules.ItemLibrary = require(RSModules:WaitForChild("ItemLibrary", 5)) end)
    end
end)

pcall(function()
    if modules.PlayerDataController and modules.PlayerDataController.WaitUntilLoaded then
        modules.PlayerDataController:WaitUntilLoaded()
    end
end)

local Connections = {}
function Bind(name, signal, fn)
    if Connections[name] then Connections[name]:Disconnect() end
    Connections[name] = signal:Connect(fn)
end
function Unbind(name)
    if Connections[name] then Connections[name]:Disconnect(); Connections[name] = nil end
end

function SafeGet(key, fallback)
    local ok, v = pcall(function()
        return modules.PlayerDataController and modules.PlayerDataController:Get(key)
    end)
    return (ok and v ~= nil) and v or fallback
end

local states = {
    spoofer_state = {
        device = "Desktop",
        old_device = tostring(modules.ControlsController and modules.ControlsController.CurrentControls or "Desktop"),
        spoof_device = false,
        device_spam = false,
        spam_rate = 1,

        display_name_value = LocalPlayer.DisplayName,
        username_value = LocalPlayer.Name,
        display_name = false,
        username = false,
        name_spoof_conn = {},

        avatar_userid = LocalPlayer.UserId,
        spoof_avatar = false,
        thumb_spoof_conn = {},

        anonymous_mode = false,
        fake_names = {},
        anon_connections = {},

        leaderboard = {
            elo_value = 0,
            streak_value = LocalPlayer:GetAttribute("StatisticDuelsWinStreak") or 0,
            kills_value = 0,
            wins_value = 0,
            level_value = LocalPlayer:GetAttribute("Level") or 0,
            old_streak_value = LocalPlayer:GetAttribute("StatisticDuelsWinStreak"),
            old_level_value = LocalPlayer:GetAttribute("Level"),
            ELO = false, Streak = false, Kills = false, Wins = false, Level = false,
        },

        badges = {Premium = false, Verified = false, Influencer = false, Admin = false},

        charm = {
            charm_rank = "Use Spoofed ELO", arch_rank = 1,
            s0_charm = false, s1_charm = false, s2_charm = false, s3_charm = false,
        },

        currency = {
            weapon_keys = false, unlock_tokens = false, event_currency = false,
            glory = false, skin_tickets = false,
            weapon_keys_value = SafeGet("WeaponKeys", 0),
            unlock_tokens_value = SafeGet("UnlockTokens", 0),
            event_currency_value = SafeGet("EventCurrency", 0),
            glory_value = SafeGet("Glory", 0),
            skin_tickets_value = SafeGet("SkinTickets", 0),
        },

        duel_history = {
            duel_history_value = SafeGet("DuelHistory", {}),
            logged_elo_events = SafeGet("LoggedELOEvents", {}),
            match_index = 1, mode = "Ranked", result = "Victory", map = "Factory",
            team1score = 5, team2score = 0, dueler_index = 1,
            username = "", display_name = "",
            kills = 15, deaths = 2, assists = 6, damage = 5000, elo = 2000, elo_change = 35,
        },

        marketplace = {
            fake_bundles_owned = SafeGet("GamepassBundlesClaimed", {}),
            fake_gift_robux_spent = SafeGet("GiftRobuxSpentProgress", 0),
            fake_gift_rewards_claimed = SafeGet("GiftRobuxSpentRewardsClaimed", {}),
            fake_robux = false, robux_amount = 0, fake_gifting = false,
        },
    },

    skinchanger_state = {
        fake_owned = SafeGet("CosmeticInventory", {}),
        fake_weapon_owned = SafeGet("WeaponInventory", {}),
        equipped = {},
        favorites = {},
        fake_weapon_names = {},
        constructing_weapon = nil,
        last_used_weapon = nil,
        placed_object_map = {},
        viewing_profile = nil,

        general_unlocker = {
            unlock_type = "Skin",
            unlock_rarity = "Common",
        },

        specific_unlocker = {
            unlock_type = "Skin",
            cosmetic_name = "",
            weapon_name = "",
        },

        equip_unlocker = {
            unlock_type = "Skin",
            cosmetic_name = "",
            weapon_name = "",
            inverted = false,
        },
    },

    inventory_state = {
        fake_owned = SafeGet("UnclaimedRewards", {}),
    },
    emote_state = {
        selected = "",
        equipped = "",
        playing = false,
        speed = 1,
    },
}

local kCharmEloMap = {
    ["Unranked"] = -1, ["Bronze 1"] = 0, ["Bronze 2"] = 200, ["Bronze 3"] = 400,
    ["Silver 1"] = 600, ["Silver 2"] = 800, ["Silver 3"] = 1000,
    ["Gold 1"] = 1200, ["Gold 2"] = 1400, ["Gold 3"] = 1600,
    ["Platinum 1"] = 1800, ["Platinum 2"] = 2000, ["Platinum 3"] = 2200,
    ["Diamond 1"] = 2400, ["Diamond 2"] = 2600, ["Diamond 3"] = 2800,
    ["Onyx 1"] = 3000, ["Onyx 2"] = 3200, ["Onyx 3"] = 3400,
    ["Nemesis"] = 3600, ["Archnemesis"] = 3600,
}
local kSeasonNameMap = {[0]="Zero",[1]="Warp",[2]="Polar",[3]="Fame", Zero=0, Warp=1, Polar=2, Fame=3}
local kSeasonToggleMap = {[0]="s0_charm",[1]="s1_charm",[2]="s2_charm",[3]="s3_charm"}
local kWordList = {
    "late","couple","economic","obscure","landscape","spot","college","rainbow","leg","mixture",
    "frank","commitment","draft","motivation","fuss","settle","jelly","confine","pin","modernize",
    "velvet","quarter","physical","differ","dish","aisle","happen","council","appeal","depart",
    "urgency","vessel",
}

pcall(function()
    if not modules.PlayerDataController then return end
    local original = modules.PlayerDataController.Get
    modules.PlayerDataController.Get = function(self, key, ...)
        local data = original(self, key, ...)
        local s = states.spoofer_state
        if key == "WeaponKeys" and s.currency.weapon_keys then return s.currency.weapon_keys_value end
        if key == "UnlockTokens" and s.currency.unlock_tokens then return s.currency.unlock_tokens_value end
        if key == "EventCurrency" and s.currency.event_currency then return s.currency.event_currency_value end
        if key == "Glory" and s.currency.glory then return s.currency.glory_value end
        if key == "SkinTickets" and s.currency.skin_tickets then return s.currency.skin_tickets_value end
        if key == "DuelHistory" then return s.duel_history.duel_history_value or data end
        if key == "LoggedELOEvents" then return s.duel_history.logged_elo_events or data end
        if key == "GamepassBundlesClaimed" then return s.marketplace.fake_bundles_owned or data end
        if key == "GiftRobuxSpentProgress" then return s.marketplace.fake_gift_robux_spent or data end
        if key == "GiftRobuxSpentRewardsClaimed" then return s.marketplace.fake_gift_rewards_claimed or data end
        if key == "CosmeticInventory" then return states.skinchanger_state.fake_owned or data end
        if key == "EquippedEmote" or key == "SelectedEmote" then
            local eq = states.emote_state and states.emote_state.equipped
            if eq and eq ~= "" then return eq end
        end
        if key == "WeaponInventory" or key == "FreeWeaponUnlockCheck" then return states.skinchanger_state.fake_weapon_owned or data end
        if key == "UnclaimedRewards" then return states.inventory_state.fake_owned or data end
        return data
    end
end)

pcall(function()
    if not modules.CosmeticLibrary then return end
    local cos = modules.CosmeticLibrary

    local function WrapOwnership(old, expected_args)
        return function(...)
            local args = {...}
            local name = args[expected_args]
            if type(name) == "string" and states.skinchanger_state.fake_owned[name] then return true end
            if old then return old(...) end
            return false
        end
    end

    local orig_normal = cos.OwnsCosmeticNormally
    cos.OwnsCosmeticNormally = WrapOwnership(orig_normal, 3)
    local orig_universal = cos.OwnsCosmeticUniversally
    cos.OwnsCosmeticUniversally = WrapOwnership(orig_universal, 3)
    local orig_for_something = cos.OwnsCosmeticForSomething
    cos.OwnsCosmeticForSomething = WrapOwnership(orig_for_something, 3)
    local orig_for_weapon = cos.OwnsCosmeticForWeapon
    cos.OwnsCosmeticForWeapon = WrapOwnership(orig_for_weapon, 3)

    local orig_owns = cos.OwnsCosmetic
    cos.OwnsCosmetic = function(self, inventory, name, weapon)
        if type(name) == "string" and states.skinchanger_state.fake_owned[name] then return true end
        if orig_owns then return orig_owns(self, inventory, name, weapon) end
        return false
    end
end)

pcall(function()
    if not modules.PlayerDataController then return end
    local pdc = modules.PlayerDataController

    if pdc.HasGamepass then
        local orig_hasgamepass = pdc.HasGamepass
        pdc.HasGamepass = function(p11, ...)
            local has_gamepass = orig_hasgamepass and orig_hasgamepass(p11, ...)
            local name = ({...})[1]
            return type(name) == "string" and states.spoofer_state.marketplace.fake_bundles_owned[name] or has_gamepass
        end
    end

    if pdc.GetWeaponData then
        local orig_getweapondata = pdc.GetWeaponData
        pdc.GetWeaponData = function(p1, ...)
            local weapon_name = ({...})[1]
            local original_data, index = orig_getweapondata(p1, ...)

            local fake_owned = false
            for _, weapon in ipairs(states.skinchanger_state.fake_weapon_owned) do
                if weapon.Name == weapon_name then fake_owned = true break end
            end

            if fake_owned and not original_data then
                local fake_data = {
                    Name = weapon_name,
                    Level = 1,
                    XP = 0,
                    IsFavorited = false,
                    Skin = nil,
                }
                local equipped = states.skinchanger_state.equipped
                if equipped and equipped[weapon_name] then
                    for cos_type, cos_data in pairs(equipped[weapon_name]) do
                        if type(cos_data) == "table" and (cos_data.Name == "NONE_COSMETIC" or cos_data.Name == "None") then
                            fake_data[cos_type] = nil
                        else
                            fake_data[cos_type] = cos_data
                        end
                    end
                end
                return fake_data, index
            end

            local equipped = states.skinchanger_state.equipped
            if original_data and equipped and equipped[weapon_name] then
                for cos_type, cos_data in pairs(equipped[weapon_name]) do
                    if type(cos_data) == "table" and (cos_data.Name == "NONE_COSMETIC" or cos_data.Name == "None") then
                        original_data[cos_type] = nil
                    else
                        original_data[cos_type] = cos_data
                    end
                end
            end

            return original_data, index
        end
    end
end)

function PatchCurrentDataGet(current_data)
    if not current_data or type(current_data.Get) ~= "function" or current_data._fakeWeaponOwnedPatched then
        return
    end
    local old_get = current_data.Get
    current_data.Get = function(self, ...)
        local data = old_get(self, ...)
        local key = ({...})[1]
        if key == "CosmeticInventory" then return states.skinchanger_state.fake_owned or data end
        if key == "UnclaimedRewards" then return states.inventory_state.fake_owned or data end
        if key == "WeaponInventory" or key == "FreeWeaponUnlockCheck" then return states.skinchanger_state.fake_weapon_owned or data end
        return data
    end
    current_data._fakeWeaponOwnedPatched = true
end

task.spawn(function()
    while not (modules.PlayerDataController and modules.PlayerDataController.CurrentData) do
        task.wait(0.1)
    end
    PatchCurrentDataGet(modules.PlayerDataController.CurrentData)
end)

local device_cancel = 0
-- Device spoof (source DeviceSpoof.txt)
local DEVICE_SPOOF_MAP = {
    Computer = "MouseKeyboard",
    Desktop = "MouseKeyboard",
    Mobile = "Touch",
    Touch = "Touch",
    Console = "Gamepad",
    Gamepad = "Gamepad",
    VR = "VR",
}

function change_device(device_name)
    local device = DEVICE_SPOOF_MAP[device_name]
    if not device then
        warn("[Matcha] invalid device:", device_name)
        return false
    end
    local ok = pcall(function()
        ReplicatedStorage.Remotes.Replication.Fighter.SetControls:FireServer(device)
    end)
    return ok
end

function SetDevice(use_old)
    local key = use_old and states.spoofer_state.old_device or states.spoofer_state.device
    -- normaliza nombres UI → source
    if key == "Desktop" then key = "Computer" end
    change_device(key or "Computer")
end

function SpoofDevice()
    if states.spoofer_state.spoof_device then
        SetDevice(false)
    else
        -- restore
        local old = states.spoofer_state.old_device or "Desktop"
        change_device(old == "Desktop" and "Computer" or old)
    end
end

function DeviceSpam()
    device_cancel += 1
    local token = device_cancel
    task.spawn(function()
        local list = {"Computer","Mobile","Console","VR"}
        while states.spoofer_state.device_spam and token == device_cancel do
            states.spoofer_state.device = list[math.random(1, #list)]
            SetDevice(false)
            task.wait(states.spoofer_state.spam_rate)
        end
        SetDevice(true)
    end)
end

function IsTextObject(obj)
    return obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox")
end

function FormatNumber(n)
    n = tostring(n)
    return n:reverse():gsub("%d%d%d", "%1,"):reverse():gsub("^,", "")
end

function ApplyNameSpoof(obj)
    if not IsTextObject(obj) then return end
    local text = obj.Text
    if typeof(text) ~= "string" then return end

    if states.spoofer_state.display_name and obj.Name == "DisplayName" then
        text = text:gsub(LocalPlayer.DisplayName, states.spoofer_state.display_name_value)
    end
    if states.spoofer_state.username and obj.Name == "Username" then
        text = text:gsub(LocalPlayer.Name, states.spoofer_state.username_value)
    end
    if states.spoofer_state.display_name and obj.Parent.Name == "NameContainer" and obj.Name == "Name" then
        text = text:gsub(LocalPlayer.DisplayName, states.spoofer_state.display_name_value)
    end
    if states.spoofer_state.username and obj.Parent.Name == "NameContainer" and obj.Name == "Handle" then
        text = text:gsub(LocalPlayer.Name, states.spoofer_state.username_value)
    end
    if states.spoofer_state.display_name and obj.Parent.Name == "Title" and obj.Name == "TitleText" then
        text = text:gsub(LocalPlayer.DisplayName, states.spoofer_state.display_name_value)
    end
    if states.spoofer_state.username and obj.Parent.Name == "Subtitle" and obj.Name == "SubtitleText" then
        text = text:gsub(LocalPlayer.Name, states.spoofer_state.username_value)
    end
    if states.spoofer_state.display_name and obj.Name == "HeaderText" then
        text = text:gsub(LocalPlayer.DisplayName, states.spoofer_state.display_name_value)
    end
    if states.spoofer_state.display_name and obj.Name == "Title" then
        text = text:gsub(LocalPlayer.DisplayName, states.spoofer_state.display_name_value)
    end
    -- Wins
    if states.spoofer_state.leaderboard.Wins and (obj.Name == "Wins" or obj.Name == "WinsValue") then
        text = FormatNumber(states.spoofer_state.leaderboard.wins_value)
    end

    -- Level
    if states.spoofer_state.leaderboard.Level then
        if (obj.Parent and obj.Parent.Name == "Level" and obj.Name == "Value")
        or obj.Name == "Level"
        or obj.Name == "LevelValue" then
            text = FormatNumber(states.spoofer_state.leaderboard.level_value)
        end
    end

    -- Streak (perfil + tab)
    if states.spoofer_state.leaderboard.Streak then
        if (obj.Parent and obj.Parent.Name == "Streak" and obj.Name == "Value")
        or (obj.Parent and obj.Parent.Name == "WinStreak" and obj.Name == "Value")
        or obj.Name == "Streak"
        or obj.Name == "WinStreak"
        or obj.Name == "CurrentStreak"
        or obj.Name == "StreakValue"
        or obj.Name == "WinStreakValue" then
            text = FormatNumber(states.spoofer_state.leaderboard.streak_value)
        end
    end

    -- ELO número
    if states.spoofer_state.leaderboard.ELO then
        if (obj.Parent and obj.Parent.Name == "ELO" and obj.Name == "Value")
        or obj.Name == "ELO"
        or obj.Name == "Elo"
        or obj.Name == "EloValue"
        or obj.Name == "ELOValue"
        or obj.Name == "HighestELO" then
            text = FormatNumber(states.spoofer_state.leaderboard.elo_value)
        end

        -- Forzar rango Nemesis / Archnemesis de forma agresiva
        local elo = tonumber(states.spoofer_state.leaderboard.elo_value) or 0
        local rankName = nil
        if elo >= 10000 then
            rankName = "Archnemesis"
        elseif elo >= 3600 then
            rankName = "Nemesis"
        end

        if rankName then
            local lower = string.lower(tostring(text))
            if lower:find("bronze")
            or lower:find("silver")
            or lower:find("gold")
            or lower:find("platinum")
            or lower:find("diamond")
            or lower:find("onyx")
            or lower:find("nemesis")
            or lower:find("archnemesis")
            or lower:find("unranked")
            or lower:find("rank") then
                text = rankName
            end
        end
    end

    if obj.Text ~= text then obj.Text = text end
end

function RefreshAllNameSpoofs()
    for obj in pairs(states.spoofer_state.name_spoof_conn) do
        if obj and obj.Parent then ApplyNameSpoof(obj) end
    end
end

function RegisterTextObject(obj)
    if not IsTextObject(obj) then return end
    local allowed = {
        DisplayName=true, Username=true, Name=true, Handle=true,
        TitleText=true, SubtitleText=true, HeaderText=true, Title=true, Wins=true, Value=true,
        ELO=true, Elo=true, EloValue=true, ELOValue=true, HighestELO=true,
        Rank=true, RankName=true, RankText=true, RankLabel=true,
        Streak=true, WinStreak=true, CurrentStreak=true, StreakValue=true,
        Level=true, LevelValue=true,
    }
    -- También capturar cualquier texto que parezca un rango
    local txt = tostring(obj.Text or "")
    local lower = string.lower(txt)
    local looksLikeRank = lower:find("bronze") or lower:find("silver") or lower:find("gold")
        or lower:find("platinum") or lower:find("diamond") or lower:find("onyx")
        or lower:find("nemesis") or lower:find("archnemesis") or lower:find("unranked")
    if not allowed[obj.Name] and not looksLikeRank then return end
    ApplyNameSpoof(obj)
    if states.spoofer_state.name_spoof_conn[obj] then return end
    states.spoofer_state.name_spoof_conn[obj] = obj:GetPropertyChangedSignal("Text"):Connect(function()
        ApplyNameSpoof(obj)
    end)
    obj.Destroying:Connect(function()
        if states.spoofer_state.name_spoof_conn[obj] then
            states.spoofer_state.name_spoof_conn[obj]:Disconnect()
            states.spoofer_state.name_spoof_conn[obj] = nil
        end
    end)
end

for _, o in ipairs(game:GetDescendants()) do RegisterTextObject(o) end
game.DescendantAdded:Connect(RegisterTextObject)

function ApplyThumbnailSpoof(obj)
    if not obj:IsA("ImageLabel") then return end
    local image = obj.Image
    if typeof(image) ~= "string" then return end

    if states.spoofer_state.spoof_avatar and obj.Name == "AvatarThumbnail" then
        image = image:gsub(LocalPlayer.UserId, states.spoofer_state.avatar_userid)
    end
    if states.spoofer_state.spoof_avatar and obj.Parent.Name == "Thumbnail" and obj.Name == "ImageLabel" then
        image = image:gsub(LocalPlayer.UserId, states.spoofer_state.avatar_userid)
    end
    if states.spoofer_state.spoof_avatar and obj.Name == "Headshot" then
        image = image:gsub(LocalPlayer.UserId, states.spoofer_state.avatar_userid)
    end

    if obj.Image ~= image then obj.Image = image end
end

function RefreshAllThumbnailSpoofs()
    for obj in pairs(states.spoofer_state.thumb_spoof_conn) do
        if obj and obj.Parent then ApplyThumbnailSpoof(obj) end
    end
end

function RegisterImageObject(obj)
    if not obj:IsA("ImageLabel") then return end
    if not ({AvatarThumbnail=true, ImageLabel=true, Headshot=true})[obj.Name] then return end
    ApplyThumbnailSpoof(obj)
    if states.spoofer_state.thumb_spoof_conn[obj] then return end
    states.spoofer_state.thumb_spoof_conn[obj] = obj:GetPropertyChangedSignal("Image"):Connect(function()
        ApplyThumbnailSpoof(obj)
    end)
    obj.Destroying:Connect(function()
        if states.spoofer_state.thumb_spoof_conn[obj] then
            states.spoofer_state.thumb_spoof_conn[obj]:Disconnect()
            states.spoofer_state.thumb_spoof_conn[obj] = nil
        end
    end)
end

for _, o in ipairs(game:GetDescendants()) do RegisterImageObject(o) end
game.DescendantAdded:Connect(RegisterImageObject)

function FindFirstMatchingAttachment(model, name)
    for _, child in pairs(model:GetChildren()) do
        if child:IsA("Attachment") and child.Name == name then return child end
        if not child:IsA("Accoutrement") and not child:IsA("Tool") then
            local f = FindFirstMatchingAttachment(child, name)
            if f then return f end
        end
    end
end

function BuildWeld(weld_name, parent, part0, part1, c0, c1)
    local weld = Instance.new("Weld")
    weld.Name = weld_name
    weld.Part0 = part0
    weld.Part1 = part1
    weld.C0 = c0
    weld.C1 = c1
    weld.Parent = parent
    return weld
end

function WeldAttachments(a1, a2)
    local w = Instance.new("Weld")
    w.Part0, w.Part1 = a1.Parent, a2.Parent
    w.C0, w.C1 = a1.CFrame, a2.CFrame
    w.Parent = a1.Parent
    return w
end

function AddAccoutrement(character, acc)
    acc.Parent = character
    local handle = acc:FindFirstChild("Handle")
    if not handle then return end
    local att = handle:FindFirstChildOfClass("Attachment")
    if att then
        local ca = FindFirstMatchingAttachment(character, att.Name)
        if ca then WeldAttachments(ca, att) end
    else
        local head = character:FindFirstChild("Head")
        if head then
            local attachmentCFrame = CFrame.new(0, 0.5, 0)
            local hatCFrame = acc.AttachmentPoint
            BuildWeld("HeadWeld", head, head, handle, attachmentCFrame, hatCFrame)
        end
    end
end

function FindHeadPart(inst)
    if not inst then return end
    if inst:IsA("BasePart") then return inst end
    local h = inst:FindFirstChild("Head")
    if h and h:IsA("BasePart") then return h end
    for _, d in ipairs(inst:GetDescendants()) do
        if d:IsA("BasePart") and d.Name == "Head" then return d end
    end
    for _, d in ipairs(inst:GetDescendants()) do
        if d:IsA("BasePart") then return d end
    end
end

function CopyHeadVisuals(real_head, source_head)
    if not real_head or not source_head then return end
    if not real_head:IsA("BasePart") or not source_head:IsA("BasePart") then return end
    for _, c in ipairs(real_head:GetChildren()) do
        if c:IsA("Decal") or c:IsA("Texture") or c:IsA("SpecialMesh") or c:IsA("Mesh") then c:Destroy() end
    end
    real_head.Material = source_head.Material
    real_head.Color = source_head.Color
    real_head.Reflectance = source_head.Reflectance
    real_head.Transparency = source_head.Transparency
    real_head.LocalTransparencyModifier = source_head.LocalTransparencyModifier
    real_head.CastShadow = source_head.CastShadow
    real_head.CanCollide = false
    if real_head:IsA("MeshPart") and source_head:IsA("MeshPart") then
        real_head.MeshId = source_head.MeshId
        real_head.TextureID = source_head.TextureID
        real_head.Size = source_head.Size
    end
    for _, c in ipairs(source_head:GetChildren()) do
        if c:IsA("Decal") or c:IsA("Texture") or c:IsA("SpecialMesh") or c:IsA("Mesh") then
            c:Clone().Parent = real_head
        end
    end
end

function ApplyFakeHead(character, source_head)
    local real = character and character:FindFirstChild("Head")
    if not real or not real:IsA("BasePart") or not source_head then return end
    CopyHeadVisuals(real, source_head)
end

function SetAvatar(use_old)
    local uid = use_old and LocalPlayer.UserId or states.spoofer_state.avatar_userid
    if not uid then return end
    if not LocalChar or not LocalChar.Parent then LocalChar = GetCharacter() end

    local ok, rig = pcall(function() return Players:CreateHumanoidModelFromUserId(uid) end)
    if not ok or not rig then return end

    for _, v in ipairs(LocalChar:GetDescendants()) do
        if v:IsA("Accessory") or v:IsA("Shirt") or v:IsA("Pants") or v:IsA("ShirtGraphic")
            or v:IsA("BodyColors") or v:IsA("CharacterMesh") or v:IsA("LayeredClothing")
            or (v:IsA("Decal") and v.Name == "face") then
            pcall(function() v:Destroy() end)
        end
    end

    for _, item in ipairs(rig:GetChildren()) do
        if item:IsA("Accessory") then
            pcall(function() AddAccoutrement(LocalChar, item:Clone()) end)
        elseif item:IsA("Shirt") or item:IsA("Pants") or item:IsA("ShirtGraphic")
            or item:IsA("BodyColors") or item:IsA("LayeredClothing") or item:IsA("CharacterMesh") then
            pcall(function() item:Clone().Parent = LocalChar end)
        end
    end

    local head = FindHeadPart(rig)
    if head then ApplyFakeHead(LocalChar, head) end
    pcall(function() rig:Destroy() end)
end

function SpoofAvatar()
    SetAvatar(not states.spoofer_state.spoof_avatar)
end
LocalPlayer.CharacterAdded:Connect(function()
    task.wait(0.5)
    if states.spoofer_state.spoof_avatar then SetAvatar(false) end
end)

function GenerateRandomName()
    local result = ""
    for i = 1, math.random(2, 4) do
        local index = math.random(1, #kWordList)
        result ..= kWordList[index]
    end
    return result .. tostring(math.random(12, 36))
end

function GetFakeName(player)
    if not states.spoofer_state.fake_names[player] then
        states.spoofer_state.fake_names[player] = GenerateRandomName()
    end
    return states.spoofer_state.fake_names[player]
end

local AnonymousRigTemplate
function GetAnonymousRig()
    if AnonymousRigTemplate and AnonymousRigTemplate.Parent then
        return AnonymousRigTemplate:Clone()
    end
    local ok, rig = pcall(function()
        return Players:CreateHumanoidModelFromUserId(6942067)
    end)
    if not ok or not rig then return nil end
    AnonymousRigTemplate = rig
    AnonymousRigTemplate.Parent = nil
    return AnonymousRigTemplate:Clone()
end

function ApplyAnonymousNameSpoof(obj)
    if not IsTextObject(obj) then return end
    local text = obj.Text
    for _, player in ipairs(Players:GetPlayers()) do
        local fakeName = GetFakeName(player)
        text = text:gsub(player.Name, fakeName)
        text = text:gsub(player.DisplayName, fakeName)
    end
    if obj.Text ~= text then obj.Text = text end
end

function ApplyAnonymousThumbnailSpoof(obj)
    if not obj:IsA("ImageLabel") then return end
    local image = obj.Image
    for _, player in ipairs(Players:GetPlayers()) do
        image = image:gsub(tostring(player.UserId), tostring(6942067))
    end
    if obj.Image ~= image then obj.Image = image end
end

function ApplyAnonymousAvatar(character)
    if not character then return end
    local rig = GetAnonymousRig()
    if not rig then return end
    for _, v in ipairs(character:GetDescendants()) do
        local isFaceDecal = v:IsA("Decal") and v.Name == "face"
        if v:IsA("Accessory") or v:IsA("Shirt") or v:IsA("Pants") or v:IsA("ShirtGraphic")
            or v:IsA("BodyColors") or v:IsA("CharacterMesh") or v:IsA("LayeredClothing")
            or isFaceDecal then
            pcall(function() v:Destroy() end)
        end
    end
    for _, item in ipairs(rig:GetChildren()) do
        if item:IsA("Accessory") then
            AddAccoutrement(character, item:Clone())
        elseif item:IsA("Shirt") or item:IsA("Pants") or item:IsA("ShirtGraphic")
            or item:IsA("BodyColors") or item:IsA("CharacterMesh") or item:IsA("LayeredClothing") then
            item:Clone().Parent = character
        end
    end
    local head = FindHeadPart(rig)
    if head then ApplyFakeHead(character, head) end
    rig:Destroy()
end

function RegisterAnonymousObject(obj)
    if IsTextObject(obj) then
        ApplyAnonymousNameSpoof(obj)
        obj:GetPropertyChangedSignal("Text"):Connect(function()
            ApplyAnonymousNameSpoof(obj)
        end)
    elseif obj:IsA("ImageLabel") then
        ApplyAnonymousThumbnailSpoof(obj)
        obj:GetPropertyChangedSignal("Image"):Connect(function()
            ApplyAnonymousThumbnailSpoof(obj)
        end)
    end
end

function DisableAnonymousMode()
    for _, c in ipairs(states.spoofer_state.anon_connections) do
        if c and c.Disconnect then
            pcall(function() c:Disconnect() end)
        end
    end
    table.clear(states.spoofer_state.anon_connections)
    table.clear(states.spoofer_state.fake_names)
end

function AnonymousMode()
    if not states.spoofer_state.anonymous_mode then
        DisableAnonymousMode()
        return
    end
    DisableAnonymousMode()
    math.randomseed(os.clock() * 100000)
    for _, player in ipairs(Players:GetPlayers()) do
        GetFakeName(player)
        if player.Character then
            task.defer(ApplyAnonymousAvatar, player.Character)
        end
        table.insert(states.spoofer_state.anon_connections,
            player.CharacterAdded:Connect(function(character)
                task.defer(ApplyAnonymousAvatar, character)
            end))
    end
    table.insert(states.spoofer_state.anon_connections,
        Players.PlayerAdded:Connect(function(player)
            GetFakeName(player)
            player.CharacterAdded:Connect(function(character)
                task.defer(ApplyAnonymousAvatar, character)
            end)
        end))
    for _, obj in ipairs(game:GetDescendants()) do
        RegisterAnonymousObject(obj)
    end
    table.insert(states.spoofer_state.anon_connections,
        game.DescendantAdded:Connect(RegisterAnonymousObject))
end

function UpdateBadgesOptions()
    LocalPlayer:SetAttribute("IsInfluencer", states.spoofer_state.badges.Influencer and true or false)
    LocalPlayer:SetAttribute("IsRobloxEmployee", states.spoofer_state.badges.Admin and true or false)
    -- Premium/Verified are usually membership flags; attribute spoof covers common UI checks
    LocalPlayer:SetAttribute("IsPremium", states.spoofer_state.badges.Premium and true or false)
    LocalPlayer:SetAttribute("HasVerifiedBadge", states.spoofer_state.badges.Verified and true or false)
end

pcall(function()
    local old_badge_index
    old_badge_index = hookmetamethod(game, "__index", function(self, key)
        if self == LocalPlayer then
            if key == "MembershipType" and states.spoofer_state.badges.Premium then
                return Enum.MembershipType.Premium
            end
            if key == "HasVerifiedBadge" and states.spoofer_state.badges.Verified then
                return true
            end
        end
        return old_badge_index(self, key)
    end)
end)

function InsertPlayerByValue(list, user_id, value)
    for i = #list, 1, -1 do
        if tonumber(list[i].key) == user_id then table.remove(list, i) break end
    end
    local rank = 1
    for i, e in ipairs(list) do
        if tonumber(e.value) >= value then rank = i + 1 else break end
    end
    table.insert(list, rank, {key = tostring(user_id), value = value})
end

function ResetLeaderboard(name, config_idx)
    if not modules.LeaderboardController then return end
    Unbind(config_idx)
    local serial = modules.LeaderboardController.LeaderboardSerials
        and modules.LeaderboardController.LeaderboardSerials[name]
    if serial and serial.Players then
        local uid = LocalPlayer.UserId
        for i = #serial.Players, 1, -1 do
            local p = serial.Players[i]
            if p.UserId == uid or p.UserID == uid or p.PlayerId == uid then
                table.remove(serial.Players, i)
            end
        end
        if modules.LeaderboardController.Refreshed then
            modules.LeaderboardController.Refreshed:Fire(name)
        end
    end
end

function ModifyLeaderboard(config_idx, name, value)
    if not modules.LeaderboardController then return end
    task.spawn(function()
        while not modules.LeaderboardController.LeaderboardSerials[name]
            or not modules.LeaderboardController.LeaderboardSerials[name].Players
            or #modules.LeaderboardController.LeaderboardSerials[name].Players <= 0 do
            task.wait(0.5)
        end

        local user_id = LocalPlayer.UserId
        local serial = modules.LeaderboardController.LeaderboardSerials[name]
        if serial and serial.Players then
            InsertPlayerByValue(serial.Players, user_id, value)
            if modules.LeaderboardController.Refreshed then
                modules.LeaderboardController.Refreshed:Fire(name)
            end
        end

        local UpdateRemote = ReplicatedStorage:FindFirstChild("Remotes")
            and ReplicatedStorage.Remotes:FindFirstChild("Misc")
            and ReplicatedStorage.Remotes.Misc:FindFirstChild("UpdateLeaderboard")
        if not (UpdateRemote and UpdateRemote:IsA("RemoteEvent")) then return end

        Bind(config_idx, UpdateRemote.OnClientEvent, function(data)
            local lb_name = data and data.Name
            if not lb_name or lb_name ~= name then return end
            if not states.spoofer_state.leaderboard[config_idx] then return end
            task.defer(function()
                local serial = modules.LeaderboardController.LeaderboardSerials[name]
                if serial and serial.Players then
                    InsertPlayerByValue(serial.Players, user_id, value)
                    if modules.LeaderboardController.Refreshed then
                        modules.LeaderboardController.Refreshed:Fire(name)
                    end
                end
            end)
        end)
    end)
end

function UpdateLeaderboardOptions()
    local lb = states.spoofer_state.leaderboard

    if lb.ELO then
        Unbind("ELO")
        ModifyLeaderboard("ELO", "Highest ELO", lb.elo_value)

        -- Auto rank según ELO (Nemesis / Archnemesis)
        local elo = tonumber(lb.elo_value) or 0
        local charm = states.spoofer_state.charm
        if elo >= 10000 then
            charm.charm_rank = "Archnemesis"
            charm.arch_rank = 3
        elseif elo >= 3600 then
            charm.charm_rank = "Nemesis"
            charm.arch_rank = 1
        elseif elo >= 3400 then
            charm.charm_rank = "Onyx 3"
        elseif elo >= 3200 then
            charm.charm_rank = "Onyx 2"
        elseif elo >= 3000 then
            charm.charm_rank = "Onyx 1"
        elseif elo >= 2800 then
            charm.charm_rank = "Diamond 3"
        elseif elo >= 2600 then
            charm.charm_rank = "Diamond 2"
        elseif elo >= 2400 then
            charm.charm_rank = "Diamond 1"
        else
            charm.charm_rank = "Use Spoofed ELO"
        end

        -- Activar charms de temporada para que se vea el rango
        charm.s0_charm = true
        charm.s1_charm = true
        charm.s2_charm = true
        charm.s3_charm = true

        pcall(UpdateCharmOptions)
    end

    if lb.Streak then
        Unbind("Streak")
        ModifyLeaderboard("Streak", "Current Highest Win Streak", lb.streak_value)
        lb.old_streak_value = LocalPlayer:GetAttribute("StatisticDuelsWinStreak")
        LocalPlayer:SetAttribute("StatisticDuelsWinStreak", lb.streak_value)
    end

    if lb.Kills then
        Unbind("Kills")
        ModifyLeaderboard("Kills", "Most Eliminations", lb.kills_value)
    end

    if lb.Wins then
        Unbind("Wins")
        ModifyLeaderboard("Wins", "Most Wins", lb.wins_value)
    end

    if lb.Level then
        Unbind("Level")
        ModifyLeaderboard("Level", "Highest Level", lb.level_value)
        lb.old_level_value = LocalPlayer:GetAttribute("Level")
        LocalPlayer:SetAttribute("Level", lb.level_value)
    end

    if not lb.ELO then ResetLeaderboard("Highest ELO", "ELO") end

    if not lb.Streak then
        ResetLeaderboard("Current Highest Win Streak", "Streak")
        if lb.old_streak_value ~= nil then
            LocalPlayer:SetAttribute("StatisticDuelsWinStreak", lb.old_streak_value)
        end
    end

    if not lb.Kills then ResetLeaderboard("Most Eliminations", "Kills") end

    if not lb.Wins then ResetLeaderboard("Most Wins", "Wins") end

    if not lb.Level then
        ResetLeaderboard("Highest Level", "Level")
        if lb.old_level_value ~= nil then
            LocalPlayer:SetAttribute("Level", lb.old_level_value)
        end
    end
end

-- Mantener atributo de racha forzado (perfil + tab)
task.spawn(function()
    while true do
        task.wait(1)
        local lb = states.spoofer_state.leaderboard
        if lb and lb.Streak then
            pcall(function()
                LocalPlayer:SetAttribute("StatisticDuelsWinStreak", lb.streak_value)
            end)
        end
    end
end)

-- FORCE RANK (Nemesis / Archnemesis) - perfil + tab + charms
task.spawn(function()
    while true do
        task.wait(0.75)
        local lb = states.spoofer_state.leaderboard
        if not (lb and lb.ELO) then continue end

        local elo = tonumber(lb.elo_value) or 0
        local rankName = nil
        local arch = 1
        if elo >= 10000 then
            rankName = "Archnemesis"
            arch = 3
        elseif elo >= 3600 then
            rankName = "Nemesis"
            arch = 1
        end
        if not rankName then continue end

        -- Forzar charm
        local charm = states.spoofer_state.charm
        charm.charm_rank = rankName
        charm.arch_rank = arch
        charm.s0_charm = true
        charm.s1_charm = true
        charm.s2_charm = true
        charm.s3_charm = true
        pcall(UpdateCharmOptions)

        -- Forzar texto de rango en toda la UI
        for _, obj in ipairs(game:GetDescendants()) do
            if obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox") then
                local t = tostring(obj.Text or "")
                local lower = string.lower(t)
                if lower:find("bronze")
                or lower:find("silver")
                or lower:find("gold")
                or lower:find("platinum")
                or lower:find("diamond")
                or lower:find("onyx")
                or lower:find("nemesis")
                or lower:find("archnemesis")
                or lower:find("arch")
                or lower:find("unranked") then
                    if obj.Text ~= rankName then
                        pcall(function() obj.Text = rankName end)
                    end
                end
            end
        end
    end
end)

pcall(function()
    if not modules.SeasonLibrary then return end
    local original = modules.SeasonLibrary.FormatSeasonRankCharm
    modules.SeasonLibrary.FormatSeasonRankCharm = function(self, p2, p3, p4, p5)
        local season_num = kSeasonNameMap[p3]
        local elo = kCharmEloMap[states.spoofer_state.charm.charm_rank]
            or states.spoofer_state.leaderboard.elo_value
        if season_num and kSeasonToggleMap[season_num]
            and states.spoofer_state.charm[kSeasonToggleMap[season_num]] then
            p4 = elo or p4
            p5 = states.spoofer_state.charm.arch_rank
        end
        -- Si Spoof ELO está activo, forzar elo/rank del charm
        local lb = states.spoofer_state.leaderboard
        if lb and lb.ELO then
            local e = tonumber(lb.elo_value) or 0
            if e >= 10000 then
                p4 = e
                p5 = 3
            elseif e >= 3600 then
                p4 = e
                p5 = 1
            end
        end
        return original(self, p2, p3, p4, p5)
    end
end)

function UpdateCharmOptions()
    task.spawn(function()
        -- aplicar ya y de nuevo a los 1s (por si el modelo carga tarde)
        local function apply()
            if not modules.SeasonLibrary then return end
            local charm = states.spoofer_state.charm
            local elo = kCharmEloMap[charm.charm_rank]
            if charm.charm_rank == "Use Spoofed ELO" or not elo then
                elo = states.spoofer_state.leaderboard.elo_value
            end
            -- Nemesis / Archnemesis fuerza elo alto
            if charm.charm_rank == "Nemesis" then
                elo = 3600
            elseif charm.charm_rank == "Archnemesis" then
                elo = 3600
            end
            for _, d in ipairs(workspace:GetDescendants()) do
                if not d:IsA("Model") or not d:FindFirstChild("Extra") then continue end
                local name = d.Name
                if string.sub(name, 1, 7) ~= "Season " then continue end
                local season_num = tonumber(string.sub(name, 8))
                if not season_num or not kSeasonToggleMap[season_num] then continue end
                if not charm[kSeasonToggleMap[season_num]] then continue end
                local season_name = kSeasonNameMap[season_num]
                if not season_name then continue end
                pcall(function()
                    modules.SeasonLibrary:FormatSeasonRankCharm(d, season_name, elo, charm.arch_rank)
                end)
            end
        end
        apply()
        task.wait(1)
        apply()
    end)
end

function UpdateCurrencyOptions()
    pcall(function()
        local cd = modules.PlayerDataController and modules.PlayerDataController.CurrentData
        if not cd then return end
        cd:Replicate("WeaponKeys")
        cd:Replicate("UnlockTokens")
        cd:Replicate("EventCurrency")
        cd:Replicate("Glory")
        cd:Replicate("SkinTickets")
    end)
end

function UpdateDuelHistory()
    local dh = states.spoofer_state.duel_history
    local list = dh.duel_history_value
    if type(list) ~= "table" then return end
    local match = list[dh.match_index]
    if not match then return end

    match.QueueName = dh.mode == "Ranked" and "Ranked" or "Casual"
    match.Map = dh.map
    match.Scores = {dh.team1score, dh.team2score}
    match.WinningTeamIndex = dh.result == "Victory" and 1 or 2

    if match.Duelers and match.Duelers[dh.dueler_index] then
        local me = match.Duelers[dh.dueler_index]
        me.Username = dh.username
        me.DisplayName = dh.display_name
        me.Eliminations = dh.kills
        me.Deaths = dh.deaths
        me.Assists = dh.assists
        me.Damage = dh.damage
        if dh.mode == "Ranked" then
            me.DisplayELO = dh.elo
            if type(dh.logged_elo_events) == "table" and dh.logged_elo_events[dh.match_index] then
                dh.logged_elo_events[dh.match_index].CurrentELO = dh.elo
                dh.logged_elo_events[dh.match_index].ELOIncrement = dh.elo_change or 0
            end
        end
    end

    pcall(function()
        local cd = modules.PlayerDataController and modules.PlayerDataController.CurrentData
        if cd then
            cd:Replicate("DuelHistory")
            cd:Replicate("LoggedELOEvents")
        end
    end)
end

local NamecallDispatcher = {
    Hooks = {},
    Original = nil,
}

function NamecallDispatcher:Register(callback)
    self.Hooks[#self.Hooks + 1] = callback
    return #self.Hooks
end

function NamecallDispatcher:Unregister(index)
    table.remove(self.Hooks, index)
end

function NamecallDispatcher:Clear()
    table.clear(self.Hooks)
end

function NamecallDispatcher:CallOriginal(object, ...)
    if self.Original then return self.Original(object, ...) end
    return nil
end

pcall(function()
    NamecallDispatcher.Original = hookmetamethod(game, "__namecall", newcclosure(function(object, ...)
        local hooks = NamecallDispatcher.Hooks
        local count = #hooks
        if count == 0 then return NamecallDispatcher.Original(object, ...) end
        local method = getnamecallmethod()
        for i = 1, count do
            local result = hooks[i](object, method, ...)
            if result ~= nil and result ~= false then
                if result == true then return nil end
                return result
            end
        end
        return NamecallDispatcher.Original(object, ...)
    end))
end)

function CloneCosmetic(name, cosmetic_type, options)
    if not modules.CosmeticLibrary or not modules.CosmeticLibrary.Cosmetics then return nil end
    local base = modules.CosmeticLibrary.Cosmetics[name]
    if not base then return nil end
    local data = table.clone(base)
    data.Name = name
    data.Type = data.Type or cosmetic_type
    data.Seed = math.random(1, 1000000)
    if options then
        if options.inverted then data.Inverted = true end
        if options.favorites_only then data.OnlyUseFavorites = true end
    end
    return data
end

function HandleSkinchangerEquip(args)
    local weapon_name = args[1]
    local cosmetic_type = args[2]
    local cosmetic_name = args[3]
    local options = args[4] or {}

    states.skinchanger_state.equipped[weapon_name] = states.skinchanger_state.equipped[weapon_name] or {}

    if not cosmetic_name or cosmetic_name == "" or cosmetic_name == "None" or cosmetic_name == "NONE_COSMETIC" then
        states.skinchanger_state.equipped[weapon_name][cosmetic_type] = { Name = "NONE_COSMETIC" }
    elseif cosmetic_name == "Random" or cosmetic_name == "RANDOM_COSMETIC" then
        states.skinchanger_state.equipped[weapon_name][cosmetic_type] = {
            Name = cosmetic_name,
            Type = cosmetic_type,
            Inverted = options.IsInverted,
            OnlyUseFavorites = options.OnlyUseFavorites,
            Seed = math.random(1, 1000000),
        }
    else
        local cloned = CloneCosmetic(cosmetic_name, cosmetic_type, {
            inverted = options.IsInverted,
            favorites_only = options.OnlyUseFavorites,
        })
        if cloned then
            states.skinchanger_state.equipped[weapon_name][cosmetic_type] = cloned
        end
    end

    task.spawn(function()
        task.wait(0.1)
        pcall(function()
            if modules.PlayerDataController and modules.PlayerDataController.CurrentData then
                modules.PlayerDataController.CurrentData:Replicate("WeaponInventory")
            end
        end)
    end)
end

function HandleSkinchangerFavorite(args)
    local weapon_name = args[1]
    local cosmetic_name = args[2]
    local state = args[3]
    states.skinchanger_state.favorites = states.skinchanger_state.favorites or {}
    states.skinchanger_state.favorites[weapon_name] = states.skinchanger_state.favorites[weapon_name] or {}
    states.skinchanger_state.favorites[weapon_name][cosmetic_name] = state or nil
end

pcall(function()
    NamecallDispatcher:Register(function(self, method, ...)
        if method ~= "FireServer" then return false end
        local args = {...}
        if self.Name == "EquipCosmetic" then
            HandleSkinchangerEquip(args)
            return true
        end
        if self.Name == "FavoriteCosmetic" then
            HandleSkinchangerFavorite(args)
            return true
        end
        return false
    end)
end)

function ResolveCosmetic(weaponName, cosmeticType)
    local equipped = states.skinchanger_state.equipped[weaponName]
        and states.skinchanger_state.equipped[weaponName][cosmeticType]
    if not equipped then return nil end
    if equipped.Name == "None" or equipped.Name == "NONE_COSMETIC" then return nil end
    return equipped
end

pcall(function()
    if not modules.ItemLibrary then return end
    local orig = modules.ItemLibrary.GetViewModelImageFromWeaponData
    modules.ItemLibrary.GetViewModelImageFromWeaponData = function(p1, p2, p3)
        if not p2 then return orig(p1, p2, p3) end
        local weapon_name = p2.Name
        local skin = ResolveCosmetic(weapon_name, "Skin")
        local should_show = skin and (
            (p2.Skin and states.skinchanger_state.equipped[weapon_name])
            or (states.skinchanger_state.viewing_profile == LocalPlayer and states.skinchanger_state.equipped[weapon_name])
        )
        if should_show and skin then
            local skin_info = p1.ViewModels[skin.Name]
            if skin_info then
                return skin_info[p3 and "ImageHighResolution" or "Image"] or skin_info.Image
            end
        end
        return orig(p1, p2, p3)
    end
end)

pcall(function()
    local ok, clientItemPath = pcall(function()
        return LocalPlayer.PlayerScripts.Modules.ClientReplicatedClasses.ClientFighter.ClientItem
    end)
    if not ok or not clientItemPath then return end
    local ClientItem = require(clientItemPath)

    if ClientItem._CreateViewModel then
        local orig = ClientItem._CreateViewModel
        ClientItem._CreateViewModel = function(self, viewmodelRef)
            local weaponName = self.Name
            local weaponPlayer = self.ClientFighter and self.ClientFighter.Player
            states.skinchanger_state.constructing_weapon = (weaponPlayer == LocalPlayer) and weaponName or nil

            if weaponPlayer == LocalPlayer and states.skinchanger_state.equipped[weaponName] and ResolveCosmetic(weaponName, "Skin") and viewmodelRef then
                pcall(function()
                    local ReplicatedClass = require(ReplicatedStorage.Modules.ReplicatedClass)
                    local dataKey = ReplicatedClass:ToEnum("Data")
                    local skinKey = ReplicatedClass:ToEnum("Skin")
                    local nameKey = ReplicatedClass:ToEnum("Name")

                    if viewmodelRef[dataKey] then
                        viewmodelRef[dataKey][skinKey] = ResolveCosmetic(weaponName, "Skin")
                        viewmodelRef[dataKey][nameKey] = ResolveCosmetic(weaponName, "Skin").Name
                    elseif viewmodelRef.Data then
                        viewmodelRef.Data.Skin = ResolveCosmetic(weaponName, "Skin")
                        viewmodelRef.Data.Name = ResolveCosmetic(weaponName, "Skin").Name
                    end
                end)
            end

            local result = orig(self, viewmodelRef)
            states.skinchanger_state.constructing_weapon = nil
            return result
        end
    end

    pcall(function()
        local viewModelModule = LocalPlayer.PlayerScripts.Modules.ClientReplicatedClasses.ClientFighter.ClientItem:FindFirstChild("ClientViewModel")
        if not viewModelModule then return end
        local ClientViewModel = require(viewModelModule)

        if ClientViewModel.GetWrap then
            local origGetWrap = ClientViewModel.GetWrap
            ClientViewModel.GetWrap = function(self)
                local weaponName = self.ClientItem and self.ClientItem.Name
                local weaponPlayer = self.ClientItem and self.ClientItem.ClientFighter and self.ClientItem.ClientFighter.Player
                if weaponName and weaponPlayer == LocalPlayer and states.skinchanger_state.equipped[weaponName] and ResolveCosmetic(weaponName, "Wrap") then
                    return ResolveCosmetic(weaponName, "Wrap")
                end
                return origGetWrap(self)
            end
        end

        local origNew = ClientViewModel.new
        ClientViewModel.new = function(replicatedData, clientItem)
            local weaponPlayer = clientItem.ClientFighter and clientItem.ClientFighter.Player
            local weaponName = states.skinchanger_state.constructing_weapon or clientItem.Name

            if weaponPlayer == LocalPlayer and states.skinchanger_state.equipped[weaponName] then
                pcall(function()
                    local ReplicatedClass = require(ReplicatedStorage.Modules.ReplicatedClass)
                    local dataKey = ReplicatedClass:ToEnum("Data")
                    replicatedData[dataKey] = replicatedData[dataKey] or {}

                    if ResolveCosmetic(weaponName, "Skin") then
                        replicatedData[dataKey][ReplicatedClass:ToEnum("Skin")] = ResolveCosmetic(weaponName, "Skin")
                    end
                    if ResolveCosmetic(weaponName, "Wrap") then
                        replicatedData[dataKey][ReplicatedClass:ToEnum("Wrap")] = ResolveCosmetic(weaponName, "Wrap")
                    end
                    if ResolveCosmetic(weaponName, "Charm") then
                        replicatedData[dataKey][ReplicatedClass:ToEnum("Charm")] = ResolveCosmetic(weaponName, "Charm")
                    end
                end)
            end

            local result = origNew(replicatedData, clientItem)

            pcall(function()
                local objectID = nil
                pcall(function()
                    local ReplicatedClass = require(ReplicatedStorage.Modules.ReplicatedClass)
                    local dataKey = ReplicatedClass:ToEnum("Data")
                    local objKey = ReplicatedClass:ToEnum("ObjectID")
                    if replicatedData[dataKey] then
                        objectID = replicatedData[dataKey][objKey]
                    end
                end)
                if objectID == nil then
                    pcall(function() objectID = result:Get("ObjectID") or result.ObjectID end)
                end
                if objectID and weaponPlayer == LocalPlayer and states.skinchanger_state.equipped[weaponName] then
                    states.skinchanger_state.placed_object_map[objectID] = weaponName
                end
            end)

            if weaponPlayer == LocalPlayer and states.skinchanger_state.equipped[weaponName] and ResolveCosmetic(weaponName, "Wrap") and result._UpdateWrap then
                task.spawn(function()
                    result:_UpdateWrap()
                    task.wait(0.1)
                    if not result._destroyed then result:_UpdateWrap() end
                end)
            end

            return result
        end
    end)
end)

function FindContainerWithRobuxPrice()
    for _, child in ipairs(CoreGui:GetChildren()) do
        local found = child:FindFirstChild("RobuxPrice", true)
        if found then return child end
    end
    return nil
end

function AddGamepass(gamepass_id)
    states.spoofer_state.marketplace.fake_bundles_owned[modules.MonetizationLibrary:GetGamepassName(gamepass_id)] = true
    pcall(function()
        if modules.PlayerDataController and modules.PlayerDataController.CurrentData then
            modules.PlayerDataController.CurrentData:Replicate("GamepassBundlesClaimed")
        end
    end)
end

function AddReward(reward_name, reward_amount, reward_weapon)
    if reward_name == "Key" then
        states.spoofer_state.currency.weapon_keys_value += reward_amount
        pcall(function()
            if modules.PlayerDataController and modules.PlayerDataController.CurrentData then
                modules.PlayerDataController.CurrentData:Replicate("WeaponKeys")
            end
        end)
    elseif modules.CosmeticLibrary and modules.CosmeticLibrary.Cosmetics and modules.CosmeticLibrary.Cosmetics[reward_name] then
        local reward_info = modules.CosmeticLibrary.Cosmetics[reward_name]
        if reward_info.Type == "Skin" or reward_info.Type == "Wrap" or reward_info.Type == "Charm" or reward_info.Type == "Finisher" or reward_info.Type == "Emote" or reward_info.Type == "Gesture" then
            states.skinchanger_state.fake_owned[reward_name] = true
        end
    elseif modules.ShopLibrary and modules.ShopLibrary.Weapons and modules.ShopLibrary.Weapons[reward_name] then
        local owned = {}
        for _, weapon_data in pairs(states.skinchanger_state.fake_weapon_owned) do
            if type(weapon_data) == "table" and weapon_data.Name then
                owned[weapon_data.Name] = true
            end
        end
        if not owned[reward_name] then
            table.insert(states.skinchanger_state.fake_weapon_owned, {
                Name = reward_name,
                Level = 1,
                XP = 0,
                IsFavorited = false,
                Skin = nil,
            })
        end
        pcall(function()
            if modules.PlayerDataController and modules.PlayerDataController.CurrentData then
                modules.PlayerDataController.CurrentData:Replicate("WeaponInventory")
            end
        end)
    elseif modules.CosmeticLibrary and modules.CosmeticLibrary.Rewards and modules.CosmeticLibrary.Rewards[reward_name] then
        local reward_info = modules.CosmeticLibrary.Rewards[reward_name]
        local reward_entry
        if reward_info.Type == "Skin" then
            if not reward_weapon or reward_weapon == "" then return end
            reward_entry = { Name = reward_name, Quantity = reward_amount, Weapon = reward_weapon }
        else
            reward_entry = { Name = reward_name, Quantity = reward_amount }
        end
        local found = false
        for _, entry in ipairs(states.inventory_state.fake_owned) do
            if entry.Name == reward_entry.Name and entry.Weapon == reward_entry.Weapon and entry.Quantity < 99 then
                entry.Quantity = (entry.Quantity or 1) + (reward_entry.Quantity or 1)
                found = true
                break
            end
        end
        if not found then
            table.insert(states.inventory_state.fake_owned, reward_entry)
        end
        pcall(function()
            if modules.PlayerDataController and modules.PlayerDataController.CurrentData then
                modules.PlayerDataController.CurrentData:Replicate("UnclaimedRewards")
            end
        end)
    end
end

local product_type = "product"
local marketplace_asset_id = 0
function UpdateMarketplace()
    if not states.spoofer_state.marketplace.fake_robux then return end

    local foundation_overlay = FindContainerWithRobuxPrice()
    if not foundation_overlay or not foundation_overlay:FindFirstChild("SafeAreaFrame") then
        return
    end

    if not marketplace_asset_id or marketplace_asset_id == 0 then
        return
    end

    local robux_text = string.format(
        '<font family="rbxasset://LuaPackages/Packages/_Index/BuilderIcons/BuilderIcons/BuilderIcons.json" weight="400">robux</font> %s',
        FormatNumber(states.spoofer_state.marketplace.robux_amount)
    )

    local info_type = product_type == "gamepass" and Enum.InfoType.GamePass or product_type == "product" and Enum.InfoType.Product or Enum.InfoType.Asset
    local product_info = MarketplaceService:GetProductInfo(marketplace_asset_id, info_type)
    local has_enough = product_info.PriceInRobux <= states.spoofer_state.marketplace.robux_amount

    local BuyButton
    local PromptTitle
    local RobuxBalance
    local SheetContentContainer

    for _, descendant in ipairs(foundation_overlay:GetDescendants()) do
        local parent = descendant.Parent
        local grandparent = parent and parent.Parent

        if descendant.Name == "RobuxPrice" and descendant:IsA("TextLabel") and grandparent and grandparent.Name == "ModalHeader" then
            descendant.RichText = true
            descendant.Text = robux_text
            RobuxBalance = descendant
        elseif descendant.Name == "RobuxAmount" and descendant:IsA("TextLabel") and grandparent and grandparent.Name == "ModalHeader" then
            descendant.Text = has_enough and "Buy item" or "Buy Robux and item"
            PromptTitle = descendant
        elseif descendant:IsA("ImageButton") and parent and parent.Name == "1" then
            BuyButton = descendant
        elseif descendant.Name == "SheetContentContainer" and descendant:IsA("Frame") and parent and parent.Name == "ScrollingFrame" then
            SheetContentContainer = descendant
        elseif descendant.Name == "UpsellSuggestions" and descendant:IsA("Frame") and parent and parent.Name == "SheetContentContainer" and has_enough then
            descendant:Destroy()
        elseif descendant.Name == "DisclosureTerms" and descendant:IsA("Frame") and parent and parent.Name == "1" and has_enough then
            descendant:Destroy()
        end
    end

    if BuyButton then
        local OverlayButton = foundation_overlay:FindFirstChild("nigga")

        if OverlayButton then
            return
        end

        OverlayButton = Instance.new("TextButton")
        OverlayButton.Name = "nigga"
        OverlayButton.BackgroundTransparency = 1
        OverlayButton.Text = ""
        OverlayButton.AutoButtonColor = false
        OverlayButton.ZIndex = BuyButton.ZIndex + 100
        OverlayButton.Size = UDim2.fromOffset(BuyButton.AbsoluteSize.X, BuyButton.AbsoluteSize.Y)
        OverlayButton.Position = UDim2.fromOffset(BuyButton.AbsolutePosition.X, BuyButton.AbsolutePosition.Y + BuyButton.AbsoluteSize.Y + (BuyButton.AbsoluteSize.Y / 2))
        OverlayButton.AnchorPoint = BuyButton.AnchorPoint
        OverlayButton.Parent = foundation_overlay

        BuyButton.Destroying:Connect(function()
            OverlayButton:Destroy()
            product_type = "product"
            marketplace_asset_id = 0
        end)

        RunService.Heartbeat:Connect(function()
            OverlayButton.Position = UDim2.fromOffset(BuyButton.AbsolutePosition.X, BuyButton.AbsolutePosition.Y + BuyButton.AbsoluteSize.Y + (BuyButton.AbsoluteSize.Y / 2))
        end)

        task.wait(3)

        local purchased = false
        OverlayButton.MouseButton1Click:Connect(function()
            local current_id = getthreadidentity()
            setthreadidentity(2)

            if purchased then
                GuiService:InspectPlayerFromUserId(LocalPlayer.UserId)
                GuiService:CloseInspectMenu()
                return
            end

            local currency_rewards = {}

            for _, bundle in pairs(modules.MonetizationLibrary.Bundles) do
                if bundle.ProductID == marketplace_asset_id or bundle.GamepassName == modules.MonetizationLibrary:GetGamepassName(marketplace_asset_id) then
                    currency_rewards = bundle.Rewards
                    break
                end
            end

            setthreadidentity(current_id)

            local original_bgcolor = BuyButton.BackgroundColor3
            BuyButton.BackgroundColor3 = Color3.fromRGB(38, 61, 143)

            local bgcolor_conn = BuyButton:GetPropertyChangedSignal("BackgroundColor3"):Connect(function()
                BuyButton.BackgroundColor3 = Color3.fromRGB(38, 61, 143)
            end)

            task.wait(2)

            bgcolor_conn:Disconnect()
            BuyButton.BackgroundColor3 = original_bgcolor

            if product_type == "gamepass" then
                MarketplaceService:SignalPromptGamePassPurchaseFinished(LocalPlayer, marketplace_asset_id, has_enough)
            elseif product_type == "product" then
                MarketplaceService:SignalPromptProductPurchaseFinished(LocalPlayer.UserId, marketplace_asset_id, has_enough)
            elseif product_type == "asset" then
                MarketplaceService:SignalPromptPurchaseFinished(LocalPlayer, marketplace_asset_id, has_enough)
            end

            PromptTitle.Text = "Purchase completed"
            PromptTitle.Name = "Title"
            RobuxBalance:Destroy()

            local BodyContainer = Instance.new("Frame")
            BodyContainer.Name = "Body"
            BodyContainer.Parent = SheetContentContainer.Parent
            BodyContainer.AutomaticSize = Enum.AutomaticSize.Y
            BodyContainer.BackgroundTransparency = 1
            BodyContainer.BorderSizePixel = 0
            BodyContainer.LayoutOrder = 1
            BodyContainer.Size = UDim2.fromScale(1, 0)
            BodyContainer.AutoLocalize = false
            SheetContentContainer:Destroy()

            local UIListLayout = Instance.new("UIListLayout")
            UIListLayout.Parent = BodyContainer
            UIListLayout.Padding = UDim.new(0, 16)
            UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
            UIListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
            UIListLayout.VerticalAlignment = Enum.VerticalAlignment.Center

            local UIPadding = Instance.new("UIPadding")
            UIPadding.Parent = BodyContainer
            UIPadding.PaddingTop = UDim.new(0, 8)

            local Icon = Instance.new("TextLabel")
            Icon.Name = "Icon"
            Icon.Parent = BodyContainer
            Icon.BackgroundTransparency = 1
            Icon.BorderSizePixel = 0
            Icon.LayoutOrder = 0
            Icon.Size = UDim2.fromOffset(48, 48)
            Icon.AutoLocalize = false
            Icon.FontFace = Font.new("rbxasset://LuaPackages/Packages/_Index/BuilderIcons/BuilderIcons/BuilderIcons.json", Enum.FontWeight.Regular)
            Icon.Text = "circle-check"
            Icon.TextColor3 = Color3.fromRGB(213, 215, 221)
            Icon.TextSize = 48

            local Message = Instance.new("TextLabel")
            Message.Name = "Message"
            Message.Parent = BodyContainer
            Message.AutomaticSize = Enum.AutomaticSize.XY
            Message.BackgroundTransparency = 1
            Message.BorderSizePixel = 0
            Message.Size = UDim2.new(0, 0, 0, 0)
            Message.AutoLocalize = false
            Message.FontFace = Font.new("rbxasset://fonts/families/LegacyArimo.json", Enum.FontWeight.Regular)
            Message.Text = "You have successfully bought " .. product_info.Name .. "."
            Message.TextColor3 = Color3.fromRGB(213, 215, 221)
            Message.TextSize = 15
            Message.TextWrapped = true
            Message.TextXAlignment = Enum.TextXAlignment.Left

            local label = BuyButton:FindFirstChildOfClass("TextLabel")
            if label then
                local clone = label:Clone()
                clone.Text = "OK"
                clone.Parent = label.Parent
                label:Destroy()
            end

            setthreadidentity(2)

            task.wait()
            modules.MonetizationController:_FinishPrompt(has_enough)

            if has_enough and product_type == "gamepass" then
                AddGamepass(marketplace_asset_id)
            end

            for _, reward in ipairs(currency_rewards) do
                AddReward(reward.Type or reward.Name, reward.Quantity or 1, reward.Weapon)
            end

            if has_enough and modules.MonetizationLibrary.Gifts[tostring(marketplace_asset_id)] and states.spoofer_state.marketplace.fake_gifting then
                local marketplace = states.spoofer_state.marketplace

                marketplace.fake_gift_robux_spent += product_info.PriceInRobux

                while true do
                    local required = modules.MonetizationLibrary:GetGiftRewardRobuxSpentRequirement(
                        marketplace.fake_gift_rewards_claimed
                    )
                    if marketplace.fake_gift_robux_spent < required then
                        break
                    end
                    marketplace.fake_gift_robux_spent -= required
                    marketplace.fake_gift_rewards_claimed += 1
                end

                modules.PlayerDataController.CurrentData:Replicate("GiftRobuxSpentProgress")
                modules.PlayerDataController.CurrentData:Replicate("GiftRobuxSpentRewardsClaimed")
            end

            purchased = true
        end)
    end
end

CoreGui.DescendantAdded:Connect(UpdateMarketplace)

task.spawn(function()
    pcall(function()
        local original = modules.MonetizationController.PromptProductPurchase

        modules.MonetizationController.PromptProductPurchase = function(self, product_id, ...)
            if product_id and product_id ~= 0 then
                marketplace_asset_id = product_id
                product_type = "product"
            end

            local results = {original(self, product_id, ...)}
            return table.unpack(results)
        end
    end)

    pcall(function()
        local original = modules.MonetizationController.PromptGamePassPurchase

        modules.MonetizationController.PromptGamePassPurchase = function(self, gamepass_id, ...)
            if gamepass_id and gamepass_id ~= 0 then
                marketplace_asset_id = gamepass_id
                product_type = "gamepass"
            end

            local results = {original(self, gamepass_id, ...)}
            return table.unpack(results)
        end
    end)

    pcall(function()
        local original = modules.MonetizationController.PromptPurchase

        modules.MonetizationController.PromptPurchase = function(self, asset_id)
            if asset_id and asset_id ~= 0 then
                marketplace_asset_id = asset_id
                product_type = "asset"
            end

            local results = {original(self, asset_id)}
            return table.unpack(results)
        end
    end)

    pcall(function()
        local remote = ReplicatedStorage.Remotes.Misc.PromptGiftProductPurchase

        NamecallDispatcher:Register(function(self, method, ...)
            if method ~= "InvokeServer" or self ~= remote then
                return false
            end

            local args = { ... }

            if args[1] and args[1] ~= 0 then
                marketplace_asset_id = args[1]
                product_type = "product"
            end

            return NamecallDispatcher:CallOriginal(self, ...)
        end)
    end)
end)

local kCosmeticTypes = {"Skin", "Wrap", "Charm", "Finisher", "Emote"}
local kCosmeticRarities = {"Common", "Rare", "Legendary", "Mythical", "Unique", "Unobtainable"}

function GetCosmeticsByRarity(rarity)
    local results = {}
    if not modules.CosmeticLibrary or not modules.CosmeticLibrary.Cosmetics then return results end
    for name, cosmetic in pairs(modules.CosmeticLibrary.Cosmetics) do
        if cosmetic.Rarity == rarity then table.insert(results, name) end
    end
    return results
end

function GetCosmeticsByType(type_name)
    local results = {}
    if not modules.CosmeticLibrary or not modules.CosmeticLibrary.Cosmetics then return results end
    for name, cosmetic in pairs(modules.CosmeticLibrary.Cosmetics) do
        if cosmetic.Type == type_name then table.insert(results, name) end
    end
    return results
end

function GetSpecificCosmetic(type_name, cos_name, weapon_name)
    local result
    if not modules.CosmeticLibrary or not modules.CosmeticLibrary.Cosmetics then return result end
    for name, cosmetic in pairs(modules.CosmeticLibrary.Cosmetics) do
        if type_name == "Skin" then
            if name == cos_name and cosmetic.Type == type_name and cosmetic.ItemName == weapon_name then
                result = name
            end
        else
            if name == cos_name and cosmetic.Type == type_name then
                result = name
            end
        end
    end
    return result
end

function GetAllCosmeticsOfWeapon(weapon_name)
    local results = {}
    if not modules.CosmeticLibrary or not modules.CosmeticLibrary.Cosmetics then return results end
    for name, cosmetic in pairs(modules.CosmeticLibrary.Cosmetics) do
        if cosmetic.Type == "Skin" then
            if cosmetic.ItemName == weapon_name then table.insert(results, name) end
        else
            table.insert(results, name)
        end
    end
    return results
end

function GetAllCosmetics()
    local results = {}
    if not modules.CosmeticLibrary or not modules.CosmeticLibrary.Cosmetics then return results end
    for name, _ in pairs(modules.CosmeticLibrary.Cosmetics) do
        table.insert(results, name)
    end
    return results
end

function UnlockSelectedRarity()
    local list = GetCosmeticsByRarity(states.skinchanger_state.general_unlocker.unlock_rarity)
    for _, cosmetic in ipairs(list) do
        if cosmetic:find("MISSING_") then continue end
        states.skinchanger_state.fake_owned[cosmetic] = true
    end
end

function LockSelectedRarity()
    local list = GetCosmeticsByRarity(states.skinchanger_state.general_unlocker.unlock_rarity)
    for _, cosmetic in ipairs(list) do
        if cosmetic:find("MISSING_") then continue end
        states.skinchanger_state.fake_owned[cosmetic] = nil
    end
end

function UnlockAllOfType()
    local list = GetCosmeticsByType(states.skinchanger_state.general_unlocker.unlock_type)
    for _, cosmetic in ipairs(list) do
        if cosmetic:find("MISSING_") then continue end
        states.skinchanger_state.fake_owned[cosmetic] = true
    end
end

function LockAllOfType()
    local list = GetCosmeticsByType(states.skinchanger_state.general_unlocker.unlock_type)
    for _, cosmetic in ipairs(list) do
        if cosmetic:find("MISSING_") then continue end
        states.skinchanger_state.fake_owned[cosmetic] = nil
    end
end

function UnlockSpecific()
    local unlocker = states.skinchanger_state.specific_unlocker
    local cosmetic = GetSpecificCosmetic(unlocker.unlock_type, unlocker.cosmetic_name, unlocker.weapon_name)
    if cosmetic and not cosmetic:find("MISSING_") then
        states.skinchanger_state.fake_owned[cosmetic] = true
    end
end

function LockSpecific()
    local unlocker = states.skinchanger_state.specific_unlocker
    local cosmetic = GetSpecificCosmetic(unlocker.unlock_type, unlocker.cosmetic_name, unlocker.weapon_name)
    if cosmetic and not cosmetic:find("MISSING_") then
        states.skinchanger_state.fake_owned[cosmetic] = nil
    end
end

function UnlockSpecificWeapon()
    local weapon_name = states.skinchanger_state.specific_unlocker.weapon_name
    local current_data = modules.PlayerDataController and modules.PlayerDataController.CurrentData
    if not current_data then return end
    if weapon_name == nil or weapon_name == "" then return end

    local inventory = current_data:Get("WeaponInventory")
    local owned = {}
    if type(inventory) == "table" then
        for _, weapon_data in pairs(inventory) do
            if type(weapon_data) == "table" and weapon_data.Name then
                owned[weapon_data.Name] = true
            end
        end
    end

    if not owned[weapon_name] then
        table.insert(states.skinchanger_state.fake_weapon_owned, {
            Name = weapon_name,
            Level = 1,
            XP = 0,
            IsFavorited = false,
            Skin = nil,
        })
        states.skinchanger_state.fake_weapon_names[weapon_name] = true
    end
end

function LockSpecificWeapon()
    local weapon_name = states.skinchanger_state.specific_unlocker.weapon_name
    if not states.skinchanger_state.fake_weapon_names[weapon_name] then return end
    local fake = states.skinchanger_state.fake_weapon_owned
    for i = #fake, 1, -1 do
        if fake[i].Name == weapon_name then
            table.remove(fake, i)
            states.skinchanger_state.fake_weapon_names[weapon_name] = nil
            break
        end
    end
end

function UnlockAllWeapons()
    local current_data = modules.PlayerDataController and modules.PlayerDataController.CurrentData
    if not current_data then return end

    local inventory = current_data:Get("WeaponInventory")
    local owned = {}
    if type(inventory) == "table" then
        for _, weapon_data in pairs(inventory) do
            if type(weapon_data) == "table" and weapon_data.Name then
                owned[weapon_data.Name] = true
            end
        end
    end

    if not modules.ShopLibrary or not modules.ShopLibrary.GetReleasedOwnableWeapons then return end
    local ownables = modules.ShopLibrary:GetReleasedOwnableWeapons()
    for _, weapon_name in pairs(ownables) do
        if not owned[weapon_name] and not states.skinchanger_state.fake_weapon_names[weapon_name] then
            table.insert(states.skinchanger_state.fake_weapon_owned, {
                Name = weapon_name,
                Level = 1,
                XP = 0,
                IsFavorited = false,
                Skin = nil,
            })
            states.skinchanger_state.fake_weapon_names[weapon_name] = true
        end
    end
end

function LockAllWeapons()
    local fake = states.skinchanger_state.fake_weapon_owned
    local names = states.skinchanger_state.fake_weapon_names
    for i = #fake, 1, -1 do
        if names[fake[i].Name] then
            table.remove(fake, i)
        end
    end
    table.clear(names)
end

function UnlockAllForWeapon()
    local weapon_name = states.skinchanger_state.specific_unlocker.weapon_name
    local list = GetAllCosmeticsOfWeapon(weapon_name)
    for _, cosmetic in ipairs(list) do
        if cosmetic:find("MISSING_") then continue end
        states.skinchanger_state.fake_owned[cosmetic] = true
    end
end

function LockAllForWeapon()
    local weapon_name = states.skinchanger_state.specific_unlocker.weapon_name
    local list = GetAllCosmeticsOfWeapon(weapon_name)
    for _, cosmetic in ipairs(list) do
        if cosmetic:find("MISSING_") then continue end
        states.skinchanger_state.fake_owned[cosmetic] = nil
    end
end

function UnlockAll()
    local list = GetAllCosmetics()
    for _, cosmetic in ipairs(list) do
        if cosmetic:find("MISSING_") then continue end
        states.skinchanger_state.fake_owned[cosmetic] = true
    end
end

function LockAll()
    local list = GetAllCosmetics()
    for _, cosmetic in ipairs(list) do
        if cosmetic:find("MISSING_") then continue end
        states.skinchanger_state.fake_owned[cosmetic] = nil
    end
end

function ReplicateSkinchanger()
    pcall(function()
        if modules.PlayerDataController and modules.PlayerDataController.CurrentData then
            modules.PlayerDataController.CurrentData:Replicate("CosmeticInventory")
            modules.PlayerDataController.CurrentData:Replicate("WeaponInventory")
        end
    end)
end


-- ==================== EMOTES: listar / equipar / jugar ====================
local _emoteTrack = nil

function findEmoteRemotes()
    local found = {}
    pcall(function()
        local root = ReplicatedStorage:FindFirstChild("Remotes")
        if not root then return end
        for _, d in ipairs(root:GetDescendants()) do
            local n = string.lower(d.Name or "")
            if (d:IsA("RemoteEvent") or d:IsA("RemoteFunction") or d:IsA("UnreliableRemoteEvent"))
                and (n:find("emote") or n:find("gesture") or n:find("dance")) then
                found[#found + 1] = d
            end
        end
    end)
    return found
end

function digAnimId(tbl, depth)
    depth = depth or 0
    if depth > 5 then return nil end
    if type(tbl) == "number" and tbl > 10000 then return tbl end
    if type(tbl) == "string" then
        local n = tbl:match("(%d%d%d%d%d%d+)")
        if n then return n end
        return nil
    end
    if type(tbl) ~= "table" then return nil end
    for _, key in ipairs({
        "AnimationId", "Animation", "AnimId", "AssetId", "EmoteAnimation",
        "EmoteId", "Anim", "RbxAnimationId", "Id", "animationId", "animation"
    }) do
        local v = rawget(tbl, key)
        if v ~= nil then
            local d = digAnimId(v, depth + 1)
            if d then return d end
        end
    end
    for k, v in pairs(tbl) do
        if type(k) == "string" and (k:lower():find("anim") or k:lower():find("emote")) then
            local d = digAnimId(v, depth + 1)
            if d then return d end
        end
    end
    for _, v in pairs(tbl) do
        if type(v) == "table" or type(v) == "string" or type(v) == "number" then
            local d = digAnimId(v, depth + 1)
            if d then return d end
        end
    end
    return nil
end

function resolveEmoteData(name)
    local cosLib = modules.CosmeticLibrary
    if not cosLib then return nil end
    for _, bagName in ipairs({"Cosmetics", "Rewards", "Emotes", "Gestures"}) do
        local bag = cosLib[bagName]
        if type(bag) == "table" and bag[name] then
            return bag[name]
        end
    end
    -- match case-insensitive / partial
    for _, bagName in ipairs({"Cosmetics", "Rewards", "Emotes"}) do
        local bag = cosLib[bagName]
        if type(bag) == "table" then
            for k, v in pairs(bag) do
                if type(k) == "string" and string.lower(k) == string.lower(name) then
                    return v, k
                end
            end
        end
    end
    return nil
end

function GetEmoteList()
    local list, seen = {}, {}
    pcall(function()
        local cosLib = modules.CosmeticLibrary
        if not cosLib then return end
        for _, bagName in ipairs({"Cosmetics", "Rewards", "Emotes", "Gestures"}) do
            local bag = cosLib[bagName]
            if type(bag) ~= "table" then continue end
            for name, cos in pairs(bag) do
                if type(name) ~= "string" or name:find("MISSING_") then continue end
                local t = type(cos) == "table" and tostring(cos.Type or cos.RewardType or cos.Category or "") or ""
                local ln = string.lower(name)
                local ok = t == "Emote" or t == "Gesture" or t == "Dance"
                    or t:lower():find("emote") or t:lower():find("gesture")
                    or ln:find("emote") or ln:find("gesture") or ln:find("dance")
                    or ln:find("floss") or ln:find("thang") or ln:find("gotta")
                -- si tiene anim id y no es skin/wrap, también
                if not ok and type(cos) == "table" and digAnimId(cos) then
                    if t == "" or (not t:find("Skin") and not t:find("Wrap") and not t:find("Charm") and not t:find("Finisher")) then
                        ok = true
                    end
                end
                if ok and not seen[name] then
                    seen[name] = true
                    list[#list + 1] = name
                end
            end
        end
    end)
    table.sort(list)
    if #list == 0 then list = {"(ninguno)"} end
    return list
end

function UnlockEmoteByName(name)
    if type(name) ~= "string" or name == "" or name == "(ninguno)" then return end
    states.skinchanger_state.fake_owned = states.skinchanger_state.fake_owned or {}
    states.skinchanger_state.fake_owned[name] = true
    -- también el nombre real de la lib
    local data, realName = resolveEmoteData(name)
    if realName and realName ~= name then
        states.skinchanger_state.fake_owned[realName] = true
    end
    pcall(function() if ReplicateSkinchanger then ReplicateSkinchanger() end end)
end

function EquipEmote(name)
    name = name or (states.emote_state and states.emote_state.selected)
    if type(name) ~= "string" or name == "" or name == "(ninguno)" then return false, "sin nombre" end

    local data, realName = resolveEmoteData(name)
    if realName then name = realName end

    UnlockEmoteByName(name)
    states.emote_state = states.emote_state or {}
    states.emote_state.equipped = name
    states.emote_state.selected = name
    states.emote_state.lastAnimId = nil

    local animId = data and digAnimId(data) or nil
    if animId then
        states.emote_state.lastAnimId = tostring(animId):gsub("%D", "")
    end

    pcall(function()
        states.skinchanger_state.equipped = states.skinchanger_state.equipped or {}
        states.skinchanger_state.equipped["Emote"] = states.skinchanger_state.equipped["Emote"] or {}
        local cloned = nil
        if CloneCosmetic then
            cloned = CloneCosmetic(name, "Emote", {})
        end
        states.skinchanger_state.equipped["Emote"]["Emote"] = cloned or { Name = name, Type = "Emote" }
    end)

    -- EquipCosmetic: muchos signatures
    local rem
    pcall(function() rem = ReplicatedStorage.Remotes.Data.EquipCosmetic end)
    if rem then
        local tries = {
            { "Emote", "Emote", name, {} },
            { "", "Emote", name, {} },
            { "Global", "Emote", name, {} },
            { 1, "Emote", name, {} },
            { 2, "Emote", name, {} },
            { name, "Emote", name, {} },
            { "Emote", "Gesture", name, {} },
            { nil, "Emote", name, {} },
        }
        for _, args in ipairs(tries) do
            pcall(function() rem:FireServer(table.unpack(args, 1, 4)) end)
        end
    end

    pcall(function()
        if HandleSkinchangerEquip then
            HandleSkinchangerEquip({ "Emote", "Emote", name, {} })
        end
    end)

    -- remotes de emote / equip
    for _, r in ipairs(findEmoteRemotes()) do
        pcall(function()
            if r:IsA("RemoteEvent") or r:IsA("UnreliableRemoteEvent") then
                r:FireServer(name)
                r:FireServer(name, true)
                r:FireServer("Emote", name)
            elseif r:IsA("RemoteFunction") then
                r:InvokeServer(name)
            end
        end)
    end

    pcall(function()
        if modules.PlayerDataController and modules.PlayerDataController.CurrentData then
            modules.PlayerDataController.CurrentData:Replicate("CosmeticInventory")
        end
    end)

    return true, name, animId
end

function StopEmotePlay()
    pcall(function()
        if _emoteTrack then
            _emoteTrack:Stop(0.15)
        end
    end)
    _emoteTrack = nil
    if states.emote_state then states.emote_state.playing = false end
end

function PlayEmoteByAnimId(animId)
    animId = tostring(animId or ""):gsub("%D", "")
    if animId == "" or #animId < 5 then return false end

    local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    local hum = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 3)
    if not hum then return false end

    local animator = hum:FindFirstChildOfClass("Animator")
    if not animator then
        animator = Instance.new("Animator")
        animator.Parent = hum
    end

    StopEmotePlay()

    local anim = Instance.new("Animation")
    anim.Name = "VnixEmote"
    anim.AnimationId = "rbxassetid://" .. animId

    local ok, track = pcall(function() return animator:LoadAnimation(anim) end)
    if not ok or not track then
        -- fallback Humanoid:LoadAnimation (legacy)
        ok, track = pcall(function() return hum:LoadAnimation(anim) end)
    end
    if not ok or not track then return false end

    local spd = (states.emote_state and tonumber(states.emote_state.speed)) or 1
    pcall(function() track.Priority = Enum.AnimationPriority.Action4 end)
    pcall(function() track.Looped = true end)
    pcall(function() track:AdjustSpeed(spd) end)
    track:Play(0.1)
    _emoteTrack = track
    if states.emote_state then
        states.emote_state.playing = true
        states.emote_state.lastAnimId = animId
    end

    task.spawn(function()
        while _emoteTrack == track and states.emote_state and states.emote_state.playing do
            task.wait(0.4)
            if track and not track.IsPlaying then
                pcall(function() track:Play(0.05) end)
            end
        end
    end)
    return true
end

function PlayEmote(name)
    name = name or (states.emote_state and states.emote_state.selected) or (states.emote_state and states.emote_state.equipped)

    -- solo números = asset id
    if type(name) == "string" then
        local onlyDigits = name:match("^%s*(%d%d%d%d%d+)%s*$")
        if onlyDigits then
            return PlayEmoteByAnimId(onlyDigits)
        end
    end

    if type(name) ~= "string" or name == "" or name == "(ninguno)" then
        return false
    end

    local okEquip, realName, animFromEquip = EquipEmote(name)
    name = realName or name

    -- 1) Remotes
    pcall(function()
        for _, r in ipairs(findEmoteRemotes()) do
            if r:IsA("RemoteEvent") or r:IsA("UnreliableRemoteEvent") then
                pcall(function() r:FireServer(name) end)
                pcall(function() r:FireServer(name, true) end)
                pcall(function() r:FireServer("Play", name) end)
            elseif r:IsA("RemoteFunction") then
                pcall(function() r:InvokeServer(name) end)
            end
        end
    end)

    -- 2) Anim id desde data / last
    local animId = animFromEquip
    if not animId then
        local data = resolveEmoteData(name)
        if data then animId = digAnimId(data) end
    end
    if not animId and states.emote_state and states.emote_state.lastAnimId then
        animId = states.emote_state.lastAnimId
    end

    if animId and PlayEmoteByAnimId(animId) then
        return true
    end

    -- 3) dígitos en el nombre
    local digits = tostring(name):match("(%d%d%d%d%d+)")
    if digits and PlayEmoteByAnimId(digits) then
        return true
    end

    return false
end

-- Emotes visuales (Type Emote / Gesture / Dance si existen)
function UnlockAllEmotes()
    local n = 0
    if modules.CosmeticLibrary and modules.CosmeticLibrary.Cosmetics then
        for name, cos in pairs(modules.CosmeticLibrary.Cosmetics) do
            if type(cos) == "table" then
                local t = tostring(cos.Type or "")
                if t == "Emote" or t == "Gesture" or t == "Dance" or t:lower():find("emote") then
                    if not tostring(name):find("MISSING_") then
                        states.skinchanger_state.fake_owned[name] = true
                        n += 1
                    end
                end
            end
        end
    end
    -- también por nombre por si Type no viene
    if modules.CosmeticLibrary and modules.CosmeticLibrary.Cosmetics then
        for name, _ in pairs(modules.CosmeticLibrary.Cosmetics) do
            local ln = string.lower(tostring(name))
            if ln:find("emote") or ln:find("gesture") then
                states.skinchanger_state.fake_owned[name] = true
            end
        end
    end
    ReplicateSkinchanger()
    return n
end

function LockAllEmotes()
    if modules.CosmeticLibrary and modules.CosmeticLibrary.Cosmetics then
        for name, cos in pairs(modules.CosmeticLibrary.Cosmetics) do
            if type(cos) == "table" then
                local t = tostring(cos.Type or "")
                if t == "Emote" or t == "Gesture" or t == "Dance" or t:lower():find("emote") then
                    states.skinchanger_state.fake_owned[name] = nil
                end
            end
        end
    end
    ReplicateSkinchanger()
end

function UnlockAllWeaponsAndNotify()
    UnlockAllWeapons()
    ReplicateSkinchanger()
end

function UnlockAllCosmeticsSkinchanger()
    UnlockAll()
    UnlockAllWeapons()
    UnlockAllEmotes()
    ReplicateSkinchanger()
end

function EquipApply()
    local EquipCosmeticRemote = ReplicatedStorage.Remotes.Data.EquipCosmetic
    local unlocker = states.skinchanger_state.equip_unlocker
    local unlock_type = unlocker.unlock_type
    local cosmetic_name = unlocker.cosmetic_name
    local weapon_name = unlocker.weapon_name
    local is_inverted = unlocker.inverted

    local name_to_send = if cosmetic_name ~= nil and cosmetic_name ~= "" then cosmetic_name else nil
    if not name_to_send then return end

    -- Emote / Gesture: no necesitan arma
    local isEmote = (unlock_type == "Emote" or unlock_type == "Gesture" or unlock_type == "Dance")
    if not isEmote and (typeof(weapon_name) ~= "string" or weapon_name == "") then return end

    local options = {}
    if unlock_type == "Wrap" then
        options.IsInverted = if is_inverted then true else nil
    end

    if isEmote then
        states.skinchanger_state.fake_owned[name_to_send] = true
        pcall(function() EquipCosmeticRemote:FireServer("Emote", unlock_type, name_to_send, options) end)
        pcall(function() EquipCosmeticRemote:FireServer("", unlock_type, name_to_send, options) end)
        pcall(function() EquipCosmeticRemote:FireServer(nil, unlock_type, name_to_send, options) end)
        pcall(function()
            if HandleSkinchangerEquip then
                HandleSkinchangerEquip({ "Emote", unlock_type, name_to_send, options })
            end
        end)
        pcall(function() if ReplicateSkinchanger then ReplicateSkinchanger() end end)
        return
    end

    EquipCosmeticRemote:FireServer(weapon_name, unlock_type, name_to_send, options)
end

function EquipApplyAll()
    local EquipCosmeticRemote = ReplicatedStorage.Remotes.Data.EquipCosmetic
    local unlocker = states.skinchanger_state.equip_unlocker
    local unlock_type = unlocker.unlock_type
    local cosmetic_name = unlocker.cosmetic_name
    local is_inverted = unlocker.inverted

    if unlock_type == "Skin" then return end
    if not modules.ItemLibrary or not modules.ItemLibrary.Items then return end

    for weapon_name, weapon_data in pairs(modules.ItemLibrary.Items) do
        if unlock_type == "Finisher" and not weapon_data.CanEliminate then
            continue
        end
        if typeof(weapon_name) ~= "string" or weapon_name == "" then
            continue
        end

        local name_to_send = if cosmetic_name ~= nil and cosmetic_name ~= "" then cosmetic_name else nil

        local options = {}
        if unlock_type == "Wrap" then
            options.IsInverted = if is_inverted then true else nil
        end

        EquipCosmeticRemote:FireServer(weapon_name, unlock_type, name_to_send, options)
    end
end




-- Notify sin Library
function Notify(title, text)
    if text == nil then text = title; title = "Vnix.lol" end
    pcall(function()
        game:GetService("StarterGui"):SetCore("SendNotification", {
            Title = tostring(title),
            Text = tostring(text),
            Duration = 3,
        })
    end)
end
if not getgenv().Toggles then getgenv().Toggles = {} end
if not getgenv().Options then getgenv().Options = {} end



-- ==================== CONFIG (custom UI flags) ====================
local CFG_FOLDER = "Vnix_lol_changer_gg/configs"
local CFG_AUTO = "Vnix_lol_changer_gg/autoload.txt"
local HttpService = game:GetService("HttpService")
local VnixFlags = {} -- [flag] = {type="toggle"/"value", get=fn, set=fn}

function cfgEnsure()
    pcall(function()
        if not isfolder("Vnix_lol_changer_gg") then makefolder("Vnix_lol_changer_gg") end
        if not isfolder(CFG_FOLDER) then makefolder(CFG_FOLDER) end
    end)
end
function cfgSanitize(n)
    n = tostring(n or "default"):gsub("[^%w%-%_]", "")
    return (n == "" and "default") or n
end
function cfgList()
    cfgEnsure()
    local list = {}
    pcall(function()
        if listfiles then
            for _, f in ipairs(listfiles(CFG_FOLDER) or {}) do
                local name = tostring(f):match("([^/\\]+)%.json$")
                if name then table.insert(list, name) end
            end
        end
    end)
    table.sort(list)
    if #list == 0 then list = {"(ninguna)"} end
    return list
end
-- Serializa tablas de config (salta funciones, convierte Color3)
local function cfgSerializeTable(src, depth)
    depth = depth or 0
    if depth > 6 or type(src) ~= "table" then return nil end
    local out = {}
    for k, v in pairs(src) do
        if type(k) == "string" and k:sub(1, 1) == "_" then
            -- skip _start/_stop/internal
        else
            local ty = typeof(v)
            if ty == "boolean" or ty == "number" or ty == "string" then
                out[k] = v
            elseif ty == "Color3" then
                out[k] = { __color3 = true, R = v.R, G = v.G, B = v.B }
            elseif ty == "table" then
                local sub = cfgSerializeTable(v, depth + 1)
                if sub and next(sub) then out[k] = sub end
            end
            -- EnumItem / function / Instance → skip
        end
    end
    return out
end

local function cfgApplyTable(dst, src)
    if type(dst) ~= "table" or type(src) ~= "table" then return end
    for k, v in pairs(src) do
        if type(k) ~= "string" or k:sub(1, 1) == "_" then
            -- skip
        elseif type(v) == "table" and v.__color3 then
            dst[k] = Color3.new(v.R or 1, v.G or 1, v.B or 1)
        elseif type(v) == "table" and type(dst[k]) == "table" and dst[k].__color3 == nil then
            -- don't deep-merge into random nested unless key exists as table of settings
            if typeof(dst[k]) == "table" then
                -- only apply plain value keys
                for k2, v2 in pairs(v) do
                    if type(v2) ~= "function" and type(k2) == "string" and k2:sub(1,1) ~= "_" then
                        if type(v2) == "table" and v2.__color3 then
                            dst[k][k2] = Color3.new(v2.R or 1, v2.G or 1, v2.B or 1)
                        elseif type(v2) ~= "table" then
                            dst[k][k2] = v2
                        end
                    end
                end
            end
        elseif type(v) ~= "function" and type(v) ~= "table" then
            dst[k] = v
        elseif type(v) == "table" and v.__color3 == nil then
            -- assign plain serializable fields onto dst
            if dst[k] == nil or type(dst[k]) ~= "table" then
                -- for simple option bags
                for k2, v2 in pairs(v) do
                    if type(v2) == "boolean" or type(v2) == "number" or type(v2) == "string" then
                        -- only if dst already has the key as leaf - skip creating random
                    end
                end
            end
            -- prefer flat: if dst has key as non-function value type matching
            if type(dst[k]) ~= "function" and (type(dst[k]) == "boolean" or type(dst[k]) == "number" or type(dst[k]) == "string" or type(dst[k]) == "nil") then
                -- shouldn't replace with table
            elseif type(dst[k]) == "table" then
                cfgApplyTable(dst[k], v)
            elseif dst[k] == nil and (type(v) == "boolean" or type(v) == "number" or type(v) == "string") then
                dst[k] = v
            end
        end
    end
end

-- apply flat keys from serialized module onto live module
local function cfgApplyModule(dst, src)
    if type(dst) ~= "table" or type(src) ~= "table" then return end
    for k, v in pairs(src) do
        if type(k) == "string" and k:sub(1, 1) ~= "_" then
            if type(v) == "table" and v.__color3 then
                dst[k] = Color3.new(v.R or 1, v.G or 1, v.B or 1)
            elseif type(v) == "boolean" or type(v) == "number" or type(v) == "string" then
                dst[k] = v
            end
        end
    end
end

function collectState()
    local t = { toggles = {}, options = {}, version = 4, matcha = true }
    for flag, meta in pairs(VnixFlags) do
        local ok, v = pcall(meta.get)
        if ok and v ~= nil then
            local ty = typeof(v)
            if ty == "boolean" then t.toggles[flag] = v
            elseif ty == "number" or ty == "string" then t.options[flag] = v end
        end
    end
    -- skins
    pcall(function()
        local sc = states.skinchanger_state
        if type(sc) ~= "table" then return end
        local equipped_out = {}
        if type(sc.equipped) == "table" then
            for weapon, types in pairs(sc.equipped) do
                if type(types) == "table" then
                    local wt = {}
                    for cos_type, cos_data in pairs(types) do
                        if type(cos_data) == "table" and cos_data.Name then
                            wt[tostring(cos_type)] = {
                                Name = tostring(cos_data.Name),
                                Type = tostring(cos_data.Type or cos_type),
                                Inverted = cos_data.Inverted and true or false,
                            }
                        end
                    end
                    if next(wt) then equipped_out[tostring(weapon)] = wt end
                end
            end
        end
        local owned = {}
        if type(sc.fake_owned) == "table" then
            for name, v in pairs(sc.fake_owned) do
                if v then owned[#owned + 1] = tostring(name) end
            end
        end
        t.skins = { equipped = equipped_out, fake_owned = owned, version = 2 }
    end)
    -- legacy emote_state
    pcall(function()
        if states.emote_state then
            t.emote = {
                selected = tostring(states.emote_state.selected or ""),
                equipped = tostring(states.emote_state.equipped or ""),
                speed = tonumber(states.emote_state.speed) or 1,
            }
        end
    end)
    -- modules Matcha
    pcall(function() t.esp = cfgSerializeTable(getgenv()._Vnix_esp) end)
    pcall(function() t.aim = cfgSerializeTable(getgenv()._Matcha_aim) end)
    pcall(function() t.silent = cfgSerializeTable(getgenv()._Matcha_silent) end)
    pcall(function() t.trigger = cfgSerializeTable(getgenv()._Matcha_trigger) end)
    pcall(function() t.char = cfgSerializeTable(getgenv()._Matcha_char) end)
    pcall(function() t.itemlib = cfgSerializeTable(getgenv()._Matcha_itemlib) end)
    pcall(function() t.manip = cfgSerializeTable(getgenv()._Matcha_manip) end)
    pcall(function() t.slide = cfgSerializeTable(getgenv()._Matcha_slide) end)
    pcall(function() t.hitbox = cfgSerializeTable(getgenv()._Matcha_hitbox) end)
    pcall(function() t.matcha_emote = cfgSerializeTable(getgenv()._Matcha_emote) end)
    pcall(function() t.anims = cfgSerializeTable(getgenv()._Vnix_anims) end)
    pcall(function()
        local r = getgenv()._Matcha_rage or getgenv()._Vnix_rage or getgenv().Rivals
        if type(r) == "table" then t.rage = cfgSerializeTable(r) end
    end)
    return t
end

function applyState(data)
    if type(data) ~= "table" then return end
    if type(data.toggles) == "table" then
        for flag, val in pairs(data.toggles) do
            local meta = VnixFlags[flag]
            if meta and meta.set then pcall(meta.set, val) end
        end
    end
    if type(data.options) == "table" then
        for flag, val in pairs(data.options) do
            local meta = VnixFlags[flag]
            if meta and meta.set then pcall(meta.set, val) end
        end
    end
    -- skins
    pcall(function()
        local sk = data.skins
        if type(sk) ~= "table" then return end
        local sc = states.skinchanger_state
        if type(sk.fake_owned) == "table" then
            sc.fake_owned = sc.fake_owned or {}
            for _, name in ipairs(sk.fake_owned) do sc.fake_owned[name] = true end
        end
        if type(sk.equipped) == "table" then
            for weapon, types in pairs(sk.equipped) do
                if type(types) == "table" then
                    for cos_type, cos_data in pairs(types) do
                        if type(cos_data) == "table" and cos_data.Name then
                            pcall(function()
                                HandleSkinchangerEquip({ weapon, cos_type, cos_data.Name, { IsInverted = cos_data.Inverted } })
                            end)
                        end
                    end
                end
            end
        end
        pcall(ReplicateSkinchanger)
    end)
    pcall(function()
        local em = data.emote
        if type(em) ~= "table" then return end
        states.emote_state = states.emote_state or {}
        if em.selected then states.emote_state.selected = em.selected end
        if em.speed then states.emote_state.speed = em.speed end
        if em.equipped and em.equipped ~= "" then
            task.defer(function() pcall(EquipEmote, em.equipped) end)
        end
    end)

    -- ESP
    pcall(function()
        local e = getgenv()._Vnix_esp
        if type(e) ~= "table" or type(data.esp) ~= "table" then return end
        cfgApplyModule(e, data.esp)
        if e.enabled and e._start then e._start()
        elseif (not e.enabled) and e._stop then e._stop() end
    end)

    -- AIMBOT
    pcall(function()
        local a = getgenv()._Matcha_aim
        if type(a) ~= "table" or type(data.aim) ~= "table" then return end
        local was = a.enabled
        cfgApplyModule(a, data.aim)
        -- no auto-enable hold state
        a.toggled = false
        if a.enabled then
            if a._start then a._start() end
        else
            if a._stop then a._stop() end
        end
    end)

    -- SILENT
    pcall(function()
        local s = getgenv()._Matcha_silent
        if type(s) ~= "table" or type(data.silent) ~= "table" then return end
        cfgApplyModule(s, data.silent)
        if s.enabled then
            if s._start then s._start() end
        else
            if s._stop then s._stop() end
        end
    end)

    -- TRIGGER
    pcall(function()
        local tbot = getgenv()._Matcha_trigger
        if type(tbot) ~= "table" or type(data.trigger) ~= "table" then return end
        cfgApplyModule(tbot, data.trigger)
        if tbot.enabled then
            if tbot._start then tbot._start() end
        else
            if tbot._stop then tbot._stop() end
        end
    end)

    -- CHARACTER (speed etc — no auto-enable movement cheats on load unless saved on)
    pcall(function()
        local ch = getgenv()._Matcha_char
        if type(ch) ~= "table" or type(data.char) ~= "table" then return end
        cfgApplyModule(ch, data.char)
        if ch._start and (ch.speed_enabled or ch.fly or ch.noclip or ch.inf_jump or ch.target_hover) then
            ch._start()
        end
    end)

    -- ITEM LIB (fast fire / melee)
    pcall(function()
        local il = getgenv()._Matcha_itemlib
        if type(il) ~= "table" or type(data.itemlib) ~= "table" then return end
        cfgApplyModule(il, data.itemlib)
        if il.rapid_fire and il._startGuns then il._startGuns() end
        if il.fast_melee and il._startMelee then il._startMelee() end
        if (not il.rapid_fire) and il._stopGuns then il._stopGuns() end
        if (not il.fast_melee) and il._stopMelee then il._stopMelee() end
    end)

    -- HITBOX
    pcall(function()
        local hb = getgenv()._Matcha_hitbox
        if type(hb) ~= "table" or type(data.hitbox) ~= "table" then return end
        cfgApplyModule(hb, data.hitbox)
        if hb.enabled and hb._start then hb._start()
        elseif (not hb.enabled) and hb._stop then hb._stop() end
    end)

    -- MATCHA EMOTE
    pcall(function()
        local em = getgenv()._Matcha_emote
        if type(em) ~= "table" or type(data.matcha_emote) ~= "table" then return end
        cfgApplyModule(em, data.matcha_emote)
        if em.enabled and em._play then
            task.defer(function() pcall(em._play, em.id or em.name, em.speed) end)
        end
    end)

    -- ANIM PACKS
    pcall(function()
        local an = getgenv()._Vnix_anims
        if type(an) ~= "table" or type(data.anims) ~= "table" then return end
        cfgApplyModule(an, data.anims)
        if an.enabled and an.selected and an._play then
            task.defer(function() pcall(an._play, an.selected) end)
        end
    end)

    
    pcall(function()
        local m = getgenv()._Matcha_manip
        if type(m) ~= "table" or type(data.manip) ~= "table" then return end
        cfgApplyModule(m, data.manip)
        if m.enabled and m._start then m._start()
        elseif (not m.enabled) and m._stop then m._stop() end
    end)
    pcall(function()
        local s = getgenv()._Matcha_slide
        if type(s) ~= "table" or type(data.slide) ~= "table" then return end
        cfgApplyModule(s, data.slide)
        if s.enabled and s._start then s._start()
        elseif (not s.enabled) and s._stop then s._stop() end
    end)
    Notify("Config", "Cargada · aim/silent/esp/char")
end
function cfgSave(name, overwrite)
    name = cfgSanitize(name)
    cfgEnsure()
    local path = CFG_FOLDER .. "/" .. name .. ".json"
    if isfile and isfile(path) and not overwrite then
        Notify("Config", "Ya existe — usa Overwrite")
        return false
    end
    local state = collectState()
    local ok, enc = pcall(function() return HttpService:JSONEncode(state) end)
    if not ok or not enc then Notify("Config", "Error encode"); return false end
    local wok = pcall(function() writefile(path, enc) end)
    if wok then
        local hasAim = state.aim and state.aim.enabled
        local hasSilent = state.silent and state.silent.enabled
        Notify("Config", "Guardada: " .. name .. (hasAim and " ·aim" or "") .. (hasSilent and " ·silent" or ""))
        print("[Matcha Config] saved", name, "bytes=", #enc, "aim=", tostring(state.aim ~= nil), "silent=", tostring(state.silent ~= nil))
    else
        Notify("Config", "Error write (executor writefile?)")
    end
    return wok
end
function cfgLoad(name)
    name = cfgSanitize(name)
    cfgEnsure()
    local path = CFG_FOLDER .. "/" .. name .. ".json"
    if not (isfile and isfile(path)) then
        Notify("Config", "No existe: " .. name)
        return false
    end
    local okRead, raw = pcall(function() return readfile(path) end)
    if not okRead or not raw or raw == "" then
        Notify("Config", "No se pudo leer")
        return false
    end
    local ok, data = pcall(function() return HttpService:JSONDecode(raw) end)
    if not ok or type(data) ~= "table" then
        Notify("Config", "JSON inválido")
        return false
    end
    -- delay apply por si módulos aún no existen al boot
    local function doApply()
        applyState(data)
    end
    if getgenv()._Matcha_aim or getgenv()._Vnix_esp then
        doApply()
    else
        task.defer(function()
            task.wait(0.3)
            doApply()
        end)
    end
    return true
end



-- Registrar flags para config
function regToggle(flag, getFn, setFn)
    VnixFlags[flag] = { get = getFn, set = setFn }
end
function regValue(flag, getFn, setFn)
    VnixFlags[flag] = { get = getFn, set = setFn }
end



do
pcall(function()
    local hit_sound_map = {
        Bameware = "rbxassetid://130791763",
        Bell = "rbxassetid://146680539",
        Bubble = "rbxassetid://146665237",
        Click = "rbxassetid://146633140",
        Pop = "rbxassetid://146681941",
        Rust = "rbxassetid://146692956",
        Fart = "rbxassetid://130791763",
        Big = "rbxassetid://130792279",
        Vine = "rbxassetid://146606035",
        Bruh = "rbxassetid://146657737",
        Skeet = "rbxassetid://130791733",
        Neverlose = "rbxassetid://146633485",
        Fatality = "rbxassetid://146665147",
        Bonk = "rbxassetid://146661497",
        Minecraft = "rbxassetid://146649495",
    }

    local function ResolveSoundId(sound_value, asset_id)
        if not sound_value or sound_value == "None" or sound_value == "" then return nil end
        if sound_value == "Use Asset Id" and asset_id and asset_id ~= "" then
            return "rbxassetid://" .. tostring(asset_id)
        end
        return hit_sound_map[sound_value]
    end

    task.spawn(function()
        task.wait(3)
        pcall(function()
            local vm_path = LocalPlayer.PlayerScripts.Modules.ClientReplicatedClasses.ClientFighter.ClientItem
            local vm_module = vm_path:WaitForChild("ClientViewModel", 5)
            if not vm_module then return end
            local ClientViewModel = require(vm_module)

            local orig_playhitmarker = ClientViewModel.PlayHitmarkerSound
            ClientViewModel.PlayHitmarkerSound = function(self, is_crit, distance)
                local ctx_player = self.ClientItem and self.ClientItem.ClientFighter and self.ClientItem.ClientFighter.Player
                if ctx_player == LocalPlayer then
                    local s = states.hitsound_state
                    if s and s.enabled then
                        local id = ResolveSoundId(s.sound_value, s.sound_asset_id)
                        if id then
                            local vol = (s.sound_volume or 50) / 100
                            local pitch = (s.sound_pitch or 10) / 10
                            while #self._last_hitmarker_sounds > 10 do
                                table.remove(self._last_hitmarker_sounds, 1):Destroy()
                            end
                            if is_crit then
                                self:_CreateHitmarkerSound(id, (3 / distance), pitch, vm_module, true, 1)
                                self:_CreateHitmarkerSound(id, (2 / distance), pitch, vm_module, true, 1)
                            else
                                self:_CreateHitmarkerSound(id, (1.5 / distance), pitch, vm_module, true, 1)
                            end
                            return
                        end
                    end
                end
                return orig_playhitmarker(self, is_crit, distance)
            end
        end)
    end)
end)

-- Ratio: solo se activa con el toggle (BindToRenderStep), sin loop pesado
task.spawn(function()
    while task.wait(1) do
        pcall(updateRatio)
    end
end)


-- ==================== ESP (off por defecto, solo auto si config) ====================
local esp_state = {
    enabled = false,
    team_check = false,
    team_colors = false,
    max_distance = 2000,
    box = true,
    box_style = "Full", -- Full | Corner
    box_fill = false,
    fill_opacity = 0.15,
    box3d = false,
    healthbar = true,
    health = true,
    name = true,
    info = true, -- distance + weapon
    skeleton = false,
    tracers = false,
    tracer_origin = "Bottom",
    headdot = false,
    arrows = false,
    arrow_radius = 200,
    arrow_size = 14,
    chams = false,
    chams_walls = true,
    chams_fill_t = 0.5,
    vis_check = false,
    text_size = 13,
    thickness = 1,
    -- colors: rosa Matcha por defecto (healthbar siempre verde)
    box_color = Color3.fromRGB(255, 105, 180),
    name_color = Color3.fromRGB(255, 105, 180),
    info_color = Color3.fromRGB(255, 105, 180),
    skeleton_color = Color3.fromRGB(255, 105, 180),
    tracer_color = Color3.fromRGB(255, 105, 180),
    headdot_color = Color3.fromRGB(255, 105, 180),
    arrow_color = Color3.fromRGB(255, 105, 180),
    chams_fill = Color3.fromRGB(255, 105, 180),
    chams_outline = Color3.fromRGB(255, 105, 180),
    vis_color = Color3.fromRGB(255, 105, 180),
    hid_color = Color3.fromRGB(255, 80, 140),
    healthbar_color = Color3.fromRGB(0, 255, 80), -- siempre verde
    -- legacy aliases for UI
    distance = true,
    corners = false,
}
getgenv()._Vnix_esp = esp_state

local ESP_White = Color3.fromRGB(255, 255, 255)
local ESP_Black = Color3.new(0, 0, 0)
local ESP_GreenHP = Color3.fromRGB(0, 255, 80) -- healthbar fijo verde
local ESP_RedHP = Color3.fromRGB(255, 40, 40)
local ESP_Font = 2
pcall(function()
    if Drawing and Drawing.Fonts and Drawing.Fonts.Plex then
        ESP_Font = Drawing.Fonts.Plex
    end
end)

local ESP_Objects = {} -- [Player] = data
local ESP_Connections = {}
local ESP_Unloaded = false
local ESP_Holder = nil
local ESP_Conn = nil

local SkeletonR15 = {
    { "Head", "UpperTorso" }, { "UpperTorso", "LowerTorso" },
    { "UpperTorso", "LeftUpperArm" }, { "LeftUpperArm", "LeftLowerArm" }, { "LeftLowerArm", "LeftHand" },
    { "UpperTorso", "RightUpperArm" }, { "RightUpperArm", "RightLowerArm" }, { "RightLowerArm", "RightHand" },
    { "LowerTorso", "LeftUpperLeg" }, { "LeftUpperLeg", "LeftLowerLeg" }, { "LeftLowerLeg", "LeftFoot" },
    { "LowerTorso", "RightUpperLeg" }, { "RightUpperLeg", "RightLowerLeg" }, { "RightLowerLeg", "RightFoot" },
}
local SkeletonR6 = {
    { "Head", "Torso" }, { "Torso", "Left Arm" }, { "Torso", "Right Arm" },
    { "Torso", "Left Leg" }, { "Torso", "Right Leg" },
}
local BoxEdges = {
    { 1, 2 }, { 2, 3 }, { 3, 4 }, { 4, 1 },
    { 5, 6 }, { 6, 7 }, { 7, 8 }, { 8, 5 },
    { 1, 5 }, { 2, 6 }, { 3, 7 }, { 4, 8 },
}

local function ESP_New(Type, Props)
    if not Drawing or not Drawing.new then return nil end
    local D = Drawing.new(Type)
    for K, V in pairs(Props) do D[K] = V end
    D.Visible = false
    return D
end

local function ESP_W2S(Position)
    local Cam = workspace.CurrentCamera
    if not Cam then return Vector2.zero, false end
    local Point, OnScreen = Cam:WorldToViewportPoint(Position)
    return Vector2.new(Point.X, Point.Y), OnScreen and Point.Z > 0
end

-- Blox Strike / universal: characters a veces viven en workspace.Characters
local function ESP_GetCharacter(Player)
    if not Player then return nil end
    if Player:GetAttribute("IsSpectating") == true then return nil end
    if Player:GetAttribute("Dead") == true then return nil end

    local c = Player.Character
    if c and c:FindFirstChildOfClass("Humanoid") then
        if c:GetAttribute("Dead") == true then return nil end
        return c
    end

    -- Blox Strike
    local folder = workspace:FindFirstChild("Characters")
    if folder then
        local named = folder:FindFirstChild(Player.Name)
        if named and named:IsA("Model") then
            if named:GetAttribute("Dead") == true then return nil end
            if named:FindFirstChild("HumanoidRootPart") or named:FindFirstChildOfClass("Humanoid") then
                return named
            end
        end
        -- a veces el model esta por UserId o tiene attribute Owner
        for _, m in ipairs(folder:GetChildren()) do
            if m:IsA("Model") then
                local owner = m:GetAttribute("Owner") or m:GetAttribute("Player") or m:GetAttribute("UserId")
                if owner == Player.Name or owner == Player.UserId or tostring(owner) == tostring(Player.UserId) then
                    if m:GetAttribute("Dead") ~= true then return m end
                end
                if m:GetAttribute("CharacterType") == "PlayerCustomCharacter" then
                    local plrVal = m:FindFirstChild("Player") or m:FindFirstChild("Owner")
                    if plrVal and ((plrVal:IsA("ObjectValue") and plrVal.Value == Player) or (plrVal:IsA("StringValue") and plrVal.Value == Player.Name)) then
                        return m
                    end
                end
            end
        end
    end
    return c
end

local function ESP_GetHealth(Character, Humanoid)
    if not Character then return 0, 100 end
    local hp = Character:GetAttribute("Health")
    local maxhp = Character:GetAttribute("MaxHealth")
    if type(hp) == "number" then
        if type(maxhp) ~= "number" or maxhp <= 0 then maxhp = 100 end
        return hp, maxhp
    end
    if Humanoid then
        return Humanoid.Health, math.max(Humanoid.MaxHealth, 1)
    end
    return 0, 100
end

local function ESP_GetRoot(Character)
    if not Character then return nil end
    return Character:FindFirstChild("HumanoidRootPart")
        or Character:FindFirstChild("RootPart")
        or Character.PrimaryPart
        or Character:FindFirstChild("Head")
end

local function ESP_IsEnemy(Player)
    if not Player or Player == LocalPlayer then return false end
    if not esp_state.team_check then return true end
    if type(MatchaIsEnemy) == "function" then
        local ok, r = pcall(MatchaIsEnemy, Player, true, false)
        if ok then return r end
    end
    -- Blox Strike: Team = "CT" / "T" / similar
    local myT = LocalPlayer:GetAttribute("Team") or LocalPlayer:GetAttribute("TeamID") or LocalPlayer:GetAttribute("TeamName")
    local ot = Player:GetAttribute("Team") or Player:GetAttribute("TeamID") or Player:GetAttribute("TeamName")
    if myT ~= nil and ot ~= nil then return tostring(myT) ~= tostring(ot) end
    if LocalPlayer.Team and Player.Team then return LocalPlayer.Team ~= Player.Team end
    -- team string values
    local myTeam = LocalPlayer:FindFirstChild("Team") or LocalPlayer:FindFirstChild("Status")
    local otTeam = Player:FindFirstChild("Team") or Player:FindFirstChild("Status")
    if myTeam and otTeam and myTeam:IsA("StringValue") and otTeam:IsA("StringValue") then
        if myTeam.Value ~= "" and otTeam.Value ~= "" then
            return myTeam.Value ~= otTeam.Value
        end
    end
    return true
end

local function ESP_Resolve(Base, Player, Visible)
    if esp_state.team_colors and Player.Team then
        return Player.TeamColor.Color
    end
    if esp_state.vis_check then
        return Visible and esp_state.vis_color or esp_state.hid_color
    end
    return Base
end

local ESP_RayParams = RaycastParams.new()
ESP_RayParams.FilterType = Enum.RaycastFilterType.Exclude

local function ESP_IsVisible(Character)
    local Target = Character:FindFirstChild("Head") or Character:FindFirstChild("HumanoidRootPart")
    if not Target then return false end
    local Cam = workspace.CurrentCamera
    ESP_RayParams.FilterDescendantsInstances = { LocalPlayer.Character, Character, Cam }
    local Origin = Cam.CFrame.Position
    return Workspace:Raycast(Origin, Target.Position - Origin, ESP_RayParams) == nil
end

local function ESP_GetHolder()
    if ESP_Holder and ESP_Holder.Parent then return ESP_Holder end
    local parent = (gethui and gethui()) or CoreGui
    pcall(function()
        local old = parent:FindFirstChild("MatchaESP_Holder")
        if old then old:Destroy() end
    end)
    ESP_Holder = Instance.new("Folder")
    ESP_Holder.Name = "MatchaESP_Holder"
    ESP_Holder.Parent = parent
    return ESP_Holder
end

local function ESP_AddPlayer(Player)
    if Player == LocalPlayer or ESP_Objects[Player] then return end
    if not Drawing or not Drawing.new then return end

    local D = { All = {}, Skel = {}, Corner = {}, CornerOut = {}, Box3D = {} }
    local function Reg(X)
        if X then table.insert(D.All, X) end
        return X
    end

    D.Fill = Reg(ESP_New("Square", { Filled = true, ZIndex = 1 }))
    D.BoxOut = Reg(ESP_New("Square", { Filled = false, Thickness = 3, Color = ESP_Black, ZIndex = 2 }))
    D.Box = Reg(ESP_New("Square", { Filled = false, Thickness = 1, ZIndex = 3 }))
    for i = 1, 8 do
        D.CornerOut[i] = Reg(ESP_New("Line", { Thickness = 3, Color = ESP_Black, ZIndex = 2 }))
        D.Corner[i] = Reg(ESP_New("Line", { Thickness = 1, ZIndex = 3 }))
    end
    for i = 1, 12 do
        D.Box3D[i] = Reg(ESP_New("Line", { Thickness = 1, ZIndex = 2 }))
    end
    D.BarOut = Reg(ESP_New("Square", { Filled = true, Color = ESP_Black, ZIndex = 2 }))
    D.Bar = Reg(ESP_New("Square", { Filled = true, ZIndex = 3 }))
    D.HP = Reg(ESP_New("Text", { Size = 11, Font = ESP_Font, Center = true, Outline = true, Color = ESP_White, ZIndex = 4 }))
    D.Name = Reg(ESP_New("Text", { Size = 13, Font = ESP_Font, Center = true, Outline = true, ZIndex = 4 }))
    D.Info = Reg(ESP_New("Text", { Size = 11, Font = ESP_Font, Center = true, Outline = true, ZIndex = 4 }))
    D.Tracer = Reg(ESP_New("Line", { Thickness = 1, ZIndex = 2 }))
    for i = 1, 14 do
        D.Skel[i] = Reg(ESP_New("Line", { Thickness = 1, ZIndex = 2 }))
    end
    D.HeadOut = Reg(ESP_New("Circle", { NumSides = 24, Filled = false, Thickness = 3, Color = ESP_Black, ZIndex = 2 }))
    D.Head = Reg(ESP_New("Circle", { NumSides = 24, Filled = false, Thickness = 1, ZIndex = 3 }))
    D.ArrowOut = Reg(ESP_New("Triangle", { Filled = false, Thickness = 2, Color = ESP_Black, ZIndex = 2 }))
    D.Arrow = Reg(ESP_New("Triangle", { Filled = true, ZIndex = 3 }))

    local H = Instance.new("Highlight")
    H.Enabled = false
    H.Parent = ESP_GetHolder()
    D.Highlight = H

    ESP_Objects[Player] = D
end

local function ESP_RemovePlayer(Player)
    local D = ESP_Objects[Player]
    if not D then return end
    for _, X in pairs(D.All) do
        pcall(function() if X and X.Remove then X:Remove() end end)
    end
    pcall(function() if D.Highlight then D.Highlight:Destroy() end end)
    ESP_Objects[Player] = nil
end

local function ESP_DrawCorners(D, X, Y, W, H, Color)
    local L = math.floor(math.min(W, H) * 0.25)
    local Lines = {
        { Vector2.new(X, Y), Vector2.new(X + L, Y) }, { Vector2.new(X, Y), Vector2.new(X, Y + L) },
        { Vector2.new(X + W, Y), Vector2.new(X + W - L, Y) }, { Vector2.new(X + W, Y), Vector2.new(X + W, Y + L) },
        { Vector2.new(X, Y + H), Vector2.new(X + L, Y + H) }, { Vector2.new(X, Y + H), Vector2.new(X, Y + H - L) },
        { Vector2.new(X + W, Y + H), Vector2.new(X + W - L, Y + H) }, { Vector2.new(X + W, Y + H), Vector2.new(X + W, Y + H - L) },
    }
    for i, Pair in ipairs(Lines) do
        if D.CornerOut[i] then
            D.CornerOut[i].From, D.CornerOut[i].To, D.CornerOut[i].Visible = Pair[1], Pair[2], true
        end
        if D.Corner[i] then
            D.Corner[i].From, D.Corner[i].To, D.Corner[i].Visible = Pair[1], Pair[2], true
            D.Corner[i].Color = Color
        end
    end
end

local function ESP_DrawArrow(D, Root, Viewport, Color)
    local Cam = workspace.CurrentCamera
    local Rel = Cam.CFrame:PointToObjectSpace(Root.Position)
    local Dir = Vector2.new(Rel.X, Rel.Z)
    if Dir.Magnitude < 0.001 then return end
    Dir = Dir.Unit
    local Pos = Viewport / 2 + Dir * (esp_state.arrow_radius or 200)
    local Perp = Vector2.new(-Dir.Y, Dir.X)
    local Size = esp_state.arrow_size or 14
    local A = Pos + Dir * Size
    local B = Pos - Dir * Size * 0.5 + Perp * Size * 0.6
    local C = Pos - Dir * Size * 0.5 - Perp * Size * 0.6
    if D.Arrow then
        D.Arrow.PointA, D.Arrow.PointB, D.Arrow.PointC = A, B, C
        D.Arrow.Color, D.Arrow.Visible = Color, true
    end
    if D.ArrowOut then
        D.ArrowOut.PointA, D.ArrowOut.PointB, D.ArrowOut.PointC = A, B, C
        D.ArrowOut.Visible = true
    end
end

local function ESP_UpdateChams(D, Character, Player, Visible)
    local H = D.Highlight
    if not H then return end
    if not esp_state.chams then
        H.Enabled = false
        return
    end
    if H.Adornee ~= Character then H.Adornee = Character end
    H.FillColor = ESP_Resolve(esp_state.chams_fill, Player, Visible)
    H.OutlineColor = ESP_Resolve(esp_state.chams_outline, Player, Visible)
    H.FillTransparency = esp_state.chams_fill_t or 0.5
    H.OutlineTransparency = 0
    H.DepthMode = esp_state.chams_walls and Enum.HighlightDepthMode.AlwaysOnTop or Enum.HighlightDepthMode.Occluded
    H.Enabled = true
end

local function ESP_Update()
    local E = esp_state
    if not E.enabled then
        for _, D in pairs(ESP_Objects) do
            for _, X in pairs(D.All) do if X then X.Visible = false end end
            if D.Highlight then D.Highlight.Enabled = false end
        end
        return
    end

    local Cam = workspace.CurrentCamera
    if not Cam then return end
    local MyChar = ESP_GetCharacter(LocalPlayer) or LocalPlayer.Character
    local MyRoot = MyChar and (MyChar:FindFirstChild("HumanoidRootPart") or MyChar.PrimaryPart)
    local Viewport = Cam.ViewportSize

    for Player, D in pairs(ESP_Objects) do
        for _, X in pairs(D.All) do if X then X.Visible = false end end
        if D.Highlight then D.Highlight.Enabled = false end

        local Character = ESP_GetCharacter(Player)
        local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
        local Root = ESP_GetRoot(Character)
        local hp, maxhp = ESP_GetHealth(Character, Humanoid)

        if not (Character and Root and hp > 0 and ESP_IsEnemy(Player)) then
            D.HPSmooth = nil
            continue
        end
        -- si no hay humanoid, inventamos uno minimo para skeleton checks
        if not Humanoid then
            Humanoid = { Health = hp, MaxHealth = maxhp, RigType = Enum.HumanoidRigType.R15 }
        end

        local Distance = MyRoot and (MyRoot.Position - Root.Position).Magnitude or 0
        if Distance > (E.max_distance or 2000) then continue end

        local Visible = E.vis_check and ESP_IsVisible(Character) or false
        ESP_UpdateChams(D, Character, Player, Visible)

        local _, RootV = ESP_W2S(Root.Position)
        local TopP, TopV = ESP_W2S(Root.Position + Vector3.new(0, 2.7, 0))
        local BotP, BotV = ESP_W2S(Root.Position - Vector3.new(0, 3.1, 0))

        if not (TopV and BotV) then
            if E.arrows and not RootV then
                ESP_DrawArrow(D, Root, Viewport, ESP_Resolve(E.arrow_color, Player, Visible))
            end
            continue
        end

        local Height = math.max(BotP.Y - TopP.Y, 4)
        local Width = Height * 0.6
        local CenterX = math.floor((TopP.X + BotP.X) / 2)
        local X = math.floor(CenterX - Width / 2)
        local Y = math.floor(TopP.Y)
        Height, Width = math.floor(Height), math.floor(Width)

        if E.box_fill and D.Fill then
            D.Fill.Position, D.Fill.Size = Vector2.new(X, Y), Vector2.new(Width, Height)
            D.Fill.Color = ESP_Resolve(E.box_color, Player, Visible)
            D.Fill.Transparency = E.fill_opacity or 0.15
            D.Fill.Visible = true
        end

        if E.box then
            local Color = ESP_Resolve(E.box_color, Player, Visible)
            local style = E.box_style or "Full"
            if style == "Corner" or E.corners then
                ESP_DrawCorners(D, X, Y, Width, Height, Color)
            else
                if D.BoxOut then
                    D.BoxOut.Position, D.BoxOut.Size, D.BoxOut.Visible = Vector2.new(X, Y), Vector2.new(Width, Height), true
                end
                if D.Box then
                    D.Box.Position, D.Box.Size, D.Box.Visible = Vector2.new(X, Y), Vector2.new(Width, Height), true
                    D.Box.Color = Color
                    D.Box.Thickness = E.thickness or 1
                end
            end
        end

        if E.box3d then
            local _, Yaw = Root.CFrame:ToEulerAnglesYXZ()
            local Base = CFrame.new(Root.Position + Vector3.new(0, -0.2, 0)) * CFrame.Angles(0, Yaw, 0)
            local Hx, Hy, Hz = 1.9, 2.9, 1.0
            local Corners3 = {
                Base * Vector3.new(-Hx, -Hy, -Hz), Base * Vector3.new(Hx, -Hy, -Hz),
                Base * Vector3.new(Hx, -Hy, Hz), Base * Vector3.new(-Hx, -Hy, Hz),
                Base * Vector3.new(-Hx, Hy, -Hz), Base * Vector3.new(Hx, Hy, -Hz),
                Base * Vector3.new(Hx, Hy, Hz), Base * Vector3.new(-Hx, Hy, Hz),
            }
            local Points, AllVisible = {}, true
            for i, Corner in ipairs(Corners3) do
                local P, V = ESP_W2S(Corner)
                Points[i] = P
                if not V then AllVisible = false end
            end
            if AllVisible then
                local Color = ESP_Resolve(E.box_color, Player, Visible)
                for i, Edge in ipairs(BoxEdges) do
                    local L = D.Box3D[i]
                    if L then
                        L.From, L.To, L.Color, L.Thickness, L.Visible = Points[Edge[1]], Points[Edge[2]], Color, E.thickness or 1, true
                    end
                end
            end
        end

        local Health = math.clamp(hp / math.max(maxhp, 1), 0, 1)
        D.HPSmooth = D.HPSmooth and (D.HPSmooth + (Health - D.HPSmooth) * 0.15) or Health

        if E.healthbar and D.Bar then
            local BarX = X - 6
            local FillH = math.max(math.floor(Height * D.HPSmooth), 1)
            if D.BarOut then
                D.BarOut.Position, D.BarOut.Size, D.BarOut.Visible = Vector2.new(BarX - 1, Y - 1), Vector2.new(4, Height + 2), true
            end
            D.Bar.Position, D.Bar.Size = Vector2.new(BarX, Y + Height - FillH), Vector2.new(2, FillH)
            D.Bar.Color = Color3.fromRGB(0, 255, 80) -- healthbar siempre verde
            D.Bar.Visible = true
            if E.health and Health < 0.99 and D.HP then
                D.HP.Size = math.max((E.text_size or 13) - 2, 8)
                D.HP.Text = tostring(math.floor(hp))
                local tw = 0
                pcall(function() tw = D.HP.TextBounds.X end)
                D.HP.Position = Vector2.new(BarX - 2 - tw / 2, Y + Height - FillH - 6)
                D.HP.Visible = true
            end
        elseif E.health and D.HP then
            D.HP.Size = math.max((E.text_size or 13) - 2, 8)
            D.HP.Text = string.format("%d HP", math.floor(Humanoid.Health))
            D.HP.Color = Color3.fromRGB(0, 255, 80)
            D.HP.Position = Vector2.new(CenterX, Y - (E.text_size or 13) - 18)
            D.HP.Visible = true
        end

        if E.name and D.Name then
            D.Name.Size = E.text_size or 13
            D.Name.Text = (Player.DisplayName ~= "" and Player.DisplayName) or Player.Name
            D.Name.Color = E.name_color or ESP_White
            D.Name.Position = Vector2.new(CenterX, Y - (E.text_size or 13) - 3)
            D.Name.Visible = true
        end

        if (E.info or E.distance) and D.Info then
            local Tool = Character:FindFirstChildOfClass("Tool")
            D.Info.Size = math.max((E.text_size or 13) - 2, 8)
            D.Info.Text = string.format("%dm%s", math.floor(Distance), Tool and (" | " .. Tool.Name) or "")
            D.Info.Color = E.info_color or Color3.fromRGB(200, 200, 200)
            D.Info.Position = Vector2.new(CenterX, Y + Height + 2)
            D.Info.Visible = true
        end

        if E.tracers and D.Tracer then
            local From
            local origin = E.tracer_origin or "Bottom"
            if origin == "Center" then
                From = Viewport / 2
            elseif origin == "Top" then
                From = Vector2.new(Viewport.X / 2, 0)
            elseif origin == "Mouse" then
                From = UserInputService:GetMouseLocation()
            else
                From = Vector2.new(Viewport.X / 2, Viewport.Y)
            end
            D.Tracer.From = From
            D.Tracer.To = Vector2.new(CenterX, Y + Height)
            D.Tracer.Color = ESP_Resolve(E.tracer_color, Player, Visible)
            D.Tracer.Thickness = E.thickness or 1
            D.Tracer.Visible = true
        end

        if E.headdot then
            local HeadPart = Character:FindFirstChild("Head")
            if HeadPart then
                local P, V = ESP_W2S(HeadPart.Position)
                if V then
                    local Radius = math.max(Height * 0.075, 2)
                    if D.HeadOut then D.HeadOut.Position, D.HeadOut.Radius, D.HeadOut.Visible = P, Radius, true end
                    if D.Head then
                        D.Head.Position, D.Head.Radius, D.Head.Visible = P, Radius, true
                        D.Head.Color = ESP_Resolve(E.headdot_color, Player, Visible)
                    end
                end
            end
        end

        if E.skeleton then
            local Color = ESP_Resolve(E.skeleton_color or Color3.fromRGB(255, 105, 180), Player, Visible)
            local rig = Humanoid.RigType or Enum.HumanoidRigType.R15
            local Joints = (rig == Enum.HumanoidRigType.R6) and SkeletonR6 or SkeletonR15
            for i, Pair in ipairs(Joints) do
                local A, B = Character:FindFirstChild(Pair[1]), Character:FindFirstChild(Pair[2])
                if A and B then
                    local PA, VA = ESP_W2S(A.Position)
                    local PB, VB = ESP_W2S(B.Position)
                    if VA and VB and D.Skel[i] then
                        local L = D.Skel[i]
                        L.From, L.To, L.Color, L.Thickness, L.Visible = PA, PB, Color, E.thickness or 1, true
                    end
                end
            end
        end
    end
end

function startESP()
    if ESP_Conn then return end
    ESP_Unloaded = false
    ESP_GetHolder()
    for _, Player in ipairs(Players:GetPlayers()) do
        ESP_AddPlayer(Player)
    end
    table.insert(ESP_Connections, Players.PlayerAdded:Connect(ESP_AddPlayer))
    table.insert(ESP_Connections, Players.PlayerRemoving:Connect(ESP_RemovePlayer))
    ESP_Conn = RunService.RenderStepped:Connect(function()
        if ESP_Unloaded then return end
        ESP_Update()
    end)
    print("[Matcha ESP] START · source Drawing ESP")
end

function stopESP()
    ESP_Unloaded = true
    if ESP_Conn then pcall(function() ESP_Conn:Disconnect() end); ESP_Conn = nil end
    for _, C in ipairs(ESP_Connections) do pcall(function() C:Disconnect() end) end
    table.clear(ESP_Connections)
    for Player in pairs(ESP_Objects) do ESP_RemovePlayer(Player) end
    if ESP_Holder then pcall(function() ESP_Holder:Destroy() end); ESP_Holder = nil end
    print("[Matcha ESP] STOPPED")
end

esp_state._start = startESP
esp_state._stop = stopESP



Players.PlayerRemoving:Connect(function(plr)
    destroyDraw(plr)
end)

-- ESP UI handled by custom Vnix UI
-- NO auto-start



end

-- UI en scope aparte (evita "Out of local registers")


-- ==================== RAGE (from shako.win — solo rage) ====================
-- ==================== RAGEBOT RIVALS SOURCE (FighterController + UseItem) ====================
local rage_cfg = {
    enabled = false,
    team_check = true,
    force_ffa = false,
    fire_rate = 0.0005,
    weapon_slot = "Primary", -- Primary / Secondary / Melee
    max_dist = 500,
    desync = true,
}
getgenv()._Vnix_rage = rage_cfg
getgenv().Config = getgenv().Config or { Enabled = false, FireRate = 0.0005, WeaponSlot = "Primary" }

local rage_slots = { Primary = 1, Secondary = 2, Melee = 3 }
local rage_lastFire = 0
local rage_deflecting = {}
local rage_util = nil
local rage_enum = nil
local rage_FighterController = nil
local rage_SpectateController = nil
local rage_ready = false

local function rageLoadModules()
    if rage_ready then return true end
    local ok1, u = pcall(function() return require(ReplicatedStorage.Modules.Utility) end)
    local ok2, e = pcall(function() return require(ReplicatedStorage.Modules.EnumLibrary) end)
    local ok3, fc = pcall(function()
        return require(LocalPlayer.PlayerScripts.Controllers.FighterController)
    end)
    local ok4, sc = pcall(function()
        return require(LocalPlayer.PlayerScripts.Controllers:WaitForChild("SpectateController"))
    end)
    if ok1 then rage_util = u end
    if ok2 then rage_enum = e end
    if ok3 then rage_FighterController = fc end
    if ok4 then rage_SpectateController = sc end
    if rage_util and rage_enum and rage_FighterController then
        rage_ready = true
        print("[Rage Rivals] modules OK (Utility + Enum + FighterController)")
        return true
    end
    return false
end

Players.PlayerRemoving:Connect(function(player)
    rage_deflecting[player] = nil
end)

local function rageUpdateDeflection()
    if not rage_FighterController or not rage_FighterController.Objects then return end
    for _, fighterObj in pairs(rage_FighterController.Objects) do
        local player = fighterObj.Player
        if not player then continue end
        local alive = true
        pcall(function()
            if not fighterObj.Entity or not fighterObj.Entity:IsAlive() or fighterObj:Get("IsSpectating") then
                alive = false
            end
        end)
        if not alive then
            rage_deflecting[player] = false
            continue
        end
        local equipped = fighterObj.EquippedItem
        local isKatana = false
        pcall(function()
            isKatana = equipped and equipped.ViewModel and equipped.ViewModel.Name == "Katana"
        end)
        local isDeflecting = false
        if isKatana then
            pcall(function()
                isDeflecting = (equipped._attack_cooldown and equipped._attack_cooldown > tick()) or false
            end)
        end
        rage_deflecting[player] = isDeflecting
    end
end

local function rageIsEnemy(player)
    if not player or player == LocalPlayer then return false end
    if rage_cfg.force_ffa or not rage_cfg.team_check then return true end

    -- SpectateController duel teams
    if rage_SpectateController then
        local ok, result = pcall(function()
            local duel = rage_SpectateController.CurrentDuelSubject
            local localDueler = duel and duel:GetDueler(LocalPlayer)
            local localTeam = localDueler and localDueler:Get("TeamID") or nil
            if localTeam and duel and duel.Duelers then
                for _, dueler in pairs(duel.Duelers) do
                    if dueler.Player == player then
                        local team = dueler:Get("TeamID")
                        return team ~= localTeam
                    end
                end
            end
            return nil
        end)
        if ok and result ~= nil then return result end
    end

    local pTeam = player:GetAttribute("TeamID") or player:GetAttribute("Team")
    local lTeam = LocalPlayer:GetAttribute("TeamID") or LocalPlayer:GetAttribute("Team")
    if pTeam ~= nil and lTeam ~= nil then
        return pTeam ~= lTeam
    end
    if LocalPlayer.Team and player.Team then
        return LocalPlayer.Team ~= player.Team
    end
    return true
end

local function rageGetClosestTarget()
    local char = LocalPlayer.Character
    if not char then return nil, nil, nil end
    local myRoot = char:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil, nil, nil end

    local closestPlayer, closestRoot, closestHead = nil, nil, nil
    local closestDist = rage_cfg.max_dist or 500

    for _, player in ipairs(Players:GetPlayers()) do
        if not rageIsEnemy(player) then continue end
        local pChar = player.Character
        if not pChar then continue end
        local pRoot = pChar:FindFirstChild("HumanoidRootPart")
        local pHead = pChar:FindFirstChild("Head")
        local pHum = pChar:FindFirstChildWhichIsA("Humanoid")
        if not (pRoot and pHead and pHum and pHum.Health > 0) then continue end
        local dist = (myRoot.Position - pRoot.Position).Magnitude
        if dist < closestDist then
            closestDist = dist
            closestPlayer = player
            closestRoot = pRoot
            closestHead = pHead
        end
    end
    return closestPlayer, closestRoot, closestHead
end

local function rageHasKnifeViewModel(targetPlayer)
    if not targetPlayer then return false end
    local viewModels = Workspace:FindFirstChild("ViewModels")
    if not viewModels then return false end
    local targetName = targetPlayer.Name
    for _, model in ipairs(viewModels:GetChildren()) do
        if model:IsA("Model")
            and string.find(model.Name, targetName, 1, true)
            and string.find(model.Name, "Knife", 1, true) then
            return true
        end
    end
    return false
end

local function rageEquipSlot()
    if not rage_FighterController then return end
    local slot = rage_slots[rage_cfg.weapon_slot] or 1
    pcall(function()
        local lf = rage_FighterController.LocalFighter
        if lf then lf:EquipItem(slot) end
    end)
end

-- main rage loop (source exacto adaptado)
RunService.Heartbeat:Connect(function()
    if not rage_cfg.enabled then return end
    if not rage_ready then
        rageLoadModules()
        return
    end

    getgenv().Config.Enabled = true
    getgenv().Config.FireRate = rage_cfg.fire_rate
    getgenv().Config.WeaponSlot = rage_cfg.weapon_slot

    rageUpdateDeflection()
    local targetPlayer, targetRoot, targetHead = rageGetClosestTarget()
    local desyncCF = nil

    if rage_cfg.desync and targetRoot and targetHead then
        local desyncPos
        if rageHasKnifeViewModel(targetPlayer) then
            desyncPos = (targetRoot.CFrame * CFrame.new(0, 6, 0)).Position
        else
            desyncPos = (targetRoot.CFrame * CFrame.new(0, 1, 2)).Position
        end
        desyncCF = CFrame.lookAt(desyncPos, targetHead.Position)
    end

    if desyncCF and LocalPlayer.Character then
        local myRoot = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if myRoot then
            local oldCF = myRoot.CFrame
            local oldVel = myRoot.AssemblyLinearVelocity
            pcall(function()
                myRoot.CFrame = desyncCF
            end)
            RunService:BindToRenderStep("__MatchaRageRestore", 101, function()
                if myRoot and myRoot.Parent then
                    pcall(function()
                        myRoot.CFrame = oldCF
                        myRoot.AssemblyLinearVelocity = oldVel
                    end)
                end
                pcall(function() RunService:UnbindFromRenderStep("__MatchaRageRestore") end)
            end)
        end
    end

    if not targetPlayer or not targetHead or not targetRoot then return end
    if rage_deflecting[targetPlayer] then return end
    if not LocalPlayer.Character or not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then return end
    if not rage_FighterController or not rage_FighterController.LocalFighter then return end

    local item = rage_FighterController.LocalFighter.EquippedItem
    if not item then
        rageEquipSlot()
        return
    end

    if tick() - rage_lastFire < (rage_cfg.fire_rate or 0.0005) then return end
    rage_lastFire = tick()

    if not rage_util or not rage_enum then return end

    local originPos = desyncCF and desyncCF.Position or targetRoot.Position
    local targetPos = targetHead.Position
    local aimCF = CFrame.lookAt(originPos, targetPos)
    local targetCF = targetHead.CFrame
    local randomOffset = Vector3.new(
        (math.random() - 0.5) * 0.1,
        (math.random() - 0.5) * 0.1,
        (math.random() - 0.5) * 0.1
    )
    local aimedPos = targetPos + randomOffset
    local objSpaceHeadOffset = targetHead.CFrame:ToObjectSpace(CFrame.new(aimedPos))

    local cameradata = {}
    cameradata[utf8.char(1)] = {
        [utf8.char(0)] = rage_util:EncodeCFrame(aimCF),
        [utf8.char(1)] = rage_util:EncodeCFrame(targetCF),
        [utf8.char(2)] = targetHead,
        [utf8.char(3)] = rage_util:EncodeCFrame(objSpaceHeadOffset)
    }

    pcall(function()
        ReplicatedStorage.Remotes.Replication.Fighter.UseItem:FireServer(
            item:Get("ObjectID"),
            rage_enum:ToEnum("StartShooting"),
            cameradata,
            nil
        )
    end)
end)

-- equip loop
task.spawn(function()
    while true do
        task.wait(1)
        if rage_cfg.enabled and rage_ready then
            rageEquipSlot()
        end
    end
end)

-- try load early
task.spawn(function()
    for _ = 1, 60 do
        if rageLoadModules() then break end
        task.wait(0.5)
    end
end)

local function rageStart()
    rage_cfg.enabled = true
    getgenv().Config.Enabled = true
    rageLoadModules()
    rageEquipSlot()
    print("[Rage Rivals] ENABLED · UseItem + desync")
end
local function rageStop()
    rage_cfg.enabled = false
    getgenv().Config.Enabled = false
    print("[Rage Rivals] DISABLED")
end
rage_cfg._start = rageStart
rage_cfg._stop = rageStop


-- ==================== MATCHA AIMBOT ====================
local aim_cfg = {
    enabled = false,
    team_check = true,
    force_ffa = false,
    wall_check = false, -- OFF: en 2v2 no suelta por paredes y no "salta"
    sticky = true,
    predict = true,
    predict_amount = 0.10,
    fov = 140,

    show_fov = true,
    fov_fill = false,
    fov_spin = false,
    fov_spin_speed = 2.5,
    fov_color = Color3.fromRGB(255, 105, 180),      -- outline
    fov_fill_color1 = Color3.fromRGB(255, 105, 180), -- fill color 1
    fov_fill_color2 = Color3.fromRGB(255, 255, 255), -- fill color 2
    fov_fill_transparency = 0.75,
    smooth = 0.28,
    max_step = 6,
    target_part = "Head",
    priority = "Crosshair",
    range = 800,
    hold_key = Enum.UserInputType.MouseButton2,
    toggle_mode = false,
    toggled = false,
    skip_ff = true,
    lock_grace = 0.35,      -- segundos que mantiene lock aunque parpadee validez
    switch_threshold = 40,  -- solo cambia si el nuevo está MUCHO más cerca del crosshair (px)
}

-- shako.win aimbot method: FOV px + wall + mousemoverel / CFrame
local function aimbotWallClear(fromPos, toPos, targetChar)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    local filter = {}
    if LocalPlayer.Character then table.insert(filter, LocalPlayer.Character) end
    if targetChar then table.insert(filter, targetChar) end
    local vm = workspace:FindFirstChild("ViewModels")
    if vm then table.insert(filter, vm) end
    params.FilterDescendantsInstances = filter
    params.IgnoreWater = true
    local delta = toPos - fromPos
    local dist = delta.Magnitude
    if dist < 0.5 then return true end
    local res = workspace:Raycast(fromPos, delta.Unit * dist, params)
    if not res then return true end
    if targetChar and res.Instance and res.Instance:IsDescendantOf(targetChar) then return true end
    return false
end

getgenv()._Matcha_aim = aim_cfg

local aim_conn = nil
local aim_fov_draw = nil
local aim_current = nil
local aim_holding = false
local Camera = workspace.CurrentCamera
local UIS = UserInputService

local function aimIsEnemy(plr)
    if not plr or plr == LocalPlayer then return false end
    -- mismo sistema que rage → funciona 1v1 2v2 3v3 4v4 5v5
    if type(MatchaIsEnemy) == "function" then
        local ok, res = pcall(MatchaIsEnemy, plr, aim_cfg.team_check, aim_cfg.force_ffa)
        if ok then return res end
    end
    if aim_cfg.force_ffa or not aim_cfg.team_check then return true end
    if LocalPlayer.Team and plr.Team then
        return LocalPlayer.Team ~= plr.Team
    end
    return true -- sin team → enemigo
end

local function aimVisible(from, to)
    if not aim_cfg.wall_check then return true end
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    local ignore = {LocalPlayer.Character}
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Character then table.insert(ignore, p.Character) end
    end
    params.FilterDescendantsInstances = ignore
    params.IgnoreWater = true
    local dir = to - from
    local result = Workspace:Raycast(from, dir, params)
    return result == nil
end

local function aimGetPart(char)
    if not char then return nil end
    local partName = aim_cfg.target_part or "Head"
    if partName == "Closest" then
        local cam = workspace.CurrentCamera
        if not cam then return char:FindFirstChild("Head") end
        local best, bestDist = nil, math.huge
        for _, n in ipairs({"Head", "HumanoidRootPart", "UpperTorso", "Torso"}) do
            local p = char:FindFirstChild(n)
            if p and p:IsA("BasePart") then
                local sp, on = cam:WorldToViewportPoint(p.Position)
                if on then
                    local d = (Vector2.new(sp.X, sp.Y) - cam.ViewportSize / 2).Magnitude
                    if d < bestDist then bestDist = d; best = p end
                end
            end
        end
        return best or char:FindFirstChild("Head")
    end
    return char:FindFirstChild(partName) or char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
end

local aim_lock_until = 0 -- grace: no soltar hasta este tick
local aim_lock_fail = 0

local function aimAlive(char)
    if not char then return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum and hum.Health > 0 then return true end
    local hp, dead = 100, false
    pcall(function()
        hp = char:GetAttribute("Health") or 100
        dead = char:GetAttribute("Dead") == true
    end)
    return (not dead) and hp > 0
end

local function aimGetTarget()
    local cam = workspace.CurrentCamera
    if not cam then return nil end
    local myChar = LocalPlayer.Character
    local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
    local origin = cam.CFrame.Position
    -- shako: FOV desde el mouse (crosshair), no centro fijo
    local center = UIS:GetMouseLocation()
    if not center or (center.X == 0 and center.Y == 0) then
        center = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)
    end
    local now = tick()
    local range = aim_cfg.range or 800
    local sticky = aim_cfg.sticky ~= false

    -- ==================== LOCK FUERTE (Matcha style) ====================
    -- Una vez agarrado un enemigo, NO cambia a otro hasta:
    --   morir / fuera de rango mucho tiempo / soltar MB2
    -- NO se suelta por wallcheck, team flicker, ni otro más cerca del crosshair
    if sticky and aim_current then
        local plr = aim_current
        local still_ok = false
        local part, pos = nil, nil

        if plr and plr.Parent and plr.Character then
            local char = plr.Character
            if aimAlive(char) then
                part = aimGetPart(char)
                if part then
                    pos = part.Position
                    if aim_cfg.predict then
                        local root = char:FindFirstChild("HumanoidRootPart")
                        if root then
                            pos = pos + root.AssemblyLinearVelocity * (aim_cfg.predict_amount or 0.10)
                        end
                    end
                    local dist3d = myHRP and (myHRP.Position - part.Position).Magnitude or 0
                    -- hysteresis de rango: permite 1.35x antes de soltar
                    if dist3d <= range * 1.35 then
                        still_ok = true
                    end
                end
            end
        end

        if still_ok and part and pos then
            aim_lock_fail = 0
            aim_lock_until = now + (aim_cfg.lock_grace or 0.35)
            return plr, part, pos
        else
            -- grace period: sigue apuntando al último punto válido un momento
            aim_lock_fail = aim_lock_fail + 1
            if now < aim_lock_until and aim_lock_fail < 12 then
                -- intenta devolver algo del character aunque falló un frame
                if plr and plr.Character then
                    local p = aimGetPart(plr.Character) or plr.Character:FindFirstChild("HumanoidRootPart")
                    if p then
                        return plr, p, p.Position
                    end
                end
            end
            -- lock perdido de verdad
            aim_current = nil
            aim_lock_fail = 0
        end
    end

    -- ==================== ADQUIRIR NUEVO TARGET ====================
    local best, bestPart, bestPos, bestScore = nil, nil, nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if not aimIsEnemy(plr) then continue end
        local char = plr.Character
        if not char or not aimAlive(char) then continue end
        local part = aimGetPart(char)
        if not part then continue end
        if aim_cfg.skip_ff and char:FindFirstChildOfClass("ForceField") then continue end

        local pos = part.Position
        if aim_cfg.predict then
            local root = char:FindFirstChild("HumanoidRootPart")
            if root then
                pos = pos + root.AssemblyLinearVelocity * (aim_cfg.predict_amount or 0.10)
            end
        end

        local dist3d = myHRP and (myHRP.Position - part.Position).Magnitude or 0
        if dist3d > range then continue end

        local sp, onScreen = cam:WorldToViewportPoint(pos)
        if not onScreen or sp.Z < 0 then continue end
        local dist2d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
        if dist2d > (aim_cfg.fov or 140) then continue end

        -- wall check SOLO al adquirir (no al mantener lock)
        if aim_cfg.wall_check and not aimVisible(origin, part.Position) then continue end

        local score = dist2d
        if aim_cfg.priority == "LowestHP" then
            local hum = char:FindFirstChildOfClass("Humanoid")
            score = hum and hum.Health or 100
        elseif aim_cfg.priority == "Closest" then
            score = dist3d
        end

        if score < bestScore then
            bestScore = score
            best = plr
            bestPart = part
            bestPos = pos
        end
    end

    if best then
        aim_current = best
        aim_lock_until = now + (aim_cfg.lock_grace or 0.35)
        aim_lock_fail = 0
    end
    return best, bestPart, bestPos
end

-- FOV visuals: outline + dual-color fill + optional spin wedges
local aim_fov = { outline = nil, fill = nil, fill2 = nil, wedges = {} }

local function aimFovClear()
    local function rem(d)
        pcall(function() if d then d:Remove() end end)
    end
    rem(aim_fov.outline); aim_fov.outline = nil
    rem(aim_fov.fill); aim_fov.fill = nil
    rem(aim_fov.fill2); aim_fov.fill2 = nil
    for _, w in ipairs(aim_fov.wedges) do rem(w) end
    table.clear(aim_fov.wedges)
    aim_fov_draw = nil
end

local function aimDrawFOV()
    aimFovClear()
    if not aim_cfg.enabled or not aim_cfg.show_fov then return end
    if not Drawing or not Drawing.new then return end

    pcall(function()
        local o = Drawing.new("Circle")
        o.Filled = false
        o.Thickness = 2
        o.NumSides = 64
        o.Visible = true
        o.ZIndex = 3
        aim_fov.outline = o
        aim_fov_draw = o -- compat

        local f = Drawing.new("Circle")
        f.Filled = true
        f.NumSides = 64
        f.Thickness = 1
        f.Visible = false
        f.ZIndex = 1
        aim_fov.fill = f

        local f2 = Drawing.new("Circle")
        f2.Filled = true
        f2.NumSides = 64
        f2.Thickness = 1
        f2.Visible = false
        f2.ZIndex = 2
        aim_fov.fill2 = f2

        -- wedges for dual-color spin (12 triangles)
        for i = 1, 12 do
            local t = Drawing.new("Triangle")
            t.Filled = true
            t.Visible = false
            t.ZIndex = 2
            aim_fov.wedges[i] = t
        end
    end)
end

local function aimUpdateFOV()
    local cam = workspace.CurrentCamera
    if not cam then return end
    local on = aim_cfg.enabled and aim_cfg.show_fov
    local center = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)
    local r = aim_cfg.fov or 140
    local c1 = aim_cfg.fov_fill_color1 or aim_cfg.fov_color or Color3.fromRGB(255, 105, 180)
    local c2 = aim_cfg.fov_fill_color2 or Color3.fromRGB(255, 255, 255)
    local outlineCol = aim_cfg.fov_color or c1
    local trans = math.clamp(tonumber(aim_cfg.fov_fill_transparency) or 0.75, 0.3, 0.95)

    if not aim_fov.outline then
        if on then aimDrawFOV() end
        if not aim_fov.outline then return end
    end

    pcall(function()
        aim_fov.outline.Position = center
        aim_fov.outline.Radius = r
        aim_fov.outline.Color = outlineCol
        aim_fov.outline.Visible = on

        local doFill = on and aim_cfg.fov_fill
        local doSpin = doFill and aim_cfg.fov_spin

        if doFill and not doSpin then
            -- fill color1 (outer) + color2 (inner half) = 2 colores a la vez
            if aim_fov.fill then
                aim_fov.fill.Position = center
                aim_fov.fill.Radius = r
                aim_fov.fill.Color = c1
                aim_fov.fill.Transparency = trans
                aim_fov.fill.Visible = true
            end
            if aim_fov.fill2 then
                aim_fov.fill2.Position = center
                aim_fov.fill2.Radius = r * 0.55
                aim_fov.fill2.Color = c2
                aim_fov.fill2.Transparency = math.clamp(trans - 0.05, 0.25, 0.9)
                aim_fov.fill2.Visible = true
            end
            for _, w in ipairs(aim_fov.wedges) do w.Visible = false end
        elseif doSpin then
            -- hide solid fills, show spinning dual-color wedges
            if aim_fov.fill then aim_fov.fill.Visible = false end
            if aim_fov.fill2 then aim_fov.fill2.Visible = false end
            local speed = tonumber(aim_cfg.fov_spin_speed) or 2.5
            local base = tick() * speed
            local n = #aim_fov.wedges
            local step = (math.pi * 2) / n
            for i, w in ipairs(aim_fov.wedges) do
                local a0 = base + (i - 1) * step
                local a1 = a0 + step * 0.85
                local p1 = center + Vector2.new(math.cos(a0) * r, math.sin(a0) * r)
                local p2 = center + Vector2.new(math.cos(a1) * r, math.sin(a1) * r)
                w.PointA = center
                w.PointB = p1
                w.PointC = p2
                w.Color = (i % 2 == 1) and c1 or c2
                w.Transparency = trans
                w.Visible = true
            end
        else
            if aim_fov.fill then aim_fov.fill.Visible = false end
            if aim_fov.fill2 then aim_fov.fill2.Visible = false end
            for _, w in ipairs(aim_fov.wedges) do w.Visible = false end
        end
    end)
end

local function aimStop()
    if aim_conn then
        pcall(function() aim_conn:Disconnect() end)
        aim_conn = nil
    end
    aimFovClear()
    aim_current = nil
    aim_holding = false
    aim_cfg.toggled = false
end

local function aimStart()
    aimStop()
    aimDrawFOV()
    aim_conn = RunService.RenderStepped:Connect(function(dt)
        if not aim_cfg.enabled then return end
        aimUpdateFOV()

        local active = false
        if aim_cfg.toggle_mode then
            active = aim_cfg.toggled
        else
            -- hold MB2: poll real button state (más fiable que solo InputBegan/Ended)
            local mb2_down = false
            pcall(function()
                mb2_down = UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)
            end)
            aim_holding = mb2_down
            active = aim_holding
        end
        if not active then
            aim_current = nil
            return
        end

        -- shako method: target por FOV (px desde mouse) + wall opcional
        local target, part, pos = aimGetTarget()
        if not target or not part or not pos then return end

        local cam = workspace.CurrentCamera
        if not cam then return end

        if aim_cfg.wall_check then
            if not aimbotWallClear(cam.CFrame.Position, pos, target.Character) then
                return
            end
        end

        local screenPos, onScreen = cam:WorldToViewportPoint(pos)
        if not onScreen and screenPos.Z < 0 then return end

        -- Assist suave: NUNCA bloquea el mouse.
        -- Solo corrige un poco hacia el target; tú sigues controlando la cámara.
        -- leer live del config (slider Smooth %)
        local live = getgenv()._Matcha_aim or aim_cfg
        local smooth = math.clamp(tonumber(live.smooth) or 0.28, 0.01, 1)
        -- smooth alto = más lento / más control del mouse
        local strength = math.clamp(1.05 - smooth, 0.08, 0.55)
        local maxStep = tonumber(live.max_step) or 6

        local movedMouse = false
        pcall(function()
            local mouse = UIS:GetMouseLocation()
            local dx = (screenPos.X - mouse.X) * strength
            local dy = (screenPos.Y - mouse.Y) * strength
            -- cap: no teletransporta el cursor, puedes pelear con el aim
            if dx > maxStep then dx = maxStep elseif dx < -maxStep then dx = -maxStep end
            if dy > maxStep then dy = maxStep elseif dy < -maxStep then dy = -maxStep end
            if math.abs(dx) < 0.2 and math.abs(dy) < 0.2 then return end
            if mousemoverel then
                mousemoverel(dx, dy)
                movedMouse = true
            end
        end)

        -- fallback CFrame MUY suave (nunca lock total)
        if not movedMouse then
            local origin = cam.CFrame.Position
            local goal = CFrame.lookAt(origin, pos)
            local alpha = 1 - math.exp(-dt / math.max(smooth, 0.12))
            alpha = math.clamp(alpha * strength, 0, 0.22) -- bajo = siempre puedes mirar a otro lado
            cam.CFrame = cam.CFrame:Lerp(goal, alpha)
        end
    end)
end

-- input for hold / toggle (MB2 = clic derecho, disimula apuntando)
-- NO filtramos gameProcessed en MB2: el juego lo usa para apuntar y si lo bloqueamos nunca activa
local function aimIsHoldInput(input)
    local key = aim_cfg.hold_key
    if key == nil then return false end
    if typeof(key) == "EnumItem" then
        if key.EnumType == Enum.UserInputType and input.UserInputType == key then
            return true
        end
        if key.EnumType == Enum.KeyCode and input.KeyCode == key then
            return true
        end
    end
    -- fallback fijo a MB2
    if input.UserInputType == Enum.UserInputType.MouseButton2 then
        return true
    end
    return false
end

UIS.InputBegan:Connect(function(input, gp)
    if not aimIsHoldInput(input) then return end
    -- solo ignoramos gp en teclado; MB2 siempre cuenta (así se disimula el aim)
    if gp and input.UserInputType ~= Enum.UserInputType.MouseButton2
        and input.UserInputType ~= Enum.UserInputType.MouseButton1
        and input.UserInputType ~= Enum.UserInputType.MouseButton3 then
        return
    end
    if aim_cfg.toggle_mode then
        if aim_cfg.enabled then
            aim_cfg.toggled = not aim_cfg.toggled
        end
    else
        aim_holding = true
    end
end)

UIS.InputEnded:Connect(function(input, gp)
    if not aimIsHoldInput(input) then return end
    if not aim_cfg.toggle_mode then
        aim_holding = false
    end
end)

-- por si el cursor se suelta fuera de la ventana
UIS.WindowFocusReleased:Connect(function()
    if not aim_cfg.toggle_mode then
        aim_holding = false
    end
end)

aim_cfg._start = aimStart
aim_cfg._stop = aimStop


-- ==================== SILENT AIM COMPLETO (Utility + Bullet + cast) ====================
--[[
  3 hooks para que las balas vayan al head CON o SIN apuntar, cualquier arma:
  1) Modules.Utility.Raycast (args[4]==999)  — source user
  2) Bullet._performRaycast                  — Ragebot2
  3) getgc().cast                            — Ragebot2
]]
local silent_cfg = {
    enabled = false,
    team_check = false,
    force_ffa = true, -- por defecto true: agarra a cualquiera on-screen
    fov_enabled = true,
    fov = 250, -- FOV más amplio para que sea fácil
    fov_color = Color3.fromRGB(255, 105, 180),
    show_fov = true,
    fov_fill = false,
    fov_spin = false,
    fov_spin_speed = 2.5,
    fov_fill_color1 = Color3.fromRGB(255, 105, 180),
    fov_fill_color2 = Color3.fromRGB(255, 255, 255),
    fov_fill_transparency = 0.75,
    fov_fill = false, -- SIN fill por defecto
    mode = "Classic", -- Classic | Mouse
}
getgenv()._Matcha_silent = silent_cfg

getgenv().Rivals = getgenv().Rivals or {}
getgenv().Rivals.Silent = { enabled = false }
getgenv().Rivals.Fov = {
    Enabled = true,
    Fov = 250,
    Color = Color3.fromRGB(255, 105, 180)
}

local silent_circle = nil
local silent_target_head = nil -- Head part actual
local silent_hooks_done = false

local silent_fov = { outline = nil, fill = nil, fill2 = nil, wedges = {} }

local function silentFovClear()
    local function rem(d)
        pcall(function() if d then d:Remove() end end)
    end
    rem(silent_fov.outline); silent_fov.outline = nil
    rem(silent_fov.fill); silent_fov.fill = nil
    rem(silent_fov.fill2); silent_fov.fill2 = nil
    for _, w in ipairs(silent_fov.wedges) do rem(w) end
    table.clear(silent_fov.wedges)
    silent_circle = nil
    silent_fill = nil
end

local function silentFovEnsure()
    if silent_fov.outline then return end
    if not Drawing or not Drawing.new then return end
    pcall(function()
        local o = Drawing.new("Circle")
        o.Filled = false
        o.Thickness = 2
        o.NumSides = 64
        o.Visible = false
        o.ZIndex = 3
        silent_fov.outline = o
        silent_circle = o

        local f = Drawing.new("Circle")
        f.Filled = true
        f.NumSides = 64
        f.Visible = false
        f.ZIndex = 1
        silent_fov.fill = f
        silent_fill = f

        local f2 = Drawing.new("Circle")
        f2.Filled = true
        f2.NumSides = 64
        f2.Visible = false
        f2.ZIndex = 2
        silent_fov.fill2 = f2

        for i = 1, 12 do
            local t = Drawing.new("Triangle")
            t.Filled = true
            t.Visible = false
            t.ZIndex = 2
            silent_fov.wedges[i] = t
        end
    end)
end

local function silentUpdateFov()
    local cam = workspace.CurrentCamera
    if not cam then return end
    silentFovEnsure()
    if not silent_fov.outline then return end

    local on = silent_cfg.enabled and silent_cfg.show_fov and (silent_cfg.fov_enabled ~= false)
    local r = silent_cfg.fov or 250
    local c1 = silent_cfg.fov_fill_color1 or silent_cfg.fov_color or Color3.fromRGB(255, 105, 180)
    local c2 = silent_cfg.fov_fill_color2 or Color3.fromRGB(255, 255, 255)
    local outlineCol = silent_cfg.fov_color or c1
    local trans = math.clamp(tonumber(silent_cfg.fov_fill_transparency) or 0.75, 0.3, 0.95)

    local pos
    if silent_cfg.mode == "Mouse" then
        local m = UserInputService:GetMouseLocation()
        pos = Vector2.new(m.X, m.Y)
    else
        pos = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)
    end

    pcall(function()
        silent_fov.outline.Position = pos
        silent_fov.outline.Radius = r
        silent_fov.outline.Color = outlineCol
        silent_fov.outline.Visible = on

        local doFill = on and silent_cfg.fov_fill
        local doSpin = doFill and silent_cfg.fov_spin

        if doFill and not doSpin then
            if silent_fov.fill then
                silent_fov.fill.Position = pos
                silent_fov.fill.Radius = r
                silent_fov.fill.Color = c1
                silent_fov.fill.Transparency = trans
                silent_fov.fill.Visible = true
            end
            if silent_fov.fill2 then
                silent_fov.fill2.Position = pos
                silent_fov.fill2.Radius = r * 0.55
                silent_fov.fill2.Color = c2
                silent_fov.fill2.Transparency = math.clamp(trans - 0.05, 0.25, 0.9)
                silent_fov.fill2.Visible = true
            end
            for _, w in ipairs(silent_fov.wedges) do w.Visible = false end
        elseif doSpin then
            if silent_fov.fill then silent_fov.fill.Visible = false end
            if silent_fov.fill2 then silent_fov.fill2.Visible = false end
            local speed = tonumber(silent_cfg.fov_spin_speed) or 2.5
            local base = tick() * speed
            local n = #silent_fov.wedges
            local step = (math.pi * 2) / math.max(n, 1)
            for i, w in ipairs(silent_fov.wedges) do
                local a0 = base + (i - 1) * step
                local a1 = a0 + step * 0.85
                w.PointA = pos
                w.PointB = pos + Vector2.new(math.cos(a0) * r, math.sin(a0) * r)
                w.PointC = pos + Vector2.new(math.cos(a1) * r, math.sin(a1) * r)
                w.Color = (i % 2 == 1) and c1 or c2
                w.Transparency = trans
                w.Visible = true
            end
        else
            if silent_fov.fill then silent_fov.fill.Visible = false end
            if silent_fov.fill2 then silent_fov.fill2.Visible = false end
            for _, w in ipairs(silent_fov.wedges) do w.Visible = false end
        end
    end)

    pcall(function()
        getgenv().Rivals.Silent.enabled = silent_cfg.enabled
        getgenv().Rivals.Fov.Enabled = silent_cfg.fov_enabled and silent_cfg.show_fov
        getgenv().Rivals.Fov.Fov = silent_cfg.fov or 250
        getgenv().Rivals.Fov.Color = silent_cfg.fov_color
    end)
end

-- loop FOV position (Mouse mode)
task.spawn(function()
    while true do
        task.wait(0.03)
        pcall(silentUpdateFov)
    end
end)

pcall(function()
    workspace.CurrentCamera:GetPropertyChangedSignal("ViewportSize"):Connect(silentUpdateFov)
end)

-- entidades: Characters (Rivals) + Workspace Humanoid + HurtEffect
local function silent_getEntities()
    local list, seen = {}, {}

    pcall(function()
        local folder = Workspace:FindFirstChild("Characters")
        if folder then
            for _, ch in ipairs(folder:GetChildren()) do
                if ch.Name ~= LocalPlayer.Name and not seen[ch] then
                    list[#list + 1] = ch
                    seen[ch] = true
                end
            end
        end
    end)

    for _, child in ipairs(Workspace:GetChildren()) do
        if seen[child] then continue end
        if child:FindFirstChildOfClass("Humanoid") then
            if child.Name ~= LocalPlayer.Name then
                list[#list + 1] = child
                seen[child] = true
            end
        elseif child.Name == "HurtEffect" then
            for _, hurt in ipairs(child:GetChildren()) do
                if hurt.ClassName ~= "Highlight" and not seen[hurt] then
                    list[#list + 1] = hurt
                    seen[hurt] = true
                end
            end
        end
    end

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and not seen[plr.Character] then
            list[#list + 1] = plr.Character
            seen[plr.Character] = true
        end
    end

    return list
end

local function silent_isEnemy(char)
    if not char or char.Name == LocalPlayer.Name then return false end
    if silent_cfg.force_ffa or not silent_cfg.team_check then return true end
    local plr = Players:FindFirstChild(char.Name) or Players:GetPlayerFromCharacter(char)
    if not plr then return true end
    local myT = LocalPlayer:GetAttribute("Team")
    local ot = plr:GetAttribute("Team")
    if myT ~= nil and ot ~= nil then return myT ~= ot end
    if LocalPlayer.Team and plr.Team then return LocalPlayer.Team ~= plr.Team end
    return true
end

-- closest head al crosshair (FOV)
local function silent_getClosestHead()
    local cam = workspace.CurrentCamera
    if not cam then return nil end
    local closest, closestDist = nil, math.huge
    local center = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y / 2)
    local fov = silent_cfg.fov or 250

    for _, char in ipairs(silent_getEntities()) do
        if not silent_isEnemy(char) then continue end

        local dead, hp = false, 100
        pcall(function()
            dead = char:GetAttribute("Dead") == true
            hp = char:GetAttribute("Health") or 100
        end)
        if dead or hp <= 0 then continue end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health <= 0 then continue end

        local head = char:FindFirstChild("Head")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not head and not hrp then continue end
        local part = head or hrp

        local pos, onScreen = cam:WorldToViewportPoint(part.Position)
        if not onScreen or pos.Z <= 0 then continue end

        local dist = (center - Vector2.new(pos.X, pos.Y)).Magnitude
        if silent_cfg.fov_enabled and dist > fov then continue end

        if dist < closestDist then
            closestDist = dist
            closest = head or hrp
        end
    end
    return closest
end

-- target loop siempre (cuando silent ON las balas usan esta head)
RunService.RenderStepped:Connect(function()
    if silent_cfg.enabled then
        silent_target_head = silent_getClosestHead()
        silentUpdateFov()
    else
        silent_target_head = nil
        if silent_circle then silent_circle.Visible = false end
    end
end)

-- HOOKS x3
task.spawn(function()
    -- 1) Utility.Raycast (source user)
    local util = nil
    for _ = 1, 40 do
        local ok, u = pcall(function()
            return require(ReplicatedStorage.Modules.Utility)
        end)
        if ok and u and type(u.Raycast) == "function" then
            util = u
            break
        end
        task.wait(0.4)
    end
    if util then
        local oldRaycast = util.Raycast
        util.Raycast = function(...)
            local args = {...}
            if silent_cfg.enabled and #args > 0 and args[4] == 999 then
                local head = silent_target_head or silent_getClosestHead()
                if head then
                    args[3] = head.Position
                end
            end
            return oldRaycast(table.unpack(args))
        end
        print("[Silent] 1/3 Utility.Raycast OK")
    else
        warn("[Silent] Utility.Raycast no encontrado")
    end

    -- 2) Bullet._performRaycast (balas reales, con o sin ADS)
    local Bullet = nil
    for _ = 1, 40 do
        local ok, b = pcall(function()
            return require(ReplicatedStorage.Components.Weapon.Classes.Bullet)
        end)
        if ok and b then Bullet = b; break end
        task.wait(0.4)
    end
    if Bullet and Bullet._performRaycast then
        local OldRaycast
        local okH = pcall(function()
            OldRaycast = hookfunction(Bullet._performRaycast, function(Self, Spread)
                local Result = OldRaycast(Self, Spread)
                if not Result then return Result end
                -- silent O rage silent
                local use = silent_cfg.enabled
                local head = silent_target_head
                if not use then
                    local rc = getgenv()._Vnix_rage
                    if rc and rc.enabled and rc.silent then
                        use = true
                        head = head -- rage tiene su propio target, pero si silent_target existe úsalo
                    end
                end
                if not use or not head then return Result end

                local Position = head.Position
                local Offset = Position - Result.Origin
                if Offset.Magnitude < 0.01 then return Result end
                local Direction = Offset.Unit

                Result.Direction = Direction
                Result.Distance = Offset.Magnitude
                Result.Hits = {
                    {
                        Position = Position,
                        Instance = head,
                        Material = "Plastic",
                        Normal = -Direction,
                        Exit = false
                    }
                }
                return Result
            end)
        end)
        if okH then
            print("[Silent] 2/3 Bullet._performRaycast OK")
        else
            warn("[Silent] Bullet hook falló")
        end
    else
        warn("[Silent] Bullet class no encontrada")
    end

    -- 3) getgc cast (dirección del proyectil)
    pcall(function()
        local n = 0
        for _, v in next, getgc(true) do
            if type(v) == "table" and rawget(v, "cast") and type(v.cast) == "function" then
                local old
                old = hookfunction(v.cast, function(...)
                    local args = { ... }
                    if silent_cfg.enabled and args[2] and typeof(args[2]) == "Vector3" then
                        local head = silent_target_head
                        if head then
                            local origin = args[1]
                            if typeof(origin) == "Vector3" then
                                args[2] = (head.Position - origin)
                            end
                        end
                    end
                    return old(unpack(args))
                end)
                n += 1
            end
        end
        print("[Silent] 3/3 cast hooks:", n)
    end)

    silent_hooks_done = true
    print("[Silent] LISTO — activa Silent Aim y dispara (con o sin apuntar)")
end)

function silentStart()
    silent_cfg.enabled = true
    getgenv().Rivals.Silent.enabled = true
    silentUpdateFov()
    print("[Silent] ENABLED — balas → head (FOV", silent_cfg.fov, ")")
end
function silentStop()
    silent_cfg.enabled = false
    getgenv().Rivals.Silent.enabled = false
    silent_target_head = nil
    if silent_circle then silent_circle.Visible = false end
    print("[Silent] DISABLED")
end
silent_cfg._start = silentStart
silent_cfg._stop = silentStop

-- ==================== MANIPULATION UE (Combat) ====================
--[[ Source: manipulation ue.txt — UseItem StartShooting con cámara manipulada ]]
local manip_cfg = {
    enabled = false,
    only_enemies = true,
}
getgenv()._Matcha_manip = manip_cfg

local manip_conn = nil
local manip_offsets = {
    Vector3.new(0, 12, 0),
    Vector3.new(0, 16, 0),
    Vector3.new(0, 20, 0),
    Vector3.new(0, 24, 0),
    Vector3.new(0, 28, 0),
    Vector3.new(0, 32, 0),
    Vector3.new(0, 36, 0),
    Vector3.new(0, 40, 0),
}

local function manipGetClosest()
    local char = LocalPlayer.Character
    if not char then return nil, nil end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil, nil end
    local target, tchar, dist = nil, nil, math.huge
    for _, v in ipairs(Players:GetPlayers()) do
        if v == LocalPlayer then continue end
        local vc = v.Character
        if not vc then continue end
        local head = vc:FindFirstChild("Head")
        if not head then continue end
        -- team check opcional
        if manip_cfg.only_enemies then
            local ok, isEnemy = pcall(function()
                if type(MatchaIsEnemy) == "function" then
                    return MatchaIsEnemy(v, true, false)
                end
                if LocalPlayer.Team and v.Team and LocalPlayer.Team == v.Team then return false end
                return true
            end)
            if ok and not isEnemy then continue end
        end
        local mag = (hrp.Position - head.Position).Magnitude
        if mag < dist then
            dist = mag
            target = head
            tchar = vc
        end
    end
    return target, tchar
end

local function manipCalculatePoint(origin, target_pos, target_char)
    local ray_params = RaycastParams.new()
    ray_params.FilterDescendantsInstances = { LocalPlayer.Character, target_char }
    ray_params.FilterType = Enum.RaycastFilterType.Exclude

    if not Workspace:Raycast(origin, target_pos - origin, ray_params) then
        return origin, nil
    end
    for _, offset in ipairs(manip_offsets) do
        local scan_pos = origin + offset
        if not Workspace:Raycast(scan_pos, target_pos - scan_pos, ray_params) then
            return scan_pos, offset.Y
        end
    end
    return nil, nil
end

local function manipStop()
    manip_cfg.enabled = false
    if manip_conn then
        pcall(function() manip_conn:Disconnect() end)
        manip_conn = nil
    end
    print("[Matcha] Manipulation UE OFF")
end

local function manipStart()
    manipStop()
    manip_cfg.enabled = true
    manip_conn = RunService.Heartbeat:Connect(function()
        if not manip_cfg.enabled then return end
        local char = LocalPlayer.Character
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then return end

        local ok, err = pcall(function()
            local fighter = nil
            pcall(function()
                fighter = require(LocalPlayer.PlayerScripts.Controllers.FighterController)
            end)
            if not fighter or not fighter.LocalFighter then return end
            local item = fighter.LocalFighter.EquippedItem
            if not item then return end

            local util = nil
            pcall(function() util = require(ReplicatedStorage.Modules.Utility) end)
            local enums = nil
            pcall(function() enums = require(ReplicatedStorage.Modules.EnumLibrary) end)
            if not util or not enums or not util.EncodeCFrame then return end

            local target_part, target_char = manipGetClosest()
            if not target_part or not target_char then return end

            local cam = Workspace.CurrentCamera
            if not cam then return end
            local manip, height = manipCalculatePoint(cam.CFrame.Position, target_part.Position, target_char)
            if not manip then return end

            local shoot_pos = (height == nil and manip) or cam.CFrame.Position
            local lookCF = CFrame.lookAt(shoot_pos, target_part.Position)
            local ox, oy, oz = lookCF:ToOrientation()

            local cameradata = {}
            cameradata[utf8.char(1)] = {
                [utf8.char(0)] = util:EncodeCFrame(
                    CFrame.new(shoot_pos.X, shoot_pos.Y + (height or 0), shoot_pos.Z) * CFrame.Angles(ox, oy, oz)
                ),
                [utf8.char(1)] = height
                    and util:EncodeCFrame(CFrame.new(target_part.Position) * CFrame.Angles(ox, oy, oz))
                    or util:EncodeCFrame(
                        CFrame.new(shoot_pos.X, shoot_pos.Y + (height or 0), shoot_pos.Z) * CFrame.Angles(ox, oy, oz)
                    ),
                [utf8.char(2)] = target_part,
                [utf8.char(3)] = util:EncodeCFrame(target_part.CFrame:ToObjectSpace(CFrame.new(target_part.Position))),
            }

            local remote = ReplicatedStorage:FindFirstChild("Remotes")
            remote = remote and remote:FindFirstChild("Replication")
            remote = remote and remote:FindFirstChild("Fighter")
            remote = remote and remote:FindFirstChild("UseItem")
            if not remote then return end

            local objectId = nil
            pcall(function() objectId = item:Get("ObjectID") end)
            if not objectId then return end

            local startEnum = nil
            pcall(function() startEnum = enums:ToEnum("StartShooting") end)
            if startEnum == nil then return end

            remote:FireServer(objectId, startEnum, cameradata, nil)
        end)
    end)
    print("[Matcha] Manipulation UE ON")
end

manip_cfg._start = manipStart
manip_cfg._stop = manipStop

-- ==================== SLIDE BOOST (Misc) ====================
--[[ Source: slide boost.txt — MechanicsController._sliding_velocity ]]
local slide_cfg = {
    enabled = false,
    speed = 100,
}
getgenv()._Matcha_slide = slide_cfg

local slide_conn = nil
local function slideStop()
    slide_cfg.enabled = false
    if slide_conn then
        pcall(function() slide_conn:Disconnect() end)
        slide_conn = nil
    end
    print("[Matcha] Slide Boost OFF")
end

local function slideStart()
    slideStop()
    slide_cfg.enabled = true
    slide_conn = RunService.RenderStepped:Connect(function()
        if not slide_cfg.enabled then return end
        pcall(function()
            local mech = require(LocalPlayer.PlayerScripts.Controllers.MechanicsController)
            if not mech or not mech.IsSliding then return end
            local sv = mech._sliding_velocity
            if not sv then return end
            local vel = sv.Velocity
            if vel.Magnitude < 0.05 then return end
            local spd = tonumber(slide_cfg.speed) or 100
            sv.Velocity = vel.Unit * spd
        end)
    end)
    print("[Matcha] Slide Boost ON · speed=", slide_cfg.speed)
end

slide_cfg._start = slideStart
slide_cfg._stop = slideStop

-- ==================== RANKEDS (Auto Queue / Vote / Loadout / Ban) ====================
local ranked_cfg = {
    auto_queue = false,
    queue_mode = "1v1",
    auto_vote = false,
    vote_map = "Arena",
    auto_loadout = false,
    primary = "Assault Rifle",
    secondary = "Handgun",
    melee = "Fists",
    utility = "Grenade",
    auto_ban = false,
    ban_list = {}, -- weapons/maps to cycle-ban
    ban_index = 1,
}
getgenv()._Matcha_ranked = ranked_cfg

local RANKED_MAPS = {
    "Arena", "Big Graveyard", "Docks", "Splash", "Bridge", "Crossroads",
    "Big Crossroads", "Big Backrooms", "Battleground", "Big Arena",
    "Construction", "Playground", "Onyx", "Graveyard", "Big Splash",
    "Big Onyx", "Backrooms", "Station", "Dimension",
}
local RANKED_PRIMARY = {
    "Crossbow", "Grenade Launcher", "Brust Rifle", "Distortion", "Flamethrower",
    "Shotgun", "RPG", "Assault Rifle", "Sniper", "Minigun", "Energy Rifle",
    "Paintball Gun", "Bow", "Gunblade",
}
local RANKED_SECONDARY = {
    "Slingshot", "Flare Gun", "Spray", "Shorty", "Energy Pistols", "Warper",
    "Daggers", "Scepter", "Exogun", "Revolver", "Uzi", "Glass Cannon", "Handgun",
}
local RANKED_MELEE = {
    "Fists", "Scythe", "Battle Axe", "Trowel", "Riot Shield", "Glast Shard",
    "Knife", "Chainsaw", "Katana",
}
local RANKED_UTILITY = {
    "Medkit", "Flashbang", "Elixir", "Subspace Tripmine", "War Horn", "Grenade",
    "Warpstone", "Molotov", "Freeze Ray", "RNG Dice", "Smoke Grenade", "Jump Pad", "Satchel",
}
local RANKED_BAN_ITEMS = {}
for _, n in ipairs(RANKED_MAPS) do table.insert(RANKED_BAN_ITEMS, n) end
for _, n in ipairs(RANKED_PRIMARY) do table.insert(RANKED_BAN_ITEMS, n) end
for _, n in ipairs(RANKED_SECONDARY) do table.insert(RANKED_BAN_ITEMS, n) end

local function rankedIsInMatch()
    -- heurística: Characters en workspace o atributo de match
    local ok, inMatch = pcall(function()
        if LocalPlayer:GetAttribute("InMatch") == true then return true end
        if LocalPlayer:GetAttribute("MatchId") then return true end
        local folder = Workspace:FindFirstChild("Characters")
        if folder and #folder:GetChildren() > 0 then
            local myChar = LocalPlayer.Character
            if myChar and folder:FindFirstChild(LocalPlayer.Name) then return true end
        end
        return false
    end)
    return ok and inMatch
end

local function rankedGetVoteRemote()
    local r = ReplicatedStorage:FindFirstChild("Remotes")
    r = r and r:FindFirstChild("Duels")
    r = r and r:FindFirstChild("Vote")
    return r
end

local function rankedGetQueueRemote()
    local r = ReplicatedStorage:FindFirstChild("Remotes")
    r = r and r:FindFirstChild("Matchmaking")
    r = r and r:FindFirstChild("JoinQueue")
    return r
end

local function rankedGetPickWeapons()
    local r = ReplicatedStorage:FindFirstChild("Remotes")
    r = r and r:FindFirstChild("Replication")
    r = r and r:FindFirstChild("Fighter")
    r = r and r:FindFirstChild("PickWeapons")
    return r
end

-- Auto Queue loop
task.spawn(function()
    while true do
        task.wait(1.5)
        if not ranked_cfg.auto_queue then continue end
        pcall(function()
            if rankedIsInMatch() then return end
            local rem = rankedGetQueueRemote()
            if not rem then return end
            local mode = ranked_cfg.queue_mode or "1v1"
            if rem:IsA("RemoteFunction") then
                rem:InvokeServer(mode)
            else
                rem:FireServer(mode)
            end
        end)
    end
end)

-- Auto Vote Map
task.spawn(function()
    while true do
        task.wait(0.75)
        if not ranked_cfg.auto_vote then continue end
        pcall(function()
            local rem = rankedGetVoteRemote()
            if not rem then return end
            rem:FireServer(ranked_cfg.vote_map or "Arena")
        end)
    end
end)

-- Auto Loadout
task.spawn(function()
    while true do
        task.wait(2)
        if not ranked_cfg.auto_loadout then continue end
        pcall(function()
            local rem = rankedGetPickWeapons()
            if not rem then return end
            rem:FireServer({
                ranked_cfg.primary or "Assault Rifle",
                ranked_cfg.secondary or "Handgun",
                ranked_cfg.melee or "Fists",
                ranked_cfg.utility or "Grenade",
            })
        end)
    end
end)

-- Auto Ban (cycle ban_list via Duels.Vote)
task.spawn(function()
    while true do
        task.wait(1)
        if not ranked_cfg.auto_ban then continue end
        local list = ranked_cfg.ban_list
        if type(list) ~= "table" or #list == 0 then continue end
        pcall(function()
            local rem = rankedGetVoteRemote()
            if not rem then return end
            local idx = ranked_cfg.ban_index or 1
            if idx < 1 or idx > #list then idx = 1 end
            rem:FireServer(list[idx])
            ranked_cfg.ban_index = (idx % #list) + 1
        end)
    end
end)

getgenv()._Matcha_ranked_lists = {
    maps = RANKED_MAPS,
    primary = RANKED_PRIMARY,
    secondary = RANKED_SECONDARY,
    melee = RANKED_MELEE,
    utility = RANKED_UTILITY,
    ban_items = RANKED_BAN_ITEMS,
}



local trigger_cfg = {
    enabled = false,
    team_check = true,
    force_ffa = false,
    wall_check = true,
    delay = 0.02,
    release = 0.05,
    hitbox_mul = 1.2,
    last_shot = 0,
}
getgenv()._Matcha_trigger = trigger_cfg

local char_cfg = {
    speed_enabled = false,
    speed = 35,
    inf_jump = false,
    noclip = false,
    fly = false,
    fly_speed = 50,
    target_hover = false,
    hover_radius = 12,
    hover_speed = 7,
    hitbox_expand = false,
    hitbox_size = 5,
}
getgenv()._Matcha_char = char_cfg

-- TRIGGERBOT: auto click when enemy under crosshair
local function MatchaEnemy(plr, tc, ffa)
    if not plr or plr == LocalPlayer then return false end
    if ffa or not tc then return true end
    if type(MatchaIsEnemy) == "function" then
        local ok, r = pcall(MatchaIsEnemy, plr, tc, ffa)
        if ok then return r end
    end
    local myT = LocalPlayer:GetAttribute("Team")
    local ot = plr:GetAttribute("Team")
    if myT ~= nil and ot ~= nil then return myT ~= ot end
    if LocalPlayer.Team and plr.Team then return LocalPlayer.Team ~= plr.Team end
    return true
end

local function triggerIsOnTarget()
    local cam = workspace.CurrentCamera
    if not cam then return false end
    local origin = cam.CFrame.Position
    local dir = cam.CFrame.LookVector * 500
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {LocalPlayer.Character}
    params.IgnoreWater = true
    local result = Workspace:Raycast(origin, dir, params)
    if not result or not result.Instance then return false end
    local model = result.Instance:FindFirstAncestorOfClass("Model")
    if not model then return false end
    local plr = Players:GetPlayerFromCharacter(model)
    if not plr then return false end
    if not MatchaEnemy(plr, trigger_cfg.team_check, trigger_cfg.force_ffa) then return false end
    local hum = model:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    return true
end

function triggerStart()
    if trigger_conn then pcall(function() trigger_conn:Disconnect() end) end
    trigger_conn = RunService.Heartbeat:Connect(function()
        if not trigger_cfg.enabled then return end
        if tick() - (trigger_cfg.last_shot or 0) < (trigger_cfg.delay or 0.02) then return end
        if triggerIsOnTarget() then
            trigger_cfg.last_shot = tick()
            pcall(function()
                mouse1press()
                task.delay(trigger_cfg.release or 0.05, function()
                    pcall(mouse1release)
                end)
            end)
            -- fallback VirtualInput
            pcall(function()
                if not mouse1press then
                    local vim = game:GetService("VirtualInputManager")
                    vim:SendMouseButtonEvent(0, 0, 0, true, game, 0)
                    task.delay(trigger_cfg.release or 0.05, function()
                        vim:SendMouseButtonEvent(0, 0, 0, false, game, 0)
                    end)
                end
            end)
        end
    end)
end
function triggerStop()
    trigger_cfg.enabled = false
    if trigger_conn then pcall(function() trigger_conn:Disconnect() end); trigger_conn = nil end
end
trigger_cfg._start = triggerStart
trigger_cfg._stop = triggerStop

-- ==================== ITEM LIBRARY: Fast Fire + Fast Melee (seguro, sin lag) ====================
--[[
  ANTES: aplicaba Cooldown a TODOS los items → el juego disparaba/picaba todo y lag.
  AHORA: solo armas con ShootCooldown / solo melees con AttackCooldown|SwingCooldown.
  Delay mínimo razonable para no saturar el cliente.
]]
local ItemBackup = {}
local gunExceptions = {
    Sniper = true, Crossbow = true, Bow = true, RPG = true,
    ["Flare Gun"] = true, Flaregun = true,
}
local il_cfg = {
    rapid_fire = false,
    rapid_delay = 0.025, -- más alto = menos lag (antes 0.001 era extremo)
    no_recoil = true,
    fast_melee = false,
    melee_delay = 0.05,
    keep_alive = true,
}
getgenv()._Matcha_itemlib = il_cfg

local function ilGetItems()
    local ok, mod = pcall(function()
        return require(ReplicatedStorage.Modules.ItemLibrary)
    end)
    if ok and mod and mod.Items then return mod.Items end
    return nil
end

local function ilSave(name, data, prop)
    if data[prop] == nil then return end
    ItemBackup[name] = ItemBackup[name] or {}
    if ItemBackup[name][prop] == nil then
        ItemBackup[name][prop] = data[prop]
    end
end

local function ilSet(name, data, prop, val)
    if data[prop] == nil then return end
    ilSave(name, data, prop)
    data[prop] = val
end

local function ilRestoreName(name)
    local Items = ilGetItems()
    if not Items or not Items[name] then return end
    local data = Items[name]
    local b = ItemBackup[name]
    if not b or typeof(data) ~= "table" then return end
    for prop, old in pairs(b) do
        if data[prop] ~= nil then data[prop] = old end
    end
end

local function ilRestoreAll()
    local Items = ilGetItems()
    if not Items then return end
    for name, _ in pairs(ItemBackup) do
        local data = Items[name]
        if typeof(data) == "table" then
            local b = ItemBackup[name]
            if b then
                for prop, old in pairs(b) do
                    if data[prop] ~= nil then data[prop] = old end
                end
            end
        end
    end
end

local function isGunItem(name, data)
    if gunExceptions[name] then return false end
    -- solo si es arma de fuego real
    if data.ShootCooldown ~= nil or data.ShootBurstCooldown ~= nil then return true end
    if data.Ammo ~= nil or data.MagazineSize ~= nil or data.MaxAmmo ~= nil then return true end
    return false
end

local function isMeleeItem(name, data)
    -- NUNCA tocar items genéricos con solo "Cooldown"
    if data.ShootCooldown ~= nil or data.Ammo ~= nil then return false end
    if data.AttackCooldown ~= nil or data.SwingCooldown ~= nil or data.MeleeCooldown ~= nil then
        return true
    end
    local n = string.lower(tostring(name))
    if n:find("katana") or n:find("knife") or n:find("melee") or n:find("sword")
        or n:find("axe") or n:find("bat") or n:find("scythe") then
        return true
    end
    return false
end

local function ApplyItemLibraryGuns(on)
    local Items = ilGetItems()
    if not Items then return false end
    local d = math.clamp(tonumber(il_cfg.rapid_delay) or 0.025, 0.012, 0.12)
    local n = 0
    for name, data in pairs(Items) do
        if typeof(data) ~= "table" then continue end
        if not isGunItem(name, data) then continue end
        if on then
            if il_cfg.no_recoil then
                ilSet(name, data, "ShootSpread", 0)
                ilSet(name, data, "ShootRecoil", 0)
                ilSet(name, data, "Recoil", 0)
                ilSet(name, data, "Spread", 0)
            end
            -- NO tocar ShootAccuracy (puede romper hitscan / causar spam raro)
            ilSet(name, data, "ShootCooldown", d)
            ilSet(name, data, "ShootBurstCooldown", d)
            n += 1
        else
            ilRestoreName(name)
        end
    end
    return n > 0 or on == false
end

local function ApplyItemLibraryMelee(on)
    local Items = ilGetItems()
    if not Items then return false end
    local d = math.clamp(tonumber(il_cfg.melee_delay) or 0.05, 0.02, 0.2)
    local n = 0
    for name, data in pairs(Items) do
        if typeof(data) ~= "table" then continue end
        if not isMeleeItem(name, data) then continue end
        if on then
            -- SOLO cooldowns de melee — NUNCA Cooldown/Reset genérico de otros items
            ilSet(name, data, "AttackCooldown", d)
            ilSet(name, data, "SwingCooldown", d)
            ilSet(name, data, "MeleeCooldown", d)
            n += 1
        else
            ilRestoreName(name)
        end
    end
    return n > 0 or on == false
end

local function StartRapidFire()
    il_cfg.rapid_fire = true
    local ok = ApplyItemLibraryGuns(true)
    if ok then
        print("[Matcha] Fast Fire ON · delay=", il_cfg.rapid_delay)
    else
        warn("[Matcha] Fast Fire FAIL — ItemLibrary / sin armas")
    end
    return ok
end

local function StopRapidFire()
    il_cfg.rapid_fire = false
    ApplyItemLibraryGuns(false)
    if il_cfg.fast_melee then ApplyItemLibraryMelee(true) end
    print("[Matcha] Fast Fire OFF")
end

local function StartFastMelee()
    il_cfg.fast_melee = true
    local ok = ApplyItemLibraryMelee(true)
    if ok then
        print("[Matcha] Fast Melee ON · delay=", il_cfg.melee_delay)
    else
        warn("[Matcha] Fast Melee FAIL")
    end
    return ok
end

local function StopFastMelee()
    il_cfg.fast_melee = false
    ApplyItemLibraryMelee(false)
    if il_cfg.rapid_fire then ApplyItemLibraryGuns(true) end
    print("[Matcha] Fast Melee OFF")
end

il_cfg._startGuns = StartRapidFire
il_cfg._stopGuns = StopRapidFire
il_cfg._startMelee = StartFastMelee
il_cfg._stopMelee = StopFastMelee
il_cfg._applyGuns = ApplyItemLibraryGuns
il_cfg._applyMelee = ApplyItemLibraryMelee
il_cfg._restoreAll = ilRestoreAll

-- keep-alive ligero (solo si está activo)
task.spawn(function()
    while true do
        task.wait(8)
        if not il_cfg.keep_alive then continue end
        if il_cfg.rapid_fire then pcall(ApplyItemLibraryGuns, true) end
        if il_cfg.fast_melee then pcall(ApplyItemLibraryMelee, true) end
    end
end)

-- CHARACTER / MOVEMENT
local fly_bv, fly_bg
local speed_conn_prop = nil
local function charApplySpeed()
    if not char_cfg.speed_enabled then return end
    local ch = LocalPlayer.Character
    if not ch then return end
    local hum = ch:FindFirstChildOfClass("Humanoid")
    local hrp = ch:FindFirstChild("HumanoidRootPart")
    local spd = tonumber(char_cfg.speed) or 22

    -- 1) WalkSpeed force (juegos vanilla)
    if hum then
        pcall(function()
            hum.WalkSpeed = spd
        end)
    end

    -- 2) Velocity boost (juegos que pisan WalkSpeed cada frame, ej. Rivals)
    if hum and hrp and hum.MoveDirection.Magnitude > 0.05 then
        local base = 16 -- walkspeed default roblox
        local mult = math.max(spd / base, 1)
        local dir = hum.MoveDirection
        local vel = hrp.AssemblyLinearVelocity
        -- mantener Y (gravedad/salto), potenciar XZ
        local targetXZ = Vector3.new(dir.X, 0, dir.Z) * spd
        hrp.AssemblyLinearVelocity = Vector3.new(targetXZ.X, vel.Y, targetXZ.Z)
    end
end

local function charHookWalkSpeed()
    if speed_conn_prop then pcall(function() speed_conn_prop:Disconnect() end) speed_conn_prop = nil end
    local ch = LocalPlayer.Character
    local hum = ch and ch:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    speed_conn_prop = hum:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
        if char_cfg.speed_enabled then
            local spd = tonumber(char_cfg.speed) or 22
            if hum.WalkSpeed ~= spd then
                pcall(function() hum.WalkSpeed = spd end)
            end
        end
    end)
end

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(0.3)
    if char_cfg.speed_enabled then
        charHookWalkSpeed()
        charApplySpeed()
    end
end)

local function charStartNoclip()
    if noclip_conn then pcall(function() noclip_conn:Disconnect() end) end
    noclip_conn = RunService.Stepped:Connect(function()
        if not char_cfg.noclip then return end
        local ch = LocalPlayer.Character
        if not ch then return end
        for _, p in ipairs(ch:GetDescendants()) do
            if p:IsA("BasePart") then
                p.CanCollide = false
            end
        end
    end)
end

local function charStartFly()
    local ch = LocalPlayer.Character
    local hrp = ch and ch:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    if fly_bv then pcall(function() fly_bv:Destroy() end) end
    if fly_bg then pcall(function() fly_bg:Destroy() end) end
    fly_bv = Instance.new("BodyVelocity")
    fly_bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
    fly_bv.Velocity = Vector3.zero
    fly_bv.Parent = hrp
    fly_bg = Instance.new("BodyGyro")
    fly_bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    fly_bg.P = 9e4
    fly_bg.Parent = hrp
end

local function charStopFly()
    if fly_bv then pcall(function() fly_bv:Destroy() end); fly_bv = nil end
    if fly_bg then pcall(function() fly_bg:Destroy() end); fly_bg = nil end
end

function charStart()
    if char_conn then pcall(function() char_conn:Disconnect() end) end
    char_conn = RunService.RenderStepped:Connect(function(dt)
        -- speed (WalkSpeed + velocity)
        if char_cfg.speed_enabled then
            charApplySpeed()
        end

        -- fly
        if char_cfg.fly then
            local ch = LocalPlayer.Character
            local hrp = ch and ch:FindFirstChild("HumanoidRootPart")
            local hum = ch and ch:FindFirstChildOfClass("Humanoid")
            if hrp then
                if not fly_bv or not fly_bv.Parent then charStartFly() end
                local cam = workspace.CurrentCamera
                local dir = Vector3.zero
                if UIS:IsKeyDown(Enum.KeyCode.W) then dir = dir + cam.CFrame.LookVector end
                if UIS:IsKeyDown(Enum.KeyCode.S) then dir = dir - cam.CFrame.LookVector end
                if UIS:IsKeyDown(Enum.KeyCode.A) then dir = dir - cam.CFrame.RightVector end
                if UIS:IsKeyDown(Enum.KeyCode.D) then dir = dir + cam.CFrame.RightVector end
                if UIS:IsKeyDown(Enum.KeyCode.Space) then dir = dir + Vector3.new(0, 1, 0) end
                if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then dir = dir - Vector3.new(0, 1, 0) end
                if dir.Magnitude > 0 then dir = dir.Unit * (char_cfg.fly_speed or 50) end
                if fly_bv then fly_bv.Velocity = dir end
                if fly_bg and cam then fly_bg.CFrame = cam.CFrame end
                if hum then hum.PlatformStand = true end
            end
        else
            charStopFly()
            local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
            if hum then hum.PlatformStand = false end
        end

        -- target hover (orbit near enemy)
        if char_cfg.target_hover then
            local myHRP = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            if myHRP then
                local hover_target = getgenv()._Matcha_hover
            if not hover_target or not hover_target.Parent then
                    local best, bd = nil, math.huge
                    for _, plr in ipairs(Players:GetPlayers()) do
                        if MatchaEnemy(plr, true, false) and plr.Character then
                            local h = plr.Character:FindFirstChild("HumanoidRootPart")
                            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
                            if h and hum and hum.Health > 0 then
                                local d = (myHRP.Position - h.Position).Magnitude
                                if d < bd then bd = d; best = h end
                            end
                        end
                    end
                    getgenv()._Matcha_hover = best
                    hover_target = best
                end
                if hover_target and hover_target.Parent then
                    local ang = tick() * (char_cfg.hover_speed or 7)
                    local r = char_cfg.hover_radius or 12
                    local goal = hover_target.Position + Vector3.new(math.cos(ang) * r, 2, math.sin(ang) * r)
                    myHRP.CFrame = myHRP.CFrame:Lerp(CFrame.new(goal, hover_target.Position), math.clamp(dt * 8, 0, 1))
                end
            end
        end

        -- hitbox: lo maneja hbLoop dedicado (no aquí)
    end)

    if char_cfg.noclip then charStartNoclip() end

    -- inf jump
    UIS.JumpRequest:Connect(function()
        if not char_cfg.inf_jump then return end
        local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
    end)
end
function charStop()
    if char_conn then pcall(function() char_conn:Disconnect() end); char_conn = nil end
    if noclip_conn then pcall(function() noclip_conn:Disconnect() end); noclip_conn = nil end
    charStopFly()
    pcall(function()
        local hb = getgenv()._Matcha_hitbox
        if hb and hb._stop then hb._stop() end
    end)
end
char_cfg._start = charStart
char_cfg._stop = charStop

-- ==================== HITBOX EXPANDER (real: partes + CanQuery + silent help) ====================
--[[
  Expande Head/Torso/HRP del enemigo en el CLIENTE.
  CanQuery=true → los raycasts locales (balas) pegan en el cubo grande.
  Visual rosa semitransparente. No CanCollide (no empuja).
]]
local hb_cfg = {
    enabled = false,
    size = 5,
    transparency = 0.65,
    color = Color3.fromRGB(255, 105, 180),
}
getgenv()._Matcha_hitbox = hb_cfg
getgenv()._Matcha_char = getgenv()._Matcha_char or char_cfg
-- mirror flags on char_cfg for UI compat
char_cfg.hitbox_expand = false
char_cfg.hitbox_size = 5

local hb_backup = {} -- [Part] = {Size, Transparency, CanCollide, CanQuery, Massless}
local hb_adorns = {} -- [Model] = BoxHandleAdornment
local hb_conn = nil
local hb_parts_names = {
    "Head", "HumanoidRootPart", "UpperTorso", "LowerTorso", "Torso",
    "LeftUpperArm", "RightUpperArm", "LeftLowerArm", "RightLowerArm",
    "LeftUpperLeg", "RightUpperLeg", "LeftLowerLeg", "RightLowerLeg",
}

local function hbRestorePart(part)
    local b = hb_backup[part]
    if not b or not part or not part.Parent then
        hb_backup[part] = nil
        return
    end
    pcall(function()
        part.Size = b.Size
        part.Transparency = b.Transparency
        part.CanCollide = b.CanCollide
        if b.CanQuery ~= nil then part.CanQuery = b.CanQuery end
        if b.Massless ~= nil then part.Massless = b.Massless end
    end)
    hb_backup[part] = nil
end

local function hbRestoreAll()
    for part, _ in pairs(hb_backup) do
        hbRestorePart(part)
    end
    table.clear(hb_backup)
    for model, ad in pairs(hb_adorns) do
        pcall(function() if ad then ad:Destroy() end end)
        hb_adorns[model] = nil
    end
    table.clear(hb_adorns)
end

local function hbExpandCharacter(char)
    if not char then return end
    local sz = math.clamp(tonumber(hb_cfg.size) or 5, 2, 25)
    local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
    if not root then return end

    for _, name in ipairs(hb_parts_names) do
        local part = char:FindFirstChild(name)
        if part and part:IsA("BasePart") then
            if not hb_backup[part] then
                hb_backup[part] = {
                    Size = part.Size,
                    Transparency = part.Transparency,
                    CanCollide = part.CanCollide,
                    CanQuery = part.CanQuery,
                    Massless = part.Massless,
                }
            end
            -- Head un poco más chico que el torso/hrp
            local s = sz
            if name == "Head" then
                s = math.max(sz * 0.55, 2)
            elseif name:find("Arm") or name:find("Leg") then
                s = math.max(sz * 0.4, 1.5)
            end
            pcall(function()
                part.Size = Vector3.new(s, s, s)
                part.CanCollide = false
                part.CanQuery = true -- CLAVE: raycast de balas pega aquí
                part.Massless = true
                -- HRP visible como "cuadro"
                if name == "HumanoidRootPart" or name == "UpperTorso" or name == "Torso" then
                    part.Transparency = hb_cfg.transparency or 0.65
                    part.Color = hb_cfg.color or Color3.fromRGB(255, 105, 180)
                    part.Material = Enum.Material.ForceField
                else
                    part.Transparency = math.min((hb_cfg.transparency or 0.65) + 0.15, 0.9)
                end
            end)
        end
    end

    -- adorn visual extra (caja rosa)
    if root and not hb_adorns[char] then
        pcall(function()
            local ad = Instance.new("BoxHandleAdornment")
            ad.Name = "MatchaHitbox"
            ad.Adornee = root
            ad.AlwaysOnTop = true
            ad.ZIndex = 5
            ad.Size = Vector3.new(sz, sz, sz)
            ad.Color3 = hb_cfg.color or Color3.fromRGB(255, 105, 180)
            ad.Transparency = 0.55
            ad.Parent = root
            hb_adorns[char] = ad
        end)
    elseif hb_adorns[char] then
        pcall(function()
            hb_adorns[char].Size = Vector3.new(sz, sz, sz)
            hb_adorns[char].Transparency = 0.55
        end)
    end
end

local function hbIsEnemy(plr)
    if not plr or plr == LocalPlayer then return false end
    -- team check suave
    local ok, enemy = pcall(function()
        if MatchaIsEnemy then return MatchaIsEnemy(plr, true, false) end
        if plr.Team and LocalPlayer.Team and plr.Team == LocalPlayer.Team then return false end
        return true
    end)
    return ok and enemy
end

local function hbTick()
    if not hb_cfg.enabled then return end
    local seen = {}
    for _, plr in ipairs(Players:GetPlayers()) do
        if hbIsEnemy(plr) and plr.Character then
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if hum and hum.Health > 0 then
                seen[plr.Character] = true
                hbExpandCharacter(plr.Character)
            end
        end
    end
    -- limpiar adorns de chars muertos/salidos
    for model, ad in pairs(hb_adorns) do
        if not seen[model] or not model.Parent then
            pcall(function() if ad then ad:Destroy() end end)
            hb_adorns[model] = nil
            -- restore parts of that model
            for part, _ in pairs(hb_backup) do
                if part.Parent == model or (part.Parent and part.Parent.Parent == model) then
                    hbRestorePart(part)
                elseif not part.Parent then
                    hb_backup[part] = nil
                end
            end
        end
    end
end

local function hbStart()
    hb_cfg.enabled = true
    char_cfg.hitbox_expand = true
    if hb_conn then pcall(function() hb_conn:Disconnect() end) end
    hb_conn = RunService.Heartbeat:Connect(function()
        pcall(hbTick)
    end)
    print("[Matcha] Hitbox Expander ON · size=", hb_cfg.size)
end

local function hbStop()
    hb_cfg.enabled = false
    char_cfg.hitbox_expand = false
    if hb_conn then pcall(function() hb_conn:Disconnect() end); hb_conn = nil end
    hbRestoreAll()
    print("[Matcha] Hitbox Expander OFF")
end

hb_cfg._start = hbStart
hb_cfg._stop = hbStop
hb_cfg._tick = hbTick

-- Hook raycast: si el ray pasa cerca del hitbox expandido, fuerza hit al Head
-- (refuerzo por si el juego no usa CanQuery de las partes)
task.spawn(function()
    task.wait(1)
    pcall(function()
        local function pointInExpanded(plr, worldPos)
            if not hb_cfg.enabled or not plr or not plr.Character then return false end
            local root = plr.Character:FindFirstChild("HumanoidRootPart")
            if not root then return false end
            local half = (tonumber(hb_cfg.size) or 5) * 0.55
            local lp = root.CFrame:PointToObjectSpace(worldPos)
            return math.abs(lp.X) <= half and math.abs(lp.Y) <= half and math.abs(lp.Z) <= half
        end

        local function closestEnemyAlongRay(origin, dir, dist)
            dir = typeof(dir) == "Vector3" and dir.Unit or Vector3.new(0, 0, -1)
            dist = tonumber(dist) or 1000
            local best, bestHead, bestT = nil, nil, dist
            for _, plr in ipairs(Players:GetPlayers()) do
                if not hbIsEnemy(plr) then continue end
                local char = plr.Character
                if not char then continue end
                local head = char:FindFirstChild("Head")
                local root = char:FindFirstChild("HumanoidRootPart")
                if not root then continue end
                local to = (root.Position - origin)
                local t = to:Dot(dir)
                if t < 0 or t > bestT then continue end
                local closest = origin + dir * t
                local half = (tonumber(hb_cfg.size) or 5) * 0.6
                if (closest - root.Position).Magnitude <= half then
                    bestT = t
                    best = plr
                    bestHead = head or root
                end
            end
            return best, bestHead
        end

        -- si silent ya hookeó, no hace falta; el CanQuery en partes grandes es el hit real
        getgenv()._Matcha_hitbox_rayHelp = closestEnemyAlongRay
    end)
end)


-- auto start character loop (lightweight)
task.defer(charStart)




-- ==================== SKYBOX + RATIO + HITSOUND STATE (from source) ====================
local SkyboxPresets = {
    ["None"] = {SkyboxBk="rbxassetid://14147881792",SkyboxDn="rbxassetid://14147882149",SkyboxFt="rbxassetid://14147882761",SkyboxLf="rbxassetid://14147883091",SkyboxRt="rbxassetid://14147882405",SkyboxUp="rbxassetid://14147881297"},
    Aurora = {SkyboxBk="rbxassetid://116533337330584",SkyboxDn="rbxassetid://80054106187171",SkyboxFt="rbxassetid://94459139270943",SkyboxLf="rbxassetid://116368999680791",SkyboxRt="rbxassetid://125758104196312",SkyboxUp="rbxassetid://107060226443967"},
    Battlerock = {SkyboxBk="rbxassetid://131136284306917",SkyboxDn="rbxassetid://89505977207531",SkyboxFt="rbxassetid://140099243548102",SkyboxLf="rbxassetid://121676169821100",SkyboxRt="rbxassetid://97183886241447",SkyboxUp="rbxassetid://107128620201556"},
    ["Beach Bowl"] = {SkyboxBk="rbxassetid://81804134601271",SkyboxDn="rbxassetid://92395364196932",SkyboxFt="rbxassetid://81804134601271",SkyboxLf="rbxassetid://81804134601271",SkyboxRt="rbxassetid://81804134601271",SkyboxUp="rbxassetid://119089964803065"},
    ["Buoy Base"] = {SkyboxBk="rbxassetid://135017685421888",SkyboxDn="rbxassetid://76610044495625",SkyboxFt="rbxassetid://89675413438577",SkyboxLf="rbxassetid://138307087837279",SkyboxRt="rbxassetid://136723547010707",SkyboxUp="rbxassetid://83404878914838"},
    Clockwork = {SkyboxBk="rbxassetid://86284761193226",SkyboxDn="rbxassetid://111425663631622",SkyboxFt="rbxassetid://115606366886873",SkyboxLf="rbxassetid://127287488325060",SkyboxRt="rbxassetid://126844150113423",SkyboxUp="rbxassetid://74510789204352"},
    ["Dark Matter"] = {SkyboxBk="rbxassetid://97629693450922",SkyboxDn="rbxassetid://97898396690232",SkyboxFt="rbxassetid://134755033418084",SkyboxLf="rbxassetid://118219143707956",SkyboxRt="rbxassetid://114940065588775",SkyboxUp="rbxassetid://95430908943263"},
    ["Flash Black"] = {SkyboxBk="rbxassetid://78426835654353",SkyboxDn="rbxassetid://6213218651",SkyboxFt="rbxassetid://71970982976722",SkyboxLf="rbxassetid://78426835654353",SkyboxRt="rbxassetid://78426835654353",SkyboxUp="rbxassetid://138004866371717"},
    Ghostly = {SkyboxBk="rbxassetid://111506743048183",SkyboxDn="rbxassetid://86198196348228",SkyboxFt="rbxassetid://86265514167302",SkyboxLf="rbxassetid://100257959405445",SkyboxRt="rbxassetid://71935101953120",SkyboxUp="rbxassetid://132011089223498"},
    ["Good Egg"] = {SkyboxBk="rbxassetid://94681381933012",SkyboxDn="rbxassetid://75843838469806",SkyboxFt="rbxassetid://97891957473259",SkyboxLf="rbxassetid://102971518965494",SkyboxRt="rbxassetid://94588890960775",SkyboxUp="rbxassetid://127368871569815"},
    ["Melty Molten"] = {SkyboxBk="rbxassetid://131463907527649",SkyboxDn="rbxassetid://116154164311420",SkyboxFt="rbxassetid://113077689016278",SkyboxLf="rbxassetid://79984367513909",SkyboxRt="rbxassetid://82395195737484",SkyboxUp="rbxassetid://117530106700350"},
    Shiverburn = {SkyboxBk="rbxassetid://113636030839991",SkyboxDn="rbxassetid://118027268179499",SkyboxFt="rbxassetid://76405010847029",SkyboxLf="rbxassetid://112044353352688",SkyboxRt="rbxassetid://121078604572355",SkyboxUp="rbxassetid://132486295432727"},
    ["Spin Dig"] = {SkyboxBk="rbxassetid://124211111366754",SkyboxDn="rbxassetid://120491795220431",SkyboxFt="rbxassetid://130119279111055",SkyboxLf="rbxassetid://70742671331562",SkyboxRt="rbxassetid://76516826791940",SkyboxUp="rbxassetid://100229310567751"},
    ["Sweet Mystery"] = {SkyboxBk="rbxassetid://107264897520277",SkyboxDn="rbxassetid://135637946277638",SkyboxFt="rbxassetid://135705252786048",SkyboxLf="rbxassetid://119667604517747",SkyboxRt="rbxassetid://75904303027092",SkyboxUp="rbxassetid://97011146822716"},
    ["Terrace Dome"] = {SkyboxBk="rbxassetid://98684100016510",SkyboxDn="rbxassetid://108354040356521",SkyboxFt="rbxassetid://95723629635852",SkyboxLf="rbxassetid://106269064939837",SkyboxRt="rbxassetid://136234415079744",SkyboxUp="rbxassetid://75385962780878"},
}
local SkyboxNames = {}
for name in pairs(SkyboxPresets) do SkyboxNames[#SkyboxNames+1] = name end
table.sort(SkyboxNames)

local skyboxData = {enabled=false, selected="None", sky_obj=nil, auto_rotate=false, rotate_speed=10, star_count=3000, sun_size=20, moon_size=11}
local skyboxConn = nil

function EnsureSky()
    if skyboxData.sky_obj and skyboxData.sky_obj.Parent then return skyboxData.sky_obj end
    local original = Lighting:FindFirstChildOfClass("Sky")
    if original then
        skyboxData.sky_obj = original:Clone()
        skyboxData.sky_obj.Parent = original.Parent
        original:Destroy()
    else
        skyboxData.sky_obj = Instance.new("Sky")
        skyboxData.sky_obj.Parent = Lighting
    end
    return skyboxData.sky_obj
end

function ApplySkybox()
    if not skyboxData.enabled then return end
    local sky = EnsureSky()
    if not sky then return end
    local tex = SkyboxPresets[skyboxData.selected] or SkyboxPresets["None"]
    if not tex then return end
    sky.SkyboxBk = tex.SkyboxBk
    sky.SkyboxDn = tex.SkyboxDn
    sky.SkyboxFt = tex.SkyboxFt
    sky.SkyboxLf = tex.SkyboxLf
    sky.SkyboxRt = tex.SkyboxRt
    sky.SkyboxUp = tex.SkyboxUp
    sky.SunAngularSize = skyboxData.sun_size
    sky.MoonAngularSize = skyboxData.moon_size
    sky.StarCount = skyboxData.star_count
end

states.ratio_state = states.ratio_state or {enabled = false, width = 100, height = 100, fov_boost = 0}
states.hitsound_state = states.hitsound_state or {enabled = false, sound_value = "None", sound_asset_id = nil, sound_volume = 50, sound_pitch = 10}

-- ==================== STRETCHED RES (source .gg/scripters) ====================
getgenv().Resolution = getgenv().Resolution or {
    [".gg/scripters"] = 0.81,
}
local stretch_cfg = {
    enabled = false,
    value = 0.81,
}
getgenv()._Matcha_stretch = stretch_cfg

local stretch_conn = nil
local function stretchStop()
    stretch_cfg.enabled = false
    if stretch_conn then
        pcall(function() stretch_conn:Disconnect() end)
        stretch_conn = nil
    end
    -- permite re-conectar si se vuelve a activar
    getgenv().gg_scripters = nil
    print("[Matcha] Stretched OFF")
end

local function stretchStart()
    stretchStop()
    stretch_cfg.enabled = true
    local v = tonumber(stretch_cfg.value) or 0.81
    getgenv().Resolution = getgenv().Resolution or {}
    getgenv().Resolution[".gg/scripters"] = v
    getgenv().gg_scripters = nil -- fuerza nuevo connect
    stretch_conn = RunService.RenderStepped:Connect(function()
        if not stretch_cfg.enabled then return end
        local cam = workspace.CurrentCamera
        if not cam then return end
        local res = tonumber(getgenv().Resolution and getgenv().Resolution[".gg/scripters"]) or stretch_cfg.value or 0.81
        pcall(function()
            cam.CFrame = cam.CFrame * CFrame.new(0, 0, 0, 1, 0, 0, 0, res, 0, 0, 0, 1)
        end)
    end)
    getgenv().gg_scripters = "MatchaStretch"
    print("[Matcha] Stretched ON ·", v)
end

stretch_cfg._start = stretchStart
stretch_cfg._stop = stretchStop

-- legacy updateRatio → redirige a stretch si alguien lo llama
function updateRatio()
    -- no-op; stretched usa RenderStepped propio
end



-- ==================== UNIVERSAL AUTO CLICKER ====================
getgenv()._Vnix_clicker = getgenv()._Vnix_clicker or {
    enabled = false,
    cps = 12,
    mode = "Left", -- Left / Right / Both
    only_when_focused = true,
    toggle_key = "E", -- tecla para activar/desactivar
}
local clicker_conn = nil
local function clickerStop()
    getgenv()._Vnix_clicker.enabled = false
    if clicker_conn then pcall(function() clicker_conn:Disconnect() end) clicker_conn = nil end
end
local function clickerStart()
    local c = getgenv()._Vnix_clicker
    c.enabled = true
    if clicker_conn then pcall(function() clicker_conn:Disconnect() end) end
    local acc = 0
    clicker_conn = RunService.Heartbeat:Connect(function(dt)
        if not c.enabled then return end
        if c.only_when_focused then
            local ok = true
            pcall(function()
                if UserInputService.WindowFocused ~= nil and not UserInputService.WindowFocused then ok = false end
            end)
            if not ok then return end
        end
        local cps = math.clamp(tonumber(c.cps) or 10, 1, 50)
        acc += dt
        local interval = 1 / cps
        while acc >= interval do
            acc -= interval
            pcall(function()
                local vim = game:GetService("VirtualInputManager")
                if c.mode == "Right" or c.mode == "Both" then
                    vim:SendMouseButtonEvent(0, 0, 1, true, game, 0)
                    vim:SendMouseButtonEvent(0, 0, 1, false, game, 0)
                end
                if c.mode == "Left" or c.mode == "Both" then
                    vim:SendMouseButtonEvent(0, 0, 0, true, game, 0)
                    vim:SendMouseButtonEvent(0, 0, 0, false, game, 0)
                end
            end)
            pcall(function()
                local vu = game:GetService("VirtualUser")
                if c.mode ~= "Right" then
                    vu:Button1Down(Vector2.new(0,0))
                    vu:Button1Up(Vector2.new(0,0))
                end
            end)
        end
    end)
end
getgenv()._Vnix_clicker._start = clickerStart
getgenv()._Vnix_clicker._stop = clickerStop

-- tecla toggle del clicker
UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    local c = getgenv()._Vnix_clicker
    if not c then return end
    local keyName = tostring(c.toggle_key or "E")
    local kc = Enum.KeyCode[keyName]
    if not kc then return end
    if input.KeyCode == kc then
        if c.enabled then
            clickerStop()
            pcall(function() Notify("Clicker", "OFF (" .. keyName .. ")") end)
        else
            clickerStart()
            pcall(function() Notify("Clicker", "ON (" .. keyName .. ")") end)
        end
    end
end)

-- ==================== UNIVERSAL AIMBOT (shako-style: hold RMB + smooth) ====================
do
    local ac = getgenv()._Vnix_aim or {}
    ac.enabled = ac.enabled or false
    ac.team_check = ac.team_check ~= false
    ac.fov = tonumber(ac.fov) or 180
    ac.smooth = tonumber(ac.smooth) or 0.35
    ac.hitpart = ac.hitpart or "Head"
    ac.wall_check = ac.wall_check == true
    ac.show_fov = ac.show_fov ~= false
    ac.fov_color = ac.fov_color or Color3.fromRGB(255, 105, 180)
    ac.fov_fill = ac.fov_fill ~= false
    ac.fov_fill_t = tonumber(ac.fov_fill_t) or 0.85
    ac.holding = false
    getgenv()._Vnix_aim = ac

    local fovOut, fovFill
    pcall(function()
        fovOut = Drawing.new("Circle")
        fovOut.Thickness = 1.5
        fovOut.NumSides = 64
        fovOut.Filled = false
        fovOut.Visible = false
        fovFill = Drawing.new("Circle")
        fovFill.Thickness = 1
        fovFill.NumSides = 64
        fovFill.Filled = true
        fovFill.Visible = false
    end)

    local function isEnemyU(plr)
        if not plr or plr == LocalPlayer then return false end
        if not ac.team_check then return true end
        if LocalPlayer.Team and plr.Team and LocalPlayer.Team == plr.Team then return false end
        local a = LocalPlayer:GetAttribute("Team") or LocalPlayer:GetAttribute("TeamID")
        local b = plr:GetAttribute("Team") or plr:GetAttribute("TeamID")
        if a ~= nil and b ~= nil then return a ~= b end
        return true
    end

    local function wallOk(from, to, char)
        if not ac.wall_check then return true end
        local p = RaycastParams.new()
        p.FilterType = Enum.RaycastFilterType.Exclude
        p.FilterDescendantsInstances = { LocalPlayer.Character, char, workspace.CurrentCamera }
        return workspace:Raycast(from, to - from, p) == nil
    end

    local function closest()
        local cam = workspace.CurrentCamera
        if not cam then return nil end
        local center = cam.ViewportSize / 2
        local best, bestD = nil, ac.fov
        for _, plr in ipairs(Players:GetPlayers()) do
            if not isEnemyU(plr) then continue end
            local char = plr.Character
            if not char then continue end
            local hum = char:FindFirstChildOfClass("Humanoid")
            local part = char:FindFirstChild(ac.hitpart) or char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
            if not hum or not part or hum.Health <= 0 then continue end
            local sp, on = cam:WorldToViewportPoint(part.Position)
            if not on or sp.Z <= 0 then continue end
            local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
            if d < bestD and wallOk(cam.CFrame.Position, part.Position, char) then
                bestD, best = d, part
            end
        end
        return best
    end

    local rot = 0
    local aimConn, inConn, outConn
    local function aimStop()
        ac.enabled = false
        if aimConn then aimConn:Disconnect() aimConn = nil end
        if fovOut then fovOut.Visible = false end
        if fovFill then fovFill.Visible = false end
    end
    local function aimStart()
        ac.enabled = true
        if aimConn then aimConn:Disconnect() end
        aimConn = RunService.RenderStepped:Connect(function(dt)
            local cam = workspace.CurrentCamera
            if not cam then return end
            local center = cam.ViewportSize / 2
            rot = (rot + dt * 40) % 360
            if ac.show_fov and fovOut then
                fovOut.Position = center
                fovOut.Radius = ac.fov
                fovOut.Color = ac.fov_color
                fovOut.Visible = true
                if fovFill then
                    fovFill.Position = center
                    fovFill.Radius = ac.fov
                    local h = (tick() * 0.15) % 1
                    local pulse = 0.5 + 0.5 * math.sin(tick() * 3)
                    fovFill.Color = ac.fov_color
                    fovFill.Transparency = math.clamp(ac.fov_fill_t + (1 - pulse) * 0.1, 0.5, 0.95)
                    fovFill.Visible = ac.fov_fill
                end
            elseif fovOut then
                fovOut.Visible = false
                if fovFill then fovFill.Visible = false end
            end
            if not ac.enabled then return end
            local holding = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)
            if not holding then return end
            local part = closest()
            if not part then return end
            local smooth = math.clamp(tonumber(ac.smooth) or 0.35, 0.05, 1)
            local goal = CFrame.lookAt(cam.CFrame.Position, part.Position)
            cam.CFrame = cam.CFrame:Lerp(goal, smooth)
        end)
    end
    ac._start = aimStart
    ac._stop = aimStop
    -- override Matcha aim if exists
    getgenv()._Vnix_aimbot = ac
    if getgenv()._Vnix_aim then
        for k,v in pairs(ac) do getgenv()._Vnix_aim[k] = v end
        getgenv()._Vnix_aim._start = aimStart
        getgenv()._Vnix_aim._stop = aimStop
    end
end

-- ==================== UNIVERSAL SILENT (Classic / Mouse) ====================
do
    local sc = getgenv()._Vnix_silent or {}
    sc.enabled = false
    sc.team_check = sc.team_check ~= false
    sc.fov = tonumber(sc.fov) or 250
    sc.hitpart = sc.hitpart or "Head"
    sc.show_fov = sc.show_fov ~= false
    sc.fov_color = sc.fov_color or Color3.fromRGB(255, 105, 180)
    sc.fov_fill = false -- OFF por defecto
    sc.fov_fill_t = 0.85
    sc.mode = sc.mode or "Classic" -- Classic | Mouse
    sc.wall_check = false
    getgenv()._Vnix_silent = sc

    -- sync with Matcha silent_cfg
    if silent_cfg then
        silent_cfg.fov_fill = false
        silent_cfg.mode = silent_cfg.mode or "Classic"
    end

    local circle, fill
    pcall(function()
        circle = Drawing.new("Circle")
        circle.Thickness = 1.4
        circle.NumSides = 64
        circle.Filled = false
        circle.Visible = false
        fill = Drawing.new("Circle")
        fill.Filled = true
        fill.NumSides = 64
        fill.Visible = false
    end)

    local function enemy(plr)
        if not plr or plr == LocalPlayer then return false end
        if silent_cfg and silent_cfg.force_ffa then return true end
        if not sc.team_check then return true end
        if LocalPlayer.Team and plr.Team and LocalPlayer.Team == plr.Team then return false end
        return true
    end

    local function getCenter()
        local cam = workspace.CurrentCamera
        if not cam then return Vector2.zero end
        local mode = (silent_cfg and silent_cfg.mode) or sc.mode or "Classic"
        if mode == "Mouse" then
            local m = UserInputService:GetMouseLocation()
            return Vector2.new(m.X, m.Y)
        end
        return cam.ViewportSize / 2
    end

    local function target()
        local cam = workspace.CurrentCamera
        if not cam then return nil end
        local center = getCenter()
        local fov = tonumber((silent_cfg and silent_cfg.fov) or sc.fov) or 250
        local hit = (silent_cfg and silent_cfg.hitpart) or sc.hitpart or "Head"
        local best, bd = nil, fov
        for _, plr in ipairs(Players:GetPlayers()) do
            if not enemy(plr) then continue end
            local char = plr.Character
            if not char then continue end
            local hum = char:FindFirstChildOfClass("Humanoid")
            local part = char:FindFirstChild(hit) or char:FindFirstChild("Head")
            if not hum or not part or hum.Health <= 0 then continue end
            local sp, on = cam:WorldToViewportPoint(part.Position)
            if sp.Z <= 0 then continue end
            -- Classic: requiere on-screen; Mouse: cerca del cursor
            local md = (silent_cfg and silent_cfg.mode) or sc.mode or "Classic"
            if md ~= "Mouse" and not on then continue end
            local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
            if d < bd then bd, best = d, part end
        end
        return best
    end

    local conn
    local function silentStopU()
        sc.enabled = false
        if silent_cfg then silent_cfg.enabled = false end
        if conn then conn:Disconnect() conn = nil end
        if circle then circle.Visible = false end
        if fill then fill.Visible = false end
        pcall(function() if silent_circle then silent_circle.Visible = false end end)
        pcall(function() if silent_fill then silent_fill.Visible = false end end)
    end

    local function silentStartU()
        sc.enabled = true
        if silent_cfg then silent_cfg.enabled = true end
        if conn then conn:Disconnect() end
        -- activar hooks rivals (Classic = silent real de balas)
        pcall(function()
            if type(silentStart) == "function" then
                -- no recursivo: original rivals silentStart
            end
        end)
        pcall(function()
            if silent_cfg then
                silent_cfg.enabled = true
                getgenv().Rivals = getgenv().Rivals or {}
                getgenv().Rivals.Silent = getgenv().Rivals.Silent or {}
                getgenv().Rivals.Silent.enabled = true
            end
        end)

        conn = RunService.RenderStepped:Connect(function()
            local cam = workspace.CurrentCamera
            if not cam then return end
            local center = getCenter()
            local fov = tonumber((silent_cfg and silent_cfg.fov) or sc.fov) or 250
            local col = (silent_cfg and silent_cfg.fov_color) or sc.fov_color
            local show = sc.show_fov ~= false and ((silent_cfg and silent_cfg.show_fov) ~= false)
            local fillOn = (silent_cfg and silent_cfg.fov_fill) or sc.fov_fill
            local mode = (silent_cfg and silent_cfg.mode) or sc.mode or "Classic"

            if show and sc.enabled and circle then
                circle.Position = center
                circle.Radius = fov
                circle.Color = col
                circle.Filled = false
                circle.Visible = true
                if fill then
                    if fillOn then
                        fill.Position = center
                        fill.Radius = fov
                        fill.Color = col
                        local pulse = 0.55 + 0.45 * math.sin(tick() * 3.2)
                        fill.Transparency = math.clamp(0.82 + (1 - pulse) * 0.08, 0.7, 0.95)
                        fill.Visible = true
                    else
                        fill.Visible = false
                    end
                end
            elseif circle then
                circle.Visible = false
                if fill then fill.Visible = false end
            end

            if not sc.enabled then return end

            local t = target()
            if silent_cfg then
                silent_target_head = t
            end

            -- Mouse mode: FOV sigue cursor + soft assist (opcional suave)
            if mode == "Mouse" and t then
                pcall(function()
                    local sp = cam:WorldToViewportPoint(t.Position)
                    local mouse = UserInputService:GetMouseLocation()
                    local dx, dy = sp.X - mouse.X, sp.Y - mouse.Y
                    if math.abs(dx) > 2 or math.abs(dy) > 2 then
                        local vim = game:GetService("VirtualInputManager")
                        vim:SendMouseMoveEvent(mouse.X + dx * 0.25, mouse.Y + dy * 0.25, game)
                    end
                end)
            end
            -- Classic: NO mueve mouse ni camara — solo silent_target_head + hooks de balas
        end)
    end

    sc._start = silentStartU
    sc._stop = silentStopU
    pcall(function()
        if silent_cfg then
            silent_cfg._start = silentStartU
            silent_cfg._stop = silentStopU
        end
    end)
end


-- ==================== BLOX STRIKE COMBAT (Rage / FastFire / Spin / Desync) ====================
getgenv()._BS = getgenv()._BS or {
    rage = false,
    rage_team = true,
    rage_fov = 9999,
    rage_rate = 0.05,
    rage_tp = true, -- server cerca del enemy
    desync = false,
    desync_y = 3,
    spin = false,
    spin_speed = 25,
    fast_fire = false,
    fire_rate = 0.02,
}
local BS = getgenv()._BS

local function bsGetChar(plr)
    if ESP_GetCharacter then return ESP_GetCharacter(plr) end
    return plr and plr.Character
end

local function bsEnemy(plr)
    if not plr or plr == LocalPlayer then return false end
    if not BS.rage_team then return true end
    if ESP_IsEnemy then return ESP_IsEnemy(plr) end
    local a = LocalPlayer:GetAttribute("Team") or LocalPlayer:GetAttribute("TeamID")
    local b = plr:GetAttribute("Team") or plr:GetAttribute("TeamID")
    if a and b then return tostring(a) ~= tostring(b) end
    if LocalPlayer.Team and plr.Team then return LocalPlayer.Team ~= plr.Team end
    return true
end

local function bsHealth(char)
    if ESP_GetHealth then
        local h = ESP_GetHealth(char, char and char:FindFirstChildOfClass("Humanoid"))
        return h
    end
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health or 0
end

local function bsFindShootRemote()
    local RS = game:GetService("ReplicatedStorage")
    local names = {"ShootGun", "Shoot", "Fire", "WeaponShoot", "GunShoot", "Attack"}
    local remotes = RS:FindFirstChild("Remotes")
    if remotes then
        for _, n in ipairs(names) do
            local r = remotes:FindFirstChild(n)
            if r and (r:IsA("RemoteEvent") or r:IsA("UnreliableRemoteEvent")) then return r end
        end
    end
    for _, d in ipairs(RS:GetDescendants()) do
        if (d:IsA("RemoteEvent") or d:IsA("UnreliableRemoteEvent")) then
            local n = d.Name:lower()
            if n:find("shoot") or n:find("fire") or n:find("bullet") then
                return d
            end
        end
    end
    return nil
end

local shootRemote = nil
task.spawn(function()
    for _ = 1, 30 do
        shootRemote = bsFindShootRemote()
        if shootRemote then break end
        task.wait(0.5)
    end
end)

local function bsClosestHead()
    local cam = workspace.CurrentCamera
    local best, bestD, bestPart = nil, BS.rage_fov or 9999, nil
    local myChar = bsGetChar(LocalPlayer)
    local myRoot = myChar and (myChar:FindFirstChild("HumanoidRootPart") or myChar.PrimaryPart)
    for _, plr in ipairs(Players:GetPlayers()) do
        if not bsEnemy(plr) then continue end
        local char = bsGetChar(plr)
        if not char then continue end
        if bsHealth(char) <= 0 then continue end
        local head = char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart
        if not head then continue end
        local d = myRoot and (myRoot.Position - head.Position).Magnitude or 0
        if d < bestD then
            bestD, best, bestPart = d, plr, head
        end
    end
    return best, bestPart
end

local function bsClick()
    pcall(function()
        local vim = game:GetService("VirtualInputManager")
        vim:SendMouseButtonEvent(0, 0, 0, true, game, 0)
        vim:SendMouseButtonEvent(0, 0, 0, false, game, 0)
    end)
    pcall(function()
        local vu = game:GetService("VirtualUser")
        vu:Button1Down(Vector2.new(0, 0))
        vu:Button1Up(Vector2.new(0, 0))
    end)
    pcall(function()
        local char = LocalPlayer.Character
        local tool = char and char:FindFirstChildOfClass("Tool")
        if tool then tool:Activate() end
    end)
end

local function bsFireAt(head)
    if not head then return end
    local cam = workspace.CurrentCamera
    local origin = cam and cam.CFrame.Position or head.Position + Vector3.new(0, 2, 0)
    local dir = (head.Position - origin)
    if shootRemote then
        pcall(function() shootRemote:FireServer(origin, dir) end)
        pcall(function() shootRemote:FireServer(origin, dir.Unit) end)
        pcall(function() shootRemote:FireServer(head.Position) end)
    end
    bsClick()
end

-- RAGEBOT
local rageConn, lastRage = nil, 0
local rageClientCF = nil
local function bsRageStop()
    BS.rage = false
    if rageConn then rageConn:Disconnect() rageConn = nil end
    pcall(function() RunService:UnbindFromRenderStep("__bs_rage_vis") end)
end
local function bsRageStart()
    BS.rage = true
    if rageConn then rageConn:Disconnect() end
    local h0 = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if h0 then rageClientCF = h0.CFrame end

    RunService:BindToRenderStep("__bs_rage_vis", Enum.RenderPriority.Camera.Value - 1, function()
        if not BS.rage or not BS.desync then return end
        local h = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if h and rageClientCF then
            h.CFrame = rageClientCF
            h.AssemblyLinearVelocity = Vector3.zero
        end
    end)

    rageConn = RunService.Heartbeat:Connect(function()
        if not BS.rage then return end
        local plr, head = bsClosestHead()
        if not head then return end
        local my = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not my then return end

        if not rageClientCF or (rageClientCF.Position - my.Position).Magnitude < 200 then
            if my.Position.Y < 150 then rageClientCF = my.CFrame end
        end

        -- server pos cerca de la head (si desync/tp)
        if BS.rage_tp or BS.desync then
            local goal = head.Position + Vector3.new(0, tonumber(BS.desync_y) or 2.5, 0)
            my.CFrame = CFrame.lookAt(goal, head.Position)
            my.AssemblyLinearVelocity = Vector3.zero
        end

        if (tick() - lastRage) < (tonumber(BS.rage_rate) or 0.05) then return end
        lastRage = tick()
        bsFireAt(head)
    end)
end

-- FAST FIRE (multi-method Blox Strike)
local ffConn, lastFF = nil, 0
local ffPatched = {}
local function bsPatchGunValues(tool)
    if not tool then return end
    pcall(function()
        for _, v in ipairs(tool:GetDescendants()) do
            local n = (v.Name or ""):lower()
            if v:IsA("NumberValue") or v:IsA("IntValue") then
                if n:find("cool") or n:find("delay") or n:find("rate") or n:find("rpm")
                    or n:find("interval") or n:find("firerate") or n:find("fire_rate")
                    or n:find("shoot") or n:find("wait") then
                    pcall(function() v.Value = 0.001 end)
                end
            elseif v:IsA("BoolValue") and (n:find("canfire") or n:find("ready")) then
                pcall(function() v.Value = true end)
            end
        end
        -- attributes
        for _, attr in ipairs({"Cooldown", "FireRate", "ShootCooldown", "Delay", "RPM", "NextShot"}) do
            pcall(function()
                if tool:GetAttribute(attr) ~= nil then
                    tool:SetAttribute(attr, 0)
                end
            end)
        end
    end)
end

local function bsPatchGC()
    pcall(function()
        if not getgc then return end
        for _, obj in ipairs(getgc(true)) do
            if type(obj) == "table" then
                local keys = {"FireRate", "ShootCooldown", "Cooldown", "DelayBetweenShots", "RPM", "Firerate", "fireRate"}
                for _, k in ipairs(keys) do
                    if rawget(obj, k) ~= nil and type(obj[k]) == "number" and obj[k] > 0.02 then
                        pcall(function() obj[k] = 0.001 end)
                    end
                end
            end
        end
    end)
end

local function bsFireConnections(tool)
    if not tool then return end
    pcall(function()
        if not getconnections then return end
        for _, conn in ipairs(getconnections(tool.Activated)) do
            pcall(function()
                if conn.Function then conn.Function() end
                if conn.Fire then conn:Fire() end
            end)
        end
    end)
end

local function bsFastFireStop()
    BS.fast_fire = false
    if ffConn then ffConn:Disconnect() ffConn = nil end
end

local function bsFastFireStart()
    BS.fast_fire = true
    if ffConn then ffConn:Disconnect() end
    lastFF = 0
    pcall(bsPatchGC)
    local char0 = LocalPlayer.Character
    if char0 then
        local t0 = char0:FindFirstChildOfClass("Tool")
        if t0 then bsPatchGunValues(t0) end
    end

    -- SOLO dispara si mantienes click izquierdo.
    -- NUNCA spam VirtualUser/mouse1click en (0,0) → eso bloqueaba la UI y el loadout.
    ffConn = RunService.RenderStepped:Connect(function()
        if not BS.fast_fire then return end
        -- no interferir con menú Matcha abierto
        if UI and UI.Open then return end
        local holding = false
        pcall(function()
            holding = UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
        end)
        if not holding then return end

        local rate = math.clamp(tonumber(BS.fire_rate) or 0.05, 0.03, 0.2)
        if (tick() - lastFF) < rate then return end
        lastFF = tick()

        local char = LocalPlayer.Character
        if not char then return end
        local tool = char:FindFirstChildOfClass("Tool")
        if not tool then
            for _, ch in ipairs(char:GetChildren()) do
                if ch:IsA("Tool") then tool = ch break end
            end
        end
        if tool then
            bsPatchGunValues(tool)
            pcall(function() tool:Activate() end)
            bsFireConnections(tool)
        end
        -- sin VirtualUser / VIM / mouse1click (rompen GUI del juego)
    end)

    task.spawn(function()
        while BS.fast_fire do
            pcall(bsPatchGC)
            local ch = LocalPlayer.Character
            local t = ch and ch:FindFirstChildOfClass("Tool")
            if t then bsPatchGunValues(t) end
            task.wait(2)
        end
    end)
end

-- SPIN BOT + DESYNC visual
local spinConn
local spinClientCF = nil
local function bsSpinStop()
    BS.spin = false
    if spinConn then spinConn:Disconnect() spinConn = nil end
    pcall(function() RunService:UnbindFromRenderStep("__bs_spin_vis") end)
end
local function bsSpinStart()
    BS.spin = true
    if spinConn then spinConn:Disconnect() end
    local h0 = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if h0 then spinClientCF = h0.CFrame end

    RunService:BindToRenderStep("__bs_spin_vis", Enum.RenderPriority.Camera.Value - 1, function()
        if not BS.spin and not BS.desync then return end
        local h = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if h and spinClientCF and not BS.rage then
            -- mantener vista local estable
            local pos = spinClientCF.Position
            if UserInputService then
                local move = Vector3.zero
                local cam = workspace.CurrentCamera
                if cam then
                    local look = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
                    local right = Vector3.new(cam.CFrame.RightVector.X, 0, cam.CFrame.RightVector.Z)
                    if look.Magnitude > 0.1 then look = look.Unit end
                    if right.Magnitude > 0.1 then right = right.Unit end
                    if UserInputService:IsKeyDown(Enum.KeyCode.W) then move += look end
                    if UserInputService:IsKeyDown(Enum.KeyCode.S) then move -= look end
                    if UserInputService:IsKeyDown(Enum.KeyCode.A) then move -= right end
                    if UserInputService:IsKeyDown(Enum.KeyCode.D) then move += right end
                    if move.Magnitude > 0 then pos = pos + move.Unit * 0.45 end
                    local lv = cam.CFrame.LookVector
                    spinClientCF = CFrame.new(pos) * CFrame.Angles(0, math.atan2(-lv.X, -lv.Z), 0)
                end
            end
            h.CFrame = spinClientCF
            -- no zero velocity (bloqueaba movimiento y menús)
        end
    end)

    spinConn = RunService.Heartbeat:Connect(function()
        if not BS.spin then return end
        local h = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not h then return end
        if not spinClientCF then spinClientCF = h.CFrame end
        local spd = math.rad(tonumber(BS.spin_speed) or 25)
        local ang = tick() * spd * 60
        -- solo yaw; conserva velocidad horizontal para poder moverte / abrir menús
        local vel = h.AssemblyLinearVelocity
        h.CFrame = CFrame.new(h.Position) * CFrame.Angles(0, ang, 0)
        h.AssemblyLinearVelocity = Vector3.new(vel.X, vel.Y, vel.Z)
    end)
end

local function bsDesyncToggle(on)
    BS.desync = on and true or false
    if on then
        local h = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if h then spinClientCF = h.CFrame; rageClientCF = h.CFrame end
    end
end

getgenv()._BS._rageStart = bsRageStart
getgenv()._BS._rageStop = bsRageStop
getgenv()._BS._ffStart = bsFastFireStart
getgenv()._BS._ffStop = bsFastFireStop
getgenv()._BS._spinStart = bsSpinStart
getgenv()._BS._spinStop = bsSpinStop
getgenv()._BS._desync = bsDesyncToggle

function __VnixBuildUI()
print("[Vnix.lol] __VnixBuildUI start")
-- ==================== CUSTOM UI (Linoria-style) ====================
if getgenv().VnixUI_Loaded then
    pcall(function()
        if type(getgenv().VnixUI_Destroy) == "function" then
            getgenv().VnixUI_Destroy()
        end
    end)
end
getgenv().VnixUI_Loaded = true
print("[Vnix.lol] UI init...")

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- Linoria-like palette
local Theme = {
    Background   = Color3.fromRGB(10, 10, 12),
    WindowBg     = Color3.fromRGB(14, 14, 16),
    Inline       = Color3.fromRGB(18, 18, 20),
    Element      = Color3.fromRGB(24, 24, 26),
    ElementHover = Color3.fromRGB(34, 34, 38),
    Accent       = Color3.fromRGB(255, 105, 180),
    AccentDim    = Color3.fromRGB(200, 70, 150),
    Outline      = Color3.fromRGB(0, 0, 0),
    OutlineSoft  = Color3.fromRGB(22, 22, 24),
    Text         = Color3.fromRGB(255, 255, 255),
    TextDark     = Color3.fromRGB(200, 200, 205),
    Risky        = Color3.fromRGB(255, 80, 100),
    Enabled      = Color3.fromRGB(255, 255, 255),
    TabActive    = Color3.fromRGB(24, 24, 26),
    TabIdle      = Color3.fromRGB(14, 14, 16),
}




local UI_W, UI_H = 560, 670
local TOGGLE_KEY = Enum.KeyCode.RightShift

local UI = {
    Open = false,
    Unloaded = false,
    Tabs = {},
    CurrentTab = nil,
    Connections = {},
    Strokes = {},
    Settings = {
        Snow = true,
        Darken = true,
        BorderTransparency = 0,
        BorderColor = Color3.fromRGB(0, 0, 0),
    },
}

local OriginalLighting = {
    Ambient = Lighting.Ambient,
    OutdoorAmbient = Lighting.OutdoorAmbient,
    Brightness = Lighting.Brightness,
    ExposureCompensation = Lighting.ExposureCompensation,
}

local function tween(obj, props, t, style)
    local tw = TweenService:Create(obj, TweenInfo.new(t or 0.15, style or Enum.EasingStyle.Quad, Enum.EasingDirection.Out), props)
    tw:Play()
    return tw
end

local function corner(p, r)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, r or 0) -- Linoria is mostly sharp; slight round on window only
    c.Parent = p
    return c
end

local function stroke(p, col, th, tr)
    local s = Instance.new("UIStroke")
    s.Color = col or UI.Settings.BorderColor or Theme.Outline
    s.Thickness = th or 1
    local base = tr or 0
    s.Transparency = math.clamp((UI.Settings.BorderTransparency or 0) + base, 0, 1)
    s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    s.Parent = p
    s:SetAttribute("VnixBaseT", base)
    table.insert(UI.Strokes, s)
    return s
end

local function applyBorderStyle()
    local col = UI.Settings.BorderColor or Theme.Outline
    local addT = tonumber(UI.Settings.BorderTransparency) or 0
    for _, s in ipairs(UI.Strokes) do
        if s and s.Parent then
            pcall(function()
                s.Color = col
                s.Transparency = math.clamp((s:GetAttribute("VnixBaseT") or 0) + addT, 0, 1)
            end)
        end
    end
end

local function bind(name, signal, fn)
    if UI.Connections[name] then UI.Connections[name]:Disconnect() end
    UI.Connections[name] = signal:Connect(fn)
end

local function getParent()
    local ok, hui = pcall(function() return gethui and gethui() end)
    if ok and hui then return hui end
    return PlayerGui
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "VnixLol_Linoria"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.DisplayOrder = 999
ScreenGui.IgnoreGuiInset = true
ScreenGui.Parent = getParent()

-- Backdrop
local Backdrop = Instance.new("Frame")
Backdrop.Size = UDim2.fromScale(1, 1)
Backdrop.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
Backdrop.BackgroundTransparency = 1
Backdrop.BorderSizePixel = 0
Backdrop.ZIndex = 1
Backdrop.Visible = false
Backdrop.Parent = ScreenGui

local BlurFrame = Instance.new("Frame")
BlurFrame.Size = UDim2.fromScale(1, 1)
BlurFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
BlurFrame.BackgroundTransparency = 1
BlurFrame.BorderSizePixel = 0
BlurFrame.ZIndex = 1
BlurFrame.Parent = Backdrop

local LightingBlur = Instance.new("BlurEffect")
LightingBlur.Size = 0
LightingBlur.Enabled = false
LightingBlur.Parent = Lighting

local ColorCorrection = Instance.new("ColorCorrectionEffect")
ColorCorrection.Enabled = false
ColorCorrection.Parent = Lighting

-- Snow
local SnowLayer = Instance.new("Frame")
SnowLayer.Size = UDim2.fromScale(1, 1)
SnowLayer.BackgroundTransparency = 1
SnowLayer.ClipsDescendants = true
SnowLayer.ZIndex = 2
SnowLayer.Visible = false
SnowLayer.Parent = ScreenGui

local function spawnSnowflake()
    if not UI.Open or not UI.Settings.Snow then return end
    local flake = Instance.new("Frame")
    local size = math.random(2, 4)
    flake.Size = UDim2.fromOffset(size, size)
    flake.Position = UDim2.new(math.random(), 0, -0.05, 0)
    flake.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    flake.BackgroundTransparency = math.random(25, 60) / 100
    flake.BorderSizePixel = 0
    flake.ZIndex = 2
    flake.Parent = SnowLayer
    corner(flake, 99)
    local dur = math.random(6, 12)
    local endX = math.clamp(flake.Position.X.Scale + (math.random() - 0.5) * 0.3, -0.05, 1.05)
    tween(flake, { Position = UDim2.new(endX, 0, 1.1, 0), BackgroundTransparency = 1 }, dur, Enum.EasingStyle.Linear)
    task.delay(dur + 0.1, function() pcall(function() flake:Destroy() end) end)
end

local snowAcc = 0
bind("Snow", RunService.RenderStepped, function(dt)
    if not UI.Open or not UI.Settings.Snow then return end
    snowAcc += dt
    if snowAcc >= 0.08 then
        snowAcc = 0
        for _ = 1, math.random(1, 2) do spawnSnowflake() end
    end
end)

-- ===== WINDOW (Linoria outer outline style) =====
local Outer = Instance.new("Frame")
Outer.Name = "Outer"
Outer.Size = UDim2.fromOffset(UI_W + 6, UI_H + 6)
Outer.Position = UDim2.fromScale(0.5, 0.5)
Outer.AnchorPoint = Vector2.new(0.5, 0.5)
Outer.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
Outer.BorderSizePixel = 0
Outer.ZIndex = 10
Outer.Visible = false
Outer.Active = true
Outer.Parent = ScreenGui
corner(Outer, 10)

local OuterInner = Instance.new("Frame")
OuterInner.Size = UDim2.new(1, -2, 1, -2)
OuterInner.Position = UDim2.fromOffset(1, 1)
OuterInner.BackgroundColor3 = Color3.fromRGB(18, 18, 20)
OuterInner.BorderSizePixel = 0
OuterInner.ZIndex = 10
OuterInner.Parent = Outer
corner(OuterInner, 9)

local Main = Instance.new("Frame")
Main.Name = "Main"
Main.Size = UDim2.new(1, -2, 1, -2)
Main.Position = UDim2.fromOffset(1, 1)
Main.BackgroundColor3 = Theme.WindowBg
Main.BorderSizePixel = 0
Main.ZIndex = 11
Main.Parent = OuterInner
corner(Main, 8)

-- Title
local TitleBar = Instance.new("Frame")
TitleBar.Size = UDim2.new(1, 0, 0, 28)
TitleBar.BackgroundColor3 = Theme.Background
TitleBar.BorderSizePixel = 0
TitleBar.ZIndex = 12
TitleBar.Parent = Main

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, -56, 1, 0)
TitleLabel.Position = UDim2.fromOffset(28, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Font = Enum.Font.GothamMedium
TitleLabel.RichText = true
TitleLabel.Text = '<font color="#FF69B4">Matcha</font>  ·  <font color="#FFFFFF">Interface</font>  ·  <font color="#FF69B4">Pro</font>'
TitleLabel.TextSize = 14
TitleLabel.TextColor3 = Theme.Text
TitleLabel.TextXAlignment = Enum.TextXAlignment.Center
TitleLabel.ZIndex = 13
TitleLabel.Parent = TitleBar

local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.fromOffset(28, 28)
CloseBtn.Position = UDim2.new(1, -28, 0, 0)
CloseBtn.BackgroundTransparency = 1
CloseBtn.Font = Enum.Font.Code
CloseBtn.Text = "×"
CloseBtn.TextSize = 16
CloseBtn.TextColor3 = Theme.TextDark
CloseBtn.ZIndex = 13
CloseBtn.Parent = TitleBar
CloseBtn.MouseEnter:Connect(function() CloseBtn.TextColor3 = Theme.Risky end)
CloseBtn.MouseLeave:Connect(function() CloseBtn.TextColor3 = Theme.TextDark end)

-- drag Outer
do
    local dragging, startIn, startPos
    TitleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            startIn = input.Position
            startPos = Outer.Position
        end
    end)
    TitleBar.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local d = input.Position - startIn
            Outer.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
        end
    end)
end

-- Tab bar (Linoria horizontal)
local TabBar = Instance.new("Frame")
TabBar.Size = UDim2.new(1, -16, 0, 26)
TabBar.Position = UDim2.fromOffset(8, 32)
TabBar.BackgroundColor3 = Theme.Background
TabBar.BorderSizePixel = 0
TabBar.ZIndex = 12
TabBar.Parent = Main
corner(TabBar, 6)
stroke(TabBar, Theme.OutlineSoft, 1, 0)

local TabList = Instance.new("UIListLayout")
TabList.FillDirection = Enum.FillDirection.Horizontal
TabList.Padding = UDim.new(0, 0)
TabList.Parent = TabBar

local Content = Instance.new("Frame")
Content.Size = UDim2.new(1, -16, 1, -70)
Content.Position = UDim2.fromOffset(8, 62)
Content.BackgroundColor3 = Theme.Background
Content.BorderSizePixel = 0
Content.ZIndex = 12
Content.ClipsDescendants = true
Content.Parent = Main
corner(Content, 6)
stroke(Content, Theme.OutlineSoft, 1, 0)

local Footer = Instance.new("TextLabel")
Footer.Size = UDim2.new(1, -16, 0, 16)
Footer.Position = UDim2.new(0, 8, 1, -20)
Footer.BackgroundTransparency = 1
Footer.Font = Enum.Font.Code
Footer.Text = "RightShift / Insert  ·  Matcha Interface Pro"
Footer.TextSize = 12
Footer.TextColor3 = Theme.TextDark
Footer.ZIndex = 12
Footer.Parent = Main

local function createTab(name)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 78, 1, 0)
    btn.BackgroundColor3 = Theme.TabIdle
    btn.BorderSizePixel = 0
    btn.Font = Enum.Font.Code
    btn.Text = name
    btn.TextSize = 12
    btn.TextColor3 = Theme.TextDark
    btn.AutoButtonColor = false
    btn.ZIndex = 13
    btn.Parent = TabBar

    local underline = Instance.new("Frame")
    underline.Size = UDim2.new(1, -8, 0, 2)
    underline.Position = UDim2.new(0, 4, 1, -2)
    underline.BackgroundColor3 = Theme.Accent
    underline.BorderSizePixel = 0
    underline.Visible = false
    underline.ZIndex = 14
    underline.Parent = btn

    local page = Instance.new("ScrollingFrame")
    page.Size = UDim2.fromScale(1, 1)
    page.BackgroundTransparency = 1
    page.BorderSizePixel = 0
    page.ScrollBarThickness = 2
    page.ScrollBarImageColor3 = Theme.TextDark
    page.CanvasSize = UDim2.fromOffset(0, 0)
    page.AutomaticCanvasSize = Enum.AutomaticSize.Y
    page.Visible = false
    page.ZIndex = 13
    page.Parent = Content

    -- two columns like Linoria
    local left = Instance.new("Frame")
    left.Name = "Left"
    left.Size = UDim2.new(0.5, -6, 0, 0)
    left.Position = UDim2.fromOffset(6, 6)
    left.AutomaticSize = Enum.AutomaticSize.Y
    left.BackgroundTransparency = 1
    left.ZIndex = 14
    left.Parent = page
    local ll = Instance.new("UIListLayout")
    ll.Padding = UDim.new(0, 8)
    ll.SortOrder = Enum.SortOrder.LayoutOrder
    ll.Parent = left

    local right = Instance.new("Frame")
    right.Name = "Right"
    right.Size = UDim2.new(0.5, -6, 0, 0)
    right.Position = UDim2.new(0.5, 2, 0, 6)
    right.AutomaticSize = Enum.AutomaticSize.Y
    right.BackgroundTransparency = 1
    right.ZIndex = 14
    right.Parent = page
    local rl = Instance.new("UIListLayout")
    rl.Padding = UDim.new(0, 8)
    rl.SortOrder = Enum.SortOrder.LayoutOrder
    rl.Parent = right

    local tab = { Button = btn, Page = page, Left = left, Right = right, Name = name, Underline = underline }
    UI.Tabs[name] = tab

    btn.MouseButton1Click:Connect(function()
        UI:SelectTab(name)
    end)
    return tab
end

function UI:SelectTab(name)
    for n, t in pairs(self.Tabs) do
        local on = (n == name)
        t.Page.Visible = on
        t.Button.TextColor3 = on and Theme.Text or Theme.TextDark
        t.Button.BackgroundColor3 = on and Theme.TabActive or Theme.TabIdle
        if t.Underline then
            t.Underline.Visible = on
        end
    end
    self.CurrentTab = name
end

-- Groupbox (Linoria style)
local function groupbox(parent, title)
    local box = Instance.new("Frame")
    box.Size = UDim2.new(1, 0, 0, 0)
    box.AutomaticSize = Enum.AutomaticSize.Y
    box.BackgroundColor3 = Theme.Inline
    box.BorderSizePixel = 0
    box.ZIndex = 15
    box.Parent = parent
    corner(box, 6)
    stroke(box, Theme.OutlineSoft, 1, 0)

    local hdr = Instance.new("TextLabel")
    hdr.Size = UDim2.new(1, -10, 0, 22)
    hdr.Position = UDim2.fromOffset(6, 2)
    hdr.BackgroundTransparency = 1
    hdr.Font = Enum.Font.Code
    hdr.Text = title
    hdr.TextSize = 13
    hdr.TextColor3 = Theme.Text
    hdr.TextXAlignment = Enum.TextXAlignment.Left
    hdr.ZIndex = 16
    hdr.Parent = box

    local line = Instance.new("Frame")
    line.Size = UDim2.new(1, -8, 0, 1)
    line.Position = UDim2.fromOffset(4, 24)
    line.BackgroundColor3 = Color3.fromRGB(28, 28, 30)
    line.BorderSizePixel = 0
    line.ZIndex = 16
    line.Parent = box

    local holder = Instance.new("Frame")
    holder.Size = UDim2.new(1, -10, 0, 0)
    holder.Position = UDim2.fromOffset(5, 28)
    holder.AutomaticSize = Enum.AutomaticSize.Y
    holder.BackgroundTransparency = 1
    holder.ZIndex = 16
    holder.Parent = box
    local lay = Instance.new("UIListLayout")
    lay.Padding = UDim.new(0, 4)
    lay.SortOrder = Enum.SortOrder.LayoutOrder
    lay.Parent = holder
    local pad = Instance.new("UIPadding")
    pad.PaddingBottom = UDim.new(0, 6)
    pad.Parent = holder

    return holder
end

local function addToggle(parent, text, default, callback)
    local row = Instance.new("TextButton")
    row.Size = UDim2.new(1, 0, 0, 22)
    row.BackgroundTransparency = 1
    row.Text = ""
    row.ZIndex = 17
    row.AutoButtonColor = false
    row.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -28, 1, 0)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Code
    label.Text = text
    label.TextSize = 13
    label.TextColor3 = Theme.Text
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 18
    label.Parent = row

    local state = default and true or false
    local box = Instance.new("Frame")
    box.Size = UDim2.fromOffset(16, 16)
    box.Position = UDim2.new(1, -18, 0.5, -8)
    box.BackgroundColor3 = Theme.Element
    box.BorderSizePixel = 0
    box.ZIndex = 18
    box.Parent = row
    corner(box, 3)
    stroke(box, Theme.OutlineSoft, 1, 0)

    local check = Instance.new("Frame")
    check.Size = UDim2.fromOffset(10, 10)
    check.Position = UDim2.fromOffset(3, 3)
    check.BackgroundColor3 = Theme.Accent
    corner(check, 2)
    check.BorderSizePixel = 0
    check.Visible = state
    check.ZIndex = 19
    check.Parent = box

    local function render()
        check.Visible = state
        label.TextColor3 = Theme.Text
    end
    render()

    row.MouseButton1Click:Connect(function()
        state = not state
        render()
        if callback then task.spawn(callback, state) end
    end)

    return {
        Set = function(_, v) state = v and true or false; render() end,
        Get = function() return state end,
    }
end

local function addButton(parent, text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 24)
    btn.BackgroundColor3 = Theme.Element
    btn.BorderSizePixel = 0
    btn.Font = Enum.Font.Gotham
    btn.Text = text
    btn.TextSize = 13
    btn.TextColor3 = Theme.Text
    btn.AutoButtonColor = false
    btn.ZIndex = 17
    btn.Parent = parent
    corner(btn, 4)
    stroke(btn, Theme.OutlineSoft, 1, 0)
    btn.MouseEnter:Connect(function() btn.BackgroundColor3 = Theme.ElementHover end)
    btn.MouseLeave:Connect(function() btn.BackgroundColor3 = Theme.Element end)
    btn.MouseButton1Click:Connect(function()
        if callback then task.spawn(callback) end
    end)
    return btn
end

local function addDropdown(parent, text, options, default, callback)
    options = options or {}
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 40)
    row.BackgroundTransparency = 1
    row.ZIndex = 17
    row.ClipsDescendants = false
    row.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 0, 14)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Code
    label.Text = text
    label.TextSize = 12
    label.TextColor3 = Theme.TextDark
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 18
    label.Parent = row

    local current = default or (options[1] or "")
    local open = false

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 22)
    btn.Position = UDim2.fromOffset(0, 16)
    btn.BackgroundColor3 = Theme.Element
    btn.BorderSizePixel = 0
    btn.Font = Enum.Font.Code
    btn.Text = "  " .. tostring(current) .. "   ▼"
    btn.TextSize = 12
    btn.TextColor3 = Theme.Text
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.AutoButtonColor = false
    btn.ZIndex = 50
    btn.Parent = row
    corner(btn, 4)
    stroke(btn, Theme.OutlineSoft, 1, 0)

    -- lista real (ScrollingFrame)
    local maxH = math.min(160, math.max(28, #options * 22))
    local list = Instance.new("ScrollingFrame")
    list.Name = "DropList"
    list.Size = UDim2.new(1, 0, 0, maxH)
    list.Position = UDim2.fromOffset(0, 40)
    list.BackgroundColor3 = Color3.fromRGB(22, 22, 26)
    list.BorderSizePixel = 0
    list.Visible = false
    list.ZIndex = 80
    list.ScrollBarThickness = 3
    list.ScrollBarImageColor3 = Theme.Accent
    list.CanvasSize = UDim2.fromOffset(0, #options * 22)
    list.AutomaticCanvasSize = Enum.AutomaticSize.Y
    list.Parent = row
    corner(list, 4)
    stroke(list, Theme.Accent, 1, 0)

    local layout = Instance.new("UIListLayout")
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Padding = UDim.new(0, 0)
    layout.Parent = list

    local function setValue(v, fire)
        current = v
        btn.Text = "  " .. tostring(current) .. "   ▼"
        if fire and callback then task.spawn(callback, current) end
    end

    local function closeList()
        open = false
        list.Visible = false
        row.Size = UDim2.new(1, 0, 0, 40)
        btn.Text = "  " .. tostring(current) .. "   ▼"
    end

    local function openList()
        -- cerrar otros dropdowns abiertos
        for _, sib in ipairs(parent:GetChildren()) do
            local dl = sib:FindFirstChild("DropList")
            if dl and dl ~= list then
                dl.Visible = false
                sib.Size = UDim2.new(1, 0, 0, 40)
            end
        end
        open = true
        list.Visible = true
        row.Size = UDim2.new(1, 0, 0, 40 + maxH + 4)
        btn.Text = "  " .. tostring(current) .. "   ▲"
        list.CanvasSize = UDim2.fromOffset(0, #options * 22)
    end

    -- items
    for i, opt in ipairs(options) do
        local item = Instance.new("TextButton")
        item.Size = UDim2.new(1, 0, 0, 22)
        item.BackgroundColor3 = Theme.Element
        item.BorderSizePixel = 0
        item.Font = Enum.Font.Code
        item.Text = "  " .. tostring(opt)
        item.TextSize = 12
        item.TextColor3 = Theme.Text
        item.TextXAlignment = Enum.TextXAlignment.Left
        item.AutoButtonColor = false
        item.ZIndex = 81
        item.LayoutOrder = i
        item.Parent = list
        item.MouseEnter:Connect(function()
            item.BackgroundColor3 = Color3.fromRGB(40, 40, 48)
            item.TextColor3 = Theme.Accent
        end)
        item.MouseLeave:Connect(function()
            item.BackgroundColor3 = Theme.Element
            item.TextColor3 = Theme.Text
        end)
        item.MouseButton1Click:Connect(function()
            setValue(opt, true)
            closeList()
        end)
    end

    btn.MouseButton1Click:Connect(function()
        if open then closeList() else openList() end
    end)

    return {
        Set = function(_, v) setValue(v, false) end,
        Get = function() return current end,
        SetValues = function(_, newOpts)
            options = newOpts or {}
            for _, ch in ipairs(list:GetChildren()) do
                if ch:IsA("TextButton") then ch:Destroy() end
            end
            maxH = math.min(160, math.max(28, #options * 22))
            list.Size = UDim2.new(1, 0, 0, maxH)
            list.CanvasSize = UDim2.fromOffset(0, #options * 22)
            for i, opt in ipairs(options) do
                local item = Instance.new("TextButton")
                item.Size = UDim2.new(1, 0, 0, 22)
                item.BackgroundColor3 = Theme.Element
                item.BorderSizePixel = 0
                item.Font = Enum.Font.Code
                item.Text = "  " .. tostring(opt)
                item.TextSize = 12
                item.TextColor3 = Theme.Text
                item.TextXAlignment = Enum.TextXAlignment.Left
                item.AutoButtonColor = false
                item.ZIndex = 81
                item.LayoutOrder = i
                item.Parent = list
                item.MouseButton1Click:Connect(function()
                    setValue(opt, true)
                    closeList()
                end)
            end
        end,
    }
end

local function addSlider(parent, text, min, max, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 36)
    row.BackgroundTransparency = 1
    row.ZIndex = 17
    row.Parent = parent

    local val = default or min
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 0, 14)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Code
    label.Text = text .. " : " .. tostring(val)
    label.TextSize = 12
    label.TextColor3 = Theme.TextDark
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 18
    label.Parent = row

    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1, 0, 0, 10)
    bar.Position = UDim2.fromOffset(0, 20)
    bar.BackgroundColor3 = Theme.Element
    bar.BorderSizePixel = 0
    bar.Active = true
    bar.ZIndex = 18
    bar.Parent = row
    corner(bar, 4)
    stroke(bar, Theme.OutlineSoft, 1, 0)

    local fill = Instance.new("Frame")
    local pct = (val - min) / math.max(max - min, 1)
    fill.Size = UDim2.new(pct, 0, 1, 0)
    fill.BackgroundColor3 = Theme.Accent
    fill.BorderSizePixel = 0
    fill.ZIndex = 19
    fill.Parent = bar
    corner(fill, 4)

    local sliding = false
    local function update(pos)
        local rel = math.clamp((pos.X - bar.AbsolutePosition.X) / math.max(bar.AbsoluteSize.X, 1), 0, 1)
        val = math.floor((min + (max - min) * rel) * 100 + 0.5) / 100
        fill.Size = UDim2.new(rel, 0, 1, 0)
        label.Text = text .. " : " .. tostring(val)
        if callback then callback(val) end
    end
    bar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            sliding = true
            update(input.Position)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            sliding = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if sliding and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            update(input.Position)
        end
    end)
end

local function addInput(parent, text, placeholder, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 40)
    row.BackgroundTransparency = 1
    row.ZIndex = 17
    row.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 0, 14)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Code
    label.Text = text
    label.TextSize = 12
    label.TextColor3 = Theme.TextDark
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 18
    label.Parent = row

    local box = Instance.new("TextBox")
    box.Size = UDim2.new(1, 0, 0, 22)
    box.Position = UDim2.fromOffset(0, 16)
    box.BackgroundColor3 = Theme.Element
    box.BorderSizePixel = 0
    box.Font = Enum.Font.Code
    box.PlaceholderText = placeholder or ""
    box.PlaceholderColor3 = Theme.TextDark
    box.Text = ""
    box.TextSize = 12
    box.TextColor3 = Theme.Text
    box.ClearTextOnFocus = false
    box.ZIndex = 18
    box.Parent = row
    corner(box, 4)
    stroke(box, Theme.OutlineSoft, 1, 0)
    local p = Instance.new("UIPadding")
    p.PaddingLeft = UDim.new(0, 6)
    p.Parent = box
    box.FocusLost:Connect(function()
        if callback then callback(box.Text) end
    end)
    return box
end


local function addColorPicker(parent, text, defaultCol, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 22)
    row.BackgroundTransparency = 1
    row.ZIndex = 17
    row.Parent = parent

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -28, 1, 0)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Code
    label.Text = text
    label.TextSize = 13
    label.TextColor3 = Theme.Text
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 18
    label.Parent = row

    local col = defaultCol or Color3.fromRGB(255, 105, 180)
    local swatch = Instance.new("TextButton")
    swatch.Size = UDim2.fromOffset(18, 18)
    swatch.Position = UDim2.new(1, -20, 0.5, -9)
    swatch.BackgroundColor3 = col
    swatch.BorderSizePixel = 0
    swatch.Text = ""
    swatch.AutoButtonColor = false
    swatch.ZIndex = 18
    swatch.Parent = row
    corner(swatch, 3)
    stroke(swatch, Theme.OutlineSoft, 1, 0)

    -- palette al click (presets + custom rosa/blanco/etc)
    local presets = {
        Color3.fromRGB(255, 105, 180), -- Matcha pink
        Color3.fromRGB(255, 255, 255),
        Color3.fromRGB(255, 80, 80),
        Color3.fromRGB(80, 200, 255),
        Color3.fromRGB(120, 255, 120),
        Color3.fromRGB(255, 200, 60),
        Color3.fromRGB(180, 100, 255),
        Color3.fromRGB(0, 0, 0),
    }
    local pidx = 1
    for i, c in ipairs(presets) do
        if math.abs(c.R - col.R) < 0.02 and math.abs(c.G - col.G) < 0.02 and math.abs(c.B - col.B) < 0.02 then
            pidx = i
            break
        end
    end

    swatch.MouseButton1Click:Connect(function()
        pidx = pidx % #presets + 1
        col = presets[pidx]
        swatch.BackgroundColor3 = col
        if callback then task.spawn(callback, col) end
    end)
    -- right click = open RGB mini (R cycle coarse)
    swatch.MouseButton2Click:Connect(function()
        -- fine: shift hue
        local h, s, v = Color3.toHSV(col)
        h = (h + 0.08) % 1
        col = Color3.fromHSV(h, math.max(s, 0.6), math.max(v, 0.85))
        swatch.BackgroundColor3 = col
        if callback then task.spawn(callback, col) end
    end)

    return {
        Set = function(_, c)
            if typeof(c) == "Color3" then
                col = c
                swatch.BackgroundColor3 = c
            end
        end,
        Get = function() return col end,
    }
end

local function addLabel(parent, text)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, 0, 0, 16)
    l.BackgroundTransparency = 1
    l.Font = Enum.Font.Code
    l.Text = text
    l.TextSize = 12
    l.TextColor3 = Theme.TextDark
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.ZIndex = 17
    l.Parent = parent
    return l
end

local function applyOpenEffects()
    local useDark = UI.Settings.Darken ~= false
    local useSnow = UI.Settings.Snow == true
    Backdrop.Visible = useDark
    SnowLayer.Visible = useSnow
    if useDark then
        LightingBlur.Enabled = true
        ColorCorrection.Enabled = true
        tween(Backdrop, { BackgroundTransparency = 0.4 }, 0.3)
        tween(BlurFrame, { BackgroundTransparency = 0.55 }, 0.3)
        tween(LightingBlur, { Size = 16 }, 0.35)
        tween(ColorCorrection, { Brightness = -0.1, Contrast = 0.06, Saturation = -0.12 }, 0.35)
    else
        Backdrop.Visible = false
        Backdrop.BackgroundTransparency = 1
        BlurFrame.BackgroundTransparency = 1
        LightingBlur.Enabled = false
        LightingBlur.Size = 0
        ColorCorrection.Enabled = false
    end
    if not useSnow then
        SnowLayer.Visible = false
        for _, c in ipairs(SnowLayer:GetChildren()) do pcall(function() c:Destroy() end) end
    end
    Outer.Visible = true
end

local function applyCloseEffects()
    local token = tick()
    UI._closeToken = token
    tween(Backdrop, { BackgroundTransparency = 1 }, 0.2)
    tween(BlurFrame, { BackgroundTransparency = 1 }, 0.2)
    pcall(function() tween(LightingBlur, { Size = 0 }, 0.2) end)
    pcall(function() tween(ColorCorrection, { Brightness = 0, Contrast = 0, Saturation = 0 }, 0.2) end)
    task.delay(0.25, function()
        if UI.Open or UI._closeToken ~= token then return end
        Outer.Visible = false
        Backdrop.Visible = false
        SnowLayer.Visible = false
        pcall(function()
            LightingBlur.Size = 0
            LightingBlur.Enabled = false
            ColorCorrection.Enabled = false
            ColorCorrection.Brightness = 0
            ColorCorrection.Contrast = 0
            ColorCorrection.Saturation = 0
            Lighting.Ambient = OriginalLighting.Ambient
            Lighting.OutdoorAmbient = OriginalLighting.OutdoorAmbient
            Lighting.Brightness = OriginalLighting.Brightness
            Lighting.ExposureCompensation = OriginalLighting.ExposureCompensation
        end)
        for _, c in ipairs(SnowLayer:GetChildren()) do pcall(function() c:Destroy() end) end
    end)
end

function UI:OpenMenu()
    if self.Unloaded then return end
    self.Open = true
    applyOpenEffects()
    pcall(function()
        local TweenService = game:GetService("TweenService")
        Main.Visible = true
        Main.BackgroundTransparency = 1
        local scale = Main:FindFirstChildOfClass("UIScale")
        if not scale then
            scale = Instance.new("UIScale")
            scale.Parent = Main
        end
        scale.Scale = 0.92
        TweenService:Create(Main, TweenInfo.new(0.22, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundTransparency = 0
        }):Play()
        TweenService:Create(scale, TweenInfo.new(0.22, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Scale = 1
        }):Play()
    end)
end

function UI:CloseMenu()
    self.Open = false
    pcall(function()
        local TweenService = game:GetService("TweenService")
        local scale = Main:FindFirstChildOfClass("UIScale")
        if not scale then
            scale = Instance.new("UIScale")
            scale.Parent = Main
        end
        local t1 = TweenService:Create(Main, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            BackgroundTransparency = 1
        })
        local t2 = TweenService:Create(scale, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            Scale = 0.92
        })
        t1:Play(); t2:Play()
        t1.Completed:Connect(function()
            if not UI.Open then
                Main.Visible = false
                applyCloseEffects()
            end
        end)
    end)
    if not Main then
        applyCloseEffects()
    end
end

function UI:Toggle()
    if self.Open then self:CloseMenu() else self:OpenMenu() end
end

CloseBtn.MouseButton1Click:Connect(function() UI:CloseMenu() end)

bind("ToggleKey", UserInputService.InputBegan, function(input)
    if UI.Unloaded then return end
    if input.KeyCode == TOGGLE_KEY or input.KeyCode == Enum.KeyCode.Insert then
        UI:Toggle()
    end
end)

-- ==================== PAGES ====================
local tabCombat = createTab("Combat")
local tabSpoof = createTab("Spoofer")
local tabUnlock = createTab("Unlocker")
local tabVisual = createTab("Visuals")
local tabESP = createTab("ESP")
local tabChar = createTab("Character")
local tabMisc = createTab("Misc")


-- COMBAT (Silent único = el que funciona · Classic/Mouse + hooks)
do
    local s = groupbox(tabCombat.Left, "Silent Aim")
    addToggle(s, "Enable Silent", false, function(val)
        -- mismo path que el silent de abajo (el que servía)
        local sc = getgenv()._Vnix_silent
        if silent_cfg then silent_cfg.enabled = val and true or false end
        if sc then
            if val then
                if sc._start then sc._start() end
            else
                if sc._stop then sc._stop() end
            end
        else
            -- fallback Matcha hooks
            local ms = getgenv()._Matcha_silent
            if ms then
                if val and ms._start then ms._start()
                elseif (not val) and ms._stop then ms._stop() end
            end
        end
        pcall(function()
            getgenv().Rivals = getgenv().Rivals or {}
            getgenv().Rivals.Silent = getgenv().Rivals.Silent or {}
            getgenv().Rivals.Silent.enabled = val and true or false
        end)
        if val and type(silentStart) == "function" then pcall(silentStart) end
        if (not val) and type(silentStop) == "function" then pcall(silentStop) end
        Notify("Silent", val and ("ON · " .. tostring((silent_cfg and silent_cfg.mode) or "Classic")) or "OFF")
    end)
    addDropdown(s, "Mode", {"Classic", "Mouse"}, "Classic", function(val)
        if silent_cfg then silent_cfg.mode = val end
        local sc = getgenv()._Vnix_silent
        if sc then sc.mode = val end
        Notify("Silent Mode", val)
    end)
    addLabel(s, "Classic = bala va sola (no ves TP)")
    addLabel(s, "Mouse = FOV sigue el cursor")
    addToggle(s, "Team Check", false, function(val)
        if silent_cfg then silent_cfg.team_check = val end
        local sc = getgenv()._Vnix_silent
        if sc then sc.team_check = val end
        local ms = getgenv()._Matcha_silent
        if ms then ms.team_check = val end
    end)
    addToggle(s, "Show FOV", true, function(val)
        if silent_cfg then
            silent_cfg.show_fov = val
            silent_cfg.fov_enabled = val
        end
        local sc = getgenv()._Vnix_silent
        if sc then sc.show_fov = val end
    end)
    addToggle(s, "FOV Fill", false, function(val)
        if silent_cfg then silent_cfg.fov_fill = val and true or false end
        local sc = getgenv()._Vnix_silent
        if sc then sc.fov_fill = val and true or false end
    end)
    addToggle(s, "FOV Spin", false, function(val)
        if silent_cfg then silent_cfg.fov_spin = val and true or false end
    end)
    addSlider(s, "FOV Size", 40, 800, 250, function(val)
        if silent_cfg then silent_cfg.fov = val end
        local sc = getgenv()._Vnix_silent
        if sc then sc.fov = val end
        local ms = getgenv()._Matcha_silent
        if ms then ms.fov = val end
    end)
    addSlider(s, "Spin Speed", 0.5, 8, 2.5, function(val)
        if silent_cfg then silent_cfg.fov_spin_speed = val end
    end)
    addDropdown(s, "Hit Part", {"Head", "HumanoidRootPart", "UpperTorso"}, "Head", function(val)
        if silent_cfg then silent_cfg.hitpart = val; silent_cfg.target_part = val end
        local sc = getgenv()._Vnix_silent
        if sc then sc.hitpart = val end
    end)
    addColorPicker(s, "FOV Color", Color3.fromRGB(255, 105, 180), function(col)
        if silent_cfg then silent_cfg.fov_color = col end
        local sc = getgenv()._Vnix_silent
        if sc then sc.fov_color = col end
    end)
    addColorPicker(s, "Fill Color 1", Color3.fromRGB(255, 105, 180), function(col)
        if silent_cfg then silent_cfg.fov_fill_color1 = col end
    end)
    addColorPicker(s, "Fill Color 2", Color3.fromRGB(255, 255, 255), function(col)
        if silent_cfg then silent_cfg.fov_fill_color2 = col end
    end)
    addLabel(s, "Un solo Silent · hooks Utility+Bullet+cast")

    -- AIMBOT (metodo shako.win: hold MB2 + FOV + mousemoverel)
    local a = groupbox(tabCombat.Left, "Aimbot")
    addToggle(a, "Enable Aimbot", false, function(val)
        local ac = getgenv()._Matcha_aim
        if not ac then return end
        ac.enabled = val and true or false
        if val then
            if ac._start then ac._start() end
            Notify("Aimbot", "ON · hold click derecho (shako method)")
        else
            if ac._stop then ac._stop() end
            Notify("Aimbot", "OFF")
        end
    end)
    addToggle(a, "Team Check (2v2-5v5)", true, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.team_check = val and true or false end
    end)
    addToggle(a, "Force FFA", false, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.force_ffa = val and true or false end
    end)
    addToggle(a, "Wall Check", true, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.wall_check = val and true or false end
    end)
    addToggle(a, "Sticky (no cambia target)", true, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.sticky = val and true or false end
    end)
    addToggle(a, "Show FOV", true, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then
            ac.show_fov = val and true or false
            if ac.enabled and ac._start then ac._start() end
        end
    end)
    addToggle(a, "FOV Fill", false, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.fov_fill = val and true or false end
    end)
    addToggle(a, "FOV Spin", false, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.fov_spin = val and true or false end
    end)
    addToggle(a, "Skip ForceField", true, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.skip_ff = val and true or false end
    end)
    addToggle(a, "Toggle Mode (sin hold)", false, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then
            ac.toggle_mode = val and true or false
            ac.toggled = false
            aim_holding = false
        end
    end)
    addDropdown(a, "Target Part", {"Head", "HumanoidRootPart", "Closest"}, "Head", function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.target_part = val end
    end)
    addDropdown(a, "Priority", {"Crosshair", "Closest", "LowestHP"}, "Crosshair", function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.priority = val end
    end)
    addSlider(a, "FOV", 20, 400, 140, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.fov = val end
    end)
    addSlider(a, "Spin Speed", 0.5, 8, 2.5, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.fov_spin_speed = val end
    end)
    addSlider(a, "Fill Transparency", 0.3, 0.95, 0.75, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.fov_fill_transparency = val end
    end)
    addColorPicker(a, "FOV Outline", Color3.fromRGB(255, 105, 180), function(col)
        local ac = getgenv()._Matcha_aim
        if ac then ac.fov_color = col end
    end)
    addColorPicker(a, "Fill Color 1", Color3.fromRGB(255, 105, 180), function(col)
        local ac = getgenv()._Matcha_aim
        if ac then ac.fov_fill_color1 = col end
    end)
    addColorPicker(a, "Fill Color 2", Color3.fromRGB(255, 255, 255), function(col)
        local ac = getgenv()._Matcha_aim
        if ac then ac.fov_fill_color2 = col end
    end)
    addSlider(a, "Smooth %", 1, 100, 28, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then
            -- 1 = muy rapido, 100 = muy suave (mouse libre)
            ac.smooth = math.clamp(val / 100, 0.01, 1)
        end
    end)
    addSlider(a, "Max Step (px)", 2, 20, 6, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.max_step = val end
    end)
    addSlider(a, "Range", 50, 2000, 800, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.range = val end
    end)
    addSlider(a, "Predict Amount", 0, 0.4, 0.12, function(val)
        local ac = getgenv()._Matcha_aim
        if ac then ac.predict_amount = val end
    end)
    addLabel(a, "Smooth %: 1=rapido · 100=suave (mouse libre)")
    addLabel(a, "Hold MB2 · FOV Fill/Spin con 2 colores")



    local t = groupbox(tabCombat.Right, "Triggerbot")
    addToggle(t, "Enable Triggerbot", false, function(val)
        local tc = getgenv()._Matcha_trigger
        if not tc then return end
        tc.enabled = val and true or false
        if val then
            if tc._start then tc._start() end
            Notify("Trigger", "ON")
        else
            if tc._stop then tc._stop() end
            Notify("Trigger", "OFF")
        end
    end)
    addToggle(t, "Team Check", true, function(val)
        local tc = getgenv()._Matcha_trigger
        if tc then tc.team_check = val and true or false end
    end)
    addToggle(t, "Force FFA", false, function(val)
        local tc = getgenv()._Matcha_trigger
        if tc then tc.force_ffa = val and true or false end
    end)
    addToggle(t, "Wall Check", true, function(val)
        local tc = getgenv()._Matcha_trigger
        if tc then tc.wall_check = val and true or false end
    end)
    addSlider(t, "Delay (s)", 0, 0.3, 0.02, function(val)
        local tc = getgenv()._Matcha_trigger
        if tc then tc.delay = val end
    end)
    addSlider(t, "Release (s)", 0, 0.2, 0.05, function(val)
        local tc = getgenv()._Matcha_trigger
        if tc then tc.release = val end
    end)
    addLabel(t, "Dispara al poner crosshair en enemigo")

    local mu = groupbox(tabCombat.Right, "Manipulation UE")
    addToggle(mu, "Enable Manipulation", false, function(val)
        local m = getgenv()._Matcha_manip
        if not m then return end
        if val then
            if m._start then m._start() end
            Notify("Manip UE", "ON · UseItem StartShooting")
        else
            if m._stop then m._stop() end
            Notify("Manip UE", "OFF")
        end
    end)
    addToggle(mu, "Only Enemies", true, function(val)
        local m = getgenv()._Matcha_manip
        if m then m.only_enemies = val and true or false end
    end)
    addLabel(mu, "Source manipulation ue.txt")
    addLabel(mu, "Equipa arma · dispara a través de paredes")

    local info = groupbox(tabCombat.Left, "Info Combat")
    addLabel(info, "Aimbot aquí · Rage → Visuals")
    addLabel(info, "Silent = balas van al target")
    addLabel(info, "Trigger = auto click")
    addLabel(info, "Manip UE = camera shoot remote")
end

-- CHARACTER (Matcha movement)
do
    local m = groupbox(tabChar.Left, "Movement")
    addToggle(m, "Speed Hack", false, function(val)
        local c = getgenv()._Matcha_char
        if not c then return end
        c.speed_enabled = val and true or false
        if val then
            pcall(charHookWalkSpeed)
            pcall(charApplySpeed)
            Notify("Speed", "ON · " .. tostring(c.speed))
        else
            local ch = LocalPlayer.Character
            local hum = ch and ch:FindFirstChildOfClass("Humanoid")
            if hum then pcall(function() hum.WalkSpeed = 16 end) end
            Notify("Speed", "OFF")
        end
    end)
    addSlider(m, "Speed", 16, 150, 35, function(val)
        local c = getgenv()._Matcha_char
        if c then
            c.speed = val
            if c.speed_enabled then pcall(charApplySpeed) end
        end
    end)
    addToggle(m, "Infinite Jump", false, function(val)
        local c = getgenv()._Matcha_char
        if c then c.inf_jump = val and true or false end
    end)
    addToggle(m, "Noclip", false, function(val)
        local c = getgenv()._Matcha_char
        if not c then return end
        c.noclip = val and true or false
        if val and c._start then c._start() end
    end)
    addToggle(m, "Fly", false, function(val)
        local c = getgenv()._Matcha_char
        if c then c.fly = val and true or false end
    end)
    addSlider(m, "Fly Speed", 10, 150, 50, function(val)
        local c = getgenv()._Matcha_char
        if c then c.fly_speed = val end
    end)
    addLabel(m, "Fly: WASD + Space/Ctrl")

    local w = groupbox(tabChar.Left, "Weapon Fire (ItemLibrary)")
    addToggle(w, "Fast Fire (Armas)", false, function(val)
        local il = getgenv()._Matcha_itemlib
        if not il then return end
        if val then
            local ok = il._startGuns and il._startGuns()
            Notify("Fast Fire", ok and "ON · ShootCooldown bajo" or "FAIL · ItemLibrary")
        else
            if il._stopGuns then il._stopGuns() end
            Notify("Fast Fire", "OFF")
        end
    end)
    addToggle(w, "No Recoil (con Fast Fire)", true, function(val)
        local il = getgenv()._Matcha_itemlib
        if il then
            il.no_recoil = val and true or false
            if il.rapid_fire and il._applyGuns then il._applyGuns(true) end
        end
    end)
    addSlider(w, "Shoot Cooldown", 0.012, 0.1, 0.025, function(val)
        local il = getgenv()._Matcha_itemlib
        if il then
            il.rapid_delay = val
            if il.rapid_fire and il._applyGuns then il._applyGuns(true) end
        end
    end)
    addSlider(w, "Melee Cooldown", 0.02, 0.15, 0.05, function(val)
        local il = getgenv()._Matcha_itemlib
        if il then
            il.melee_delay = val
            if il.fast_melee and il._applyMelee then il._applyMelee(true) end
        end
    end)
    addToggle(w, "Fast Melee", false, function(val)
        local il = getgenv()._Matcha_itemlib
        if not il then return end
        if val then
            local ok = il._startMelee and il._startMelee()
            Notify("Fast Melee", ok and "ON · AttackCooldown bajo" or "FAIL · ItemLibrary")
        else
            if il._stopMelee then il._stopMelee() end
            Notify("Fast Melee", "OFF")
        end
    end)
    addLabel(w, "Solo armas/melee · no toca otros items")
    addLabel(w, "Equipa arma/melee en ronda")

    local h = groupbox(tabChar.Right, "Combat Helpers")
    addToggle(h, "Target Hovering", false, function(val)
        local c = getgenv()._Matcha_char
        if c then
            c.target_hover = val and true or false
            getgenv()._Matcha_hover = nil
        end
    end)
    addSlider(h, "Hover Radius", 4, 30, 12, function(val)
        local c = getgenv()._Matcha_char
        if c then c.hover_radius = val end
    end)
    addSlider(h, "Hover Speed", 1, 20, 7, function(val)
        local c = getgenv()._Matcha_char
        if c then c.hover_speed = val end
    end)
    addToggle(h, "Hitbox Expander", false, function(val)
        local hb = getgenv()._Matcha_hitbox
        local c = getgenv()._Matcha_char
        if val then
            if hb then
                hb.size = (c and c.hitbox_size) or hb.size or 5
                if hb._start then hb._start() end
            end
            if c then c.hitbox_expand = true end
            Notify("Hitbox", "ON · dispara al cubo rosa")
        else
            if hb and hb._stop then hb._stop() end
            if c then c.hitbox_expand = false end
            Notify("Hitbox", "OFF")
        end
    end)
    addSlider(h, "Hitbox Size", 2, 20, 5, function(val)
        local c = getgenv()._Matcha_char
        local hb = getgenv()._Matcha_hitbox
        if c then c.hitbox_size = val end
        if hb then
            hb.size = val
            -- re-aplica en vivo
            if hb.enabled and hb._tick then pcall(hb._tick) end
        end
    end)
    addLabel(h, "Hitbox = partes grandes + CanQuery")
    addLabel(h, "Dispara al cubo rosa = hit")
    addLabel(h, "Hover = orbita al enemigo")

    -- Rankeds (antes tab Rankeds)
    local q = groupbox(tabChar.Left, "Auto Queue")
    addToggle(q, "Enabled", false, function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.auto_queue = val and true or false end
        Notify("Queue", val and "ON" or "OFF")
    end)
    addDropdown(q, "Select Match", {"1v1", "2v2", "3v3", "4v4", "5v5", "2v2_beginner"}, "1v1", function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.queue_mode = val end
    end)
    addLabel(q, "JoinQueue si no estás en partida")

    local v = groupbox(tabChar.Left, "Auto Vote Map")
    addToggle(v, "Enabled", false, function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.auto_vote = val and true or false end
        Notify("Vote", val and "ON" or "OFF")
    end)
    local maps = (getgenv()._Matcha_ranked_lists and getgenv()._Matcha_ranked_lists.maps) or {"Arena"}
    addDropdown(v, "Map", maps, "Arena", function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.vote_map = val end
    end)
    addLabel(v, "Duels.Vote → mapa elegido")

    local lo = groupbox(tabChar.Right, "Auto Loadout")
    addToggle(lo, "Enabled", false, function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.auto_loadout = val and true or false end
        Notify("Loadout", val and "ON" or "OFF")
    end)
    local lists = getgenv()._Matcha_ranked_lists or {}
    addDropdown(lo, "Primary", lists.primary or {"Assault Rifle"}, "Assault Rifle", function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.primary = val end
    end)
    addDropdown(lo, "Secondary", lists.secondary or {"Handgun"}, "Handgun", function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.secondary = val end
    end)
    addDropdown(lo, "Melee", lists.melee or {"Fists"}, "Fists", function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.melee = val end
    end)
    addDropdown(lo, "Utility", lists.utility or {"Grenade"}, "Grenade", function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.utility = val end
    end)
    addLabel(lo, "PickWeapons cada 2s")

    local bn = groupbox(tabChar.Right, "Auto Ban")
    addToggle(bn, "Enabled", false, function(val)
        local r = getgenv()._Matcha_ranked
        if r then r.auto_ban = val and true or false end
        Notify("Ban", val and "ON · cycle lista" or "OFF")
    end)
    addDropdown(bn, "Add Ban Target", lists.ban_items or maps, "Arena", function(val)
        local r = getgenv()._Matcha_ranked
        if not r then return end
        r.ban_list = r.ban_list or {}
        for _, x in ipairs(r.ban_list) do
            if x == val then return end
        end
        table.insert(r.ban_list, val)
        Notify("Ban", "Added: " .. tostring(val) .. " (" .. #r.ban_list .. ")")
    end)
    addButton(bn, "Clear Ban List", function()
        local r = getgenv()._Matcha_ranked
        if r then
            r.ban_list = {}
            r.ban_index = 1
        end
        Notify("Ban", "Lista vacía")
    end)
    addLabel(bn, "Vota ban en ciclo cada 1s")

    -- Other (device quick)
    local o = groupbox(tabChar.Right, "Device Spoof (quick)")
    addDropdown(o, "Device", {"Computer", "Mobile", "Console", "VR"}, "VR", function(val)
        states.spoofer_state.device = val
    end)
    addButton(o, "Apply SetControls", function()
        local d = states.spoofer_state.device or "VR"
        pcall(change_device, d)
        Notify("Device", "→ " .. d)
    end)
    addToggle(o, "Spoof ON", false, function(val)
        states.spoofer_state.spoof_device = val
        if val then
            pcall(change_device, states.spoofer_state.device or "VR")
        else
            pcall(change_device, "Computer")
        end
    end)
    addLabel(o, "Source DeviceSpoof.txt")
end


-- SPOOFER L/R
do
    local s = groupbox(tabSpoof.Left, "Platform")
    -- DeviceSpoof.txt: Computer / Mobile / Console / VR → SetControls
    local devices = {"Computer", "Mobile", "Console", "VR"}
    addDropdown(s, "Device", devices, "Computer", function(v)
        states.spoofer_state.device = v
        if states.spoofer_state.spoof_device then
            pcall(change_device, v)
        end
    end)
    addToggle(s, "Spoof Device", false, function(v)
        states.spoofer_state.spoof_device = v
        if v then
            pcall(change_device, states.spoofer_state.device or "Computer")
            Notify("Device", "Spoof → " .. tostring(states.spoofer_state.device))
        else
            pcall(change_device, "Computer")
            Notify("Device", "OFF · Computer")
        end
    end)
    addButton(s, "Apply Device Now", function()
        local d = states.spoofer_state.device or "Computer"
        pcall(change_device, d)
        Notify("Device", "SetControls → " .. d)
    end)
    addToggle(s, "Device Spam", false, function(v)
        states.spoofer_state.device_spam = v
        if v then pcall(DeviceSpam) end
    end)
    addSlider(s, "Spam Rate", 0.1, 5, 1, function(v)
        states.spoofer_state.spam_rate = v
    end)
    addLabel(s, "Source DeviceSpoof.txt · SetControls")

    local p = groupbox(tabSpoof.Left, "Player")
    addInput(p, "Display Name", LocalPlayer.DisplayName, function(v)
        states.spoofer_state.display_name_value = v
    end)
    addInput(p, "Username", LocalPlayer.Name, function(v)
        states.spoofer_state.username_value = v
    end)
    addToggle(p, "Spoof Display Name", false, function(v)
        states.spoofer_state.display_name = v
    end)
    addToggle(p, "Spoof Username", false, function(v)
        states.spoofer_state.username = v
    end)
    addInput(p, "Avatar UserId", tostring(LocalPlayer.UserId), function(v)
        states.spoofer_state.avatar_userid = tonumber(v) or LocalPlayer.UserId
    end)
    addToggle(p, "Spoof Avatar", false, function(v)
        states.spoofer_state.spoof_avatar = v
        pcall(SpoofAvatar)
    end)
    addToggle(p, "Anonymous Mode", false, function(v)
        states.spoofer_state.anonymous_mode = v
    end)

    local b = groupbox(tabSpoof.Right, "Badges")
    for _, badge in ipairs({"Premium", "Verified", "Influencer", "Admin"}) do
        addToggle(b, "Badge: " .. badge, false, function(v)
            states.spoofer_state.badges = states.spoofer_state.badges or {}
            states.spoofer_state.badges[badge] = v
        end)
    end

    local lb = groupbox(tabSpoof.Right, "Leaderboard")
    addInput(lb, "ELO", "0", function(v)
        states.spoofer_state.leaderboard.elo_value = tonumber(v) or 0
        if states.spoofer_state.leaderboard.ELO then
            pcall(UpdateLeaderboardOptions)
        end
    end)
    addInput(lb, "Streak", "0", function(v)
        states.spoofer_state.leaderboard.streak_value = tonumber(v) or 0
        if states.spoofer_state.leaderboard.Streak then
            pcall(UpdateLeaderboardOptions)
        end
    end)
    addInput(lb, "Kills", "0", function(v) states.spoofer_state.leaderboard.kills_value = tonumber(v) or 0 end)
    addInput(lb, "Wins", "0", function(v) states.spoofer_state.leaderboard.wins_value = tonumber(v) or 0 end)
    addInput(lb, "Level", "0", function(v) states.spoofer_state.leaderboard.level_value = tonumber(v) or 0 end)
    for _, k in ipairs({"ELO", "Streak", "Kills", "Wins", "Level"}) do
        addToggle(lb, "Spoof " .. k, false, function(v)
            states.spoofer_state.leaderboard[k] = v
            pcall(UpdateLeaderboardOptions)
        end)
    end

    -- ===== CHARMS (robado del otro script) =====
    local ch = groupbox(tabSpoof.Left, "Charm")
    addDropdown(ch, "Charm Rank", {
        "Use Spoofed ELO", "Unranked",
        "Bronze 1", "Bronze 2", "Bronze 3",
        "Silver 1", "Silver 2", "Silver 3",
        "Gold 1", "Gold 2", "Gold 3",
        "Platinum 1", "Platinum 2", "Platinum 3",
        "Diamond 1", "Diamond 2", "Diamond 3",
        "Onyx 1", "Onyx 2", "Onyx 3",
        "Nemesis", "Archnemesis",
    }, "Use Spoofed ELO", function(v)
        states.spoofer_state.charm.charm_rank = v
        pcall(UpdateCharmOptions)
    end)
    addSlider(ch, "Arch Rank", 1, 100, 1, function(v)
        states.spoofer_state.charm.arch_rank = v
        pcall(UpdateCharmOptions)
    end)
    for i = 0, 3 do
        addToggle(ch, "Season " .. i .. " Charm", false, function(v)
            states.spoofer_state.charm["s" .. i .. "_charm"] = v
            pcall(UpdateCharmOptions)
        end)
    end
    addButton(ch, "Apply Charms Now", function()
        pcall(UpdateCharmOptions)
        Notify("Charm", "Applied")
    end)

    local mp = groupbox(tabSpoof.Right, "Marketplace")
    addToggle(mp, "Fake Robux", false, function(v)
        states.spoofer_state.marketplace.fake_robux = v
        pcall(UpdateMarketplace)
    end)
    addInput(mp, "Robux Amount", "0", function(v)
        states.spoofer_state.marketplace.robux_amount = tonumber(v) or 0
        pcall(UpdateMarketplace)
    end)
    addToggle(mp, "Fake Gifting", false, function(v)
        states.spoofer_state.marketplace.fake_gifting = v
        pcall(UpdateMarketplace)
    end)

    local cur = groupbox(tabSpoof.Left, "Currency")
    for _, x in ipairs({
        {"Weapon Keys", "weapon_keys", "weapon_keys_value"},
        {"Unlock Tokens", "unlock_tokens", "unlock_tokens_value"},
        {"Event Currency", "event_currency", "event_currency_value"},
        {"Glory", "glory", "glory_value"},
        {"Skin Tickets", "skin_tickets", "skin_tickets_value"},
    }) do
        addToggle(cur, "Spoof " .. x[1], false, function(v)
            states.spoofer_state.currency[x[2]] = v
            pcall(UpdateCurrencyOptions)
        end)
        addInput(cur, x[1], "0", function(v)
            states.spoofer_state.currency[x[3]] = tonumber(v) or 0
            pcall(UpdateCurrencyOptions)
        end)
    end

    local dh = groupbox(tabSpoof.Right, "Duel History")
    addInput(dh, "Match Index", "1", function(v)
        states.spoofer_state.duel_history.match_index = tonumber(v) or 1
        pcall(UpdateDuelHistory)
    end)
    addDropdown(dh, "Mode", {"Ranked", "Casual"}, "Ranked", function(v)
        states.spoofer_state.duel_history.mode = v
        pcall(UpdateDuelHistory)
    end)
    addDropdown(dh, "Result", {"Victory", "Defeat"}, "Victory", function(v)
        states.spoofer_state.duel_history.result = v
        pcall(UpdateDuelHistory)
    end)
    addInput(dh, "Map", "Factory", function(v)
        states.spoofer_state.duel_history.map = v
        pcall(UpdateDuelHistory)
    end)
    addInput(dh, "Kills", "15", function(v)
        states.spoofer_state.duel_history.kills = tonumber(v) or 0
        pcall(UpdateDuelHistory)
    end)
    addInput(dh, "Deaths", "2", function(v)
        states.spoofer_state.duel_history.deaths = tonumber(v) or 0
        pcall(UpdateDuelHistory)
    end)
    addInput(dh, "ELO", "2000", function(v)
        states.spoofer_state.duel_history.elo = tonumber(v) or 0
        pcall(UpdateDuelHistory)
    end)
    addButton(dh, "Apply Duel History", function()
        pcall(UpdateDuelHistory)
        Notify("Duel", "Applied")
    end)
end

-- UNLOCKER
do
    local types = {"Skin", "Wrap", "Charm", "Finisher", "Emote"}
    local rarities = {"Common", "Rare", "Epic", "Legendary", "Mythic", "Contraband"}
    local u = groupbox(tabUnlock.Left, "Unlock")
    addDropdown(u, "Type", types, "Skin", function(v)
        states.skinchanger_state.general_unlocker.unlock_type = v
    end)
    addDropdown(u, "Rarity", rarities, "Common", function(v)
        states.skinchanger_state.general_unlocker.unlock_rarity = v
    end)
    addButton(u, "Unlock Selected Rarity", function()
        pcall(UnlockSelectedRarity); Notify("Unlock", "Rarity OK")
    end)
    addButton(u, "Unlock All of Type", function()
        pcall(UnlockAllOfType); Notify("Unlock", "Type OK")
    end)
    addButton(u, "Lock All of Type", function()
        pcall(LockAllOfType); Notify("Unlock", "Locked")
    end)

    local sp = groupbox(tabUnlock.Left, "Specific")
    addDropdown(sp, "Type", types, "Skin", function(v)
        states.skinchanger_state.specific_unlocker.unlock_type = v
    end)
    addInput(sp, "Cosmetic", "", function(v)
        states.skinchanger_state.specific_unlocker.cosmetic_name = v
    end)
    addInput(sp, "Weapon", "", function(v)
        states.skinchanger_state.specific_unlocker.weapon_name = v
    end)
    addButton(sp, "Unlock Specific", function()
        pcall(UnlockSpecific); Notify("Unlock", "OK")
    end)
    addButton(sp, "Unlock Weapon Cosmetics", function()
        pcall(UnlockSpecificWeapon); Notify("Unlock", "Weapon OK")
    end)

    local eq = groupbox(tabUnlock.Right, "Equip")
    addDropdown(eq, "Type", types, "Skin", function(v)
        states.skinchanger_state.equip_unlocker.unlock_type = v
    end)
    addInput(eq, "Cosmetic", "", function(v)
        states.skinchanger_state.equip_unlocker.cosmetic_name = v
    end)
    addInput(eq, "Weapon", "", function(v)
        states.skinchanger_state.equip_unlocker.weapon_name = v
    end)
    addToggle(eq, "Inverted", false, function(v)
        states.skinchanger_state.equip_unlocker.inverted = v
    end)
    addButton(eq, "Apply", function()
        pcall(EquipApply); Notify("Equip", "Applied")
    end)
    addButton(eq, "Apply All Weapons", function()
        pcall(EquipApplyAll); Notify("Equip", "All")
    end)

    local bulk = groupbox(tabUnlock.Right, "Bulk")
    addButton(bulk, "Unlock All Cosmetics", function()
        pcall(UnlockAll); Notify("Unlock", "All cosmetics")
    end)
    addButton(bulk, "Unlock All Weapons", function()
        pcall(UnlockAllWeapons); Notify("Unlock", "Weapons")
    end)
    addButton(bulk, "Unlock EVERYTHING", function()
        pcall(UnlockAllCosmeticsSkinchanger); Notify("Unlock", "Everything")
    end)
    addButton(bulk, "Lock All", function()
        pcall(LockAll); pcall(LockAllEmotes); pcall(ReplicateSkinchanger)
        Notify("Unlock", "Locked")
    end)

    local em = groupbox(tabUnlock.Right, "Emotes")
    addInput(em, "Emote / Anim ID", "Do That Thang", function(v)
        states.emote_state = states.emote_state or {}
        states.emote_state.selected = v
    end)
    addSlider(em, "Speed", 0.25, 3, 1, function(v)
        states.emote_state = states.emote_state or {}
        states.emote_state.speed = v
    end)
    addButton(em, "Equip Emote", function()
        local n = states.emote_state and states.emote_state.selected
        local ok = EquipEmote(n)
        Notify("Emote", ok and "Equipped" or "Fail")
    end)
    addButton(em, "Play Emote", function()
        local n = states.emote_state and states.emote_state.selected
        Notify("Emote", PlayEmote(n) and "Playing" or "Fail — usa Anim ID")
    end)
    addButton(em, "Stop Emote", function()
        pcall(StopEmotePlay); Notify("Emote", "Stop")
    end)
    addButton(em, "Unlock All Emotes", function()
        Notify("Emote", "Unlocked " .. tostring(UnlockAllEmotes()))
    end)
end

-- VISUALS
do
    local v = groupbox(tabVisual.Left, "Handicap")
    addToggle(v, "Enable Handicaps", false, function(val)
        states.handicap_state = states.handicap_state or {}
        states.handicap_state.enabled = val
        pcall(function()
            local ps = LocalPlayer:FindFirstChild("PlayerScripts")
            local controllers = ps and ps:FindFirstChild("Controllers")
            local debugMod = controllers and controllers:FindFirstChild("DebugController")
            if not debugMod then return end
            require(debugMod):SetHandicapsEnabled(val)
        end)
    end)

    local ratio = groupbox(tabVisual.Left, "Stretched")
    addToggle(ratio, "Enable Stretched", false, function(val)
        local st = getgenv()._Matcha_stretch
        if not st then return end
        if val then
            if st._start then st._start() end
            Notify("Stretched", "ON · " .. tostring(st.value or 0.81))
        else
            if st._stop then st._stop() end
            Notify("Stretched", "OFF")
        end
    end)
    addSlider(ratio, "Resolution", 0.5, 1.2, 0.81, function(val)
        local st = getgenv()._Matcha_stretch
        if st then
            st.value = val
            getgenv().Resolution = getgenv().Resolution or {}
            getgenv().Resolution[".gg/scripters"] = val
        end
    end)
    addLabel(ratio, "Source: Camera CFrame matrix")
    addLabel(ratio, "0.81 = stretched tipico")

    local sky = groupbox(tabVisual.Left, "Skybox")
    addToggle(sky, "Enabled", false, function(val)
        skyboxData.enabled = val
        if val then ApplySkybox()
        else
            if skyboxConn then skyboxConn:Disconnect(); skyboxConn = nil end
        end
    end)
    addDropdown(sky, "Skybox", SkyboxNames, "None", function(val)
        skyboxData.selected = val
        ApplySkybox()
    end)
    addSlider(sky, "Stars", 0, 5000, 3000, function(val)
        skyboxData.star_count = val
        ApplySkybox()
    end)
    addSlider(sky, "Sun Size", 0, 50, 20, function(val)
        skyboxData.sun_size = val
        ApplySkybox()
    end)
    addSlider(sky, "Moon Size", 0, 30, 11, function(val)
        skyboxData.moon_size = val
        ApplySkybox()
    end)
    addToggle(sky, "Auto Rotate", false, function(val)
        skyboxData.auto_rotate = val
        if val then
            if skyboxConn then skyboxConn:Disconnect() end
            skyboxConn = RunService.RenderStepped:Connect(function(dt)
                if not skyboxData.enabled or not skyboxData.auto_rotate then return end
                local s = EnsureSky()
                if s then
                    s.SkyboxOrientation = Vector3.new(0, (s.SkyboxOrientation.Y + skyboxData.rotate_speed * dt) % 360, 0)
                end
            end)
        else
            if skyboxConn then skyboxConn:Disconnect(); skyboxConn = nil end
        end
    end)
    addSlider(sky, "Rotate Speed", 1, 100, 10, function(val)
        skyboxData.rotate_speed = val
    end)

    local hs = groupbox(tabVisual.Left, "Hit Sounds")
    addToggle(hs, "Enabled", false, function(val)
        states.hitsound_state = states.hitsound_state or {}
        states.hitsound_state.enabled = val
    end)
    addDropdown(hs, "Sound", {
        "None","Bameware","Bell","Bubble","Click","Pop","Rust","Fart","Big","Vine","Bruh","Skeet","Neverlose","Fatality","Bonk","Minecraft"
    }, "None", function(val)
        states.hitsound_state = states.hitsound_state or {}
        states.hitsound_state.sound_value = val
    end)
    addInput(hs, "Asset ID", "", function(val)
        states.hitsound_state = states.hitsound_state or {}
        states.hitsound_state.sound_asset_id = val
    end)
    addSlider(hs, "Volume", 1, 100, 50, function(val)
        states.hitsound_state = states.hitsound_state or {}
        states.hitsound_state.sound_volume = val
    end)
    addSlider(hs, "Pitch", 1, 100, 10, function(val)
        states.hitsound_state = states.hitsound_state or {}
        states.hitsound_state.sound_pitch = val
    end)
    addButton(hs, "Preview Sound", function()
        local s = states.hitsound_state
        if not s or not s.sound_value or s.sound_value == "None" then return end
        local map = {
            Bameware="rbxassetid://130791763", Bell="rbxassetid://146680539", Bubble="rbxassetid://146665237",
            Click="rbxassetid://146633140", Pop="rbxassetid://146681941", Rust="rbxassetid://146692956",
            Fart="rbxassetid://130791763", Big="rbxassetid://130792279", Vine="rbxassetid://146606035",
            Bruh="rbxassetid://146657737", Skeet="rbxassetid://130791733", Neverlose="rbxassetid://146633485",
            Fatality="rbxassetid://146665147", Bonk="rbxassetid://146661497", Minecraft="rbxassetid://146649495",
        }
        local id = map[s.sound_value]
        if s.sound_asset_id and s.sound_asset_id ~= "" then id = "rbxassetid://"..tostring(s.sound_asset_id) end
        if not id then return end
        local snd = Instance.new("Sound")
        snd.SoundId = id
        snd.Volume = (s.sound_volume or 50) / 100
        snd.PlaybackSpeed = (s.sound_pitch or 10) / 10
        snd.Parent = SoundService
        snd:Play()
        snd.Ended:Connect(function() snd:Destroy() end)
    end)

    
-- AUTO CLICKER (universal)
do
    local cl = groupbox(tabCombat and tabCombat.Left or tabVisual.Left, "Auto Clicker")
    addToggle(cl, "Enable Clicker", false, function(val)
        local c = getgenv()._Vnix_clicker
        if not c then return end
        if val then c._start() else c._stop() end
        Notify("Clicker", val and "ON" or "OFF")
    end)
    addDropdown(cl, "Toggle Key", {
        "E","Q","R","T","Y","U","F","G","H","Z","X","C","V","B",
        "LeftAlt","LeftControl","LeftShift","CapsLock","MouseButton3"
    }, "E", function(val)
        local c = getgenv()._Vnix_clicker
        if c then c.toggle_key = val end
        Notify("Clicker", "Tecla: " .. tostring(val))
    end)
    addSlider(cl, "CPS", 1, 30, 12, function(val)
        local c = getgenv()._Vnix_clicker
        if c then c.cps = val end
    end)
    addDropdown(cl, "Button", {"Left", "Right", "Both"}, "Left", function(val)
        local c = getgenv()._Vnix_clicker
        if c then c.mode = val end
    end)
    addToggle(cl, "Only when focused", true, function(val)
        local c = getgenv()._Vnix_clicker
        if c then c.only_when_focused = val end
    end)
    addLabel(cl, "Tecla = ON/OFF del clicker")
end

-- BLOX STRIKE · Rage / FastFire / Spin / Desync
do
    local parent = tabCombat and tabCombat.Right or (tabVisual and tabVisual.Right)
    if parent then
        local b = groupbox(parent, "Blox Strike Rage")
        addToggle(b, "Ragebot", false, function(val)
            local bs = getgenv()._BS
            if not bs then return end
            if val then bs._rageStart() else bs._rageStop() end
            Notify("BS Rage", val and "ON" or "OFF")
        end)
        addToggle(b, "Team Check", true, function(val)
            local bs = getgenv()._BS
            if bs then bs.rage_team = val end
        end)
        addToggle(b, "TP / Stick Head", true, function(val)
            local bs = getgenv()._BS
            if bs then bs.rage_tp = val end
        end)
        addSlider(b, "Fire Rate", 0.01, 0.2, 0.05, function(val)
            local bs = getgenv()._BS
            if bs then bs.rage_rate = val end
        end)
        addToggle(b, "Desync (no ves TP)", false, function(val)
            local bs = getgenv()._BS
            if bs then bs._desync(val) end
            Notify("Desync", val and "ON" or "OFF")
        end)
        addSlider(b, "Desync Height", 1, 8, 3, function(val)
            local bs = getgenv()._BS
            if bs then bs.desync_y = val end
        end)
        addToggle(b, "Spin Bot", false, function(val)
            local bs = getgenv()._BS
            if not bs then return end
            if val then bs._spinStart() else bs._spinStop() end
            Notify("Spin", val and "ON" or "OFF")
        end)
        addSlider(b, "Spin Speed", 5, 80, 25, function(val)
            local bs = getgenv()._BS
            if bs then bs.spin_speed = val end
        end)
        addToggle(b, "Fast Fire", false, function(val)
            local bs = getgenv()._BS
            if not bs then return end
            if val then bs._ffStart() else bs._ffStop() end
            Notify("FastFire", val and "ON · equipa arma" or "OFF")
        end)
        addSlider(b, "FastFire Rate", 0.01, 0.12, 0.02, function(val)
            local bs = getgenv()._BS
            if bs then bs.fire_rate = val end
        end)
        addLabel(b, "FF = solo con click izquierdo (no bloquea UI)")
        addLabel(b, "Mantén M1 para disparar rápido")
    end
end

-- UNIVERSAL AIMBOT (shako-style)
do
    local a = groupbox(tabCombat and tabCombat.Left or tabVisual.Left, "Aimbot")
    addToggle(a, "Enable Aimbot", false, function(val)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if not ac then return end
        if val then ac._start() else ac._stop() end
        Notify("Aimbot", val and "ON · hold RMB" or "OFF")
    end)
    addToggle(a, "Team Check", true, function(val)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if ac then ac.team_check = val end
    end)
    addToggle(a, "Wall Check", false, function(val)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if ac then ac.wall_check = val end
    end)
    addToggle(a, "Show FOV", true, function(val)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if ac then ac.show_fov = val end
    end)
    addToggle(a, "FOV Fill", true, function(val)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if ac then ac.fov_fill = val end
    end)
    addSlider(a, "FOV", 40, 600, 180, function(val)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if ac then ac.fov = val end
    end)
    addSlider(a, "Smooth %", 1, 100, 35, function(val)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if ac then
            ac.smooth = math.clamp(val / 100, 0.01, 1)
        end
    end)
    addDropdown(a, "Hit Part", {"Head", "HumanoidRootPart", "UpperTorso"}, "Head", function(val)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if ac then ac.hitpart = val end
    end)
    addColorPicker(a, "FOV Color", Color3.fromRGB(255, 105, 180), function(c)
        local ac = getgenv()._Vnix_aim or getgenv()._Vnix_aimbot
        if ac then ac.fov_color = c end
    end)
    addLabel(a, "Hold Right Click to aim")
end


-- RAGE (Rivals: FighterController + UseItem desync)
    local r = groupbox(tabVisual.Right, "Rage")
    addToggle(r, "Ragebot", false, function(val)
        local rc = getgenv()._Vnix_rage
        if not rc then return end
        if val then
            if rc._start then rc._start() end
            Notify("Rage", "ON · UseItem + desync")
        else
            if rc._stop then rc._stop() end
            Notify("Rage", "OFF")
        end
    end)
    addToggle(r, "Desync", true, function(val)
        local rc = getgenv()._Vnix_rage
        if rc then rc.desync = val and true or false end
    end)
    addToggle(r, "Team Check (2v2-5v5)", true, function(val)
        local rc = getgenv()._Vnix_rage
        if rc then rc.team_check = val and true or false end
    end)
    addToggle(r, "Force FFA", false, function(val)
        local rc = getgenv()._Vnix_rage
        if rc then rc.force_ffa = val and true or false end
    end)
    addDropdown(r, "Weapon Slot", {"Primary", "Secondary", "Melee"}, "Primary", function(val)
        local rc = getgenv()._Vnix_rage
        if rc then rc.weapon_slot = val end
    end)
    addSlider(r, "Max Distance", 50, 1000, 500, function(val)
        local rc = getgenv()._Vnix_rage
        if rc then rc.max_dist = val end
    end)
    addLabel(r, "Source: UseItem + EncodeCFrame")
    addLabel(r, "Activa EN RONDA · auto equip")
end

-- ESP
do
    local e = groupbox(tabESP.Left, "ESP")
    local function refreshESP()
        local es = getgenv()._Vnix_esp
        if not es then return end
        if es.enabled and es._start then es._start()
        elseif (not es.enabled) and es._stop then es._stop() end
    end
    addToggle(e, "Enable ESP", false, function(v) local es=getgenv()._Vnix_esp; if es then es.enabled=v; refreshESP() end end)
    addToggle(e, "Team Check", false, function(v) local es=getgenv()._Vnix_esp; if es then es.team_check=v end end)
    addToggle(e, "Box", true, function(v) local es=getgenv()._Vnix_esp; if es then es.box=v end end)
    addDropdown(e, "Box Style", {"Full", "Corner"}, "Full", function(v) local es=getgenv()._Vnix_esp; if es then es.box_style=v end end)
    addToggle(e, "Box Fill", false, function(v) local es=getgenv()._Vnix_esp; if es then es.box_fill=v end end)
    addToggle(e, "3D Box", false, function(v) local es=getgenv()._Vnix_esp; if es then es.box3d=v end end)
    addToggle(e, "Health Bar", true, function(v) local es=getgenv()._Vnix_esp; if es then es.healthbar=v end end)
    addToggle(e, "Health Text", true, function(v) local es=getgenv()._Vnix_esp; if es then es.health=v end end)
    addToggle(e, "Name", true, function(v) local es=getgenv()._Vnix_esp; if es then es.name=v end end)
    addToggle(e, "Distance / Weapon", true, function(v) local es=getgenv()._Vnix_esp; if es then es.info=v; es.distance=v end end)
    addToggle(e, "Skeleton", false, function(v) local es=getgenv()._Vnix_esp; if es then es.skeleton=v end end)
    addToggle(e, "Tracers", false, function(v) local es=getgenv()._Vnix_esp; if es then es.tracers=v end end)
    addDropdown(e, "Tracer Origin", {"Bottom", "Center", "Top", "Mouse"}, "Bottom", function(v) local es=getgenv()._Vnix_esp; if es then es.tracer_origin=v end end)
    addToggle(e, "Head Dot", false, function(v) local es=getgenv()._Vnix_esp; if es then es.headdot=v end end)
    addToggle(e, "Off-screen Arrows", false, function(v) local es=getgenv()._Vnix_esp; if es then es.arrows=v end end)
    addToggle(e, "Chams", false, function(v) local es=getgenv()._Vnix_esp; if es then es.chams=v end end)
    addToggle(e, "Chams Through Walls", true, function(v) local es=getgenv()._Vnix_esp; if es then es.chams_walls=v end end)
    addToggle(e, "Visibility Check", false, function(v) local es=getgenv()._Vnix_esp; if es then es.vis_check=v end end)
    addSlider(e, "Max Distance", 100, 5000, 2000, function(v) local es=getgenv()._Vnix_esp; if es then es.max_distance=v end end)
    
    addLabel(e, "— Colores ESP (no afecta healthbar) —")
    addColorPicker(e, "Color Global (todo)", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp
        if not es then return end
        es.box_color = col
        es.name_color = col
        es.info_color = col
        es.skeleton_color = col
        es.tracer_color = col
        es.headdot_color = col
        es.arrow_color = col
        es.chams_fill = col
        -- outline un poco más claro
        es.chams_outline = Color3.new(
            math.min(col.R + 0.25, 1),
            math.min(col.G + 0.25, 1),
            math.min(col.B + 0.25, 1)
        )
    end)
    addColorPicker(e, "Box Color", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp; if es then es.box_color = col end
    end)
    addColorPicker(e, "Name Color", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp; if es then es.name_color = col end
    end)
    addColorPicker(e, "Info / Distance", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp; if es then es.info_color = col end
    end)
    addColorPicker(e, "Skeleton Color", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp; if es then es.skeleton_color = col end
    end)
    addColorPicker(e, "Tracer Color", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp; if es then es.tracer_color = col end
    end)
    addColorPicker(e, "Head Dot Color", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp; if es then es.headdot_color = col end
    end)
    addColorPicker(e, "Arrow Color", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp; if es then es.arrow_color = col end
    end)
    addColorPicker(e, "Chams Fill", Color3.fromRGB(255, 105, 180), function(col)
        local es = getgenv()._Vnix_esp; if es then es.chams_fill = col end
    end)
    addColorPicker(e, "Chams Outline", Color3.fromRGB(255, 255, 255), function(col)
        local es = getgenv()._Vnix_esp; if es then es.chams_outline = col end
    end)
    addLabel(e, "Healthbar = verde/rojo fijo (no cambia)")
    addLabel(e, "Click color = presets · click der = rueda")
end


-- ==================== EMOTES (estilo shako.win) ====================
local MATCHA_EMOTES = {
    ["Floss"] = "5917459365",
    ["Orange Justice"] = "3934987096",
    ["Take The L"] = "3333499508",
    ["Default Dance"] = "4265725525",
    ["Electro Shuffle"] = "3489178132",
    ["Fresh"] = "3183986796",
    ["Robot"] = "4555808220",
    ["Hype"] = "3360689775",
    ["Dab"] = "3333499505",
    ["Laugh"] = "3333430984",
    ["Wave"] = "507770239",
    ["Point"] = "507770453",
    ["Cheer"] = "507770677",
    ["Gangnam"] = "5918726674",
    ["Shuffle"] = "434870490",
    ["Breakdance"] = "5918726674",
    ["Twist"] = "3333492741",
    ["Zombie"] = "3489174152",
}
local EMOTE_NAMES = {}
for n in pairs(MATCHA_EMOTES) do table.insert(EMOTE_NAMES, n) end
table.sort(EMOTE_NAMES)

local emote_cfg = {
    enabled = false,
    name = "Floss",
    id = "5917459365",
    speed = 1,
    keep = true,
}
getgenv()._Matcha_emote = emote_cfg

local emote_track = nil
local emote_conn = nil
local emote_cache = {}

local function emoteGetAnimator(hum)
    if not hum then return nil end
    local a = hum:FindFirstChildOfClass("Animator")
    if not a then
        a = Instance.new("Animator")
        a.Parent = hum
    end
    return a
end

local function emoteResolve(id)
    id = tostring(id or ""):gsub("%D", "")
    if id == "" then return nil end
    if emote_cache[id] and emote_cache[id].Parent ~= nil then
        return emote_cache[id]
    end
    -- GetObjects
    local ok, objs = pcall(function()
        return game:GetObjects("rbxassetid://" .. id)
    end)
    if ok and type(objs) == "table" then
        for _, o in ipairs(objs) do
            if o:IsA("Animation") then
                emote_cache[id] = o
                return o
            end
            local found = o:FindFirstChildWhichIsA("Animation", true)
            if found then
                emote_cache[id] = found
                return found
            end
        end
    end
    local a = Instance.new("Animation")
    a.Name = "MatchaEmote_" .. id
    a.AnimationId = "rbxassetid://" .. id
    emote_cache[id] = a
    return a
end

local function emoteLoadTrack(animator, id)
    local anim = emoteResolve(id)
    if not anim or not animator then return nil end
    local ok, track = pcall(function() return animator:LoadAnimation(anim) end)
    if ok and track then return track end
    local hum = animator.Parent
    if hum and hum:IsA("Humanoid") then
        ok, track = pcall(function() return hum:LoadAnimation(anim) end)
        if ok and track then return track end
    end
    return nil
end

local function emoteStopCompeting(animator, keep)
    pcall(function()
        for _, t in ipairs(animator:GetPlayingAnimationTracks()) do
            if t ~= keep then
                local pr = t.Priority
                if pr == Enum.AnimationPriority.Movement
                    or pr == Enum.AnimationPriority.Idle
                    or pr == Enum.AnimationPriority.Action
                    or pr == Enum.AnimationPriority.Action2
                    or pr == Enum.AnimationPriority.Action3 then
                    pcall(function() t:Stop(0) end)
                end
            end
        end
    end)
end

local function EmoteStop()
    emote_cfg.enabled = false
    if emote_conn then pcall(function() emote_conn:Disconnect() end); emote_conn = nil end
    if emote_track then pcall(function() emote_track:Stop(0.1) end); emote_track = nil end
end

local function EmotePlay(nameOrId, speed)
    local char = LocalPlayer.Character
    if not char then return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return false end
    local animator = emoteGetAnimator(hum)
    if not animator then return false end

    local id = tostring(nameOrId or "")
    if MATCHA_EMOTES[id] then
        emote_cfg.name = id
        id = MATCHA_EMOTES[id]
    end
    id = id:gsub("%D", "")
    if id == "" then
        id = MATCHA_EMOTES[emote_cfg.name] or emote_cfg.id or "5917459365"
        id = tostring(id):gsub("%D", "")
    end
    emote_cfg.id = id
    emote_cfg.speed = tonumber(speed) or emote_cfg.speed or 1
    emote_cfg.enabled = true

    if emote_track then pcall(function() emote_track:Stop(0) end); emote_track = nil end

    local track = emoteLoadTrack(animator, id)
    if not track then
        task.wait(0.1)
        track = emoteLoadTrack(animator, id)
    end
    if not track then
        warn("[Matcha Emote] no cargó id=" .. id)
        return false
    end

    pcall(function()
        track.Priority = Enum.AnimationPriority.Action4
        track.Looped = true
        track:Play(0.05, 1, emote_cfg.speed)
        track:AdjustWeight(1, 0)
    end)
    emote_track = track
    emoteStopCompeting(animator, track)

    if emote_conn then pcall(function() emote_conn:Disconnect() end) end
    emote_conn = RunService.Heartbeat:Connect(function()
        if not emote_cfg.enabled or not emote_cfg.keep then return end
        local ch = LocalPlayer.Character
        local h = ch and ch:FindFirstChildOfClass("Humanoid")
        local an = h and emoteGetAnimator(h)
        if not an then return end
        if not emote_track or not emote_track.IsPlaying then
            local nt = emoteLoadTrack(an, emote_cfg.id)
            if nt then
                pcall(function()
                    nt.Priority = Enum.AnimationPriority.Action4
                    nt.Looped = true
                    nt:Play(0, 1, emote_cfg.speed or 1)
                end)
                emote_track = nt
                emoteStopCompeting(an, nt)
            end
        else
            pcall(function()
                emote_track:AdjustSpeed(emote_cfg.speed or 1)
                emoteStopCompeting(an, emote_track)
            end)
        end
    end)
    return true
end

emote_cfg._play = EmotePlay
emote_cfg._stop = EmoteStop
emote_cfg._list = EMOTE_NAMES
emote_cfg._map = MATCHA_EMOTES

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1)
    if emote_cfg.enabled and emote_cfg.id then
        pcall(function() EmotePlay(emote_cfg.id, emote_cfg.speed) end)
    end
end)

-- ==================== ROBLOX ANIM PACKS (walk/run/jump reales) ====================
getgenv()._Vnix_anims = getgenv()._Vnix_anims or {
    enabled = false,
    selected = "Zombie",
    track = nil,
    applied = false,
}

-- Packs completos R15: idle / walk / run / jump / climb / fall
local ANIM_PACKS = {
    Zombie = {
        idle = 616158929,
        walk = 616163682,
        run = 616163682,
        jump = 616161997,
        climb = 616156119,
        fall = 616157476,
    },
    Ninja = {
        idle = 656117401,
        walk = 656121766,
        run = 656118852,
        jump = 656117878,
        climb = 656118341,
        fall = 656115606,
    },
    Robot = {
        idle = 616136211,
        walk = 616140816,
        run = 616139451,
        jump = 616139089,
        climb = 616133816,
        fall = 616134815,
    },
    Superhero = {
        idle = 616111295,
        walk = 616115533,
        run = 616112941,
        jump = 616113199,
        climb = 616113939,
        fall = 616112266,
    },
    Levitation = {
        idle = 616006778,
        walk = 616010546,
        run = 616010546,
        jump = 616008936,
        climb = 616005863,
        fall = 616005863,
    },
    Stylish = {
        idle = 616136781,
        walk = 616140816,
        run = 616140816,
        jump = 616139089,
        climb = 616133816,
        fall = 616134815,
    },
    Elder = {
        idle = 616138050,
        walk = 616142078,
        run = 616140816,
        jump = 616139089,
        climb = 616133816,
        fall = 616134815,
    },
    Toy = {
        idle = 782841498,
        walk = 782843345,
        run = 782842708,
        jump = 782847020,
        climb = 782843345,
        fall = 782846423,
    },
    Bubbly = {
        idle = 910004836,
        walk = 910034870,
        run = 910025107,
        jump = 910016857,
        climb = 910004836,
        fall = 910016857,
    },
    Knight = {
        idle = 657595757,
        walk = 657552124,
        run = 657564596,
        jump = 657552124,
        climb = 657552124,
        fall = 657552124,
    },
    Mage = {
        idle = 707742142,
        walk = 707897309,
        run = 707861613,
        jump = 707853694,
        climb = 707853694,
        fall = 707853694,
    },
    Pirate = {
        idle = 750781874,
        walk = 750785693,
        run = 750782770,
        jump = 750782770,
        climb = 750785693,
        fall = 750782770,
    },
    Vampire = {
        idle = 1083445855,
        walk = 1083462077,
        run = 1083466986,
        jump = 1083455352,
        climb = 1083462077,
        fall = 1083455352,
    },
    Werewolf = {
        idle = 1083021282,
        walk = 1083049820,
        run = 1083054349,
        jump = 1083046690,
        climb = 1083049820,
        fall = 1083046690,
    },
    Astronaut = {
        idle = 891621366,
        walk = 891636393,
        run = 891627522,
        jump = 891627522,
        climb = 891636393,
        fall = 891627522,
    },
}

local PACK_NAMES = {}
for n in pairs(ANIM_PACKS) do table.insert(PACK_NAMES, n) end
table.sort(PACK_NAMES)

local function setAnimId(obj, id)
    if not obj or not id then return end
    pcall(function()
        if obj:IsA("Animation") then
            obj.AnimationId = "rbxassetid://" .. tostring(id)
        end
    end)
end

local function applyPackToAnimate(char, pack)
    if not char or not pack then return false end
    local animate = char:FindFirstChild("Animate")
    if not animate then return false end

    -- estructura tipica R15 Animate
    local map = {
        idle = pack.idle,
        walk = pack.walk,
        run = pack.run,
        jump = pack.jump,
        climb = pack.climb,
        fall = pack.fall,
    }

    for folderName, animId in pairs(map) do
        local folder = animate:FindFirstChild(folderName)
        if folder then
            for _, child in ipairs(folder:GetChildren()) do
                if child:IsA("Animation") then
                    setAnimId(child, animId)
                end
            end
        end
    end

    -- algunos games usan StringValues con AnimationId
    for _, d in ipairs(animate:GetDescendants()) do
        if d:IsA("StringValue") and d.Name:lower():find("anim") then
            -- skip
        end
        if d:IsA("Animation") then
            local parentName = d.Parent and d.Parent.Name:lower() or ""
            local n = d.Name:lower()
            if parentName == "idle" or n:find("idle") then setAnimId(d, pack.idle)
            elseif parentName == "walk" or n:find("walk") then setAnimId(d, pack.walk)
            elseif parentName == "run" or n:find("run") then setAnimId(d, pack.run)
            elseif parentName == "jump" or n:find("jump") then setAnimId(d, pack.jump)
            elseif parentName == "climb" or n:find("climb") then setAnimId(d, pack.climb)
            elseif parentName == "fall" or n:find("fall") then setAnimId(d, pack.fall)
            end
        end
    end
    return true
end

-- controller: reproduce la anim correcta segun movimiento (si Animate no existe)
local anim_conn = nil
local current_track = nil
local function stopController()
    if anim_conn then pcall(function() anim_conn:Disconnect() end) anim_conn = nil end
    if current_track then pcall(function() current_track:Stop(0.1) end) current_track = nil end
end

local function loadTrack(hum, id)
    if not hum or not id then return nil end
    local animator = hum:FindFirstChildOfClass("Animator")
    if not animator then
        animator = Instance.new("Animator")
        animator.Parent = hum
    end
    local anim = Instance.new("Animation")
    anim.AnimationId = "rbxassetid://" .. tostring(id)
    local ok, track = pcall(function() return animator:LoadAnimation(anim) end)
    if ok and track then
        track.Looped = true
        track.Priority = Enum.AnimationPriority.Movement
        return track
    end
    return nil
end

local function startController(pack)
    stopController()
    local tracks = {}
    local lastKey = nil

    anim_conn = RunService.RenderStepped:Connect(function()
        local st = getgenv()._Vnix_anims
        if not st or not st.enabled then return end
        local char = LocalPlayer.Character
        if not char then return end
        local hum = char:FindFirstChildOfClass("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hum or not hrp then return end

        local state = hum:GetState()
        local speed = hrp.AssemblyLinearVelocity * Vector3.new(1, 0, 1)
        local spd = speed.Magnitude
        local key = "idle"
        if state == Enum.HumanoidStateType.Jumping then
            key = "jump"
        elseif state == Enum.HumanoidStateType.Freefall then
            key = "fall"
        elseif state == Enum.HumanoidStateType.Climbing then
            key = "climb"
        elseif spd > 16 then
            key = "run"
        elseif spd > 0.8 then
            key = "walk"
        else
            key = "idle"
        end

        if key == lastKey and current_track and current_track.IsPlaying then return end
        lastKey = key

        if current_track then pcall(function() current_track:Stop(0.12) end) end
        if not tracks[key] then
            tracks[key] = loadTrack(hum, pack[key])
        end
        current_track = tracks[key]
        if current_track then
            pcall(function()
                current_track:Play(0.12)
                -- velocidad de playback segun movimiento real
                if key == "walk" or key == "run" then
                    current_track:AdjustSpeed(math.clamp(spd / 12, 0.6, 1.6))
                else
                    current_track:AdjustSpeed(1)
                end
            end)
        end
    end)
end

local function animApply(packName)
    local st = getgenv()._Vnix_anims
    local pack = ANIM_PACKS[packName]
    if not pack then return false end
    st.selected = packName
    st.enabled = true
    local char = LocalPlayer.Character
    if not char then return false end

    -- 1) parchear Animate del character (lo ideal)
    local ok = applyPackToAnimate(char, pack)

    -- 2) controller por si el game ignora Animate o lo sobreescribe
    startController(pack)

    -- forzar refresh: a veces hay que tocar WalkSpeed un frame
    pcall(function()
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            local ws = hum.WalkSpeed
            hum.WalkSpeed = ws
        end
    end)
    st.applied = true
    return true
end

local function animStopAll()
    local st = getgenv()._Vnix_anims
    if st then st.enabled = false st.applied = false end
    stopController()
end

getgenv()._Vnix_anims._play = animApply
getgenv()._Vnix_anims._stop = animStopAll
getgenv()._Vnix_anims._list = PACK_NAMES
getgenv()._Vnix_anims._packs = ANIM_PACKS

LocalPlayer.CharacterAdded:Connect(function(char)
    task.wait(1.2)
    local st = getgenv()._Vnix_anims
    if st and st.enabled and st.selected then
        pcall(function() animApply(st.selected) end)
    end
end)

-- UI Character: Animation Packs + Emotes
if tabChar then
    local an = groupbox(tabChar.Right, "Animation Packs")
    local animNames = (getgenv()._Vnix_anims and getgenv()._Vnix_anims._list) or PACK_NAMES or {"Zombie"}
    local selectedAnim = (getgenv()._Vnix_anims and getgenv()._Vnix_anims.selected) or "Zombie"
    addDropdown(an, "Walk Pack", animNames, selectedAnim, function(v)
        selectedAnim = v
        local st = getgenv()._Vnix_anims
        if st then st.selected = v end
    end)
    addButton(an, "Apply Pack", function()
        local st = getgenv()._Vnix_anims
        if not st or not st._play then return end
        local ok = st._play(selectedAnim)
        Notify("Anim", ok and ("Pack: " .. tostring(selectedAnim)) or "Fail")
    end)
    addButton(an, "Remove Pack", function()
        local st = getgenv()._Vnix_anims
        if st and st._stop then st._stop() end
        Notify("Anim", "Default walk")
    end)
    addLabel(an, "Cambia idle/walk/run/jump")

    local em = groupbox(tabChar.Right, "Emotes")
    local emoteNames = (getgenv()._Matcha_emote and getgenv()._Matcha_emote._list) or EMOTE_NAMES or {"Floss"}
    local selectedEmote = "Floss"
    addDropdown(em, "Emote", emoteNames, selectedEmote, function(v)
        selectedEmote = v
        local ec = getgenv()._Matcha_emote
        if ec then
            ec.name = v
            if ec._map and ec._map[v] then ec.id = ec._map[v] end
        end
    end)
    addSlider(em, "Speed", 0.25, 3, 1, function(val)
        local ec = getgenv()._Matcha_emote
        if ec then
            ec.speed = val
            if ec.enabled and ec._play then pcall(ec._play, ec.id or selectedEmote, val) end
        end
    end)
    addToggle(em, "Keep Playing", true, function(val)
        local ec = getgenv()._Matcha_emote
        if ec then ec.keep = val and true or false end
    end)
    addButton(em, "Play Emote", function()
        local ec = getgenv()._Matcha_emote
        if not ec or not ec._play then return end
        local ok = ec._play(selectedEmote, ec.speed)
        Notify("Emote", ok and ("ON · " .. selectedEmote) or "Fail · ID inválido")
    end)
    addButton(em, "Stop Emote", function()
        local ec = getgenv()._Matcha_emote
        if ec and ec._stop then ec._stop() end
        Notify("Emote", "OFF")
    end)
    addInput(em, "Custom Anim ID", "rbxassetid://...", function(val)
        local ec = getgenv()._Matcha_emote
        if not ec then return end
        local id = tostring(val or ""):gsub("%D", "")
        if id ~= "" then
            ec.id = id
            if ec._play then
                local ok = ec._play(id, ec.speed)
                Notify("Emote", ok and ("Custom " .. id) or "Fail")
            end
        end
    end)
    addLabel(em, "Como shako · loop + keep alive")
end


-- MISC
-- MISC
do
    local sb = groupbox(tabMisc.Left, "Slide Boost")
    addToggle(sb, "Enable Slide Boost", false, function(val)
        local s = getgenv()._Matcha_slide
        if not s then return end
        if val then
            s.speed = s.speed or 100
            if s._start then s._start() end
            Notify("Slide", "ON · speed " .. tostring(s.speed))
        else
            if s._stop then s._stop() end
            Notify("Slide", "OFF")
        end
    end)
    addSlider(sb, "Slide Speed", 30, 200, 100, function(val)
        local s = getgenv()._Matcha_slide
        if s then s.speed = val end
    end)
    addLabel(sb, "Mantén slide (C) · velocity boost")
    addLabel(sb, "Source slide boost.txt")

    local c = groupbox(tabMisc.Left, "Config")
    local cfgName, selected = "default", "(ninguna)"
    addInput(c, "Config name", "default", function(v) cfgName = v end)
    local listDrop = addDropdown(c, "Saved", cfgList(), "(ninguna)", function(v) selected = v end)
    addButton(c, "Refresh", function()
        local list = cfgList()
        if listDrop and listDrop.Set then listDrop:Set(list[1] or "(ninguna)") end
        Notify("Config", #list .. " configs")
    end)
    addButton(c, "Save", function() cfgSave(cfgName, false) end)
    addButton(c, "Overwrite", function() cfgSave((selected ~= "(ninguna)" and selected) or cfgName, true) end)
    addButton(c, "Load", function() cfgLoad((selected ~= "(ninguna)" and selected) or cfgName) end)
    addButton(c, "Delete", function()
        if selected ~= "(ninguna)" then
            pcall(function() delfile(CFG_FOLDER .. "/" .. cfgSanitize(selected) .. ".json") end)
            Notify("Config", "Deleted")
        end
    end)
    addButton(c, "Set AutoLoad", function()
        local n = cfgSanitize((selected ~= "(ninguna)" and selected) or cfgName)
        cfgEnsure()
        if not (isfile and isfile(CFG_FOLDER .. "/" .. n .. ".json")) then cfgSave(n, true) end
        pcall(function() writefile(CFG_AUTO, n) end)
        Notify("Config", "AutoLoad → " .. n)
    end)
    addButton(c, "Remove AutoLoad", function()
        pcall(function() if isfile and isfile(CFG_AUTO) then delfile(CFG_AUTO) end end)
        Notify("Config", "AutoLoad off")
    end)

    local an = groupbox(tabMisc.Right, "Roblox Animations")
    local animNames = (getgenv()._Vnix_anims and getgenv()._Vnix_anims._list) or {"Zombie"}
    local selectedAnim = "Zombie"
    addDropdown(an, "Pack", animNames, selectedAnim, function(v)
        selectedAnim = v
        local st = getgenv()._Vnix_anims
        if st then st.selected = v end
    end)
    addButton(an, "Apply Pack", function()
        local st = getgenv()._Vnix_anims
        if not st or not st._play then return end
        local ok = st._play(selectedAnim)
        Notify("Anim", ok and ("Pack: " .. selectedAnim .. " · walk/run/jump") or "Fail")
    end)
    addButton(an, "Remove Pack", function()
        local st = getgenv()._Vnix_anims
        if st and st._stop then st._stop() end
        Notify("Anim", "Default animations")
    end)
    addLabel(an, "Caminar / correr / saltar reales")
    addLabel(an, "Zombie = estilo Da Hood")

    local ui = groupbox(tabMisc.Right, "UI Settings")
    addLabel(ui, "RightShift / Insert toggle")
    addToggle(ui, "Snow when open", true, function(v)
        UI.Settings.Snow = v and true or false
        if not UI.Settings.Snow then
            SnowLayer.Visible = false
            for _, c in ipairs(SnowLayer:GetChildren()) do pcall(function() c:Destroy() end) end
        elseif UI.Open then SnowLayer.Visible = true end
    end)
    addToggle(ui, "Darken background", true, function(v)
        UI.Settings.Darken = v and true or false
        if not UI.Settings.Darken then
            Backdrop.Visible = false
            LightingBlur.Enabled = false
            ColorCorrection.Enabled = false
        elseif UI.Open then applyOpenEffects() end
    end)
    addSlider(ui, "Border transparency", 0, 1, 0, function(v)
        UI.Settings.BorderTransparency = v
        applyBorderStyle()
    end)
    addDropdown(ui, "Border color", {"Black", "Gray", "White", "Red", "None"}, "Black", function(v)
        local map = {
            Black = Color3.fromRGB(0,0,0), Gray = Color3.fromRGB(60,60,65),
            White = Color3.fromRGB(220,220,225), Red = Color3.fromRGB(160,40,50),
            None = Color3.fromRGB(0,0,0),
        }
        UI.Settings.BorderColor = map[v] or map.Black
        if v == "None" then UI.Settings.BorderTransparency = 1 end
        applyBorderStyle()
    end)
    addButton(ui, "Close UI", function() UI:CloseMenu() end)
    addButton(ui, "Unload UI", function()
        UI.Unloaded = true
        UI.Open = false
        getgenv().VnixUI_Loaded = false
        for _, c in pairs(UI.Connections) do pcall(function() c:Disconnect() end) end
        pcall(function() LightingBlur:Destroy() end)
        pcall(function() ColorCorrection:Destroy() end)
        pcall(function()
            Lighting.Ambient = OriginalLighting.Ambient
            Lighting.OutdoorAmbient = OriginalLighting.OutdoorAmbient
            Lighting.Brightness = OriginalLighting.Brightness
            Lighting.ExposureCompensation = OriginalLighting.ExposureCompensation
        end)
        pcall(function() ScreenGui:Destroy() end)
        Notify("Vnix.lol", "UI unloaded")
    end)
end

pcall(function() UI:SelectTab("Combat") end)

getgenv().VnixUI_Destroy = function()
    UI.Unloaded = true
    UI.Open = false
    for _, c in pairs(UI.Connections) do pcall(function() c:Disconnect() end) end
    pcall(function() LightingBlur:Destroy() end)
    pcall(function() ColorCorrection:Destroy() end)
    pcall(function()
        Lighting.Ambient = OriginalLighting.Ambient
        Lighting.OutdoorAmbient = OriginalLighting.OutdoorAmbient
        Lighting.Brightness = OriginalLighting.Brightness
        Lighting.ExposureCompensation = OriginalLighting.ExposureCompensation
    end)
    pcall(function() ScreenGui:Destroy() end)
    getgenv().VnixUI_Loaded = false
end

task.spawn(function()
    task.wait(1.5)
    pcall(function()
        if isfile and isfile(CFG_AUTO) then
            local n = readfile(CFG_AUTO)
            if n and n ~= "" then cfgLoad(n) end
        end
    end)
end)

pcall(function() UI:OpenMenu() end)
print("[Matcha · Universal · Pro] Skin Changer + Spoofer loaded · RightShift toggle")

end

print("[Matcha · Universal · Pro] core loaded, building UI...")
local okUI, errUI = pcall(__VnixBuildUI)
if not okUI then
    warn("[Vnix.lol] UI ERROR: ", errUI)
    pcall(function()
        game:GetService("StarterGui"):SetCore("SendNotification", {
            Title = "Vnix.lol UI Error",
            Text = tostring(errUI):sub(1, 120),
            Duration = 8,
        })
    end)
else
    print("[Matcha · Universal · Pro] UI OK")
end
