--[[
    Vitin Hub (CPX TG Edition)
    - Fly estilo Infinity Yield (CFrame, sem bugs, WASD/Space/Ctrl, PlatformStand)
    - Teleport gradual (anti-rollback)
    - Hotkeys personalizáveis
    - Minimizar (CTRL restaura), Fechar (confirmação)
    - ESP compatível com Drawing API
    - Abas: Aimbot, ESP, Outros
    - Key igual antes
    Feito especial para CPX do TG por vitin-scripter
]]

local KEY_CORRETA = "vitin1234"
local DEFAULT_FLY_HOTKEY = Enum.KeyCode.B
local DEFAULT_AIMBOT_HOTKEY = Enum.KeyCode.Q

local FLY_SPEED_MIN, FLY_SPEED_MAX = 1, 100
local flySpeed = 2

local ESP_RANGE_MIN = 100
local ESP_RANGE_MAX = workspace:GetChildren() and #workspace:GetChildren()*100 or 10000
local espRange = ESP_RANGE_MAX

local AIMBOT_SMOOTH_MIN, AIMBOT_SMOOTH_MAX = 0.05, 1.00
local aimbotSmooth = 0.15

local AIMBOT_FOV_MIN, AIMBOT_FOV_MAX = 50, 1000  -- <--- AGORA AIMFOV É DE 50 A 1000
local aimbotFOV = 100 -- Valor padrão do aimfov

local currentFlyHotkey = DEFAULT_FLY_HOTKEY
local currentFlyMouseButton = nil
local currentAimbotHotkey = DEFAULT_AIMBOT_HOTKEY
local currentAimbotMouseButton = nil
local flyEnabled, aimbotEnabled, espEnabled = false, false, false

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

local lastNotification = nil
local hubFrame
local aimFOVCircle -- Para desenhar o AimFOV

-- Notification
local function showNotification(msg, tipo)
    if lastNotification then pcall(function() lastNotification:Destroy() end) end
    local ScreenGui = game:GetService("CoreGui"):FindFirstChild("VitinHub_Notify") or Instance.new("ScreenGui", game:GetService("CoreGui"))
    ScreenGui.Name = "VitinHub_Notify"
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 250, 0, 40)
    frame.Position = UDim2.new(0.5, -125, 0.05, 0)
    frame.BackgroundColor3 = tipo == "error" and Color3.fromRGB(30,30,30) or Color3.fromRGB(40,40,40)
    frame.BorderSizePixel = 0
    frame.BackgroundTransparency = 0.1
    frame.Parent = ScreenGui
    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(1,0,1,0)
    txt.BackgroundTransparency = 1
    txt.Font = Enum.Font.GothamBold
    txt.TextSize = 18
    txt.TextColor3 = tipo == "error" and Color3.fromRGB(255,70,70) or Color3.fromRGB(200,200,200)
    txt.Text = msg
    txt.Parent = frame
    lastNotification = frame
    TweenService:Create(frame, TweenInfo.new(0.5), {BackgroundTransparency=0}):Play()
    delay(2, function()
        TweenService:Create(frame, TweenInfo.new(0.4), {BackgroundTransparency=1}):Play()
        wait(0.5)
        pcall(function() frame:Destroy() end)
    end)
end

local function createConfirmPanel(parent, text, onYes, onNo)
    local confirmGui = Instance.new("ScreenGui", game:GetService("CoreGui"))
    confirmGui.Name = "VitinHub_Confirm"
    local frame = Instance.new("Frame", confirmGui)
    frame.Size = UDim2.new(0, 320, 0, 140)
    frame.Position = UDim2.new(0.5, -160, 0.45, -70)
    frame.BackgroundColor3 = Color3.fromRGB(25,25,25)
    frame.BorderSizePixel = 0
    frame.AnchorPoint = Vector2.new(0.5,0.5)
    local label = Instance.new("TextLabel", frame)
    label.Size = UDim2.new(1,0,0.6,0)
    label.Position = UDim2.new(0,0,0,0)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.GothamBold
    label.TextSize = 20
    label.TextColor3 = Color3.fromRGB(220,220,220)
    label.Text = text
    local yesBtn = Instance.new("TextButton", frame)
    yesBtn.Text = "Sim"
    yesBtn.Size = UDim2.new(0.48,0,0.28,0)
    yesBtn.Position = UDim2.new(0.02,0,0.7,0)
    yesBtn.BackgroundColor3 = Color3.fromRGB(40,120,40)
    yesBtn.TextColor3 = Color3.new(1,1,1)
    yesBtn.Font = Enum.Font.GothamBold
    yesBtn.TextSize = 18
    yesBtn.MouseButton1Click:Connect(function()
        confirmGui:Destroy()
        onYes()
    end)
    local noBtn = Instance.new("TextButton", frame)
    noBtn.Text = "Não"
    noBtn.Size = UDim2.new(0.48,0,0.28,0)
    noBtn.Position = UDim2.new(0.5,0,0.7,0)
    noBtn.BackgroundColor3 = Color3.fromRGB(120,40,40)
    noBtn.TextColor3 = Color3.new(1,1,1)
    noBtn.Font = Enum.Font.GothamBold
    noBtn.TextSize = 18
    noBtn.MouseButton1Click:Connect(function()
        confirmGui:Destroy()
        if onNo then onNo() end
    end)
end

