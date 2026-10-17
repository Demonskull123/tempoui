-- ============================================================
--  TempoUI v2 - Tempo's own menu (2026-10-09)
--
--  Same API as v1 (2026-10-07). Loaded AFTER UILib, which keeps TeekUI.Draw
--  (ESP), Palette and Fonts; this file takes over TeekUI.new.
--
--  New in v2: seven style presets (Glass = default, Anime, Sleek, Gamer,
--  Manga, Arcade, Grimoire), per-game accent + header art, snappy motion, and
--  a Settings > Interface tab with every look option (saved in store.Look).
--
--  v2.1 (2026-10-12): five character presets (Madara, Mugetsu, Rengoku,
--  Itachi, Urahara) - each sits on a base style and adds its own palette,
--  type, shape and decorations (PNG emblems from the repo, see DECOS) - and a
--  Preset Builder page: every structural value of a style can be overridden,
--  saved as your own named preset (one file shared by every Tempo hub) and
--  shared as a theme code.
--
--  Layout: top bar (brand, game, art), a rail of groups on the left with the
--  search box, the group's tabs across the top, content in two columns of
--  section cards (one column when narrow).
--
--  Built lazily: every element call stores a spec and returns its handle at
--  once; a tab only creates GUI the first time it is opened. Specs are the
--  truth - building (or rebuilding after a preset switch) never re-runs a
--  callback.
--
--  Re-theming: every look-dependent property is registered in a paint list
--  (colour, transparency, font, corner radius). Accent / colours / opacity /
--  roundness / glow / font = one pass over that list. Preset and density
--  change structure, so they rebuild the chrome and drop built tabs back to
--  specs (values kept, nothing re-fires).
--
--  Draws above the Roblox pause menu (OnTopOfCoreBlur + top DisplayOrder).
-- ============================================================
do
local Players          = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local RunService       = game:GetService("RunService")
local HttpService      = game:GetService("HttpService")
local LocalPlayer      = Players.LocalPlayer
local IS_MOBILE        = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

local DEFAULT_MENU_KEY = "RightControl"   -- never K (shift lock in ABA)
local MIN_W, MIN_H = 600, 380
local LOGO_URL = "https://raw.githubusercontent.com/teekmill17-droid/TempoHub/main/assets/logo.png"
local ART_URL = "https://raw.githubusercontent.com/teekmill17-droid/TempoHub/main/assets/art/%s.png"
local CLICK_SOUND = "rbxasset://sounds/electronicpingshort.wav"

-- ------------------------------------------------------------ where GUI goes
-- gethui() when it actually renders (MacSploit's doesn't), else CoreGui, else
-- PlayerGui. getgenv().TempoUIParent = "CoreGui" / "PlayerGui" forces one.
local function guiRenders(inst)
	local ok, r = pcall(function()
		local cg = game:GetService("CoreGui")
		local pg = LocalPlayer and LocalPlayer:FindFirstChildOfClass("PlayerGui")
		return inst == cg or inst == pg or inst:IsDescendantOf(cg) or (pg ~= nil and inst:IsDescendantOf(pg))
	end)
	return ok and r == true
end
local function guiRoot()
	local forced
	pcall(function() forced = getgenv and getgenv().TempoUIParent end)
	if forced == "PlayerGui" then return LocalPlayer:WaitForChild("PlayerGui") end
	if forced ~= "CoreGui" then
		local h
		pcall(function() h = gethui and gethui() end)
		if h and guiRenders(h) then return h end
	end
	local root
	pcall(function()
		local probe = Instance.new("ScreenGui")
		probe.Parent = game:GetService("CoreGui")
		probe:Destroy()
		root = game:GetService("CoreGui")
	end)
	return root or LocalPlayer:WaitForChild("PlayerGui")
end
TeekUI.guiRoot = guiRoot

local function screenGui(name, order)
	local g = Instance.new("ScreenGui")
	g.Name = name
	g.IgnoreGuiInset = true
	g.ResetOnSpawn = false
	g.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	g.DisplayOrder = order
	pcall(function() g.OnTopOfCoreBlur = true end)   -- over the pause menu
	pcall(function() g.Parent = guiRoot() end)
	if not g.Parent then g.Parent = LocalPlayer:WaitForChild("PlayerGui") end
	return g
end

-- ------------------------------------------------------------ remote images
-- Fetched once to disk and served as a local asset (a remote URL can't be an
-- image); anything that isn't a PNG (a 404 page) or any failure = no image.
local function fetchAsset(url, path, minBytes)
	local ok, res = pcall(function()
		local getAsset = getcustomasset or getsynasset
		if not (getAsset and writefile) then return nil end
		local function good(body) return type(body) == "string" and #body >= minBytes and string.sub(body, 2, 4) == "PNG" end
		local cached = isfile and isfile(path)
		if cached then
			local okr, body = pcall(readfile, path)
			if not okr or not good(body) then cached = false end
		end
		if not cached and not url then return nil end
		if not cached then
			local body, done = nil, false
			task.spawn(function()
				pcall(function() body = game:HttpGet(url) end)
				done = true
			end)
			local t0 = os.clock()
			while not done and os.clock() - t0 < 6 do task.wait(0.1) end
			if not good(body) then return nil end
			writefile(path, body)
		end
		return getAsset(path)
	end)
	return ok and res or nil
end
-- The mark in the top bar and on the boot splash (teek's 10-12 logos, PNGs in
-- the repo under assets/logo). Flower = cream + orange, made for dark menus;
-- Shuriken = black + orange, made for light ones (a cream recolour stands in
-- on a dark menu); Classic = the ice-blue T. Auto goes by what it sits on.
local MARKS = { "Auto", "Flower", "Shuriken", "Classic" }
local MARK_URL = "https://raw.githubusercontent.com/teekmill17-droid/TempoHub/main/assets/logo/%s.png"
local logoCache = {}
-- yields on the first use of a mark (HttpGet); cached to disk after that
local function resolveLogo(pick, light)
	local file = "flower"
	if pick == "Classic" then file = "classic"
	elseif pick == "Shuriken" then file = light and "shuriken" or "shuriken_light"
	elseif pick ~= "Flower" and light then file = "shuriken" end
	if logoCache[file] == nil then
		if file == "classic" then logoCache[file] = fetchAsset(LOGO_URL, "tempo_logo_v3.png", 4096) or false
		else logoCache[file] = fetchAsset(string.format(MARK_URL, file), "tempo_mark_" .. file .. "_v1.png", 4096) or false end
	end
	-- a mark that would not download: the classic one is on most disks already
	return logoCache[file] or (file ~= "classic" and resolveLogo("Classic")) or nil
end
-- custom background: a fresh upload sits in Roblox's review for a few minutes
-- and rbxthumb draws the grey BrokenImage placeholder meanwhile - which the
-- client then keeps for the whole session (teek 10-09). So ask the thumbnail
-- API first and only draw a Completed image (works for decal and image ids).
local function bgState(id)
	local ok, j = pcall(function()
		return HttpService:JSONDecode(game:HttpGet("https://thumbnails.roblox.com/v1/assets?assetIds=" .. id .. "&size=768x432&format=Png&isCircular=false"))
	end)
	local d = ok and type(j) == "table" and type(j.data) == "table" and j.data[1]
	if type(d) == "table" then return d.state, d.imageUrl end
end
local bgCache = {}
local artCache = {}
-- yields (HttpGet): call from a task
local function resolveArt(url)
	if not url then return nil end
	if artCache[url] == nil then
		local file = "tempo_art_" .. (string.match(url, "([%w_%-]+)%.png$") or "x") .. ".png"
		artCache[url] = fetchAsset(url, file, 64) or false
	end
	return artCache[url] or nil
end

-- preset decorations (assets/deco/<name>.png in the repo), same disk cache
local DECO_URL = "https://raw.githubusercontent.com/teekmill17-droid/TempoHub/main/assets/deco/%s.png"
local decoCache = {}
-- yields (HttpGet): call from a task
local function resolveDeco(name)
	if decoCache[name] == nil then
		decoCache[name] = fetchAsset(string.format(DECO_URL, name), "tempo_deco_" .. name .. "_v1.png", 64) or false
	end
	return decoCache[name] or nil
end
-- an image the user gave by asset id (logo / header art): approved decals and images both draw
local function assetImage(id) return "rbxthumb://type=Asset&id=" .. id .. "&w=420&h=420" end

