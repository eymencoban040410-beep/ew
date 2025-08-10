local ReplicatedStorage = game:GetService("ReplicatedStorage") 
local SoundService = game:GetService("SoundService")
local Terrain = workspace:FindFirstChild("Terrain")
local TweenService = game:GetService("TweenService")

local ZombieWaves = require(ReplicatedStorage.Modules:WaitForChild("Wave"))
local ZombieFolder = ReplicatedStorage:WaitForChild("Zombies")
local DayValue = ReplicatedStorage:WaitForChild("Days")
local DayNightValue = ReplicatedStorage:WaitForChild("DayNight")
local BloodMoonValue = ReplicatedStorage:WaitForChild("BloodMoon")
local RadiationNightValue = ReplicatedStorage:WaitForChild("RadiationNight")
local AliveZombiesValue = ReplicatedStorage:WaitForChild("AliveZombies")
local CurrentMapValue = ReplicatedStorage:WaitForChild("CurrentMap")
local WeatherType = ReplicatedStorage:WaitForChild("WeatherType")
local TemperatureType = ReplicatedStorage:WaitForChild("TemperatureType")

local Remotes = ReplicatedStorage:WaitForChild("Remotes")
local BossDiedEvent = Remotes:WaitForChild("BossDiedEvent")
local GoodEndingRemote = Remotes:WaitForChild("GoodEndingRemote")
local StartMusic = SoundService.Musics:WaitForChild("StartMusic")
local DayMusic = SoundService.Musics:WaitForChild("DayMusic")
local UpdateWeatherUIEvent = Remotes:WaitForChild("UpdateWeatherGUI")

local currentDayMusicTask = nil
local MapFolder = nil
local SpawnPoints = {}
local Generator


local WeatherPerDay = {}

local WeatherTypes = {
	{ Name = "Radioactive", Rarity = 0.5 },
	{ Name = "Blood Moon", Rarity = 1 },
	{ Name = "Full Moon", Rarity = 3 },
	{ Name = "Rain", Rarity = 4 },
	{ Name = "Normal", Rarity = 10 }
}

local TemperatureTypes = {
	{ Name = "Hot", Chance = 25 },
	{ Name = "Cold", Chance = 25 },
	{ Name = "Normal", Chance = 50 }
}