local function getInputString(hotkey, mouseBtn)
    if mouseBtn then
        local map = {[Enum.UserInputType.MouseButton1]="Mouse1",[Enum.UserInputType.MouseButton2]="Mouse2",[Enum.UserInputType.MouseButton3]="Mouse3"}
        return map[mouseBtn] or "Mouse?"
    end
    if hotkey then
        return tostring(hotkey):match("%w+")
    end
    return "Nenhum"
end

local function shakeKeyField(field)
    local origPos = field.Position
    for i=1,4 do
        field.Position = origPos + UDim2.new(0,math.random(-10,10),0,0)
        wait(0.04)
    end
    field.Position = origPos
end

local function makeDraggable(frame)
    local dragging, dragInput, dragStart, startPos
    frame.Active = true
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
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
        if input.UserInputType == Enum.UserInputType.MouseMovement then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            frame.Position = startPos + UDim2.new(0, delta.X, 0, delta.Y)
        end
    end)
end

local function createKeyScreen()
    local gui = Instance.new("ScreenGui", game:GetService("CoreGui"))
    gui.Name = "VitinHub_Key"
    local frame = Instance.new("Frame", gui)
    frame.Size = UDim2.new(0,340,0,140)
    frame.Position = UDim2.new(0.5,-170,0.45,-70)
    frame.BackgroundColor3 = Color3.fromRGB(20,20,20)
    frame.BorderSizePixel = 0
    frame.AnchorPoint = Vector2.new(0.5,0.5)
    local title = Instance.new("TextLabel", frame)
    title.Size = UDim2.new(1,0,0.3,0)
    title.Position = UDim2.new(0,0,0,0)
    title.BackgroundTransparency = 1
    title.Text = "Vitin Hub 🔑"
    title.Font = Enum.Font.GothamBold
    title.TextSize = 28
    title.TextColor3 = Color3.fromRGB(150,150,150)
    local input = Instance.new("TextBox", frame)
    input.Size = UDim2.new(0.7,0,0.28,0)
    input.Position = UDim2.new(0.15,0,0.45,0)
    input.PlaceholderText = "Digite a key..."
    input.Text = ""
    input.BackgroundColor3 = Color3.fromRGB(30,30,30)
    input.Font = Enum.Font.GothamSemibold
    input.TextSize = 18
    input.TextColor3 = Color3.fromRGB(220,220,220)
    input.ClearTextOnFocus = false
    input.Selectable = true
    input.Active = true
    input.FocusLost:Connect(function(enter)
        if enter then
            if input.Text == KEY_CORRETA then
                gui:Destroy()
                showNotification("Key correta! Bem-vindo ao Vitin Hub.", "success")
                createHub()
            else
                showNotification("Key inválida!", "error")
                shakeKeyField(input)
                input.Text = ""
            end
        end
    end)
    local btn = Instance.new("TextButton", frame)
    btn.Size = UDim2.new(0.7,0,0.22,0)
    btn.Position = UDim2.new(0.15,0,0.75,0)
    btn.BackgroundColor3 = Color3.fromRGB(40,40,40)
    btn.Text = "CONFIRMAR"
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 18
    btn.TextColor3 = Color3.fromRGB(200,200,200)
    btn.MouseButton1Click:Connect(function()
        if input.Text == KEY_CORRETA then
            gui:Destroy()
            showNotification("Key correta! Bem-vindo ao Vitin Hub.", "success")
            createHub()
        else
            showNotification("Key inválida!", "error")
            shakeKeyField(input)
            input.Text = ""
        end
    end)
    input:CaptureFocus()
end

