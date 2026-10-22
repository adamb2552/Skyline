# Skyline — Design Doc (working title)

An original, iOS‑native vertical tower‑building simulation inspired by the classic 1990s "build a skyscraper" genre. The player builds a tower floor by floor, rents space to tenants, moves people with elevators and stairs, and grows the tower's star rating.

> **IP rule:** This is a spiritual successor, not a port. Do not use the name "SimTower", "Yoot Tower", or any original sprites, sounds, music, UI screenshots, or text. Game mechanics are fair to reimplement; all assets and names must be original. Facility names, numbers, and balance values in this doc are our own.

---

## 1. Goals and non‑goals

**Goals**
- Fun, readable tower sim playable one‑handed on iPhone and comfortably on iPad.
- A deterministic simulation that is fully unit‑testable without a device.
- Data‑driven content so balance changes are JSON edits, not code edits.
- Small, shippable milestones that each end in something playable.

**Non‑goals (v1)**
- Multiplayer, cloud saves, in‑app purchases, Android, macOS (Mac Catalyst may come free later).
- 3D. The game is a 2D side‑view cross‑section.

---

## 2. Core gameplay

### 2.1 Loop
1. Build floors and a lobby, then place facilities (offices, condos, hotel rooms, shops).
2. Connect floors with stairs, escalators, and elevators.
3. People arrive on schedules, travel through the tower, and generate income.
4. Bad transit (long waits, long walks, crowding) raises **stress**; stressed tenants complain and eventually leave.
5. Income funds expansion. Hitting population and facility goals raises the **star rating**, unlocking new facility types and taller limits.

### 2.2 The world grid
- Side view. The tower lives on a grid of **floors (rows)** × **units (columns)**.
- Width: 200 units. Height: up to 100 floors above ground and 10 basement levels (limits unlock with stars).
- Floor 1 is the ground. A **lobby** must exist on floor 1 before anything else can be built.
- Every facility occupies a contiguous horizontal span on one floor (some, like a cinema, span 2 floors). A floor segment must exist under a facility; building a facility auto‑builds empty floor beneath it and charges for it.
- Floors above ground must be supported: the span you build on floor N must sit over existing floor on N‑1 (no floating floors). Basements must be under existing floor 1 footprint.
- Lobbies can additionally be placed every 15 floors as **sky lobbies** (transfer points for express elevators).

### 2.3 Facilities (initial set)

All values live in `Content/facilities.json`. Numbers below are starting points to tune.

| Facility | Width | Cost | Income | Occupants / behavior | Unlock |
|---|---|---|---|---|---|
| Lobby | 4+ (drag) | 5k / 4 units | — | Entry point; people spawn and exit here | ★1 |
| Floor (empty) | 1+ (drag) | 500 / unit | — | Structure | ★1 |
| Stairs | 8 | 5k | — | Connects floor N and N+1; slow, tiring | ★1 |
| Escalator | 8 | 20k | — | Connects N and N+1; good for shops/lobbies | ★1 |
| Standard elevator | 4 per car column | 200k + 80k/car | — | Serves a range of up to 30 floors; up to 8 cars | ★1 |
| Office | 9 | 40k | Rent per quarter (tunable) | 6 workers, weekday 8:00–17:00, lunch out | ★1 |
| Fast food | 16 | 100k | Per customer | Lunch/dinner traffic | ★1 |
| Condo | 16 | 80k | One‑time sale price | 3 residents; leave AM, return PM; sale refunded if they move out | ★1 |
| Hotel single | 4 | 20k | Per night | 1 guest; arrive evening, leave morning | ★2 |
| Hotel twin | 6 | 50k | Per night | 2 guests | ★2 |
| Housekeeping | 15 | 50k | — (upkeep) | Cleans hotel rooms; dirty rooms can't be rented | ★2 |
| Restaurant | 24 | 200k | Per customer | Evening traffic | ★2 |
| Retail shop | 12 | 100k | Rent + traffic bonus | Weekend and evening traffic | ★2 |
| Parking (basement) | 8 | 20k | Small | Required by some tenants at ★3+ | ★3 |
| Service elevator | 4 | 100k | — | Staff only (housekeeping, security) | ★2 |
| Express elevator | 6 | 400k | — | Stops only at lobbies/sky lobbies and basement | ★3 |
| Security office | 16 | 100k | — | Needed for ★3; responds to incidents | ★3 |
| Medical center | 26 | 500k | — | Needed for ★4 | ★4 |
| Recycling center | 25 (2 floors, basement) | 500k | — | Needed for ★4 | ★4 |
| Hotel suite | 10 | 100k | Per night (high) | Picky guests (low stress tolerance) | ★4 |
| Cinema | 30 (2 floors) | 500k | Per show | Big evening/weekend crowds | ★3 |
| Event hall | 30 | 1M | Per event | Weekend spikes | ★4 |
| Transit station | wide, bottom basement | 1M | Bonus foot traffic | Needed for ★5 | ★5 |
| Observation crown | top floor | 3M | Win condition | Placed on floor 100 at ★5 | ★5 |

