local player = game.Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local PathfindingService = game:GetService("PathfindingService")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local ITENS_UTEIS = { "Chave", "Cartao", "Disco" }
local LOCAIS_USO = { Chave = "Porta", Cartao = "Computador", Disco = "Terminal" }
local ITENS_ATAQUE = { "Banana", "Bomba" }
local SAIDA = workspace:FindFirstChild("SaidaFinal")
local CAÇADOR = workspace:FindFirstChild("Caçador")
local SOBREVIVENTES = {}

local modo = nil
local itensUsados = {}
local menuGui
local ESPObjs = {}

-- Detecta armadilhas
function getArmadilhas()
    local traps = {}
    for _, obj in ipairs(workspace:GetChildren()) do
        if obj:IsA("BasePart") and obj.Name == "Armadilha" then
            table.insert(traps, obj)
        end
    end
    return traps
end

function isItemUtil(itemName)
    for _, nome in ipairs(ITENS_UTEIS) do
        if itemName == nome then return true end
    end
    return false
end

function isItemAtaque(itemName)
    for _, nome in ipairs(ITENS_ATAQUE) do
        if itemName == nome then return true end
    end
    return false
end

function getNextItem()
    for _, nome in ipairs(ITENS_UTEIS) do
        if not itensUsados[nome] then
            local item = workspace:FindFirstChild(nome)
            if item and item.Parent == workspace then
                return item
            end
        end
    end
    return nil
end

function getLocalUso(itemName)
    return workspace:FindFirstChild(LOCAIS_USO[itemName])
end

function distancia(obj1, obj2)
    return (obj1.Position - obj2.Position).Magnitude
end

-- Verifica se o caminho passa por armadilhas
function caminhoTemArmadilha(path)
    local traps = getArmadilhas()
    if not path or not path.Status == Enum.PathStatus.Complete then return false end
    for _, waypoint in ipairs(path:GetWaypoints()) do
        for _, trap in ipairs(traps) do
            if (waypoint.Position - trap.Position).Magnitude < 7 then -- raio de detecção da armadilha
                return true
            end
        end
    end
    return false
end

-- Faz caminho evitando armadilhas (se possível)
function moverAteEvitarArmadilha(objeto)
    if not objeto or not objeto.Position then return end
    local traps = getArmadilhas()
    local path = PathfindingService:CreatePath({
        AgentRadius = 2,
        AgentHeight = 5,
        AgentCanJump = true,
        Start = character.HumanoidRootPart.Position,
        End = objeto.Position,
    })
    path:ComputeAsync()
    if not caminhoTemArmadilha(path) then
        path:MoveTo(character)
        return
    end
    -- Se todas rotas com armadilhas, tenta a mais longe do caçador
    local caçadorHRP = CAÇADOR and CAÇADOR:FindFirstChild("HumanoidRootPart")
    local maisDistante = nil
    local maiorDist = -math.huge
    for _, trap in ipairs(traps) do
        local dist = caçadorHRP and distancia(trap, caçadorHRP) or 0
        if dist > maiorDist then
            maiorDist = dist
            maisDistante = trap
        end
    end
    if maisDistante then
        local pos = maisDistante.Position + (maisDistante.Position - (caçadorHRP and caçadorHRP.Position or Vector3.new())) * 2
        local pathEscape = PathfindingService:CreatePath({
            AgentRadius = 2,
            AgentHeight = 5,
            AgentCanJump = true,
            Start = character.HumanoidRootPart.Position,
            End = pos,
        })
        pathEscape:ComputeAsync()
        pathEscape:MoveTo(character)
    else
        path:MoveTo(character)
    end
end

function coletarItem(item)
    if not item then return end
    if distancia(character.HumanoidRootPart, item) < 5 then
        item.Parent = player.Backpack
        return true
    else
        moverAteEvitarArmadilha(item)
        return false
    end
end

function usarItem(itemName)
    local localUso = getLocalUso(itemName)
    if localUso then
        moverAteEvitarArmadilha(localUso)
        if distancia(character.HumanoidRootPart, localUso) < 7 then
            itensUsados[itemName] = true
            local backpackItem = player.Backpack:FindFirstChild(itemName)
            if backpackItem then
                backpackItem:Destroy()
            end
            return true
        end
    end
    return false
end

function irParaSaida()
    if SAIDA then
        moverAteEvitarArmadilha(SAIDA)
    end
end

function fugirDoCaçador()
    if not CAÇADOR or not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local caçadorHRP = CAÇADOR:FindFirstChild("HumanoidRootPart")
    if not caçadorHRP then return end
    local dir = (character.HumanoidRootPart.Position - caçadorHRP.Position).Unit
    local destino = character.HumanoidRootPart.Position + dir * 20
    moverAteEvitarArmadilha({Position = destino})
