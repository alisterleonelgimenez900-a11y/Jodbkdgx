-- Module: SlayerCore
-- Descrição: Template seguro para migrar funcionalidades de um loader remoto
-- Uso: salve em ServerScriptService as ModuleScript chamado "SlayerCore"
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

local SlayerCore = {}
SlayerCore.__index = SlayerCore

-- Configurações padrão (imutáveis)
local DEFAULTS = {
    maxHealth = 99999,
    useRemoteEvents = true,
    remoteFolderName = "SlayerRemotes", -- será criado em ReplicatedStorage se inexistente
    debug = true,
    rateLimitSeconds = 1, -- exemplo simples de rate limit por jogador
}

-- Cria nova instância (estado local)
function SlayerCore.new(config)
    config = config or {}
    local self = setmetatable({}, SlayerCore)
    self.config = {
        maxHealth = config.maxHealth or DEFAULTS.maxHealth,
        useRemoteEvents = (config.useRemoteEvents ~= nil) and config.useRemoteEvents or DEFAULTS.useRemoteEvents,
        remoteFolderName = config.remoteFolderName or DEFAULTS.remoteFolderName,
        debug = config.debug or DEFAULTS.debug,
        rateLimitSeconds = config.rateLimitSeconds or DEFAULTS.rateLimitSeconds,
    }
    self._players = {}         -- estado por jogador
    self._remotes = {}         -- referências a RemoteEvents/Functions
    self._connections = {}     -- para desconectar facilmente
    return self
end

-- ======= Utilitários privados =======
local function safeWarn(self, ...)
    if self.config.debug then
        warn("[SlayerCore] ", ...)
    end
end

-- cria/obtém pasta de remotes em ReplicatedStorage
function SlayerCore:_ensureRemotesFolder()
    local folder = ReplicatedStorage:FindFirstChild(self.config.remoteFolderName)
    if not folder then
        folder = Instance.new("Folder")
        folder.Name = self.config.remoteFolderName
        folder.Parent = ReplicatedStorage
    end
    return folder
end

-- cria um RemoteEvent/Function se necessário (retorna objeto)
function SlayerCore:_getOrCreateRemote(name, className)
    className = className or "RemoteEvent"
    local folder = self:_ensureRemotesFolder()
    local obj = folder:FindFirstChild(name)
    if not obj then
        obj = Instance.new(className)
        obj.Name = name
        obj.Parent = folder
    end
    self._remotes[name] = obj
    return obj
end

-- validação básica de jogador (exemplo)
function SlayerCore:_validatePlayer(player)
    return typeof(player) == "Instance" and player:IsA("Player")
end

-- simples rate limiter (por jogador)
function SlayerCore:_checkRateLimit(player)
    local pid = player.UserId
    local now = tick()
    local last = (self._players[pid] and self._players[pid].lastAction) or 0
    if now - last < self.config.rateLimitSeconds then
        return false
    end
    self._players[pid].lastAction = now
    return true
end

-- ======= API pública =======

-- Inicializa: registra eventos e remotes
function SlayerCore:Init()
    safeWarn(self, "Init chamado")
    -- inicializar estado para jogadores já presentes
    for _, player in pairs(Players:GetPlayers()) do
        self:_onPlayerAdded(player)
    end

    -- conectar eventos de player
    table.insert(self._connections, Players.PlayerAdded:Connect(function(p) self:_onPlayerAdded(p) end))
    table.insert(self._connections, Players.PlayerRemoving:Connect(function(p) self:_onPlayerRemoving(p) end))

    -- RemoteEvents (exemplo de criação centralizada)
    if self.config.useRemoteEvents then
        local folder = self:_ensureRemotesFolder()
        self._remotes["RequestAction"] = self:_getOrCreateRemote("RequestAction", "RemoteEvent")
        -- handler server-side
        local conn = self._remotes["RequestAction"].OnServerEvent:Connect(function(player, actionName, ...)
            -- proteger execução com pcall e validações
            local ok, err = pcall(function()
                if not self:_validatePlayer(player) then return end
                if not self:_checkRateLimit(player) then return end
                self:HandleAction(player, actionName, ...)
            end)
            if not ok then
                warn("[SlayerCore] erro ao processar RequestAction:", err)
            end
        end)
        table.insert(self._connections, conn)
    end
