-- ============================================
-- CONFIGURAÇÕES INICIAIS
-- ============================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local StarterGui = game:GetService("StarterGui")

local LocalPlayer = Players.LocalPlayer

-- ============================================
-- SISTEMA DE SEGURANÇA BÁSICO
-- ============================================
local function safeCall(func, ...)
    local success, result = pcall(func, ...)
    if not success then
        warn("[EB] Erro: " .. tostring(result))
    end
    return success, result
end

-- ============================================
-- VERIFICAÇÃO DE EXECUTOR
-- ============================================
local ExecutorFeatures = {
    HasFileSystem = pcall(function() return isfile end),
    HasWriteFile = pcall(function() return writefile end),
    HasReadFile = pcall(function() return readfile end),
    HasSetClipboard = pcall(function() return setclipboard end),
    HasSynRequest = pcall(function() return syn and syn.request end),
    HasHttpRequest = pcall(function() return request or http_request end)
}

-- ============================================
-- SISTEMA DE SAVE ADAPTATIVO
-- ============================================
local SaveSystem = {
    Data = {},
    CanSave = ExecutorFeatures.HasWriteFile and ExecutorFeatures.HasReadFile
}

function SaveSystem:Load()
    if not self.CanSave then
        self.Data = {
            JJ = {final = 10, suffix = "!", interval = 1.5},
            Routes = {},
            Settings = {}
        }
        return
    end
    
    local success, content = pcall(function()
        return readfile("EB_Data.json")
    end)
    
    if success and content then
        local success2, data = pcall(function()
            return game:GetService("HttpService"):JSONDecode(content)
        end)
        if success2 then
            self.Data = data
        end
    end
end

function SaveSystem:Save()
    if not self.CanSave then return end
    
    pcall(function()
        local json = game:GetService("HttpService"):JSONEncode(self.Data)
        writefile("EB_Data.json", json)
    end)
end

-- ============================================
-- DETECTOR DE SISTEMA DE CHAT REAL
-- ============================================
local ChatSystem = {
    Method = nil -- "legacy", "textchat", "remote"
}

function ChatSystem:Detect()
    -- Método 1: Procurar por remotes de chat comuns
    local chatRemotes = {
        "SayMessageRequest",
        "ChatRequest",
        "MessageRequest",
        "SendMessage"
    }
    
    for _, remoteName in ipairs(chatRemotes) do
        local remote = ReplicatedStorage:FindFirstChild(remoteName, true)
        if remote and remote:IsA("RemoteEvent") then
            self.Method = "remote"
            self.Remote = remote
            return true
        end
    end
    
    -- Método 2: TextChatService (Roblox moderno)
    local textChatService = game:GetService("TextChatService")
    if textChatService then
        local textChannels = textChatService:FindFirstChild("TextChannels")
        if textChannels then
            for _, channel in ipairs(textChannels:GetChildren()) do
                if channel:IsA("TextChannel") then
                    self.Method = "textchannel"
                    self.Channel = channel
                    return true
                end
            end
        end
    end
    
    -- Método 3: Chat legado
    local chatService = game:GetService("Chat")
    if chatService and chatService.Chat then
        self.Method = "legacy"
        return true
    end
    
    return false
end

function ChatSystem:SendMessage(message)
    local character = LocalPlayer.Character
    if not character then return false end
    
    if self.Method == "remote" and self.Remote then
        self.Remote:FireServer(message, "All")
        return true
    elseif self.Method == "textchannel" and self.Channel then
        pcall(function()
            self.Channel:SendAsync(message)
        end)
        return true
    elseif self.Method == "legacy" then
        local head = character:FindFirstChild("Head")
        if head then
            game:GetService("Chat"):Chat(head, message, Enum.ChatColor.White)
            return true
        end
    end
    
    return false
end

-- ============================================
-- CONVERSOR DE NÚMEROS (REAL E TESTADO)
-- ============================================
local NumberToWords = {}