end

function perseguirSobrevivente()
    local alvo = nil
    local menorDist = math.huge
    for _, sobrevivente in ipairs(SOBREVIVENTES) do
        local hrp = sobrevivente.Character and sobrevivente.Character:FindFirstChild("HumanoidRootPart")
        if hrp and distancia(character.HumanoidRootPart, hrp) < menorDist then
            menorDist = distancia(character.HumanoidRootPart, hrp)
            alvo = hrp
        end
    end
    if alvo then moverAteEvitarArmadilha(alvo) end
end

function criarESP(target, color)
    if not target then return end
    local esp = Instance.new("BillboardGui")
    esp.Size = UDim2.new(0, 100, 0, 40)
    esp.Adornee = target
    esp.Parent = target
    esp.AlwaysOnTop = true
    esp.Name = "ESP"
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = target.Parent.Name
    label.TextColor3 = color
    label.TextScaled = true
    label.Parent = esp
    table.insert(ESPObjs, esp)
end

function limparESP()
    for _, esp in ipairs(ESPObjs) do
        if esp and esp.Parent then
            esp:Destroy()
        end
    end
    ESPObjs = {}
end

function atualizarSobreviventes()
    SOBREVIVENTES = {}
    for _, plr in ipairs(game.Players:GetPlayers()) do
        if plr ~= player and plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
            table.insert(SOBREVIVENTES, plr)
        end
    end
end

spawn(function()
    while true do
        wait(0.2)
        if modo == "sobrevivente" then
            if CAÇADOR and distancia(character.HumanoidRootPart, CAÇADOR.HumanoidRootPart) < 15 then
                fugirDoCaçador()
            else
                local item = getNextItem()
                if item then
                    if coletarItem(item) then
                        usarItem(item.Name)
                    end
                else
                    irParaSaida()
                end
            end
        elseif modo == "caçador" then
            atualizarSobreviventes()
            perseguirSobrevivente()
        end
    end
end)

RunService.RenderStepped:Connect(function()
    limparESP()
    if modo == "sobrevivente" and CAÇADOR and distancia(character.HumanoidRootPart, CAÇADOR.HumanoidRootPart) < 20 then
        criarESP(CAÇADOR.HumanoidRootPart, Color3.new(1,0,0))
    elseif modo == "caçador" then
        atualizarSobreviventes()
        for _, sobrevivente in ipairs(SOBREVIVENTES) do
            if sobrevivente.Character and sobrevivente.Character:FindFirstChild("HumanoidRootPart") then
                criarESP(sobrevivente.Character.HumanoidRootPart, Color3.new(0,1,0))
            end
        end
    end
end)

function criarMenu()
    menuGui = Instance.new("ScreenGui")
    menuGui.Name = "MenuRobo"
    menuGui.Parent = player:WaitForChild("PlayerGui")

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 160, 0, 50)
    frame.Position = UDim2.new(0, 60, 0, 60)
    frame.BackgroundColor3 = Color3.fromRGB(40,40,40)
    frame.BorderSizePixel = 2
    frame.BackgroundTransparency = 0 -- SEM transparência!
    frame.Parent = menuGui

    local btnSobrevivente = Instance.new("TextButton")
    btnSobrevivente.Size = UDim2.new(0.5, -2, 1, 0)
    btnSobrevivente.Position = UDim2.new(0,0,0,0)
    btnSobrevivente.Text = "Sobrevivente"
    btnSobrevivente.TextColor3 = Color3.new(1,1,1)
    btnSobrevivente.BackgroundColor3 = Color3.fromRGB(20,160,20)
    btnSobrevivente.Parent = frame

    local btnCacador = Instance.new("TextButton")
    btnCacador.Size = UDim2.new(0.5, -2, 1, 0)
    btnCacador.Position = UDim2.new(0.5,2,0,0)
    btnCacador.Text = "Caçador"
    btnCacador.TextColor3 = Color3.new(1,1,1)
    btnCacador.BackgroundColor3 = Color3.fromRGB(160,40,40)
    btnCacador.Parent = frame

    btnSobrevivente.MouseButton1Click:Connect(function()
        modo = "sobrevivente"
        btnSobrevivente.BackgroundColor3 = Color3.fromRGB(20,200,20)
        btnCacador.BackgroundColor3 = Color3.fromRGB(160,40,40)
    end)
    btnCacador.MouseButton1Click:Connect(function()
        modo = "caçador"
        btnCacador.BackgroundColor3 = Color3.fromRGB(200,40,40)
        btnSobrevivente.BackgroundColor3 = Color3.fromRGB(20,160,20)
    end)

    -- Movimentação do menu (mobile-friendly)
    local dragging, dragInput, dragStart, startPos
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    frame.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

criarMenu()