end

-- Inicia (efetua ações que dependem de Init ter rodado)
function SlayerCore:Start()
    safeWarn(self, "Start chamado")
    -- faça o que precisa: timers, loops, etc.
    -- Exemplo simples: loop de limpeza se rodando no servidor
    if not self._heartbeatConn then
        self._heartbeatConn = RunService.Heartbeat:Connect(function(dt)
            -- periodic tasks (ex.: salvar, checar timeouts)
        end)
        table.insert(self._connections, self._heartbeatConn)
    end
end

-- Encerra e desconecta tudo
function SlayerCore:Stop()
    safeWarn(self, "Stop chamado - desconectando")
    for _, conn in pairs(self._connections) do
        if conn and typeof(conn.Disconnect) == "function" then
            conn:Disconnect()
        end
    end
    self._connections = {}

    -- limpar estado de jogadores
    self._players = {}
end

-- Manipula ação vinda do cliente (exemplo)
function SlayerCore:HandleAction(player, actionName, ...)
    if not self:_validatePlayer(player) then return end
    -- padrão: tratar ações por nome
    if actionName == "Ping" then
        -- responder via RemoteEvent/Function opcionalmente
        safeWarn(self, "Ping recebido de", player.Name)
        local rem = self._remotes["RequestAction"]
        -- cuidado: não repassar dados sensíveis!
        if rem and rem.FireClient then
            rem:FireClient(player, "Pong")
        end
        return true
    elseif actionName == "UseSkill" then
        -- tratar uso de habilidade (aplicar validações etc)
        local ok, err = pcall(function()
            local skillId = ...
            -- validações
            if type(skillId) ~= "number" then error("skillId inválido") end
            -- lógica de habilidade aqui (ex.: verificar cooldown, consumir recurso)
            safeWarn(self, ("Player %s usou skill %d"):format(player.Name, skillId))
        end)
        if not ok then
            warn("[SlayerCore] Erro em UseSkill:", err)
            return false, err
        end
        return true
    else
        warn("[SlayerCore] Ação desconhecida:", actionName)
        return false, "Ação desconhecida"
    end
end

-- Exemplo de API pública para atualizar configuração em runtime
function SlayerCore:SetConfig(key, value)
    if DEFAULTS[key] == nil and self.config[key] == nil then
        error("Chave de configuração inválida: " .. tostring(key))
    end
    self.config[key] = value
end

-- ======= Player handlers privados =======
function SlayerCore:_onPlayerAdded(player)
    if not player or not player.UserId then return end
    self._players[player.UserId] = {
        joinedAt = tick(),
        lastAction = 0,
        data = {}, -- armazene estado temporário
    }
    safeWarn(self, "Jogador adicionado:", player.Name)
    -- inicializações por jogador (por exemplo, dar stats)
    -- exemplo seguro: criar leaderstats sem expor internals
    local leaderstats = Instance.new("Folder")
    leaderstats.Name = "leaderstats"
    leaderstats.Parent = player
    local hp = Instance.new("IntValue")
    hp.Name = "HP"
    hp.Value = self.config.maxHealth
    hp.Parent = leaderstats
end

function SlayerCore:_onPlayerRemoving(player)
    if not player or not player.UserId then return end
    safeWarn(self, "Jogador saindo:", player.Name)
    -- salvar estado se necessário (persistência)
    self._players[player.UserId] = nil
end

-- ======= Exemplos de métodos utilitários que você provavelmente precisa migrar =======
-- Use este padrão para funções que alteram o estado do jogo
function SlayerCore:ApplyDamage(player, amount, source)
    assert(type(amount) == "number", "amount deve ser número")
    local pid = player.UserId
    if not self._players[pid] then return false, "Player não registrado" end
    local ok, err = pcall(function()
        -- exemplo: reduzir leaderstat HP
        local stats = player:FindFirstChild("leaderstats")
        if stats and stats:FindFirstChild("HP") then
            local hp = stats.HP
            hp.Value = math.max(0, hp.Value - amount)
        end
    end)
    if not ok then
        warn("[SlayerCore] ApplyDamage falhou:", err)
        return false, err
    end
    return true
end

return SlayerCore
