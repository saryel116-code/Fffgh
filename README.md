--[[
    SCRIPT: Reroll Manager com Toggle de Save State
    JOGO ALVO: Defesa do Reino dos Doces (Monero Studios)
    
    AVISO CRÍTICO: Este script tenta impedir que o CLIENTE envie dados de gasto
    para o servidor. Se o servidor já tiver processado o gasto antes do hook,
    ou se o jogo usar uma arquitetura diferente, o rollback NÃO funcionará.
    Use com moderação e por sua conta e risco.
]]

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local LocalPlayer = Players.LocalPlayer

-- ==========================================
-- VARIÁVEIS DE ESTADO
-- ==========================================
local savedJobId = nil
local isSaveStateActive = false -- Controla se o bloqueio está ativo
local isPanelOpen = true
local originalNamecall = nil -- Armazena a função original para restaurar

-- ==========================================
-- 1. CRIAÇÃO DA UI (Floating Panel)
-- ==========================================

local oldGui = CoreGui:FindFirstChild("RerollManagerUI")
if oldGui then oldGui:Destroy() end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "RerollManagerUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = CoreGui
ScreenGui.DisplayOrder = 999

-- Botão Flutuante
local ToggleButton = Instance.new("TextButton")
ToggleButton.Size = UDim2.new(0, 50, 0, 50)
ToggleButton.Position = UDim2.new(0, 20, 0, 200)
ToggleButton.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
ToggleButton.TextColor3 = Color3.fromRGB(255, 200, 0)
ToggleButton.Text = "⚙️"
ToggleButton.TextSize = 24
ToggleButton.Font = Enum.Font.GothamBold
ToggleButton.Parent = ScreenGui
ToggleButton.Active = true
ToggleButton.Draggable = true

local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(1, 0)
ToggleCorner.Parent = ToggleButton

-- Painel Principal
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 260, 0, 280) -- Altura aumentada para o novo botão
MainFrame.Position = UDim2.new(0, 20, 0, 260)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui
MainFrame.Active = true
MainFrame.Draggable = true

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 8)
MainCorner.Parent = MainFrame

-- Título
local TitleLabel = Instance.new("TextLabel")
TitleLabel.Size = UDim2.new(1, 0, 0, 35)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "REROLL MANAGER (Doces)"
TitleLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextSize = 14
TitleLabel.Parent = MainFrame

-- Botão TOGGLE: Ligar/Desligar Save State
local SaveToggleButton = Instance.new("TextButton")
SaveToggleButton.Size = UDim2.new(1, -20, 0, 40)
SaveToggleButton.Position = UDim2.new(0, 10, 0, 45)
SaveToggleButton.BackgroundColor3 = Color3.fromRGB(60, 60, 60) -- Cinza (Desligado)
SaveToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
SaveToggleButton.Text = "🔴 Salvar Ponto: DESLIGADO"
SaveToggleButton.Font = Enum.Font.Gotham
SaveToggleButton.TextSize = 13
SaveToggleButton.Parent = MainFrame

local SaveToggleCorner = Instance.new("UICorner")
SaveToggleCorner.CornerRadius = UDim.new(0, 5)
SaveToggleCorner.Parent = SaveToggleButton

-- Status Label
local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, -20, 0, 40)
StatusLabel.Position = UDim2.new(0, 10, 0, 90)
StatusLabel.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
StatusLabel.TextColor3 = Color3.fromRGB(150, 150, 150)
StatusLabel.Text = "Status: Aguardando..."
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextSize = 11
StatusLabel.TextWrapped = true
StatusLabel.Parent = MainFrame

local StatusCorner = Instance.new("UICorner")
StatusCorner.CornerRadius = UDim.new(0, 4)
StatusCorner.Parent = StatusLabel

-- Botão Relogar
local RejoinButton = Instance.new("TextButton")
RejoinButton.Size = UDim2.new(1, -20, 0, 40)
RejoinButton.Position = UDim2.new(0, 10, 0, 140)
RejoinButton.BackgroundColor3 = Color3.fromRGB(80, 40, 40)
RejoinButton.TextColor3 = Color3.fromRGB(255, 255, 255)
RejoinButton.Text = "🔄 Relogar (Rejoin)"
RejoinButton.Font = Enum.Font.Gotham
RejoinButton.TextSize = 13
RejoinButton.Parent = MainFrame

