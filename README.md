# Local WoW Server Sandbox (WotLK 3.3.5a)

I set up a local AzerothCore server to have a personal sandbox where I could test features, break things, and see how a full stack project handles a heavy data load. Going into this, I didn't have experience with C++ compilation, database administration, or server infrastructure. I started out just wanting to build a "Vanilla+" world filled with active playerbots, but it turned into a very hands on way to learn backend troubleshooting.

## Tech Stack

* **Languages:** C++, Lua, SQL
* **Tools:** Visual Studio, CMake, MySQL/MariaDB, HeidiSQL, Git

---

## Roadblocks & Fixes

### 1. Database Crashes & Filter Optimization

* **The Problem:** MySQL 8.4 was crashing immediately on startup during the initial install. Later, once the server was running and the bots started generating data, HeidiSQL hit its default 1,000 row display limit. This meant my character data was completely buried under thousands of bot rows and hidden in the UI.
* **How I Fixed It:** I tracked the startup crash down to an outdated `--no-beep` directive in the `my.ini` configuration file and commented it out. As far as the visibility issue in HeidiSQL, I stopped trying to browse raw tables and wrote a targeted SQL query using character GUIDs (`SELECT * FROM character_settings WHERE guid = xxxx AND source = 'mod-challenge-modes'`)

### 2. UI Layout Adjustments & Asset Camera Math

* **The Problem:** I implemented a backported dialogue UI addon (Immersion) but since I was also running an HD client asset patch. The new HD models use different skeletal structures than the original 2010 client expects, the addon's camera anchored completely wrong—zooming straight into the NPC's hair or empty space instead of framing their face.
* **How I Fixed It:** I went into the addon's Lua files to see how it handles the 3D viewport rendering. They had hardcoded the camera distance scale which was set way too close (`0.47`). Which I updated to `1.1` scale factor pulling the lens back and realigning the vertical vectors so the HD models frame properly.
```lua
if apply then
    apply(self, creatureID or unit)
    self:SetCamDistanceScale(1.1)
    self.creatureID = creatureID
    self.unit = creatureID and 'npc' or unit
end
```
### 3. Build Pipeline Troubleshooting & Broken Character Encodings

* **The Problem:** Integrating a custom "Challenge Mode" module triggered fatal compilation errors inside Visual Studio. Once I finally got it to build, the in game NPC menus were corrupted, rendering text strings as unreadable question marks (`?`).
* **How I Fixed It:** I cleared the build cache, regenerated the pipeline build scripts using CMake, and re-targeted the dependencies. For the broken text, I looked through `ChallengeModes.cpp` and found the previous developer had hardcoded the system messages in Chinese characters. Since I am on the English client it obviously couldn't parse that so it broke.

```cpp
ChatHandler(player->GetSession()).PSendSysMessage("挑战模式已启用。");

// Translated Code

ChatHandler(player->GetSession()).PSendSysMessage("Challenge mode has been enabled.");

```

### 4. Load Balancing & Zone Concurrency Optimization
* **The Problem:** Populating the world with active playerbots created an extreme configuration challenge across different level brackets. High bot density specifically ~200 bots in Elwynn Forest, 80+ in Westfall, and 70+ in Redridge broke the server's default dynamic respawn mechanics. The core engine's scaling math was aggressively overreacting to these player counts, forcing mob respawn timers to ~10 seconds. This turned these crowded starting areas into non stop meat grinders.

```text
Respawn.DynamicRateCreature = 30
Respawn.DynamicMinimumCreature = 60
```
* **How I Fixed It:** By dropping the threshold rate to 30, I told the server to start scaling spawn timers back much earlier. At Westfall's 80 bot load this flattened the math loop to calculate ~1.8 minute respawn pacing (30 / 80 = 0.375$ multiplier on a standard 5-minute spawn).

### Future Roadmap
*  I have plans to modify both the game server and client systems, along with some custom server side features and frontend client side modifications.