local function createTeleportPanel(parentFrame)
    local panel = Instance.new("Frame", parentFrame)
    panel.Size = UDim2.new(0.93, 0, 0.5, 0)
    panel.Position = UDim2.new(0.03, 0, 0.28, 0)
    panel.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    panel.BorderSizePixel = 0
    panel.Name = "TPPanel"
    local tptitle = Instance.new("TextLabel", panel)
    tptitle.Size = UDim2.new(1,0,0.11,0)
    tptitle.Position = UDim2.new(0,0,0,0)
    tptitle.BackgroundTransparency = 1
    tptitle.Font = Enum.Font.GothamBold
    tptitle.TextSize = 18
    tptitle.TextColor3 = Color3.fromRGB(200,200,200)
    tptitle.Text = "Teleportar para alguém"
    local scroll = Instance.new("ScrollingFrame", panel)
    scroll.Size = UDim2.new(1,-10,0.72,0)
    scroll.Position = UDim2.new(0,5,0.12,0)
    scroll.BackgroundTransparency = 1
    scroll.BorderSizePixel = 0
    scroll.CanvasSize = UDim2.new(0,0,0,0)
    scroll.ScrollBarThickness = 6
    local selectedPlayer = nil
    local function refreshPlayerList()
        scroll:ClearAllChildren()
        local y = 0
        for _,plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
                local btn = Instance.new("TextButton", scroll)
                btn.Size = UDim2.new(1, -10, 0, 32)
                btn.Position = UDim2.new(0, 5, 0, y)
                btn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
                btn.Text = plr.DisplayName .. " (" .. plr.Name .. ")"
                btn.Font = Enum.Font.GothamSemibold
                btn.TextSize = 15
                btn.TextColor3 = Color3.fromRGB(220,220,220)
                btn.TextXAlignment = Enum.TextXAlignment.Left
                btn.AutoButtonColor = true
                btn.BorderSizePixel = 0
                btn.MouseButton1Click:Connect(function()
                    selectedPlayer = plr
                    for _,b in ipairs(scroll:GetChildren()) do
                        if b:IsA("TextButton") then
                            b.BackgroundColor3 = Color3.fromRGB(30,30,30)
                        end
                    end
                    btn.BackgroundColor3 = Color3.fromRGB(60,120,220)
                end)
                y = y + 36
            end
        end
        scroll.CanvasSize = UDim2.new(0,0,0,math.max(y,180))
    end
    local tpBtn = Instance.new("TextButton", panel)
    tpBtn.Size = UDim2.new(0.7,0,0.13,0)
    tpBtn.Position = UDim2.new(0.15,0,0.87,0)
    tpBtn.BackgroundColor3 = Color3.fromRGB(40, 120, 40)
    tpBtn.Font = Enum.Font.GothamBold
    tpBtn.TextSize = 18
    tpBtn.TextColor3 = Color3.new(1,1,1)
    tpBtn.Text = "TP"
    tpBtn.MouseButton1Click:Connect(function()
        if selectedPlayer and selectedPlayer.Character and selectedPlayer.Character:FindFirstChild("HumanoidRootPart") and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
            local hrp = LocalPlayer.Character.HumanoidRootPart
            local destino = selectedPlayer.Character.HumanoidRootPart.Position + Vector3.new(0,3,0)
            local passos = 30
            local delta = (destino - hrp.Position) / passos
            for i=1,passos do
                hrp.CFrame = hrp.CFrame + delta
                wait(0.02)
            end
            showNotification("Teleportado para "..selectedPlayer.DisplayName, "success")
        else
            showNotification("Selecione um player ativo para teleportar!", "error")
        end
    end)
    refreshPlayerList()
    Players.PlayerAdded:Connect(refreshPlayerList)
    Players.PlayerRemoving:Connect(refreshPlayerList)
    return panel
end