function NumberToWords.Convert(num)
    local unidades = {
        "UM", "DOIS", "TRÊS", "QUATRO", "CINCO",
        "SEIS", "SETE", "OITO", "NOVE", "DEZ",
        "ONZE", "DOZE", "TREZE", "QUATORZE", "QUINZE",
        "DEZESSEIS", "DEZESSETE", "DEZOITO", "DEZENOVE"
    }
    
    local dezenas = {
        [2] = "VINTE", [3] = "TRINTA", [4] = "QUARENTA", [5] = "CINQUENTA",
        [6] = "SESSENTA", [7] = "SETENTA", [8] = "OITENTA", [9] = "NOVENTA"
    }
    
    local centenas = {
        [1] = "CENTO", [2] = "DUZENTOS", [3] = "TREZENTOS", [4] = "QUATROCENTOS",
        [5] = "QUINHENTOS", [6] = "SEISCENTOS", [7] = "SETECENTOS", [8] = "OITOCENTOS",
        [9] = "NOVECENTOS"
    }
    
    if type(num) ~= "number" or num < 1 or num > 999 then
        return "NÚMERO INVÁLIDO"
    end
    
    if num == 100 then return "CEM" end
    
    if num <= 19 then
        return unidades[num]
    elseif num <= 99 then
        local dezena = math.floor(num / 10)
        local unidade = num % 10
        if unidade == 0 then
            return dezenas[dezena]
        else
            return dezenas[dezena] .. " E " .. unidades[unidade]
        end
    else
        local centena = math.floor(num / 100)
        local resto = num % 100
        
        local prefixo
        if centena == 1 then
            prefixo = "CENTO"
        else
            prefixo = centenas[centena]
        end
        
        if resto == 0 then
            if centena == 1 then return "CEM"
            else return centenas[centena] end
        elseif resto <= 19 then
            return prefixo .. " E " .. unidades[resto]
        else
            local dezena = math.floor(resto / 10)
            local unidade = resto % 10
            if unidade == 0 then
                return prefixo .. " E " .. dezenas[dezena]
            else
                return prefixo .. " E " .. dezenas[dezena] .. " E " .. unidades[unidade]
            end
        end
    end
end

-- ============================================
-- SISTEMA DE JJ'S (FUNCIONAL REAL)
-- ============================================
local JJSystem = {
    Running = false,
    Paused = false,
    CurrentNumber = 1,
    Config = {
        FinalNumber = 10,
        Suffix = "!",
        Interval = 1.5
    },
    Thread = nil
}

function JJSystem:Start()
    if self.Running then return end
    
    self.Running = true
    self.Paused = false
    self.CurrentNumber = 1
    
    self.Thread = task.spawn(function()
        while self.Running and self.CurrentNumber <= self.Config.FinalNumber do
            if not self.Paused then
                local text = NumberToWords.Convert(self.CurrentNumber) .. " " .. self.Config.Suffix
                ChatSystem:SendMessage(text)
                self.CurrentNumber = self.CurrentNumber + 1
            end
            task.wait(self.Config.Interval)
        end
        self:Stop()
    end)
end

function JJSystem:Pause()
    self.Paused = true
end

function JJSystem:Resume()
    self.Paused = false
end

function JJSystem:Stop()
    self.Running = false
    self.Paused = false
    self.CurrentNumber = 1
end

-- ============================================
-- SISTEMA DE PARKOUR (SEM BUGS FÍSICOS)
-- ============================================
local ParkourSystem = {
    Recording = false,
    Playing = false,
    RecordedFrames = {},
    Routes = {},
    CurrentRoute = nil
}

function ParkourSystem:StartRecording()
    if self.Recording then return end
    
    local char = LocalPlayer.Character
    if not char then return end
    
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    
    self.Recording = true
    self.RecordedFrames = {}
    
    task.spawn(function()
        local lastPos = root.Position
        local startTime = tick()
        
        while self.Recording do
            local currentChar = LocalPlayer.Character
            if not currentChar then break end
            
            local currentRoot = currentChar:FindFirstChild("HumanoidRootPart")
            if not currentRoot then break end
            
            local humanoid = currentChar:FindFirstChild("Humanoid")
            
            -- Só gravar se houve movimento significativo
            if (currentRoot.Position - lastPos).Magnitude > 0.1 then
                table.insert(self.RecordedFrames, {
                    Position = currentRoot.Position,
                    Velocity = currentRoot.Velocity,
                    Jump = humanoid and humanoid.Jump or false,
                    Time = tick() - startTime
                })
                lastPos = currentRoot.Position
            end
            
            task.wait(0.05) -- 20 FPS para economizar memória
        end
    end)
