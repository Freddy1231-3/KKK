-- ==========================================
-- DRIP MENU + GRAVADOR E GERENCIADOR DE ROTAS
-- ==========================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoidRootPart = character:WaitForChild("HumanoidRootPart")

-- Variáveis do Gravador
local gravando = false
local reproduzindo = false
local rotaAtual = {}
local tempoInicial = 0

-- Configurações de Arquivos
local PASTA_ROTAS = "DripRotas"
if not isfolder(PASTA_ROTAS) then
    makefolder(PASTA_ROTAS)
end

-- Verifica se o executor suporta arquivos
local suportaArquivos = writefile and readfile and isfolder and makefolder

-- ==========================================
-- 1. CRIANDO A INTERFACE (UI)
-- ==========================================

if CoreGui:FindFirstChild("DripMenuUI") then
    CoreGui:FindFirstChild("DripMenuUI"):Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "DripMenuUI"
ScreenGui.Parent = CoreGui
ScreenGui.ResetOnSpawn = false

-- Botão para abrir/fechar o menu
local ToggleButton = Instance.new("TextButton")
ToggleButton.Parent = ScreenGui
ToggleButton.Size = UDim2.new(0, 50, 0, 50)
ToggleButton.Position = UDim2.new(0, 20, 0, 20)
ToggleButton.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
ToggleButton.Text = "👑"
ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleButton.TextSize = 24
ToggleButton.Font = Enum.Font.GothamBold
ToggleButton.BorderSizePixel = 0
ToggleButton.Visible = true
Instance.new("UICorner", ToggleButton).CornerRadius = UDim.new(0, 10)

-- Frame Principal (Menu)
local MainFrame = Instance.new("Frame")
MainFrame.Parent = ScreenGui
MainFrame.Size = UDim2.new(0, 600, 0, 400)
MainFrame.Position = UDim2.new(0.5, -300, 0.5, -200)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
MainFrame.BorderSizePixel = 0
MainFrame.Visible = false
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 12)

-- Título "Drip Menu"
local TitleLabel = Instance.new("TextLabel")
TitleLabel.Parent = MainFrame
TitleLabel.Size = UDim2.new(1, -40, 0, 40)
TitleLabel.Position = UDim2.new(0, 20, 0, 10)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "👑 Drip Menu"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.TextSize = 22
TitleLabel.Font = Enum.Font.GothamBold
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
TitleLabel.ZIndex = 2

-- Botão de Fechar (X)
local CloseButton = Instance.new("TextButton")
CloseButton.Parent = MainFrame
CloseButton.Size = UDim2.new(0, 30, 0, 30)
CloseButton.Position = UDim2.new(1, -40, 0, 10)
CloseButton.BackgroundTransparency = 1
CloseButton.Text = "✖"
CloseButton.TextColor3 = Color3.fromRGB(255, 100, 100)
CloseButton.TextSize = 18
CloseButton.Font = Enum.Font.GothamBold
CloseButton.ZIndex = 2

-- Container dos Botões (Lado Esquerdo)
local ButtonContainer = Instance.new("Frame")
ButtonContainer.Parent = MainFrame
ButtonContainer.Size = UDim2.new(0, 150, 1, -60)
ButtonContainer.Position = UDim2.new(0, 10, 0, 50)
ButtonContainer.BackgroundTransparency = 1
ButtonContainer.ZIndex = 2

-- Lista de Botões do Menu
local menuItems = {
    {name = "Config", icon = "⚙️"},
    {name = "Parkours", icon = "🏃"},
    {name = "Torres", icon = "🏰"},
    {name = "Civils", icon = "🛡️"},
    {name = "Gravador", icon = "🎥"},
    {name = "Fazenda", icon = "🌿"},
    {name = "Characters", icon = "👤"},
    {name = "ESP", icon = "👁️"},
    {name = "Rádio", icon = "🎵"}
}

local botoes = {}
for i, item in ipairs(menuItems) do
    local btn = Instance.new("TextButton")
    btn.Parent = ButtonContainer
    btn.Size = UDim2.new(1, 0, 0, 30)
    btn.Position = UDim2.new(0, 0, 0, (i-1) * 32)
    btn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    btn.BackgroundTransparency = 0.5
    btn.Text = "  " .. item.icon .. "  " .. item.name
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.TextSize = 14
    btn.Font = Enum.Font.Gotham
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.BorderSizePixel = 0
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    botoes[item.name] = btn
end