function createHub()
    local gui = Instance.new("ScreenGui", game:GetService("CoreGui"))
    gui.Name = "VitinHub_Main"
    local frame = Instance.new("Frame", gui)
    frame.Size = UDim2.new(0,520,0,500)
    frame.Position = UDim2.new(0.5,-260,0.15,0)
    frame.BackgroundColor3 = Color3.fromRGB(15,15,15)
    frame.BorderSizePixel = 0
    frame.AnchorPoint = Vector2.new(0.5,0)
    makeDraggable(frame)
    hubFrame = frame

    -- Minimize & Close Buttons
    local minBtn = Instance.new("TextButton", frame)
    minBtn.Size = UDim2.new(0,32,0,32)
    minBtn.Position = UDim2.new(1,-74,0,6)
    minBtn.Text = "-"
    minBtn.Font = Enum.Font.GothamBlack
    minBtn.TextSize = 26
    minBtn.BackgroundColor3 = Color3.fromRGB(30,30,30)
    minBtn.TextColor3 = Color3.fromRGB(220,220,220)
    minBtn.BorderSizePixel = 0
    minBtn.MouseButton1Click:Connect(function()
        frame.Visible = false
        showNotification("Painel minimizado! Pressione CTRL para abrir.", "info")
    end)

    local closeBtn = Instance.new("TextButton", frame)
    closeBtn.Size = UDim2.new(0,32,0,32)
    closeBtn.Position = UDim2.new(1,-38,0,6)
    closeBtn.Text = "X"
    closeBtn.Font = Enum.Font.GothamBlack
    closeBtn.TextSize = 20
    closeBtn.BackgroundColor3 = Color3.fromRGB(60,30,30)
    closeBtn.TextColor3 = Color3.fromRGB(255,100,100)
    closeBtn.BorderSizePixel = 0
    closeBtn.MouseButton1Click:Connect(function()
        createConfirmPanel(frame, "Tem certeza que quer fechar o HUB?", function()
            gui:Destroy()
            showNotification("Painel fechado! Execute o script novamente para abrir.", "info")
        end)
    end)

    local title = Instance.new("TextLabel", frame)
    title.Size = UDim2.new(1,0,0.1,0)
    title.Position = UDim2.new(0,0,0,0)
    title.BackgroundTransparency = 1
    title.Font = Enum.Font.GothamBlack
    title.TextSize = 32
    title.Text = "Vitin Hub BETA"
    title.TextColor3 = Color3.fromRGB(200,200,200)

    -- Abas
    local panels = {}

    -- ESP Panel
    local espPanel = Instance.new("Frame", frame)
    espPanel.Size = UDim2.new(0.93,0,0.67,0)
    espPanel.Position = UDim2.new(0.03,0,0.20,0)
    espPanel.BackgroundTransparency = 1
    panels["ESP"] = espPanel

    local espRangeLbl = Instance.new("TextLabel", espPanel)
    espRangeLbl.Text = "Range do ESP:"
    espRangeLbl.Position = UDim2.new(0.05,0,0.05,0)
    espRangeLbl.Size = UDim2.new(0.4,0,0.07,0)
    espRangeLbl.BackgroundTransparency = 1
    espRangeLbl.TextColor3 = Color3.fromRGB(200,200,200)
    espRangeLbl.Font = Enum.Font.GothamSemibold
    espRangeLbl.TextSize = 15

    local espRangeBox = Instance.new("TextBox", espPanel)
    espRangeBox.Text = tostring(espRange)
    espRangeBox.Position = UDim2.new(0.45,0,0.05,0)
    espRangeBox.Size = UDim2.new(0.18,0,0.07,0)
    espRangeBox.BackgroundColor3 = Color3.fromRGB(25,25,25)
    espRangeBox.PlaceholderText = tostring(ESP_RANGE_MIN).." a "..tostring(ESP_RANGE_MAX)
    espRangeBox.TextColor3 = Color3.new(1,1,1)
    espRangeBox.Font = Enum.Font.GothamSemibold
    espRangeBox.TextSize = 15

    local espRangeConfirm = Instance.new("TextButton", espPanel)
    espRangeConfirm.Text = "OK"
    espRangeConfirm.Position = UDim2.new(0.64,0,0.05,0)
    espRangeConfirm.Size = UDim2.new(0.08,0,0.07,0)
    espRangeConfirm.BackgroundColor3 = Color3.fromRGB(40,40,40)
    espRangeConfirm.TextColor3 = Color3.new(1,1,1)
    espRangeConfirm.Font = Enum.Font.GothamBold
    espRangeConfirm.TextSize = 15
    espRangeConfirm.MouseButton1Click:Connect(function()
        local val = tonumber(espRangeBox.Text)
        if val and val >= ESP_RANGE_MIN and val <= ESP_RANGE_MAX then
            espRange = val
            showNotification("Range do ESP ajustado!", "success")
        else
            showNotification("Valor inválido! ("..ESP_RANGE_MIN.." a "..ESP_RANGE_MAX..")", "error")
            espRangeBox.Text = tostring(espRange)
        end
    end)

    local espToggle = Instance.new("TextButton", espPanel)
    espToggle.Size = UDim2.new(0.7,0,0.09,0)
    espToggle.Position = UDim2.new(0.05,0,0.18,0)
    espToggle.BackgroundColor3 = Color3.fromRGB(30,30,30)
    espToggle.Text = "ESP: OFF"
    espToggle.Font = Enum.Font.GothamBold
    espToggle.TextSize = 20
    espToggle.TextColor3 = Color3.new(1,1,1)
    espToggle.MouseButton1Click:Connect(function()
        espEnabled = not espEnabled
        espToggle.Text = espEnabled and "ESP: ON" or "ESP: OFF"
        showNotification(espEnabled and "ESP Ativado!" or "ESP Desativado!", espEnabled and "success" or "info")
    end)

    -- Aimbot Panel
    local aimbotPanel = Instance.new("Frame", frame)
    aimbotPanel.Size = UDim2.new(0.93,0,0.67,0)
    aimbotPanel.Position = UDim2.new(0.03,0,0.20,0)
    aimbotPanel.BackgroundTransparency = 1
    panels["Aimbot"] = aimbotPanel

    -- AimFOV config!
    local aimbotFOVLbl = Instance.new("TextLabel", aimbotPanel)
    aimbotFOVLbl.Text = "AimFOV (raio):"
    aimbotFOVLbl.Position = UDim2.new(0.05,0,0.13,0)
    aimbotFOVLbl.Size = UDim2.new(0.4,0,0.07,0)
    aimbotFOVLbl.BackgroundTransparency = 1
    aimbotFOVLbl.TextColor3 = Color3.fromRGB(200,200,200)
    aimbotFOVLbl.Font = Enum.Font.GothamSemibold
    aimbotFOVLbl.TextSize = 15

    local aimbotFOVBox = Instance.new("TextBox", aimbotPanel)
    aimbotFOVBox.Text = tostring(aimbotFOV)
    aimbotFOVBox.Position = UDim2.new(0.45,0,0.13,0)
    aimbotFOVBox.Size = UDim2.new(0.18,0,0.07,0)
    aimbotFOVBox.BackgroundColor3 = Color3.fromRGB(25,25,25)
    aimbotFOVBox.PlaceholderText = tostring(AIMBOT_FOV_MIN).." a "..tostring(AIMBOT_FOV_MAX)
    aimbotFOVBox.TextColor3 = Color3.new(1,1,1)
    aimbotFOVBox.Font = Enum.Font.GothamSemibold
    aimbotFOVBox.TextSize = 15

    local aimbotFOVConfirm = Instance.new("TextButton", aimbotPanel)
    aimbotFOVConfirm.Text = "OK"
    aimbotFOVConfirm.Position = UDim2.new(0.64,0,0.13,0)
    aimbotFOVConfirm.Size = UDim2.new(0.08,0,0.07,0)
    aimbotFOVConfirm.BackgroundColor3 = Color3.fromRGB(40,40,40)
    aimbotFOVConfirm.TextColor3 = Color3.new(1,1,1)
    aimbotFOVConfirm.Font = Enum.Font.GothamBold
    aimbotFOVConfirm.TextSize = 15
    aimbotFOVConfirm.MouseButton1Click:Connect(function()
        local val = tonumber(aimbotFOVBox.Text)
        if val and val >= AIMBOT_FOV_MIN and val <= AIMBOT_FOV_MAX then
            aimbotFOV = val
            showNotification("AimFOV ajustado!", "success")
        else
            showNotification("Valor inválido! ("..AIMBOT_FOV_MIN.." a "..AIMBOT_FOV_MAX..")", "error")
            aimbotFOVBox.Text = tostring(aimbotFOV)
        end
    end)

    -- Desenha o círculo do FOV na tela
    if Drawing then
        aimFOVCircle = Drawing.new("Circle")
        aimFOVCircle.Visible = true
        aimFOVCircle.Transparency = 0.6
        aimFOVCircle.Thickness = 2
        aimFOVCircle.Filled = false
        aimFOVCircle.Color = Color3.fromRGB(60,120,220)
    end

    local aimbotSmoothLbl = Instance.new("TextLabel", aimbotPanel)
    aimbotSmoothLbl.Text = "Suavidade do Aimbot:"
    aimbotSmoothLbl.Position = UDim2.new(0.05,0,0.05,0)
    aimbotSmoothLbl.Size = UDim2.new(0.4,0,0.07,0)
    aimbotSmoothLbl.BackgroundTransparency = 1
    aimbotSmoothLbl.TextColor3 = Color3.fromRGB(200,200,200)
    aimbotSmoothLbl.Font = Enum.Font.GothamSemibold
    aimbotSmoothLbl.TextSize = 15

    local aimbotSmoothBox = Instance.new("TextBox", aimbotPanel)
    aimbotSmoothBox.Text = tostring(aimbotSmooth)
    aimbotSmoothBox.Position = UDim2.new(0.45,0,0.05,0)
    aimbotSmoothBox.Size = UDim2.new(0.18,0,0.07,0)
    aimbotSmoothBox.BackgroundColor3 = Color3.fromRGB(25,25,25)
    aimbotSmoothBox.PlaceholderText = tostring(AIMBOT_SMOOTH_MIN).." a "..tostring(AIMBOT_SMOOTH_MAX)
    aimbotSmoothBox.TextColor3 = Color3.new(1,1,1)
    aimbotSmoothBox.Font = Enum.Font.GothamSemibold
    aimbotSmoothBox.TextSize = 15

    local aimbotSmoothConfirm = Instance.new("TextButton", aimbotPanel)
    aimbotSmoothConfirm.Text = "OK"
    aimbotSmoothConfirm.Position = UDim2.new(0.64,0,0.05,0)
    aimbotSmoothConfirm.Size = UDim2.new(0.08,0,0.07,0)
    aimbotSmoothConfirm.BackgroundColor3 = Color3.fromRGB(40,40,40)
    aimbotSmoothConfirm.TextColor3 = Color3.new(1,1,1)
    aimbotSmoothConfirm.Font = Enum.Font.GothamBold
    aimbotSmoothConfirm.TextSize = 15
    aimbotSmoothConfirm.MouseButton1Click:Connect(function()
        local val = tonumber(aimbotSmoothBox.Text)
        if val and val >= AIMBOT_SMOOTH_MIN and val <= AIMBOT_SMOOTH_MAX then
            aimbotSmooth = val
            showNotification("Suavidade do Aimbot ajustada!", "success")
        else
            showNotification("Valor inválido! ("..AIMBOT_SMOOTH_MIN.." a "..AIMBOT_SMOOTH_MAX..")", "error")
            aimbotSmoothBox.Text = tostring(aimbotSmooth)
        end
    end)

    local aimbotToggle = Instance.new("TextButton", aimbotPanel)
    aimbotToggle.Size = UDim2.new(0.7,0,0.09,0)
    aimbotToggle.Position = UDim2.new(0.05,0,0.18,0)
    aimbotToggle.BackgroundColor3 = Color3.fromRGB(30,30,30)
    aimbotToggle.Text = "Aimbot: OFF"
    aimbotToggle.Font = Enum.Font.GothamBold
    aimbotToggle.TextSize = 20
    aimbotToggle.TextColor3 = Color3.new(1,1,1)
    aimbotToggle.MouseButton1Click:Connect(function()
        aimbotEnabled = not aimbotEnabled
        aimbotToggle.Text = aimbotEnabled and "Aimbot: ON" or "Aimbot: OFF"
        showNotification(aimbotEnabled and "Aimbot ON!" or "Aimbot OFF", aimbotEnabled and "success" or "info")
    end)

    local aimbotHotkeyBtn = Instance.new("TextButton", aimbotPanel)
    aimbotHotkeyBtn.Size = UDim2.new(0.23,0,0.07,0)
    aimbotHotkeyBtn.Position = UDim2.new(0.74,0,0.18,0)
    aimbotHotkeyBtn.BackgroundColor3 = Color3.fromRGB(40,40,40)
    aimbotHotkeyBtn.Text = "Hotkey: ["..getInputString(currentAimbotHotkey,currentAimbotMouseButton).."]"
    aimbotHotkeyBtn.Font = Enum.Font.GothamBold
    aimbotHotkeyBtn.TextSize = 14
    aimbotHotkeyBtn.TextColor3 = Color3.new(1,1,1)
    aimbotHotkeyBtn.MouseButton1Click:Connect(function()
        showNotification("Pressione uma tecla ou mouse para Aimbot...", "info")
        local conn; conn = UserInputService.InputBegan:Connect(function(input,_)
            if input.UserInputType == Enum.UserInputType.Keyboard then
                currentAimbotHotkey = input.KeyCode
                currentAimbotMouseButton = nil
            elseif input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.MouseButton2 or input.UserInputType == Enum.UserInputType.MouseButton3 then
                currentAimbotMouseButton = input.UserInputType
                currentAimbotHotkey = nil
            end
            aimbotHotkeyBtn.Text = "Hotkey: ["..getInputString(currentAimbotHotkey,currentAimbotMouseButton).."]"
            showNotification("Hotkey do Aimbot trocada!", "success")
            conn:Disconnect()
        end)
    end)

    -- Outros Panel
    local outrosPanel = Instance.new("Frame", frame)
    outrosPanel.Size = UDim2.new(0.93,0,0.67,0)
    outrosPanel.Position = UDim2.new(0.03,0,0.20,0)
    outrosPanel.BackgroundTransparency = 1
    panels["Outros"] = outrosPanel

    -- Fly Infinity Yield
    local flySpeedLbl = Instance.new("TextLabel", outrosPanel)
    flySpeedLbl.Text = "Velocidade do Fly (Infinity Yield):"
    flySpeedLbl.Position = UDim2.new(0.05,0,0.05,0)
    flySpeedLbl.Size = UDim2.new(0.4,0,0.07,0)
    flySpeedLbl.BackgroundTransparency = 1
    flySpeedLbl.TextColor3 = Color3.fromRGB(200,200,200)
    flySpeedLbl.Font = Enum.Font.GothamSemibold
    flySpeedLbl.TextSize = 15

    local flySpeedBox = Instance.new("TextBox", outrosPanel)
    flySpeedBox.Text = tostring(flySpeed)
    flySpeedBox.Position = UDim2.new(0.45,0,0.05,0)
    flySpeedBox.Size = UDim2.new(0.18,0,0.07,0)
    flySpeedBox.BackgroundColor3 = Color3.fromRGB(25,25,25)
    flySpeedBox.PlaceholderText = tostring(FLY_SPEED_MIN).." a "..tostring(FLY_SPEED_MAX)
    flySpeedBox.TextColor3 = Color3.new(1,1,1)
    flySpeedBox.Font = Enum.Font.GothamSemibold
    flySpeedBox.TextSize = 15

    local flySpeedConfirm = Instance.new("TextButton", outrosPanel)
    flySpeedConfirm.Text = "OK"
    flySpeedConfirm.Position = UDim2.new(0.64,0,0.05,0)
    flySpeedConfirm.Size = UDim2.new(0.08,0,0.07,0)
    flySpeedConfirm.BackgroundColor3 = Color3.fromRGB(40,40,40)
    flySpeedConfirm.TextColor3 = Color3.new(1,1,1)
    flySpeedConfirm.Font = Enum.Font.GothamBold
    flySpeedConfirm.TextSize = 15
    flySpeedConfirm.MouseButton1Click:Connect(function()
        local val = tonumber(flySpeedBox.Text)
        if val and val >= FLY_SPEED_MIN and val <= FLY_SPEED_MAX then
            flySpeed = val
            showNotification("Velocidade do Fly ajustada!", "success")
        else
            showNotification("Valor inválido! ("..FLY_SPEED_MIN.." a "..FLY_SPEED_MAX..")", "error")
            flySpeedBox.Text = tostring(flySpeed)
        end
    end)

    local flyToggle = Instance.new("TextButton", outrosPanel)
    flyToggle.Size = UDim2.new(0.7,0,0.09,0)
    flyToggle.Position = UDim2.new(0.05,0,0.18,0)
    flyToggle.BackgroundColor3 = Color3.fromRGB(30,30,30)
    flyToggle.Text = "Fly: OFF"
    flyToggle.Font = Enum.Font.GothamBold
    flyToggle.TextSize = 20
    flyToggle.TextColor3 = Color3.new(1,1,1)
    flyToggle.MouseButton1Click:Connect(function()
        flyEnabled = not flyEnabled
        flyToggle.Text = flyEnabled and "Fly: ON" or "Fly: OFF"
        if flyEnabled then startFlyInfinityYield() else stopFlyInfinityYield() end
        showNotification(flyEnabled and "Fly Ativado!" or "Fly Desativado!", flyEnabled and "success" or "info")
    end)

    local flyHotkeyBtn = Instance.new("TextButton", outrosPanel)
    flyHotkeyBtn.Size = UDim2.new(0.23,0,0.09,0)
    flyHotkeyBtn.Position = UDim2.new(0.74,0,0.18,0)
    flyHotkeyBtn.BackgroundColor3 = Color3.fromRGB(40,40,40)
    flyHotkeyBtn.Text = "Hotkey: ["..getInputString(currentFlyHotkey,currentFlyMouseButton).."]"
    flyHotkeyBtn.Font = Enum.Font.GothamBold
    flyHotkeyBtn.TextSize = 14
    flyHotkeyBtn.TextColor3 = Color3.new(1,1,1)
    flyHotkeyBtn.MouseButton1Click:Connect(function()
        showNotification("Pressione uma tecla ou mouse para Fly...", "info")
        local conn; conn = UserInputService.InputBegan:Connect(function(input,_)
            if input.UserInputType == Enum.UserInputType.Keyboard then
                currentFlyHotkey = input.KeyCode
                currentFlyMouseButton = nil
            elseif input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.MouseButton2 or input.UserInputType == Enum.UserInputType.MouseButton3 then
                currentFlyMouseButton = input.UserInputType
                currentFlyHotkey = nil
            end
            flyHotkeyBtn.Text = "Hotkey: ["..getInputString(currentFlyHotkey,currentFlyMouseButton).."]"
            showNotification("Hotkey do Fly trocada!", "success")
            conn:Disconnect()
        end)
    end)

    local tpPanel = createTeleportPanel(outrosPanel)
    tpPanel.Position = UDim2.new(0.03,0,0.3,0)
    tpPanel.Size = UDim2.new(0.94,0,0.5,0)

    -- TabBar
    local tabNames = {"Aimbot","ESP","Outros"}
    local tabBar = Instance.new("Frame", frame)
    tabBar.Size = UDim2.new(1,0,0,40)
    tabBar.Position = UDim2.new(0,0,0.1,0)
    tabBar.BackgroundTransparency = 1
    tabBar.Name = "TabBar"
    local tabBtns = {}
    for i,tab in ipairs(tabNames) do
        local btn = Instance.new("TextButton", tabBar)
        btn.Size = UDim2.new(0.18,0,1,0)
        btn.Position = UDim2.new(0.01 + (i-1)*0.2,0,0,0)
        btn.BackgroundColor3 = Color3.fromRGB(30,30,30)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 18
        btn.TextColor3 = Color3.fromRGB(220,220,220)
        btn.Text = tab
        btn.BorderSizePixel = 0
        btn.Name = tab.."Btn"
        btn.MouseButton1Click:Connect(function()
            for _,p in pairs(panels) do p.Visible = false end
            panels[tab].Visible = true
            for _,b in pairs(tabBtns) do b.BackgroundColor3 = Color3.fromRGB(30,30,30) end
            btn.BackgroundColor3 = Color3.fromRGB(60,120,220)
        end)
        tabBtns[tab] = btn
    end
    tabBtns["Aimbot"].BackgroundColor3 = Color3.fromRGB(60,120,220)
    for _,p in pairs(panels) do p.Visible = false end panels["Aimbot"].Visible = true
