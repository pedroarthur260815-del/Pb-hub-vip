-- Serviços do Roblox
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Configurações Iniciais
local assistenciaAtiva = false
local fovMaximo = 50 -- Começa em 50 (escala de 10 a 100)
local alvoAtual = nil

----------------------------------------------------------------
-- 1. CRIAÇÃO AUTOMÁTICA DA INTERFACE (UI) COM SUPORTE A TOQUE
----------------------------------------------------------------

local ScreenGui = script.Parent
ScreenGui.Name = "PbHubVip"

-- Botão de Abrir (Fica flutuando na tela)
local OpenButton = Instance.new("TextButton")
OpenButton.Name = "OpenButton"
OpenButton.Size = UDim2.new(0, 100, 0, 40)
OpenButton.Position = UDim2.new(0, 10, 0.4, 0)
OpenButton.Text = "Abrir Pb Hub"
OpenButton.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
OpenButton.TextColor3 = Color3.fromRGB(255, 255, 255)
OpenButton.Font = Enum.Font.SourceSansBold
OpenButton.TextSize = 16
OpenButton.Parent = ScreenGui

-- Painel Principal (Pb Hub vip)
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 250, 0, 200)
MainFrame.Position = UDim2.new(0.5, -125, 0.5, -100)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
MainFrame.BorderSizePixel = 2
MainFrame.Visible = false
MainFrame.Parent = ScreenGui

-- Título do Painel
local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.Text = "Pb Hub vip - Painel de Mira"
Title.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
Title.TextColor3 = Color3.fromRGB(255, 215, 0) -- Dourado VIP
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 18
Title.Parent = MainFrame

-- Botão Fechar
local CloseButton = Instance.new("TextButton")
CloseButton.Size = UDim2.new(0, 30, 0, 30)
CloseButton.Position = UDim2.new(1, -30, 0, 0)
CloseButton.Text = "X"
CloseButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.Parent = MainFrame

-- Botão Ativar/Desativar Mira Assistida
local ToggleButton = Instance.new("TextButton")
ToggleButton.Size = UDim2.new(0, 200, 0, 40)
ToggleButton.Position = UDim2.new(0.5, -100, 0.3, 0)
ToggleButton.Text = "Mira: DESATIVADA"
ToggleButton.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
ToggleButton.Font = Enum.Font.SourceSansBold
ToggleButton.TextSize = 16
ToggleButton.Parent = MainFrame

-- Botão/Medidor do FOV (De 10 a 100)
local FovButton = Instance.new("TextButton")
FovButton.Size = UDim2.new(0, 200, 0, 40)
FovButton.Position = UDim2.new(0.5, -100, 0.6, 0)
FovButton.Text = "Tamanho do FOV: " .. fovMaximo
FovButton.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
FovButton.TextColor3 = Color3.fromRGB(255, 255, 255)
FovButton.Font = Enum.Font.SourceSansBold
FovButton.TextSize = 16
FovButton.Parent = MainFrame

-- Círculo Visual do FOV na Tela (Para o jogador ver o limite do Hitbox)
local FovCircle = Instance.new("Frame")
FovCircle.Name = "FovCircle"
FovCircle.AnchorPoint = Vector2.new(0.5, 0.5)
FovCircle.Position = UDim2.new(0.5, 0, 0.5, 0)
FovCircle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
FovCircle.BackgroundTransparency = 0.9 -- Quase transparente
FovCircle.Visible = false
FovCircle.Parent = ScreenGui

-- Deixar o círculo redondo
local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(1, 0)
UICorner.Parent = FovCircle

----------------------------------------------------------------
-- 2. SISTEMA DE TOQUE E CLIQUES (INTERATIVIDADE)
----------------------------------------------------------------

-- Abrir
OpenButton.Activated:Connect(function()
	MainFrame.Visible = true
	OpenButton.Visible = false
end)

-- Fechar
CloseButton.Activated:Connect(function()
	MainFrame.Visible = false
	OpenButton.Visible = true
end)

-- Alternar Ativado/Desativar
ToggleButton.Activated:Connect(function()
	assistenciaAtiva = not assistenciaAtiva
	FovCircle.Visible = assistenciaAtiva
	if assistenciaAtiva then
		ToggleButton.Text = "Mira: ATIVADA"
		ToggleButton.BackgroundColor3 = Color3.fromRGB(50, 150, 50)
	else
		ToggleButton.Text = "Mira: DESATIVADA"
		ToggleButton.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
		alvoAtual = nil
	end
end)