### 2.4 Time
- 1 sim tick = 1 game minute. At 1× speed: 10 ticks per real second, so a game day ≈ 2.4 real minutes. Speeds: pause, 1×, 2×, 4×.
- Calendar: days are **Weekday, Weekday, Weekend** repeating. 4 days = 1 **quarter** (tunable).
- Time of day drives the sky color, lights in windows, and every agent's schedule.
- Quarterly: collect rent, pay maintenance, run tenant evaluations, check star rating.

### 2.5 People (agents)
- Each occupant is an agent with: home facility, schedule, current location, current trip, stress (0–100).
- Visitors (shoppers, diners, hotel guests, cinema goers) spawn at the lobby (or transit station) based on time of day, star rating, and how many destination facilities exist.
- Schedules are data (`Content/schedules.json`), e.g. office worker: arrive 07:30–09:00 → lunch 12:00–13:00 at a food place within the tower (or outside if none reachable) → leave 17:00–18:00.
- Agents are rendered as tiny sprites. Simulate every agent, but render only those visible.

### 2.6 Transit and routing
- Build a **transit graph**: nodes are (floor, x) points at stairs/escalators/elevator stops; edges are walking segments along a floor and vertical links.
- Each trip is planned once with Dijkstra/A* using cost = walk time + expected wait + ride time, with a per‑agent rule: max 1 stairs hop for most people before preferring elevators, and a cap on transfers (e.g. 2).
- If no route exists, the trip fails, the agent gets a large stress hit, and the facility logs "can't reach".
- Stairs/escalators: instant to board, slow to traverse, capacity‑limited.
- **Elevators** (the hardest system, isolate it well):
  - A shaft has a served floor range and 1–8 cars.
  - Each floor stop has an up queue and down queue.
  - Cars use a collective‑control (SCAN‑style) algorithm: keep going in the current direction while there are calls or riders ahead, then reverse. Assign hall calls to the car with the lowest estimated arrival time.
  - Car capacity (e.g. 15), door time, travel time per floor, acceleration ignored in v1.
  - Player settings per shaft: which floors are served (toggle per floor), idle "home" floor, and a rush‑hour mode (cars return to lobby in the morning).
  - Shafts can't overlap other facilities; placing a shaft carves through existing floors.

### 2.7 Stress and evaluations
- Stress rises with: waiting (per minute in queue), walking distance, stairs used, crowded cars, noise from neighbors (e.g. hotel next to restaurant), failed trips. It decays slowly while at the home facility.
- Each facility's **evaluation** = average stress of its occupants over the quarter → Good (blue), Fair (yellow), Poor (red).
- Poor for 2 quarters in a row → tenant moves out (office empties, condo sale refunded, hotel rooms get no guests).
- An **overlay mode** toggles color‑coding of all facilities by evaluation so the player can find problems.

### 2.8 Economy
- Start with $2,000,000.
- Costs: construction (one‑time), maintenance (per quarter, per facility and per elevator car), demolition refunds 0%.
- Income: office rent (quarterly), condo sale (one‑time), hotel nights (daily), food/shop/cinema (per customer).
- Player can set rent tier (low / normal / high) per office and shop: high rent = more money, lower tolerance for stress.
- Going below $0: no construction allowed. Never a game over in v1; the player can always pause and demolish.

### 2.9 Star rating

| Stars | Requirements (tunable) | Unlocks |
|---|---|---|
| ★1 | Start | Basics |
| ★2 | Population 300 | Hotels, housekeeping, restaurants, shops, service elevator |
| ★3 | Population 1,000 + security office | Express elevators, parking, cinema, height to 60 floors |
| ★4 | Population 5,000 + medical + recycling + hotel suites + good hotel evaluations | Event hall, height to 100 floors |
| ★5 | Population 10,000 + transit station + VIP visit rated good | Observation crown |
| Crown | Crown built on top floor | "You win" screen, sandbox continues |

