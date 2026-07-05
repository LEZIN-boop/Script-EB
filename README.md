--[[ JECK DE CALCINHA SCRIPTS - JJ's AUTO (POLICHINELOS) ]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local LocalPlayer = Players.LocalPlayer

-- Variáveis
local running = false
local paused = false
local target = 10
local delayTime = 0.3
local doneCount = 0
local jjThread = nil

-- Função para detectar qual tecla está sendo pedida
local function detectKey()
    local gui = LocalPlayer.PlayerGui
    if not gui then return nil end
    
    -- Procura o template de input
    local polichinelos = gui:FindFirstChild("Polichinelos")
    if polichinelos then
        local screen = polichinelos:FindFirstChild("Screen")
        if screen then
            local template = screen:FindFirstChild("InputTemplate")
            if template then
                -- Procura por texto "Q" ou "E" dentro do template
                for _, child in ipairs(template:GetDescendants()) do
                    if child:IsA("TextLabel") or child:IsA("TextButton") then
                        local text = child.Text:upper():gsub("%s", "")
                        if text == "Q" then
                            return Enum.KeyCode.Q
                        elseif text == "E" then
                            return Enum.KeyCode.E
                        end
                    end
                end
            end
        end
    end
    return nil
end

-- Função para contar quantos já foram feitos
local function getProgress()
    local char = LocalPlayer.Character
    if not char then return 0 end
    
    local head = char:FindFirstChild("Head")
    if not head then return 0 end
    
    -- Procura BillboardGui ou similar acima da cabeça
    for _, child in ipairs(head:GetChildren()) do
        if child:IsA("BillboardGui") then
            for _, label in ipairs(child:GetDescendants()) do
                if label:IsA("TextLabel") then
                    -- Tenta extrair número do texto (ex: "5/10")
                    local num = label.Text:match("(%d+)/%d+")
                    if num then
                        return tonumber(num) or 0
                    end
                    -- Ou número puro
                    local pure = label.Text:match("(%d+)")
                    if pure then
                        return tonumber(pure) or 0
                    end
                end
            end
        end
    end
    return doneCount
end

-- Função para pressionar tecla
local function pressKey(key)
    pcall(function()
        VirtualInputManager:SendKeyEvent(true, key, false, nil)
        task.wait(0.05)
        VirtualInputManager:SendKeyEvent(false, key, false, nil)
    end)
end

-- Loop principal
local function startJJ()
    if running then return end
    running = true
    paused = false
    
    jjThread = task.spawn(function()
        local lastCount = 0
        local noProgressCount = 0
        
        while running do
            if not paused then
                -- Detecta qual tecla o jogo está pedindo
                local key = detectKey()
                
                if key then
                    pressKey(key)
                    task.wait(delayTime)
                    noProgressCount = 0
                else
                    -- Se não encontrou o template, espera um pouco
                    task.wait(0.5)
                    noProgressCount = noProgressCount + 1
                end
                
                -- Verifica progresso
                local current = getProgress()
                if current > lastCount then
                    doneCount = current
                    lastCount = current
                end
                
                -- Se atingiu a meta, para
                if doneCount >= target then
                    running = false
                    break
                end
                
                -- Se ficou muito tempo sem progresso, alerta
                if noProgressCount > 10 then
                    warn("[JJ's] Template não encontrado. O jogo está aberto?")
                    noProgressCount = 0
                end
            else
                task.wait(0.1)
            end
        end
        running = false
    end)
end

local function stopJJ()
    running = false
    paused = false
    if jjThread then
        task.cancel(jjThread)
        jjThread = nil
    end
end

local function pauseJJ()
    paused = true
end

local function resumeJJ()
    paused = false
end

-- UI
local gui = Instance.new("ScreenGui")
gui.Name = "JJsUI"
gui.ResetOnSpawn = false
gui.Parent = LocalPlayer.PlayerGui

local main = Instance.new("Frame")
main.Size = UDim2.new(0, 300, 0, 360)
main.Position = UDim2.new(0.5, -150, 0.5, -180)
main.BackgroundColor3 = Color3.fromRGB(18, 18, 30)
main.BorderSizePixel = 0
main.ClipsDescendants = true
main.Parent = gui

Instance.new("UICorner", main).CornerRadius = UDim.new(0, 18)
local ms = Instance.new("UIStroke", main)
ms.Thickness = 2
ms.Color = Color3.fromRGB(0, 0, 0)

-- Título
local t1 = Instance.new("TextLabel", main)
t1.Size = UDim2.new(1, 0, 0, 40)
t1.Position = UDim2.new(0, 0, 0, 10)
t1.BackgroundTransparency = 1
t1.Text = "JECK DE CALCINHA SCRIPTS"
t1.TextColor3 = Color3.fromRGB(255, 255, 255)
t1.Font = Enum.Font.GothamBlack
t1.TextSize = 18
t1.Parent = main

local t2 = Instance.new("TextLabel", main)
t2.Size = UDim2.new(1, 0, 0, 20)
t2.Position = UDim2.new(0, 0, 0, 48)
t2.BackgroundTransparency = 1
t2.Text = "JJ's Auto (Polichinelos)"
t2.TextColor3 = Color3.fromRGB(150, 150, 170)
t2.Font = Enum.Font.Gotham
t2.TextSize = 11
t2.Parent = main

-- Status
local st = Instance.new("TextLabel", main)
st.Size = UDim2.new(1, -30, 0, 25)
st.Position = UDim2.new(0, 15, 0, 75)
st.BackgroundColor3 = Color3.fromRGB(25, 25, 40)
st.BorderSizePixel = 0
st.Text = "PRONTO"
st.TextColor3 = Color3.fromRGB(76, 175, 80)
st.Font = Enum.Font.GothamBold
st.TextSize = 11
st.Parent = main
Instance.new("UICorner", st).CornerRadius = UDim.new(0, 8)

-- Inputs
local function createInput(label, default, y)
    local lbl = Instance.new("TextLabel", main)
    lbl.Size = UDim2.new(0, 140, 0, 20)
    lbl.Position = UDim2.new(0, 15, 0, y)
    lbl.BackgroundTransparency = 1
    lbl.Text = label
    lbl.TextColor3 = Color3.fromRGB(200, 200, 200)
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    
    local box = Instance.new("TextBox", main)
    box.Size = UDim2.new(1, -30, 0, 28)
    box.Position = UDim2.new(0, 15, 0, y + 20)
    box.BackgroundColor3 = Color3.fromRGB(25, 25, 40)
    box.Text = default
    box.TextColor3 = Color3.fromRGB(255, 255, 255)
    box.Font = Enum.Font.Gotham
    box.TextSize = 12
    box.BorderSizePixel = 0
    Instance.new("UICorner", box).CornerRadius = UDim.new(0, 6)
    return box
end

local qtdInput = createInput("Quantidade:", "10", 110)
local delayInput = createInput("Delay (s):", "0.3", 165)

-- Botão
local function btn(txt, cor, y, cb)
    local b = Instance.new("TextButton", main)
    b.Size = UDim2.new(1, -30, 0, 35)
    b.Position = UDim2.new(0, 15, 0, y)
    b.BackgroundColor3 = cor
    b.Text = txt
    b.TextColor3 = Color3.fromRGB(255, 255, 255)
    b.Font = Enum.Font.GothamBold
    b.TextSize = 13
    b.BorderSizePixel = 0
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
    b.MouseButton1Click:Connect(cb)
    return b
end

btn("▶ INICIAR (F2)", Color3.fromRGB(76, 175, 80), 225, function()
    target = tonumber(qtdInput.Text) or 10
    delayTime = tonumber(delayInput.Text) or 0.3
    doneCount = 0
    startJJ()
    if running then
        st.Text = "EXECUTANDO..."
        st.TextColor3 = Color3.fromRGB(33, 150, 243)
    end
end)

btn("⏸ PAUSAR", Color3.fromRGB(255, 152, 0), 265, function()
    pauseJJ()
    st.Text = "PAUSADO"
    st.TextColor3 = Color3.fromRGB(255, 152, 0)
end)

btn("▶ CONTINUAR", Color3.fromRGB(76, 175, 80), 305, function()
    resumeJJ()
    st.Text = "EXECUTANDO..."
    st.TextColor3 = Color3.fromRGB(33, 150, 243)
end)

btn("⏹ PARAR", Color3.fromRGB(244, 67, 54), 345, function()
    stopJJ()
    st.Text = "PRONTO"
    st.TextColor3 = Color3.fromRGB(76, 175, 80)
end)

-- Progresso
local progLabel = Instance.new("TextLabel", main)
progLabel.Size = UDim2.new(1, -30, 0, 20)
progLabel.Position = UDim2.new(0, 15, 0, 395)
progLabel.BackgroundTransparency = 1
progLabel.Text = "Progresso: 0/10"
progLabel.TextColor3 = Color3.fromRGB(150, 150, 170)
progLabel.Font = Enum.Font.Gotham
progLabel.TextSize = 10

-- Atualizador de status
task.spawn(function()
    while gui.Parent do
        if running then
            local key = detectKey()
            local keyName = key and (key == Enum.KeyCode.Q and "Q" or "E") or "?"
            progLabel.Text = "Progresso: "..doneCount.."/"..target.." | Tecla: "..keyName
        end
        task.wait(0.1)
    end
end)

-- Arrastar
local dr, ds, sp = false, nil, nil
t1.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then
        dr, ds, sp = true, i.Position, main.Position
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if dr and i.UserInputType == Enum.UserInputType.MouseMovement then
        local d = i.Position - ds
        main.Position = UDim2.new(sp.X.Scale, sp.X.Offset + d.X, sp.Y.Scale, sp.Y.Offset + d.Y)
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.MouseButton1 then dr = false end
end)

-- Atalho F2
UserInputService.InputBegan:Connect(function(i, g)
    if g then return end
    if i.KeyCode == Enum.KeyCode.F2 then
        target = tonumber(qtdInput.Text) or 10
        delayTime = tonumber(delayInput.Text) or 0.3
        if running then
            stopJJ()
            st.Text = "PRONTO"
            st.TextColor3 = Color3.fromRGB(76, 175, 80)
        else
            doneCount = 0
            startJJ()
            if running then
                st.Text = "EXECUTANDO..."
                st.TextColor3 = Color3.fromRGB(33, 150, 243)
            end
        end
    end
end)

print("Jeck de Calcinha Scripts - JJ's Auto carregado!")
print("Detecta Q/E automaticamente do jogo!")
print("F2 = Iniciar/Parar")