-- Painel de Conteúdo (Lado Direito)
local ContentPanel = Instance.new("Frame")
ContentPanel.Parent = MainFrame
ContentPanel.Size = UDim2.new(0, 410, 1, -70)
ContentPanel.Position = UDim2.new(0, 170, 0, 50)
ContentPanel.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
ContentPanel.BackgroundTransparency = 0.3
ContentPanel.BorderSizePixel = 0
Instance.new("UICorner", ContentPanel).CornerRadius = UDim.new(0, 8)
ContentPanel.ZIndex = 2

-- ==========================================
-- 2. FUNÇÕES AUXILIARES (Arquivos e UI)
-- ==========================================

local function cframeParaTabela(cf)
    return {cf:GetComponents()}
end

local function tabelaParaCFrame(t)
    return CFrame.new(unpack(t))
end

local function salvarRotaNoArquivo(nome, dados)
    if not suportaArquivos then return false end
    local caminho = PASTA_ROTAS .. "/" .. nome .. ".json"
    local sucesso, erro = pcall(function()
        writefile(caminho, HttpService:JSONEncode(dados))
    end)
    return sucesso, erro
end

local function carregarRotaDoArquivo(nome)
    if not suportaArquivos then return nil end
    local caminho = PASTA_ROTAS .. "/" .. nome .. ".json"
    if isfile(caminho) then
        local sucesso, conteudo = pcall(function()
            return readfile(caminho)
        end)
        if sucesso then
            return HttpService:JSONDecode(conteudo)
        end
    end
    return nil
end

local function deletarRotaDoArquivo(nome)
    if not suportaArquivos then return false end
    local caminho = PASTA_ROTAS .. "/" .. nome .. ".json"
    if isfile(caminho) then
        delfile(caminho)
        return true
    end
    return false
end

local function listarRotasSalvas()
    if not suportaArquivos then return {} end
    local rotas = {}
    local arquivos = listfiles(PASTA_ROTAS)
    for _, arquivo in ipairs(arquivos) do
        local nome = arquivo:match("([^/\\]+)%.json$")
        if nome then
            table.insert(rotas, nome)
        end
    end
    return rotas
end

-- ==========================================
-- 3. LÓGICA DO GRAVADOR E GERENCIADOR
-- ==========================================

local function limparPainel()
    for _, child in ipairs(ContentPanel:GetChildren()) do
        child:Destroy()
    end
end

local function criarBotao(texto, posX, posY, tamX, tamY, cor, funcao, parent)
    local btn = Instance.new("TextButton")
    btn.Parent = parent or ContentPanel
    btn.Size = UDim2.new(0, tamX, 0, tamY)
    btn.Position = UDim2.new(0, posX, 0, posY)
    btn.BackgroundColor3 = cor
    btn.Text = texto
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.TextSize = 12
    btn.Font = Enum.Font.GothamBold
    btn.BorderSizePixel = 0
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
    btn.MouseButton1Click:Connect(funcao)
    return btn
end

-- Função para iniciar gravação
local function iniciarGravacao()
    gravando = true
    reproduzindo = false
    rotaAtual = {}
    tempoInicial = tick()
    print("🔴 Gravando rota...")
end