-- Mudar tamanho do FOV (Mecânica de 10 a 100)
FovButton.Activated:Connect(function()
	fovMaximo = fovMaximo + 15
	if fovMaximo > 100 then
		fovMaximo = 10 -- Reseta para o mínimo
	end
	FovButton.Text = "Tamanho do FOV: " .. fovMaximo
end)

----------------------------------------------------------------
-- 3. LÓGICA DE ASSISTÊNCIA DE MIRA (MATEMÁTICA E RENDER)
----------------------------------------------------------------

-- Atualiza o tamanho físico do círculo na tela com base no valor do FOV
local function atualizarRaioFov()
	-- Multiplicamos por 5 para converter a escala 10-100 em pixels visíveis na tela
	local tamanhoEmPixels = fovMaximo * 5 
	FovCircle.Size = UDim2.new(0, tamanhoEmPixels, 0, tamanhoEmPixels)
end

-- Função para achar o inimigo mais próximo vivo que esteja dentro do círculo de FOV
local function obterInimigoMaisProximo()
	local meuChar = LocalPlayer.Character
	if not meuChar or not meuChar:FindFirstChild("HumanoidRootPart") then return nil end

	local melhorAlvo = nil
	local menorDistanciaDaTela = math.huge -- Medido em pixels a partir do centro da tela

	for _, jogador in pairs(Players:GetPlayers()) do
		if jogador ~= LocalPlayer then
			local char = jogador.Character
			
			-- 1. Verifica se o personagem existe e tem cabeça/corpo
			if char and char:FindFirstChild("Head") and char:FindFirstChild("Humanoid") then
				local humanoid = char.Humanoid
				
				-- 2. DETECÇÃO DE MORTE: Só mira se estiver vivo (Health > 0)
				if humanoid.Health > 0 then
					-- Converte a posição 3D do personagem para a posição 2D da tela do celular/PC
					local posicaoTela, naTela = Camera:WorldToViewportPoint(char.Head.Position)
					
					if naTela then
						-- Calcula a distância entre o centro da tela e onde o jogador está na tela
						local centroTela = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
						local posicaoInimigo2D = Vector2.new(posicaoTela.X, posicaoTela.Y)
						local distanciaDoCentro = (centroTela - posicaoInimigo2D).Magnitude
						
						-- 3. Validação do Hitbox/FOV: Precisa estar dentro do raio escolhido (10 a 100)
						local raioMaximoPixels = fovMaximo * 2.5
						if distanciaDoCentro <= raioMaximoPixels then
							if distanciaDoCentro < menorDistanciaDaTela then
								menorDistanciaDaTela = distanciaDoCentro
								melhorAlvo = char
							end
						end
					end
				end
			end
		end
	end
	return melhorAlvo
end

-- LOOP PRINCIPAL: Roda em cada atualização de frame na tela
RunService.RenderStepped:Connect(function()
	if not assistenciaAtiva then return end
	
	atualizarRaioFov()
	
	-- FORÇA DO AIMBOT: Se o alvo atual morreu ou saiu do FOV, ele limpa o alvo para buscar o próximo
	if alvoAtual then
		if not alvoAtual:FindFirstChild("Humanoid") or alvoAtual.Humanoid.Health <= 0 then
			alvoAtual = nil -- Inimigo morreu, limpa para trocar instantaneamente
		end
	end
	
	-- Se está sem alvo, busca o mais próximo disponível
	if not alvoAtual then
		alvoAtual = obterInimigoMaisProximo()
	end
	
	-- Se achou o alvo, move a câmera suavemente em direção à cabeça dele
	if alvoAtual and alvoAtual:FindFirstChild("Head") then
		local posicaoAlvo = alvoAtual.Head.Position
		
		-- Cria a rotação da câmera apontando para o alvo
		local cframeObjetivo = CFrame.new(Camera.CFrame.Position, posicaoAlvo)
		
		-- O valor 0.15 é a força de retenção/suavidade. 
		-- Quanto maior, mais rápido ele gruda. 0.15 é excelente para mobile não travar.
		Camera.CFrame = Camera.CFrame:Lerp(cframeObjetivo, 0.15)
	end
end)