end

-- Minimizar com CTRL
UserInputService.InputBegan:Connect(function(input, processed)
    if (input.KeyCode == Enum.KeyCode.LeftControl or input.KeyCode == Enum.KeyCode.RightControl) then
        if hubFrame and hubFrame.Parent and not hubFrame.Visible then
            hubFrame.Visible = true
        end
    end
end)

-- Fly Infinity Yield system
local keys = {w=false, a=false, s=false, d=false}
local flyConn, bodyGyro, bodyVel

function startFlyInfinityYield()
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    local hrp = character.HumanoidRootPart

    bodyGyro = Instance.new("BodyGyro")
    bodyGyro.P = 9e4
    bodyGyro.MaxTorque = Vector3.new(9e9,9e9,9e9)
    bodyGyro.CFrame = hrp.CFrame
    bodyGyro.Parent = hrp

    bodyVel = Instance.new("BodyVelocity")
    bodyVel.Velocity = Vector3.new(0,0,0)
    bodyVel.MaxForce = Vector3.new(9e9,9e9,9e9)
    bodyVel.Parent = hrp

    if character:FindFirstChildOfClass("Humanoid") then
        character:FindFirstChildOfClass("Humanoid").PlatformStand = true
    end

    flyConn = RunService.RenderStepped:Connect(function()
        if not flyEnabled or not hrp or not hrp.Parent then return end
        local cam = workspace.CurrentCamera
        bodyGyro.CFrame = cam.CFrame
        local move = Vector3.new()
        if keys.w then move = move + cam.CFrame.LookVector end
        if keys.s then move = move - cam.CFrame.LookVector end
        if keys.a then move = move - cam.CFrame.RightVector end
        if keys.d then move = move + cam.CFrame.RightVector end
        bodyVel.Velocity = move * math.clamp(flySpeed, FLY_SPEED_MIN, FLY_SPEED_MAX)
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
            bodyVel.Velocity = bodyVel.Velocity + Vector3.new(0,math.clamp(flySpeed, FLY_SPEED_MIN, FLY_SPEED_MAX),0)
        end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
            bodyVel.Velocity = bodyVel.Velocity - Vector3.new(0,math.clamp(flySpeed, FLY_SPEED_MIN, FLY_SPEED_MAX),0)
        end
    end)