### 2.10 Events (late milestone)
- **Fire:** starts in a random facility; security staff must reach it; destroys the facility if unreached in time.
- **Security threat:** a ransom demand; pay or rely on security office coverage.
- **Pests in hotel rooms:** dirty rooms left too long get pests; housekeeping fixes them.
- **VIP visit (★4→★5):** VIP tours the tower; their rating depends on stress along their route.
- Keep all events skippable in settings ("relaxed mode").

---

## 3. Recommended tech stack

**Recommendation: Swift + SpriteKit for the tower view, SwiftUI for menus and HUD, and a pure‑Swift simulation package.**

Why:
- Native, free, no engine license or runtime; small app; good battery life.
- SpriteKit handles thousands of small sprites, camera, and pinch/pan well for a 2D side view.
- SwiftUI is the fastest way to build panels, inspectors, and menus that adapt to iPhone and iPad.
- A Swift Package for the simulation can be built and tested with `swift test` from the command line (even on Linux), so Claude Code can verify logic without a simulator.
- Claude Code is strong at Swift and SPM; Xcode projects are generated with **XcodeGen** from a `project.yml`, so nobody hand‑edits `.pbxproj` files (a common source of breakage when an AI edits the project).

Alternatives considered:
- **Godot 4 (GDScript/C#):** great 2D tooling and cross‑platform, but iOS export adds friction, and the scene files are less pleasant for an AI to edit blind. Pick this if Android matters soon.
- **Unity:** heavy for a 2D sim, licensing and editor‑centric workflow make vibe coding harder.
- **SwiftUI Canvas only (no SpriteKit):** simpler, but camera/zoom and many moving sprites get harder at scale.

Versions: latest stable Xcode and Swift (Swift 6 language mode, strict concurrency), iOS 17+ deployment target.

Tooling:
- XcodeGen (`brew install xcodegen`) to generate `Skyline.xcodeproj`.
- SwiftLint + SwiftFormat (optional but recommended).
- GitHub Actions on `macos-latest`: `swift test` for the package, `xcodebuild build` for the app.

---

## 4. Architecture

```
Skyline/
├── project.yml                  # XcodeGen spec (source of truth for the Xcode project)
├── CLAUDE.md
├── docs/DESIGN.md               # this file
├── Packages/TowerCore/          # pure Swift, Foundation only, no UIKit/SpriteKit
│   ├── Sources/TowerCore/
│   │   ├── Model/               # Grid, Floor, Facility, Shaft, Car, Agent, Money, Clock
│   │   ├── Content/             # FacilityDefinition, Schedule decoding + validation
│   │   ├── Simulation/          # Simulation.step(), systems run in fixed order
│   │   ├── Systems/             # Spawning, Routing, Elevators, Stress, Economy, Rating, Events
│   │   ├── Commands/            # PlaceFacility, Demolish, ConfigureShaft, SetRent, SetSpeed
│   │   ├── Persistence/         # SaveGame (Codable), versioned migrations
│   │   └── Util/                # SeededRNG, geometry helpers
│   └── Tests/TowerCoreTests/
├── App/
│   ├── SkylineApp.swift         # SwiftUI entry
│   ├── GameController.swift     # owns Simulation, runs fixed‑step loop, applies Commands
│   ├── Scene/                   # SpriteKit: TowerScene, CameraController, layers, sprite pools
│   ├── UI/                      # SwiftUI HUD, build palette, inspector, menus, settings
│   ├── Input/                   # gesture handling → Commands
│   ├── Audio/
│   └── Resources/
│       ├── Content/             # facilities.json, schedules.json, ratings.json, balance.json
│       └── Assets.xcassets
└── Tools/                       # scripts: placeholder art generator, content validator
```

### 4.1 Rules
- **Simulation is the source of truth and knows nothing about rendering.** `TowerCore` must compile with only Foundation.
- **Deterministic:** fixed time step, seeded RNG stored in the save, no `Date()` or random calls inside systems. Same seed + same commands = same state. This makes bugs reproducible and tests reliable.
- **Commands in, snapshots out.** The UI never mutates state directly. It sends `Command` values; the simulation validates (cost, placement, unlocks) and returns a result (`.ok` or `.rejected(reason)`). The scene reads state each frame (or a lightweight snapshot) to draw.
- **Fixed‑step loop:** `GameController` accumulates real time × speed and calls `simulation.step()` once per tick, capped per frame to avoid spiral‑of‑death. Rendering interpolates agent positions between ticks.
- **Systems run in a fixed order each tick:** Clock → Spawning → Routing (new trips) → Movement (walk/stairs) → Elevators → Facility service (eat/shop/sleep) → Stress → Economy (on schedule) → Evaluation/Rating (quarterly) → Events.
- **Data‑driven content:** facility types, schedules, ratings, and balance numbers load from JSON at launch and are validated (unit test loads the shipped JSON). Adding a facility type should require no new code unless it has new behavior.
- **Performance budget:** 60 fps on an iPhone 12 with 3,000 agents. Simulation step < 4 ms at 4×. Use struct arrays and integer IDs for agents rather than class graphs; pool `SKSpriteNode`s; cull offscreen agents; draw facilities as static textures and only redraw on change.

### 4.2 Save/load
- `SaveGame` is a `Codable` struct with `schemaVersion`, seed, tick, money, grid, facilities, shafts and cars, agents, stats history, settings.
- JSON on disk in Application Support (compressed with LZFSE if large). 3 manual slots + 1 autosave.
- Autosave every in‑game day and when the app moves to background (`scenePhase == .background`).
- Migrations: `migrate(from: version)` chain; a test keeps a fixture save from each released version and asserts it still loads.
- Round‑trip test: save → load → step N ticks must equal stepping the original N ticks.

---

## 5. Controls and UI

### 5.1 Camera
- One‑finger drag to pan (when not placing). Pinch to zoom (3 zoom levels snap: overview, normal, close). Double‑tap to zoom in on a point.
- Camera clamps to the tower bounds plus some sky and ground margin.

### 5.2 Building
- Bottom **build palette** (iPhone) / left sidebar (iPad) grouped by category: Structure, Transit, Commercial, Residential, Hotel, Services. Locked items show the star needed.
- Select a tool → a **ghost preview** follows the finger: green if valid, red with a short reason if not ("Needs floor below", "Not enough money").
- Horizontal tools (floor, lobby) support drag‑to‑extend; elevators drag vertically to set the shaft range.
- Lift finger to place. A small **Undo** toast appears for 5 seconds after any placement or demolition (refund in full during that window).
- Bulldozer tool for demolition, with a confirm for expensive facilities.
- Haptic tick on valid placement, error haptic on rejection.

### 5.3 Information
- Tap a facility → inspector sheet: name, occupants, evaluation with top stress causes, rent tier, income this quarter.
- Tap an elevator shaft → floor‑service toggles, number of cars (+/−), home floor, rush‑hour mode.
- Tap a person → their name, destination, stress, and a "follow" camera button.
- HUD top bar: money, population, stars, date/time, speed controls.
- Overlay toggle: evaluation colors, elevator wait heatmap.
- Notifications as small banners ("Office on 12F is unhappy", "★2 reached!").

### 5.4 Device layout
- Landscape‑first on iPhone (portrait supported with a narrower palette). Full support for iPad split view sizes.
- Respect safe areas and Dynamic Type in SwiftUI panels. Minimum 44 pt touch targets.

---

## 6. Art and audio

- **Style:** clean, colorful pixel art at a fixed unit size (1 grid unit = 8 px wide at 1×, floor height = 36 px), scaled by an integer factor with nearest‑neighbor filtering.
- **Phase 1 placeholders:** facilities drawn as flat colored rectangles with labels; agents as 2×5 px colored figures. Generate these in code so gameplay never waits on art.
- **Phase 2:** original sprite sheets per facility with day and night variants (lit windows), interior details, and occupied/empty states. Sources: commission an artist, draw in Aseprite, or use CC0 packs (e.g. Kenney) as a base. Record the source and license of every asset in `ASSETS.md`.
- **Sky:** gradient that shifts with time of day, plus clouds and a skyline silhouette in the background. Ground strip with street and trees.
- **Audio:** ambient city loop, elevator ding, construction thud, cash register, crowd murmur scaled by population; all original or CC0 (freesound.org CC0 filter). Music: original or royalty‑free, with a mute toggle.
- **Naming:** pick an original title. Ideas: Skyline, Highrise Story, Vertical City, Tower Tycoon (check App Store for conflicts). Facility names stay generic.

---

## 7. Milestones

Each milestone should be one or a few Claude Code sessions, ends with a runnable build, and is merged to `main` only when its acceptance criteria pass. Keep `swift test` green at every step.

### M0. Project skeleton
- XcodeGen `project.yml`, `TowerCore` package, app target that depends on it, SwiftUI app hosting a `SpriteView` with a sky gradient and ground strip.
- GitHub Actions: `swift test` + `xcodebuild build` for iOS Simulator.
- **Accept:** app launches in the simulator showing sky and ground; `swift test` passes with one trivial test; CI is green.

### M1. Grid, camera, floors, lobby
- Grid model and placement validation in `TowerCore` (lobby on 1F first, support rule, bounds).
- Money with construction costs. Pan and pinch zoom. Build palette with Floor and Lobby. Ghost preview with valid/invalid colors.
- **Accept:** player can build a lobby and stack floors; invalid placements show a reason and cost nothing; money decreases correctly; tests cover each placement rule.

### M2. Facilities and content loading
- `facilities.json` with lobby, floor, office, condo, fast food, stairs. Loader + validator with tests.
- Place and demolish facilities; inspector shows basic info; Undo toast.
- **Accept:** placing a facility auto‑builds floor beneath it; overlapping placements are rejected; changing a cost in JSON changes it in game without code changes.

### M3. Time and economy
- Clock with weekday/weekend and quarters; speed controls; day/night sky and lit windows.
- Income and maintenance on schedule; HUD shows money, date, time.
- **Accept:** at 4× a quarter passes in about 2.4 real minutes; office rent and condo sales arrive at the right times; economy has tests using a fixed seed.

### M4. People and stairs
- Agents with schedules spawn from offices/condos and visit fast food at lunch. Walking along floors; stairs and escalators between floors.
- Transit graph + routing with tests (reachable, unreachable, prefers fewer stairs).
- Placeholder agent sprites with pooling and culling.
- **Accept:** in a 5‑floor tower with stairs, workers arrive in the morning, eat at lunch, and leave in the evening; an unreachable office logs failed trips; 1,000 agents keep 60 fps on device.

### M5. Elevators
- Shafts, cars, hall queues, SCAN dispatch, capacity, per‑floor service toggles, car count, home floor.
- Elevator inspector UI and drag‑to‑place shafts.
- **Accept:** a 20‑floor office tower with one shaft and 4 cars moves the morning rush with no deadlock; tests cover dispatch ordering, capacity, and toggled‑off floors; average wait is shown in the inspector.

### M6. Stress, evaluations, move‑outs
- Stress sources and decay; quarterly evaluations; move‑outs; evaluation overlay; banners.
- **Accept:** removing elevator cars from a busy tower turns offices yellow then red and they move out after 2 poor quarters; adding cars back recovers; tests cover stress math.

### M7. Hotels, services, retail
- Hotel singles/twins, check‑in/out, housekeeping and dirty rooms, restaurants, shops, service elevator, rent tiers.
- **Accept:** hotel rooms only rent when cleaned; housekeeping uses service elevators; restaurant traffic peaks in the evening; rent tier changes income and stress tolerance.

### M8. Star ratings and progression
- `ratings.json`; rating checks; unlocks reflected in palette; height limits; celebration banner.
- Express elevators and sky lobbies; parking; security office; cinema.
- **Accept:** a scripted test tower reaches ★3 in simulation; locked items cannot be placed; express elevators only stop at lobbies.

### M9. Save/load and settings
- Save slots, autosave on background, title screen (New / Continue / Load), settings (sound, music, relaxed mode).
- **Accept:** killing the app and relaunching resumes within one game day; round‑trip determinism test passes; a fixture save from M9 loads in all later milestones.

### M10. ★4–★5, events, and win
- Medical, recycling, suites, event hall, transit station, VIP visit, fire, security threat, pests, observation crown and win screen.
- **Accept:** each event can be triggered from a debug menu and resolved; a full playthrough to the crown is possible.

### M11. Art, audio, polish
- Replace placeholders with original art; audio; tutorial (first 5 minutes guided); app icon and launch screen; accessibility pass.
- **Accept:** no placeholder art remains; `ASSETS.md` lists the license for every asset; the tutorial teaches build, elevators, and evaluations.

### M12. Performance and TestFlight
- Profile with Instruments; hit the performance budget with 3,000 agents; fix memory growth; crash reporting via Xcode Organizer.
- **Accept:** 60 fps on an iPhone 12 class device at a ★4 tower; build uploaded to TestFlight and installable by testers.

---

## 8. Debug tools (build early, from M1)
- Debug menu (only in Debug builds): add money, set stars, jump time, spawn N agents, trigger event, show transit graph, show elevator queues, show FPS and sim step time.
- `Tools/sim-cli`: a command‑line target in the package that loads a scenario JSON and runs N days headless, printing stats. Great for balance and for Claude Code to verify changes without a device.

## 9. Open questions
- Final title and art direction (pixel art vs. clean vector).
- Whether condos should resell after move‑out or stay vacant until re‑bought.
- Height of the tallest tower on small iPhones: is 100 floors fun to scroll? Consider a floor‑jump minimap.