-- ------------------------------------------------------------ shared file
-- Your own presets and the "same look in every game" switch live in ONE file
-- every Tempo hub reads (a hub's config is per game).
local SHARED_FILE = "TempoHub/ui.json"
local shared
local function sharedLoad()
	if shared then return shared end
	shared = { presets = {}, global = false }
	pcall(function()
		if isfile and isfile(SHARED_FILE) then
			local t = HttpService:JSONDecode(readfile(SHARED_FILE))
			if type(t) == "table" then
				if type(t.presets) == "table" then shared.presets = t.presets end
				shared.global = t.global == true
				if type(t.look) == "table" then shared.look = t.look end
			end
		end
	end)
	return shared
end
-- false = it could not be written (an executor without file functions)
local function sharedSave()
	if not shared then return false end
	return (pcall(function()
		assert(writefile, "no writefile")
		if makefolder and isfolder and not isfolder("TempoHub") then makefolder("TempoHub") end
		writefile(SHARED_FILE, HttpService:JSONEncode(shared))
	end))
end

-- ------------------------------------------------------------ fonts
-- Roblox draws a weight a family doesn't ship as broken glyphs, not a fallback
-- (live test 10-09: TitilliumWeb Medium lost every F and half of "Kill Aura")
local NO_MEDIUM = { TitilliumWeb = Enum.FontWeight.SemiBold, Merriweather = Enum.FontWeight.Regular }
-- Every weight each built-in family ships (read off the client's own
-- fonts/families/*.json, 10-12). A weight a family lacks is snapped to the
-- nearest lighter one it has - which is what the engine measures with anyway
-- (Nunito Medium = Nunito Regular, measured) - so no family / weight pair can
-- come out as broken glyphs. The preset builder lets you pick any pair.
local FONT_WEIGHTS = {
	AccanthisADFStd = { 400 }, AmaticSC = { 400, 700 }, Arimo = { 400, 500, 600, 700 }, Balthazar = { 400 }, Bangers = { 400 },
	BuilderSans = { 100, 300, 400, 500, 600, 700, 800 }, ComicNeueAngular = { 400 }, Creepster = { 400 }, DenkOne = { 400 },
	Fondamento = { 400 }, FredokaOne = { 400 }, GrenzeGotisch = { 100, 200, 300, 400, 500, 600, 700, 800, 900 }, Guru = { 400 },
	HighwayGothic = { 400 }, Inconsolata = { 200, 300, 400, 500, 600, 700, 800, 900 }, IndieFlower = { 400 },
	JosefinSans = { 100, 200, 300, 400, 500, 600, 700 }, Jura = { 300, 400, 500, 600, 700 }, Kalam = { 300, 400, 700 }, LuckiestGuy = { 400 },
	Merriweather = { 300, 400, 700, 900 }, Michroma = { 400 }, Montserrat = { 100, 200, 300, 400, 500, 600, 700, 800, 900 },
	Nunito = { 200, 300, 400, 600, 700, 800, 900 }, Oswald = { 200, 300, 400, 500, 600, 700 }, PatrickHand = { 400 }, PermanentMarker = { 400 },
	PressStart2P = { 400 }, Roboto = { 100, 300, 400, 500, 700, 900 }, RobotoCondensed = { 300, 400, 700 },
	RobotoMono = { 100, 200, 300, 400, 500, 600, 700 }, RomanAntique = { 400 }, Sarpanch = { 400, 500, 600, 700, 800, 900 },
	SourceSansPro = { 200, 300, 400, 600, 700, 900 }, SpecialElite = { 400 }, TitilliumWeb = { 200, 300, 400, 600, 700, 900 },
	Ubuntu = { 300, 400, 500, 700 }, Zekton = { 400 },
}
local WEIGHT_OF = {}
for _, w in ipairs(Enum.FontWeight:GetEnumItems()) do WEIGHT_OF[w.Value] = w end
local function font(family, weight, style)
	weight = weight or Enum.FontWeight.Regular
	if weight == Enum.FontWeight.Medium and NO_MEDIUM[family] then weight = NO_MEDIUM[family] end
	local have = FONT_WEIGHTS[family]
	if have and not table.find(have, weight.Value) then
		local best = have[1]
		for _, w in ipairs(have) do if w <= weight.Value then best = w end end
		weight = WEIGHT_OF[best] or Enum.FontWeight.Regular
	end
	local ok, f = pcall(function()
		return Font.new("rbxasset://fonts/families/" .. family .. ".json", weight or Enum.FontWeight.Regular, style or Enum.FontStyle.Normal)
	end)
	return ok and f or Font.fromEnum(Enum.Font.Gotham)
end
local F = {
	brand = (pcall(function() return Font.fromEnum(Enum.Font.Michroma) end)) and Font.fromEnum(Enum.Font.Michroma) or font("GothamSSm", Enum.FontWeight.Bold),
	body  = font("BuilderSans", Enum.FontWeight.Medium),
	bold  = font("BuilderSans", Enum.FontWeight.Bold),
	mono  = font("RobotoMono", Enum.FontWeight.Medium),
}
-- what the Font option offers (display name; the family is the name without spaces)
local FONT_CHOICES = { "Nunito", "Oswald", "Montserrat", "Titillium Web", "Sarpanch", "Jura", "Roboto Condensed", "Ubuntu", "Merriweather", "Fondamento", "Roboto Mono",
	"Builder Sans", "Arimo", "Roboto", "Josefin Sans", "Source Sans Pro", "Kalam", "Patrick Hand", "Inconsolata" }
-- heading / title faces the preset builder offers (the first = whatever the style uses)
local STYLE_OWN = "Style"
local HEAD_CHOICES = { STYLE_OWN, "Oswald", "Bangers", "Michroma", "Sarpanch", "Fondamento", "Grenze Gotisch", "Permanent Marker", "Josefin Sans",
	"Denk One", "Luckiest Guy", "Fredoka One", "Creepster", "Special Elite", "Amatic SC", "Balthazar", "Roman Antique", "Zekton", "Press Start 2P",
	"Kalam", "Indie Flower", "Patrick Hand", "Merriweather", "Montserrat", "Nunito", "Jura", "Titillium Web", "Roboto Condensed", "Roboto Mono",
	"Ubuntu", "Builder Sans", "Arimo", "Inconsolata" }
local function family(name) return (string.gsub(tostring(name or "BuilderSans"), " ", "")) end

-- ------------------------------------------------------------ colour
local WHITE, BLACK = Color3.new(1, 1, 1), Color3.new(0, 0, 0)
local function hex(s)
	local kind = typeof(s)
	if kind == "Color3" then return s end
	local h = type(s) == "string" and string.match(s, "^#?(%x%x%x%x%x%x)$")
	if not h then return nil end
	return Color3.fromRGB(tonumber(string.sub(h, 1, 2), 16), tonumber(string.sub(h, 3, 4), 16), tonumber(string.sub(h, 5, 6), 16))
end
local function toHex(c)
	return string.format("#%02x%02x%02x", math.floor(c.R * 255 + 0.5), math.floor(c.G * 255 + 0.5), math.floor(c.B * 255 + 0.5))
end
local function lum(c) return 0.299 * c.R + 0.587 * c.G + 0.114 * c.B end

-- ------------------------------------------------------------ games
-- universe id (game.GameId) -> default accent + header art. A user-picked
-- accent wins; "Use this game's colour" goes back to this.
local DEFAULT_ACCENT = Color3.fromRGB(124, 196, 255)
local GAMES = {
	[3936365689] = { accent = hex("#4fd68b"), art = string.format(ART_URL, "clover") },      -- Clover Retribution
	[578392296]  = { accent = hex("#ff4d5e"), art = string.format(ART_URL, "aba") },         -- Anime Battle Arena
	[1946714362] = { accent = hex("#a78bfa"), art = string.format(ART_URL, "bloodlines") },  -- Bloodlines
	[5595353122] = { accent = hex("#e0b25a"), art = string.format(ART_URL, "ps2") },         -- Project Slayers 2
	[2459091562] = { accent = hex("#ff8a3d"), art = string.format(ART_URL, "rh2") },         -- RH2 The Journey
}
local function thisGame()
	local id = 0
	pcall(function() id = game.GameId end)
	return GAMES[id] or {}
end

-- v1 theme names (saved in old configs / passed to SetTheme) -> an accent.
-- "Tempo" = the game's own colour.
local LEGACY_THEMES = {
	Tempo = false, Ember = hex("#ff8a4c"), Mint = hex("#5fe0a8"), Violet = hex("#b493ff"), Rose = hex("#ff78a0"), Mono = hex("#dce0e6"),
}

-- ------------------------------------------------------------ presets
-- look = the user-tweakable values a preset resets to; the rest is structure
-- (only changes on a preset switch, which rebuilds the chrome).
--   cut     corner cuts of the window: { tl, tr, bl, br } px, or "stairs" (pixel steps)
--   shade   a hard copy behind the window: x/y offset or grow (border), colour token
--   nav/tab/h4/btn  text role, size, upper-case for rail entries, tabs, card titles, buttons
--   base    (character presets) the style whose chrome this one is drawn with;
--           everything not set here comes from it
--   deco    a DECOS set: emblems / bands drawn round the window
--   tokens  fixed colour tokens of its own (name = "#hex")
local PRESET_ORDER = { "Haunted", "Spooky", "Glass", "Anime", "Sleek", "Gamer", "Manga", "Arcade", "Grimoire", "Madara", "Mugetsu", "Rengoku", "Itachi", "Urahara" }
local PRESETS = {
	-- HAUNTED (teek 10-13: "make that the default for everyone"): crypt purple-
	-- black, pumpkin orange + ectoplasm green, bone text. Everything that moves
	-- (candle flicker, slime drips, bats, the ghost, fog, cobwebs + spider,
	-- pumpkin switches, candle-flame slider handles, the lightning boot) is
	-- Win:_haunt / _hauntBoot; haunt = true switches it on.
	Haunted = {
		base = "Sleek", haunt = true,
		look = { font = "Titillium Web", rad = 8, glow = 10, alpha = 98, blur = 0, density = "Cozy", pattern = "Vignette", strength = 55,
			bg = "#0e0a14", text = "#ece4d4", accent = "#ff7a1a", accent2 = "#8fe36a", card = "#151019", rail = "#0b0810", line = "#2a2131" },
		head = "TitilliumWeb", headW = Enum.FontWeight.Bold, topH = 52,
		brand = { size = 16, upper = true, spaced = true, weight = Enum.FontWeight.Bold },
		nav = { role = "semi", size = 13 }, tab = { role = "head", size = 12, upper = true },
		h4 = { role = "head", size = 11, upper = true, tok = "dim" }, btn = { role = "head", size = 12, upper = true },
		cardCut = { 0, 14, 0, 0 },
	},
	-- SPOOKY (teek 10-16, picked from five mockups: "Inferno Noir's style with
	-- Void's font"). White on black: the window burns in black fire - flame
	-- tongues rimmed in white heat lick off the sides and the bottom, sparks,
	-- little ghosts riding the heat up the sides, two guardian ghosts on the
	-- bottom corners. Jura display type, spaced capitals. Win:_spooky /
	-- _spookyBoot; spooky = true switches them on.
	Spooky = {
		base = "Sleek", spooky = true,
		look = { font = "Montserrat", rad = 12, glow = 0, alpha = 100, blur = 0, density = "Cozy", pattern = "None", strength = 0,
			bg = "#070707", text = "#f4f4f6", accent = "#f4f4f6", accent2 = "#9a9aa2", card = "#0b0b0b", rail = "#070707", line = "#1c1c1f" },
		head = "Jura", headW = Enum.FontWeight.Bold, topH = 60,
		winStroke = { tok = "rim", thick = 1 }, tokens = { rim = "#2b2b2e" },
		brand = { size = 16, upper = true, spaced = true, weight = Enum.FontWeight.Bold },
		nav = { role = "head", size = 13, upper = true }, tab = { role = "head", size = 13, upper = true },
		h4 = { role = "head", size = 11, upper = true, tok = "text" }, btn = { role = "head", size = 12, upper = true },
	},
	Glass = {
		-- live test 10-09: at 62% / blur 18 it was unreadable over ABA's bright training map and the
		-- whole game went soft; darker + more solid, a light blur, brighter secondary text
		look = { font = "Nunito", rad = 18, glow = 0, alpha = 84, blur = 6, density = "Roomy", pattern = "None", strength = 0, bg = "#10141c", text = "#f4f7fb" },
		head = "Nunito", headW = Enum.FontWeight.ExtraBold, topH = 52, see = 0.4, pill = true,
		brand = { size = 17 }, nav = { role = "semi", size = 13 }, tab = { role = "semi", size = 13 },
		h4 = { role = "head", size = 11, upper = true, tok = "dim" }, btn = { role = "semi", size = 13 },
	},
	Anime = {
		look = { font = "Oswald", rad = 0, glow = 0, alpha = 97, blur = 0, density = "Cozy", pattern = "Accent wash", strength = 30, bg = "#0b0b0e", text = "#f5f5f7" },
		head = "Oswald", headW = Enum.FontWeight.SemiBold, topH = 64, pop = true,
		cut = { 0, 30, 24, 0 }, shade = { x = 8, y = 8, tok = "accent" }, noStroke = true,
		brand = { size = 27, upper = true, italic = true, weight = Enum.FontWeight.Bold },
		nav = { role = "head", size = 15, upper = true }, tab = { role = "head", size = 15, upper = true },
		h4 = { role = "head", size = 14, upper = true, tok = "text", prefix = "// " }, btn = { role = "head", size = 14, upper = true, shadow = "text" },
		cardCut = { 0, 14, 0, 0 }, cardBar = true,
	},
	Sleek = {
		look = { font = "Montserrat", rad = 10, glow = 0, alpha = 98, blur = 0, density = "Cozy", pattern = "None", strength = 0, bg = "#0c0e12", text = "#e9edf2" },
		head = "Michroma", headW = Enum.FontWeight.Regular, topH = 52,
		brand = { size = 15, spaced = true }, nav = { role = "semi", size = 13 }, tab = { role = "semi", size = 13 },
		h4 = { role = "head", size = 10, upper = true, tok = "dim" }, btn = { role = "semi", size = 13 },
	},
	Gamer = {
		-- (live test 10-09: 1 px scanlines every 2 px sliced through the small text; off by default)
		look = { font = "Titillium Web", rad = 2, glow = 16, alpha = 94, blur = 0, density = "Compact", pattern = "None", strength = 0, bg = "#06080b", text = "#dce6f0" },
		head = "Sarpanch", headW = Enum.FontWeight.SemiBold, topH = 52,
		cut = { 16, 16, 16, 16 }, shade = { grow = 1, tok = "accent", t = 0.3 }, noStroke = true,
		brand = { size = 18, upper = true, spaced = true, weight = Enum.FontWeight.Bold },
		nav = { role = "head", size = 13, upper = true }, tab = { role = "head", size = 13, upper = true },
		h4 = { role = "mono", size = 11, upper = true, tok = "accent", prefix = "> " }, btn = { role = "semi", size = 12, upper = true },
		cardCut = { 10, 0, 0, 0 },
	},
	Manga = {
		look = { font = "Roboto Condensed", rad = 0, glow = 0, alpha = 100, blur = 0, density = "Cozy", pattern = "Accent wash", strength = 100, bg = "#f3efe6", text = "#111111" },
		head = "Bangers", headW = Enum.FontWeight.Regular, topH = 64, pop = true,
		shade = { x = 9, y = 9, tok = "ink" }, winStroke = { tok = "ink", thick = 3 }, inkLines = true,
		brand = { size = 36, shadow = "accent" }, nav = { role = "head", size = 19 }, tab = { role = "head", size = 19 },
		h4 = { role = "head", size = 21, tok = "text" }, btn = { role = "head", size = 17, shadow = "ink" },
		cardShadow = "ink",
	},
	Arcade = {
		look = { font = "Roboto Mono", rad = 0, glow = 0, alpha = 100, blur = 0, density = "Cozy", pattern = "Vignette", strength = 70, bg = "#15112a", text = "#f3efe4" },
		head = "PressStart2P", headW = Enum.FontWeight.Regular, topH = 60, pop = true, pixel = true,
		cut = { 8, 8, 8, 8 }, stairs = true, shade = { grow = 4, tok = "accent" }, noStroke = true,
		brand = { size = 16 }, nav = { role = "head", size = 10, upper = true }, tab = { role = "head", size = 10 },
		h4 = { role = "head", size = 9, upper = true, tok = "accent" }, btn = { role = "head", size = 9, upper = true, bevel = true },
	},
	Grimoire = {
		look = { font = "Merriweather", rad = 3, glow = 0, alpha = 98, blur = 0, density = "Cozy", pattern = "Vignette", strength = 100, bg = "#18130e", text = "#eadfc6" },
		head = "Fondamento", headW = Enum.FontWeight.Regular, topH = 64, cameo = true, gold = true,
		brand = { size = 24, tok = "gold", spaced = true }, nav = { role = "head", size = 15 }, tab = { role = "head", size = 16 },
		h4 = { role = "head", size = 15, tok = "gold", rules = true }, btn = { role = "head", size = 15 },
	},
	-- CHARACTER PRESETS (teek 10-12, quieted the same night: "less AI made").
	-- Each is a palette, a type treatment and ONE mark - a one-colour emblem
	-- printed faintly inside the window and cropped by its edge (DECOS). No
	-- stickers over the frame, no novelty wordmarks, shadows in a dark tone of
	-- the accent instead of the accent itself.
	-- Madara: lacquered armour red on warm black, bone-white titles, the
	-- Sharingan as the mark.
	Madara = {
		base = "Anime", deco = "Uchiha eyes",
		look = { font = "Roboto Condensed", rad = 0, glow = 0, alpha = 97, blur = 0, density = "Cozy", pattern = "Vignette", strength = 45,
			bg = "#0e0b0c", text = "#ece3dc", accent = "#b8262c", accent2 = "#cdbfb3", card = "#181213", rail = "#090607", line = "#33191c" },
		topH = 60, cut = { 0, 22, 22, 0 }, shade = { x = 6, y = 6, tok = "ember" }, tokens = { ember = "#4a1216" },
		brand = { size = 21, upper = true, spaced = true, weight = Enum.FontWeight.Bold },
		h4 = { role = "head", size = 13, upper = true, tok = "accent2" },
	},
	-- Mugetsu ("moonless sky"): near-black and cold steel, thin spaced type,
	-- an eclipse ring as the mark.
	Mugetsu = {
		base = "Sleek", deco = "Moonless sky",
		look = { font = "Jura", rad = 0, glow = 0, alpha = 97, blur = 0, density = "Cozy", pattern = "None", strength = 0,
			bg = "#060607", text = "#e9ecf1", accent = "#b4bfcf", accent2 = "#5f6878", card = "#0d0e11", rail = "#030304", line = "#1f2228" },
		head = "JosefinSans", headW = Enum.FontWeight.SemiBold, topH = 56,
		cut = { 0, 20, 20, 0 }, shade = { grow = 1, tok = "accent2", t = 0.35 }, noStroke = true,
		brand = { size = 17, upper = true, spaced = true, weight = Enum.FontWeight.Light },
		nav = { role = "head", size = 12, upper = true }, tab = { role = "head", size = 13, upper = true },
		h4 = { role = "head", size = 11, upper = true, tok = "accent" }, btn = { role = "head", size = 13, upper = true },
		cardCut = { 0, 8, 0, 0 },
	},
	-- Rengoku: ember orange and flame gold on charred brown, a row of flame
	-- tongues along the bottom like the hem of his haori.
	Rengoku = {
		base = "Anime", deco = "Flame Hashira",
		look = { font = "Roboto Condensed", rad = 0, glow = 0, alpha = 97, blur = 0, density = "Cozy", pattern = "Vignette", strength = 40,
			bg = "#150c08", text = "#f6ead7", accent = "#e8842a", accent2 = "#f0c45a", card = "#21130c", rail = "#0e0704", line = "#3c2013" },
		topH = 60, shade = { x = 6, y = 6, tok = "ember" }, tokens = { ember = "#6e2410" },
		brand = { size = 21, upper = true, spaced = true, weight = Enum.FontWeight.Bold },
		h4 = { role = "head", size = 13, upper = true, tok = "accent2" },
	},
	-- Itachi: the Akatsuki cloak - black, cloud red, a hairline in the clouds'
	-- off-white, one cloud outline as the mark.
	Itachi = {
		base = "Sleek", deco = "Akatsuki clouds",
		look = { font = "Montserrat", rad = 10, glow = 0, alpha = 98, blur = 0, density = "Cozy", pattern = "None", strength = 0,
			bg = "#0a0a0c", text = "#f1ece6", accent = "#c8202f", accent2 = "#d9d2ca", card = "#131316", rail = "#060607", line = "#26262c" },
		head = "Oswald", headW = Enum.FontWeight.SemiBold, topH = 56,
		winStroke = { tok = "rim", thick = 1 }, tokens = { rim = "#5a5650" }, topRule = { tok = "accent", h = 2 },
		brand = { size = 18, upper = true, spaced = true, weight = Enum.FontWeight.SemiBold },
		nav = { role = "head", size = 14, upper = true }, tab = { role = "head", size = 14, upper = true },
		h4 = { role = "head", size = 12, upper = true, tok = "accent" }, btn = { role = "head", size = 13, upper = true },
	},
	-- Urahara: shopkeeper green and straw, the hat's stripes under the top
	-- bar, his fan as the mark.
	Urahara = {
		base = "Grimoire", deco = "Urahara's shop",
		look = { font = "Nunito", rad = 6, glow = 0, alpha = 98, blur = 0, density = "Cozy", pattern = "Vignette", strength = 60,
			bg = "#0e1713", text = "#ece6d2", accent = "#78b086", accent2 = "#cdb878", card = "#15201a", rail = "#0a110e" },
		topH = 60,
	},
}
-- ------------------------------------------------------------ decorations
-- A character preset's stickers (teek 10-12: "i like the stickers, just make
-- them fit better"): the full-colour emblems die-cut with a white border and a
-- soft shadow (assets/deco/stk_*, make_preset_art.py ... stickers), kept small
-- and tilted, stuck half on the top edge over the header's empty right side,
-- or on a corner - stuck ON the window rather than floating round it. An edge
-- (assets/deco/edge_*: Susanoo, flames, black reiatsu, a cloud bank) rises
-- from behind the top edge, window-wide, faded at both ends (fade) so a cut
-- or rounded corner never shows a hard end.
-- Images are cached to disk like the logo; an executor that can't load them
-- just doesn't draw them. anchor = { x, y }; pos / size = the four UDim2
-- numbers; z = ZIndex on the window (4 = in front). An item with tint is a
-- one-colour mark printed faintly inside the top bar instead (a = strength).
-- stripes = a band of alternating colour tokens along the bottom of the top
-- bar (Urahara's hat).
local DECO_ORDER = { "Akatsuki clouds", "Uchiha eyes", "Flame Hashira", "Moonless sky", "Urahara's shop" }
local DECOS = {
	["Akatsuki clouds"] = {
		{ img = "edge_clouds", anchor = { 0.5, 1 }, pos = { 0.5, 0, 0, 3 }, size = { 1, -8, 0, 48 }, z = 0, fit = "Stretch", fade = true },
		{ img = "stk_cloud", anchor = { 0.5, 0.5 }, pos = { 1, -206, 0, 6 }, size = { 0, 124, 0, 74 }, rot = -6 },
		{ img = "stk_cloud", anchor = { 0.5, 0.5 }, pos = { 1, -10, 1, -6 }, size = { 0, 84, 0, 50 }, rot = 9 },
	},
	["Uchiha eyes"] = {
		{ img = "edge_susanoo", anchor = { 0.5, 1 }, pos = { 0.5, 0, 0, 3 }, size = { 1, -8, 0, 48 }, z = 0, fit = "Stretch", fade = true },
		{ img = "stk_sharingan", anchor = { 0.5, 0.5 }, pos = { 1, -212, 0, 4 }, size = { 0, 74, 0, 74 }, rot = -10 },
		{ img = "stk_rinnegan", anchor = { 0.5, 0.5 }, pos = { 1, -158, 0, 18 }, size = { 0, 52, 0, 52 }, rot = 8 },
		{ img = "stk_uchiwa", anchor = { 0.5, 0.5 }, pos = { 1, -10, 1, -8 }, size = { 0, 56, 0, 65 }, rot = -14 },
	},
	["Flame Hashira"] = {
		{ img = "edge_flames", anchor = { 0.5, 1 }, pos = { 0.5, 0, 0, 3 }, size = { 1, -8, 0, 48 }, z = 0, fit = "Stretch", fade = true },
		{ img = "stk_flame", anchor = { 0.5, 0.5 }, pos = { 1, -208, 0, 4 }, size = { 0, 62, 0, 74 }, rot = -8 },
		{ img = "stk_flame", anchor = { 0.5, 0.5 }, pos = { 1, -160, 0, 16 }, size = { 0, 44, 0, 52 }, rot = 11 },
	},
	["Moonless sky"] = {
		{ img = "edge_blackflames", anchor = { 0.5, 1 }, pos = { 0.5, 0, 0, 3 }, size = { 1, -8, 0, 48 }, z = 0, fit = "Stretch", fade = true },
		{ img = "stk_eclipse", anchor = { 0.5, 0.5 }, pos = { 1, -200, 0, 6 }, size = { 0, 76, 0, 75 }, rot = 0 },
	},
	["Urahara's shop"] = {
		{ img = "stk_fan", anchor = { 0.5, 0.5 }, pos = { 1, -206, 0, 8 }, size = { 0, 122, 0, 72 }, rot = 8 },
		{ stripes = { "accent", "accent2" }, w = 13, h = 5 },
	},
}
local DENSITY = { Compact = { 28, 10, 10 }, Cozy = { 34, 14, 14 }, Roomy = { 42, 18, 18 } }   -- row height, page pad, card gap
local PATTERNS = { "None", "Scanlines", "Vignette", "Accent wash", "Custom image" }
local DOCKS = { "Free", "Left", "Center", "Right" }

-- every look value; a theme code carries all of these but position
local LOOK_DEFAULTS = {
	preset = "Haunted", accent = "", w = 780, h = 500, scale = 100, dock = "Center", remember = false, x = 0, y = 0,
	image = "", anim = true, sounds = false,
	-- the preset builder (10-12). "" / -1 / "Preset" = whatever the style says.
	card = "", rail = "", line = "", dim = "", accent2 = "",                       -- surface colours ("" = worked out from the background)
	head = "", titleFont = "", title = "", titleSize = -1, caps = "Preset", headSize = 0, prefix = "Preset", h4c = "Preset",
	cutTL = -1, cutTR = -1, cutBL = -1, cutBR = -1, steps = "Preset",
	shade = "Preset", shadeSize = 8, shadeColor = "", border = -1, borderColor = "",
	cardCut = -1, cardBar = "Preset", cardShadow = "Preset", switch = "Preset",
	topH = -1, railW = -1,
	fit = "Crop", logo = "", mark = "Auto", art = "", deco = "Preset", decoAlpha = 100,
	speed = 100, volume = 25, sound = "",
	name = "",                                                                     -- the saved preset this look was loaded from
}
-- (the style's surface colours stay "" here: they are what a reset hands
-- back, and another style must not inherit Haunted's)
for k, v in pairs(PRESETS.Haunted.look) do
	if not ({ accent = 1, accent2 = 1, card = 1, rail = 1, line = 1 })[k] then LOOK_DEFAULTS[k] = v end
end
-- what picking a style puts back to "the style's own" (plus the style's look)
local RESET_KEYS = { "card", "rail", "line", "dim", "accent2", "head", "titleFont", "title", "titleSize", "caps", "headSize", "prefix", "h4c",
	"cutTL", "cutTR", "cutBL", "cutBR", "steps", "shade", "shadeSize", "shadeColor", "border", "borderColor", "cardCut", "cardBar", "cardShadow",
	"switch", "topH", "railW", "logo", "mark", "art", "deco", "decoAlpha", "name" }
local CODE_KEYS = { "preset", "accent", "bg", "text", "alpha", "blur", "rad", "glow", "font", "scale", "w", "h", "density", "dock", "pattern", "image", "strength", "anim", "sounds",
	"card", "rail", "line", "dim", "accent2", "head", "titleFont", "title", "titleSize", "caps", "headSize", "prefix", "h4c", "cutTL", "cutTR", "cutBL", "cutBR", "steps",
	"shade", "shadeSize", "shadeColor", "border", "borderColor", "cardCut", "cardBar", "cardShadow", "switch", "topH", "railW", "fit", "logo", "mark", "art", "deco",
	"decoAlpha", "speed", "volume", "sound", "name" }

-- ------------------------------------------------------------ base64 (theme codes)
local B64 = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
local function b64enc(s)
	local out = {}
	for i = 1, #s, 3 do
		local a, b, c = string.byte(s, i, i + 2)
		local n = a * 65536 + (b or 0) * 256 + (c or 0)
		local q = {}
		for j = 1, 4 do
			local k = bit32.band(bit32.rshift(n, (4 - j) * 6), 63) + 1
			q[j] = string.sub(B64, k, k)
		end
		if not b then q[3] = "=" end
		if not c then q[4] = "=" end
		out[#out + 1] = table.concat(q)
	end
	return table.concat(out)
end
local function b64dec(s)
	s = string.gsub(s, "[^%w%+/]", "")
	local out = {}
	for i = 1, #s, 4 do
		local n, got = 0, 0
		for j = 0, 3 do
			local ch = string.sub(s, i + j, i + j)
			local k = ch ~= "" and string.find(B64, ch, 1, true)
			if k then
				n = n * 64 + (k - 1)
				got += 1
			else
				n = n * 64
			end
		end
		local bytes = { bit32.band(bit32.rshift(n, 16), 255), bit32.band(bit32.rshift(n, 8), 255), bit32.band(n, 255) }
		for j = 1, got - 1 do out[#out + 1] = string.char(bytes[j]) end
	end
	return table.concat(out)
end

-- ------------------------------------------------------------ small helpers
local function make(class, props, parent)
	local i = Instance.new(class)
	for k, v in pairs(props or {}) do i[k] = v end
	if parent then i.Parent = parent end
	return i
end
local function pad(inst, l, r, t, b)
	make("UIPadding", { PaddingLeft = UDim.new(0, l), PaddingRight = UDim.new(0, r or l), PaddingTop = UDim.new(0, t or l), PaddingBottom = UDim.new(0, b or t or l) }, inst)
end
local function stepFor(min, max) return (max - min) <= 3 and 0.05 or 1 end
local function fmtNum(v, step)
	if step < 1 then return string.format("%.2f", v) end
	return tostring(math.floor(v + 0.5))
end
local function dedupe(values)
	local out, seen = {}, {}
	for _, v in ipairs(values or {}) do
		local s = tostring(v)
		if not seen[s] then
			seen[s] = true
			out[#out + 1] = s
		end
	end
	if #out == 0 then out[1] = "-" end
	return out
end
local function spaced(s) return (string.gsub(s, "(.)", "%1\u{200A}")) end
local function isClick(i) return i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch end
local function isMove(i) return i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch end
-- KeyCode name -> what a person reads ("RightControl" -> "Right Ctrl")
local function keyLabel(name)
	name = tostring(name or "None")
	local side, mod = string.match(name, "^(%u%l+)(Control)$")
	if not side then side, mod = string.match(name, "^(%u%l+)(Shift)$") end
	if not side then side, mod = string.match(name, "^(%u%l+)(Alt)$") end
	if side then return side .. " " .. (mod == "Control" and "Ctrl" or mod) end
	return name
end

-- ============================================================
--  Window
-- ============================================================
local Win = {}
Win.__index = Win
local Tab = {}
Tab.__index = Tab
local Group = {}
Group.__index = Group

-- Paint: every look-dependent property goes through here. `token` is a key
-- of self.tok or a function returning the value, so a re-theme is one pass
-- over this list instead of a rebuild.
function Win:paint(inst, prop, token)
	if type(token) == "function" then inst[prop] = token() else inst[prop] = self.tok[token] end
	self.painted[#self.painted + 1] = { inst, prop, token }
	if #self.painted > (self.pruneAt or 3000) then
		local keep = {}
		for _, p in ipairs(self.painted) do if p[1].Parent then keep[#keep + 1] = p end end
		self.painted = keep
		self.pruneAt = math.max(3000, #keep * 2)
	end
	return inst
end
function Win:_repaint()
	local keep = {}
	for _, p in ipairs(self.painted) do
		local inst, prop, token = p[1], p[2], p[3]
		if inst.Parent then
			pcall(function()
				if type(token) == "function" then inst[prop] = token() else inst[prop] = self.tok[token] end
			end)
			keep[#keep + 1] = p
		end
	end
	self.painted = keep
	-- hand-set state colours (rail selection, tab strip, toggles mid-hover)
	for _, e in ipairs(self.rail) do if e.paintState then e.paintState() end end
	if self.railSel then self:_paintStrip() end
	self:_cursorTint()
end
-- a UIStroke whose colour and transparency follow tokens (t<token> if the
-- token table has one, so glass lines can be see-through)
function Win:stroke(inst, token, thick)
	token = token or "ln"
	local s = make("UIStroke", { Thickness = thick or 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, inst)
	self:paint(s, "Color", token)
	if type(token) == "string" and self.tok["t" .. token] ~= nil then self:paint(s, "Transparency", "t" .. token) end
	return s
end
function Win:conn(c)
	self.conns[#self.conns + 1] = c
	return c
end
-- radius: number = share of the Roundness value; "pill" = fully round in
-- presets that use pills (glass), else 0.6; "round" = always fully round
function Win:corner(inst, factor)
	local c = make("UICorner", {}, inst)
	self:paint(c, "CornerRadius", function()
		if factor == "round" or (factor == "pill" and self.st.pill) then return UDim.new(0.5, 0) end
		local f = (type(factor) == "number") and factor or 0.6
		return UDim.new(0, math.floor((self.look.rad or 0) * f + 0.5))
	end)
	return c
end
function Win:_tw(inst, t, props, style, dir)
	if not self.look.anim then
		for k, v in pairs(props) do pcall(function() inst[k] = v end) end
		return nil
	end
	t = t * 100 / math.clamp(tonumber(self.look.speed) or 100, 25, 400)   -- Animation speed
	local tw = TweenService:Create(inst, TweenInfo.new(t, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props)
	tw:Play()
	return tw
end
function Win:_tick(on)
	if not (self.look.sounds and self.click) then return end
	pcall(function()
		local id = string.match(tostring(self.look.sound or ""), "%d+")
		local want = id and ("rbxassetid://" .. id) or CLICK_SOUND
		if self.click.SoundId ~= want then self.click.SoundId = want end
		self.click.Volume = math.clamp((tonumber(self.look.volume) or 25) / 100, 0, 1)
		self.click.PlaybackSpeed = on == false and 0.8 or 1.15
		game:GetService("SoundService"):PlayLocalSound(self.click)
	end)
end

-- text roles: body / semi follow the Font option, head is the preset's
-- display face, mono is fixed. A Font value is used as is.
local fontCache = {}
local function cachedFont(fam, weight, style)
	local key = fam .. tostring(weight) .. tostring(style)
	if not fontCache[key] then fontCache[key] = font(fam, weight, style) end
	return fontCache[key]
end
function Win:_face(role)
	local kind = typeof(role)
	if kind == "Font" then return role end
	if role == "head" then return cachedFont(self.st.head, self.st.headW) end
	if role == "mono" then return F.mono end
	if role == "brand" then
		local b = self.st.brand
		return cachedFont(b.font or self.st.head, b.weight or self.st.headW, b.italic and Enum.FontStyle.Italic or nil)
	end
	if role == "italic" then return cachedFont(family(self.look.font), Enum.FontWeight.Regular, Enum.FontStyle.Italic) end
	return cachedFont(family(self.look.font), (role == "semi" or role == "bold") and Enum.FontWeight.SemiBold or Enum.FontWeight.Medium)
end
function Win:text(props, parent, token, role, size)
	local t = make("TextLabel", {
		BackgroundTransparency = 1, TextSize = size or 13,
		TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Center,
	}, parent)
	for k, v in pairs(props or {}) do t[k] = v end
	self:paint(t, "TextColor3", token or "text")
	local kind = typeof(role)
	if kind == "Font" then t.FontFace = role else self:paint(t, "FontFace", function() return self:_face(role or "body") end) end
	return t
end

-- ------------------------------------------------------------ tokens
-- The mock's apply(): neutrals derived from the background tint (shaded
-- toward black on a light background, toward white on a dark one), then the
-- preset's surface rules. s<Name>/t<Name> = colour + transparency of a surface.
function Win:_tokens()
	local L, st, name = self.look, self.st, self.presetName
	local acc = hex(L.accent) or self.gameAccent
	local bg = hex(L.bg) or hex(LOOK_DEFAULTS.bg)
	local tx = hex(L.text) or hex(LOOK_DEFAULTS.text)
	local light = lum(bg) > 0.5
	local tw = light and BLACK or WHITE
	local see = st.see or 1
	local a = math.clamp((tonumber(L.alpha) or 100) / 100, 0.2, 1)
	local t = {
		accent = acc, onAccent = lum(acc) > 0.55 and Color3.fromRGB(10, 10, 10) or WHITE,
		bg = bg, rail = light and bg:Lerp(BLACK, 0.03) or bg:Lerp(BLACK, 0.35),
		card = bg:Lerp(tw, 0.035), row = bg:Lerp(tw, 0.07), off = bg:Lerp(tw, 0.14),
		line = bg:Lerp(tw, 0.1), line2 = bg:Lerp(tw, 0.17),
		text = tx, dim = tx:Lerp(bg, 0.38 * see), faint = tx:Lerp(bg, 0.6 * see),
		good = Color3.fromRGB(61, 220, 132), warn = Color3.fromRGB(255, 181, 71), bad = Color3.fromRGB(255, 90, 90),
		ink = Color3.fromRGB(17, 17, 17), paper = WHITE, gold = Color3.fromRGB(201, 164, 92), black = BLACK, white = WHITE,
		light = light,
	}
	t.accentDim = acc:Lerp(bg, 0.55)
	-- the second accent (a character preset's, or the builder's): it is the
	-- "gold" of the styles that use one
	local acc2 = hex(L.accent2)
	if acc2 then t.gold = acc2 end
	t.accent2 = acc2 or (name == "Grimoire" and t.gold or acc)
	local dimC = hex(L.dim)
	if dimC then t.dim, t.faint = dimC, dimC:Lerp(bg, 0.4) end
	-- surfaces (default: the mock's rgba layers)
	t.sWin, t.tWin = bg, 1 - a
	t.sTop, t.tTop = t.rail, 1 - a
	t.sRail, t.tRail = t.rail, 1 - a
	t.sCard, t.tCard = t.card, 1 - math.min(1, a + 0.04)
	t.sCtl, t.tCtl = t.row, 0
	t.sPop, t.tPop = t.card, 0
	t.sHover, t.tHover = t.row, 0
	t.sNote, t.tNote = t.card, 0
	t.ln, t.tln = t.line, 0
	t.ln2, t.tln2 = t.line2, 0
	t.frameLn, t.tframeLn = t.line, 0
	t.noteText, t.noteHead = tx, acc
	t.ctlText = tx
	if name == "Glass" then
		t.sTop, t.tTop, t.sRail, t.tRail = WHITE, 0.96, WHITE, 0.96
		t.sCard, t.tCard = WHITE, 0.945
		t.sCtl, t.tCtl = WHITE, 0.93
		t.sHover, t.tHover = WHITE, 0.94
		t.sPop, t.tPop = t.card, 0.18
		t.sNote, t.tNote = t.card, 0.3
		t.ln, t.tln, t.ln2, t.tln2 = WHITE, 0.9, WHITE, 0.88
		t.frameLn, t.tframeLn = WHITE, 0.84
	elseif name == "Manga" then
		t.tTop, t.tRail = 1, 1
		t.sCard, t.tCard, t.sCtl, t.sPop, t.sNote = WHITE, 0, WHITE, WHITE, WHITE
		t.sHover = Color3.fromRGB(241, 237, 228)
		t.ln, t.ln2, t.frameLn = t.ink, t.ink, t.ink
		t.noteText, t.noteHead, t.ctlText = t.ink, t.ink, t.ink
	elseif name == "Arcade" then
		t.sTop, t.tTop = acc, 0
	elseif name == "Grimoire" then
		local gold = t.gold
		t.tTop, t.tRail = 1, 1
		t.tCard = 0.45
		t.ln, t.tln, t.ln2, t.tln2 = gold, 0.7, gold, 0.65
		t.frameLn, t.tframeLn = gold, 0.45
		t.sNote, t.noteText, t.noteHead = Color3.fromRGB(239, 227, 200), Color3.fromRGB(42, 31, 20), Color3.fromRGB(122, 85, 32)
	end
	-- surface colours set by hand (a character preset or the builder) win over
	-- the style's rules; picked colours are solid
	local card, rail, line = hex(L.card), hex(L.rail), hex(L.line)
	if card then
		t.card, t.row, t.off = card, card:Lerp(tw, 0.06), card:Lerp(tw, 0.14)
		t.sCard, t.tCard = card, 1 - math.min(1, a + 0.04)
		t.sCtl, t.tCtl, t.sHover, t.tHover = t.row, 0, t.row, 0
		t.sPop, t.tPop, t.sNote, t.tNote = card, 0, card, 0
	end
	if rail then
		t.rail = rail
		t.sRail, t.tRail = rail, 1 - a
		if name ~= "Arcade" then t.sTop, t.tTop = rail, 1 - a end
	end
	if line then
		t.line, t.line2 = line, line:Lerp(tw, 0.1)
		t.ln, t.tln, t.ln2, t.tln2, t.frameLn, t.tframeLn = line, 0, t.line2, 0, line, 0
	end
	for k, v in pairs(st.tokens or {}) do t[k] = hex(v) or v end   -- a preset's own fixed tokens
	local ownBorder = hex(L.borderColor)
	t.shadeC = hex(L.shadeColor) or t[st.shadeTok or "accent"] or acc
	t.borderC = ownBorder or t[st.borderTok or "frameLn"] or t.frameLn
	t.tborderC = ownBorder and 0 or t["t" .. (st.borderTok or "frameLn")] or 0
	-- the top bar's own text colour (arcade's bar is the accent)
	t.topText = name == "Arcade" and t.onAccent or tx
	t.topDim = name == "Arcade" and t.onAccent or t.dim
	t.topFaint = name == "Arcade" and t.onAccent or t.faint
	self.tok = t
	return t
end

-- ------------------------------------------------------------ surfaces
-- A filled surface. cut = { tl, tr, bl, br } px corner cuts (no clip-path in
-- Roblox: the fill is built from strips plus a 45-degree square per cut,
-- whose outer edge IS the cut; "stairs" presets use 4 px pixel steps).
-- `group` puts the pieces in a CanvasGroup so overlaps don't double up the
-- transparency. Without a cut it's the instance's own background + UICorner;
-- `round` = { tl, tr, bl, br } booleans keeps some corners square.
function Win:_fill(inst, colorTok, transTok, cut, group, round, cornerFactor)
	local function setT(i, prop)
		if type(transTok) == "number" then i[prop] = transTok else self:paint(i, prop, transTok) end
	end
	if not cut and not round then
		self:paint(inst, "BackgroundColor3", colorTok)
		setT(inst, "BackgroundTransparency")
		inst.BorderSizePixel = 0
		self:corner(inst, cornerFactor or 1)
		return inst
	end
	inst.BackgroundTransparency = 1
	local holder
	if group or round then pcall(function() holder = make("CanvasGroup", { BackgroundTransparency = 1, BorderSizePixel = 0 }, nil) end) end
	local grouped = holder ~= nil
	holder = holder or make("Frame", { BackgroundTransparency = 1, BorderSizePixel = 0 })
	holder.Name = "Fill"
	holder.Size = UDim2.fromScale(1, 1)
	holder.ZIndex = 0
	holder.Parent = inst
	if grouped then setT(holder, "GroupTransparency") end
	local function piece(props)
		local p = make("Frame", props, holder)
		p.BorderSizePixel = 0
		self:paint(p, "BackgroundColor3", colorTok)
		if grouped then p.BackgroundTransparency = 0 else setT(p, "BackgroundTransparency") end
		return p
	end
	if not cut then
		-- rounded, but some corners square: a rounded piece plus a square
		-- filler per square corner (grouped, so the overlap doesn't show)
		local base = piece({ Size = UDim2.fromScale(1, 1) })
		self:corner(base, cornerFactor or 1)
		local spots = { { 0, 0 }, { 1, 0 }, { 0, 1 }, { 1, 1 } }
		for i, r in ipairs(round) do
			if not r then
				local sp = spots[i]
				local f = piece({ AnchorPoint = Vector2.new(sp[1], sp[2]), Position = UDim2.fromScale(sp[1], sp[2]) })
				self:paint(f, "Size", function()
					local r2 = math.floor((self.look.rad or 0) * (cornerFactor or 1) + 0.5)
					return UDim2.fromOffset(r2, r2)
				end)
			end
		end
		return holder
	end
	local tl, tr, bl, br = cut[1] or 0, cut[2] or 0, cut[3] or 0, cut[4] or 0
	if self.st.stairs then
		-- two 4 px steps per cut corner, strips that never overlap
		local topCut, botCut = (tl > 0 or tr > 0), (bl > 0 or br > 0)
		local function inset(c, n) return c > 0 and n or 0 end
		if topCut then
			piece({ Position = UDim2.fromOffset(inset(tl, 8), 0), Size = UDim2.new(1, -inset(tl, 8) - inset(tr, 8), 0, 4) })
			piece({ Position = UDim2.fromOffset(inset(tl, 4), 4), Size = UDim2.new(1, -inset(tl, 4) - inset(tr, 4), 0, 4) })
		end
		local y0, y1 = topCut and 8 or 0, botCut and 8 or 0
		piece({ Position = UDim2.fromOffset(0, y0), Size = UDim2.new(1, 0, 1, -y0 - y1) })
		if botCut then
			piece({ Position = UDim2.new(0, inset(bl, 4), 1, -8), Size = UDim2.new(1, -inset(bl, 4) - inset(br, 4), 0, 4) })
			piece({ Position = UDim2.new(0, inset(bl, 8), 1, -4), Size = UDim2.new(1, -inset(bl, 8) - inset(br, 8), 0, 4) })
		end
		return holder
	end
	local ctop, cbot = math.max(tl, tr), math.max(bl, br)
	if ctop > 0 then piece({ Position = UDim2.fromOffset(tl, 0), Size = UDim2.new(1, -tl - tr, 0, ctop) }) end
	piece({ Position = UDim2.fromOffset(0, ctop), Size = UDim2.new(1, 0, 1, -ctop - cbot) })
	if cbot > 0 then piece({ Position = UDim2.new(0, bl, 1, -cbot), Size = UDim2.new(1, -bl - br, 0, cbot) }) end
	local centres = { UDim2.fromOffset(tl, tl), UDim2.new(1, -tr, 0, tr), UDim2.new(0, bl, 1, -bl), UDim2.new(1, -br, 1, -br) }
	for i, c in ipairs({ tl, tr, bl, br }) do
		if c > 0 then
			piece({ AnchorPoint = Vector2.new(0.5, 0.5), Position = centres[i], Size = UDim2.fromOffset(c * math.sqrt(2), c * math.sqrt(2)), Rotation = 45 })
		end
	end
	return holder
end
-- the same cut grown by n px on every side (a shade that frames the window)
local function grownCut(cut, n)
	if not cut then return nil end
	return { (cut[1] > 0) and cut[1] + n or 0, (cut[2] > 0) and cut[2] + n or 0, (cut[3] > 0) and cut[3] + n or 0, (cut[4] > 0) and cut[4] + n or 0 }
end

-- ------------------------------------------------------------ look
local function cleanLook(src)
	local L = table.clone(LOOK_DEFAULTS)
	if type(src) ~= "table" then return L end
	for k, def in pairs(LOOK_DEFAULTS) do
		local v = src[k]
		if v ~= nil and type(v) == type(def) then L[k] = v end
	end
	if not PRESETS[L.preset] then L.preset = "Glass" end
	if not DENSITY[L.density] then L.density = "Cozy" end
	if not table.find(DOCKS, L.dock) then L.dock = "Center" end
	if not table.find(PATTERNS, L.pattern) then L.pattern = "None" end
	if not hex(L.bg) then L.bg = PRESETS[L.preset].look.bg end
	if not hex(L.text) then L.text = PRESETS[L.preset].look.text end
	if L.accent ~= "" and not hex(L.accent) then L.accent = "" end
	L.alpha = math.clamp(L.alpha, 20, 100)
	L.blur = math.clamp(L.blur, 0, 30)
	L.rad = math.clamp(L.rad, 0, 22)
	L.glow = math.clamp(L.glow, 0, 40)
	L.scale = math.clamp(L.scale, 50, 160)
	L.w = math.clamp(L.w, MIN_W, 1400)
	L.h = math.clamp(L.h, MIN_H, 900)
	L.strength = math.clamp(L.strength, 0, 100)
	-- the preset builder's values
	for _, k in ipairs({ "card", "rail", "line", "dim", "accent2", "shadeColor", "borderColor" }) do
		if L[k] ~= "" and not hex(L[k]) then L[k] = "" end
	end
	local function oneOf(k, list) if not table.find(list, L[k]) then L[k] = LOOK_DEFAULTS[k] end end
	oneOf("caps", { "Preset", "Upper", "Normal" })
	oneOf("steps", { "Preset", "On", "Off" })
	oneOf("shade", { "Preset", "None", "Offset", "Rim" })
	oneOf("cardBar", { "Preset", "On", "Off" })
	oneOf("cardShadow", { "Preset", "On", "Off" })
	oneOf("switch", { "Preset", "Round", "Square" })
	oneOf("fit", { "Crop", "Fit", "Stretch", "Tile" })
	oneOf("prefix", { "Preset", "None", "//", ">", "#", "+", "~", "::", "*" })
	oneOf("h4c", { "Preset", "Text", "Accent", "Second accent", "Muted" })
	oneOf("mark", MARKS)
	if L.deco ~= "Preset" and L.deco ~= "None" and not DECOS[L.deco] then L.deco = "Preset" end
	if L.head ~= "" and not FONT_WEIGHTS[family(L.head)] then L.head = "" end
	if L.titleFont ~= "" and not FONT_WEIGHTS[family(L.titleFont)] then L.titleFont = "" end
	L.title = string.sub((string.gsub(L.title, "[^\32-\126]", "")), 1, 18)
	L.name = string.sub((string.gsub(L.name, "[^%w %-_]", "")), 1, 24)
	L.titleSize = math.clamp(math.floor(L.titleSize), -1, 48)
	L.headSize = math.clamp(math.floor(L.headSize), -4, 8)
	for _, k in ipairs({ "cutTL", "cutTR", "cutBL", "cutBR" }) do L[k] = math.clamp(math.floor(L[k]), -1, 40) end
	L.shadeSize = math.clamp(math.floor(L.shadeSize), 0, 20)
	L.border = math.clamp(math.floor(L.border), -1, 4)
	L.cardCut = math.clamp(math.floor(L.cardCut), -1, 24)
	L.topH = (L.topH < 0) and -1 or math.clamp(math.floor(L.topH), 40, 90)
	L.railW = (L.railW < 0) and -1 or math.clamp(math.floor(L.railW), 120, 260)
	L.decoAlpha = math.clamp(L.decoAlpha, 0, 100)
	L.speed = math.clamp(L.speed, 50, 300)
	L.volume = math.clamp(L.volume, 0, 100)
	return L
end

-- ------------------------------------------------------------ structure
-- The structure in force: the preset (a character preset on top of its base
-- style), then whatever the preset builder overrides. Returns it with the
-- name of the BASE style - the chrome's per-style branches go by that.
local function resolveStyle(L)
	local def = PRESETS[L.preset] or PRESETS.Glass
	local base = def.base and PRESETS[def.base] or nil
	local st = {}
	if base then for k, v in pairs(base) do st[k] = v end end
	for k, v in pairs(def) do st[k] = v end
	for _, k in ipairs({ "nav", "tab", "h4", "btn", "brand" }) do st[k] = table.clone(st[k]) end
	if L.head ~= "" then
		st.head, st.headW = family(L.head), Enum.FontWeight.Bold
		st.brand.font, st.brand.weight = nil, nil
	end
	if L.titleFont ~= "" then st.brand.font, st.brand.weight = family(L.titleFont), Enum.FontWeight.Bold end
	if L.titleSize >= 0 then st.brand.size = math.max(8, L.titleSize) end
	if L.caps ~= "Preset" then
		local up = L.caps == "Upper"
		st.nav.upper, st.tab.upper, st.h4.upper, st.btn.upper = up, up, up, up
	end
	if L.headSize ~= 0 then
		for _, k in ipairs({ "nav", "tab", "h4", "btn" }) do st[k].size = math.clamp(st[k].size + L.headSize, 8, 30) end
	end
	if L.prefix == "None" then st.h4.prefix = nil elseif L.prefix ~= "Preset" then st.h4.prefix = L.prefix .. " " end
	if L.h4c ~= "Preset" then st.h4.tok = ({ Text = "text", Accent = "accent", ["Second accent"] = "accent2", Muted = "dim" })[L.h4c] end
	if L.cutTL >= 0 or L.cutTR >= 0 or L.cutBL >= 0 or L.cutBR >= 0 then
		local c = st.cut or { 0, 0, 0, 0 }
		c = { L.cutTL >= 0 and L.cutTL or c[1], L.cutTR >= 0 and L.cutTR or c[2], L.cutBL >= 0 and L.cutBL or c[3], L.cutBR >= 0 and L.cutBR or c[4] }
		st.cut = (c[1] + c[2] + c[3] + c[4] > 0) and c or nil
	end
	if L.steps ~= "Preset" then st.stairs = L.steps == "On" end
	-- the shade and the border always paint from shadeC / borderC (which fall
	-- back to the style's own token), so their two colours change live
	st.shadeTok = st.shade and st.shade.tok or "accent"
	if L.shade == "None" then st.shade = nil
	elseif L.shade == "Offset" then st.shade = { x = L.shadeSize, y = L.shadeSize }
	elseif L.shade == "Rim" then st.shade = { grow = math.max(1, L.shadeSize) }
	elseif st.shade then st.shade = table.clone(st.shade) end
	if st.shade then st.shade.tok = "shadeC" end
	st.borderTok = st.winStroke and st.winStroke.tok or "frameLn"
	local thick = st.winStroke and st.winStroke.thick or 1
	if L.border == 0 then st.noStroke = true elseif L.border > 0 then st.noStroke, thick = false, L.border end
	st.winStroke = { tok = "borderC", thick = thick }
	-- a cut window can't carry a UIStroke (it would outline the uncut rectangle)
	if st.cut then st.noStroke = true end
	if L.cardCut >= 0 then st.cardCut = L.cardCut > 0 and { 0, L.cardCut, 0, 0 } or nil end
	if L.cardBar ~= "Preset" then st.cardBar = L.cardBar == "On" end
	if L.cardShadow ~= "Preset" then st.cardShadow = L.cardShadow == "On" and "shadeC" or nil end
	if L.switch ~= "Preset" then st.pill, st.sharp = L.switch == "Round", L.switch == "Square" end
	if L.topH >= 0 then st.topH = L.topH end
	if L.railW >= 0 then st.railW = L.railW end
	if L.deco == "None" then st.deco = nil elseif L.deco ~= "Preset" then st.deco = L.deco end
	return st, def.base or L.preset
end
function Win:_resolve()
	self.st, self.presetName = resolveStyle(self.look)
end

-- ------------------------------------------------------------ constructor
function TeekUI.new(opts)
	opts = opts or {}
	local self = setmetatable({}, Win)
	self.title = opts.Title or "Tempo Hub"
	self.subtitle = opts.Subtitle or "Universal"
	self.version = opts.Version or "v6"
	self.theme = "Tempo"
	self.painted, self.conns = {}, {}
	self.tabs, self.tabList, self.rail = {}, {}, {}
	self.index = {}   -- search: { label, tab, item }
	self.notifySeconds = 4
	self.toggleKey = DEFAULT_MENU_KEY
	local g = thisGame()
	self.gameAccent = hex(opts.Accent) or g.accent or DEFAULT_ACCENT
	self.artUrl = opts.Art or g.art
	-- the look: defaults, the hub's preset, then what the player saved (the
	-- config is autoloaded before the menu is built, so read it now and skip a
	-- rebuild when mountInterface hands over the same table)
	local seed = { preset = PRESETS[opts.Preset or ""] and opts.Preset or "Haunted" }
	for k, v in pairs(PRESETS[seed.preset].look) do seed[k] = v end
	pcall(function()
		local tc = getgenv and getgenv().TeekConfig
		local k = tc and tc.Theme and tc.Theme.ToggleKey
		if k and Enum.KeyCode[k] and k ~= "K" then self.toggleKey = k end
		local saved = tc and tc.Theme and tc.Theme.Look
		if type(saved) == "table" then for kk, v in pairs(saved) do seed[kk] = v end end
	end)
	-- "Same look in every game": the shared look wins over this hub's own
	pcall(function()
		local S = sharedLoad()
		if S.global and type(S.look) == "table" then for kk, v in pairs(S.look) do seed[kk] = v end end
	end)
	self.look = cleanLook(seed)
	self.undo = {}
	self:_resolve()
	self.uiScale = self.look.scale / 100
	self:_tokens()
	pcall(function() if getgenv and getgenv().TempoDev then getgenv().TempoUI = self end end)

	self.gui = screenGui("Tempo_Menu", 2 ^ 31 - 3)
	self.overlay = screenGui("Tempo_Overlay", 2 ^ 31 - 2)
	self.scale = make("UIScale", { Scale = self.uiScale }, self.gui)
	-- TeekConfig.applyTheme takes the window path (SetTheme / win:SetUIScale)
	-- when .win is set, instead of multiplying the scale a second time
	self.win = self
	self.overlayScale = make("UIScale", { Scale = self.uiScale }, self.overlay)
	pcall(function() self.click = make("Sound", { SoundId = CLICK_SOUND, Volume = 0.25 }, self.overlay) end)

	-- notification stack (overlay, bottom right) - survives rebuilds
	local stack = make("Frame", { Name = "Notes", AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -18, 1, -18),
		Size = UDim2.fromOffset(300, 400), BackgroundTransparency = 1 }, self.overlay)
	make("UIListLayout", { Padding = UDim.new(0, 8), VerticalAlignment = Enum.VerticalAlignment.Bottom, HorizontalAlignment = Enum.HorizontalAlignment.Right,
		SortOrder = Enum.SortOrder.LayoutOrder }, stack)
	self.notes = stack

	self:_build()
	self:_keys()
	if IS_MOBILE then self:_mobileButton() end

	-- built closed: Boot opens it; a script that never boots opens on its own
	self.root.Visible = false
	task.delay(1.5, function()
		if not self.dead and not self.opened and not self.booted then self:Show() end
	end)
	return self
end

-- ------------------------------------------------------------ chrome
-- build-scoped connections: dropped on a preset rebuild
function Win:bconn(c)
	self.bconns[#self.bconns + 1] = c
	return c
end

-- a frame with a hard offset copy behind it (the mock's box-shadow: N N 0)
-- returns wrapper, face; the face is what gets the content
function Win:_shadowed(parent, props, shadowTok, dx, dy)
	local wrap = make("Frame", { BackgroundTransparency = 1, BorderSizePixel = 0 }, parent)
	for k, v in pairs(props or {}) do wrap[k] = v end
	local sh = make("Frame", { Name = "Shadow", Position = UDim2.fromOffset(dx, dy), Size = UDim2.fromScale(1, 1), BorderSizePixel = 0, ZIndex = 0 }, wrap)
	self:paint(sh, "BackgroundColor3", shadowTok)
	local face = make("Frame", { Name = "Face", Size = UDim2.fromScale(1, 1), BorderSizePixel = 0, ZIndex = 1 }, wrap)
	return wrap, face, sh
end

function Win:_place(keep)
	local L, st = self.look, self.st
	local cam = workspace.CurrentCamera
	local vp = cam and cam.ViewportSize or Vector2.new(1920, 1080)
	local sc = self.uiScale
	local W, H = L.w, self.minimized and st.topH or L.h
	local x, y
	if L.dock ~= "Free" then
		local topMargin = st.pop and 130 or 30
		if L.dock == "Left" then x = 24
		elseif L.dock == "Right" then x = vp.X - L.w * sc - 24
		else x = (vp.X - L.w * sc) / 2 end
		y = math.max(topMargin, (vp.Y - L.h * sc) / 2 + (st.pop and 40 or 0))
		x, y = x / sc, y / sc
	elseif keep then
		x, y = keep.X.Offset, keep.Y.Offset
	elseif L.remember and (L.x ~= 0 or L.y ~= 0) then
		x = math.clamp(L.x, 0, math.max(0, vp.X / sc - 80))
		y = math.clamp(L.y, 0, math.max(0, vp.Y / sc - 40))
	else
		x, y = (vp.X / sc - L.w) / 2, (vp.Y / sc - L.h) / 2
	end
	self.root.Position = UDim2.fromOffset(math.floor(x), math.floor(y))
	self.root.Size = UDim2.fromOffset(W, H)
end

function Win:_build(keepPos)
	local st, L = self.st, self.look
	local name = self.presetName
	local topH = st.topH
	self.bconns = {}
	local root = make("Frame", { Name = "Window", Size = UDim2.fromOffset(L.w, L.h), BackgroundTransparency = 1, BorderSizePixel = 0 }, self.gui)
	self.root = root
	self:_place(keepPos)

	-- shade: anime / manga = hard offset copy, gamer / arcade = an accent rim.
	-- Glow grows it; without a shade, glow is an accent stroke round the window.
	if st.shade then
		local sh = st.shade
		local f = make("Frame", { Name = "Shade", BackgroundTransparency = 1, ZIndex = 0 }, root)
		self:paint(f, "Position", function()
			local g = (sh.grow or 0) + math.floor((self.look.glow or 0) / 5)
			return UDim2.fromOffset((sh.x or 0) - g, (sh.y or 0) - g)
		end)
		self:paint(f, "Size", function()
			local g = (sh.grow or 0) + math.floor((self.look.glow or 0) / 5)
			return UDim2.new(1, g * 2, 1, g * 2)
		end)
		self:_fill(f, sh.tok, sh.t or 0, grownCut(st.cut, sh.grow or 0), true)
	else
		self:corner(root, 1)
		local gs = make("UIStroke", { ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, root)
		self.glowStroke = gs
		self:paint(gs, "Color", "accent")
		self:paint(gs, "Thickness", function() return 1 + (self.look.glow or 0) / 5 end)
		self:paint(gs, "Transparency", function() return 1 - math.min(1, (self.look.glow or 0) / 28) * 0.7 end)
		self:paint(gs, "Enabled", function() return (self.look.glow or 0) > 0 end)
	end

	local frame = make("Frame", { Name = "Frame", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0, ClipsDescendants = true, ZIndex = 1 }, root)
	self.frame = frame
	self:_fill(frame, "sWin", "tWin", st.cut, true)
	if not st.noStroke then
		local ws = st.winStroke
		self:stroke(frame, ws and ws.tok or "frameLn", ws and ws.thick or 1)
	end
	self.patLayer = make("Frame", { Name = "Pattern", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ClipsDescendants = true, ZIndex = 0 }, frame)
	self:_pattern()
	if st.gold then
		-- grimoire: inner rule + corner brackets
		local inner = make("Frame", { Position = UDim2.fromOffset(5, 5), Size = UDim2.new(1, -10, 1, -10), BackgroundTransparency = 1, ZIndex = 0 }, frame)
		local s = make("UIStroke", { Thickness = 1, Transparency = 0.7 }, inner)
		self:paint(s, "Color", "gold")
		for _, c in ipairs({ { 0, 0 }, { 1, 0 }, { 0, 1 }, { 1, 1 } }) do
			for _, sz in ipairs({ { 22, 1 }, { 1, 22 } }) do
				local b = make("Frame", { AnchorPoint = Vector2.new(c[1], c[2]), Position = UDim2.new(c[1], c[1] == 0 and 10 or -10, c[2], c[2] == 0 and 10 or -10),
					Size = UDim2.fromOffset(sz[1], sz[2]), BorderSizePixel = 0, ZIndex = 0 }, frame)
				self:paint(b, "BackgroundColor3", "gold")
			end
		end
	end

	-- ---------------------------------------------------------- top bar
	local top = make("Frame", { Name = "Top", Size = UDim2.new(1, 0, 0, topH), BackgroundTransparency = 1, ClipsDescendants = true }, frame)
	self.top = top
	if st.cut then self:_fill(top, "sTop", "tTop", { st.cut[1], st.cut[2], 0, 0 }, true)
	else self:_fill(top, "sTop", "tTop", nil, false, { true, true, false, false }) end
	if st.topRule then
		local r = st.topRule
		local bar = make("Frame", { Position = UDim2.new(0, 0, 1, -r.h), Size = UDim2.new(1, 0, 0, r.h), BorderSizePixel = 0 }, top)
		self:paint(bar, "BackgroundColor3", r.tok)
	elseif name == "Anime" then
		local bar = make("Frame", { Position = UDim2.new(0, 0, 1, -3), Size = UDim2.new(1, -6, 0, 3), BorderSizePixel = 0 }, top)
		self:paint(bar, "BackgroundColor3", "accent")
	elseif name ~= "Arcade" then
		local th = name == "Manga" and 3 or 1
		local rule = make("Frame", { Position = UDim2.new(0, 0, 1, -th), Size = UDim2.new(1, 0, 0, th), BorderSizePixel = 0 }, top)
		self:paint(rule, "BackgroundColor3", "ln")
		self:paint(rule, "BackgroundTransparency", "tln")
	end

	local padL = (name == "Grimoire" and 26) or (name == "Anime" and 18) or 16
	local left = make("Frame", { Name = "Left", BackgroundTransparency = 1, Position = UDim2.fromOffset(padL, 0), Size = UDim2.new(1, -padL - 84, 1, 0), ZIndex = 2 }, top)
	make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, VerticalAlignment = Enum.VerticalAlignment.Center, Padding = UDim.new(0, 10),
		SortOrder = Enum.SortOrder.LayoutOrder }, left)
	-- the logo: Tempo's, or the builder's own image ("none" hides it)
	local logo
	local ownLogo = string.match(L.logo, "%d+")
	if ownLogo then logo = assetImage(ownLogo)
	elseif string.lower(L.logo) ~= "none" then
		local t = self.tok
		local under = (t.tTop or 0) >= 0.9 and t.bg or t.sTop   -- what the mark sits on (a see-through bar = the window)
		-- Spooky wears its white flame unless a mark was picked on purpose
		if st.spooky and L.mark == "Auto" then logo = resolveDeco("spooky_icon") end
		logo = logo or resolveLogo(L.mark, lum(under) > 0.5)
	end
	if logo then
		local flameMark = st.spooky and L.mark == "Auto" and not ownLogo
		local li = make("ImageLabel", { Image = logo, BackgroundTransparency = 1, Size = flameMark and UDim2.fromOffset(16, 21) or UDim2.fromOffset(22, 22), LayoutOrder = 1 }, left)
		self.logoLabel = li
		if st.pixel then pcall(function() li.ResampleMode = Enum.ResamplerMode.Pixelated end) end
	end
	-- wordmark (manga / arcade: with a hard text shadow)
	local b = st.brand
	local title = L.title ~= "" and L.title or "TEMPO"   -- (the builder's Title text)
	local word = b.spaced and spaced(title) or title
	local bw = make("Frame", { BackgroundTransparency = 1, Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, LayoutOrder = 2 }, left)
	local bshadow = (name == "Manga" and "accent") or (name == "Arcade" and "black") or b.shadow or nil
	if bshadow then
		local s = self:text({ Text = word, Position = UDim2.fromOffset(3, 3), Size = UDim2.fromScale(1, 1), ZIndex = 1 }, bw, bshadow, "brand", b.size)
		if name == "Arcade" then s.TextTransparency = 0.7 end
	end
	self.brandLabel = self:text({ Text = word, Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, ZIndex = 2 }, bw, b.tok or "topText", "brand", b.size)
	-- the slash between wordmark and game
	if name ~= "Arcade" then
		local sl = ({ Anime = { 4, 34, "text", 22 }, Gamer = { 1, 18, "accent", 20 }, Manga = { 3, 30, "text", 20 }, Grimoire = { 7, 7, "gold", 45 } })[name] or { 1, 18, "faint", 20 }
		local slash = make("Frame", { Size = UDim2.fromOffset(sl[1], sl[2]), Rotation = sl[4], BorderSizePixel = 0, LayoutOrder = 3 }, left)
		self:paint(slash, "BackgroundColor3", sl[3])
	end
	-- game name
	local gameName = self.subtitle
	if name == "Anime" then
		local wrap, face = self:_shadowed(left, { Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, LayoutOrder = 4 }, "text", 4, 4)
		face.Size = UDim2.new()
		face.AutomaticSize = Enum.AutomaticSize.XY
		self:paint(face, "BackgroundColor3", "accent")
		pad(face, 16, 16, 5, 5)
		self:text({ Text = string.upper(gameName), Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY }, face, "onAccent", "head", 15)
		wrap.Size = UDim2.new()
	elseif name == "Manga" then
		local pill = make("Frame", { Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, BorderSizePixel = 0, LayoutOrder = 4 }, left)
		self:paint(pill, "BackgroundColor3", "paper")
		make("UICorner", { CornerRadius = UDim.new(0, 18) }, pill)
		self:stroke(pill, "ink", 2)
		pad(pill, 14, 14, 4, 4)
		self:text({ Text = string.upper(gameName), Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY }, pill, "ink", cachedFont("RobotoCondensed", Enum.FontWeight.Bold), 13)
	elseif name == "Gamer" then
		self:text({ Text = string.upper(gameName), Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, LayoutOrder = 4 }, left, "accent", "mono", 12)
		local dot = make("Frame", { Size = UDim2.fromOffset(6, 6), BorderSizePixel = 0, LayoutOrder = 5 }, left)
		self:paint(dot, "BackgroundColor3", "good")
		make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, dot)
	elseif name == "Arcade" then
		self:text({ Text = "1P  " .. string.upper(gameName), Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, LayoutOrder = 4 }, left, "topText", "head", 9)
	elseif name == "Grimoire" then
		self:text({ Text = gameName, Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, LayoutOrder = 4 }, left, "dim", cachedFont("Merriweather", Enum.FontWeight.Regular, Enum.FontStyle.Italic), 14)
	elseif st.spooky then
		self:text({ Text = spaced(string.upper(gameName)), Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, LayoutOrder = 4 }, left, "topDim", "head", 12)
	else
		self:text({ Text = gameName, Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, LayoutOrder = 4 }, left, "topDim", "body", 13)
	end
	-- version chip
	local vs = ({ Anime = { "text", 1, "head" }, Manga = { "ink", 2, "mono" }, Arcade = { "topText", 2, "head" }, Grimoire = { "gold", 1, "head" } })[name] or { "ln2", 1, "mono" }
	local ver = make("Frame", { Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY, BackgroundTransparency = 1, LayoutOrder = 6 }, left)
	make("UICorner", { CornerRadius = UDim.new(0, (name == "Glass" or name == "Sleek") and 4 or (name == "Grimoire" and 2 or 0)) }, ver)
	local vst = self:stroke(ver, vs[1], vs[2])
	if name == "Grimoire" then vst.Transparency = 0.45 end
	pad(ver, name == "Arcade" and 5 or 6, nil, name == "Arcade" and 4 or 1, nil)
	self:text({ Text = self.version, Size = UDim2.new(), AutomaticSize = Enum.AutomaticSize.XY }, ver, (vs[1] == "ln2") and "topDim" or vs[1], vs[3],
		(name == "Arcade" and 8) or (name == "Grimoire" and 13) or 11)

	local function topButton(glyph, xoff, onClick)
		local bt = make("TextButton", { Text = glyph, AutoButtonColor = false, BackgroundTransparency = 1, Size = UDim2.fromOffset(30, 30),
			AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, xoff, 0.5, 0), TextSize = 17, ZIndex = 3 }, top)
		self:paint(bt, "FontFace", function() return self:_face("body") end)
		self:paint(bt, "TextColor3", "topFaint")
		self:paint(bt, "BackgroundColor3", name == "Arcade" and "black" or "sHover")
		self:corner(bt, 0.4)
		bt.MouseEnter:Connect(function()
			bt.TextColor3 = self.tok.topText
			bt.BackgroundTransparency = name == "Arcade" and 0.8 or self.tok.tHover
		end)
		bt.MouseLeave:Connect(function()
			bt.TextColor3 = self.tok.topFaint
			bt.BackgroundTransparency = 1
		end)
		bt.MouseButton1Click:Connect(onClick)
		return bt
	end
	-- glyphs as escapes: the build reads sources as ANSI and mangles raw UTF-8
	topButton("\u{00D7}", -8, function() self:Hide() end)
	topButton("\u{2013}", -40, function() self:_minimise(not self.minimized) end)
	self:_art(top, root)
	self:_deco(top, root)

	-- drag by the top bar
	local dragging, start, startPos
	top.InputBegan:Connect(function(i)
		if self.growing then return end
		if isClick(i) then
			dragging, start, startPos = true, i.Position, root.Position
		end
	end)
	self:bconn(UserInputService.InputChanged:Connect(function(i)
		if dragging and isMove(i) then
			local d = (i.Position - start) / self.uiScale
			if d.Magnitude > 2 and self.look.dock ~= "Free" then self:_setLook("dock", "Free") end
			root.Position = UDim2.fromOffset(startPos.X.Offset + d.X, startPos.Y.Offset + d.Y)
		end
	end))
	self:bconn(UserInputService.InputEnded:Connect(function(i)
		if dragging and isClick(i) then
			dragging = false
			self.look.x, self.look.y = root.Position.X.Offset, root.Position.Y.Offset
			self:_save()
			self:_syncInterface()
		end
	end))

	-- ---------------------------------------------------------- body
	local body = make("Frame", { Name = "Body", Position = UDim2.fromOffset(0, topH), Size = UDim2.new(1, 0, 1, -topH), BackgroundTransparency = 1 }, frame)
	self.body = body
	local railW = st.railW or 156
	local railBox = make("Frame", { Name = "Rail", Size = UDim2.new(0, railW, 1, 0), BackgroundTransparency = 1 }, body)
	if st.cut then self:_fill(railBox, "sRail", "tRail", { 0, 0, st.cut[3], 0 }, true)
	else self:_fill(railBox, "sRail", "tRail", nil, false, { false, false, true, false }) end
	local ruleW = (name == "Anime" and 0) or (name == "Manga" and 3) or 1
	if ruleW > 0 then
		local r = make("Frame", { Position = UDim2.fromOffset(railW, 0), Size = UDim2.new(0, ruleW, 1, 0), BorderSizePixel = 0 }, body)
		self:paint(r, "BackgroundColor3", "ln")
		self:paint(r, "BackgroundTransparency", "tln")
	end

	-- search lives at the top of the rail
	local sbox = make("Frame", { Position = UDim2.fromOffset(8, 10), Size = UDim2.new(1, -16, 0, 30), BorderSizePixel = 0 }, railBox)
	self:_fill(sbox, "sCtl", "tCtl", nil, false, nil, 0.6)
	local sst = self:stroke(sbox, "ln2", (name == "Manga" or name == "Arcade") and 2 or 1)
	local sin = make("TextBox", { Text = "", PlaceholderText = "Search", ClearTextOnFocus = false, BackgroundTransparency = 1,
		Size = UDim2.new(1, -20, 1, 0), Position = UDim2.fromOffset(10, 0), TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left }, sbox)
	self:paint(sin, "FontFace", function() return self:_face(name == "Grimoire" and "italic" or (name == "Arcade" and "mono" or "body")) end)
	self:paint(sin, "TextColor3", "ctlText")
	self:paint(sin, "PlaceholderColor3", "faint")
	sin.Focused:Connect(function() sst.Color = self.tok.accent end)
	sin.FocusLost:Connect(function() sst.Color = self.tok.ln2 end)
	self.searchBox = sin
	sin:GetPropertyChangedSignal("Text"):Connect(function() self:_search(sin.Text) end)

	local list = make("ScrollingFrame", { Name = "Entries", Position = UDim2.fromOffset(0, 48), Size = UDim2.new(1, 0, 1, -48 - 50),
		BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 0, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y }, railBox)
	make("UIListLayout", { Padding = UDim.new(0, 2), SortOrder = Enum.SortOrder.LayoutOrder }, list)
	pad(list, name == "Grimoire" and 14 or 8, 8, 0, 6)
	self.railFrame = list

	-- who's playing
	local who = make("Frame", { Name = "Who", AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 8, 1, -4), Size = UDim2.new(1, -16, 0, 44), BackgroundTransparency = 1 }, railBox)
	local wr = make("Frame", { Size = UDim2.new(1, 0, 0, (name == "Anime" and 2) or (name == "Manga" and 3) or 1), BorderSizePixel = 0 }, who)
	if name == "Anime" then self:paint(wr, "BackgroundColor3", "accent")
	else
		self:paint(wr, "BackgroundColor3", "ln")
		self:paint(wr, "BackgroundTransparency", "tln")
	end
	local pname = "player"
	pcall(function() pname = LocalPlayer.DisplayName end)
	self:text({ Text = pname, Position = UDim2.fromOffset(2, 8), Size = UDim2.new(1, -4, 0, 16), TextTruncate = Enum.TextTruncate.AtEnd }, who, "text",
		name == "Arcade" and "head" or "semi", name == "Arcade" and 9 or 12)
	self:text({ Text = string.lower(self.title) .. " \u{00B7} " .. self.version, Position = UDim2.fromOffset(2, 25), Size = UDim2.new(1, -4, 0, 14),
		TextTruncate = Enum.TextTruncate.AtEnd }, who, "faint", "mono", 11)

	-- tab strip + content
	local mainX = railW + ruleW
	local main = make("Frame", { Name = "Main", Position = UDim2.fromOffset(mainX, 0), Size = UDim2.new(1, -mainX, 1, 0), BackgroundTransparency = 1 }, body)
	self.main = main
	local strip = make("Frame", { Name = "Strip", Size = UDim2.new(1, 0, 0, 42), BackgroundTransparency = 1 }, main)
	make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 20), VerticalAlignment = Enum.VerticalAlignment.Center,
		SortOrder = Enum.SortOrder.LayoutOrder }, strip)
	pad(strip, 18, 18, 0, 0)
	local sth = (name == "Anime" and 2) or (name == "Manga" and 3) or 1
	local srule = make("Frame", { Position = UDim2.fromOffset(0, 42), Size = UDim2.new(1, 0, 0, sth), BorderSizePixel = 0 }, main)
	self:paint(srule, "BackgroundColor3", "ln")
	self:paint(srule, "BackgroundTransparency", "tln")
	self.strip = strip
	self.pages = make("Frame", { Name = "Pages", Position = UDim2.fromOffset(0, 42 + sth), Size = UDim2.new(1, 0, 1, -42 - sth), BackgroundTransparency = 1, ClipsDescendants = true }, main)

	-- resize grip
	local grip = make("TextButton", { Text = "", AutoButtonColor = false, BackgroundTransparency = 1,
		Size = UDim2.fromOffset(16, 16), Position = UDim2.new(1, -16, 1, -16), ZIndex = 5 }, frame)
	for _, sz in ipairs({ { 7, 2 }, { 2, 7 } }) do
		local m = make("Frame", { AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -4, 1, -4), Size = UDim2.fromOffset(sz[1], sz[2]), BorderSizePixel = 0,
			BackgroundTransparency = name == "Manga" and 0 or 0.4, ZIndex = 5 }, grip)
		self:paint(m, "BackgroundColor3", "faint")
	end
	self.grip = grip
	if st.haunt then self:_haunt() end
	if st.spooky then self:_spooky() end
	local rs, rstart, rsize
	grip.InputBegan:Connect(function(i)
		if self.growing then return end
		if isClick(i) then rs, rstart, rsize = true, i.Position, root.Size end
	end)
	self:bconn(UserInputService.InputChanged:Connect(function(i)
		if rs and isMove(i) then
			local d = (i.Position - rstart) / self.uiScale
			root.Size = UDim2.fromOffset(math.max(MIN_W, rsize.X.Offset + d.X), math.max(MIN_H, rsize.Y.Offset + d.Y))
			if self.current then self.current:_flow() end
		end
	end))
	self:bconn(UserInputService.InputEnded:Connect(function(i)
		if rs and isClick(i) then
			rs = false
			self.look.w, self.look.h = root.Size.X.Offset, root.Size.Y.Offset
			if self.look.dock == "Center" then self.look.dock = "Free" end
			self:_save()
			self:_syncInterface()
		end
	end))
	if self.minimized then self:_minimise(true) end
end

function Win:_minimise(on)
	self.minimized = on and true or false
	if self.body then self.body.Visible = not self.minimized end
	if self.grip then self.grip.Visible = not self.minimized end
	if self.root then
		local h = self.minimized and self.st.topH or self.look.h
		self.root.Size = UDim2.fromOffset(self.root.Size.X.Offset, h)
	end
	self:_closePopover()
end

-- header art: per game, from a URL, cached to disk; missing = no art
function Win:_art(top, root)
	local L = self.look
	if string.lower(L.art) == "none" then return end
	local own = string.match(L.art, "%d+")   -- the builder's own image
	local url = self.artUrl
	-- a character preset's emblem sits where the game's art would
	if not own and (not url or self.st.deco) then return end
	local st = self.st
	local imgs = {}
	local function art(props, parent)
		local i = make("ImageLabel", { BackgroundTransparency = 1, ScaleType = Enum.ScaleType.Fit, Visible = false }, parent)
		for k, v in pairs(props) do i[k] = v end
		if st.pixel then pcall(function() i.ResampleMode = Enum.ResamplerMode.Pixelated end) end
		imgs[#imgs + 1] = i
		return i
	end
	if st.cameo then
		-- grimoire: a round cameo with a gold rim and an outer hairline
		local ring = make("Frame", { AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -80, 0, 3), Size = UDim2.fromOffset(58, 58), BackgroundTransparency = 1, Visible = false, ZIndex = 2 }, top)
		make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, ring)
		local rs = make("UIStroke", { Transparency = 0.6 }, ring)
		self:paint(rs, "Color", "gold")
		local c = art({ AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(50, 50), BackgroundTransparency = 0.7,
			BackgroundColor3 = BLACK, ScaleType = Enum.ScaleType.Crop, ZIndex = 2 }, ring)
		make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, c)
		local cs = make("UIStroke", { Thickness = 1 }, c)
		self:paint(cs, "Color", "gold")
		imgs[#imgs + 1] = ring
	elseif st.pop then
		-- sticks out above the window (anime / manga / arcade)
		local size = st.pixel and 150 or 178
		local y = st.topH + (st.pixel and 0 or 3)
		if not st.pixel then
			local echo = art({ AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -74 + 9, 0, y + 4), Size = UDim2.fromOffset(size, size), ZIndex = 2 }, root)
			self:paint(echo, "ImageColor3", "accent")
		end
		art({ AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, -74, 0, y), Size = UDim2.fromOffset(size, size), ZIndex = 3 }, root)
	else
		local a = art({ AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -70, 0, -14), Size = UDim2.fromOffset(118, 118), ImageTransparency = 0.25, ZIndex = 1 }, top)
		make("UIGradient", { Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.45, 0), NumberSequenceKeypoint.new(1, 0) }) }, a)
	end
	task.spawn(function()
		local asset = own and assetImage(own) or resolveArt(url)
		if not asset then return end
		for _, i in ipairs(imgs) do
			if i.Parent then
				if i:IsA("ImageLabel") then i.Image = asset end
				i.Visible = true
			end
		end
	end)
end

-- a preset's decorations (DECOS): emblems round the window, bands on the top bar
function Win:_deco(top, root)
	local set = self.st.deco and DECOS[self.st.deco]
	if not set then return end
	local s = math.clamp((tonumber(self.look.decoAlpha) or 100) / 100, 0, 1)
	if s <= 0 then return end
	local imgs = {}
	-- the marks sit in the top bar's empty right half (where a game's header
	-- art goes), under its text and buttons, cropped by the bar
	local mark = make("Frame", { Name = "Marks", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ClipsDescendants = true, ZIndex = 1 }, top)
	for _, d in ipairs(set) do
		if d.img then
			local a, p, sz = d.anchor, d.pos, d.size
			local i = make("ImageLabel", { Name = "Deco", BackgroundTransparency = 1, Visible = false, AnchorPoint = Vector2.new(a[1], a[2]),
				Position = UDim2.new(p[1], p[2], p[3], p[4]), Size = UDim2.new(sz[1], sz[2], sz[3], sz[4]), Rotation = d.rot or 0,
				ZIndex = d.tint and 0 or (d.z or 4), ScaleType = d.fit == "Stretch" and Enum.ScaleType.Stretch or Enum.ScaleType.Fit,
				ImageTransparency = 1 - (d.a or 1) * s }, d.tint and mark or root)
			if d.tint then self:paint(i, "ImageColor3", d.tint) end
			if d.fade then
				make("UIGradient", { Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.07, 0),
					NumberSequenceKeypoint.new(0.93, 0), NumberSequenceKeypoint.new(1, 1) }) }, i)
			end
			imgs[#imgs + 1] = { i, d.img }
		elseif d.stripes then
			local band = make("Frame", { Name = "Stripes", AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 0, 1, 0), Size = UDim2.new(1, 0, 0, d.h),
				BackgroundTransparency = 1, ClipsDescendants = true, ZIndex = 2 }, top)
			make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, SortOrder = Enum.SortOrder.LayoutOrder }, band)
			for n = 1, 112 do
				local f = make("Frame", { Size = UDim2.new(0, d.w, 1, 0), BorderSizePixel = 0, BackgroundTransparency = 1 - s, LayoutOrder = n }, band)
				self:paint(f, "BackgroundColor3", d.stripes[(n % 2) + 1])
			end
		end
	end
	if #imgs == 0 then return end
	task.spawn(function()
		for _, it in ipairs(imgs) do
			local asset = resolveDeco(it[2])
			if asset and it[1].Parent then
				it[1].Image = asset
				it[1].Visible = true
			end
		end
	end)
end

-- ============================================================
--  HAUNTED: the moving parts (teek 10-13). Built with the chrome (so a
--  preset rebuild drops it), one Heartbeat loop that sleeps while the menu
--  is hidden, and nothing moves with Animations off. Sprites are
--  assets/deco/haunt_*.png (make_haunt_art.py); a sprite that won't load
--  just isn't drawn.
-- ============================================================
-- a sprite that appears once its image is in (fetched + cached like the decos)
function Win:_hauntSprite(name, props, parent, button, onLoad)
	local i = make(button and "ImageButton" or "ImageLabel", props, parent)
	i.BackgroundTransparency = 1
	i.Visible = false
	task.spawn(function()
		local a = resolveDeco("haunt_" .. name)
		if a and i.Parent then
			i.Image = a
			i.Visible = true
			if onLoad then pcall(onLoad, i) end
		end
	end)
	return i
end
function Win:_haunt()
	local root, frame, main, strip = self.root, self.frame, self.main, self.strip
	local L, st = self.look, self.st
	local anim = L.anim ~= false
	local H = { lit = setmetatable({}, { __mode = "k" }), flames = setmetatable({}, { __mode = "k" }) }
	self.haunt = H
	local rnd = Random.new()
	local function r(a, b) return rnd:NextNumber(a, b) end

	-- candle light on the wordmark: a soft accent outline that breathes
	local glow
	if self.brandLabel then
		glow = make("UIStroke", { Thickness = 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual, Transparency = 0.5 }, self.brandLabel)
		self:paint(glow, "Color", "accent")
	end

	-- slime: drips hang from under the top bar across the content side, under
	-- the tab names (the strip is lifted over them), grow, and let a drop go
	strip.ZIndex = 3
	local drips = {}
	for _, x in ipairs({ 0.04, 0.12, 0.23, 0.37, 0.51, 0.6, 0.73, 0.86, 0.95 }) do
		local d = make("Frame", { Name = "Drip", Position = UDim2.new(x, 0, 0, 0), Size = UDim2.fromOffset(5, r(5, 16)), BorderSizePixel = 0, ZIndex = 2 }, main)
		self:paint(d, "BackgroundColor3", "accent2")
		make("UICorner", { CornerRadius = UDim.new(0, 3) }, d)
		local bulb = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, 1, -2), Size = UDim2.fromOffset(9, 9), BorderSizePixel = 0, ZIndex = 2 }, d)
		self:paint(bulb, "BackgroundColor3", "accent2")
		make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, bulb)
		drips[#drips + 1] = d
	end
	local function drop(d)
		local f = make("Frame", { AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(d.Position.X.Scale, 2, 0, d.Size.Y.Offset), Size = UDim2.fromOffset(5, 7),
			BorderSizePixel = 0, ZIndex = 2 }, main)
		self:paint(f, "BackgroundColor3", "accent2")
		make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, f)
		self:_tw(f, 0.9, { Position = f.Position + UDim2.fromOffset(0, 110), BackgroundTransparency = 1 }, Enum.EasingStyle.Quad)
		task.delay(0.95, function() pcall(function() f:Destroy() end) end)
	end

	-- fog rolling along the bottom, two layers at their own speed (over the
	-- content, faint, clicks go through)
	local fogBox = make("Frame", { Name = "Fog", AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 0, 1, 0), Size = UDim2.new(1, 0, 0, 96),
		BackgroundTransparency = 1, ClipsDescendants = true, ZIndex = 6 }, frame)
	local fogs = {}
	for i, t in ipairs({ 0.8, 0.88 }) do
		local f = self:_hauntSprite("fog", { Size = UDim2.new(2, 0, 1, 0), ScaleType = Enum.ScaleType.Tile, TileSize = UDim2.new(0.5, 0, 1, 0),
			ImageTransparency = t, ZIndex = 6 }, fogBox)
		fogs[i] = { f, 0, i == 1 and 0.012 or 0.021 }
	end

	-- cobwebs in two corners, a spider on a thread from the top one
	local topH = st.topH
	self:_hauntSprite("web", { Position = UDim2.fromOffset(-3, topH - 3), Size = UDim2.fromOffset(88, 88), ImageTransparency = 0.25, ZIndex = 7 }, root)
	self:_hauntSprite("web", { AnchorPoint = Vector2.new(1, 1), Position = UDim2.new(1, 3, 1, 3), Size = UDim2.fromOffset(78, 78), Rotation = 180,
		ImageTransparency = 0.3, ZIndex = 7 }, root)
	local thread = make("Frame", { Position = UDim2.fromOffset(34, topH), Size = UDim2.fromOffset(1, 0), BorderSizePixel = 0, BackgroundTransparency = 0.5, ZIndex = 7 }, root)
	self:paint(thread, "BackgroundColor3", "text")
	local spider = self:_hauntSprite("spider", { AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.fromOffset(34, topH + 2), Size = UDim2.fromOffset(16, 16), ZIndex = 7 }, root)

	-- the ghost lives behind the window and peeks out of its right side;
	-- click it and it's gone
	local ghost = self:_hauntSprite("ghost", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(1, -60, 0.4, 0), Size = UDim2.fromOffset(48, 56),
		ZIndex = 0, AutoButtonColor = false }, root, true)
	local ghostBusy = false
	local function peek()
		if ghostBusy or not anim or not ghost.Parent then return end
		ghostBusy = true
		ghost.ImageTransparency, ghost.Rotation = 0, 0
		ghost.Position = UDim2.new(1, -60, 0.4, 0)
		self:_tw(ghost, 0.8, { Position = UDim2.new(1, -8, 0.4, 0), Rotation = 14 }, Enum.EasingStyle.Back)
		task.delay(3.2, function()
			if not ghost.Parent then return end
			self:_tw(ghost, 0.6, { Position = UDim2.new(1, -60, 0.4, 0), Rotation = 0 })
			task.delay(0.65, function() ghostBusy = false end)
		end)
	end
	ghost.MouseButton1Click:Connect(function()
		self:_tw(ghost, 0.3, { ImageTransparency = 1, Size = UDim2.fromOffset(72, 84) })
		task.delay(0.35, function()
			if not ghost.Parent then return end
			ghost.Size, ghost.Position, ghost.Rotation = UDim2.fromOffset(48, 56), UDim2.new(1, -60, 0.4, 0), 0
			ghost.ImageTransparency = 0
		end)
	end)

	-- bats flap across the header now and then (two wing frames)
	local bats, batImg = {}, {}
	task.spawn(function()
		batImg.up, batImg.down = resolveDeco("haunt_bat_up"), resolveDeco("haunt_bat_down")
	end)
	local function spawnBats()
		if not (batImg.up and batImg.down) then return end
		local W = root.Size.X.Offset
		local ltr = rnd:NextNumber() < 0.5
		for n = 1, rnd:NextInteger(2, 4) do
			local sc = r(0.75, 1.1)
			local b = make("ImageLabel", { BackgroundTransparency = 1, Image = batImg.up, Size = UDim2.fromOffset(30 * sc, 15 * sc), ZIndex = 8 }, root)
			bats[#bats + 1] = { b, ltr and (-40 - n * 34) or (W + 40 + n * 34), r(4, topH - 10), (ltr and 1 or -1) * r(150, 230), r(0, 6) }
		end
	end

	-- one loop: flicker, fog, bats, drips, spider, and the ghost's visits
	local fl, flAt, dripAt, batAt, ghostAt, wasOpen = 1, 0, 0, os.clock() + r(4, 9), os.clock() + r(40, 80), false
	local sp = { y = 0, dir = 0, rest = os.clock() + r(3, 8), up = false }
	self:bconn(RunService.Heartbeat:Connect(function(dt)
		if not root.Parent then return end
		local open = root.Visible and not self.growing
		if open and not wasOpen then task.delay(0.7, peek) end
		wasOpen = open
		if not open or not anim then return end
		local now = os.clock()
		-- candle flicker: the wordmark, logo, window glow, lit pumpkins, flames
		if now >= flAt then
			flAt = now + r(0.06, 0.14)
			fl += (r(0.72, 1.05) - fl) * 0.35
			if rnd:NextNumber() < 0.03 then fl = 0.55 end
			if glow then glow.Transparency = 1 - 0.55 * fl end
			if self.logoLabel and self.logoLabel.Parent then self.logoLabel.ImageTransparency = (1 - fl) * 0.3 end
			local gs = self.glowStroke
			if gs and gs.Parent then gs.Transparency = 1 - math.min(1, (self.look.glow or 0) / 28) * 0.7 * fl end
			for face in pairs(H.lit) do
				if face.Parent and face:GetAttribute("on") then face.ImageTransparency = (1 - fl) * 0.6 end
			end
			for fi in pairs(H.flames) do
				if fi.Parent then
					fi.ImageTransparency = (1 - fl) * 0.35
					fi.Rotation = (fl - 0.88) * 18
				end
			end
		end
		-- fog
		for _, f in ipairs(fogs) do
			f[2] = (f[2] + f[3] * dt) % 1
			f[1].Position = UDim2.new(-f[2], 0, 0, 0)
		end
		-- drips grow; a long one lets a drop go
		if now >= dripAt then
			dripAt = now + 0.7
			for _, d in ipairs(drips) do
				local len = d.Size.Y.Offset + r(0.6, 2.2)
				if len > 26 then
					drop(d)
					len = r(4, 9)
				end
				d.Size = UDim2.fromOffset(5, len)
			end
		end
		-- bats
		if now >= batAt then
			batAt = now + r(9, 16)
			spawnBats()
		end
		for i = #bats, 1, -1 do
			local b = bats[i]
			b[2] += b[4] * dt
			b[5] += dt * 18
			local img = b[1]
			img.Position = UDim2.fromOffset(b[2], b[3] + math.sin(b[5] / 3) * 5)
			img.Image = (math.sin(b[5]) > 0) and batImg.up or batImg.down
			if b[2] < -160 or b[2] > root.Size.X.Offset + 160 then
				img:Destroy()
				table.remove(bats, i)
			end
		end
		-- spider: drops on its thread, hangs, climbs back up
		if sp.dir == 0 and not sp.up and now >= sp.rest then sp.dir = 1 end
		if sp.dir ~= 0 then
			sp.y += sp.dir * 22 * dt
			if sp.y >= 44 then sp.y, sp.dir, sp.rest, sp.up = 44, 0, now + r(2, 4), true end
			if sp.y <= 0 then sp.y, sp.dir, sp.rest = 0, 0, now + r(8, 15) end
		elseif sp.up and now >= sp.rest then
			sp.up, sp.dir = false, -1
		end
		thread.Size = UDim2.fromOffset(1, sp.y)
		spider.Position = UDim2.fromOffset(34, topH + 2 + sp.y)
		-- the ghost drops by now and then
		if now >= ghostAt then
			ghostAt = now + r(45, 90)
			peek()
		end
	end))
end

-- ============================================================
--  SPOOKY: the moving parts (teek 10-16, the "Inferno Noir" mockup with
--  Void's type). Black flame tongues with a white-hot rim are thrown off the
--  window's sides and bottom and climb; white sparks; small ghosts ride the
--  heat up the sides; two guardian ghosts float at the bottom corners - they
--  bob, blink, watch the cursor and drift away from it, and a click puffs one
--  away for a moment. Everything is pooled (no instances made per frame) and
--  driven by one Heartbeat loop that sleeps while the menu is hidden; with
--  Animations off only the still parts show. Decorations strength (decoAlpha)
--  scales the fire. Sprites: assets/deco/spooky_*.png (make_spooky_art.py).
-- ============================================================
function Win:_spookySprite(name, props, parent, button, onLoad)
	local i = make(button and "ImageButton" or "ImageLabel", props, parent)
	i.BackgroundTransparency = 1
	i.Visible = false
	task.spawn(function()
		local a = resolveDeco("spooky_" .. name)
		if a and i.Parent then
			i.Image = a
			i.Visible = true
			if onLoad then pcall(onLoad, i) end
		end
	end)
	return i
end
function Win:_spooky()
	local root, frame = self.root, self.frame
	local L = self.look
	local anim = L.anim ~= false
	local k = math.clamp((tonumber(L.decoAlpha) or 100) / 100, 0, 1)
	local rnd = Random.new()
	local function r(a, b) return rnd:NextNumber(a, b) end
	local GuiService = game:GetService("GuiService")
	local img = {}
	task.spawn(function()
		img.flame, img.dot, img.ghost_mini = resolveDeco("spooky_fire"), resolveDeco("spooky_dot"), resolveDeco("spooky_ghost_mini")
	end)

	-- the dark body of the aura: a soft black glow round the window (9-slice)
	local halo = self:_spookySprite("halo", { Name = "SpookyHalo", AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
		Size = UDim2.new(1, 80, 1, 80), ScaleType = Enum.ScaleType.Slice, SliceCenter = Rect.new(79, 79, 81, 81),
		ImageTransparency = 1 - 0.7 * (0.35 + 0.65 * k), ZIndex = 0 }, root)
	local fireBox = make("Frame", { Name = "SpookyFire", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ZIndex = 0 }, root)
	local sparkBox = make("Frame", { Name = "SpookySparks", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ZIndex = 0 }, root)
	local swarmBox = make("Frame", { Name = "SpookySwarm", Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ZIndex = 0 }, root)
	-- white heat rising from inside the bottom of the window, under the content
	local heat = make("Frame", { Name = "SpookyHeat", AnchorPoint = Vector2.new(0, 1), Position = UDim2.fromScale(0, 1), Size = UDim2.new(1, 0, 0, 110),
		BackgroundColor3 = WHITE, BackgroundTransparency = 0.35, BorderSizePixel = 0, ZIndex = 0 }, frame)
	make("UIGradient", { Rotation = 90, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0.86) }) }, heat)
	self:corner(heat, 1)

	-- pools
	local function pool(n, parent, anchor)
		local t = {}
		for i = 1, n do
			t[i] = { on = false, lbl = make("ImageLabel", { BackgroundTransparency = 1, AnchorPoint = anchor or Vector2.new(0.5, 0.5), Visible = false, ZIndex = 0 }, parent) }
		end
		return t
	end
	local FL = pool(anim and math.floor(150 * k + 0.5) or 0, fireBox)
	local SPK = pool(anim and math.floor(34 * k + 0.5) or 0, sparkBox)
	local MINI = pool(anim and 7 or 0, swarmBox, Vector2.new(0.5, 0.36))

	-- a point on the window's edge and its outward normal: 1 top, 2 bottom, 3 left, 4 right, 5 a corner
	local function edge(side, inset, W, H)
		local rad = math.max(4, L.rad or 12)
		local m, u = rad + 4, rnd:NextNumber()
		if side == 1 then return m + u * (W - 2 * m), inset, 0, -1 end
		if side == 2 then return m + u * (W - 2 * m), H - inset, 0, 1 end
		if side == 3 then return inset, m + u * (H - 2 * m), -1, 0 end
		if side == 4 then return W - inset, m + u * (H - 2 * m), 1, 0 end
		local c = rnd:NextInteger(1, 4)
		local sx, sy, D = (c == 1 or c == 3) and -1 or 1, c <= 2 and -1 or 1, math.sqrt(0.5)
		local ox, oy = sx < 0 and rad or W - rad, sy < 0 and rad or H - rad
		return ox + sx * D * (rad - inset), oy + sy * D * (rad - inset), sx * D, sy * D
	end
	local function side(w)
		local x = rnd:NextNumber() * (w[1] + w[2] + w[3] + w[4] + w[5])
		for i = 1, 5 do x -= w[i] if x <= 0 then return i end end
		return 4
	end
	local FIRE_SIDES, SPARK_SIDES = { 0.03, 0.14, 0.38, 0.38, 0.07 }, { 1, 0.15, 1.3, 1.3, 0.3 }

	-- black fire: tongues born under the window's edge, thrown out and up,
	-- swaying - short-lived and close in, so it hugs the frame like an aura
	-- (live 10-16: longer-lived tongues drifted off the top as lone thorns)
	local RATE, LIFE, LIFT, OUT, RISE, SIZE = 125 * k, 1.05, 26, 16, 22, 1.5
	local fAcc, sAcc, mAcc = 0, 0, 0
	local function stepFire(dt, t, W, H)
		if not img.flame then return end
		fAcc += dt * RATE
		while fAcc >= 1 do
			fAcc -= 1
			for _, p in ipairs(FL) do
				if not p.on then
					local x, y, nx, ny = edge(side(FIRE_SIDES), 18, W, H)
					local out, rise = OUT * r(0.55, 1.45), RISE * r(0.45, 1.25)
					p.on, p.x, p.y, p.ny = true, x, y, ny
					p.vx, p.vy = nx * out + r(-5, 5), ny * out * (ny > 0 and 0.5 or 1) - rise
					p.age, p.life = 0, LIFE * r(0.6, 1.4)
					p.s0, p.s1, p.seed = r(20, 30), r(30, 56) * SIZE, r(0, 100)
					if p.lbl.Image ~= img.flame then p.lbl.Image = img.flame end
					p.lbl.Visible = true
					break
				end
			end
		end
		local kk = 0.35 + 0.65 * k
		for _, p in ipairs(FL) do
			if p.on then
				p.age += dt
				if p.age >= p.life then
					p.on = false
					p.lbl.Visible = false
				else
					local wob = math.sin(t * 2.3 + p.seed) * 16 + math.sin(t * 4.3 + p.seed * 1.9) * 8
					if p.ny ~= 0 then p.vx += wob * dt else p.vy += wob * 0.35 * dt end
					p.vy -= LIFT * dt
					p.vx *= 1 - 0.5 * dt
					p.x += p.vx * dt
					p.y += p.vy * dt
					local q = p.age / p.life
					local s = (p.s0 + (p.s1 - p.s0) * (1 - (1 - q) * (1 - q))) * (1 - 0.32 * q)
					-- the tongue turns about its base: GUI rotation pivots on the centre,
					-- so the centre is placed along the tongue from the base point
					local th = math.atan2(p.vy, p.vx) + math.pi / 2
					local d = s * 0.8125
					local lb = p.lbl
					lb.Size = UDim2.fromOffset(s, s * 2.5)
					lb.Position = UDim2.fromOffset(p.x + math.sin(th) * d, p.y - math.cos(th) * d)
					lb.Rotation = math.deg(th)
					lb.ImageTransparency = 1 - (math.sin(math.pi * q) ^ 0.85) * 0.95 * kk
				end
			end
		end
	end
	-- white sparks off the fire
	local function stepSparks(dt, t, W, H)
		if not img.dot then return end
		sAcc += dt * 26 * k
		while sAcc >= 1 do
			sAcc -= 1
			for _, p in ipairs(SPK) do
				if not p.on then
					local x, y, nx = edge(side(SPARK_SIDES), -4, W, H)
					p.on, p.x, p.y = true, x, y
					p.vx, p.vy = nx * r(12, 34), -r(22, 68)
					p.age, p.life, p.s, p.seed = 0, r(1.4, 3.8), r(2, 5) * 3.2, r(0, 60)
					if p.lbl.Image ~= img.dot then p.lbl.Image = img.dot end
					p.lbl.Size = UDim2.fromOffset(p.s, p.s)
					p.lbl.Visible = true
					break
				end
			end
		end
		for _, p in ipairs(SPK) do
			if p.on then
				p.age += dt
				if p.age >= p.life then
					p.on = false
					p.lbl.Visible = false
				else
					p.x += (p.vx + math.sin(t * 3.1 + p.seed) * 14) * dt
					p.y += p.vy * dt
					p.vx *= 1 - 0.8 * dt
					local q = p.age / p.life
					p.lbl.Position = UDim2.fromOffset(p.x, p.y)
					p.lbl.ImageTransparency = 1 - math.sin(math.pi * q) * (0.55 + 0.45 * math.sin(t * 11 + p.seed * 3)) * 0.85
				end
			end
		end
	end
	-- little ghosts riding the heat up the sides
	local function stepSwarm(dt, t, W, H)
		if not img.ghost_mini then return end
		mAcc += dt * 2.2 * k
		while mAcc >= 1 do
			mAcc -= 1
			for _, p in ipairs(MINI) do
				if not p.on then
					local left = rnd:NextNumber() < 0.5
					p.on = true
					p.x0 = left and -14 - r(0, 44) or W + 14 + r(0, 44)
					p.y, p.vy = H * r(0.45, 1), -r(26, 60)
					p.amp, p.fr, p.R = r(10, 26), r(1, 2.2), r(6, 11)
					p.age, p.life, p.ph = 0, r(4, 7.5), r(0, 6)
					if p.lbl.Image ~= img.ghost_mini then p.lbl.Image = img.ghost_mini end
					p.lbl.Size = UDim2.fromOffset(p.R * 2.82, p.R * 3.76)
					p.lbl.Visible = true
					break
				end
			end
		end
		for _, p in ipairs(MINI) do
			if p.on then
				p.age += dt
				if p.age >= p.life then
					p.on = false
					p.lbl.Visible = false
				else
					p.y += p.vy * dt
					local q, w = p.age / p.life, p.age * p.fr * 2 + p.ph
					p.lbl.Position = UDim2.fromOffset(p.x0 + math.sin(w) * p.amp, p.y)
					p.lbl.Rotation = math.deg(math.cos(w) * 0.25)
					p.lbl.ImageTransparency = 1 - math.min(1, q * 5) * math.min(1, (1 - q) * 2.5) * 0.9
				end
			end
		end
	end

	-- the guardians: a ghost on each bottom corner. The body is a 6-frame
	-- strip of the hem rippling, two frames crossfaded (no stepping); the eyes
	-- are their own image so they can blink and look.
	local G = {}
	for _, sd in ipairs({ -1, 1 }) do
		local g = { side = sd, R = 30, sp = sd < 0 and 1.4 or 1.6, ph = sd < 0 and 0.4 or 2.1, ox = 0, oy = 0, lx = 0, ly = 0,
			blinkAt = r(1, 4), blinkT = -9, poof = 0, fade = 1 }
		local sc = g.R / 34
		g.w, g.h = 96 * sc, 128 * sc
		g.glow = self:_spookySprite("dot", { AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(g.R * 4.4, g.R * 4.4), ImageTransparency = 0.72, ZIndex = 8 }, root)
		local ok, box = pcall(function() return make("CanvasGroup", { BackgroundTransparency = 1 }, nil) end)
		if not ok then box = make("Frame", { BackgroundTransparency = 1 }, nil) end
		box.Name = "SpookyGhost"
		box.AnchorPoint = Vector2.new(0.5, 0.36)
		box.Size = UDim2.fromOffset(g.w, g.h)
		box.ZIndex = 9
		box.Parent = root
		g.box, g.group = box, ok
		g.a = self:_spookySprite("ghost", { Size = UDim2.fromScale(1, 1), ImageRectSize = Vector2.new(96, 128), ZIndex = 1 }, box)
		g.b = self:_spookySprite("ghost", { Size = UDim2.fromScale(1, 1), ImageRectSize = Vector2.new(96, 128), ImageTransparency = 1, ZIndex = 2 }, box)
		g.eyes = self:_spookySprite("eyes", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.349), Size = UDim2.fromScale(0.5, 0.1875), ZIndex = 3 }, box)
		local hit = make("TextButton", { Text = "", AutoButtonColor = false, BackgroundTransparency = 1, Position = UDim2.fromScale(0.12, 0.04), Size = UDim2.fromScale(0.76, 0.8), ZIndex = 4 }, box)
		hit.MouseButton1Click:Connect(function() if g.poof == 0 and not g.goneUntil then g.poof = 0.001 self:_tick(true) end end)
		G[#G + 1] = g
	end
	local function mouseLocal()
		local ok, p = pcall(function()
			return (UserInputService:GetMouseLocation() - GuiService:GetGuiInset() - root.AbsolutePosition) / (self.uiScale or 1)
		end)
		return ok and p or nil
	end
	local function stepGhost(g, dt, t, W, H, m)
		local hx, hy = g.side < 0 and -16 or W + 16, H - 40
		local by = hy + math.sin(t * g.sp + g.ph) * g.R * 0.16
		local tx, ty = 0, 0
		if m then
			local dx, dy = hx - m.X, by - m.Y
			local dd, zone = math.sqrt(dx * dx + dy * dy), g.R * 3 + 50
			if dd < zone and dd > 0.001 then
				local f = (1 - dd / zone) * g.R * 0.8
				tx, ty = dx / dd * f, dy / dd * f
			end
		end
		local e1, e2 = 1 - math.exp(-dt * 5), 1 - math.exp(-dt * 9)
		g.ox += (tx - g.ox) * e1
		g.oy += (ty - g.oy) * e1
		local cx, cy = hx + g.ox, by + g.oy
		local lx, ly = math.sin(t * 0.5 + g.ph) * g.R * 0.06, 0
		if m then
			local dx, dy = m.X - cx, m.Y - cy
			local dd = math.max(1, math.sqrt(dx * dx + dy * dy))
			lx, ly = dx / dd * g.R * 0.1, dy / dd * g.R * 0.08
		end
		g.lx += (lx - g.lx) * e2
		g.ly += (ly - g.ly) * e2
		if t >= g.blinkAt then g.blinkT, g.blinkAt = t, t + r(2.4, 7) end
		local bp = (t - g.blinkT) / 0.18
		local blink = (bp >= 0 and bp < 1) and (1 - math.sin(math.pi * bp) * 0.94) or 1
		if g.poof > 0 then
			g.poof += dt * 2.4
			if g.poof >= 1 then g.poof, g.goneUntil, g.fade = 0, t + 1.6, 0 end
		end
		if g.goneUntil and t >= g.goneUntil then
			g.fade = math.min(1, g.fade + dt * 1.8)
			if g.fade >= 1 then g.goneUntil = nil end
		end
		local alpha = (1 - g.poof) * ((g.goneUntil and t < g.goneUntil) and 0 or g.fade)
		local s = 1 + g.poof * 0.7
		local fpos = ((t * 3.4 + g.ph) / (2 * math.pi) % 1) * 6
		local f0 = math.floor(fpos)
		g.a.ImageRectOffset = Vector2.new((f0 % 6) * 96, 0)
		g.b.ImageRectOffset = Vector2.new(((f0 + 1) % 6) * 96, 0)
		g.b.ImageTransparency = 1 - (fpos - f0)
		g.eyes.Position = UDim2.new(0.5, g.lx, 0.349, g.ly)
		g.eyes.Size = UDim2.fromScale(0.5, 0.1875 * blink)
		local box = g.box
		box.Position = UDim2.fromOffset(cx, cy)
		box.Size = UDim2.fromOffset(g.w * s, g.h * s)
		box.Rotation = math.sin(t * g.sp * 0.55 + g.ph) * 4 + g.ox * 0.23
		if g.group then box.GroupTransparency = 1 - alpha else box.Visible = alpha > 0.05 end
		g.glow.Position = UDim2.fromOffset(cx, cy + g.R * 0.25)
		g.glow.ImageTransparency = 1 - 0.28 * alpha * (0.85 + 0.15 * math.sin(t * 2 + g.ph))
	end
	-- place the still parts once (Animations off, or before the first frame)
	for _, g in ipairs(G) do stepGhost(g, 0, 0, root.Size.X.Offset, root.Size.Y.Offset, nil) end

	local clock, warm = 0, false
	self:bconn(RunService.Heartbeat:Connect(function(dt)
		if not root.Parent then return end
		if not (root.Visible and not self.growing) or not anim then return end
		dt = math.min(dt, 0.05)
		local W, H = root.Size.X.Offset, root.Size.Y.Offset
		-- open with the fire already burning, not lighting from nothing
		if not warm and img.flame then
			warm = true
			for _ = 1, 45 do clock += 1 / 30 stepFire(1 / 30, clock, W, H) stepSparks(1 / 30, clock, W, H) end
		end
		clock += dt
		local m = mouseLocal()
		stepFire(dt, clock, W, H)
		stepSparks(dt, clock, W, H)
		stepSwarm(dt, clock, W, H)
		for _, g in ipairs(G) do stepGhost(g, dt, clock, W, H, m) end
		heat.BackgroundTransparency = 0.25 + 0.2 * (0.5 + 0.5 * math.sin(clock * 7.3) * math.sin(clock * 2.9))
		if halo.Parent then halo.ImageTransparency = 1 - (0.78 + 0.12 * math.sin(clock * 0.9)) * (0.35 + 0.65 * k) end
	end))
end

-- background layer: no uploaded assets, so the patterns are frames and gradients
function Win:_pattern()
	local layer = self.patLayer
	if not (layer and layer.Parent) then return end
	for _, c in ipairs(layer:GetChildren()) do c:Destroy() end
	local L = self.look
	local s = (L.strength or 0) / 100
	if s <= 0 or L.pattern == "None" then return end
	if L.pattern == "Scanlines" then
		local holder = make("Frame", { Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1, ZIndex = 0 }, layer)
		make("UIListLayout", { Padding = UDim.new(0, 2) }, holder)
		local col = self.tok.light and BLACK or WHITE
		for _ = 1, 320 do
			make("Frame", { Size = UDim2.new(1, 0, 0, 1), BackgroundColor3 = col, BackgroundTransparency = 1 - 0.06 * s, BorderSizePixel = 0, ZIndex = 0 }, holder)
		end
	elseif L.pattern == "Vignette" then
		for _, rot in ipairs({ 0, 90 }) do
			local f = make("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = BLACK, BorderSizePixel = 0, ZIndex = 0 }, layer)
			local e = 1 - 0.5 * s
			make("UIGradient", { Rotation = rot, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, e), NumberSequenceKeypoint.new(0.3, 1),
				NumberSequenceKeypoint.new(0.7, 1), NumberSequenceKeypoint.new(1, e) }) }, f)
		end
	elseif L.pattern == "Accent wash" then
		local f = make("Frame", { Size = UDim2.fromScale(1, 1), BorderSizePixel = 0, ZIndex = 0 }, layer)
		self:paint(f, "BackgroundColor3", "accent")
		make("UIGradient", { Rotation = 135, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1 - 0.35 * s), NumberSequenceKeypoint.new(0.45, 1),
			NumberSequenceKeypoint.new(1, 1) }) }, f)
	elseif L.pattern == "Custom image" then
		local id = string.match(tostring(L.image or ""), "%d+")
		if id then
			local img = make("ImageLabel", { Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ScaleType = Enum.ScaleType.Crop,
				ImageTransparency = 1 - s, ZIndex = 0 }, layer)
			pcall(function()   -- the builder's Background image fit
				img.ScaleType = Enum.ScaleType[L.fit or "Crop"]
				if L.fit == "Tile" then img.TileSize = UDim2.fromOffset(220, 220) end
			end)
			task.spawn(self._bgImage, self, img, id)
		end
	end
end

-- draws once Roblox has approved the image; saved to disk, so a grey placeholder
-- cached earlier in the session can't come back. Pending = retry every 30 s.
function Win:_bgImage(img, id)
	-- v2: v1 cached Roblox's "private image" placeholder (the crossed-out
	-- circle) to disk and showed it for good (teek 10-10, a decal not shared
	-- publicly) - a fresh name so those bad copies are never read again
	local file = "tempo_bg2_" .. id .. ".png"
	local told = false
	while img.Parent and string.match(tostring(self.look.image or ""), "%d+") == id do
		local got = bgCache[id] or fetchAsset(nil, file, 64)
		local state, url
		if not got then
			state, url = bgState(id)
			if not img.Parent then return end   -- changed while we asked
			-- a private (not shared) asset "completes" with a placeholder picture
			if state == "Completed" and url and string.find(url, "PrivateImage", 1, true) then
				self:Notify(("image %s is private - on Creator Hub set it to public (or use a public image) and pick it again"):format(id), "warn")
				return
			end
			if state == "Completed" and url then got = fetchAsset(url, file, 64) end
		end
		-- no http / getcustomasset: rbxthumb, which still works for approved images
		if got or state == nil or state == "Completed" then
			got = got or ("rbxthumb://type=Asset&id=" .. id .. "&w=768&h=432")
			bgCache[id] = got
			img.Image = got
			return
		end
		if state == "Blocked" or state == "Error" then
			self:Notify(("Roblox won't show image %s (%s) - check the id"):format(id, state), "warn")
			return
		end
		if not told then told = true self:Notify("that image is still in Roblox's review - it shows up by itself once approved", "info") end
		task.wait(30)
	end
end

-- ------------------------------------------------------------ rail + tabs
-- A rail entry is either a Group (its tabs fill the strip) or a lone tab.
-- The entry is data; its button is chrome (rebuilt with the preset).
function Win:_railEntry(title)
	local entry = { tabs = {}, title = title, order = #self.rail + 1 }
	self.rail[#self.rail + 1] = entry
	self:_railButton(entry)
	return entry
end

function Win:_railButton(entry)
	local name, nav = self.presetName, self.st.nav
	local h = name == "Arcade" and 36 or 34
	local shadowTok = (name == "Anime" and "text") or (name == "Manga" and "accent") or nil
	local wrap, face, shadow
	if shadowTok then
		wrap, face, shadow = self:_shadowed(self.railFrame, { Size = UDim2.new(1, 0, 0, h), LayoutOrder = entry.order }, shadowTok, 4, 4)
		shadow.Visible = false
	else
		wrap = make("Frame", { Size = UDim2.new(1, 0, 0, h), BackgroundTransparency = 1, LayoutOrder = entry.order }, self.railFrame)
		face = make("Frame", { Size = UDim2.fromScale(1, 1), BorderSizePixel = 0 }, wrap)
	end
	face.BackgroundTransparency = 1
	self:corner(face, "pill")
	local bar = make("Frame", { Size = UDim2.new(0, name == "Gamer" and 3 or 2, 1, 0), BorderSizePixel = 0, Visible = false }, face)
	local dot = make("Frame", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 10, 0.5, 0), Size = UDim2.fromOffset(6, 6), BorderSizePixel = 0, Visible = false }, face)
	make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, dot)
	local mark = make("Frame", { AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -12, 0.5, 0), Size = UDim2.fromOffset(6, 6), Rotation = 45, BorderSizePixel = 0, Visible = false }, face)
	local suffix = self:text({ Text = "<", AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -8, 0, 0), Size = UDim2.new(0, 12, 1, 0), Visible = false,
		TextXAlignment = Enum.TextXAlignment.Right }, face, "onAccent", "head", 9)
	-- (the right 12 px is the arcade "<" / the grimoire diamond; the other styles give it to the text)
	local label = self:text({ Text = nav.upper and string.upper(entry.title) or entry.title, Position = UDim2.fromOffset(10, 0),
		Size = UDim2.new(1, (name == "Arcade" or name == "Grimoire") and -22 or -14, 1, 0),
		TextTruncate = Enum.TextTruncate.AtEnd }, face, "dim", nav.role, nav.size)
	local hit = make("TextButton", { Text = "", AutoButtonColor = false, BackgroundTransparency = 1, Size = UDim2.fromScale(1, 1), ZIndex = 3 }, wrap)
	entry.wrap, entry.face, entry.label = wrap, face, label
	local shift = (name == "Anime" and 4) or (name == "Manga" and 3) or 0
	entry.paintState = function()
		if not face.Parent then return end
		local tok = self.tok
		local on = self.railSel == entry
		local hov = entry.hover and not on
		local bg, bgT, txt = tok.sHover, 1, tok.dim
		bar.Visible, dot.Visible, mark.Visible, suffix.Visible = false, false, false, false
		if shadow then shadow.Visible = false end
		if on then
			bg, bgT, txt = tok.sHover, tok.tHover, tok.text
			if name == "Glass" then
				bg, bgT = WHITE, 0.87
				dot.Visible = true
				dot.BackgroundColor3 = tok.accent
			elseif name == "Anime" or name == "Arcade" then
				bg, bgT, txt = tok.accent, 0, tok.onAccent
			elseif name == "Manga" then
				bg, bgT, txt = tok.ink, 0, tok.paper
			elseif name == "Gamer" then
				bg, bgT, txt = tok.accent, 0.86, tok.accent
			elseif name == "Grimoire" then
				bgT, txt = 1, tok.gold
				mark.Visible = true
				mark.BackgroundColor3 = tok.accent
			end
			if self.st.spooky then
				-- Spooky: the picked entry is a white block, black type
				bg, bgT, txt = tok.text, 0, tok.bg
			elseif name == "Sleek" or name == "Gamer" then
				bar.Visible = true
				bar.BackgroundColor3 = tok.accent
			end
			if shadow then shadow.Visible = true end
			if name == "Arcade" then suffix.Visible = true end
		elseif hov then
			txt = name == "Grimoire" and tok.gold or tok.text
			if shift == 0 and name ~= "Grimoire" then bg, bgT = tok.sHover, tok.tHover end
		end
		face.BackgroundColor3, face.BackgroundTransparency = bg, bgT
		label.TextColor3 = txt
		label.Position = UDim2.fromOffset(dot.Visible and 22 or 10, 0)
		local x = (on or hov) and shift or 0
		if face.Position.X.Offset ~= x then self:_tw(face, 0.12, { Position = UDim2.fromOffset(x, 0) }, Enum.EasingStyle.Back) end
	end
	hit.MouseEnter:Connect(function()
		entry.hover = true
		entry.paintState()
	end)
	hit.MouseLeave:Connect(function()
		entry.hover = false
		entry.paintState()
	end)
	hit.MouseButton1Click:Connect(function()
		self:_tick(true)
		self:_selectEntry(entry)
	end)
	entry.paintState()
end

function Win:_selectEntry(entry, tab)
	local old = self.railSel
	self.railSel = entry
	if old and old ~= entry and old.paintState then old.paintState() end
	if entry.paintState then entry.paintState() end
	-- the strip shows this entry's tabs
	for _, c in ipairs(self.strip:GetChildren()) do if c:IsA("GuiButton") then c:Destroy() end end
	local name, ts = self.presetName, self.st.tab
	for i, t in ipairs(entry.tabs) do
		local b = make("TextButton", { Text = ts.upper and string.upper(t.name) or t.name, AutoButtonColor = false, BackgroundTransparency = 1,
			Size = UDim2.new(0, 0, 1, 0), AutomaticSize = Enum.AutomaticSize.X, TextSize = ts.size, LayoutOrder = i }, self.strip)
		self:paint(b, "FontFace", function() return self:_face(ts.role) end)
		local bar
		if name == "Grimoire" then
			-- a double rule: ONE frame, its middle faded out (two scale-wide lines
			-- nested in a holder stretched the picked tab across the whole strip)
			bar = make("Frame", { AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 0, 1, 0), Size = UDim2.new(1, 0, 0, 4), BorderSizePixel = 0, Visible = false }, b)
			self:paint(bar, "BackgroundColor3", "gold")
			make("UIGradient", { Rotation = 90, Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0), NumberSequenceKeypoint.new(0.26, 0),
				NumberSequenceKeypoint.new(0.27, 1), NumberSequenceKeypoint.new(0.73, 1), NumberSequenceKeypoint.new(0.74, 0), NumberSequenceKeypoint.new(1, 0) }) }, bar)
		else
			local bh = ({ Glass = 3, Anime = 5, Manga = 6, Arcade = 4 })[name] or 2
			local wide = name == "Glass" and 0.4 or 1
			bar = make("Frame", { AnchorPoint = Vector2.new(0.5, 1), Position = UDim2.new(0.5, 0, 1, 0), Size = UDim2.new(wide, name == "Anime" and 8 or 0, 0, bh),
				BorderSizePixel = 0, Visible = false }, b)
			if name == "Glass" then make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, bar) end
			self:paint(bar, "BackgroundColor3", "accent")
		end
		t.stripButton, t.stripBar = b, bar
		b.MouseEnter:Connect(function() if self.current ~= t then b.TextColor3 = self.tok.text end end)
		b.MouseLeave:Connect(function() if self.current ~= t then b.TextColor3 = self.tok.dim end end)
		b.MouseButton1Click:Connect(function()
			self:_tick(true)
			self:_selectTab(t)
		end)
	end
	self:_selectTab(tab or entry.last or entry.tabs[1])
end

function Win:_paintStrip()
	if not self.railSel then return end
	local name = self.presetName
	local onTok = (name == "Gamer" or name == "Arcade") and "accent" or (name == "Grimoire" and "gold" or "text")
	for _, t in ipairs(self.railSel.tabs) do
		if t.stripButton and t.stripButton.Parent then
			local on = self.current == t
			t.stripButton.TextColor3 = on and self.tok[onTok] or self.tok.dim
			t.stripBar.Visible = on
		end
	end
end

function Win:_selectTab(t)
	if not t then return end
	local prev = self.current
	if prev and prev ~= t and prev.page then prev.page.Visible = false end
	self.current = t
	t.entry.last = t
	t:_ensureBuilt()
	t.page.Visible = true
	t:_flow()
	self:_paintStrip()
	self:_closePopover()
	-- content slides in
	if prev ~= t and t.colsFrame then
		local p = t.colsPos
		t.colsFrame.Position = p + UDim2.fromOffset(14, 0)
		self:_tw(t.colsFrame, 0.18, { Position = p })
	end
end

function Win:_newTab(name, entry)
	local t = setmetatable({ name = name, ui = self, entry = entry, sections = {}, built = false }, Tab)
	t.cur = nil
	entry.tabs[#entry.tabs + 1] = t
	self.tabs[name] = t
	self.tabList[#self.tabList + 1] = t
	return t
end

function Win:Tab(name)
	local entry = self:_railEntry(name)
	return self:_newTab(name, entry)
end
function Win:Group(title)
	return setmetatable({ ui = self, entry = self:_railEntry(title), title = title }, Group)
end
function Group:Tab(name)
	return self.ui:_newTab(name, self.entry)
end

function Win:Select(name)
	local t = self.tabs[name]
	if t then self:_selectEntry(t.entry, t) end
end

-- ============================================================
--  Tab: specs now, GUI when first opened
-- ============================================================
local function rowHeight(kind, rowH)
	if kind == "slider" then return rowH + 14 end
	if kind == "button" then return math.max(rowH, 38) end
	if kind == "input" then return math.max(rowH, 36) end
	if kind == "segment" then return rowH + 30 end
	if kind == "label" then return 18 end
	return rowH
end
local function estHeight(item, rowH)
	local h = rowHeight(item.kind, rowH)
	if item.desc then h += 15 * (select(2, string.gsub(item.desc, "\n", "")) + 1) end
	return h
end

function Tab:_section(title)
	local s = { title = title, items = {}, tab = self }
	self.sections[#self.sections + 1] = s
	self.cur = s
	if self.built then
		self.ui:_buildCard(self, s)
		self:_flow()
	end
	return s
end
function Tab:_add(item)
	if not self.cur then self:_section(nil) end
	local s = self.cur
	item.section = s
	s.items[#s.items + 1] = item
	self.last = item
	if item.label and item.kind ~= "label" then
		self.ui.index[#self.ui.index + 1] = { label = item.label, tab = self, item = item }
	end
	if self.built and s.card then self.ui:_buildItem(s, item) end
	return item
end

function Tab:_ensureBuilt()
	if self.built then return end
	self.built = true
	local ui = self.ui
	local D = DENSITY[ui.look.density] or DENSITY.Cozy
	local padPx, gap = D[2], D[3]
	local page = make("ScrollingFrame", { Name = self.name, Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0,
		ScrollBarThickness = 3, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, Visible = false,
		ScrollingDirection = Enum.ScrollingDirection.Y }, ui.pages)
	ui:paint(page, "ScrollBarImageColor3", "line2")
	-- inset by position, not a UIPadding: padding on a ScrollingFrame doesn't
	-- narrow scale-sized children, so the right column ran off the edge
	local cols = make("Frame", { Name = "Cols", Position = UDim2.fromOffset(padPx, padPx), Size = UDim2.new(1, -padPx * 2, 0, 0),
		AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1 }, page)
	make("UIPadding", { PaddingBottom = UDim.new(0, padPx + 6) }, cols)
	self.page, self.colsFrame, self.colsPos, self.cols, self.gap = page, cols, cols.Position, {}, gap
	for i = 1, 2 do
		local c = make("Frame", { Name = "C" .. i, BackgroundTransparency = 1, AutomaticSize = Enum.AutomaticSize.Y, Size = UDim2.new(0.5, -gap / 2, 0, 0),
			Position = UDim2.new((i - 1) * 0.5, (i - 1) * gap / 2, 0, 0) }, cols)
		make("UIListLayout", { Padding = UDim.new(0, gap), SortOrder = Enum.SortOrder.LayoutOrder }, c)
		self.cols[i] = c
	end
	self.flowed = false
	for _, s in ipairs(self.sections) do ui:_buildCard(self, s) end
end

-- Cards are placed once into the shorter column by estimated height (never
-- reshuffled as content changes); under 620 px wide it's one column.
function Tab:_flow()
	if not self.built then return end
	local wide = self.page.AbsoluteSize.X >= 620 * self.ui.uiScale or self.page.AbsoluteSize.X == 0
	if wide ~= self.wide or not self.flowed then
		self.wide, self.flowed = wide, true
		local gap = self.gap or 12
		self.cols[1].Size = wide and UDim2.new(0.5, -gap / 2, 0, 0) or UDim2.new(1, 0, 0, 0)
		self.cols[2].Visible = wide
		local rowH = (DENSITY[self.ui.look.density] or DENSITY.Cozy)[1]
		local h = { 0, 0 }
		for i, s in ipairs(self.sections) do
			local col = (wide and h[2] < h[1]) and 2 or 1
			if s.card then
				s.card.Parent = self.cols[col]
				s.card.LayoutOrder = i
			end
			local est = 40
			for _, it in ipairs(s.items) do est += estHeight(it, rowH) end
			h[col] += est
		end
	end
end

local function richEscape(s) return (string.gsub(tostring(s), "[&<>]", { ["&"] = "&amp;", ["<"] = "&lt;", [">"] = "&gt;" })) end

function Win:_buildCard(tab, s)
	local st, name = self.st, self.presetName
	local outer, card
	if st.cardShadow then
		local face
		outer, face = self:_shadowed(tab.cols[1], { Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y }, st.cardShadow, 5, 5)
		face.Size = UDim2.new(1, 0, 0, 0)
		face.AutomaticSize = Enum.AutomaticSize.Y
		card = face
	else
		card = make("Frame", { Name = "Card", BorderSizePixel = 0, Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y }, tab.cols[1])
		outer = card
	end
	local fill = self:_fill(card, "sCard", "tCard", st.cardCut, false, nil, 0.8)
	if not st.cardCut and not st.cardBar then
		if name == "Arcade" then self:stroke(card, "ln2", 2) else self:stroke(card, "ln", name == "Manga" and 2 or 1) end
	end
	local bar
	if st.cardBar then
		bar = make("Frame", { Size = UDim2.new(0, 3, 1, 0), BorderSizePixel = 0, ZIndex = 2 }, card)
		self:paint(bar, "BackgroundColor3", "accent")
	end
	local content = make("Frame", { Name = "Content", Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1 }, card)
	-- A scale-high child holds an auto-sized card open: it settles near the
	-- page's height instead of its content's (live 10-12: 356 px cards round
	-- 130 px of rows in every cut / barred / shadowed style). So the cut fill,
	-- the accent bar and the hard shadow take the content's height instead.
	local tall = { fill ~= card and fill or nil, bar, outer ~= card and outer:FindFirstChild("Shadow") or nil }
	if next(tall) then
		local function fit()
			local h = content.AbsoluteSize.Y / self.uiScale
			for _, f in pairs(tall) do f.Size = UDim2.new(f.Size.X.Scale, f.Size.X.Offset, 0, h) end
		end
		content:GetPropertyChangedSignal("AbsoluteSize"):Connect(fit)
		fit()
	end
	pad(content, 4, 4, 10, 6)
	make("UIListLayout", { Padding = UDim.new(0, 0), SortOrder = Enum.SortOrder.LayoutOrder }, content)
	if s.title then
		local h4 = st.h4
		local hh = make("Frame", { Name = "Head", Size = UDim2.new(1, 0, 0, h4.size + 10), BackgroundTransparency = 1, LayoutOrder = 0 }, content)
		pad(hh, 10, 10, 0, 4)
		local title = h4.upper and string.upper(s.title) or s.title
		if h4.role == "head" and h4.size <= 11 and h4.upper then title = spaced(title) end
		local lbl = self:text({ Text = title, Size = UDim2.fromScale(1, 1), TextTruncate = Enum.TextTruncate.AtEnd }, hh, h4.tok, h4.role, h4.size)
		if st.spooky then
			-- a small white flame before each card title
			self:_spookySprite("icon", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 0, 0.5, -1), Size = UDim2.fromOffset(9, 12) }, hh)
			lbl.Position = UDim2.fromOffset(16, 0)
			lbl.Size = UDim2.new(1, -16, 1, 0)
		end
		if h4.prefix then
			lbl.RichText = true
			local prefix = h4.prefix
			self:paint(lbl, "Text", function() return '<font color="' .. toHex(self.tok.accent) .. '">' .. richEscape(prefix) .. "</font>" .. richEscape(title) end)
		end
		if h4.rules then
			-- grimoire: --- Title ----------
			lbl.Position = UDim2.fromOffset(20, 0)
			lbl.Size = UDim2.new(0, 0, 1, 0)
			lbl.AutomaticSize = Enum.AutomaticSize.X
			local r1 = make("Frame", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.fromScale(0, 0.5), Size = UDim2.fromOffset(12, 1), BorderSizePixel = 0, BackgroundTransparency = 0.65 }, hh)
			local r2 = make("Frame", { AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.fromScale(1, 0.5), Size = UDim2.new(1, -40, 0, 1), BorderSizePixel = 0, BackgroundTransparency = 0.65 }, hh)
			self:paint(r1, "BackgroundColor3", "gold")
			self:paint(r2, "BackgroundColor3", "gold")
			local function fit() r2.Size = UDim2.new(1, -(28 + lbl.TextBounds.X + 10), 0, 1) end
			lbl:GetPropertyChangedSignal("TextBounds"):Connect(fit)
			fit()
		end
	end
	s.card, s.content = outer, content
	for _, it in ipairs(s.items) do self:_buildItem(s, it) end
end

-- One row frame per item; the kind's builder fills it.
function Win:_buildItem(s, item)
	if item.row and item.row.Parent then return end
	local rowH = (DENSITY[self.look.density] or DENSITY.Cozy)[1]
	local row = make("Frame", { Name = item.kind, BackgroundTransparency = 1, BorderSizePixel = 0, Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y,
		LayoutOrder = table.find(s.items, item) or 99 }, s.content)
	item.row = row
	pad(row, 10, 10, 0, 0)
	self:corner(row, 0.5)
	self:paint(row, "BackgroundColor3", "sHover")
	local h = (item.kind == "label") and 0 or rowHeight(item.kind, rowH)
	local body = make("Frame", { Name = "Body", BackgroundTransparency = 1, Size = UDim2.new(1, 0, 0, h) }, row)
	item.body = body
	make("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder }, row)
	if item.kind ~= "label" then
		row.MouseEnter:Connect(function() row.BackgroundTransparency = self.tok.tHover end)
		row.MouseLeave:Connect(function() row.BackgroundTransparency = 1 end)
	end
	local ok, err = pcall(item.build, item, body)
	if not ok then warn("[TempoUI] " .. tostring(item.kind) .. ": " .. tostring(err)) end
	if item.desc then self:_descLabel(item) end
end

function Win:_descLabel(item)
	if not (item.row and item.row.Parent) then return end
	if item.descLabel and item.descLabel.Parent then
		item.descLabel.Text = item.desc or ""
		item.descLabel.Visible = item.desc ~= nil
		return
	end
	if not item.desc then return end
	item.descLabel = self:text({ Text = item.desc, Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, TextWrapped = true,
		TextYAlignment = Enum.TextYAlignment.Top, LayoutOrder = 2 }, item.row, "faint", "body", 11)
	make("UIPadding", { PaddingBottom = UDim.new(0, 5) }, item.descLabel)
end

-- the row label every control shares
function Win:_rowLabel(item, body, rightWidth)
	return self:text({ Text = item.label or "", Size = UDim2.new(1, -(rightWidth or 0) - 8, 1, 0), TextTruncate = Enum.TextTruncate.AtEnd }, body, "text", "body", 13)
end

-- ------------------------------------------------------------ Label / Section / Divider
-- A Label describes the element above it (like WindLib); with nothing above,
-- it stands alone. Either way the handle's .Text rewrites just that label
-- (TeekOps/TeekConfig fill status text in later).
local function textHandle(get, set)
	return setmetatable({}, {
		__index = function(_, k)
			if k == "Text" then return get() end
			return nil
		end,
		__newindex = function(_, k, v) if k == "Text" then set(tostring(v)) end end,
	})
end
function Tab:Label(text)
	text = tostring(text or "")
	local ui = self.ui
	local last = self.last
	if last and last.kind ~= "label" and last.section == self.cur then
		last.descParts = last.descParts or {}
		local parts = last.descParts
		local i = #parts + 1
		parts[i] = text
		local function refresh()
			local shown = {}
			for _, p in ipairs(parts) do if p ~= "" then shown[#shown + 1] = p end end
			last.desc = #shown > 0 and table.concat(shown, "\n") or nil
			if last.row then ui:_descLabel(last) end
		end
		refresh()
		return textHandle(function() return parts[i] end, function(v)
			parts[i] = v
			refresh()
		end)
	end
	local it = { kind = "label", label = text }
	it.build = function(item, body)
		body.Size = UDim2.new(1, 0, 0, 0)
		body.AutomaticSize = Enum.AutomaticSize.Y
		item.textLabel = ui:text({ Text = item.label, Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, TextWrapped = true,
			TextYAlignment = Enum.TextYAlignment.Top }, body, "dim", "body", 12)
		make("UIPadding", { PaddingTop = UDim.new(0, 3), PaddingBottom = UDim.new(0, 5) }, item.textLabel)
	end
	self:_add(it)
	self.last = nil
	it.handle = textHandle(function() return it.label end, function(v)
		it.label = v
		if it.textLabel then it.textLabel.Text = v end
	end)
	return it.handle
end
function Tab:Section(text)
	self:_section(text)
	self.last = nil
	return {}
end
function Tab:Divider()
	self.last = nil
	self.cur = nil   -- next element starts a fresh card
	return {}
end

-- shape helpers shared by the controls
local function sharp(ui)
	if ui.st.sharp ~= nil then return ui.st.sharp end   -- the preset builder's Switches option
	return ui.presetName == "Anime" or ui.presetName == "Arcade" or ui.presetName == "Manga"
end
local function lineW(ui) return (ui.presetName == "Manga" or ui.presetName == "Arcade") and 2 or 1 end

-- ------------------------------------------------------------ Toggle
function Tab:Toggle(label, default, cb)
	local ui = self.ui
	local item = { kind = "toggle", label = label, value = default and true or false }
	-- track, border, knob for a state (the mock's .tg / .tg.on per preset)
	local function colors(on)
		local tok, name = ui.tok, ui.presetName
		if name == "Manga" then return (on and tok.accent or tok.paper), tok.ink, tok.ink end
		if on and (name == "Anime" or name == "Arcade") then return tok.accent, tok.accent, tok.onAccent end
		if on and ui.st.spooky then return tok.text, tok.text, tok.bg end   -- Spooky: lit white, black knob
		if on then return tok.accent:Lerp(tok.bg, 0.72), tok.accent, tok.accent end
		return tok.off, tok.line2, tok.faint
	end
	local function paint(animate)
		if not (item.track and item.track.Parent) then return end
		local tb, bd, kn = colors(item.value)
		item.track.BackgroundColor3, item.tstroke.Color, item.knob.BackgroundColor3 = tb, bd, kn
		local pos = UDim2.new(0, item.value and item.tw - 14 - lineW(ui) + 1 or 2, 0.5, 0)
		local rot = 0
		if item.kw then   -- Haunted: the knob is a pumpkin that lights up
			pos = UDim2.new(0, item.value and item.tw - item.kw or 0, 0.5, 0)
			rot = item.value and 12 or 0
			if item.face then
				item.face:SetAttribute("on", item.value)
				item.face.ImageTransparency = item.value and 0 or 1
			end
		end
		if animate then ui:_tw(item.knob, 0.16, { Position = pos, Rotation = rot }, Enum.EasingStyle.Back) else item.knob.Position, item.knob.Rotation = pos, rot end
	end
	item.build = function(it, body)
		ui:_rowLabel(it, body, 42)
		local name = ui.presetName
		local haunt = ui.st.haunt
		local w, h = (name == "Arcade") and 36 or 34, (name == "Arcade") and 20 or 18
		if haunt then w, h = 40, 20 end
		local track = make("Frame", { AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, 0, 0.5, 0), Size = UDim2.fromOffset(w, h), BorderSizePixel = 0 }, body)
		local sq, soft = name == "Anime" or name == "Arcade", name == "Gamer"
		if ui.st.sharp ~= nil then sq, soft = ui.st.sharp, false end   -- the preset builder's Switches option
		make("UICorner", { CornerRadius = sq and UDim.new(0, 0) or (soft and UDim.new(0, 2) or UDim.new(0.5, 0)) }, track)
		local knob = make("Frame", { AnchorPoint = Vector2.new(0, 0.5), Size = UDim2.fromOffset(12, 12), BorderSizePixel = 0 }, track)
		make("UICorner", { CornerRadius = sq and UDim.new(0, 0) or (soft and UDim.new(0, 1) or UDim.new(0.5, 0)) }, knob)
		local stroke = make("UIStroke", { Thickness = lineW(ui), ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, track)
		it.track, it.knob, it.tstroke, it.tw = track, knob, stroke, w
		it.kw, it.face = nil, nil
		if haunt then
			it.kw = 20
			knob.Size = UDim2.fromOffset(20, 20)
			-- the plain knob stays until the pumpkin is in (a failed download keeps it)
			ui:_hauntSprite("pumpkin", { Size = UDim2.fromScale(1, 1), ZIndex = 2 }, knob, false, function() knob.BackgroundTransparency = 1 end)
			it.face = ui:_hauntSprite("pumpkin_face", { Size = UDim2.fromScale(1, 1), ZIndex = 3, ImageTransparency = 1 }, knob)
			if ui.haunt then ui.haunt.lit[it.face] = true end
		end
		ui.painted[#ui.painted + 1] = { track, "BackgroundColor3", function() return (colors(it.value)) end }
		ui.painted[#ui.painted + 1] = { stroke, "Color", function() return select(2, colors(it.value)) end }
		ui.painted[#ui.painted + 1] = { knob, "BackgroundColor3", function() return select(3, colors(it.value)) end }
		paint(false)
		local hit = make("TextButton", { Text = "", BackgroundTransparency = 1, Size = UDim2.fromScale(1, 1) }, body)
		hit.MouseButton1Click:Connect(function()
			it.value = not it.value
			paint(true)
			ui:_tick(it.value)
			if cb then task.spawn(pcall, cb, it.value) end
		end)
	end
	self:_add(item)
	item.handle = {
		Set = function(v, fire)
			item.value = v and true or false
			paint(true)
			if fire ~= false and cb then pcall(cb, item.value) end
		end,
		Get = function() return item.value end,
	}
	return item.handle
end

-- ------------------------------------------------------------ Slider
-- typeMax (optional): how high a TYPED number may go. The bar still ends at
-- max - a typed 2000 on a 0-500 bar just shows it full.
function Tab:Slider(label, min, max, default, cb, suffix, typeMax)
	local ui = self.ui
	local step = stepFor(min, max)
	local top = math.max(max, tonumber(typeMax) or max)
	local item = { kind = "slider", label = label, value = math.clamp(tonumber(default) or min, min, top) }
	local sfx = suffix and tostring(suffix) or ""
	local function show()
		if not (item.fill and item.fill.Parent) then return end
		local a = (max > min) and math.clamp((item.value - min) / (max - min), 0, 1) or 0
		item.fill.Size = UDim2.new(a, 0, 1, 0)
		item.thumb.Position = UDim2.new(a, 0, 0.5, 0)
		item.valueText.Text = fmtNum(item.value, step) .. sfx
	end
	item.build = function(it, body)
		local name = ui.presetName
		ui:text({ Text = label, Size = UDim2.new(1, -90, 0, 22), TextTruncate = Enum.TextTruncate.AtEnd }, body, "text", "body", 13)
		it.valueText = ui:text({ Text = "", AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, 0, 0, 0), Size = UDim2.fromOffset(88, 22),
			TextXAlignment = Enum.TextXAlignment.Right }, body, "dim", "mono", 12)
		local th = ({ Anime = 6, Gamer = 3, Manga = 9, Arcade = 6 })[name] or 4
		local track = make("Frame", { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 0, 1, -9), Size = UDim2.new(1, 0, 0, th), BorderSizePixel = 0 }, body)
		local flat = sharp(ui) or name == "Gamer"
		if name == "Manga" then
			ui:paint(track, "BackgroundColor3", "paper")
			ui:stroke(track, "ink", 2)
		else
			ui:paint(track, "BackgroundColor3", "off")
		end
		if not flat then make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, track) end
		it.fill = make("Frame", { Size = UDim2.new(0, 0, 1, 0), BorderSizePixel = 0 }, track)
		ui:paint(it.fill, "BackgroundColor3", "accent")
		if not flat then make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, it.fill) end
		local tsz = ({ Anime = { 10, 16 }, Manga = { 12, 20 }, Arcade = { 12, 16 } })[name] or { 13, 13 }
		it.thumb = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(tsz[1], tsz[2]), BorderSizePixel = 0, ZIndex = 2 }, track)
		ui:paint(it.thumb, "BackgroundColor3", name == "Manga" and "ink" or "text")
		make("UICorner", { CornerRadius = flat and UDim.new(0, name == "Gamer" and 1 or 0) or UDim.new(0.5, 0) }, it.thumb)
		if name ~= "Manga" and name ~= "Arcade" then
			local ts = make("UIStroke", { Thickness = 2, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, it.thumb)
			ui:paint(ts, "Color", "accent")
		end
		if ui.st.haunt then   -- Haunted: the handle is a candle flame sitting on the wax
			-- the plain handle stays until the flame is in (a failed download keeps it)
			local th = it.thumb
			local fi = ui:_hauntSprite("flame", { Size = UDim2.fromScale(1, 1), ZIndex = 3 }, th, false, function()
				th.BackgroundTransparency = 1
				th.AnchorPoint = Vector2.new(0.5, 0.8)
				th.Size = UDim2.fromOffset(14, 21)
				for _, c in ipairs(th:GetChildren()) do if c:IsA("UIStroke") or c:IsA("UICorner") then c:Destroy() end end
			end)
			if ui.haunt then ui.haunt.flames[fi] = true end
		end
		local grow = make("UIScale", { Scale = 1 }, it.thumb)
		show()
		-- click the number to type one (Discord 10-11: "change most of the
		-- sliders to number type in"): an unseen box over the value takes the
		-- typing and shows itself only while it has focus
		local box = make("TextBox", { Text = "", ClearTextOnFocus = false, BackgroundTransparency = 1, TextTransparency = 1,
			AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, 0, 0, 0), Size = UDim2.fromOffset(88, 22),
			TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 3 }, body)
		box.Focused:Connect(function()
			box.FontFace, box.TextSize, box.TextColor3 = it.valueText.FontFace, it.valueText.TextSize, it.valueText.TextColor3
			box.Text = fmtNum(it.value, step)
			box.TextTransparency, it.valueText.TextTransparency = 0, 1
		end)
		box.FocusLost:Connect(function()
			local v = tonumber((box.Text:gsub("[^%d%.%-]", "")))
			box.TextTransparency, it.valueText.TextTransparency = 1, 0
			box.Text = ""
			if not v then return end
			v = math.clamp(v, min, top)
			if v ~= it.value then
				it.value = v
				show()
				if cb then task.spawn(pcall, cb, v) end
			end
		end)
		local hit = make("TextButton", { Text = "", BackgroundTransparency = 1, Position = UDim2.new(0, -6, 1, -20), Size = UDim2.new(1, 12, 0, 20) }, body)
		local dragging = false
		local function setFrom(x)
			local a = math.clamp((x - track.AbsolutePosition.X) / math.max(1, track.AbsoluteSize.X), 0, 1)
			local v = min + (max - min) * a
			v = math.clamp(math.floor(v / step + 0.5) * step, min, max)
			if v ~= it.value then
				it.value = v
				show()
				if cb then task.spawn(pcall, cb, v) end
			end
		end
		hit.InputBegan:Connect(function(i)
			if isClick(i) then
				dragging = true
				ui:_tw(grow, 0.12, { Scale = 1.25 }, Enum.EasingStyle.Back)
				setFrom(i.Position.X)
			end
		end)
		ui:bconn(UserInputService.InputChanged:Connect(function(i)
			if dragging and isMove(i) then setFrom(i.Position.X) end
		end))
		ui:bconn(UserInputService.InputEnded:Connect(function(i)
			if dragging and isClick(i) then
				dragging = false
				ui:_tw(grow, 0.12, { Scale = 1 })
			end
		end))
	end
	self:_add(item)
	item.handle = {
		Get = function() return item.value end,
		Set = function(v)
			item.value = math.clamp(tonumber(v) or item.value, min, top)
			show()
		end,
	}
	return item.handle
end

-- ------------------------------------------------------------ Dropdown / Cycle / PlayerList
-- the chip a dropdown / colour / keybind shows on the right of its row
function Win:_chip(body, width, height)
	local chip = make("TextButton", { Text = "", AutoButtonColor = false, AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, 0, 0.5, 0),
		Size = width, BorderSizePixel = 0 }, body)
	if height then chip.Size = UDim2.new(width.X.Scale, width.X.Offset, 0, height) end
	self:_fill(chip, "sCtl", "tCtl", nil, false, nil, "pill")
	local st = self:stroke(chip, "ln2", lineW(self))
	chip.MouseEnter:Connect(function() st.Color = self.tok.faint end)
	chip.MouseLeave:Connect(function() st.Color = self.tok.ln2 end)
	return chip, st
end

function Tab:Dropdown(label, values, default, cb)
	local ui = self.ui
	local item = { kind = "dropdown", label = label, values = dedupe(values) }
	item.value = default ~= nil and tostring(default) or item.values[1]
	local function show() if item.chipText then item.chipText.Text = item.value or "-" end end
	item.build = function(it, body)
		local chip = ui:_chip(body, UDim2.new(0.5, 0, 0, 27))
		make("UISizeConstraint", { MinSize = Vector2.new(126, 0) }, chip)
		it.chipText = ui:text({ Text = "", Position = UDim2.fromOffset(10, 0), Size = UDim2.new(1, -28, 1, 0), TextTruncate = Enum.TextTruncate.AtEnd }, chip, "ctlText", "body", 12)
		ui:text({ Text = "\u{25BC}", AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -9, 0, 0), Size = UDim2.fromOffset(10, 27),
			TextXAlignment = Enum.TextXAlignment.Right }, chip, "faint", "body", 9)
		ui:text({ Text = label, Size = UDim2.new(0.5, -10, 1, 0), TextTruncate = Enum.TextTruncate.AtEnd }, body, "text", "body", 13)
		show()
		chip.MouseButton1Click:Connect(function()
			-- re-picking the shown value still fires: a PlayerList shows its
			-- first name before anyone has chosen it
			ui:_openPopover(chip, it.values, it.value, function(v)
				it.value = v
				show()
				ui:_tick(true)
				if cb then task.spawn(pcall, cb, v) end
			end, it)
		end)
	end
	self:_add(item)
	item.handle = {
		Get = function() return item.value end,
		Set = function(v)
			item.value = tostring(v)
			show()
		end,
		-- new options; keeps `keep` if it is still there, else the first (as
		-- WindLib - ABA's variant refresh relies on the reset). Never fires the
		-- callback (refilling a list is not the user picking).
		SetItems = function(newValues, keep)
			item.values = dedupe(newValues)
			local want = keep ~= nil and tostring(keep) or nil
			item.value = (want and table.find(item.values, want)) and want or item.values[1]
			if ui.popOwner == item then ui:_closePopover() end
			show()
		end,
		Close = function() ui:_closePopover() end,
	}
	item.dd = item.handle
	return item.handle
end

function Tab:Cycle(label, values, default, cb)
	local d = self:Dropdown(label, values, default, cb)
	return { Get = d.Get, Set = d.Set }
end

function Tab:PlayerList(height, onPick)
	local selected
	local function names()
		local out = {}
		for _, p in ipairs(Players:GetPlayers()) do if p ~= LocalPlayer then out[#out + 1] = p.Name end end
		table.sort(out)
		return out
	end
	local d = self:Dropdown("Player", names(), nil, function(v)
		selected = v
		if onPick then pcall(onPick, v) end
	end)
	local function refresh() d.SetItems(names(), selected) end
	self.ui:conn(Players.PlayerAdded:Connect(refresh))
	self.ui:conn(Players.PlayerRemoving:Connect(refresh))
	return { Refresh = refresh, Selected = function() return selected end }
end

-- The shared popover: one at a time, closes on an outside click, a pick, a
-- tab change or Escape. Options are only built while it is open.
function Win:_closePopover()
	self.popOwner = nil
	if self.pop then
		pcall(function() self.pop:Destroy() end)
		self.pop = nil
	end
	if self.popShield then
		pcall(function() self.popShield:Destroy() end)
		self.popShield = nil
	end
end
-- An element's box in overlay coordinates, plus the overlay's size. Measured
-- against the overlay itself: AbsolutePosition leaves out the top-bar inset
-- even in an IgnoreGuiInset gui, so raw values land ~58 px high.
function Win:_overlaySpace(inst)
	local s = self.uiScale
	local o = self.overlay.AbsolutePosition
	return (inst.AbsolutePosition - o) / s, inst.AbsoluteSize / s, self.overlay.AbsoluteSize / s
end
-- an empty popover frame under (or above) `anchor`
function Win:_popFrame(anchor, w, h, owner)
	self:_closePopover()
	self.popOwner = owner
	local shield = make("TextButton", { Text = "", BackgroundTransparency = 1, AutoButtonColor = false, Size = UDim2.fromScale(1, 1), ZIndex = 50 }, self.overlay)
	shield.MouseButton1Click:Connect(function() self:_closePopover() end)
	self.popShield = shield
	local ap, as, vp = self:_overlaySpace(anchor)
	local x = math.clamp(ap.X + as.X - w, 4, math.max(4, vp.X - w - 4))
	local y = ap.Y + as.Y + 4
	if y + h > vp.Y - 8 then y = ap.Y - h - 4 end
	local pop = make("Frame", { Position = UDim2.fromOffset(x, y - 4), Size = UDim2.fromOffset(w, h), BorderSizePixel = 0, ZIndex = 51 }, self.overlay)
	self:_fill(pop, "sPop", "tPop", nil, false, nil, 0.6)
	self:stroke(pop, "ln2", lineW(self))
	self:_tw(pop, 0.12, { Position = UDim2.fromOffset(x, y) })
	self.pop = pop
	return pop
end
function Win:_openPopover(anchor, values, current, onPick, owner)
	local searchable = #values > 10
	local rowsShown = math.min(#values, 9)
	local h = rowsShown * 28 + 8 + (searchable and 34 or 0)
	local _, as = self:_overlaySpace(anchor)
	local pop = self:_popFrame(anchor, math.max(as.X, 180), h, owner)
	local name = self.presetName
	local filter = ""
	local list = make("ScrollingFrame", { Position = UDim2.fromOffset(4, searchable and 38 or 4), Size = UDim2.new(1, -8, 1, searchable and -42 or -8),
		BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3, CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y, ZIndex = 52 }, pop)
	self:paint(list, "ScrollBarImageColor3", "line2")
	make("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder }, list)
	local fontOf = owner and owner.fontOf
	local function fill()
		for _, c in ipairs(list:GetChildren()) do if c:IsA("GuiButton") then c:Destroy() end end
		local f = string.lower(filter)
		for i, v in ipairs(values) do
			if f == "" or string.find(string.lower(v), f, 1, true) then
				local on = v == current
				local b = make("TextButton", { Text = "", AutoButtonColor = false, Size = UDim2.new(1, 0, 0, 28), BorderSizePixel = 0, LayoutOrder = i, ZIndex = 53,
					BackgroundColor3 = (on and name == "Manga") and self.tok.accent or self.tok.sHover,
					BackgroundTransparency = (on and name == "Manga") and 0 or 1 }, list)
				make("UICorner", { CornerRadius = UDim.new(0, math.floor((self.look.rad or 0) * 0.4 + 0.5)) }, b)
				local t = make("TextLabel", { Text = v, BackgroundTransparency = 1, Position = UDim2.fromOffset(10, 0), Size = UDim2.new(1, -20, 1, 0),
					FontFace = fontOf and fontOf(v) or self:_face("body"), TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left,
					TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 54 }, b)
				local idle = on and ((name == "Manga") and self.tok.ink or self.tok.accent) or ((name == "Manga") and self.tok.ink or self.tok.dim)
				t.TextColor3 = idle
				b.MouseEnter:Connect(function()
					if not (on and name == "Manga") then b.BackgroundTransparency = self.tok.tHover end
					if not on then t.TextColor3 = self.tok.ctlText end
				end)
				b.MouseLeave:Connect(function()
					if not (on and name == "Manga") then b.BackgroundTransparency = 1 end
					t.TextColor3 = idle
				end)
				b.MouseButton1Click:Connect(function()
					self:_closePopover()
					onPick(v)
				end)
			end
		end
	end
	if searchable then
		local box = make("TextBox", { Text = "", PlaceholderText = "Filter", ClearTextOnFocus = false, Position = UDim2.fromOffset(6, 6), Size = UDim2.new(1, -12, 0, 26),
			BorderSizePixel = 0, FontFace = self:_face("body"), TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 53 }, pop)
		box.BackgroundColor3 = self.tok.sCtl
		box.BackgroundTransparency = math.min(self.tok.tCtl, 0.5)
		box.TextColor3 = self.tok.ctlText
		box.PlaceholderColor3 = self.tok.faint
		make("UICorner", { CornerRadius = UDim.new(0, math.floor((self.look.rad or 0) * 0.5 + 0.5)) }, box)
		pad(box, 8, 8, 0, 0)
		box:GetPropertyChangedSignal("Text"):Connect(function()
			filter = box.Text
			fill()
		end)
	end
	fill()
end

-- ------------------------------------------------------------ Button
-- TeekConfig writes btn.Text to relabel a row and checks IsA, so the handle
-- answers both the way the old TextButton did.
function Tab:Button(label, primary, cb)
	local ui = self.ui
	local item = { kind = "button", label = label, text = label }
	local function shown() return ui.st.btn.upper and string.upper(item.text) or item.text end
	item.build = function(it, body)
		local name, bs = ui.presetName, ui.st.btn
		local shadowTok = (name == "Manga" and "ink") or ((name == "Anime" and primary) and "text") or nil
		local props = { AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 0, 0.5, 0), Size = UDim2.new(1, shadowTok and -4 or 0, 0, name == "Arcade" and 32 or 30) }
		local face
		if shadowTok then
			local _, f = ui:_shadowed(body, props, shadowTok, name == "Manga" and 3 or 4, name == "Manga" and 3 or 4)
			face = f
		else
			face = make("Frame", props, body)
		end
		if primary then
			ui:_fill(face, "accent", 0, nil, false, nil, "pill")
		else
			ui:_fill(face, name == "Manga" and "paper" or "sCtl", name == "Grimoire" and 1 or "tCtl", nil, false, nil, "pill")
		end
		local st
		if name ~= "Arcade" then st = ui:stroke(face, primary and "accent" or (name == "Manga" and "ink" or "ln2"), lineW(ui)) end
		if name == "Arcade" then
			-- 8-bit bevel: dark bottom/right, light top/left
			for _, e in ipairs({ { 0, 1, 1, 0, 0, 3, BLACK, 0.65 }, { 1, 0, 0, 1, 3, 0, BLACK, 0.65 }, { 0, 0, 1, 0, 0, 3, WHITE, 0.88 }, { 0, 0, 0, 1, 3, 0, WHITE, 0.88 } }) do
				make("Frame", { AnchorPoint = Vector2.new(e[1], e[2]), Position = UDim2.fromScale(e[1], e[2]), Size = UDim2.new(e[3], e[5], e[4], e[6]),
					BackgroundColor3 = e[7], BackgroundTransparency = e[8], BorderSizePixel = 0, ZIndex = 2 }, face)
			end
		end
		local press = make("UIScale", { Scale = 1 }, face)
		it.textLabel = ui:text({ Text = shown(), Size = UDim2.fromScale(1, 1), TextXAlignment = Enum.TextXAlignment.Center,
			TextTruncate = Enum.TextTruncate.AtEnd, ZIndex = 3 }, face, primary and "onAccent" or (name == "Manga" and "ink" or "ctlText"), bs.role, bs.size)
		local b = make("TextButton", { Text = "", AutoButtonColor = false, BackgroundTransparency = 1, Size = UDim2.fromScale(1, 1), ZIndex = 4 }, face)
		b.MouseEnter:Connect(function() if st and not primary then st.Color = ui.tok.faint end end)
		b.MouseLeave:Connect(function()
			if st and not primary then st.Color = ui.tok.ln2 end
			ui:_tw(press, 0.1, { Scale = 1 })
		end)
		b.MouseButton1Down:Connect(function() ui:_tw(press, 0.08, { Scale = 0.97 }) end)
		b.MouseButton1Up:Connect(function() ui:_tw(press, 0.14, { Scale = 1 }, Enum.EasingStyle.Back) end)
		b.MouseButton1Click:Connect(function()
			ui:_tick(true)
			if cb then task.spawn(pcall, cb) end
		end)
	end
	self:_add(item)
	local function relabel(v)
		item.text = tostring(v)
		if item.textLabel then item.textLabel.Text = shown() end
	end
	item.handle = setmetatable({}, {
		__index = function(_, k)
			if k == "Text" then return item.text end
			if k == "IsA" then return function(_, cls) return cls == "TextButton" or cls == "GuiObject" end end
			if k == "SetTitle" then return function(_, t) relabel(t) end end
			return nil
		end,
		__newindex = function(_, k, v) if k == "Text" then relabel(v) end end,
	})
	return item.handle
end

-- ------------------------------------------------------------ Input
-- (placeholder, height-or-initial-text, cb). Fires only when the user leaves
-- the box - never while it is being built.
function Tab:Input(placeholder, second, cb)
	local ui = self.ui
	local item = { kind = "input", label = placeholder, text = type(second) == "string" and second or "" }
	item.build = function(it, body)
		local box = make("TextBox", { Text = it.text, PlaceholderText = placeholder or "", ClearTextOnFocus = false, AnchorPoint = Vector2.new(0, 0.5),
			Position = UDim2.new(0, 0, 0.5, 0), Size = UDim2.new(1, 0, 0, 28), BorderSizePixel = 0, TextSize = 13, TextXAlignment = Enum.TextXAlignment.Left,
			TextTruncate = Enum.TextTruncate.AtEnd }, body)
		ui:_fill(box, ui.presetName == "Manga" and "paper" or "sCtl", "tCtl", nil, false, nil, 0.6)
		ui:paint(box, "TextColor3", "ctlText")
		ui:paint(box, "PlaceholderColor3", "faint")
		ui:paint(box, "FontFace", function() return ui:_face("body") end)
		local st = ui:stroke(box, ui.presetName == "Manga" and "ink" or "ln2", lineW(ui))
		pad(box, 10, 10, 0, 0)
		it.box = box
		box.Focused:Connect(function() st.Color = ui.tok.accent end)
		box.FocusLost:Connect(function()
			st.Color = ui.presetName == "Manga" and ui.tok.ink or ui.tok.ln2
			it.text = box.Text
			if cb then task.spawn(pcall, cb, it.text) end
		end)
	end
	self:_add(item)
	item.handle = setmetatable({}, {
		__index = function(_, k)
			if k == "Text" then return item.text end
			if k == "IsA" then return function(_, cls) return cls == "TextBox" or cls == "GuiObject" end end
			return nil
		end,
		__newindex = function(_, k, v)
			if k == "Text" then
				item.text = tostring(v)
				if item.box then item.box.Text = item.text end
			end
		end,
	})
	return item.handle
end

-- ------------------------------------------------------------ Stat
function Tab:Stat(label, initial)
	local ui = self.ui
	local item = { kind = "stat", label = label, value = tostring(initial or "-") }
	item.build = function(it, body)
		ui:text({ Text = label, Size = UDim2.new(0.45, 0, 1, 0), TextTruncate = Enum.TextTruncate.AtEnd }, body, "dim", "body", 13)
		it.valueLabel = ui:text({ Text = it.value, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, 0, 0, 0), Size = UDim2.new(0.55, 0, 1, 0),
			TextXAlignment = Enum.TextXAlignment.Right, TextTruncate = Enum.TextTruncate.AtEnd }, body, "text", "mono", 12)
		ui.painted[#ui.painted + 1] = { it.valueLabel, "TextColor3", function() return it.color or ui.tok.text end }
		if it.color then it.valueLabel.TextColor3 = it.color end
	end
	self:_add(item)
	self.last = nil   -- a Label after a Stat must not become its description
	item.handle = {
		-- color is optional (ABA tints its state rows); nil goes back to the theme
		Set = function(t, color, extra)
			-- stat:Set(text[, color]) and stat.Set(text[, color]) both work
			if t == item.handle then t, color = color, extra end
			if type(color) ~= "userdata" then color = nil end
			local v = tostring(t)
			if v == item.value and color == item.color then return end
			item.value, item.color = v, color
			if item.valueLabel then
				item.valueLabel.Text = v
				item.valueLabel.TextColor3 = color or ui.tok.text
			end
		end,
	}
	return item.handle
end

-- ------------------------------------------------------------ Keybind
-- (label, default KeyCode name or "None", cb(name)). Click, press a key;
-- Escape cancels, Backspace clears.
function Tab:Keybind(label, default, cb)
	local ui = self.ui
	local item = { kind = "keybind", label = label, value = default or "None" }
	item.build = function(it, body)
		ui:_rowLabel(it, body, 110)
		local chip, st = ui:_chip(body, UDim2.fromOffset(100, 24))
		it.chipText = ui:text({ Text = keyLabel(it.value), Size = UDim2.fromScale(1, 1), TextXAlignment = Enum.TextXAlignment.Center }, chip, "dim", "mono", 11)
		chip.MouseButton1Click:Connect(function()
			if it.listening then return end
			it.listening = true
			it.chipText.Text = "..."
			it.chipText.TextColor3 = ui.tok.accent
			st.Color = ui.tok.accent
			-- keyboard only (the menu key listens to the keyboard); owned by the
			-- window so a teardown mid-capture can't leave it firing
			local c
			c = ui:conn(UserInputService.InputBegan:Connect(function(i)
				if i.UserInputType ~= Enum.UserInputType.Keyboard then return end
				local name = i.KeyCode.Name
				c:Disconnect()
				it.listening = false
				st.Color = ui.tok.ln2
				it.chipText.TextColor3 = ui.tok.dim
				if name == "Escape" then
					it.chipText.Text = keyLabel(it.value)
					return
				end
				if name == "Backspace" then name = "None" end
				it.value = name
				it.chipText.Text = keyLabel(name)
				if cb then task.spawn(pcall, cb, name) end
			end))
		end)
	end
	self:_add(item)
	item.handle = {
		Get = function() return item.value end,
		Set = function(v)
			item.value = tostring(v)
			if item.chipText then item.chipText.Text = keyLabel(item.value) end
		end,
	}
	return item.handle
end

-- ------------------------------------------------------------ Color (new in v2)
-- (label, default Color3 or "#rrggbb", cb(Color3)). A swatch chip; the picker
-- is a popover with swatches, hue / saturation / brightness bars and a hex box.
local SWATCHES = { "#7cc4ff", "#4fd68b", "#ff4d5e", "#a78bfa", "#e0b25a", "#ff8a3d", "#ff78a0", "#dce0e6" }
function Tab:Color(label, default, cb, swatches)
	local ui = self.ui
	local item = { kind = "color", label = label, value = hex(default) or DEFAULT_ACCENT, swatches = swatches }
	local function show() if item.swatch then item.swatch.BackgroundColor3 = item.value end end
	item.build = function(it, body)
		ui:_rowLabel(it, body, 56)
		local chip = ui:_chip(body, UDim2.fromOffset(44, 24))
		it.swatch = make("Frame", { Position = UDim2.fromOffset(3, 3), Size = UDim2.new(1, -6, 1, -6), BorderSizePixel = 0 }, chip)
		ui:corner(it.swatch, 0.4)
		show()
		chip.MouseButton1Click:Connect(function()
			ui:_openColor(chip, it, function(c)
				it.value = c
				show()
				if cb then task.spawn(pcall, cb, c) end
			end)
		end)
	end
	self:_add(item)
	item.handle = {
		Get = function() return item.value end,
		Set = function(v)
			item.value = hex(v) or item.value
			show()
		end,
	}
	return item.handle
end

function Win:_openColor(anchor, item, onPick)
	local pop = self:_popFrame(anchor, 232, 186, item)
	local h, s, v = item.value:ToHSV()
	local hexBox
	local bars = {}
	local function push()
		local c = Color3.fromHSV(h, s, v)
		for _, b in ipairs(bars) do b.refresh() end
		if hexBox and not hexBox:IsFocused() then hexBox.Text = toHex(c) end
		onPick(c)
	end
	-- swatches
	local row = make("Frame", { Position = UDim2.fromOffset(10, 10), Size = UDim2.new(1, -20, 0, 20), BackgroundTransparency = 1, ZIndex = 52 }, pop)
	make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 6) }, row)
	for _, sw in ipairs(item.swatches or SWATCHES) do
		local c = hex(sw) or WHITE
		local b = make("TextButton", { Text = "", AutoButtonColor = false, Size = UDim2.fromOffset(20, 20), BackgroundColor3 = c, BorderSizePixel = 0, ZIndex = 53 }, row)
		make("UICorner", { CornerRadius = UDim.new(0, 4) }, b)
		b.MouseButton1Click:Connect(function()
			h, s, v = c:ToHSV()
			push()
		end)
	end
	-- H / S / V bars
	local function bar(y, label, keys, get, set)
		self:text({ Text = label, Position = UDim2.fromOffset(10, y), Size = UDim2.fromOffset(14, 14), ZIndex = 52 }, pop, "faint", "mono", 10)
		local track = make("Frame", { Position = UDim2.fromOffset(28, y), Size = UDim2.new(1, -38, 0, 14), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 52 }, pop)
		make("UICorner", { CornerRadius = UDim.new(0, 3) }, track)
		local grad = make("UIGradient", {}, track)
		local knob = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(6, 18), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 53 }, track)
		make("UIStroke", { Color = BLACK, Thickness = 1 }, knob)
		local b = {}
		b.refresh = function()
			grad.Color = keys()
			knob.Position = UDim2.new(get(), 0, 0.5, 0)
		end
		local hit = make("TextButton", { Text = "", BackgroundTransparency = 1, Size = UDim2.fromScale(1, 1), ZIndex = 54 }, track)
		local drag = false
		local function at(x)
			set(math.clamp((x - track.AbsolutePosition.X) / math.max(1, track.AbsoluteSize.X), 0, 1))
			push()
		end
		hit.InputBegan:Connect(function(i)
			if isClick(i) then
				drag = true
				at(i.Position.X)
			end
		end)
		local c1 = UserInputService.InputChanged:Connect(function(i) if drag and isMove(i) then at(i.Position.X) end end)
		local c2 = UserInputService.InputEnded:Connect(function(i) if isClick(i) then drag = false end end)
		track.Destroying:Connect(function()
			c1:Disconnect()
			c2:Disconnect()
		end)
		bars[#bars + 1] = b
		b.refresh()
	end
	local hueKeys = {}
	for i = 0, 6 do hueKeys[#hueKeys + 1] = ColorSequenceKeypoint.new(i / 6, Color3.fromHSV((i / 6) % 1, 1, 1)) end
	bar(42, "H", function() return ColorSequence.new(hueKeys) end, function() return h end, function(x) h = math.min(x, 0.999) end)
	bar(66, "S", function() return ColorSequence.new(Color3.fromHSV(h, 0, v), Color3.fromHSV(h, 1, v)) end, function() return s end, function(x) s = x end)
	bar(90, "V", function() return ColorSequence.new(BLACK, Color3.fromHSV(h, s, 1)) end, function() return v end, function(x) v = x end)
	-- hex
	hexBox = make("TextBox", { Text = toHex(item.value), ClearTextOnFocus = false, Position = UDim2.fromOffset(10, 118), Size = UDim2.new(1, -20, 0, 26),
		BorderSizePixel = 0, FontFace = F.mono, TextSize = 12, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 53 }, pop)
	hexBox.BackgroundColor3 = self.tok.sCtl
	hexBox.BackgroundTransparency = math.min(self.tok.tCtl, 0.5)
	hexBox.TextColor3 = self.tok.ctlText
	make("UICorner", { CornerRadius = UDim.new(0, 4) }, hexBox)
	pad(hexBox, 8, 8, 0, 0)
	hexBox.FocusLost:Connect(function()
		local c = hex(hexBox.Text)
		if c then
			h, s, v = c:ToHSV()
			push()
		else
			hexBox.Text = toHex(Color3.fromHSV(h, s, v))
		end
	end)
	self:text({ Text = "Click outside to close", Position = UDim2.fromOffset(10, 152), Size = UDim2.new(1, -20, 0, 22), ZIndex = 52 }, pop, "faint", "body", 11)
end

-- ------------------------------------------------------------ Segment (new in v2)
-- (label, values, default, cb(value)): a row of buttons, one lit.
function Tab:Segment(label, values, default, cb)
	local ui = self.ui
	local item = { kind = "segment", label = label, values = dedupe(values) }
	item.value = default ~= nil and tostring(default) or item.values[1]
	local function show()
		if not (item.btns and item.box and item.box.Parent) then return end
		for v, b in pairs(item.btns) do
			local on = v == item.value
			b.BackgroundTransparency = on and 0 or 1
			b.TextColor3 = on and ui.tok.onAccent or ui.tok.dim
		end
	end
	item.build = function(it, body)
		ui:text({ Text = label, Size = UDim2.new(1, 0, 0, 22), TextTruncate = Enum.TextTruncate.AtEnd }, body, "text", "body", 13)
		local box = make("Frame", { AnchorPoint = Vector2.new(0, 1), Position = UDim2.new(0, 0, 1, -4), Size = UDim2.new(1, 0, 0, 27), BackgroundTransparency = 1, ClipsDescendants = true }, body)
		ui:corner(box, "pill")
		ui:stroke(box, ui.presetName == "Manga" and "ink" or "ln2", lineW(ui))
		it.box, it.btns = box, {}
		local n = #it.values
		for i, v in ipairs(it.values) do
			local b = make("TextButton", { Text = v, AutoButtonColor = false, Position = UDim2.new((i - 1) / n, 0, 0, 0), Size = UDim2.new(1 / n, 0, 1, 0),
				BorderSizePixel = 0, TextSize = 12 }, box)
			ui:paint(b, "BackgroundColor3", "accent")
			ui:paint(b, "FontFace", function() return ui:_face("semi") end)
			ui:corner(b, "pill")
			if i > 1 then
				local sep = make("Frame", { Size = UDim2.new(0, 1, 1, 0), BorderSizePixel = 0 }, b)
				ui:paint(sep, "BackgroundColor3", "ln2")
				ui:paint(sep, "BackgroundTransparency", "tln2")
			end
			it.btns[v] = b
			ui.painted[#ui.painted + 1] = { b, "TextColor3", function() return v == it.value and ui.tok.onAccent or ui.tok.dim end }
			b.MouseButton1Click:Connect(function()
				it.value = v
				show()
				ui:_tick(true)
				if cb then task.spawn(pcall, cb, v) end
			end)
		end
		show()
	end
	self:_add(item)
	item.handle = {
		Get = function() return item.value end,
		Set = function(v)
			item.value = tostring(v)
			show()
		end,
	}
	return item.handle
end

function Tab:Destroy()
	if self.page then pcall(function() self.page:Destroy() end) end
end

-- ============================================================
--  Search: over every element's label, built or not. Picking a result opens
--  only that tab, then scrolls to the row once it has laid out.
-- ============================================================
function Win:_search(q)
	if self.results then
		pcall(function() self.results:Destroy() end)
		self.results = nil
	end
	q = string.lower(q or "")
	if #q < 2 then return end
	local hits = {}
	for _, e in ipairs(self.index) do
		if string.find(string.lower(e.label), q, 1, true) then
			hits[#hits + 1] = e
			if #hits >= 10 then break end
		end
	end
	local box = self.searchBox
	local ap, as = self:_overlaySpace(box.Parent)
	local w = 300
	local res = make("Frame", { Position = UDim2.fromOffset(ap.X, ap.Y + as.Y + 6), Size = UDim2.fromOffset(w, math.max(1, #hits) * 34 + 8),
		BorderSizePixel = 0, ZIndex = 60 }, self.overlay)
	self:_fill(res, "sPop", "tPop", nil, false, nil, 0.6)
	self:stroke(res, "ln2", lineW(self))
	make("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder }, res)
	pad(res, 4, 4, 4, 4)
	self.results = res
	if #hits == 0 then
		self:text({ Text = "Nothing matches that.", Size = UDim2.new(1, 0, 0, 32), ZIndex = 61, TextXAlignment = Enum.TextXAlignment.Center }, res, "faint", "mono", 12)
		return
	end
	for i, e in ipairs(hits) do
		local b = make("TextButton", { Text = "", AutoButtonColor = false, Size = UDim2.new(1, 0, 0, 34), BorderSizePixel = 0, LayoutOrder = i, ZIndex = 61,
			BackgroundColor3 = self.tok.sHover, BackgroundTransparency = 1 }, res)
		make("UICorner", { CornerRadius = UDim.new(0, math.floor((self.look.rad or 0) * 0.4 + 0.5)) }, b)
		self:text({ Text = e.label, Position = UDim2.fromOffset(10, 3), Size = UDim2.new(1, -20, 0, 16), ZIndex = 62, TextTruncate = Enum.TextTruncate.AtEnd }, b, "ctlText", "body", 13)
		local where = (e.tab.entry.title ~= e.tab.name) and (e.tab.entry.title .. "  \u{203A}  " .. e.tab.name) or e.tab.name
		self:text({ Text = where, Position = UDim2.fromOffset(10, 18), Size = UDim2.new(1, -20, 0, 13), ZIndex = 62 }, b, "faint", "body", 11)
		b.MouseEnter:Connect(function() b.BackgroundTransparency = self.tok.tHover end)
		b.MouseLeave:Connect(function() b.BackgroundTransparency = 1 end)
		b.MouseButton1Click:Connect(function()
			self.searchBox.Text = ""
			self:_selectEntry(e.tab.entry, e.tab)
			task.defer(function()
				task.wait()
				local row = e.item.row
				if row and row.Parent and e.tab.page then
					local page = e.tab.page
					page.CanvasPosition = Vector2.new(0, math.max(0, row.AbsolutePosition.Y - page.AbsolutePosition.Y + page.CanvasPosition.Y - 20))
					local hl = make("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = self.tok.accent, BackgroundTransparency = 0.85, BorderSizePixel = 0, ZIndex = 9 }, row)
					make("UICorner", { CornerRadius = UDim.new(0, 5) }, hl)
					TweenService:Create(hl, TweenInfo.new(1.2), { BackgroundTransparency = 1 }):Play()
					task.delay(1.3, function() pcall(function() hl:Destroy() end) end)
				end
			end)
		end)
	end
end

-- ============================================================
--  Notifications
-- ============================================================
local KIND_TOKEN = { good = "good", warn = "warn", bad = "bad" }
function Win:Notify(text, kind, seconds)
	if self.silent or self.dead then return end
	local tok, name = self.tok, self.presetName
	-- newest at the bottom, at most 4
	local notes = {}
	for _, c in ipairs(self.notes:GetChildren()) do if c:IsA("Frame") then notes[#notes + 1] = c end end
	table.sort(notes, function(a, b) return a.LayoutOrder < b.LayoutOrder end)
	for i = 1, #notes - 3 do pcall(function() notes[i]:Destroy() end) end
	self.noteN = (self.noteN or 0) + 1
	local holder = make("Frame", { Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1, LayoutOrder = self.noteN }, self.notes)
	local shadow = ({ Anime = { 6, tok.accent, 0 }, Manga = { 4, tok.ink, 0 }, Arcade = { 6, BLACK, 0.55 } })[name]
	-- box slides; the hard shadow is the card's sibling so it draws behind it
	local box = make("Frame", { Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BackgroundTransparency = 1, Position = UDim2.fromOffset(30, 0) }, holder)
	local card = make("Frame", { Size = UDim2.new(1, -(shadow and shadow[1] or 0), 0, 0), AutomaticSize = Enum.AutomaticSize.Y, BorderSizePixel = 0, BackgroundColor3 = tok.sNote,
		BackgroundTransparency = tok.tNote, ZIndex = 2 }, box)
	local fades = { { card, "BackgroundTransparency", tok.tNote } }
	if shadow then
		local sh = make("Frame", { Position = UDim2.fromOffset(shadow[1], shadow[1]), Size = UDim2.new(1, -shadow[1], 1, -shadow[1]), BackgroundColor3 = shadow[2],
			BackgroundTransparency = shadow[3], BorderSizePixel = 0, ZIndex = 1 }, box)
		make("UIPadding", { PaddingBottom = UDim.new(0, shadow[1]) }, box)
		fades[#fades + 1] = { sh, "BackgroundTransparency", shadow[3] }
	end
	local r = (name == "Glass" and 16) or (name == "Manga" and 22) or math.floor((self.look.rad or 0) * 0.8 + 0.5)
	make("UICorner", { CornerRadius = UDim.new(0, r) }, card)
	local sc = ({ Gamer = { tok.accent, 1, 0 }, Manga = { tok.ink, 2, 0 }, Arcade = { tok.accent, 3, 0 }, Grimoire = { tok.gold, 1, 0 }, Glass = { WHITE, 1, 0.86 } })[name]
		or { tok.line2, 1, 0 }
	local st = make("UIStroke", { Color = sc[1], Thickness = sc[2], Transparency = sc[3] }, card)
	fades[#fades + 1] = { st, "Transparency", sc[3] }
	local barW = (name == "Anime" and 6) or ((name == "Sleek") and 3) or 0
	if barW > 0 then
		local bar = make("Frame", { Size = UDim2.new(0, barW, 1, 0), BackgroundColor3 = tok[KIND_TOKEN[kind] or "accent"], BorderSizePixel = 0, ZIndex = 3 }, card)
		fades[#fades + 1] = { bar, "BackgroundTransparency", 0 }
	end
	pad(card, 14 + barW, 14, 9, 10)
	local hs = ({ Anime = 12, Manga = 17, Arcade = 8, Grimoire = 14 })[name] or 10
	local head = string.upper(self.subtitle ~= "Universal" and self.subtitle or self.title)
	if name == "Manga" or name == "Grimoire" then head = self.subtitle ~= "Universal" and self.subtitle or self.title end
	local headCol = (KIND_TOKEN[kind] and tok[KIND_TOKEN[kind]]) or tok.noteHead
	local t1 = make("TextLabel", { Text = (hs <= 10 and name ~= "Arcade") and spaced(head) or head, BackgroundTransparency = 1, Size = UDim2.new(1, 0, 0, hs + 4),
		FontFace = name == "Gamer" and F.mono or self:_face("head"), TextSize = hs, TextColor3 = headCol, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 3 }, card)
	local body = make("TextLabel", { Text = tostring(text), BackgroundTransparency = 1, Position = UDim2.fromOffset(0, hs + 6), Size = UDim2.new(1, 0, 0, 0),
		AutomaticSize = Enum.AutomaticSize.Y, TextWrapped = true, FontFace = self:_face("body"), TextSize = 13, TextColor3 = tok.noteText,
		TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top, ZIndex = 3 }, card)
	make("UIPadding", { PaddingBottom = UDim.new(0, 2) }, body)
	self:_tw(box, 0.18, { Position = UDim2.fromOffset(0, 0) }, Enum.EasingStyle.Back)
	task.delay(seconds or self.notifySeconds or 4, function()
		if not card.Parent then return end
		for _, f in ipairs(fades) do pcall(function() TweenService:Create(f[1], TweenInfo.new(0.15), { [f[2]] = 1 }):Play() end) end
		for _, l in ipairs({ t1, body }) do TweenService:Create(l, TweenInfo.new(0.15), { TextTransparency = 1 }):Play() end
		TweenService:Create(box, TweenInfo.new(0.15), { Position = UDim2.fromOffset(16, 0) }):Play()
		task.wait(0.17)
		pcall(function() holder:Destroy() end)
	end)
end

-- ============================================================
--  Show / hide, menu key, mobile button
-- ============================================================
function Win:_windowOpen() return self.root ~= nil and self.root.Visible end

function Win:Show()
	if self.dead then return end
	local was = self.root.Visible
	self.opened = true
	self.root.Visible = true
	if not self.current then
		local first = self.rail[1]
		if first then self:_selectEntry(first) end
	end
	if not was and not self.growing then
		local p = self.root.Position
		self.root.Position = p + UDim2.fromOffset(0, 10)
		self:_tw(self.root, 0.18, { Position = p })
	end
	self:_cursor(true)
	self:_blur()
	if self.padOn then self:_padStep(true) end
end
function Win:Hide()
	self.opened = false
	self.root.Visible = false
	self:_closePopover()
	if self.results then
		pcall(function() self.results:Destroy() end)
		self.results = nil
	end
	self:_blur()
end

-- Blur: Roblox can't blur behind a ScreenGui, so this blurs the game view
-- (a Lighting BlurEffect) only while the menu is open.
function Win:_blur()
	local want = not self.dead and self:_windowOpen() and (self.look.blur or 0) > 0
	if want then
		if not (self.blurFx and self.blurFx.Parent) then
			pcall(function() self.blurFx = make("BlurEffect", { Name = "TempoUIBlur" }, game:GetService("Lighting")) end)
		end
		if self.blurFx then self.blurFx.Size = self.look.blur end
	elseif self.blurFx then
		pcall(function() self.blurFx:Destroy() end)
		self.blurFx = nil
	end
end

function Win:_keys()
	self:conn(UserInputService.InputBegan:Connect(function(i, gp)
		if self.dead or IS_MOBILE then return end
		if i.KeyCode == Enum.KeyCode.Escape and self.pop then
			self:_closePopover()
			return
		end
		-- a focused text box only blocks keys you could be typing; Ctrl/Shift/
		-- Alt/F-keys/Insert... always toggle (a game's hidden console box can
		-- hold focus and ate the menu key)
		local k = self.toggleKey
		local typing = not (k:find("Control") or k:find("Shift") or k:find("Alt") or k:match("^F%d+$")
			or k == "Insert" or k == "Home" or k == "End" or k == "PageUp" or k == "PageDown" or k == "Delete")
		if typing and gp and UserInputService:GetFocusedTextBox() then return end
		if i.UserInputType ~= Enum.UserInputType.Keyboard then return end
		-- Roblox reports Right Ctrl as LeftControl on some keyboards (teek
		-- 10-07: his never arrived as RightControl), so a Ctrl menu key is a
		-- lone TAP of either Ctrl - Ctrl+T / Ctrl+click are left alone
		if k:find("Control") then
			if i.KeyCode == Enum.KeyCode.LeftControl or i.KeyCode == Enum.KeyCode.RightControl then
				self.ctrlTap = { at = os.clock(), clean = true }
			elseif self.ctrlTap then
				self.ctrlTap.clean = false
			end
			return
		end
		if i.KeyCode.Name == k then
			if self:_windowOpen() then self:Hide() else self:Show() end
		end
	end))
	self:conn(UserInputService.InputBegan:Connect(function(i)
		-- a mouse click while Ctrl is held is Ctrl+click, not a tap
		if self.ctrlTap and (i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.MouseButton2) then self.ctrlTap.clean = false end
	end))
	self:conn(UserInputService.InputEnded:Connect(function(i)
		if self.dead or IS_MOBILE or not self.toggleKey:find("Control") then return end
		if i.KeyCode ~= Enum.KeyCode.LeftControl and i.KeyCode ~= Enum.KeyCode.RightControl then return end
		local t = self.ctrlTap
		self.ctrlTap = nil
		if t and t.clean and os.clock() - t.at < 0.4 then
			if self:_windowOpen() then self:Hide() else self:Show() end
		end
	end))
end

function Win:SetToggleKey(name)
	local ok, kc = pcall(function() return Enum.KeyCode[name] end)
	if not (ok and kc) or kc == Enum.KeyCode.K then return false end
	self.toggleKey = kc.Name
	return true
end

function Win:_mobileButton()
	local b = make("TextButton", { Text = "TEMPO", AutoButtonColor = false, Size = UDim2.fromOffset(74, 30), Position = UDim2.new(0, 16, 0.5, -15),
		BorderSizePixel = 0, FontFace = F.brand, TextSize = 11 }, self.overlay)
	self:paint(b, "BackgroundColor3", "card")
	self:paint(b, "TextColor3", "text")
	make("UICorner", { CornerRadius = UDim.new(0, 15) }, b)
	local s = make("UIStroke", { Thickness = 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, b)
	self:paint(s, "Color", "accent")
	local down, moved, start, startPos
	b.InputBegan:Connect(function(i)
		if isClick(i) then down, moved, start, startPos = true, false, i.Position, b.Position end
	end)
	self:conn(UserInputService.InputChanged:Connect(function(i)
		if down and isMove(i) then
			local d = i.Position - start
			if d.Magnitude > 6 then moved = true end
			b.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
		end
	end))
	b.InputEnded:Connect(function(i)
		if down and isClick(i) then
			down = false
			if not moved then
				if self:_windowOpen() then self:Hide() else self:Show() end
			end
		end
	end)
	self.mobileButton = b
end

-- Hubs that call UI:Watermark (ASTD, BladeBall, BloxFruits, Fisch, SniperArena)
-- get a no-op handle instead of an error.
function Win:Watermark()
	return { Set = function() end, Destroy = function() end, Visible = false }
end

-- ============================================================
--  Look: presets, accent, every Interface option
-- ============================================================
-- write the look back to the hub's store (TeekConfig.Theme), so it saves
-- with the profile / autosave and survives a rejoin
function Win:_save()
	local store = self.store
	if store then
		store.Look = table.clone(self.look)
		store.UIScale = self.uiScale
	end
	-- "Same look in every game": the shared file follows every change
	local S = sharedLoad()
	if S.global then
		S.look = table.clone(self.look)
		self:_sharedSoon()
	end
end
-- one write a second at most (sliders call _save on every step)
function Win:_sharedSoon()
	if self.sharedWait then return end
	self.sharedWait = true
	task.delay(1, function()
		self.sharedWait = false
		sharedSave()
	end)
end

-- tear the chrome down and build it again in the current preset; tabs fall
-- back to specs (values kept, nothing re-fires) and the open tab comes back
function Win:_rebuild()
	if self.dead then return end
	self:_resolve()   -- the style + the builder's overrides, as they stand now
	local keepPos = self.root and self.root.Position
	local vis = self.root and self.root.Visible
	local cur, sel = self.current, self.railSel
	-- the page you were on comes back scrolled to where it was (the builder
	-- rebuilds on every structural change)
	local scroll = cur and cur.page and cur.page.CanvasPosition
	self:_closePopover()
	if self.results then
		pcall(function() self.results:Destroy() end)
		self.results = nil
	end
	for _, c in ipairs(self.bconns or {}) do pcall(function() c:Disconnect() end) end
	local keep = {}
	for _, p in ipairs(self.painted) do if p[1].Parent and p[1]:IsDescendantOf(self.overlay) then keep[#keep + 1] = p end end
	self.painted = keep
	if self.root then self.root:Destroy() end
	for _, t in ipairs(self.tabList) do
		t.built, t.page, t.cols, t.colsFrame, t.flowed, t.wide, t.stripButton, t.stripBar = false, nil, nil, nil, false, nil, nil, nil
		for _, s in ipairs(t.sections) do s.card, s.content = nil, nil end
	end
	self.current, self.railSel = nil, nil
	self:_tokens()
	self:_build(self.look.dock == "Free" and keepPos or nil)
	for _, e in ipairs(self.rail) do self:_railButton(e) end
	self.root.Visible = vis and true or false
	if sel then self:_selectEntry(sel, cur) end
	self:_repaint()
	self:_blur()
	if scroll and scroll.Y > 0 and cur and cur.page then
		local page = cur.page
		for _, wait in ipairs({ 0.03, 0.2 }) do   -- the canvas sizes itself over a frame or two
			task.delay(wait, function() if page.Parent then page.CanvasPosition = scroll end end)
		end
	end
end
-- structural values change the chrome: rebuild once the drag that is changing
-- them has ended (a rebuild mid-drag would take the slider out of your hand)
function Win:_rebuildSoon()
	self.rebuildAt = os.clock() + (IS_MOBILE and 0.5 or 0.15)
	if self.rebuildWait then return end
	self.rebuildWait = true
	task.spawn(function()
		while not self.dead and (os.clock() < self.rebuildAt or UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)) do task.wait(0.05) end
		self.rebuildWait = false
		if not self.dead then
			self:_rebuild()
			self:_syncInterface()
		end
	end)
end

-- one look value changed: the cheapest update that shows it
local REPAINT_KEYS = { accent = true, bg = true, text = true, alpha = true, rad = true, glow = true, font = true,
	card = true, rail = true, line = true, dim = true, accent2 = true, shadeColor = true, borderColor = true }
-- the preset builder's structural values: these rebuild the chrome
local STRUCT_KEYS = { anim = true, head = true, titleFont = true, title = true, titleSize = true, caps = true, headSize = true, prefix = true, h4c = true,
	cutTL = true, cutTR = true, cutBL = true, cutBR = true, steps = true, shade = true, shadeSize = true, border = true,
	cardCut = true, cardBar = true, cardShadow = true, switch = true, topH = true, railW = true, logo = true, mark = true, art = true, deco = true, decoAlpha = true }
local function contrast(a, b)
	local la, lb = lum(a) + 0.05, lum(b) + 0.05
	return la > lb and la / lb or lb / la
end
function Win:_setLook(key, value)
	local L = self.look
	if L[key] == value then return end
	-- Undo: what this value was (one step per value while it is being dragged)
	if not self.undoing then
		local u = self.undo
		local last = u[#u]
		if not (last and last[1] == key and os.clock() - last[3] < 1.5) then
			u[#u + 1] = { key, L[key], os.clock(), L.name }
			if #u > 60 then table.remove(u, 1) end
		else
			last[3] = os.clock()
		end
	end
	L[key] = value
	if key ~= "name" and L.name ~= "" and not self.loadingPreset then L.name = "" end   -- no longer the saved preset as it was saved
	if key == "preset" or key == "density" then
		self:_rebuild()
	elseif STRUCT_KEYS[key] then
		self:_rebuildSoon()
	elseif REPAINT_KEYS[key] then
		self:_tokens()
		self:_repaint()
		if key == "bg" then self:_pattern() end
		-- unreadable pairs are the builder's commonest accident: say so, once in a while
		if (key == "bg" or key == "text" or key == "card") and os.clock() - (self.contrastAt or 0) > 12 then
			local t = self.tok
			if contrast(t.text, t.card) < 2.6 then
				self.contrastAt = os.clock()
				self:Notify("that text is hard to read on that background - Undo is on the Preset Builder page", "warn")
			end
		end
	elseif key == "blur" then
		self:_blur()
	elseif key == "w" or key == "h" then
		self.root.Size = UDim2.fromOffset(L.w, self.minimized and self.st.topH or L.h)
		if L.dock ~= "Free" then self:_place() end
		if self.current then self.current:_flow() end
	elseif key == "dock" then
		if value ~= "Free" then self:_place() end
	elseif key == "pattern" or key == "strength" or key == "image" or key == "fit" then
		self:_pattern()
	elseif key == "scale" then
		self:SetUIScale(value / 100)
	elseif key == "remember" then
		L.x, L.y = self.root.Position.X.Offset, self.root.Position.Y.Offset
	end
	self:_save()
end

function Win:SetPreset(name, keepLook)
	if not PRESETS[name] then return false end
	self:_snapshot()
	local leaving = PRESETS[self.look.preset]
	self.look.preset = name
	if not keepLook then
		-- the style's own look, and everything the builder had overridden handed back
		for _, k in ipairs(RESET_KEYS) do self.look[k] = LOOK_DEFAULTS[k] end
		self.look.name = ""
		if leaving and leaving.look.accent then self.look.accent = "" end   -- a character's colour doesn't follow you out
		for k, v in pairs(PRESETS[name].look) do self.look[k] = v end
	end
	self:_rebuild()
	self:_save()
	self:_syncInterface()
	return true
end
function Win:GetLook() return table.clone(self.look) end

-- v1 theme names map to an accent ("Tempo" = this game's colour); a preset
-- name switches preset
function Win:SetTheme(name)
	if PRESETS[name] then return self:SetPreset(name) end
	local acc = LEGACY_THEMES[name]
	if acc == nil then return false end
	self.theme = name
	self:_setLook("accent", acc and toHex(acc) or "")
	self:_syncInterface()
	return true
end
function Win:SetUIScale(s)
	s = tonumber(s)
	if not s then return end
	self.uiScale = math.clamp(s, 0.5, 1.6)
	self.look.scale = math.floor(self.uiScale * 100 + 0.5)
	self.scale.Scale = self.uiScale
	self.overlayScale.Scale = self.uiScale
	if self.root and self.look.dock ~= "Free" then self:_place() end
end
-- a Color3 / "#rrggbb" sets the accent; a number (v1 hue, TeekConfig still
-- pushes AccentHue) is ignored so it can't override the game's colour
function Win:SetAccent(c)
	local kind = typeof(c)
	local col = (kind == "Color3" or kind == "string") and hex(c) or nil
	if not col then return end
	self:_setLook("accent", toHex(col))
	self:_syncInterface()
	return true
end
function Win:GetAccent() return 0 end

-- theme code: "TEMPO1-" + base64(JSON of the look, position left out)
function Win:ThemeCode()
	local t = {}
	for _, k in ipairs(CODE_KEYS) do t[k] = self.look[k] end
	return "TEMPO1-" .. b64enc(HttpService:JSONEncode(t))
end
function Win:ApplyThemeCode(code)
	local raw = string.gsub(string.gsub(tostring(code or ""), "^%s+", ""), "%s+$", "")
	raw = string.gsub(raw, "^TEMPO1%-", "")
	if raw == "" then return false end
	local ok, t = pcall(function() return HttpService:JSONDecode(b64dec(raw)) end)
	if not ok or type(t) ~= "table" then return false end
	self:_applyLook(t)
	return true
end
-- a whole look at once (a theme code, a saved preset). What it doesn't carry
-- goes back to the style's own (a code from before the builder has none of
-- the builder's values) instead of keeping whatever is set here.
function Win:_applyLook(t, name)
	local merged = table.clone(self.look)
	for _, k in ipairs(RESET_KEYS) do merged[k] = LOOK_DEFAULTS[k] end
	merged.accent = ""
	for _, k in ipairs(CODE_KEYS) do if t[k] ~= nil then merged[k] = t[k] end end
	merged.name = name or ""
	local L = cleanLook(merged)
	L.x, L.y, L.remember = self.look.x, self.look.y, self.look.remember
	self:_snapshot()
	self.look = L
	self:SetUIScale(L.scale / 100)
	self:_rebuild()
	self:_save()
	self:_syncInterface()
end

-- ------------------------------------------------------------ undo
function Win:_snapshot()
	local u = self.undo
	u[#u + 1] = { "*", table.clone(self.look), os.clock() }
	if #u > 60 then table.remove(u, 1) end
end
function Win:_undo()
	local u = table.remove(self.undo)
	if not u then return false end
	self.undoing = true
	if u[1] == "*" then
		self.look = u[2]
		self:SetUIScale(self.look.scale / 100)
		self:_rebuild()
		self:_save()
	else
		self:_setLook(u[1], u[2])
		self.look.name = u[4] or ""   -- it is the saved preset again
		self:_save()
	end
	local top = self.undo[#self.undo]
	if top then top[3] = 0 end   -- the next edit of that value is a new step
	self.undoing = false
	self:_syncInterface()
	return true
end

-- ------------------------------------------------------------ your presets
-- Saved in TempoHub/ui.json, so every Tempo hub lists them.
local MINE = "My: "
local function presetName(s)
	s = string.gsub(tostring(s or ""), "[^%w %-_]", "")
	s = string.gsub(string.gsub(s, "^%s+", ""), "%s+$", "")
	return string.sub(s, 1, 24)
end
function Win:PresetNames()
	local out = {}
	for n in pairs(sharedLoad().presets) do out[#out + 1] = n end
	table.sort(out)
	return out
end
function Win:SavePreset(name)
	name = presetName(name)
	if name == "" then return false, "give it a name first" end
	local S = sharedLoad()
	if not S.presets[name] and #self:PresetNames() >= 30 then return false, "30 saved presets is the limit - delete one first" end
	local t = {}
	for _, k in ipairs(CODE_KEYS) do t[k] = self.look[k] end
	t.name = name
	S.presets[name] = t
	local kept = sharedSave()
	self.look.name = name
	self:_save()
	self:_syncInterface()
	return name, (not kept) and "this executor can't write files" or nil
end
function Win:LoadPreset(name)
	local t = sharedLoad().presets[name]
	if type(t) ~= "table" then return false end
	self:_applyLook(t, name)
	return true
end
function Win:DeletePreset(name)
	local S = sharedLoad()
	if not S.presets[name] then return false end
	S.presets[name] = nil
	sharedSave()
	if self.look.name == name then self.look.name = "" end
	self:_save()
	self:_syncInterface()
	return true
end
-- the Style list: the built-in styles, then yours
function Win:_styleList()
	local out = table.clone(PRESET_ORDER)
	for _, n in ipairs(self:PresetNames()) do out[#out + 1] = MINE .. n end
	return out
end

-- push the live look into the Interface controls (no callbacks fire)
function Win:_syncInterface()
	local H, L = self.ifc, self.look
	if not H then return end
	pcall(function()
		H.preset.SetItems(self:_styleList(), L.name ~= "" and (MINE .. L.name) or L.preset)
		H.accent.Set(hex(L.accent) or self.gameAccent)
		H.bg.Set(L.bg)
		H.text.Set(L.text)
		for _, k in ipairs({ "alpha", "blur", "rad", "glow", "scale", "w", "h", "strength" }) do H[k].Set(L[k]) end
		H.font.Set(L.font)
		H.density.Set(L.density)
		H.dock.Set(L.dock)
		H.pattern.Set(L.pattern)
		H.image.Text = L.image
		H.remember.Set(L.remember, false)
		H.anim.Set(L.anim, false)
		H.sounds.Set(L.sounds, false)
	end)
	self:_syncBuilder()
end

-- ============================================================
--  Cursor (from WindLib): the menu draws its own pointer and takes the mouse
--  back from shift-lock games while open, then hands it back exactly as it was.
--  The render step only runs while the menu is open.
-- ============================================================
function Win:_cursor(on)
	if IS_MOBILE then return end
	if on then
		if self.curWatching then return end
		self.curWatching = true
		if not self.curHolder then
			local gui = screenGui("Tempo_Cursor", 2 ^ 31 - 1)
			local holder = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(24, 24), BackgroundTransparency = 1, Visible = false }, gui)
			local ring = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(15, 15), BackgroundTransparency = 1 }, holder)
			make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, ring)
			local rs = make("UIStroke", { Thickness = 1.5 }, ring)
			local dot = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(3, 3), BorderSizePixel = 0 }, holder)
			make("UICorner", { CornerRadius = UDim.new(0.5, 0) }, dot)
			self.curGui, self.curHolder, self.curRing, self.curDot = gui, holder, rs, dot
		end
		self:_cursorTint()
		pcall(function()
			RunService:BindToRenderStep("TempoFreeCursor", Enum.RenderPriority.Last.Value + 10, function()
				if not (self.curHolder and self.curHolder.Parent) then return end
				if self:_windowOpen() then
					if not self.curActive then
						self.curActive = true
						self.curPrev = UserInputService.MouseIconEnabled
						self.curMousePrev = UserInputService.MouseBehavior
						pcall(function() UserInputService.MouseIconEnabled = false end)
						self.curHolder.Visible = true
						self:_fireVisibility(true)
					end
					if UserInputService.MouseBehavior ~= Enum.MouseBehavior.Default then UserInputService.MouseBehavior = Enum.MouseBehavior.Default end
					local m = UserInputService:GetMouseLocation()
					self.curHolder.Position = UDim2.fromOffset(m.X, m.Y)
				else
					-- closed: hand the mouse back and stop running every frame
					if self.curActive then
						self:_cursorRelease()
						self:_fireVisibility(false)
					end
					pcall(function() RunService:UnbindFromRenderStep("TempoFreeCursor") end)
					self.curWatching = false
				end
			end)
		end)
		return
	end
	pcall(function() RunService:UnbindFromRenderStep("TempoFreeCursor") end)
	self.curWatching = false
	self:_cursorRelease()
end
function Win:_cursorRelease()
	self.curActive = false
	if self.curHolder then self.curHolder.Visible = false end
	pcall(function() UserInputService.MouseIconEnabled = (self.curPrev == nil) and true or self.curPrev end)
	if self.curMousePrev then pcall(function() UserInputService.MouseBehavior = self.curMousePrev end) end
	self.curPrev, self.curMousePrev = nil, nil
end
function Win:_cursorTint()
	if self.curRing then self.curRing.Color = self.tok.accent end
	if self.curDot then self.curDot.BackgroundColor3 = self.tok.accent end
end
function Win:OnVisibility(fn)
	self.visFns = self.visFns or {}
	table.insert(self.visFns, fn)
end
function Win:_fireVisibility(open)
	for _, fn in ipairs(self.visFns or {}) do task.spawn(pcall, fn, open) end
end

-- ============================================================
--  Controller (from WindLib): the stick drives the pointer while the menu is
--  open and the player is on a pad; A clicks, B closes, D-pad scrolls. The
--  render step is bound on Show and unbinds itself once the menu closes.
-- ============================================================
local PAD_ACTION = "TempoPadCursor"
local PAD_KEYS = { Enum.KeyCode.Thumbstick1, Enum.KeyCode.ButtonA, Enum.KeyCode.ButtonB, Enum.KeyCode.DPadUp, Enum.KeyCode.DPadDown }
-- menu buttons a player can pick (Settings > Interface > Controller). Combos
-- (hold one, press the other) can't clash with a game's move binds.
local PAD_CHOICES = {
	{ "R3", { "ButtonR3" } }, { "L3", { "ButtonL3" } }, { "Select", { "ButtonSelect" } },
	{ "D-pad Right", { "DPadRight" } }, { "D-pad Left", { "DPadLeft" } }, { "D-pad Down", { "DPadDown" } },
	{ "L1 + R1", { "ButtonL1", "ButtonR1" } }, { "L3 + R3", { "ButtonL3", "ButtonR3" } },
	{ "Select + A", { "ButtonSelect", "ButtonA" } }, { "Off", {} },
}
local PAD_MAP, PAD_NAMES = {}, {}
for _, c in ipairs(PAD_CHOICES) do PAD_MAP[c[1]] = c[2] PAD_NAMES[#PAD_NAMES + 1] = c[1] end
PAD_MAP.B = { "ButtonB" }
-- the old single-button setting (ButtonR3, DPadRight...) -> its new name
local PAD_LEGACY = { ButtonR3 = "R3", ButtonL3 = "L3", ButtonSelect = "Select", DPadRight = "D-pad Right", DPadDown = "D-pad Down" }
local function padMatches(choice, input)
	local keys = PAD_MAP[choice]
	if not keys or #keys == 0 then return false end
	local name, hit = input.KeyCode.Name, false
	for _, k in ipairs(keys) do if k == name then hit = true end end
	if not hit then return false end
	for _, k in ipairs(keys) do
		if k ~= name and not UserInputService:IsGamepadButtonDown(input.UserInputType, Enum.KeyCode[k]) then return false end
	end
	return true
end
function Win:_padChoice(which, legacy)
	local s = self.store or {}
	if which == "open" then
		local v = s.PadOpen or PAD_LEGACY[tostring(legacy)] or legacy
		return PAD_MAP[v] and v or "R3"
	end
	local v = s.PadClose
	return (v == "Same as open" or PAD_MAP[v]) and v or "B"
end
function Win:EnableGamepad(getButton)
	if self.padOn then return end
	self.padOn = true
	self.padLegacy = getButton and getButton()
	local VIM = game:GetService("VirtualInputManager")
	local CAS = game:GetService("ContextActionService")
	local GuiService = game:GetService("GuiService")
	local stick, scroll, bound, pos = Vector2.zero, 0, false, nil
	local function isPad(t) return string.sub(t.Name, 1, 7) == "Gamepad" end
	local padMode = isPad(UserInputService:GetLastInputType())
	local synthUntil = 0
	local function mark()
		synthUntil = os.clock() + 0.3
		if TeekUI.synthMark then pcall(TeekUI.synthMark) end
	end
	local function moveTo(p)
		pos = p
		mark()
		pcall(function() VIM:SendMouseMoveEvent(p.X, p.Y, game) end)
	end
	local function onAnyInput(input)
		local t = input.UserInputType
		if isPad(t) then
			padMode = true
			if self:_windowOpen() then self:_padStep(true) end
		elseif os.clock() > synthUntil and (t == Enum.UserInputType.Keyboard or t == Enum.UserInputType.MouseMovement
			or t == Enum.UserInputType.MouseButton1 or t == Enum.UserInputType.Touch) then padMode = false end
	end
	self.padConns = { UserInputService.InputBegan:Connect(onAnyInput), UserInputService.InputChanged:Connect(onAnyInput) }
	local function sink(_, state, input)
		local kc = input.KeyCode
		if kc == Enum.KeyCode.Thumbstick1 then stick = Vector2.new(input.Position.X, -input.Position.Y)
		elseif kc == Enum.KeyCode.ButtonA then
			if state == Enum.UserInputState.Begin or state == Enum.UserInputState.End then
				local p = pos or UserInputService:GetMouseLocation()
				mark()
				pcall(function() VIM:SendMouseButtonEvent(p.X, p.Y, 0, state == Enum.UserInputState.Begin, game, 1) end)
			end
		elseif kc == Enum.KeyCode.ButtonB then
			-- (closing is padConn's job now - the close button is a setting)
		elseif kc == Enum.KeyCode.DPadUp or kc == Enum.KeyCode.DPadDown then
			if state == Enum.UserInputState.Begin then scroll = (kc == Enum.KeyCode.DPadUp) and 1 or -1
			elseif state == Enum.UserInputState.End then scroll = 0 end
		end
		return Enum.ContextActionResult.Sink
	end
	local function bind(on)
		if on == bound then return end
		bound = on
		if on then
			stick, scroll = Vector2.zero, 0
			pcall(function() GuiService.SelectedObject = nil end)
			local cam = workspace.CurrentCamera
			moveTo(cam and cam.ViewportSize / 2 or UserInputService:GetMouseLocation())
			pcall(function() CAS:BindActionAtPriority(PAD_ACTION, sink, false, Enum.ContextActionPriority.High.Value + 50, table.unpack(PAD_KEYS)) end)
		else
			pcall(function() CAS:UnbindAction(PAD_ACTION) end)
			stick, scroll = Vector2.zero, 0
		end
	end
	-- open button toggles; the close button (B by default) only closes
	self.padConn = UserInputService.InputBegan:Connect(function(input)
		if self.dead or not isPad(input.UserInputType) then return end
		local open = self:_windowOpen()
		local o = self:_padChoice("open", getButton and getButton())
		local c = self:_padChoice("close")
		if c == "Same as open" then c = o end
		if open and padMatches(c, input) then self:Hide() return end
		if padMatches(o, input) then
			if open then self:Hide() else self:Show() end
		end
	end)
	local lastScroll = 0
	self.padStepFn = function(dt)
		local open = self:_windowOpen()
		local active = open and padMode
		bind(active)
		if not open then
			self:_padStep(false)
			return
		end
		if not active then return end
		local mag = stick.Magnitude
		if mag > 0.18 then
			local speed = 1100 * (tonumber(self.store and self.store.PadSpeed) or 100) / 100 * ((mag - 0.18) / 0.82) ^ 2
			local cam = workspace.CurrentCamera
			local vp = cam and cam.ViewportSize or Vector2.new(1920, 1080)
			local p = (pos or UserInputService:GetMouseLocation()) + stick.Unit * speed * dt
			moveTo(Vector2.new(math.clamp(p.X, 0, vp.X - 1), math.clamp(p.Y, 0, vp.Y - 1)))
		end
		if scroll ~= 0 and os.clock() - lastScroll > 0.06 then
			lastScroll = os.clock()
			local p = pos or UserInputService:GetMouseLocation()
			mark()
			pcall(function() VIM:SendMouseWheelEvent(p.X, p.Y, scroll > 0, game) end)
		end
	end
	if self:_windowOpen() then self:_padStep(true) end
end
function Win:_padStep(on)
	if on and not self.padStepping and self.padStepFn then
		self.padStepping = true
		pcall(function() RunService:BindToRenderStep("TempoPadStep", Enum.RenderPriority.Input.Value + 1, self.padStepFn) end)
	elseif not on and self.padStepping then
		self.padStepping = false
		pcall(function() RunService:UnbindFromRenderStep("TempoPadStep") end)
	end
end
function Win:_padOff()
	if not self.padOn then return end
	self.padOn = false
	pcall(function() if self.padConn then self.padConn:Disconnect() end end)
	for _, c in ipairs(self.padConns or {}) do pcall(function() c:Disconnect() end) end
	self:_padStep(false)
	pcall(function() game:GetService("ContextActionService"):UnbindAction(PAD_ACTION) end)
end

-- ============================================================
--  Boot: "the beat". A row of equalizer bars kicks on every beat (it's
--  Tempo) and lights left to right as the steps finish, the wordmark drops in
--  letter by letter, then the bars flatten into one line that stretches out
--  and opens into the window.
-- ============================================================
local BEAT = 0.34   -- seconds per beat, ~176 BPM

local function ease(inst, t, props, style)
	local tw = TweenService:Create(inst, TweenInfo.new(t, style or Enum.EasingStyle.Quint, Enum.EasingDirection.Out), props)
	tw:Play()
	return tw
end

-- HAUNTED boot: two cracks of lightning, TEMPO rises out of the ground, the
-- steps flicker past underneath, then the window swells open
function Win:_hauntBoot(list, onDone)
	local tok = self.tok
	local gui = screenGui("Tempo_Boot", 2 ^ 30)
	local shade = make("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.fromRGB(4, 2, 7), BackgroundTransparency = 1, BorderSizePixel = 0 }, gui)
	local flash = make("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.fromRGB(220, 216, 255), BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 5 }, gui)
	local stage = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(560, 220), BackgroundTransparency = 1 }, gui)
	local WORD, LW = "TEMPO", 64
	local well = make("Frame", { AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 40), Size = UDim2.fromOffset(LW * #WORD + 40, 78),
		BackgroundTransparency = 1, ClipsDescendants = true }, stage)
	local letters = {}
	for i = 1, #WORD do
		local l = make("TextLabel", { Text = WORD:sub(i, i), FontFace = F.brand, TextSize = 56, TextColor3 = tok.text, BackgroundTransparency = 1,
			AnchorPoint = Vector2.new(0.5, 0), Size = UDim2.fromOffset(LW, 72), Rotation = 9,
			Position = UDim2.new(0.5, (i - (#WORD + 1) / 2) * LW, 0, 86) }, well)
		make("UIStroke", { Color = tok.accent, Transparency = 0.45, Thickness = 1.5, ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual }, l)
		letters[i] = l
	end
	local ground = make("Frame", { AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 118), Size = UDim2.fromOffset(380, 1), BorderSizePixel = 0,
		BackgroundColor3 = tok.accent, BackgroundTransparency = 1 }, stage)
	make("UIGradient", { Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.5, 0), NumberSequenceKeypoint.new(1, 1) }) }, ground)
	local caption = make("TextLabel", { Text = "", FontFace = F.mono, TextSize = 12, TextColor3 = tok.dim, TextTransparency = 1, BackgroundTransparency = 1,
		AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 132), Size = UDim2.fromOffset(400, 16) }, stage)
	self.bootCleanup = function()
		pcall(function() gui:Destroy() end)
		self.growing = false
	end
	task.spawn(function()
		ease(shade, 0.35, { BackgroundTransparency = 0.3 })
		local function strike(a)
			flash.BackgroundTransparency = 1 - a
			ease(flash, 0.14, { BackgroundTransparency = 1 })
		end
		task.wait(0.12)
		strike(0.55)
		task.wait(0.18)
		strike(0.35)
		task.wait(0.3)
		if self.dead then return end
		ease(ground, 0.4, { BackgroundTransparency = 0.2 })
		for i, l in ipairs(letters) do
			task.delay((i - 1) * 0.11, function()
				ease(l, 0.55, { Position = l.Position - UDim2.fromOffset(0, 82), Rotation = 0 }, Enum.EasingStyle.Back)
			end)
		end
		ease(caption, 0.4, { TextTransparency = 0 })
		task.wait(0.75)
		for _, step in ipairs(list) do
			if self.dead then return end
			caption.Text = tostring(step)
			task.wait(0.32)
		end
		-- out: the word sinks back, the window swells open
		for _, l in ipairs(letters) do ease(l, 0.25, { TextTransparency = 1 }) end
		ease(caption, 0.25, { TextTransparency = 1 })
		ease(ground, 0.25, { BackgroundTransparency = 1 })
		task.wait(0.26)
		if self.dead then return end
		local size, pos = self.root.Size, self.root.Position
		local W, Hh = size.X.Offset, size.Y.Offset
		self.growing = true
		self.root.Size = UDim2.fromOffset(W * 0.94, Hh * 0.94)
		self.root.Position = pos + UDim2.fromOffset(W * 0.03, Hh * 0.03)
		self:Show()
		ease(self.root, 0.42, { Size = size, Position = pos }, Enum.EasingStyle.Back)
		ease(shade, 0.4, { BackgroundTransparency = 1 })
		task.wait(0.44)
		if self.dead then return end
		self.root.Size, self.root.Position = size, pos
		self.growing = false
		self.bootCleanup = nil
		pcall(function() gui:Destroy() end)
		if onDone then pcall(onDone) end
	end)
end

-- SPOOKY boot: the screen dims, a bed of black fire lights along a line,
-- TEMPO rises out of it letter by letter, sparks fly, the steps flicker past
-- underneath, then the fire sinks and the window swells open
function Win:_spookyBoot(list, onDone)
	local gui = screenGui("Tempo_Boot", 2 ^ 30)
	local shade = make("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = BLACK, BackgroundTransparency = 1, BorderSizePixel = 0 }, gui)
	local stage = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(620, 260), BackgroundTransparency = 1 }, gui)
	local head = cachedFont("Jura", Enum.FontWeight.Bold)
	local WORD, LW = "TEMPO", 58
	local letters = {}
	for i = 1, #WORD do
		local l = make("TextLabel", { Text = WORD:sub(i, i), FontFace = head, TextSize = 54, TextColor3 = Color3.fromRGB(244, 244, 246), TextTransparency = 1,
			BackgroundTransparency = 1, AnchorPoint = Vector2.new(0.5, 0), Size = UDim2.fromOffset(LW, 64), Position = UDim2.new(0.5, (i - (#WORD + 1) / 2) * LW, 0, 122), ZIndex = 2 }, stage)
		letters[i] = l
	end
	local caption = make("TextLabel", { Text = "", FontFace = head, TextSize = 12, TextColor3 = Color3.fromRGB(150, 150, 156), TextTransparency = 1, BackgroundTransparency = 1,
		AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 200), Size = UDim2.fromOffset(420, 16) }, stage)
	local bed = make("Frame", { AnchorPoint = Vector2.new(0.5, 1), Position = UDim2.new(0.5, 0, 0, 178), Size = UDim2.fromOffset(440, 110), BackgroundTransparency = 1, ZIndex = 0 }, stage)
	local tongues, sparks, conn = {}, {}, nil
	local fire = { on = 0 }
	self.bootCleanup = function()
		pcall(function() conn:Disconnect() end)
		pcall(function() gui:Destroy() end)
		self.growing = false
	end
	task.spawn(function()
		local fimg, dimg = resolveDeco("spooky_fire"), resolveDeco("spooky_dot")
		if self.dead then return end
		local rnd = Random.new()
		-- (the middle tongues tallest: the word stands in the fire)
		for i = 1, (fimg and 21 or 0) do
			local mid = 1 - math.abs((i - 11) / 11)
			tongues[i] = { ph = rnd:NextNumber(0, 6), sz = rnd:NextNumber(0.75, 1.15) * (0.6 + 0.6 * mid),
				lbl = make("ImageLabel", { Image = fimg, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0.5, 0.825), Position = UDim2.new((i - 0.5) / 21, rnd:NextNumber(-6, 6), 1, 0),
					Size = UDim2.fromOffset(2, 2), ImageTransparency = 1, ZIndex = 0 }, bed) }
		end
		for i = 1, (dimg and 14 or 0) do
			sparks[i] = { x = rnd:NextNumber(-170, 170), t = rnd:NextNumber(0, 2.4), s = rnd:NextNumber(5, 12),
				lbl = make("ImageLabel", { Image = dimg, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0.5, 0.5), Size = UDim2.fromOffset(8, 8), ImageTransparency = 1 }, stage) }
		end
		local t0 = os.clock()
		conn = RunService.RenderStepped:Connect(function(dt)
			local t = os.clock() - t0
			for _, tg in ipairs(tongues) do
				local f = 0.78 + 0.22 * math.sin(t * 9 + tg.ph) * math.sin(t * 3.7 + tg.ph * 2)
				local s = tg.sz * f * fire.on
				tg.lbl.Size = UDim2.fromOffset(44 * s + 1, 110 * s + 1)
				tg.lbl.Rotation = math.sin(t * 2.4 + tg.ph) * 8
				tg.lbl.ImageTransparency = 1 - 0.95 * fire.on
			end
			for _, sp in ipairs(sparks) do
				sp.t += dt
				local q = (sp.t % 2.4) / 2.4
				sp.lbl.Position = UDim2.new(0.5, sp.x + math.sin(sp.t * 3 + sp.x) * 10, 0, 180 - q * 150)
				sp.lbl.Size = UDim2.fromOffset(sp.s, sp.s)
				sp.lbl.ImageTransparency = 1 - math.sin(math.pi * q) * 0.85 * fire.on
			end
		end)
		ease(shade, 0.4, { BackgroundTransparency = 0.25 })
		-- the fire lights
		local lit = os.clock()
		while os.clock() - lit < 0.5 and not self.dead do fire.on = math.min(1, (os.clock() - lit) / 0.5) task.wait() end
		fire.on = 1
		if self.dead then return end
		for i, l in ipairs(letters) do
			task.delay((i - 1) * 0.09, function()
				ease(l, 0.6, { Position = l.Position - UDim2.fromOffset(0, 26), TextTransparency = 0 }, Enum.EasingStyle.Back)
			end)
		end
		ease(caption, 0.4, { TextTransparency = 0 })
		task.wait(0.8)
		for _, step in ipairs(list) do
			if self.dead then return end
			caption.Text = spaced(string.upper(tostring(step)))
			task.wait(0.32)
		end
		-- out: the word fades, the fire sinks, the window swells open
		for _, l in ipairs(letters) do ease(l, 0.25, { TextTransparency = 1 }) end
		ease(caption, 0.25, { TextTransparency = 1 })
		local out = os.clock()
		while os.clock() - out < 0.3 and not self.dead do fire.on = 1 - (os.clock() - out) / 0.3 task.wait() end
		fire.on = 0
		if self.dead then return end
		local size, pos = self.root.Size, self.root.Position
		local W, Hh = size.X.Offset, size.Y.Offset
		self.growing = true
		self.root.Size = UDim2.fromOffset(W * 0.94, Hh * 0.94)
		self.root.Position = pos + UDim2.fromOffset(W * 0.03, Hh * 0.03)
		self:Show()
		ease(self.root, 0.42, { Size = size, Position = pos }, Enum.EasingStyle.Back)
		ease(shade, 0.4, { BackgroundTransparency = 1 })
		task.wait(0.44)
		if self.dead then return end
		self.root.Size, self.root.Position = size, pos
		self.growing = false
		self.bootCleanup = nil
		pcall(function() conn:Disconnect() end)
		pcall(function() gui:Destroy() end)
		if onDone then pcall(onDone) end
	end)
end

function Win:Boot(steps, onDone)
	if self.booted then
		if onDone then task.spawn(onDone) end
		return
	end
	self.booted = true
	self.opened = false
	self.root.Visible = false
	if self.st.haunt and self.look.anim ~= false then return self:_hauntBoot(steps or { "loading" }, onDone) end
	if self.st.spooky and self.look.anim ~= false then return self:_spookyBoot(steps or { "loading" }, onDone) end
	local tok = self.tok
	local list = steps or { "loading" }
	local gui = screenGui("Tempo_Boot", 2 ^ 30)
	local shade = make("Frame", { Size = UDim2.fromScale(1, 1), BackgroundColor3 = Color3.new(0, 0, 0), BackgroundTransparency = 1, BorderSizePixel = 0 }, gui)
	local stage = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), Size = UDim2.fromOffset(560, 300),
		BackgroundTransparency = 1 }, gui)

	local logo = resolveLogo(self.look.mark, false)   -- (the splash is always on black)
	local mark = logo and make("ImageLabel", { Image = logo, BackgroundTransparency = 1, ImageTransparency = 1, AnchorPoint = Vector2.new(0.5, 0),
		Position = UDim2.new(0.5, 0, 0, 0), Size = UDim2.fromOffset(52, 52) }, stage)

	-- wordmark, one label per letter so they can land one after another
	local WORD, LW = "TEMPO", 46
	local letters = {}
	for i = 1, #WORD do
		letters[i] = make("TextLabel", { Text = WORD:sub(i, i), FontFace = F.brand, TextSize = 40, TextColor3 = tok.text, TextTransparency = 1,
			BackgroundTransparency = 1, AnchorPoint = Vector2.new(0.5, 0), Size = UDim2.fromOffset(LW, 48),
			Position = UDim2.new(0.5, (i - (#WORD + 1) / 2) * LW, 0, 56) }, stage)
	end
	local sub = make("TextLabel", { Text = (string.upper(self.subtitle):gsub("(.)", "%1 ")), FontFace = F.mono, TextSize = 12, TextColor3 = tok.dim,
		TextTransparency = 1, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 122),
		Size = UDim2.fromOffset(400, 14) }, stage)

	-- the bars: a fixed spectrum shape (loud in the middle) with some wobble
	local N, BW, GAP, MAXH, MID = 40, 5, 3, 72, 200
	local width = N * BW + (N - 1) * GAP
	local bars, amp = {}, {}
	for i = 1, N do
		bars[i] = make("Frame", { AnchorPoint = Vector2.new(0, 0.5), BorderSizePixel = 0, BackgroundColor3 = tok.line2, BackgroundTransparency = 1,
			Position = UDim2.new(0.5, -width / 2 + (i - 1) * (BW + GAP), 0, MID), Size = UDim2.fromOffset(BW, 2) }, stage)
		make("UICorner", { CornerRadius = UDim.new(0, 2) }, bars[i])
		local centre = 1 - math.abs((i - (N + 1) / 2) / (N / 2))
		amp[i] = 0.3 + 0.7 * centre * (0.75 + 0.25 * math.sin(i * 2.3))
	end
	local caption = make("TextLabel", { Text = "", FontFace = F.mono, TextSize = 11, TextColor3 = tok.dim, TextTransparency = 1, BackgroundTransparency = 1,
		AnchorPoint = Vector2.new(0, 0), Position = UDim2.new(0.5, -width / 2, 0, MID + 48), Size = UDim2.fromOffset(width - 60, 14),
		TextXAlignment = Enum.TextXAlignment.Left }, stage)
	local bpm = make("TextLabel", { Text = math.floor(60 / BEAT + 0.5) .. " BPM", FontFace = F.mono, TextSize = 11, TextColor3 = tok.faint,
		TextTransparency = 1, BackgroundTransparency = 1, AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(0.5, width / 2, 0, MID + 48),
		Size = UDim2.fromOffset(60, 14), TextXAlignment = Enum.TextXAlignment.Right }, stage)

	-- one render loop drives every bar for the ~2 s this runs
	local t0, lit, litTarget, flatAt, last = os.clock(), 0, 0, nil, os.clock()
	local conn
	-- Destroy() mid-boot calls this; every step after a wait checks self.dead
	self.bootCleanup = function()
		pcall(function() conn:Disconnect() end)
		pcall(function() gui:Destroy() end)
		self.growing = false
	end
	conn = RunService.RenderStepped:Connect(function()
		local now = os.clock()
		local t, dt = now - t0, now - last
		last = now
		lit += (litTarget - lit) * math.min(1, dt * 12)
		local kick = math.exp(-(t % BEAT) * 9)          -- sharp hit, quick decay
		local fade = math.min(1, t / 0.3)
		local flat = flatAt and math.min(1, (now - flatAt) / 0.22) or 0
		for i, b in ipairs(bars) do
			local wob = 0.5 + 0.5 * math.sin(t * 7 + i * 0.7)
			local h = 3 + (MAXH - 3) * amp[i] * (0.22 * wob + 0.78 * kick * (0.55 + 0.45 * wob))
			h = h * (1 - flat) + 2 * flat
			b.Size = UDim2.fromOffset(BW, math.floor(h + 0.5))
			b.BackgroundColor3 = (i <= lit + 0.5) and tok.accent or tok.line2
			b.BackgroundTransparency = 1 - fade
		end
	end)

	task.spawn(function()
		-- in: shade, mark, letters dropping in one by one
		ease(shade, 0.4, { BackgroundTransparency = 0.3 })
		if mark then ease(mark, 0.4, { ImageTransparency = 0 }) end
		for i, l in ipairs(letters) do
			task.delay((i - 1) * 0.06, function()
				ease(l, 0.45, { TextTransparency = 0, Position = l.Position + UDim2.fromOffset(0, 10) }, Enum.EasingStyle.Back)
			end)
		end
		ease(sub, 0.5, { TextTransparency = 0 })
		ease(caption, 0.4, { TextTransparency = 0 })
		ease(bpm, 0.4, { TextTransparency = 0.2 })
		task.wait(BEAT * 2)
		if self.dead then return end

		-- a beat or two per step; the bars fill as they finish
		local per = math.max(1, math.ceil(3 / #list))
		for i, step in ipairs(list) do
			if self.dead then break end
			caption.Text = tostring(step)
			litTarget = N * i / #list
			task.wait(BEAT * per)
		end

		-- out: bars flatten, everything else fades
		flatAt = os.clock()
		for _, l in ipairs(letters) do ease(l, 0.22, { TextTransparency = 1 }) end
		for _, x in ipairs({ sub, caption, bpm }) do ease(x, 0.22, { TextTransparency = 1 }) end
		if mark then ease(mark, 0.22, { ImageTransparency = 1 }) end
		task.wait(0.24)
		if self.dead then return end

		-- the flat bars become one line that stretches to the window's width,
		-- at the window's middle, and the window opens out of it
		conn:Disconnect()
		local s = self.uiScale
		local pos, size = self.root.Position, self.root.Size
		local W, H = size.X.Offset, size.Y.Offset
		local cx, cy = (pos.X.Offset + W / 2) * s, (pos.Y.Offset + H / 2) * s
		local line = make("Frame", { AnchorPoint = Vector2.new(0.5, 0.5), BorderSizePixel = 0, BackgroundColor3 = tok.accent,
			Position = UDim2.new(0.5, 0, 0.5, MID - 150), Size = UDim2.fromOffset(width, 2) }, gui)
		for _, b in ipairs(bars) do b.Visible = false end
		ease(line, 0.3, { Position = UDim2.fromOffset(cx, cy), Size = UDim2.fromOffset(W * s, 2) })
		task.wait(0.3)
		if self.dead then return end

		-- the window starts as the line (2 screen px, so 2/s before UIScale)
		-- and grows; drag/resize wait until it has
		self.growing = true
		self.root.Size = UDim2.fromOffset(W, 2 / s)
		self.root.Position = UDim2.fromOffset(pos.X.Offset, pos.Y.Offset + H / 2 - 1 / s)
		self:Show()
		line:Destroy()
		ease(self.root, 0.38, { Size = size, Position = pos })
		ease(shade, 0.38, { BackgroundTransparency = 1 })
		task.wait(0.4)
		if self.dead then return end
		self.root.Size, self.root.Position = size, pos
		self.growing = false
		self.bootCleanup = nil
		pcall(function() gui:Destroy() end)
		if onDone then pcall(onDone) end
	end)
end

-- ============================================================
--  Interface tab: every look option, saved in store.Look (store is the
--  hub's TeekConfig.Theme, so it rides the profile / autosave)
-- ============================================================
local BG_SWATCHES = { "#151b26", "#0b0b0e", "#0c0e12", "#06080b", "#15112a", "#18130e", "#1d2a24", "#f3efe6" }
local TEXT_SWATCHES = { "#f2f6fb", "#e9edf2", "#dce6f0", "#f3efe4", "#eadfc6", "#ffffff", "#111111", "#2a1f14" }
function Win:mountInterface(tab, store)
	store = type(store) == "table" and store or {}
	self.store = store
	-- first run on v2: carry the v1 theme name / scale over
	if type(store.Look) ~= "table" then
		local L = table.clone(self.look)
		local legacy = store.Theme and LEGACY_THEMES[store.Theme]
		if legacy then L.accent = toHex(legacy) end
		local sc = tonumber(store.UIScale)
		if sc then L.scale = math.floor(sc * 100 + 0.5) end
		store.Look = L
	end
	if type(store.Theme) == "string" then self.theme = store.Theme end
	-- "Same look in every game": the shared look is this hub's look
	do
		local S = sharedLoad()
		if S.global and type(S.look) == "table" then store.Look = table.clone(S.look) end
	end
	local want = cleanLook(store.Look)
	local changed = false
	for k, v in pairs(want) do if self.look[k] ~= v then changed = true end end
	if changed then
		self.look = want
		self:SetUIScale(want.scale / 100)
		self:_rebuild()
	end
	self:_save()
	-- teek 10-13: Haunted is everyone's new default - once, for anyone still
	-- on the old default (Glass, not a saved preset of their own)
	if not store.HauntedOnce then
		store.HauntedOnce = true
		if self.look.preset == "Glass" and self.look.name == "" then self:SetPreset("Haunted") else self:_save() end
	end
	local L = self.look
	local H = {}
	self.ifc = H
	local function set(key) return function(v) self:_setLook(key, v) end end

	tab:Section("Preset")
	H.preset = tab:Dropdown("Style", self:_styleList(), L.name ~= "" and (MINE .. L.name) or L.preset, function(v)
		if string.sub(v, 1, #MINE) == MINE then self:LoadPreset(string.sub(v, #MINE + 1)) else self:SetPreset(v) end
	end)
	tab:Label("Picking a style resets the look below to that style.\nMake your own on the Preset Builder page.")
	tab:Button("Reset look to preset", false, function()
		self:SetPreset(self.look.preset)
		self:Notify(self.look.preset .. " look restored", "good")
	end)

	tab:Section("Colors")
	H.accent = tab:Color("Accent", hex(L.accent) or self.gameAccent, function(c) self:_setLook("accent", toHex(c)) end)
	tab:Button("Use this game's color", false, function()
		self:_setLook("accent", "")
		self:_syncInterface()
	end)
	H.bg = tab:Color("Background", L.bg, function(c) self:_setLook("bg", toHex(c)) end, BG_SWATCHES)
	H.text = tab:Color("Text", L.text, function(c) self:_setLook("text", toHex(c)) end, TEXT_SWATCHES)

	tab:Section("Look & feel")
	H.alpha = tab:Slider("Opacity", 20, 100, L.alpha, set("alpha"), "%")
	H.blur = tab:Slider("Blur", 0, 30, L.blur, set("blur"), " px")
	tab:Label("Blurs the game view while the menu is open.")
	H.rad = tab:Slider("Roundness", 0, 22, L.rad, set("rad"), " px")
	H.glow = tab:Slider("Glow", 0, 40, L.glow, set("glow"))
	H.font = tab:Dropdown("Font", FONT_CHOICES, L.font, set("font"))
	local fontItem = tab.last
	if fontItem then fontItem.fontOf = function(v) return cachedFont(family(v), Enum.FontWeight.Medium) end end

	tab:Section("Size & layout")
	H.scale = tab:Slider("UI scale", 60, 140, L.scale, set("scale"), "%")
	tab:Label("Shrink it if the window runs off your screen.")
	H.w = tab:Slider("Width", 600, 1000, L.w, set("w"), " px")
	H.h = tab:Slider("Height", 380, 680, L.h, set("h"), " px")
	H.density = tab:Segment("Density", { "Compact", "Cozy", "Roomy" }, L.density, set("density"))
	H.dock = tab:Segment("Dock", DOCKS, L.dock, set("dock"))
	H.remember = tab:Toggle("Remember position", L.remember, set("remember"))
	tab:Label("Free dock only: the window opens where you left it.")
	tab:Button("Recentre Window", false, function()
		self:_setLook("dock", "Center")
		self:_syncInterface()
	end)

	tab:Section("Extras")
	H.pattern = tab:Dropdown("Background", PATTERNS, L.pattern, function(v)
		if v ~= "None" and self.look.strength < 25 then self.look.strength = 45 end
		self:_setLook("pattern", v)
		self:_syncInterface()
	end)
	H.image = tab:Input("Image asset id (Custom image)", L.image, function(v)
		local id = string.match(tostring(v or ""), "%d+") or ""
		-- typing an id used to switch to Custom image at strength 0 = nothing drawn (teek 10-09)
		if id ~= "" and self.look.strength < 25 then self.look.strength = 45 end
		self:_setLook("image", id)
		if id ~= "" and self.look.pattern ~= "Custom image" then
			self:_setLook("pattern", "Custom image")
		end
		self:_syncInterface()
	end)
	H.strength = tab:Slider("Background strength", 0, 100, L.strength, set("strength"), "%")
	H.anim = tab:Toggle("Animations", L.anim, set("anim"))
	H.sounds = tab:Toggle("Toggle sounds", L.sounds, set("sounds"))
	if IS_MOBILE then
		tab:Label("Open and close the hub with the TEMPO button.")
	else
		if store.ToggleKey then self:SetToggleKey(store.ToggleKey) end
		tab:Keybind("Menu key", self.toggleKey, function(name)
			if name == "None" or not self:SetToggleKey(name) then
				self:Notify("can't use that key for the menu", "warn")
				return
			end
			store.ToggleKey = name
			self:Notify("menu key: " .. keyLabel(name), "good")
		end)
		tab:Label("Opens and closes the hub. Default Right Ctrl - not K, which is shift lock in some games.")
		-- backup for when a key won't reach the game: the draggable TEMPO pill
		local function setPill(on)
			if on and not self.mobileButton then self:_mobileButton() end
			if self.mobileButton then self.mobileButton.Visible = on end
		end
		setPill(store.OpenButton == true)
		tab:Toggle("On-screen Open Button", store.OpenButton == true, function(v)
			store.OpenButton = v
			setPill(v)
		end)
	end

	-- controller: which pad buttons open / close the menu (EnableGamepad reads these)
	if self.padOn then
		tab:Section("Controller")
		tab:Dropdown("Open / close menu", PAD_NAMES, self:_padChoice("open", self.padLegacy), function(v)
			store.PadOpen = v
			self:Notify("controller menu button: " .. v, "good")
		end)
		local closeNames = { "B", "Same as open" }
		for _, n in ipairs(PAD_NAMES) do if n ~= "Off" then closeNames[#closeNames + 1] = n end end
		tab:Dropdown("Close menu", closeNames, self:_padChoice("close"), function(v) store.PadClose = v end)
		tab:Slider("Pointer speed", 40, 250, tonumber(store.PadSpeed) or 100, function(v) store.PadSpeed = v end, "%")
		tab:Label("Combos: hold the first button, press the second - they never clash with your move binds. While the menu is open the left stick moves the pointer, A clicks, D-pad up/down scrolls.")
	end

	tab:Section("Theme code")
	local codeBox
	tab:Button("Copy theme code", true, function()
		local code = self:ThemeCode()
		if codeBox then codeBox.Text = code end
		local ok = false
		pcall(function()
			if setclipboard then
				setclipboard(code)
				ok = true
			end
		end)
		self:Notify(ok and "Theme code copied. Send it to a friend." or "Theme code is in the box below. Copy it from there.", "good")
	end)
	codeBox = tab:Input("Paste a code", "", nil)
	tab:Button("Apply code", false, function()
		if self:ApplyThemeCode(codeBox.Text) then self:Notify("Theme applied", "good")
		else self:Notify("That code didn't work", "warn") end
	end)
	if tab.entry and not self.tabs["Preset Builder"] then
		-- right after the Interface page, wherever the hub put that
		local bt, tabs = self:_newTab("Preset Builder", tab.entry), tab.entry.tabs
		table.remove(tabs)
		table.insert(tabs, (table.find(tabs, tab) or #tabs) + 1, bt)
		self:_mountBuilder(bt)
	end
end

-- ============================================================
--  Preset Builder page: every structural value of a style, overridable.
--  "Style" on a control = leave it as the style you picked has it. Sliders
--  show the value in force (the style's until you move them).
-- ============================================================
local OWN = STYLE_OWN
local function toUi(v) return v == "Preset" and OWN or v end
local function fromUi(v) return v == OWN and "Preset" or v end
local AUTO_COLORS = { "card", "rail", "line", "dim", "accent2", "shadeColor", "borderColor" }
function Win:_mountBuilder(b)
	local L = self.look
	local B = {}
	self.bld = B
	local function set(key) return function(v) self:_setLook(key, v) end end
	local function pick(key) return function(v) self:_setLook(key, fromUi(v)) end end
	local function col(key) return function(c) self:_setLook(key, toHex(c)) end end
	local function fontPick(key) return function(v) self:_setLook(key, v == OWN and "" or v) end end
	local function fontRow(label, key)
		local h = b:Dropdown(label, HEAD_CHOICES, L[key] ~= "" and L[key] or OWN, fontPick(key))
		if b.last then b.last.fontOf = function(v) return v ~= OWN and cachedFont(family(v), Enum.FontWeight.Regular) or nil end end
		return h
	end
	-- an image id box: digits, or "none" where that hides the thing
	local function idBox(placeholder, key, allowNone)
		local h
		h = b:Input(placeholder, L[key], function(v)
			v = tostring(v or "")
			local id = string.match(v, "%d+") or ((allowNone and string.lower(v) == "none") and "none" or "")
			self:_setLook(key, id)
			h.Text = id
		end)
		return h
	end

	b:Section("Your presets")
	b:Label("Change anything on this page, name it, save it.\nSaved presets show up in every Tempo hub.")
	B.name = b:Input("Preset name", L.name, nil)
	b:Button("Save as my preset", true, function()
		local name, why = self:SavePreset(B.name.Text)
		if not name then self:Notify(why, "warn")
		elseif why then self:Notify('"' .. name .. '" is saved until you leave - ' .. why .. ". Copy its theme code to keep it.", "warn", 8)
		else self:Notify('saved "' .. name .. '" - it is in the Style list now', "good") end
	end)
	B.mine = b:Dropdown("My presets", self:PresetNames(), L.name, nil)
	b:Button("Load", false, function()
		local n = B.mine.Get()
		if not (n and self:LoadPreset(n)) then self:Notify("no saved preset picked", "warn") end
	end)
	b:Button("Delete", false, function()
		local n = B.mine.Get()
		if n and self:DeletePreset(n) then self:Notify('deleted "' .. n .. '"', "good") else self:Notify("no saved preset picked", "warn") end
	end)
	b:Button("Undo last change", false, function()
		if not self:_undo() then self:Notify("nothing to undo", "warn") end
	end)
	b:Button("Start over from the style", false, function()
		self:SetPreset(self.look.preset)
		self:Notify("back to plain " .. self.look.preset, "good")
	end)
	B.global = b:Toggle("Same look in every game", sharedLoad().global, function(v)
		local S = sharedLoad()
		S.global = v
		S.look = v and table.clone(self.look) or nil
		sharedSave()
	end)
	b:Label("Off: each game keeps its own look.")
	b:Button("Copy theme code", false, function()
		local code, ok = self:ThemeCode(), false
		pcall(function()
			if setclipboard then
				setclipboard(code)
				ok = true
			end
		end)
		self:Notify(ok and "Theme code copied - it carries everything on this page." or "No clipboard here - copy it from the Interface page.", ok and "good" or "warn")
	end)

	b:Section("Colors")
	B.card = b:Color("Cards", hex(L.card) or self.tok.card, col("card"))
	B.rail = b:Color("Sidebar & top bar", hex(L.rail) or self.tok.rail, col("rail"))
	B.line = b:Color("Lines", hex(L.line) or self.tok.line, col("line"))
	B.dim = b:Color("Muted text", hex(L.dim) or self.tok.dim, col("dim"))
	B.accent2 = b:Color("Second accent", hex(L.accent2) or self.tok.accent2, col("accent2"))
	B.h4c = b:Dropdown("Card titles", { OWN, "Text", "Accent", "Second accent", "Muted" }, toUi(L.h4c), pick("h4c"))
	b:Button("Colors back to automatic", false, function()
		for _, k in ipairs(AUTO_COLORS) do self:_setLook(k, "") end
		self:_syncInterface()
	end)
	b:Label("Accent, background and text are on the Interface page.")

	b:Section("Type")
	B.head = fontRow("Heading font", "head")
	B.titleFont = fontRow("Title font", "titleFont")
	local titleBox
	titleBox = b:Input("Title text (TEMPO)", L.title, function(v)
		self:_setLook("title", string.sub((string.gsub(tostring(v or ""), "[^\32-\126]", "")), 1, 18))
		titleBox.Text = self.look.title
	end)
	B.title = titleBox
	B.titleSize = b:Slider("Title size", 10, 48, self.st.brand.size, set("titleSize"), " px")
	B.headSize = b:Slider("Heading size", -4, 8, L.headSize, set("headSize"), " px")
	B.caps = b:Segment("Headings", { OWN, "Upper", "Normal" }, toUi(L.caps), pick("caps"))
	B.prefix = b:Dropdown("Card title marker", { OWN, "None", "//", ">", "#", "+", "~", "::", "*" }, toUi(L.prefix), pick("prefix"))
	b:Label("The body font is on the Interface page.")

	b:Section("Window shape")
	local cut = self.st.cut or { 0, 0, 0, 0 }
	B.cutTL = b:Slider("Cut top left", 0, 40, cut[1], set("cutTL"), " px")
	B.cutTR = b:Slider("Cut top right", 0, 40, cut[2], set("cutTR"), " px")
	B.cutBL = b:Slider("Cut bottom left", 0, 40, cut[3], set("cutBL"), " px")
	B.cutBR = b:Slider("Cut bottom right", 0, 40, cut[4], set("cutBR"), " px")
	B.steps = b:Segment("Stepped cuts", { OWN, "On", "Off" }, toUi(L.steps), pick("steps"))
	B.shade = b:Segment("Shadow", { OWN, "None", "Offset", "Rim" }, toUi(L.shade), pick("shade"))
	B.shadeSize = b:Slider("Shadow size", 0, 20, L.shadeSize, set("shadeSize"), " px")
	B.shadeColor = b:Color("Shadow color", self.tok.shadeC, col("shadeColor"))
	B.border = b:Slider("Border", 0, 4, self.st.noStroke and 0 or self.st.winStroke.thick, set("border"), " px")
	B.borderColor = b:Color("Border color", self.tok.borderC, col("borderColor"))
	b:Label("A window with cut corners has no border - give it a Rim shadow.\nRoundness and Glow are on the Interface page.")

	b:Section("Cards & controls")
	B.cardCut = b:Slider("Card corner cut", 0, 24, self.st.cardCut and self.st.cardCut[2] or 0, set("cardCut"), " px")
	B.cardBar = b:Segment("Card accent bar", { OWN, "On", "Off" }, toUi(L.cardBar), pick("cardBar"))
	B.cardShadow = b:Segment("Card shadow", { OWN, "On", "Off" }, toUi(L.cardShadow), pick("cardShadow"))
	B.switch = b:Segment("Switches & buttons", { OWN, "Round", "Square" }, toUi(L.switch), pick("switch"))

	b:Section("Layout")
	B.topH = b:Slider("Top bar height", 40, 90, self.st.topH, set("topH"), " px")
	B.railW = b:Slider("Sidebar width", 120, 260, self.st.railW or 156, set("railW"), " px")
	b:Label("Window size, density and dock are on the Interface page.")

	b:Section("Images")
	b:Label("Paste a Roblox image / decal id. Your own art, anywhere.")
	B.mark = b:Dropdown("Tempo logo", MARKS, L.mark, set("mark"))
	B.logo = idBox("Your own logo id (or none)", "logo", true)
	B.art = idBox("Header art id (or none)", "art", true)
	B.image = b:Input("Background image id", L.image, function(v)
		local id = string.match(tostring(v or ""), "%d+") or ""
		if id ~= "" and self.look.strength < 25 then self.look.strength = 45 end
		self:_setLook("image", id)
		if id ~= "" and self.look.pattern ~= "Custom image" then self:_setLook("pattern", "Custom image") end
		self:_syncInterface()
	end)
	B.fit = b:Segment("Background fit", { "Crop", "Fit", "Stretch", "Tile" }, L.fit, set("fit"))
	local decos = { OWN, "None" }
	for _, n in ipairs(DECO_ORDER) do decos[#decos + 1] = n end
	B.deco = b:Dropdown("Decorations", decos, toUi(L.deco), pick("deco"))
	B.decoAlpha = b:Slider("Decoration strength", 0, 100, L.decoAlpha, set("decoAlpha"), "%")
	b:Label("Decorations: the emblems of the character styles, on any style.")

	b:Section("Motion & sound")
	B.speed = b:Slider("Animation speed", 50, 300, L.speed, set("speed"), "%")
	B.volume = b:Slider("Click volume", 0, 100, L.volume, set("volume"), "%")
	B.sound = idBox("Click sound id", "sound", false)
	b:Label("Animations and Toggle sounds switch on from the Interface page.")
end

-- push the live look into the builder's controls (no callbacks fire)
function Win:_syncBuilder()
	local B, L, st, tok = self.bld, self.look, self.st, self.tok
	if not B then return end
	local ok, err = pcall(function()
		if L.name ~= "" then B.name.Text = L.name end   -- (else: leave the name being typed)
		B.mine.SetItems(self:PresetNames(), L.name ~= "" and L.name or B.mine.Get())
		B.global.Set(sharedLoad().global, false)
		B.card.Set(hex(L.card) or tok.card)
		B.rail.Set(hex(L.rail) or tok.rail)
		B.line.Set(hex(L.line) or tok.line)
		B.dim.Set(hex(L.dim) or tok.dim)
		B.accent2.Set(hex(L.accent2) or tok.accent2)
		B.shadeColor.Set(tok.shadeC)
		B.borderColor.Set(tok.borderC)
		B.head.Set(L.head ~= "" and L.head or OWN)
		B.titleFont.Set(L.titleFont ~= "" and L.titleFont or OWN)
		B.title.Text = L.title
		B.titleSize.Set(st.brand.size)
		B.headSize.Set(L.headSize)
		for _, k in ipairs({ "caps", "prefix", "h4c", "steps", "shade", "cardBar", "cardShadow", "switch", "deco" }) do B[k].Set(toUi(L[k])) end
		local cut = st.cut or { 0, 0, 0, 0 }
		B.cutTL.Set(cut[1])
		B.cutTR.Set(cut[2])
		B.cutBL.Set(cut[3])
		B.cutBR.Set(cut[4])
		B.shadeSize.Set(L.shadeSize)
		B.border.Set(st.noStroke and 0 or st.winStroke.thick)
		B.cardCut.Set(st.cardCut and st.cardCut[2] or 0)
		B.topH.Set(st.topH)
		B.railW.Set(st.railW or 156)
		B.logo.Text, B.art.Text, B.image.Text, B.sound.Text = L.logo, L.art, L.image, L.sound
		B.fit.Set(L.fit)
		B.mark.Set(L.mark)
		B.decoAlpha.Set(L.decoAlpha)
		B.speed.Set(L.speed)
		B.volume.Set(L.volume)
	end)
	if not ok then warn("[TempoUI] builder sync: " .. tostring(err)) end
end

-- ============================================================
--  Teardown
-- ============================================================
function Win:Destroy()
	self.dead = true
	if self.bootCleanup then self.bootCleanup() end
	self:_cursor(false)
	pcall(function() self:_padOff() end)
	for _, c in ipairs(self.conns) do pcall(function() c:Disconnect() end) end
	for _, c in ipairs(self.bconns or {}) do pcall(function() c:Disconnect() end) end
	self:_blur()
	for _, g in ipairs({ self.gui, self.overlay, self.curGui }) do pcall(function() g:Destroy() end) end
end

TeekUI.legacyNew = TeekUI.legacyNew or TeekUI.new
TeekUI.TempoUI = true
TeekUI.TempoUIVersion = 2
end
