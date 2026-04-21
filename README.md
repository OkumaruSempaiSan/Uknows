
local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local DataStoreService = game:GetService("DataStoreService")
local LocalizationService = game:GetService("LocalizationService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Remote = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("Quest")
local BanStore = DataStoreService:GetDataStore("Bans_v3")

local Config = {
    MinAccountAge = 30,
    BypassUserIds = {
        [2061757991] = true
    },
    AdminUserIds = {
        [2061757991] = true
    },
    Webhooks = {
        Ban = "https://discord.com/api/webhooks/1495359977659830382/2X2uATJmR64sq1i8cJsZWmMRnHDTBxTBvxbsdOqHtpWEW6o95XIIpBWNi9ibqF0unt50",
        Join = "https://discord.com/api/webhooks/1495687277769392188/3PHfEUGAVXTGzR6zRc4rIanvo9izGLtKvUeyBpcU8Exg5dUTnvtfu_ZAHBxJWgcPKE1P",
        Leave = "https://discord.com/api/webhooks/1495687277769392188/3PHfEUGAVXTGzR6zRc4rIanvo9izGLtKvUeyBpcU8Exg5dUTnvtfu_ZAHBxJWgcPKE1P",
        Chat = {
            "https://discord.com/api/webhooks/1448166489080336596/nV4-klorFBPe9Y8sFMgE5gFr1FwflCK2Luudlwt3S6U8a4v8zd7UQxRHHvnRC0Xe_L9b",
            "https://discord.com/api/webhooks/1495693634635436093/DSNaL8IiVEu-czTtD1XbGSclrHe8TzBv3VXnL7LZvt2rceHkJlHfldPkUg7Vt29rc5hm"
        }
    }
}

local SessionStore = {}
local Connections = {}
local WebhookState = {
    ChatIndex = 1
}

local function nowUtc()
    return os.date("!%Y-%m-%d %H:%M:%S")
end

local function formatDuration(seconds)
    local h = math.floor(seconds / 3600)
    local m = math.floor((seconds % 3600) / 60)
    local s = seconds % 60
    return string.format("%02d:%02d:%02d", h, m, s)
end

local function getRegion(player)
    local ok, result = pcall(function()
        return LocalizationService:GetCountryRegionForPlayerAsync(player)
    end)
    return ok and result or "Unknown"
end

local function postWebhook(url, payload)
    if type(url) == "table" then
        local count = #url
        if count == 0 then
            return
        end

        task.spawn(function()
            for i = 1, count do
                local index = ((WebhookState.ChatIndex + i - 2) % count) + 1
                local selected = url[index]

                if selected and selected ~= "" and not string.find(selected, "TU_WEBHOOK_") then
                    local success = pcall(function()
                        HttpService:PostAsync(
                            selected,
                            HttpService:JSONEncode(payload),
                            Enum.HttpContentType.ApplicationJson
                        )
                    end)

                    if success then
                        WebhookState.ChatIndex = (index % count) + 1
                        return
                    end
                end
            end
        end)

        return
    end

    if not url or url == "" or string.find(url, "TU_WEBHOOK_") then
        return
    end

    task.spawn(function()
        pcall(function()
            HttpService:PostAsync(
                url,
                HttpService:JSONEncode(payload),
                Enum.HttpContentType.ApplicationJson
            )
        end)
    end)
end

local function makeUserEmbed(title, player, fields)
    return {
        title = title,
        description = player.Name,
        fields = fields
    }
end

local function baseFields(player, region)
    return {
        {name = "UserId", value = tostring(player.UserId), inline = true},
        {name = "Hora", value = nowUtc(), inline = true},
        {name = "Pais", value = region or "Unknown", inline = true}
    }
end

local function getBanData(userId)
    local ok, data = pcall(function()
        return BanStore:GetAsync(tostring(userId))
    end)
    if not ok then
        return nil
    end
    return data
end

local function setBanData(userId, data)
    local ok = pcall(function()
        BanStore:SetAsync(tostring(userId), data)
    end)
    return ok
end

local function isBanned(userId)
    local data = getBanData(userId)
    return data and data.Banned == true, data
end

local function banUser(userId, reason, moderatorId)
    local payload = {
        Banned = true,
        Reason = reason or "No especificada",
        Timestamp = os.time(),
        ModeratorUserId = moderatorId
    }
    return setBanData(userId, payload)
end

local function disconnectPlayerConnections(player)
    local bucket = Connections[player]
    if not bucket then
        return
    end

    for _, connection in ipairs(bucket) do
        if connection.Connected then
            connection:Disconnect()
        end
    end

    Connections[player] = nil
end

local function registerConnection(player, connection)
    if not Connections[player] then
        Connections[player] = {}
    end
    table.insert(Connections[player], connection)
end

local function handleJoin(player)
    if player.AccountAge <= Config.MinAccountAge then
        player:Kick("Cuenta demasiado nueva")
        return
    end

    local banned, banData = isBanned(player.UserId)
    if banned then
        player:Kick("Baneado permanentemente\nMotivo: " .. tostring(banData.Reason or "No especificado"))
        return
    end

    local region = getRegion(player)

    SessionStore[player] = {
        JoinTime = os.time(),
        Region = region
    }

    if not Config.BypassUserIds[player.UserId] then
        postWebhook(Config.Webhooks.Join, {
            content = "Usuario entro",
            embeds = {
                makeUserEmbed("Join", player, baseFields(player, region))
            }
        })
    end

    registerConnection(player, player.Chatted:Connect(function(message)
        local fields = baseFields(player, region)
        table.insert(fields, {
            name = "Mensaje",
            value = tostring(message):sub(1, 1000),
            inline = false
        })

        postWebhook(Config.Webhooks.Chat, {
            content = "Nuevo mensaje",
            embeds = {
                makeUserEmbed("Chat", player, fields)
            }
        })
    end))
end

local function handleLeave(player)
    local session = SessionStore[player]

    if session and not Config.BypassUserIds[player.UserId] then
        local fields = baseFields(player, session.Region)
        table.insert(fields, {
            name = "Duracion",
            value = formatDuration(os.time() - session.JoinTime),
            inline = true
        })

        postWebhook(Config.Webhooks.Leave, {
            content = "Usuario salio",
            embeds = {
                makeUserEmbed("Leave", player, fields)
            }
        })
    end

    SessionStore[player] = nil
    disconnectPlayerConnections(player)
end

Players.PlayerAdded:Connect(handleJoin)
Players.PlayerRemoving:Connect(handleLeave)

Remote.OnServerEvent:Connect(function(player, targetUserId, reason)
    if not Config.AdminUserIds[player.UserId] then
        postWebhook(Config.Webhooks.Ban, {
            content = "Intento no autorizado de ban",
            embeds = {
                makeUserEmbed("Security", player, {
                    {name = "UserId", value = tostring(player.UserId), inline = true},
                    {name = "Hora", value = nowUtc(), inline = true}
                })
            }
        })
        player:Kick("Accion no autorizada")
        return
    end

    targetUserId = tonumber(targetUserId)
    if not targetUserId or targetUserId <= 0 then
        return
    end

    local ok = banUser(targetUserId, reason or "Ban remoto", player.UserId)
    if not ok then
        return
    end

    postWebhook(Config.Webhooks.Ban, {
        content = "Usuario baneado",
        embeds = {{
            title = "Ban ejecutado",
            description = player.Name,
            fields = {
                {name = "ModeratorUserId", value = tostring(player.UserId), inline = true},
                {name = "TargetUserId", value = tostring(targetUserId), inline = true},
                {name = "Hora", value = nowUtc(), inline = true},
                {name = "Motivo", value = tostring(reason or "Ban remoto"), inline = false}
            }
        }}
    })

    local targetPlayer = Players:GetPlayerByUserId(targetUserId)
    if targetPlayer then
        targetPlayer:Kick("Has sido baneado\nMotivo: " .. tostring(reason or "Ban remoto"))
    end
end)