end

function ParkourSystem:StopRecording()
    self.Recording = false
    
    if #self.RecordedFrames == 0 then return end
    
    local routeName = "Rota " .. (#self.Routes + 1)
    self.Routes[routeName] = {
        Frames = self.RecordedFrames,
        Date = os.date("%H:%M:%S"),
        Duration = self.RecordedFrames[#self.RecordedFrames].Time
    }
    
    -- Salvar
    SaveSystem.Data.Routes = {}
    for name, route in pairs(self.Routes) do
        SaveSystem.Data.Routes[name] = {
            frames = #route.Frames,
            duration = route.Duration
        }
    end
    SaveSystem:Save()
    
    return routeName
end

function ParkourSystem:PlayRoute(routeName)
    if self.Playing then return end
    
    local route = self.Routes[routeName]
    if not route then return end
    
    self.Playing = true
    
    task.spawn(function()
        local char = LocalPlayer.Character
        if not char then 
            self.Playing = false
            return 
        end
        
        local root = char:FindFirstChild("HumanoidRootPart")
        local humanoid = char:FindFirstChild("Humanoid")
        
        if not root or not humanoid then
            self.Playing = false
            return
        end
        
        -- Usar movimentação natural, não teleporte
        for i, frame in ipairs(route.Frames) do
            if not self.Playing then break end
            
            -- Verificar se personagem ainda existe
            if not LocalPlayer.Character or LocalPlayer.Character ~= char then
                break
            end
            
            -- Movimento usando Humanoid:MoveTo (mais natural)
            humanoid:MoveTo(frame.Position)
            
            -- Pulo no momento certo
            if frame.Jump then
                humanoid.Jump = true
            end
            
            -- Aguardar o tempo correto
            local waitTime = i < #route.Frames and 
                           (route.Frames[i+1].Time - frame.Time) or 0.1
            task.wait(math.max(0.05, waitTime))
        end
        
        self.Playing = false
    end)
end

function ParkourSystem:StopPlayback()
    self.Playing = false
end

-- ============================================
-- SISTEMA DE IA (SEM EXPOR API KEY)
-- ============================================
local IASystem = {
    Enabled = false,
    ProxyURL = nil -- Opcional: URL de proxy seguro
}

function IASystem:GenerateText(theme, type, formality)
    -- Método seguro: usar proxy ou avisar que precisa de configuração
    if not self.ProxyURL then
        return [[
⚠️ SISTEMA DE IA NÃO CONFIGURADO

Para usar o gerador de textos:
1. Configure um proxy seguro
2. Ou use a API diretamente (não recomendado)

Tema solicitado: ]] .. theme .. [[
Tipo: ]] .. type .. [[
Formalidade: ]] .. formality .. [[

Este é um texto de exemplo que seria gerado.
Configure a API para obter textos reais.
        ]]
    end
    
    -- Se tiver proxy configurado
    if ExecutorFeatures.HasSynRequest or ExecutorFeatures.HasHttpRequest then
        local requestFunc = syn and syn.request or request
        
        local body = game:GetService("HttpService"):JSONEncode({
            theme = theme,
            type = type,
            formality = formality
        })
        
        local response = requestFunc({
            Url = self.ProxyURL,
            Method = "POST",
            Headers = {["Content-Type"] = "application/json"},
            Body = body
        })
        
        if response.StatusCode == 200 then
            local data = game:GetService("HttpService"):JSONDecode(response.Body)
            return data.text or "Erro ao gerar texto"
        end
    end
    
    return "Falha na comunicação com o servidor de IA"
end

-- ============================================
-- INTERFACE GRÁFICA (COMPLETA E FUNCIONAL)
-- ============================================
local GUI = {
    ScreenGui = nil,
    MainFrame = nil,
    Visible = true
}

function GUI:Create()
    -- Destruir anterior
    if self.ScreenGui then
        self.ScreenGui:Destroy()
    end
    
    -- Criar nova
    self.ScreenGui = Instance.new("ScreenGui")
    self.ScreenGui.Name = "EB_System"
    self.ScreenGui.Parent = game:GetService("CoreGui") -- Mais seguro
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 550, 0, 450)
    frame.Position = UDim2.new(0.5, -275, 0.5, -225)
    frame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    frame.BorderSizePixel = 0
    frame.Parent = self.ScreenGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = frame
    
    -- Título
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 35)
    title.BackgroundColor3 = Color3.fromRGB(45, 35, 25)
    title.Text = "EB TRAINING SYSTEM v4.0"
    title.TextColor3 = Color3.fromRGB(220, 220, 220)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 16
    title.Parent = frame
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 10)
    titleCorner.Parent = title
    
    -- Botão fechar
    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0, 30, 0, 25)
    close.Position = UDim2.new(1, -35, 0, 5)
    close.BackgroundColor3 = Color3.fromRGB(244, 67, 54)
    close.Text = "X"
    close.TextColor3 = Color3.fromRGB(255, 255, 255)
    close.Font = Enum.Font.GothamBold
    close.BorderSizePixel = 0
    close.Parent = title
    
    local closeCorner = Instance.new("UICorner")
    closeCorner.CornerRadius = UDim.new(0, 5)
    closeCorner.Parent = close
    
    close.MouseButton1Click:Connect(function()
        self.ScreenGui.Enabled = not self.ScreenGui.Enabled
    end)
    
    -- Sistema de abas simples
    local tabFrame = Instance.new("Frame")
    tabFrame.Size = UDim2.new(1, 0, 0, 35)
    tabFrame.Position = UDim2.new(0, 0, 0, 35)
    tabFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    tabFrame.Parent = frame
    
    local content = Instance.new("ScrollingFrame")
    content.Size = UDim2.new(1, -20, 1, -85)
    content.Position = UDim2.new(0, 10, 0, 75)
    content.BackgroundTransparency = 1
    content.ScrollBarThickness = 4
    content.CanvasSize = UDim2.new(0, 0, 0, 500)
    content.Parent = frame
    
    -- Criar abas
    local tabs = {
        {Name = "JJ's", Content = self:CreateJJContent(content)},
        {Name = "Parkour", Content = self:CreateParkourContent(content)},
        {Name = "IA", Content = self:CreateIAContent(content)}
    }
    
    local tabButtons = {}
    local tabContents = {}
    
    for i, tab in ipairs(tabs) do
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 100, 1, -5)
        btn.Position = UDim2.new(0, (i-1)*105 + 10, 0, 3)
        btn.BackgroundColor3 = i == 1 and Color3.fromRGB(60, 50, 40) or Color3.fromRGB(40, 40, 40)
        btn.Text = tab.Name
        btn.TextColor3 = Color3.fromRGB(220, 220, 220)
        btn.Font = Enum.Font.Gotham
        btn.TextSize = 13
        btn.BorderSizePixel = 0
        btn.Parent = tabFrame
        
        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 5)
        btnCorner.Parent = btn
        
        table.insert(tabButtons, btn)
        table.insert(tabContents, tab.Content)
        
        btn.MouseButton1Click:Connect(function()
            for _, b in ipairs(tabButtons) do
                b.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
            end
            btn.BackgroundColor3 = Color3.fromRGB(60, 50, 40)
            
            for _, c in ipairs(tabContents) do
                c.Visible = false
            end
            tab.Content.Visible = true
        end)
    end
    
    -- Mostrar primeira aba
    if tabContents[1] then
        tabContents[1].Visible = true
    end
    
    -- Tornar arrastável
    self:MakeDraggable(frame, title)
    
    self.MainFrame = frame
    return self.ScreenGui