local function SetCloudCover(target)
	local clouds = Terrain:FindFirstChildOfClass("Clouds")
	if clouds then
		local tween = TweenService:Create(
			clouds,
			TweenInfo.new(5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
			{Cover = target}
		)
		tween:Play()
	end
end

local function UpdateRainEffects()
	local currentMapName = ReplicatedStorage:WaitForChild("CurrentMap").Value
	local mapObjectValue = script:FindFirstChild(currentMapName .. "Map")
	local mapFolder = mapObjectValue and mapObjectValue.Value
	if not mapFolder then return end

	local rainPartFolder = mapFolder:FindFirstChild("RainPartFolder")
	if not rainPartFolder then return end

	for _, rainPart in ipairs(rainPartFolder:GetChildren()) do
		if rainPart:IsA("BasePart") then
			local rainParticle = rainPart:FindFirstChild("RainParticle")
			if rainParticle and rainParticle:IsA("ParticleEmitter") then
				rainParticle.Enabled = (WeatherType.Value == "Rain")
			end
	end

	if WeatherType.Value == "Rain" then
		SetCloudCover(0.81)
	else
		SetCloudCover(0)
		end
	end
end




local function GetRandomWeather(currentDay)
	local filteredWeatherTypes = {}

	-- Eğer gün 10 veya altı ise, Radiation ve Blood Moon çıkar
	for _, w in ipairs(WeatherTypes) do
		if currentDay > 10 or (w.Name ~= "Radioactive" and w.Name ~= "Blood Moon") then
			table.insert(filteredWeatherTypes, w)
		end
	end

	local totalRarity = 0
	for _, w in ipairs(filteredWeatherTypes) do
		totalRarity += w.Rarity
	end

	local pick = math.random(1, totalRarity)
	local sum = 0

	for _, w in ipairs(filteredWeatherTypes) do
		sum += w.Rarity
		if pick <= sum then
			return w.Name
		end
	end
	return "Normal"
end


local function GetRandomTemperature()
	local pick = math.random(1, 100)
	local sum = 0

	for _, t in ipairs(TemperatureTypes) do
		sum += t.Chance
		if pick <= sum then
			return t.Name
		end
	end
	return "Normal"
end



local function GenerateRandomTemperatureValue(tempType)
	if tempType == "Cold" then
		return math.random() * 10 -- 0 ile 10 arası
	elseif tempType == "Normal" then
		return 11 + math.random() * 8 -- 11 ile 19 arası
	elseif tempType == "Hot" then
		return 20 + math.random() * 10 -- 20 ile 30 arası
	end
	return 15 -- Varsayılan (Normal gibi)
end

local function GenerateWeatherForDays(totalDays)
	local result = {}
	for i = 1, totalDays do
		local tempType = GetRandomTemperature()
		local avgTemp = GenerateRandomTemperatureValue(tempType)
		local weather = GetRandomWeather(i)  -- Gün parametresi ile çağırıyoruz

		result[i] = {
			Weather = weather,
			Temperature = tempType,
			AverageTemp = math.floor(avgTemp * 10) / 10
		}
	end
	return result
end


local totalDays = ZombieWaves:GetLastWaveDay()
WeatherPerDay = GenerateWeatherForDays(totalDays)
UpdateWeatherUIEvent:FireAllClients(WeatherPerDay)


local function UpdateMapFolder()
	local mapName = CurrentMapValue.Value
	local objectValue = script:FindFirstChild(mapName .. "Map")
	if objectValue and objectValue:IsA("ObjectValue") and objectValue.Value then
		MapFolder = objectValue.Value
		Generator = objectValue.Value.Generator.OnOff
		print("MapFolder güncellendi:", MapFolder.Name)

		local spawnFolder = MapFolder:FindFirstChild("Wave") and MapFolder.Wave:FindFirstChild("Spawns")
		if spawnFolder then
			SpawnPoints = {}
			for _, part in pairs(spawnFolder:GetChildren()) do
				if part:IsA("BasePart") then
					table.insert(SpawnPoints, part)
				end
			end
		else
			warn("Spawns klasörü bulunamadı.")
		end
	else
		MapFolder = nil
		SpawnPoints = {}
		warn("Harita ObjectValue bulunamadı ya da geçersiz: " .. mapName)
	end
end

UpdateMapFolder()
CurrentMapValue.Changed:Connect(UpdateMapFolder)

local function GetRandomSpawnPoint()
	if #SpawnPoints == 0 then
		warn("Spawn point yok!")
		return nil
	end
	return SpawnPoints[math.random(1, #SpawnPoints)]
end

local function SpawnZombie(zombieName, temperature, buffPercent)
	buffPercent = buffPercent or 0

	local zombieModel = ZombieFolder:FindFirstChild(zombieName)
	if not zombieModel then
		warn("Zombi modeli bulunamadı: " .. zombieName)
		return
	end

	local spawnPoint = GetRandomSpawnPoint()
	if not spawnPoint then return end

	local clone = zombieModel:Clone()
	clone:PivotTo(CFrame.new(spawnPoint.Position))
	clone.Parent = workspace

	local humanoid = clone:FindFirstChildWhichIsA("Humanoid")
	if humanoid then
		-- 🌡️ Sıcaklık etkisi
		if temperature == "Hot" then
			humanoid.WalkSpeed *= 1.2
		elseif temperature == "Cold" then
			humanoid.WalkSpeed *= 0.8
		end

		-- 🌕 Full Moon buff
		if buffPercent > 0 then
			local healthBoost = humanoid.MaxHealth * (buffPercent / 100)
			humanoid.MaxHealth += healthBoost
			humanoid.Health = humanoid.MaxHealth

			local damageValue = clone:FindFirstChild("Damage")
			if damageValue and damageValue:IsA("IntValue") then
				damageValue.Value += damageValue.Value * (buffPercent / 100)
			end
		end

		-- ✅ Zombi başarıyla doğdu, sayacı artır
		AliveZombiesValue.Value += 1

		humanoid.Died:Connect(function()
			AliveZombiesValue.Value -= 1
			print("☠️ Zombi öldü. Kalan:", AliveZombiesValue.Value)
		end)
	else
		-- Humanoid yoksa yine de öldü kabul et
		warn("Zombi modelinde humanoid yok: " .. zombieName)
	end
end





local function SetDayNight(value)
	DayNightValue.Value = value
	print("Zaman değişti:", value)
end

local function shuffle(tbl)
	local n = #tbl
	for i = n, 2, -1 do
		local j = math.random(i)
		tbl[i], tbl[j] = tbl[j], tbl[i]
	end
end

local waveStarted = false
local waveSpawnCompleted = false
AliveZombiesValue.Value = 0

local function StartWave()
	if not MapFolder or #SpawnPoints == 0 then
		warn("Harita ya da spawn noktaları hazır değil.")
		return
	end

	local currentDay = DayValue.Value
	local weatherData = WeatherPerDay[currentDay]
	local weather = GetRandomWeather(currentDay)  -- Günü parametre olarak veriyoruz
	local temperature = weatherData.Temperature

	WeatherType.Value = weather
	TemperatureType.Value = temperature

	-- BloodMoon ve RadiationNight hava durumuna göre ayarlanır
	local isBloodMoon = (weather == "Blood Moon")
	local isRadiationNight = (weather == "Radioactive")

	BloodMoonValue.Value = isBloodMoon
	RadiationNightValue.Value = isRadiationNight

	print("🌤 Gün " .. currentDay .. ": Weather = " .. weather .. ", Temperature = " .. temperature)


	local wave
	if isBloodMoon then
		wave = ZombieWaves:GetBloodMoonWave()
	elseif isRadiationNight then
		wave = ZombieWaves:GetRadiationNightWave()
	else
		wave = ZombieWaves:GetWaveForDay(currentDay)
	end

	if not wave then
		warn("Wave verisi bulunamadı.")
		return false
	end


	WeatherType.Value = weather
	TemperatureType.Value = temperature

	local avgTemp = weatherData.AverageTemp
	print(string.format(
		"🌤 Gün %d: Weather = %s, Temperature = %s, Ortalama = %.1f°C",
		currentDay, weather, temperature, avgTemp
		))


	local buffPercent = 0
	if weather == "Full Moon" then
		buffPercent = math.random(5, 10)
		print("🌕 Zombilere %" .. buffPercent .. " güçlülük buff'ı uygulandı.")
	end


	-- Sıcaklık etkisi
	if temperature == "Hot" then
		print("🔥 Hava sıcak, zombiler daha hızlı olacak.")
	elseif temperature == "Cold" then
		print("❄️ Hava soğuk, zombiler yavaş.")
	end

	-- Spawn
	AliveZombiesValue.Value = 0
	waveSpawnCompleted = false
	waveStarted = true

	local spawnQueue = {}
	for zombieType, amount in pairs(wave) do
		for i = 1, amount do
			table.insert(spawnQueue, zombieType)
		end
	end

	if currentDay ~= 10 and math.random(1, 20) == 1 and not isBloodMoon and not isRadiationNight then
		table.insert(spawnQueue, "Golden")
		print("Golden zombi geldi!")
	end

	shuffle(spawnQueue)

	task.spawn(function()
		local index = 1
		local maxAliveZombies = math.clamp(1 + math.floor((currentDay - 1) * 0.8), 3, 6)

		while index <= #spawnQueue do
			if AliveZombiesValue.Value < maxAliveZombies then
				SpawnZombie(spawnQueue[index], temperature, buffPercent)
				index += 1
			end
			local spawnDelay = Generator and Generator.Value == false and 0.5 or 1

			if weather == "Rain" then
				spawnDelay = spawnDelay * 0.4  -- Yağmurda bekleme süresi %30 azalır
			end

			task.wait(spawnDelay)

		end
		waveSpawnCompleted = true
	end)

	repeat task.wait(1) until waveSpawnCompleted and AliveZombiesValue.Value <= 0

	task.wait(5)
	SetDayNight("Day")
	waveStarted = false
	DayValue.Value += 1

	BloodMoonValue.Value = false
	RadiationNightValue.Value = false

	return true
end

if DayValue.Value == 1 then
	if UpdateWeatherUIEvent then
		task.wait(1)
		UpdateWeatherUIEvent:FireAllClients(WeatherPerDay, DayValue.Value)
	end
end




DayValue.Changed:Connect(function()
	UpdateWeatherUIEvent:FireAllClients(WeatherPerDay, DayValue.Value)
	if DayValue.Value + 1 == ZombieWaves:GetLastWaveDay() then
		GoodEndingRemote:FireAllClients()
	end
end)

DayNightValue:GetPropertyChangedSignal("Value"):Connect(function()
	if DayNightValue.Value == "Night" and not waveStarted then
		task.delay(math.random(8, 10), function()
			if DayNightValue.Value == "Night" and not waveStarted then
				StartWave()
			end
		end)
	end
	if DayNightValue.Value == "Day" and DayValue.Value > 1 then
		local delayTime = math.random(10, 60)
		print("DayMusic " .. delayTime .. " saniye sonra çalacak (gün: " .. DayValue.Value .. ")")
		currentDayMusicTask = task.delay(delayTime, function()
			if DayNightValue.Value == "Day" then
				DayMusic:Play()
				print("DayMusic çalmaya başladı (gün: " .. DayValue.Value .. ")")
			end
		end)
	elseif DayNightValue.Value == "Night" then
		if DayMusic.IsPlaying then
			DayMusic:Stop()
			print("Gece oldu, DayMusic durduruldu.")
		end
	end
end)
WeatherType.Changed:Connect(UpdateRainEffects)