end

function stopFlyInfinityYield()
    if flyConn then flyConn:Disconnect() end
    local character = LocalPlayer.Character
    if character and character:FindFirstChild("HumanoidRootPart") then
        local hrp = character.HumanoidRootPart
        if hrp:FindFirstChildOfClass("BodyGyro") then hrp:FindFirstChildOfClass("BodyGyro"):Destroy() end
        if hrp:FindFirstChildOfClass("BodyVelocity") then hrp:FindFirstChildOfClass("BodyVelocity"):Destroy() end
        if character:FindFirstChildOfClass("Humanoid") then
            character:FindFirstChildOfClass("Humanoid").PlatformStand = false
        end
    end
end

UserInputService.InputBegan:Connect(function(input, gp)
    if not flyEnabled then return end
    if input.KeyCode == Enum.KeyCode.W then keys.w = true end
    if input.KeyCode == Enum.KeyCode.A then keys.a = true end
    if input.KeyCode == Enum.KeyCode.S then keys.s = true end
    if input.KeyCode == Enum.KeyCode.D then keys.d = true end
end)

UserInputService.InputEnded:Connect(function(input, gp)
    if not flyEnabled then return end
    if input.KeyCode == Enum.KeyCode.W then keys.w = false end
    if input.KeyCode == Enum.KeyCode.A then keys.a = false end
    if input.KeyCode == Enum.KeyCode.S then keys.s = false end
    if input.KeyCode == Enum.KeyCode.D then keys.d = false end
end)