end

function GUI:CreateJJContent(parent)
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, 0, 1, 0)
    container.BackgroundTransparency = 1
    container.Visible = false
    container.Parent = parent
    
    -- Configurações
    local configLabel = Instance.new("TextLabel")
    configLabel.Size = UDim2.new(1, 0, 0, 25)
    configLabel.BackgroundTransparency = 1
    configLabel.Text = "CONFIGURAÇÕES"
    configLabel.TextColor3 = Color3.fromRGB(180, 150, 100)
    configLabel.Font = Enum.Font.GothamBold
    configLabel.TextSize = 14
    configLabel.Parent = container
    
    -- Inputs
    local inputs = {
        {Label = "Número Final", Default = "10", Y = 30},
        {Label = "Sufixo", Default = "!", Y = 60},
        {Label = "Intervalo (s)", Default = "1.5", Y = 90}
    }
    
    local inputBoxes = {}
    for _, input in ipairs(inputs) do
        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0, 90, 0, 25)
        label.Position = UDim2.new(0, 5, 0, input.Y)
        label.BackgroundTransparency = 1
        label.Text = input.Label .. ":"
        label.TextColor3 = Color3.fromRGB(200, 200, 200)
        label.Font = Enum.Font.Gotham
        label.TextSize = 12
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = container
        
        local box = Instance.new("TextBox")
        box.Size = UDim2.new(0, 150, 0, 25)
        box.Position = UDim2.new(0, 100, 0, input.Y)
        box.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
        box.Text = input.Default
        box.TextColor3 = Color3.fromRGB(220, 220, 220)
        box.Font = Enum.Font.Gotham
        box.TextSize = 12
        box.BorderSizePixel = 0
        box.Parent = container
        
        local boxCorner = Instance.new("UICorner")
        boxCorner.CornerRadius = UDim.new(0, 4)
        boxCorner.Parent = box
        
        table.insert(inputBoxes, box)
    end
    
    -- Botões
    local buttons = {
        {Text = "▶ INICIAR", Color = Color3.fromRGB(76, 175, 80), Y = 130, Callback = function()
            JJSystem.Config.FinalNumber = tonumber(inputBoxes[1].Text) or 10
            JJSystem.Config.Suffix = inputBoxes[2].Text or "!"
            JJSystem.Config.Interval = tonumber(inputBoxes[3].Text) or 1.5
            JJSystem:Start()
        end},
        {Text = "⏸ PAUSAR", Color = Color3.fromRGB(255, 152, 0), Y = 170, Callback = function()
            JJSystem:Pause()
        end},
        {Text = "▶ CONTINUAR", Color = Color3.fromRGB(76, 175, 80), Y = 210, Callback = function()
            JJSystem:Resume()
        end},
        {Text = "⏹ PARAR", Color = Color3.fromRGB(244, 67, 54), Y = 250, Callback = function()
            JJSystem:Stop()
        end}
    }
    
    for _, btn in ipairs(buttons) do
        local button = Instance.new("TextButton")
        button.Size = UDim2.new(1, -10, 0, 30)
        button.Position = UDim2.new(0, 5, 0, btn.Y)
        button.BackgroundColor3 = btn.Color
        button.Text = btn.Text
        button.TextColor3 = Color3.fromRGB(255, 255, 255)
        button.Font = Enum.Font.GothamBold
        button.TextSize = 12
        button.BorderSizePixel = 0
        button.Parent = container
        
        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 5)
        btnCorner.Parent = button
        
        button.MouseButton1Click:Connect(btn.Callback)
    end
    
    -- Status
    local status = Instance.new("TextLabel")
    status.Size = UDim2.new(1, -10, 0, 25)
    status.Position = UDim2.new(0, 5, 0, 300)
    status.BackgroundTransparency = 1
    status.Text = "Pronto"
    status.TextColor3 = Color3.fromRGB(150, 150, 150)
    status.Font = Enum.Font.Gotham
    status.TextSize = 11
    status.Parent = container
    
    -- Atualizar status
    task.spawn(function()
        while container.Parent do
            if JJSystem.Running then
                status.Text = string.format("Enviando: %d/%d", JJSystem.CurrentNumber - 1, JJSystem.Config.FinalNumber)
            else
                status.Text = "Pronto"
            end
            task.wait(0.5)
        end
    end)
    
    return container