local RejoinCorner = Instance.new("UICorner")
RejoinCorner.CornerRadius = UDim.new(0, 5)
RejoinCorner.Parent = RejoinButton

-- ==========================================
-- 2. LÓGICA DO TOGGLE (LIGAR/DESLIGAR SAVE)
-- ==========================================

local function updateStatus(text, color)
    StatusLabel.Text = "Status: " .. text
    StatusLabel.TextColor3 = color or Color3.fromRGB(150, 150, 150)
end

-- Função que ativa o "bloqueio" de envio de dados
local function ativarSaveState()
    if isSaveStateActive then return end
    
    -- Salva a função original de namecall para poder restaurar depois
    originalNamecall = hookmetamethod(game, "__namecall", function(self, ...)
        local method = getnamecallmethod()
        local args = {...}
        
        -- AQUI ESTÁ O TRUQUE: Bloqueia o envio de dados de reroll
        -- Procuramos por RemoteEvents com nomes comuns de reroll/trait
        -- e impedimos que o cliente envie a confirmação para o servidor.
        if isSaveStateActive and method == "FireServer" then
            local name = tostring(self.Name):lower()
            -- Se o remote for de reroll ou trait, bloqueia o envio
            if name:find("reroll") or name:find("trait") or name:find("crystal") then
                print("[Reroll Manager] Bloqueado envio para: " .. name)
                return nil -- Impede o FireServer
            end
        end
        
        return originalNamecall(self, ...)
    end)
    
    isSaveStateActive = true
    SaveToggleButton.Text = "🟢 Salvar Ponto: LIGADO"
    SaveToggleButton.BackgroundColor3 = Color3.fromRGB(40, 100, 40) -- Verde
    updateStatus("Save State ATIVADO. Rerolls não serão enviados.", Color3.fromRGB(100, 255, 100))
end

local function desativarSaveState()
    if not isSaveStateActive then return end
    
    -- Restaura a função original (se possível)
    if originalNamecall then
        -- Nota: Restaurar hookmetamethod é complicado em Lua.
        -- Na prática, apenas marcamos a flag como false, o que já para o bloqueio.
        originalNamecall = nil
    end
    
    isSaveStateActive = false
    SaveToggleButton.Text = "🔴 Salvar Ponto: DESLIGADO"
    SaveToggleButton.BackgroundColor3 = Color3.fromRGB(60, 60, 60) -- Cinza
    updateStatus("Save State DESATIVADO. Jogo normal.", Color3.fromRGB(255, 100, 100))
end

-- Conexão do botão Toggle
SaveToggleButton.MouseButton1Click:Connect(function()
    if isSaveStateActive then
        desativarSaveState()
    else
        ativarSaveState()
        -- Também salva o JobId atual para o Relogar
        savedJobId = game.JobId
        updateStatus("Ponto Salvo (Servidor: " .. string.sub(savedJobId, 1, 8) .. "...)", Color3.fromRGB(100, 255, 100))
    end
end)

-- ==========================================
-- 3. LÓGICA DO RELOGAR
-- ==========================================

RejoinButton.MouseButton1Click:Connect(function()
    updateStatus("Reconectando...", Color3.fromRGB(255, 200, 0))
    
    local success, errorMessage = pcall(function()
        if savedJobId and savedJobId ~= "" then
            TeleportService:TeleportToPlaceInstance(game.PlaceId, savedJobId, LocalPlayer)
        else
            TeleportService:Teleport(game.PlaceId, LocalPlayer)
        end
    end)
    
    if not success then
        warn("[Reroll Manager] Falha: " .. tostring(errorMessage))
        updateStatus("Falha no Rejoin. Tentando servidor novo...", Color3.fromRGB(255, 100, 100))
        pcall(function()
            TeleportService:Teleport(game.PlaceId, LocalPlayer)
        end)
    end
end)

-- Toggle UI (Minimizar)
ToggleButton.MouseButton1Click:Connect(function()
    isPanelOpen = not isPanelOpen
    MainFrame.Visible = isPanelOpen
end)

MainFrame.Visible = true
updateStatus("Pronto. Ative o Save antes de Rolar.", Color3.fromRGB(150, 150, 150))