-- Hotkeys para Fly (hub/painel/tecla)
UserInputService.InputBegan:Connect(function(input)
    if (currentFlyHotkey and input.KeyCode == currentFlyHotkey) or (currentFlyMouseButton and input.UserInputType == currentFlyMouseButton) then
        flyEnabled = not flyEnabled
        if flyEnabled then startFlyInfinityYield() else stopFlyInfinityYield() end
        if hubFrame then
            local minimapanel = hubFrame:FindFirstChildOfClass("Frame")
            local flyPanel = minimapanel and (minimapanel:FindFirstChild("Fly: OFF") or minimapanel:FindFirstChild("Fly: ON"))
            if flyPanel then flyPanel.Text = flyEnabled and "Fly: ON" or "Fly: OFF" end
        end
        showNotification(flyEnabled and "Fly Ativado!" or "Fly Desativado!", flyEnabled and "success" or "info")
    end
end)

-- Aimbot com AimFOV!
local function getClosestEnemyFOV()
    local closest, dist = nil, math.huge
    local mousePos = UserInputService:GetMouseLocation()
    for _,plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("Head") then
            local humanoid = plr.Character:FindFirstChild("Humanoid")
            if humanoid and humanoid.Health > 0 then
                local head = plr.Character.Head
                local screenPos = workspace.CurrentCamera:WorldToViewportPoint(head.Position)
                local pos2D = Vector2.new(screenPos.X, screenPos.Y)
                local d = (pos2D - mousePos).Magnitude
                if d < dist and d <= aimbotFOV then -- Só pega se dentro do FOV!
                    closest = head
                    dist = d
                end
            end
        end
    end
    return closest