end

function GUI:CreateParkourContent(parent)
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, 0, 1, 0)
    container.BackgroundTransparency = 1
    container.Visible = false
    container.Parent = parent
    
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 0, 25)
    label.BackgroundTransparency = 1
    label.Text = "PARKOUR RECORDER"
    label.TextColor3 = Color3.fromRGB(180, 150, 100)
    label.Font = Enum.Font.GothamBold
    label.TextSize = 14
    label.Parent = container
    
    local buttons = {
        {Text = "⏺ GRAVAR", Color = Color3.fromRGB(244, 67, 54), Y = 35, Callback = function()
            ParkourSystem:StartRecording()
        end},
        {Text = "⏹ PARAR GRAVAÇÃO", Color = Color3.fromRGB(255, 152, 0), Y = 75, Callback = function()
            ParkourSystem:StopRecording()
        end},
        {Text = "▶ REPRODUZIR ÚLTIMA", Color = Color3.fromRGB(76, 175, 80), Y = 115, Callback = function()
            local lastRoute = nil
            for name, _ in pairs(ParkourSystem.Routes) do
                lastRoute = name
            end
            if lastRoute then
                ParkourSystem:PlayRoute(lastRoute)
            end
        end},
        {Text = "⏹ PARAR", Color = Color3.fromRGB(244, 67, 54), Y = 155, Callback = function()
            ParkourSystem:StopPlayback()
        end}
    }
    
    for _, btn in ipairs(buttons) do
        local button = Instance.new("TextButton")
        button.Size = UDim2.new(1, -10, 0, 30)
        button.Position = UDim2.new(0, 5, 0, btn.Y)
        button.BackgroundColor3 = btn.Color
        button.Text = btn.Text
        button.TextColor3 = Color3.fromRGB(255, 255, 255)
        button.Font = Enum.Font.GothamBold
        button.TextSize = 12
        button.BorderSizePixel = 0
        button.Parent = container
        
        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 5)
        btnCorner.Parent = button
        
        button.MouseButton1Click:Connect(btn.Callback)
    end
    
    -- Lista de rotas
    local routesLabel = Instance.new("TextLabel")
    routesLabel.Size = UDim2.new(1, 0, 0, 20)
    routesLabel.Position = UDim2.new(0, 5, 0, 200)
    routesLabel.BackgroundTransparency = 1
    routesLabel.Text = "ROTAS SALVAS:"
    routesLabel.TextColor3 = Color3.fromRGB(150, 150, 150)
    routesLabel.Font = Enum.Font.Gotham
    routesLabel.TextSize = 11
    routesLabel.TextXAlignment = Enum.TextXAlignment.Left
    routesLabel.Parent = container
    
    local routesList = Instance.new("ScrollingFrame")
    routesList.Size = UDim2.new(1, -10, 0, 150)
    routesList.Position = UDim2.new(0, 5, 0, 225)
    routesList.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    routesList.BorderSizePixel = 0
    routesList.ScrollBarThickness = 3
    routesList.Parent = container
    
    local listLayout = Instance.new("UIListLayout")
    listLayout.Padding = UDim.new(0, 3)
    listLayout.Parent = routesList
    
    -- Atualizar lista
    local function updateRoutes()
        for _, child in ipairs(routesList:GetChildren()) do
            if child:IsA("TextLabel") then
                child:Destroy()
            end
        end
        
        for name, route in pairs(ParkourSystem.Routes) do
            local routeLabel = Instance.new("TextLabel")
            routeLabel.Size = UDim2.new(1, -10, 0, 25)
            routeLabel.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
            routeLabel.Text = string.format("%s | %s | %d frames", name, route.Date, #route.Frames)
            routeLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
            routeLabel.Font = Enum.Font.Gotham
            routeLabel.TextSize = 10
            routeLabel.Parent = routesList
        end
        
        routesList.CanvasSize = UDim2.new(0, 0, 0, #ParkourSystem.Routes * 28)
    end
    
    task.spawn(function()
        while container.Parent do
            updateRoutes()
            task.wait(1)
        end
    end)
    
    return container
end

function GUI:CreateIAContent(parent)
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, 0, 1, 0)
    container.BackgroundTransparency = 1
    container.Visible = false
    container.Parent = parent
    
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 0, 25)
    label.BackgroundTransparency = 1
    label.Text = "GERADOR DE TEXTOS (IA)"
    label.TextColor3 = Color3.fromRGB(180, 150, 100)
    label.Font = Enum.Font.GothamBold
    label.TextSize = 14
    label.Parent = container
    
    -- Aviso de configuração
    local aviso = Instance.new("TextLabel")
    aviso.Size = UDim2.new(1, -10, 0, 60)
    aviso.Position = UDim2.new(0, 5, 0, 35)
    aviso.BackgroundTransparency = 1
    aviso.Text = [[
⚠️ Configure um proxy seguro para usar a IA
Sem proxy, textos de exemplo serão gerados
API Keys NÃO devem ficar no client
    ]]
    aviso.TextColor3 = Color3.fromRGB(255, 152, 0)
    aviso.Font = Enum.Font.Gotham
    aviso.TextSize = 10
    aviso.TextWrapped = true
    aviso.Parent = container
    
    -- Input de tema
    local themeLabel = Instance.new("TextLabel")
    themeLabel.Size = UDim2.new(0, 50, 0, 25)
    themeLabel.Position = UDim2.new(0, 5, 0, 110)
    themeLabel.BackgroundTransparency = 1
    themeLabel.Text = "Tema:"
    themeLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    themeLabel.Font = Enum.Font.Gotham
    themeLabel.TextSize = 12
    themeLabel.Parent = container
    
    local themeBox = Instance.new("TextBox")
    themeBox.Size = UDim2.new(1, -65, 0, 25)
    themeBox.Position = UDim2.new(0, 60, 0, 110)
    themeBox.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    themeBox.Text = "Disciplina militar"
    themeBox.TextColor3 = Color3.fromRGB(220, 220, 220)
    themeBox.Font = Enum.Font.Gotham
    themeBox.TextSize = 12
    themeBox.BorderSizePixel = 0
    themeBox.Parent = container
    
    local themeCorner = Instance.new("UICorner")
    themeCorner.CornerRadius = UDim.new(0, 4)
    themeCorner.Parent = themeBox
    
    -- Botão gerar
    local generateBtn = Instance.new("TextButton")
    generateBtn.Size = UDim2.new(1, -10, 0, 30)
    generateBtn.Position = UDim2.new(0, 5, 0, 150)
    generateBtn.BackgroundColor3 = Color3.fromRGB(60, 50, 40)
    generateBtn.Text = "🤖 GERAR TEXTO"
    generateBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    generateBtn.Font = Enum.Font.GothamBold
    generateBtn.TextSize = 12
    generateBtn.BorderSizePixel = 0
    generateBtn.Parent = container
    
    local genCorner = Instance.new("UICorner")
    genCorner.CornerRadius = UDim.new(0, 5)
    genCorner.Parent = generateBtn
    
    -- Área de texto
    local textArea = Instance.new("TextLabel")
    textArea.Size = UDim2.new(1, -10, 0, 250)
    textArea.Position = UDim2.new(0, 5, 0, 190)
    textArea.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    textArea.Text = ""
    textArea.TextColor3 = Color3.fromRGB(220, 220, 220)
    textArea.Font = Enum.Font.Gotham
    textArea.TextSize = 11
    textArea.TextWrapped = true
    textArea.TextXAlignment = Enum.TextXAlignment.Left
    textArea.TextYAlignment = Enum.TextYAlignment.Top
    textArea.BorderSizePixel = 0
    textArea.Parent = container
    
    local textCorner = Instance.new("UICorner")
    textCorner.CornerRadius = UDim.new(0, 5)
    textCorner.Parent = textArea
    
    generateBtn.MouseButton1Click:Connect(function()
        local theme = themeBox