-- Função para parar gravação e salvar
local function pararESalvar(nomeRota)
    if not gravando then return end
    gravando = false
    print("⏹️ Gravação parada. Total de frames: " .. #rotaAtual)
    
    if #rotaAtual == 0 then return end
    if nomeRota == "" then nomeRota = "Rota_Sem_Nome" end
    
    -- Converte CFrames para tabelas para salvar em JSON
    local dadosParaSalvar = {}
    for _, frame in ipairs(rotaAtual) do
        table.insert(dadosParaSalvar, {
            cf = cframeParaTabela(frame.cf),
            t = frame.t
        })
    end
    
    local sucesso = salvarRotaNoArquivo(nomeRota, dadosParaSalvar)
    if sucesso then
        print("✅ Rota '" .. nomeRota .. "' salva com sucesso!")
    else
        warn("❌ Falha ao salvar a rota. Verifique se o executor suporta arquivos.")
    end
end

-- Função para reproduzir rota
local function iniciarReproducao(nomeRota)
    local dados = carregarRotaDoArquivo(nomeRota)
    if not dados or #dados == 0 then
        warn("⚠️ Rota não encontrada ou vazia.")
        return
    end
    
    reproduzindo = true
    gravando = false
    print("▶️ Reproduzindo rota: " .. nomeRota)
    
    task.spawn(function()
        local tempoInicioPlay = tick()
        local tempoInicioRota = dados[1].t
        
        for _, frame in ipairs(dados) do
            if not reproduzindo then break end
            
            local espera = (frame.t - tempoInicioRota) - (tick() - tempoInicioPlay)
            if espera > 0 then task.wait(espera) end
            
            if humanoidRootPart and humanoidRootPart.Parent then
                humanoidRootPart.CFrame = tabelaParaCFrame(frame.cf)
            else
                break
            end
        end
        reproduzindo = false
        print("✅ Rota finalizada.")
    end)
end

-- ==========================================
-- 4. CONSTRUINDO A ABA GRAVADOR
-- ==========================================

local function construirAbaGravador()
    limparPainel()
    
    local titulo = Instance.new("TextLabel")
    titulo.Parent = ContentPanel
    titulo.Size = UDim2.new(1, 0, 0, 30)
    titulo.Position = UDim2.new(0, 0, 0, 5)
    titulo.BackgroundTransparency = 1
    titulo.Text = "🎥 Gerenciador de Rotas"
    titulo.TextColor3 = Color3.fromRGB(255, 255, 255)
    titulo.TextSize = 16
    titulo.Font = Enum.Font.GothamBold

    -- Caixa de Texto para Nome da Rota
    local txtNome = Instance.new("TextBox")
    txtNome.Parent = ContentPanel
    txtNome.Size = UDim2.new(0, 200, 0, 30)
    txtNome.Position = UDim2.new(0, 15, 0, 40)
    txtNome.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    txtNome.TextColor3 = Color3.fromRGB(255, 255, 255)
    txtNome.PlaceholderText = "Nome da Rota..."
    txtNome.Text = ""
    txtNome.Font = Enum.Font.Gotham
    txtNome.TextSize = 12
    txtNome.BorderSizePixel = 0
    Instance.new("UICorner", txtNome).CornerRadius = UDim.new(0, 6)

    -- Botões de Ação Principais
    local btnGravar = criarBotao("🔴 Iniciar Gravação", 15, 80, 100, 30, Color3.fromRGB(200, 50, 50), function()
        if gravando then
            pararESalvar(txtNome.Text)
        else
            iniciarGravacao()
        end
    end)
    
    criarBotao("▶️ Reproduzir", 120, 80, 95, 30, Color3.fromRGB(50, 200, 50), function()
        iniciarReproducao(txtNome.Text)
    end)

    criarBotao("⏹️ Parar Play", 220, 80, 80, 30, Color3.fromRGB(100, 100, 100), function()
        reproduzindo = false
        gravando = false
    end)

    -- Lista de Rotas Salvas (ScrollingFrame)
    local listaFrame = Instance.new("ScrollingFrame")
    listaFrame.Parent = ContentPanel
    listaFrame.Size = UDim2.new(0, 380, 1, -170)
    listaFrame.Position = UDim2.new(0, 15, 0, 120)
    listaFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    listaFrame.BorderSizePixel = 0
    listaFrame.ScrollBarThickness = 4
    Instance.new("UICorner", listaFrame).CornerRadius = UDim.new(0, 6)
    
    local layout = Instance.new("UIListLayout")
    layout.Parent = listaFrame
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Padding = UDim.new(0, 5)

    -- Função para atualizar a lista
    local function atualizarLista()
        for _, child in ipairs(listaFrame:GetChildren()) do
            if child:IsA("Frame") then child:Destroy() end
        end
        
        local rotas = listarRotasSalvas()
        if #rotas == 0 then
            local aviso = Instance.new("TextLabel")
            aviso.Parent = listaFrame
            aviso.Size = UDim2.new(1, 0, 0, 30)
            aviso.BackgroundTransparency = 1
            aviso.Text = "Nenhuma rota salva."
            aviso.TextColor3 = Color3.fromRGB(150, 150, 150)
            aviso.Font = Enum.Font.Gotham
            aviso.TextSize = 12
            return
        end

        for i, nome in ipairs(rotas) do
            local item = Instance.new("Frame")
            item.Parent = listaFrame
            item.Size = UDim2.new(1, -10, 0, 35)
            item.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
            item.BorderSizePixel = 0
            Instance.new("UICorner", item).CornerRadius = UDim.new(0, 4)

            local label = Instance.new("TextLabel")
            label.Parent = item
            label.Size = UDim2.new(0, 150, 1, 0)
            label.Position = UDim2.new(0, 5, 0, 0)
            label.BackgroundTransparency = 1
            label.Text = nome
            label.TextColor3 = Color3.fromRGB(255, 255, 255)
            label.TextSize = 12
            label.Font = Enum.Font.Gotham
            label.TextXAlignment = Enum.TextXAlignment.Left

            -- Botão Reproduzir
            criarBotao("▶️", 160, 0, 30, 35, Color3.fromRGB(50, 200, 50), function()
                iniciarReproducao(nome)
            end, item)

            -- Botão Renomear
            criarBotao("✏️", 195, 0, 30, 35, Color3.fromRGB(200, 150, 50), function()
                local novoNome = txtNome.Text
                if novoNome ~= "" and novoNome ~= nome then
                    local dados = carregarRotaDoArquivo(nome)
                    if dados then
                        deletarRotaDoArquivo(nome)
                        salvarRotaNoArquivo(novoNome, dados)
                        atualizarLista()
                    end
                end
            end, item)

            -- Botão Deletar
            criarBotao("🗑️", 230, 0, 30, 35, Color3.fromRGB(200, 50, 50), function()
                deletarRotaDoArquivo(nome)
                atualizarLista()
            end, item)
        end
    end

    -- Botão de Atualizar Lista
    criarBotao("🔄 Atualizar Lista", 310, 80, 85, 30, Color3.fromRGB(80, 80, 80), atualizarLista)

    -- Conecta o clique de gravar para mudar o texto do botão
    btnGravar.MouseButton1Click:Connect(function()
        if gravando then
            btnGravar.Text = "🔴 Iniciar Gravação"
            atualizarLista()
        else
            btnGravar.Text = "⏹️ Parar e Salvar"
        end
    end)

    atualizarLista()
end

-- ==========================================
-- 5. CONECTANDO O MENU
-- ==========================================

botoes["Gravador"].MouseButton1Click:Connect(construirAbaGravador)

-- Outros botões (Apenas visual)
for nome, btn in pairs(botoes) do
    if nome ~= "Gravador" then
        btn.MouseButton1Click:Connect(function()
            limparPainel()
            local aviso = Instance.new("TextLabel")
            aviso.Parent = ContentPanel
            aviso.Size = UDim2.new(1, -20, 1, -20)
            aviso.Position = UDim2.new(0, 10, 0, 10)
            aviso.BackgroundTransparency = 1
            aviso.Text = "Função '" .. nome .. "' ainda não implementada."
            aviso.TextColor3 = Color3.fromRGB(150, 150, 150)
            aviso.Font = Enum.Font.Gotham
            aviso.TextSize = 14
        end)
    end
end

-- ==========================================
-- 6. SISTEMA DE ARRASTAR E ABRIR/FECHAR
-- ==========================================

ToggleButton.MouseButton1Click:Connect(function()
    MainFrame.Visible = not MainFrame.Visible
end)

CloseButton.MouseButton1Click:Connect(function()
    MainFrame.Visible = false
end)

local dragging = false
local dragInput, dragStart, startPos

local function update(input)
    local delta = input.Position - dragStart
    MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end

MainFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)

MainFrame.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then update(input) end
end)

print("✅ Drip Menu com Gerenciador de Rotas carregado!")