end

local aimbotActive = false
UserInputService.InputBegan:Connect(function(input,_)
    if aimbotEnabled and ((currentAimbotMouseButton and input.UserInputType == currentAimbotMouseButton) or (not currentAimbotMouseButton and input.UserInputType == Enum.UserInputType.MouseButton2)) then
        aimbotActive = true
    end
end)
UserInputService.InputEnded:Connect(function(input,_)
    if input.UserInputType == Enum.UserInputType.MouseButton2 then aimbotActive = false end
    if currentAimbotMouseButton and input.UserInputType == currentAimbotMouseButton then aimbotActive = false end
end)

RunService.RenderStepped:Connect(function()
    -- Desenha AimFOV
    if aimFOVCircle then
        local mousePos = UserInputService:GetMouseLocation()
        aimFOVCircle.Visible = panels["Aimbot"] and panels["Aimbot"].Visible
        aimFOVCircle.Position = mousePos
        aimFOVCircle.Radius = math.clamp(aimbotFOV, AIMBOT_FOV_MIN, AIMBOT_FOV_MAX)
    end
    -- Aimbot com AimFOV
    if aimbotEnabled and aimbotActive and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local target = getClosestEnemyFOV()
        if target then
            local cam = workspace.CurrentCamera
            local goal = CFrame.new(cam.CFrame.Position, target.Position)
            cam.CFrame = cam.CFrame:Lerp(goal,math.clamp(aimbotSmooth,AIMBOT_SMOOTH_MIN,AIMBOT_SMOOTH_MAX))
        end
    end
end)

-- ESP (Drawing API)
local espDrawings = {}
RunService.RenderStepped:Connect(function()
    for _,drawing in pairs(espDrawings) do drawing.Visible = false end
    if not espEnabled then return end
    if not Drawing then return end
    for _,plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("Head") then
            local head = plr.Character.Head
            local pos,onscreen = workspace.CurrentCamera:WorldToViewportPoint(head.Position)
            local dist = (LocalPlayer.Character.HumanoidRootPart.Position - head.Position).Magnitude
            if onscreen and dist < math.clamp(espRange,ESP_RANGE_MIN,ESP_RANGE_MAX) then
                if not espDrawings[plr] then
                    local box = Drawing.new("Square")
                    box.Thickness = 2
                    box.Filled = false
                    box.Color = Color3.fromRGB(255,255,255)
                    espDrawings[plr] = box
                end
                local box = espDrawings[plr]
                box.Visible = true
                box.Size = Vector2.new(60,80)
                box.Position = Vector2.new(pos.X-30,pos.Y-40)
                if not espDrawings[plr.."name"] then
                    local txt = Drawing.new("Text")
                    txt.Size = 20
                    txt.Color = Color3.fromRGB(220,220,220)
                    txt.Center = true
                    txt.Outline = true
                    espDrawings[plr.."name"] = txt
                end
                local txt = espDrawings[plr.."name"]
                txt.Visible = true
                txt.Text = plr.Name
                txt.Position = Vector2.new(pos.X,pos.Y-55)
                if not espDrawings[plr.."life"] then
                    local bar = Drawing.new("Square")
                    bar.Filled = true
                    bar.Color = Color3.fromRGB(60,200,60)
                    espDrawings[plr.."life"] = bar
                end
                local bar = espDrawings[plr.."life"]
                bar.Visible = true
                local hp = plr.Character:FindFirstChild("Humanoid") and plr.Character.Humanoid.Health or 0
                bar.Size = Vector2.new(60*(math.clamp(hp,0,100)/100),5)
                bar.Position = Vector2.new(pos.X-30,pos.Y+40)
            end
        end
    end
end)

createKeyScreen()
