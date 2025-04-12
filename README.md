 Troca automática para o time "Marines" antes de iniciar
local debounceTime = 1
local canChangeTeam = true

if canChangeTeam then
    canChangeTeam = false
    task.wait(debounceTime)
    pcall(function()
        game:GetService("ReplicatedStorage").Remotes.CommF_:InvokeServer("SetTeam", "Marines")
    end)
    canChangeTeam = true
end

repeat task.wait() until game.Players.LocalPlayer.Team and game.Players.LocalPlayer.Team.Name == "Marines"

-- Resto do script começa aqui:
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local hrp = character:WaitForChild("HumanoidRootPart")

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "DomiusFruitFinder"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = player:WaitForChild("PlayerGui")

local Title = Instance.new("TextLabel")
Title.Name = "Title"
Title.Parent = ScreenGui
Title.AnchorPoint = Vector2.new(0.5, 0.5)
Title.Position = UDim2.new(0.5, 0, 0.18, 0)
Title.Size = UDim2.new(0, 500, 0, 50)
Title.Text = "Domius BF ( Find Fruit )"
Title.TextColor3 = Color3.fromRGB(255, 215, 100)
Title.BackgroundTransparency = 1
Title.TextScaled = true
Title.Font = Enum.Font.GothamBold

local Timer = Instance.new("TextLabel")
Timer.Name = "Timer"
Timer.Parent = ScreenGui
Timer.AnchorPoint = Vector2.new(0.5, 0.5)
Timer.Position = UDim2.new(0.5, 0, 0.30, 0)
Timer.Size = UDim2.new(0, 150, 0, 30)
Timer.Text = "Time: 0h0m0s"
Timer.TextColor3 = Color3.fromRGB(255, 255, 255)
Timer.BackgroundTransparency = 1
Timer.TextScaled = true
Timer.Font = Enum.Font.Gotham

local Status = Instance.new("TextLabel")
Status.Name = "Status"
Status.Parent = ScreenGui
Status.AnchorPoint = Vector2.new(0.5, 0.5)
Status.Position = UDim2.new(0.5, 0, 0.95, 0)
Status.Size = UDim2.new(0, 400, 0, 35)
Status.Text = "Status: Searching..."
Status.TextColor3 = Color3.fromRGB(255, 255, 255)
Status.BackgroundTransparency = 1
Status.TextScaled = true
Status.Font = Enum.Font.GothamBold

-- Atualiza tempo
local start = tick()
spawn(function()
    while true do
        task.wait(1)
        local t = math.floor(tick() - start)
        local h = string.format("%01d", math.floor(t / 3600))
        local m = string.format("%01d", math.floor((t % 3600) / 60))
        local s = string.format("%01d", t % 60)
        Timer.Text = "Time: " .. h .. "h" .. m .. "m" .. s .. "s"
    end
end)

-- Desativa colisão
local function disableCollision()
    for _, part in ipairs(character:GetDescendants()) do
        if part:IsA("BasePart") then
            part.CanCollide = false
        end
    end
end

-- Voa até fruta
local currentConnection
local function flyTo(position, speed)
    speed = speed or 300
    if currentConnection then currentConnection:Disconnect() end

    currentConnection = RunService.RenderStepped:Connect(function()
        disableCollision()
        if character and hrp and position then
            local direction = (position - hrp.Position).Unit
            hrp.Velocity = direction * speed
            if (hrp.Position - position).Magnitude < 5 then
                hrp.Velocity = Vector3.zero
                Status.Text = "Status: Fruta coletada!"
                currentConnection:Disconnect()
            end
        end
    end)
end

-- Acha fruta
local function findFruit()
    local areas = {Workspace, Workspace:FindFirstChild("_WorldOrigin")}
    for _, area in ipairs(areas) do
        if area then
            for _, v in ipairs(area:GetChildren()) do
                if v:IsA("Model") and string.find(v.Name, "Fruit") and v:FindFirstChild("Handle") then
                    return v.Handle
                end
            end
        end
    end
    return nil
end

-- Sistema de troca de servidor após 10 segundos sem encontrar fruta
local function serverHop()
    local placeId = game.PlaceId
    local cursor = ""
    local tried = {}
    local function getServers()
        local url = "https://games.roblox.com/v1/games/" .. placeId .. "/servers/Public?sortOrder=Asc&limit=100"
        if cursor ~= "" then url = url .. "&cursor=" .. cursor end
        local response = HttpService:JSONDecode(game:HttpGet(url))
        cursor = response.nextPageCursor or ""
        return response.data
    end

    for _, server in pairs(getServers()) do
        if server.playing < server.maxPlayers and not tried[server.id] then
            tried[server.id] = true
            TeleportService:TeleportToPlaceInstance(placeId, server.id, player)
            break
        end
    end
end

-- Loop principal
spawn(function()
    while task.wait(0.5) do
        local fruit = findFruit()
        if fruit then
            Status.Text = "Status: Find Fruit (" .. fruit.Parent.Name .. ")"
            flyTo(fruit.Position)
        else
            for i = 10, 1, -1 do
                Status.Text = "Status: Hop server in " .. i .. "s"
                task.wait(1)
            end
            serverHop()
        end
    end
end)
