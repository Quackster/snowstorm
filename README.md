# SnowWar Implementation Guide for Habbo Servers

## Preface

This was written by **Quackster**.

Licensed under the **Apache License, Version 2.0** (Apache 2.0).  
You may use, modify, and distribute this project in accordance with the license.

**Attribution required:**  
Please credit **Quackster** when using this as a reference guide.

## Executive Summary

SnowWar (also known as SnowStorm) is a multiplayer snowball-fighting minigame where players throw snowballs at opponents to score points. The system uses a tick-based simulation with client synchronization via checksums, supports 1-4 teams, and includes features like snowball machines, player stunning, and configurable game durations.

---

## 1. System Architecture Overview

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ GAMEEVENT    │  │ REQUESTFULL  │  │ STARTGAME    │  ...          │
│  │ (incoming)   │  │ GAMESTATUS   │  │ LEAVEGAME    │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
└─────────┼─────────────────┼─────────────────┼───────────────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      MESSAGE HANDLER LAYER                           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ SnowWarMessageHandler (dispatches to specific handlers)        │ │
│  │   - SnowWarWalkMessage (event 0)                               │ │
│  │   - SnowWarAttackPlayerMessage (event 1)                       │ │
│  │   - SnowWarThrowLocationMessage (event 2)                      │ │
│  │   - SnowWarCreateSnowballMessage (event 3)                     │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        GAME LOGIC LAYER                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │ SnowWarGame      │  │ SnowWarGameTask  │  │ SnowWarPlayers    │  │
│  │ (extends Game)   │◄─┤ (tick processor) │  │ (attribute store) │  │
│  └──────────────────┘  └──────────────────┘  └───────────────────┘  │
│           │                     │                                    │
│           ▼                     ▼                                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ GAME OBJECTS                                                  │   │
│  │  - SnowWarAvatarObject (players)                             │   │
│  │  - SnowWarSnowballObject (projectiles)                       │   │
│  │  - SnowWarMachineObject (dispensers)                         │   │
│  └──────────────────────────────────────────────────────────────┘   │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ EVENTS (queued per turn)                                      │   │
│  │  - SnowWarAvatarMoveEvent, SnowWarThrowEvent                 │   │
│  │  - SnowWarHitEvent, SnowWarStunEvent                         │   │
│  │  - SnowWarMachineAddSnowballEvent, etc.                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      PERSISTENCE LAYER                               │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────┐ │
│  │ Map files      │  │ Configuration  │  │ Database (game history)│ │
│  │ (arena_*.dat)  │  │ (game.config)  │  │                        │ │
│  └────────────────┘  └────────────────┘  └────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Required Dependencies & Interfaces

### 2.1 Core Framework Dependencies

Your server must provide these abstractions:

| Dependency | Purpose |
|------------|---------|
| `GameScheduler` | Scheduled executor service for game ticks |
| `Player` | Your player/session abstraction with `send(packet)` capability |
| `Room` | Room abstraction with entity management and task scheduling |
| `Position` | 2D coordinate class with X/Y getters and `getDistanceSquared()` |
| `PacketReader` / `PacketWriter` | Packet reading/writing abstractions |
| `OutgoingPacket` | Base class for outgoing packets |
| `IncomingPacket` | Interface for incoming packet handlers |

### 2.2 Base Game Infrastructure

You need a generic games framework with:

```
Game (abstract)
├── id, mapId, name, gameType
├── teams: Map<Integer, GameTeam>
├── gameState: GameState (WAITING, STARTED, ENDED)
├── objects: List<GameObject> (active game objects)
├── eventsQueue: BlockingQueue<GameEvent>
├── preparingGameSecondsLeft, totalSecondsLeft (AtomicInteger)
├── Methods: startGame(), beginGame(), finishGame(), restartGame()
└── Abstract: assignSpawnPoints(), buildMap(), gameTick(), hasEnoughPlayers()

GamePlayer
├── player, userId, teamId, gameId, objectId
├── spawnPosition, score, inGame, isSpectator
└── gameObject: GameObject

GameTeam
├── id, players: List<GamePlayer>
└── points (calculated from player scores)

GameObject (abstract)
├── id, gameObjectType
└── serialize(PacketWriter)
```

---

## 3. Configuration Values

Add these to your game configuration system:

```properties
# SnowWar Game Settings
snowwar.create.game.enabled=true
snowwar.start.minimum.active.teams=2
snowwar.preparing.game.seconds=10
snowwar.restart.game.seconds=30
snowwar.ticket.charge=2
snowwar.increase.points=true

# Optional: override game length (default uses gameLengthChoice)
# snowwar.game.lifetime.seconds=180
```

### 3.1 Game Constants

```pseudocode
constants SnowWarConstants:
    // ========================================================================
    // TIMING (Critical for synchronization)
    // ========================================================================
    SUBTURNS_PER_TICK = 5              // Subturns (frames) processed per server tick
    SERVER_TICK_MS = 300               // Milliseconds between scheduler invocations
    SUBTURN_MS = 60                    // Effective ms per subturn (300 / 5)

    // ========================================================================
    // MOVEMENT
    // ========================================================================
    SUBTURN_MOVEMENT = 640             // World units moved per subturn
    TILE_SIZE = 3200                   // World units per tile (32 * 100)
    VELOCITY_DIVISOR = 255
    BASE_VELOCITY_MULTIPLIER = 2000

    // ========================================================================
    // COLLISION
    // ========================================================================
    COLLISION_DISTANCE = 2000          // Circle radius for hit detection
    BALL_HEIGHT_THRESHOLD = 5000       // Max height for ball to hit player

    // ========================================================================
    // HEALTH AND SNOWBALLS
    // ========================================================================
    INITIAL_HEALTH = 4                 // Hits before stun
    MAX_SNOWBALLS = 5                  // Max snowballs player can hold

    // ========================================================================
    // ACTIVITY TIMERS (in subturns/frames, NOT milliseconds)
    // To convert: real_time_ms = timer_value * 60
    // ========================================================================
    STUNNED_TIMER = 125                // 125 frames = 7.5 seconds
    INVINCIBILITY_TIMER = 60           // 60 frames = 3.6 seconds
    CREATING_TIMER = 20                // 20 frames = 1.2 seconds

    // ========================================================================
    // SCORING
    // ========================================================================
    HIT_SCORE = 1                      // Points for damaging opponent
    STUN_SCORE = 5                     // Points for knocking out opponent

    // ========================================================================
    // COOLDOWNS (in real milliseconds)
    // ========================================================================
    THROW_COOLDOWN_MS = 300            // Minimum time between throws

    // ========================================================================
    // SNOWBALL MACHINE (timers in subturns/frames)
    // ========================================================================
    MACHINE_SNOWBALL_GENERATOR_TIME = 100  // 100 frames = 6.0 seconds
    MACHINE_MAX_SNOWBALL_CAPACITY = 5
```

---

## 4. Database Schema

### 4.1 Game Maps Table

```sql
CREATE TABLE games_maps (
    id INT AUTO_INCREMENT PRIMARY KEY,
    game_type ENUM('BATTLEBALL', 'SNOWWAR') NOT NULL,
    map_id INT NOT NULL,
    heightmap TEXT NOT NULL,
    UNIQUE KEY (game_type, map_id)
);
```

### 4.2 Player Spawns Table

```sql
CREATE TABLE games_player_spawns (
    id INT AUTO_INCREMENT PRIMARY KEY,
    game_type ENUM('BATTLEBALL', 'SNOWWAR') NOT NULL,
    map_id INT NOT NULL,
    team_id INT NOT NULL,
    x INT NOT NULL,
    y INT NOT NULL,
    z DOUBLE DEFAULT 0.0
);
```

### 4.3 Room Models Update

```sql
INSERT INTO rooms_models (id, model_id, trigger_class, heightmap, ...)
VALUES (?, 'snowwar_lobby_1', 'SnowWarLobbyTrigger', ...);
```

---

## 5. Map Data System

### 5.1 Map File Format

Maps are stored as external files in `tools/snowwar_maps/`:

| File | Purpose |
|------|---------|
| `arena_{id}.dat` | Item placements (sprite, name, x, y, rotation, height) |
| `arena_{id}_heightmap.txt` | Height map (pipe-separated rows, `0`=walkable, `X`=blocked) |
| `arena_{id}_snowmachines.dat` | Snowball machine positions |
| `arena_{id}_spawn_clusters.dat` | Spawn regions (x y radius minDistance) |

### 5.2 Height Map Format

```
XXXX000XXXX|
XXX00000XXX|
XX0000000XX|
...
```

- `0` = walkable tile
- `X` = blocked tile
- `|` = row separator

### 5.3 Map Manager

```pseudocode
class SnowWarMapsManager:
    maps = {}  // mapId -> SnowWarMap

    function parseMap(mapId):
        // 1. Load item placements from arena file
        itemsFile = loadFile("arena_{mapId}.dat")
        items = []

        for each line in itemsFile.split(CR):  // Split by carriage return (char 13)
            parts = line.split(" ")
            item = new SnowWarItem(
                itemId: parts[0],
                itemName: parts[1],
                x: parseInt(parts[2]),
                y: parseInt(parts[3]),
                z: parseInt(parts[4]),
                rotation: parseInt(parts[5]),
                height: getItemHeight(parts[1])  // Lookup from SnowWarItemProperties
            )
            items.add(item)

        // 2. Load snowball machines (adds 3 tiles per machine)
        machinesFile = loadFile("arena_{mapId}_snowmachines.dat")
        for each line in machinesFile.split(CR):
            parts = line.split(" ")
            x = parseInt(parts[0])
            y = parseInt(parts[1])

            // Main machine tile
            items.add(new SnowWarItem("", "snowball_machine", x, y, 0, 0, 1))
            // Hidden collision tiles (extend machine hitbox)
            items.add(new SnowWarItem("", "snowball_machine_hidden", x + 1, y, 0, 0, 1))
            items.add(new SnowWarItem("", "snowball_machine_hidden", x + 2, y, 0, 0, 1))

        // 3. Load spawn clusters
        spawnsFile = loadFile("arena_{mapId}_spawn_clusters.dat")
        spawnClusters = []
        for each cluster in spawnsFile.split("|"):
            parts = cluster.split(" ")
            spawnClusters.add(new SpawnCluster(
                x: parseInt(parts[0]),
                y: parseInt(parts[1]),
                radius: parseInt(parts[2]),
                minDistance: parseInt(parts[3])
            ))

        // 4. Load heightmap and create map
        heightmap = loadHeightMap(mapId)
        maps[mapId] = new SnowWarMap(mapId, items, heightmap, spawnClusters)

    function getMap(mapId):
        return maps[mapId]
```

### 5.4 Heightmap Parsing and Tile Creation

When the map is created, the heightmap is parsed and combined with item data to create a 2D tile grid:

```pseudocode
class SnowWarMap:
    tiles = [][]      // 2D array of SnowWarTile
    mapSizeX = 0
    mapSizeY = 0

    function parseHeightMap():
        lines = heightMap.split("|")
        mapSizeY = lines.length
        mapSizeX = lines[0].length

        tiles = new SnowWarTile[mapSizeX][mapSizeY]

        for y = 0 to mapSizeY - 1:
            line = lines[y]

            for x = 0 to mapSizeX - 1:
                char = line.charAt(x)

                // Check if tile is blocked by heightmap
                isBlocked = (char == 'X' or char == 'x')

                // Find all items at this position
                itemsAtPosition = items.filter(item =>
                    item.x == x and item.y == y
                )

                // Sort items by Z height (stacking order)
                itemsAtPosition.sort(by: item.z)

                // Create tile with collision data
                tiles[x][y] = new SnowWarTile(x, y, isBlocked, itemsAtPosition)
```

### 5.5 Tile Walkability (Collision Detection)

A tile's walkability is determined by TWO factors:

```pseudocode
class SnowWarTile:
    x, y = 0
    isBlocked = false       // From heightmap ('X' = blocked)
    items = []              // Items placed on this tile
    highestItem = null      // Tallest item (for collision)

    constructor(x, y, isBlocked, items):
        this.x = x
        this.y = y
        this.isBlocked = isBlocked
        this.items = items

        // Find the highest item on this tile
        for each item in items:
            if highestItem == null or item.z > highestItem.z:
                highestItem = item

    // ========================================================================
    // PATHFINDER COLLISION: Can a player walk on this tile?
    // ========================================================================
    function isWalkable():
        // Rule 1: Heightmap blocks tile
        if isBlocked:
            return false

        // Rule 2: ANY item on tile blocks walking
        // (Items have height > 0, making the tile unwalkable)
        if highestItem != null:
            return false

        return true

    // ========================================================================
    // SNOWBALL COLLISION: Can a snowball pass over this tile?
    // ========================================================================
    function isHeightBlocking(trajectory):
        if highestItem == null:
            return false  // No obstacle

        // Long trajectory (high arc) passes over everything
        if trajectory == LONG_TRAJECTORY:
            return false

        // Short trajectory blocked by medium/tall obstacles
        if trajectory == SHORT_TRAJECTORY:
            return highestItem.height > 1

        // Quick throw blocked by any obstacle
        if trajectory == QUICK_THROW:
            return highestItem.height > 0

        return false
```

### 5.6 Item Properties Registry

Items have TWO height properties for different collision systems:

```pseudocode
class SnowWarItemProperties:
    // Registry: itemName -> (walkableHeight, collisionHeight)
    PROPERTIES = {
        // ====== TREES (block walking, block low snowballs) ======
        "sw_tree1":     (walkableHeight: 3, collisionHeight: 4600),
        "sw_tree2":     (walkableHeight: 3, collisionHeight: 4600),
        "sw_tree3":     (walkableHeight: 3, collisionHeight: 4600),
        "sw_tree4":     (walkableHeight: 3, collisionHeight: 4600),

        // ====== BASIC BLOCKS (various heights) ======
        "block_basic":  (walkableHeight: 1, collisionHeight: 2300),
        "block_basic2": (walkableHeight: 2, collisionHeight: 4600),
        "block_basic3": (walkableHeight: 3, collisionHeight: 6900),
        "block_small":  (walkableHeight: 0, collisionHeight: 1150),

        // ====== ICE BLOCKS ======
        "block_ice":    (walkableHeight: 1, collisionHeight: 2300),
        "block_ice2":   (walkableHeight: 2, collisionHeight: 4600),

        // ====== ARCH BLOCKS (bottom parts) ======
        "block_arch1b": (walkableHeight: 3, collisionHeight: 6900),
        "block_arch2b": (walkableHeight: 3, collisionHeight: 6900),
        "block_arch3b": (walkableHeight: 3, collisionHeight: 6900),

        // ====== ARCH BLOCKS (top parts - can walk under) ======
        "block_arch1":  (walkableHeight: 3, collisionHeight: 2300),
        "block_arch2":  (walkableHeight: 0, collisionHeight: 2300),
        "block_arch3":  (walkableHeight: 3, collisionHeight: 2300),

        // ====== OBSTACLES ======
        "obst_duck":    (walkableHeight: 1, collisionHeight: 2300),
        "obst_snowman": (walkableHeight: 3, collisionHeight: 4600),

        // ====== FENCE ======
        "sw_fence":     (walkableHeight: 1, collisionHeight: 2500),

        // ====== SNOWBALL MACHINE ======
        "snowball_machine":        (walkableHeight: 1, collisionHeight: 2400),
        "snowball_machine_hidden": (walkableHeight: 1, collisionHeight: 0),
    }

    // Used during map loading to set item.height for pathfinding
    function getWalkableHeight(itemName):
        return PROPERTIES[itemName].walkableHeight or 0

    // Used during snowball physics to check if ball hits obstacle
    function getCollisionHeight(itemName):
        return PROPERTIES[itemName].collisionHeight or -1
```

### 5.7 Pathfinder Integration

The pathfinder uses tile walkability plus player collision:

```pseudocode
class SnowWarPathfinder:
    MAX_PATHFIND_ITERATIONS = 50

    // Check if a tile is valid for movement
    function isValidTile(game, player, position):
        // Step 1: Get tile from map
        tile = game.getMap().getTile(position)

        // Step 2: Check tile exists and is walkable
        if tile == null or not tile.isWalkable():
            return false

        // Step 3: Check no other player is on/moving to this tile
        for each otherPlayer in game.getActivePlayers():
            if otherPlayer == player:
                continue

            attr = SnowWarPlayers.get(otherPlayer)

            // Check if player is moving to this tile
            if attr.nextGoal != null:
                if attr.nextGoal.equals(position):
                    return false
            else:
                // Check current position
                if attr.currentPosition.equals(position):
                    return false

        return true

    // Get next tile in path towards goal
    function getNextDirection(game, player):
        attr = SnowWarPlayers.get(player)
        positions = []

        // Check all 8 diagonal directions
        for each direction in DIAGONAL_MOVE_POINTS:
            candidatePos = attr.currentPosition.copy().add(direction)

            if isValidTile(game, player, candidatePos):
                positions.add(candidatePos)

        // Sort by distance to goal (closest first)
        positions.sort(by: pos.getDistanceSquared(attr.walkGoal))

        return positions.isEmpty() ? null : positions[0]
```

### 5.8 Collision Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MAP LOADING FLOW                                     │
│                                                                              │
│  arena_1.dat ───────────────┐                                               │
│  (items: x, y, z, name)     │                                               │
│                             ▼                                               │
│  arena_1_snowmachines.dat ──┼──► Parse Items ──► SnowWarItem[]              │
│  (machine positions)        │         │              │                      │
│                             │         │              ▼                      │
│  arena_1_heightmap.txt ─────┼─────────┼──────► Parse Heightmap              │
│  (X = blocked, 0 = open)    │         │              │                      │
│                             │         │              ▼                      │
│                             │         └────────► Create Tile Grid           │
│                             │                        │                      │
│                             │                        ▼                      │
│                             │              ┌─────────────────────┐          │
│                             │              │   SnowWarTile[x][y] │          │
│                             │              │   - isBlocked       │          │
│                             │              │   - items[]         │          │
│                             │              │   - highestItem     │          │
│                             │              └─────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                      RUNTIME COLLISION CHECKS                                │
│                                                                              │
│  PLAYER MOVEMENT:                                                           │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │ Player wants │ ──► │ Check tile   │ ──► │ Check other  │ ──► Allow/Deny │
│  │ to move      │     │ isWalkable() │     │ players      │                │
│  └──────────────┘     └──────────────┘     └──────────────┘                │
│                              │                                              │
│                              ▼                                              │
│                       isBlocked? ──► NO                                     │
│                              │                                              │
│                              ▼                                              │
│                       hasItem? ──► YES = BLOCKED                            │
│                                                                              │
│  SNOWBALL PHYSICS:                                                          │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │ Snowball at  │ ──► │ Get tile at  │ ──► │ Check item   │ ──► Hit/Pass   │
│  │ position     │     │ world coords │     │ collision    │                │
│  └──────────────┘     └──────────────┘     │ height       │                │
│                                            └──────────────┘                │
│                                                   │                         │
│                                                   ▼                         │
│                                     ballHeight < collisionHeight? ──► HIT   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Game Lifecycle

### 6.1 State Machine

```
WAITING ──[STARTGAME]──► STARTED (preparing) ──[timer=0]──► STARTED (playing)
                                                                  │
                                                    [timer=0 or all left]
                                                                  │
                                                                  ▼
                                              ENDED ──[restart/leave]──► WAITING
```

### 6.2 Initialization Flow

```pseudocode
// 1. Player enters lobby
function onPlayerEntersLobby(player, room):
    player.send(new LOUNGEINFO())
    player.send(new GAMEPLAYERINFO(SNOWWAR, room.players))
    room.broadcast(new GAMEPLAYERINFO(SNOWWAR, [player]))


// 2. Player creates game
function createGame(creator, parameters):
    mapId = parameters["fieldType"]
    teams = parameters["numTeams"]
    name = parameters["name"]
    lengthChoice = parameters["gameLengthChoice"]

    game = new SnowWarGame(generateId(), mapId, name, teams, creator, lengthChoice)

    gamePlayer = new GamePlayer(creator)
    gamePlayer.gameId = game.id
    gamePlayer.teamId = 0

    creator.roomUser.gamePlayer = gamePlayer
    game.movePlayer(gamePlayer, -1, 0)  // Add to team 0

    GameManager.games.add(game)


// 3. Other players join via INITIATEJOINGAME handler

// 4. Creator starts game
function handleStartGame(player):
    gamePlayer = player.roomUser.gamePlayer
    game = GameManager.getGameById(gamePlayer.gameId)

    if game.state != WAITING:
        return

    if game.creatorId != player.id:
        return

    if not game.canStart():
        player.send(new STARTFAILED())
        return

    game.startGame()
```

### 6.3 Game Start Sequence

```pseudocode
class SnowWarGame extends Game:

    function startGame():
        initialise()
        assignSpawnPoints()
        broadcast(new GAMELOCATION())
        startPreparingTimer()


    function initialise():
        // Create room model from height map
        model = new RoomModel("snowwar_arena_0", getHeightMap())

        // Initialize timers
        preparingSecondsLeft = getPreparingSeconds()
        totalSecondsLeft = calculateGameLengthSeconds()

        // Add snowball machines to objects list
        for each item in getMap().items:
            if item.isSnowballMachine():
                machine = new SnowWarMachineObject(
                    generateObjectId(),
                    item.x,
                    item.y,
                    MACHINE_MAX_CAPACITY
                )
                objects.add(machine)


    function assignSpawnPoints():
        for each player in getActivePlayers():
            generateSpawn(player)  // Random position within spawn cluster

            // Initialize player attributes
            attr = SnowWarPlayers.get(player)
            attr.activityState = NORMAL
            attr.snowballCount = MAX_SNOWBALLS
            attr.health = INITIAL_HEALTH
            attr.score = 0

            // Create avatar object
            avatar = new SnowWarAvatarObject(player)
            player.gameObject = avatar
            objects.add(avatar)


    function calculateGameLengthSeconds():
        // Check for server override
        configured = GameManager.getLifetimeSeconds(SNOWWAR)
        if configured > 0:
            return configured

        // Use player's selection
        switch gameLengthChoice:
            case 2: return 180  // 3 minutes
            case 3: return 300  // 5 minutes
            default: return 120 // 2 minutes
```

---

## 7. The Turn/Subturn System

The SnowWar game uses a **tick-based simulation** where the server processes game state at fixed intervals. Each server tick contains multiple **sub-turns** (frames) to create smooth movement without sending excessive network packets.

### 7.1 Two-Level Timing System

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SERVER TICK (300ms interval)                         │
│                                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Subturn  │  │ Subturn  │  │ Subturn  │  │ Subturn  │  │ Subturn  │       │
│  │    0     │  │    1     │  │    2     │  │    3     │  │    4     │       │
│  │  (60ms)  │  │  (60ms)  │  │  (60ms)  │  │  (60ms)  │  │  (60ms)  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│       │             │             │             │             │              │
│       ▼             ▼             ▼             ▼             ▼              │
│   [events]      [events]      [events]      [events]      [events]          │
│                                                                              │
│  After all subturns: Calculate checksum, send GAMESTATUS packet             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Exact Timing Values

| Parameter | Value | Description |
|-----------|-------|-------------|
| **Server Tick Interval** | **300ms** | Time between scheduler invocations |
| **Subturns Per Tick** | **5** | Game frames processed per tick |
| **Subturn Duration** | **60ms** | Effective time per subturn (300ms ÷ 5) |
| **Packets Per Second** | **~3.33** | GAMESTATUS packets sent per second |
| **Game Frames Per Second** | **~16.67** | Subturns processed per second (5 × 3.33) |

```pseudocode
// Task scheduling (in RoomTaskManager)
scheduleTask("UpdateTask", new SnowWarGameTask(room, game),
             initialDelay: 0,
             period: 300,
             unit: MILLISECONDS)

// Constants
SUBTURNS_PER_TICK = 5        // MAX_GAME_TURNS in code
SERVER_TICK_MS = 300         // Scheduler period
SUBTURN_MS = 60              // 300 / 5 = 60ms per subturn
```

### 7.3 Timer Conversions

Activity timers in the code are measured in **subturns (frames)**, not milliseconds:

```pseudocode
// Convert timer values to real time:
STUNNED_TIMER = 125 frames     → 125 × 60ms = 7,500ms = 7.5 seconds
INVINCIBILITY_TIMER = 60 frames → 60 × 60ms = 3,600ms = 3.6 seconds
CREATING_TIMER = 20 frames      → 20 × 60ms = 1,200ms = 1.2 seconds
MACHINE_GENERATOR_TIME = 100    → 100 × 60ms = 6,000ms = 6.0 seconds
```

**Why this design?**

1. **Network efficiency**: Instead of sending 5 packets per tick, send 1 packet containing 5 subturns of events
2. **Smooth animation**: Client interpolates between subturns for fluid movement
3. **Deterministic simulation**: Both client and server simulate the same subturns, verified by checksum

### 7.5 Main Game Loop

```pseudocode
class SnowWarGameTask:

    const SUBTURNS_PER_TICK = 5

    currentTurn = 0          // Increments each tick (not subturn)
    currentChecksum = 0

    snowballs = []           // Active projectiles
    turnEventsList = []      // List of lists: events per subturn


    // Called by scheduler every ~200ms
    function run():
        if game.isFinished():
            return

        // Queue movement events for players who are walking
        for each player in game.activePlayers:
            if player.attributes.isWalking:
                queueMovementEvent(player)

        // Process all subturns and build event lists
        processSubturns()

        // Send single packet with all subturn data
        packet = new GAMESTATUS(turnEventsList, currentTurn, currentChecksum)
        broadcast(packet)


    function processSubturns():
        turnEventsList = []
        currentTurn++

        delayedCollisionEvents = []
        machinesPendingSnowball = []

        // Process each subturn
        for subturnIndex = 0 to SUBTURNS_PER_TICK - 1:

            // Create event list for this subturn
            subturnEvents = []
            turnEventsList.append(subturnEvents)

            // Snapshot current snowballs (avoid modification during iteration)
            snowballSnapshot = copy(snowballs)

            // === PHASE 1: Move all avatars ===
            for each player in game.activePlayers:
                avatar = player.gameObject

                // Calculate one frame of movement
                avatar.calculateFrameMovement()

                // Check if any snowball hit this player
                collisions = avatar.checkCollisions(snowballSnapshot)
                delayedCollisionEvents.addAll(collisions)

            // === PHASE 2: Move all snowballs ===
            for each ball in snowballSnapshot:
                ball.calculateFrameMovement()

                if ball.hitGround():
                    snowballs.remove(ball)
                    subturnEvents.add(new DeleteObjectEvent(ball.id))

            // === PHASE 3: Process snowball machines ===
            for each machine in game.machines:
                if machine.shouldGenerateSnowball():
                    subturnEvents.add(new MachineAddSnowballEvent(machine.id))
                    machinesPendingSnowball.add(machine)

        // === PHASE 4: Calculate checksum BEFORE applying deferred state ===
        currentChecksum = calculateChecksum(currentTurn)

        // === PHASE 5: Apply deferred state changes ===
        for each machine in machinesPendingSnowball:
            machine.snowballCount++

        // === PHASE 6: Process machine pickups ===
        for each player in game.activePlayers:
            attr = player.attributes
            for each machine in game.machines:
                if machine.canPlayerPickup(attr):
                    turnEventsList[0].add(new MachineTransferEvent(player.id, machine.id))
                    machine.transferSnowballTo(attr)
                    break

        // === PHASE 7: Apply collision results (hits, stuns) ===
        processDelayedCollisions(delayedCollisionEvents)


    function queueMovementEvent(player):
        attr = player.attributes
        event = new AvatarMoveEvent(
            player.objectId,
            attr.goalWorldX,
            attr.goalWorldY
        )
        turnEventsList[0].add(event)  // Add to first subturn
```

### 7.4 Timing Diagram

```
Time (ms)  0        300        600        900       1200
           │         │          │          │          │
           ▼         ▼          ▼          ▼          ▼
           ├─────────┼──────────┼──────────┼──────────┤
           │  Tick 1 │  Tick 2  │  Tick 3  │  Tick 4  │
           └─────────┴──────────┴──────────┴──────────┘


Detail of one server tick (300ms):
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SERVER TICK (300ms)                                 │
│                                                                                  │
│  0ms      60ms     120ms     180ms     240ms     300ms                          │
│   │        │         │         │         │         │                             │
│   ▼        ▼         ▼         ▼         ▼         ▼                             │
│ ┌────┐  ┌────┐   ┌────┐   ┌────┐   ┌────┐                                       │
│ │ S0 │  │ S1 │   │ S2 │   │ S3 │   │ S4 │  → Checksum → Send GAMESTATUS        │
│ └────┘  └────┘   └────┘   └────┘   └────┘                                       │
│                                                                                  │
│ Each subturn (S0-S4):                                                           │
│   1. Move all avatars one frame                                                 │
│   2. Check collisions                                                           │
│   3. Move all snowballs one frame                                               │
│   4. Process machine generators                                                 │
│   5. Collect events for this subturn                                            │
└─────────────────────────────────────────────────────────────────────────────────┘


Full timeline example:
┌───────────────────────┐    ┌───────────────────────┐    ┌───────────────────────┐
│ Turn 1  (0-300ms)     │    │ Turn 2  (300-600ms)   │    │ Turn 3  (600-900ms)   │
│                       │    │                       │    │                       │
│ Subturn 0: [events]   │    │ Subturn 0: [events]   │    │ Subturn 0: [events]   │
│ Subturn 1: [events]   │    │ Subturn 1: [events]   │    │ Subturn 1: [events]   │
│ Subturn 2: [events]   │    │ Subturn 2: [events]   │    │ Subturn 2: [events]   │
│ Subturn 3: [events]   │    │ Subturn 3: [events]   │    │ Subturn 3: [events]   │
│ Subturn 4: [events]   │    │ Subturn 4: [events]   │    │ Subturn 4: [events]   │
│                       │    │                       │    │                       │
│ Checksum: 0x1A2B3C4D  │    │ Checksum: 0x5E6F7A8B  │    │ Checksum: 0x9C0D1E2F  │
└───────────┬───────────┘    └───────────┬───────────┘    └───────────┬───────────┘
            │                            │                            │
            ▼                            ▼                            ▼
      [GAMESTATUS]                 [GAMESTATUS]                 [GAMESTATUS]
       @ 300ms                      @ 600ms                      @ 900ms
            │                            │                            │
            ▼                            ▼                            ▼
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENT                                          │
│                                                                                    │
│  Receives packet, simulates same 5 subturns locally                               │
│  Compares calculated checksum with server checksum                                │
│  If mismatch: request FULLGAMESTATUS to resync                                    │
│  Interpolates between subturns for smooth 60fps animation                         │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Player Attributes System

### 8.1 Player Registry

A static registry keeps SnowWar-specific state separate from the generic `GamePlayer`:

```pseudocode
class SnowWarPlayers:
    static attributes = {}  // userId -> SnowWarAttributes

    static function get(player):
        if player.userId not in attributes:
            attributes[player.userId] = new SnowWarAttributes()
        return attributes[player.userId]

    static function remove(player):
        delete attributes[player.userId]
```

### 8.2 Player Attributes

```pseudocode
class SnowWarAttributes:
    // Movement
    isWalking = false
    currentPosition = null       // Current tile position
    worldPosition = null         // High-precision world position
    walkGoal = null              // Final destination tile
    nextGoal = null              // Next tile in path
    goalWorldCoordinates = null  // Raw client coordinates

    // Combat
    snowballCount = 0            // Current ammo (atomic)
    health = 0                   // Current HP (atomic)
    pendingHealth = 0            // Pending HP change for sync
    pendingStun = false          // Pending stun for sync
    lastThrowTime = 0            // Cooldown timestamp
    immunityExpiry = 0           // Invincibility timestamp

    // State
    activityState = NORMAL
    activityTimer = 0            // Frames remaining in current state
    rotation = 0                 // 0-7 facing direction

    // Score
    score = 0


    function isWalkable():
        return activityState == NORMAL
            or activityState == INVINCIBLE


    function isDamageable():
        return currentTimeMillis() > immunityExpiry
            and activityState == NORMAL
```

### 8.3 Activity States

```pseudocode
enum SnowWarActivityState:
    NORMAL              = (stateId: 0, timer: 0)
    CREATING_SNOWBALL   = (stateId: 1, timer: 20)    // Making snowball
    STUNNED             = (stateId: 2, timer: 125)   // Knocked out
    INVINCIBLE          = (stateId: 3, timer: 60)    // Recovery immunity
```

---

## 9. Game Objects

### 9.1 Base Class

```pseudocode
abstract class SnowWarGameObject extends GameObject:
    syncValues = []  // Reusable list for checksum calculation

    abstract function calculateFrameMovement()
    abstract function getChecksumValues() -> List<Integer>
    abstract function isAlive() -> boolean


    function getChecksumContribution():
        values = getChecksumValues()
        sum = 0
        for i = 0 to values.length - 1:
            sum += values[i] * (i + 1)  // Weighted by position
        return sum
```

### 9.2 Avatar Object

```pseudocode
class SnowWarAvatarObject extends SnowWarGameObject:
    player = null


    function calculateFrameMovement():
        attr = SnowWarPlayers.get(player)

        // === Handle activity state timer ===
        if attr.activityTimer > 0:
            attr.activityTimer--

            if attr.activityTimer == 0:
                onActivityTimerExpired()

        // === Handle movement ===
        if not hasDestination():
            return

        if not canMoveInCurrentState():
            return

        // Try to move one frame towards destination
        reachedDestination = moveOneFrame()

        if reachedDestination:
            stopWalking()


    function moveOneFrame():
        attr = SnowWarPlayers.get(player)

        currentWorldX = attr.worldPosition.x
        currentWorldY = attr.worldPosition.y

        targetWorldX = tileToWorld(attr.walkGoal.x)
        targetWorldY = tileToWorld(attr.walkGoal.y)

        // Already at destination?
        if currentWorldX == targetWorldX and currentWorldY == targetWorldY:
            return true

        // Get next tile in path
        nextTile = attr.nextGoal
        if nextTile == null:
            nextTile = findNextTileTowards(attr.walkGoal)
            attr.nextGoal = nextTile

        nextWorldX = tileToWorld(nextTile.x)
        nextWorldY = tileToWorld(nextTile.y)

        // Move towards next tile by SUBTURN_MOVEMENT units
        currentWorldX = moveTowards(currentWorldX, nextWorldX, SUBTURN_MOVEMENT)
        currentWorldY = moveTowards(currentWorldY, nextWorldY, SUBTURN_MOVEMENT)

        // Update world position
        attr.worldPosition = (currentWorldX, currentWorldY)

        // Update tile position if crossed boundary
        newTileX = worldToTile(currentWorldX)
        newTileY = worldToTile(currentWorldY)

        if newTileX != attr.currentPosition.x or newTileY != attr.currentPosition.y:
            attr.currentPosition = (newTileX, newTileY)

        // Check if reached next tile
        if currentWorldX == nextWorldX and currentWorldY == nextWorldY:
            attr.nextGoal = null

        return currentWorldX == targetWorldX and currentWorldY == targetWorldY


    function moveTowards(current, target, maxStep):
        delta = target - current
        if delta == 0:
            return current
        if abs(delta) <= maxStep:
            return target
        if delta < 0:
            return current - maxStep
        else:
            return current + maxStep


    function onActivityTimerExpired():
        attr = SnowWarPlayers.get(player)

        switch attr.activityState:
            case CREATING_SNOWBALL:
                attr.activityState = NORMAL
                attr.snowballCount++

            case STUNNED:
                attr.activityState = INVINCIBLE
                attr.activityTimer = INVINCIBILITY_FRAMES
                attr.health = INITIAL_HEALTH

            case INVINCIBLE:
                attr.activityState = NORMAL


    function canMoveInCurrentState():
        attr = SnowWarPlayers.get(player)
        state = attr.activityState
        return state == NORMAL or state == CREATING_SNOWBALL or state == INVINCIBLE


    function getChecksumValues():
        attr = SnowWarPlayers.get(player)
        return [
            OBJECT_TYPE_AVATAR,        // 0
            player.objectId,           // 1
            attr.worldPosition.x,      // 2
            attr.worldPosition.y,      // 3
            attr.rotation,             // 4
            attr.health,               // 5
            attr.snowballCount,        // 6
            0,                         // 7: is_bot
            attr.activityTimer,        // 8
            attr.activityState.id,     // 9
            attr.nextGoal.x,           // 10
            attr.nextGoal.y,           // 11
            attr.walkGoal.worldX,      // 12
            attr.walkGoal.worldY,      // 13
            attr.score,                // 14
            player.userId,             // 15
            player.teamId,             // 16
            player.objectId            // 17
        ]


    function serialize(response):
        attr = SnowWarPlayers.get(player)
        response.writeInt(OBJECT_TYPE_AVATAR)
        response.writeInt(player.objectId)
        response.writeInt(attr.worldPosition.x)
        response.writeInt(attr.worldPosition.y)
        response.writeInt(attr.rotation)
        response.writeInt(attr.health)
        response.writeInt(attr.snowballCount)
        response.writeInt(0)  // is_bot
        response.writeInt(attr.activityTimer)
        response.writeInt(attr.activityState.id)
        response.writeInt(attr.nextGoal.x)
        response.writeInt(attr.nextGoal.y)
        response.writeInt(attr.walkGoal.worldX)
        response.writeInt(attr.walkGoal.worldY)
        response.writeInt(attr.score)
        response.writeInt(player.userId)
        response.writeInt(player.teamId)
        response.writeInt(player.objectId)
        response.writeString(player.name)
        response.writeString(player.motto)
        response.writeString(player.figure)
        response.writeString(player.gender)
```

### 9.3 Snowball Object

```pseudocode
class SnowWarSnowballObject extends SnowWarGameObject:
    thrower = null

    // Position in world coordinates
    locH = 0       // Horizontal (X)
    locV = 0       // Vertical (Y)
    height = 0     // Z height (for parabola)

    // Flight parameters
    direction = 0         // 0-359 degrees
    timeToLive = 0        // Frames until landing
    parabolaOffset = 0    // Peak of arc
    trajectory = 0        // 0=quick, 1=lob, 2=long

    alive = true


    function calculateFrameMovement():
        if not alive:
            return

        timeToLive--

        // === Calculate horizontal movement ===
        // Use lookup tables for velocity based on direction
        deltaH = (BASE_VELOCITY_X[direction] * 2000) / 255
        deltaV = (BASE_VELOCITY_Y[direction] * 2000) / 255

        locH = locH + deltaH
        locV = locV + deltaV

        // === Calculate parabolic height ===
        distanceFromPeak = timeToLive - parabolaOffset

        switch trajectory:
            case 0:  // Quick throw (flat arc)
                heightMultiplier = 4
                baseHeight = 4000
            case 1:  // Lob (medium arc)
                heightMultiplier = 10
                baseHeight = 3000
            case 2:  // Long throw (high arc)
                heightMultiplier = 100
                baseHeight = 3000

        // Parabola formula
        height = baseHeight + heightMultiplier * (parabolaOffset^2 - distanceFromPeak^2)

        // === Check for ground/obstacle collision ===
        if height < 0:
            alive = false
            return

        currentTileX = worldToTile(locH)
        currentTileY = worldToTile(locV)

        for each obstacle in game.map.obstacles:
            if obstacle.x == currentTileX and obstacle.y == currentTileY:
                if height < obstacle.collisionHeight:
                    alive = false
                    return


    function getChecksumValues():
        return [
            OBJECT_TYPE_SNOWBALL,      // 0
            objectId,                  // 1
            locH,                      // 2
            locV,                      // 3
            height,                    // 4
            direction,                 // 5
            trajectory,                // 6
            timeToLive,                // 7
            thrower.objectId,          // 8
            parabolaOffset             // 9
        ]


    function serialize(response):
        response.writeInt(OBJECT_TYPE_SNOWBALL)
        response.writeInt(objectId)
        response.writeInt(locH)
        response.writeInt(locV)
        response.writeInt(height)
        response.writeInt(direction)
        response.writeInt(trajectory)
        response.writeInt(timeToLive)
        response.writeInt(thrower.objectId)
        response.writeInt(parabolaOffset)
```

### 9.4 Machine Object

```pseudocode
class SnowWarMachineObject extends SnowWarGameObject:
    x = 0
    y = 0
    snowballCount = 0
    generatorTimer = MACHINE_GENERATOR_TIME


    function calculateFrameMovement():
        // Machines don't move
        pass


    function processGeneratorTick():
        if generatorTimer > 0:
            generatorTimer--
            return false

        generatorTimer = MACHINE_GENERATOR_TIME
        return canGenerateSnowball()


    function canGenerateSnowball():
        return snowballCount < MACHINE_MAX_CAPACITY


    function canPlayerPickup(attr):
        if not isPlayerAtPickupPosition(attr):
            return false
        if attr.isWalking:
            return false
        if attr.activityState != NORMAL and attr.activityState != INVINCIBLE:
            return false
        return snowballCount > 0 and attr.snowballCount < MAX_SNOWBALLS


    function isPlayerAtPickupPosition(attr):
        pickupX = x
        pickupY = y + 1
        return attr.currentPosition.x == pickupX and attr.currentPosition.y == pickupY


    function transferSnowballTo(attr):
        if snowballCount > 0 and attr.snowballCount < MAX_SNOWBALLS:
            snowballCount--
            attr.snowballCount++


    function getChecksumValues():
        return [
            OBJECT_TYPE_MACHINE,       // 0
            objectId,                  // 1
            tileToWorld(x),            // 2
            tileToWorld(y),            // 3
            snowballCount              // 4
        ]


    function serialize(response):
        response.writeInt(OBJECT_TYPE_MACHINE)
        response.writeInt(objectId)
        response.writeInt(convertToWorldCoordinate(x))
        response.writeInt(convertToWorldCoordinate(y))
        response.writeInt(snowballCount)
```

---

## 10. Collision Detection

```pseudocode
class SnowWarAvatarObject:

    function checkCollisions(snowballs):
        delayedEvents = []

        if snowballs is empty:
            return delayedEvents

        if isImmune():
            return delayedEvents

        for each ball in snowballs:
            if testCollision(ball):
                event = processHit(ball)
                if event != null:
                    delayedEvents.add(event)

        return delayedEvents


    function testCollision(ball):
        // Snowball must be low enough to hit player
        if ball.height > BALL_HEIGHT_THRESHOLD:
            return false

        // Can't hit yourself or teammates
        if not game.isOpponent(player, ball.thrower):
            return false

        // Circle-to-circle collision
        attr = SnowWarPlayers.get(player)

        playerX = attr.worldPosition.x
        playerY = attr.worldPosition.y

        ballX = ball.locH
        ballY = ball.locV

        distanceX = abs(ballX - playerX)
        distanceY = abs(ballY - playerY)

        // Quick rejection
        if distanceX >= COLLISION_RADIUS:
            return false
        if distanceY >= COLLISION_RADIUS:
            return false

        // Actual circle test
        distanceSquared = distanceX^2 + distanceY^2
        return distanceSquared < COLLISION_RADIUS^2


    function processHit(ball):
        attr = SnowWarPlayers.get(player)

        if attr.pendingHealth > 0:
            // Take damage, not yet stunned
            attr.pendingHealth--
            game.removeSnowball(ball)
            queueHitEvent(ball.thrower, player, ball)
            return new DelayedEvent(HIT, player, ball)

        else:
            // Health depleted, get stunned
            if not attr.pendingStun:
                attr.pendingStun = true
                game.removeSnowball(ball)
                queueStunEvent(ball.thrower, player, ball)
                return new DelayedEvent(STUN, player, ball)

        return null


    function isImmune():
        attr = SnowWarPlayers.get(player)
        return attr.activityState == STUNNED or attr.activityState == INVINCIBLE


// Process delayed events AFTER checksum calculation
function processDelayedCollisions(events):
    for each event in events:
        switch event.type:

            case HIT:
                attr = SnowWarPlayers.get(event.player)
                attr.health = attr.pendingHealth

                throwerAttr = SnowWarPlayers.get(event.ball.thrower)
                throwerAttr.score += HIT_SCORE

            case STUN:
                attr = SnowWarPlayers.get(event.player)
                stopWalking(event.player)

                attr.activityState = STUNNED
                attr.activityTimer = STUNNED_FRAMES
                attr.snowballCount = 0  // Drop all snowballs

                throwerAttr = SnowWarPlayers.get(event.ball.thrower)
                throwerAttr.score += STUN_SCORE
```

---

## 11. Coordinate Systems & Math Utilities

### 11.1 World vs Tile Coordinates

SnowWar uses two coordinate systems:

- **Tile coordinates**: Integer grid positions (0-50 typically)
- **World coordinates**: High-precision positions (tile * 3200)

The multiplier `3200` comes from: `tileSize (32) * accuracyFactor (100) = 3200`

### 11.2 Core Coordinate Conversion Functions

```pseudocode
// ========================================================================
// COORDINATE CONVERSION FUNCTIONS
// ========================================================================
// NOTE: tileToWorld and convertToWorldCoordinate are MATHEMATICALLY IDENTICAL
// Both return: num * 3200
// The difference is SEMANTIC - they're used in different contexts:
//   - tileToWorld: Used for INTERNAL calculations and CHECKSUM values
//   - convertToWorldCoordinate: Used for CLIENT SERIALIZATION (packet writing)
// ========================================================================

function tileToWorld(num):
    return num * 3200

function worldToTile(num):
    // Adds half a tile (1600) for rounding to nearest tile
    return (num + 1600) / 3200

function convertToWorldCoordinate(num):
    accuracyFactor = 100
    tileSize = 32
    multiplier = tileSize * accuracyFactor  // = 3200
    return num * multiplier

function convertToGameCoordinate(num):
    accuracyFactor = 100
    tileSize = 32
    multiplier = tileSize * accuracyFactor  // = 3200
    return num / multiplier
```

### 11.3 Usage Pattern: Checksum vs Serialization

**IMPORTANT**: Although `tileToWorld` and `convertToWorldCoordinate` produce the same result, the codebase uses them in specific contexts for clarity:

#### Machine Object Example:
```pseudocode
class SnowWarMachineObject:
    x = 0  // Stored in TILE coordinates
    y = 0

    // CHECKSUM: Uses tileToWorld
    function getChecksumValues():
        values = []
        values.add(OBJECT_TYPE_MACHINE)
        values.add(objectId)
        values.add(tileToWorld(x))       // <-- tileToWorld for checksum
        values.add(tileToWorld(y))       // <-- tileToWorld for checksum
        values.add(snowballCount)
        return values

    // SERIALIZATION: Uses convertToWorldCoordinate
    function serialize(response):
        response.writeInt(OBJECT_TYPE_MACHINE)
        response.writeInt(objectId)
        response.writeInt(convertToWorldCoordinate(x))  // <-- convertToWorldCoordinate for client
        response.writeInt(convertToWorldCoordinate(y))  // <-- convertToWorldCoordinate for client
        response.writeInt(snowballCount)
```

#### Avatar Object Example:
```pseudocode
class SnowWarAvatarObject:
    // CHECKSUM: Uses tileToWorld for target coordinates
    function getChecksumValues():
        values = []
        values.add(OBJECT_TYPE_AVATAR)
        values.add(objectId)
        values.add(tileToWorld(currentPosition.x))      // <-- tileToWorld
        values.add(tileToWorld(currentPosition.y))      // <-- tileToWorld
        values.add(rotation)
        values.add(health)
        values.add(snowballCount)
        values.add(0)  // is_bot
        values.add(activityTimer)
        values.add(activityState)
        values.add(nextGoal.x)
        values.add(nextGoal.y)
        values.add(tileToWorld(checksumTargetX))        // <-- tileToWorld
        values.add(tileToWorld(checksumTargetY))        // <-- tileToWorld
        values.add(score)
        values.add(userId)
        values.add(teamId)
        values.add(objectId)
        return values

    // SERIALIZATION: Uses convertToWorldCoordinate
    function serialize(response):
        response.writeInt(OBJECT_TYPE_AVATAR)
        response.writeInt(objectId)
        response.writeInt(convertToWorldCoordinate(currentPosition.x))  // <-- convertToWorldCoordinate
        response.writeInt(convertToWorldCoordinate(currentPosition.y))  // <-- convertToWorldCoordinate
        response.writeInt(rotation)
        // ... etc
```

#### Snowball Object Example:
```pseudocode
class SnowWarSnowballObject:
    // Snowballs store positions in WORLD coordinates from the start
    locH = 0  // Already in world coords (initialized via tileToWorld)
    locV = 0

    function initialize(fromX, fromY):
        // Convert ONCE at creation time
        locH = tileToWorld(fromX)
        locV = tileToWorld(fromY)

    // BOTH checksum and serialize use raw world coords
    // (no conversion needed - already in world coordinates)
    function getChecksumValues():
        values = []
        values.add(OBJECT_TYPE_SNOWBALL)
        values.add(objectId)
        values.add(locH)           // <-- Already world coords
        values.add(locV)           // <-- Already world coords
        values.add(height)
        values.add(direction)
        values.add(trajectory)
        values.add(timeToLive)
        values.add(thrower.objectId)
        values.add(parabolaOffset)
        return values

    function serialize(response):
        response.writeInt(OBJECT_TYPE_SNOWBALL)
        response.writeInt(objectId)
        response.writeInt(locH)    // <-- Already world coords
        response.writeInt(locV)    // <-- Already world coords
        response.writeInt(height)
        // ... etc
```

### 11.4 Direction System

- **8-direction** (rotation): 0=N, 1=NE, 2=E, 3=SE, 4=S, 5=SW, 6=W, 7=NW
- **360-direction**: 0-359 degrees for projectiles

```pseudocode
function direction360To8(angle360):
    validated = validateDirection360(angle360 - 22)
    return validateDirection8((validated / 45) + 1)

function validateDirection360(value):
    if value > 359:
        return value % 360
    if value < 0:
        return 360 + (value % 360)
    return value

function validateDirection8(value):
    if value > 7:
        return value % 8
    if value < 0:
        return (8 + (value % 8)) % 8
    return value

function rotateDirection45DegreesCw(value):
    return validateDirection360(value + 45)

function rotateDirection45DegreesCcw(value):
    return validateDirection360(value - 45)
```

---

## 12. Complete Math Utilities

This section contains all mathematical functions and lookup tables required for SnowWar physics simulation.

### 12.1 Lookup Tables

```pseudocode
// ========================================================================
// LOOKUP TABLE: Fast Square Root
// Used for distance calculations without expensive sqrt operations
// Index: scaled input value, Value: approximate sqrt * 16
// ========================================================================
TABLE = [
    0, 16, 22, 27, 32, 35, 39, 42, 45, 48, 50, 53, 55, 57, 59, 61, 64, 65, 67, 69, 71, 73, 75, 76, 78,
    80, 81, 83, 84, 86, 87, 89, 90, 91, 93, 94, 96, 97, 98, 99, 101, 102, 103, 104, 106, 107, 108, 109,
    110, 112, 113, 114, 115, 116, 117, 118, 119, 120, 121, 122, 123, 124, 125, 126, 128, 128, 129, 130, 131,
    132, 133, 134, 135, 136, 137, 138, 139, 140, 141, 142, 143, 144, 144, 145, 146, 147, 148, 149, 150, 150,
    151, 152, 153, 154, 155, 155, 156, 157, 158, 159, 160, 160, 161, 162, 163, 163, 164, 165, 166, 167, 167,
    168, 169, 170, 170, 171, 172, 173, 173, 174, 175, 176, 176, 177, 178, 178, 179, 180, 181, 181, 182, 183,
    183, 184, 185, 185, 186, 187, 187, 188, 189, 189, 190, 191, 192, 192, 193, 193, 194, 195, 195, 196, 197,
    197, 198, 199, 199, 200, 201, 201, 202, 203, 203, 204, 204, 205, 206, 206, 207, 208, 208, 209, 209, 210,
    211, 211, 212, 212, 213, 214, 214, 215, 215, 216, 217, 217, 218, 218, 219, 219, 220, 221, 221, 222, 222,
    223, 224, 224, 225, 225, 226, 226, 227, 227, 228, 229, 229, 230, 230, 231, 231, 232, 232, 233, 234, 234,
    235, 235, 236, 236, 237, 237, 238, 238, 239, 240, 240, 241, 241, 242, 242, 243, 243, 244, 244, 245, 245,
    246, 246, 247, 247, 248, 248, 249, 249, 250, 250, 251, 251, 252, 252, 253, 253, 254, 254, 255
]

// ========================================================================
// LOOKUP TABLE: Angle Components
// Used to convert X/Y ratios to degrees (0-45 range)
// Index: (component * 256 / other_component), Value: angle in degrees
// ========================================================================
COMPONENT = [
    0, 0, 0, 1, 1, 1, 1, 2, 2, 2, 2, 2, 3, 3, 3, 3, 4, 4, 4, 4, 4, 5, 5, 5, 5, 6, 6, 6, 6, 6, 7,
    7, 7, 7, 8, 8, 8, 8, 8, 9, 9, 9, 9, 10, 10, 10, 10, 10, 11, 11, 11, 11, 12, 12, 12, 12, 12, 13, 13,
    13, 13, 13, 14, 14, 14, 14, 15, 15, 15, 15, 15, 16, 16, 16, 16, 16, 17, 17, 17, 17, 17, 18, 18, 18,
    18, 18, 19, 19, 19, 19, 19, 20, 20, 20, 20, 20, 21, 21, 21, 21, 21, 22, 22, 22, 22, 22, 23, 23, 23,
    23, 23, 24, 24, 24, 24, 24, 24, 25, 25, 25, 25, 25, 26, 26, 26, 26, 26, 26, 27, 27, 27, 27, 27, 28,
    28, 28, 28, 28, 28, 29, 29, 29, 29, 29, 29, 30, 30, 30, 30, 30, 30, 31, 31, 31, 31, 31, 31, 32, 32,
    32, 32, 32, 32, 33, 33, 33, 33, 33, 33, 34, 34, 34, 34, 34, 34, 34, 35, 35, 35, 35, 35, 35, 36, 36,
    36, 36, 36, 36, 36, 37, 37, 37, 37, 37, 37, 37, 38, 38, 38, 38, 38, 38, 38, 39, 39, 39, 39, 39, 39,
    39, 39, 40, 40, 40, 40, 40, 40, 40, 41, 41, 41, 41, 41, 41, 41, 41, 42, 42, 42, 42, 42, 42, 42, 42,
    43, 43, 43, 43, 43, 43, 43, 43, 44, 44, 44, 44, 44, 44, 44, 44, 44, 45, 45, 45, 45, 45
]

// ========================================================================
// LOOKUP TABLE: Base Velocity X (Horizontal)
// Index: direction in degrees (0-359), Value: X velocity component (-256 to 256)
// 0 degrees = East, 90 degrees = South, 180 degrees = West, 270 degrees = North
// ========================================================================
BASE_VEL_X = [
    0, 4, 8, 13, 17, 22, 26, 31, 35, 40, 44, 48, 53, 57, 61, 66, 70, 74, 79, 83, 87, 91, 95, 100, 104,
    108, 112, 116, 120, 124, 127, 131, 135, 139, 143, 146, 150, 154, 157, 161, 164, 167, 171, 174, 177, 181,
    184, 187, 190, 193, 196, 198, 201, 204, 207, 209, 212, 214, 217, 219, 221, 223, 226, 228, 230, 232, 233,
    235, 237, 238, 240, 242, 243, 244, 246, 247, 248, 249, 250, 251, 252, 252, 253, 254, 254, 255, 255, 255,
    255, 255, 256, 255, 255, 255, 255, 255, 254, 254, 253, 252, 252, 251, 250, 249, 248, 247, 246, 244, 243,
    242, 240, 238, 237, 235, 233, 232, 230, 228, 226, 223, 221, 219, 217, 214, 212, 209, 207, 204, 201, 198,
    196, 193, 190, 187, 184, 181, 177, 174, 171, 167, 164, 161, 157, 154, 150, 146, 143, 139, 135, 131, 127,
    124, 120, 116, 112, 108, 104, 100, 95, 91, 87, 83, 79, 74, 70, 66, 61, 57, 53, 48, 44, 40, 35, 31, 26,
    22, 17, 13, 8, 4, 0, -4, -8, -13, -17, -22, -26, -31, -35, -40, -44, -48, -53, -57, -61, -66, -70, -74,
    -79, -83, -87, -91, -95, -100, -104, -108, -112, -116, -120, -124, -128, -131, -135, -139, -143, -146,
    -150, -154, -157, -161, -164, -167, -171, -174, -177, -181, -184, -187, -190, -193, -196, -198, -201, -204,
    -207, -209, -212, -214, -217, -219, -221, -223, -226, -228, -230, -232, -233, -235, -237, -238, -240, -242,
    -243, -244, -246, -247, -248, -249, -250, -251, -252, -252, -253, -254, -254, -255, -255, -255, -255, -255,
    -256, -255, -255, -255, -255, -255, -254, -254, -253, -252, -252, -251, -250, -249, -248, -247, -246, -244,
    -243, -242, -240, -238, -237, -235, -233, -232, -230, -228, -226, -223, -221, -219, -217, -214, -212, -209,
    -207, -204, -201, -198, -196, -193, -190, -187, -184, -181, -177, -174, -171, -167, -164, -161, -157, -154,
    -150, -146, -143, -139, -135, -131, -128, -124, -120, -116, -112, -108, -104, -100, -95, -91, -87, -83,
    -79, -74, -70, -66, -61, -57, -53, -48, -44, -40, -35, -31, -26, -22, -17, -13, -8, -4
]

// ========================================================================
// LOOKUP TABLE: Base Velocity Y (Vertical)
// Index: direction in degrees (0-359), Value: Y velocity component (-256 to 256)
// 0 degrees = East, 90 degrees = South, 180 degrees = West, 270 degrees = North
// Note: Y is 90 degrees offset from X (cos/sin relationship)
// ========================================================================
BASE_VEL_Y = [
    -256, -255, -255, -255, -255, -255, -254, -254, -253, -252, -252, -251, -250, -249, -248, -247, -246, -244,
    -243, -242, -240, -238, -237, -235, -233, -232, -230, -228, -226, -223, -221, -219, -217, -214, -212, -209,
    -207, -204, -201, -198, -196, -193, -190, -187, -184, -181, -177, -174, -171, -167, -164, -161, -157, -154,
    -150, -146, -143, -139, -135, -131, -128, -124, -120, -116, -112, -108, -104, -100, -95, -91, -87, -83, -79,
    -74, -70, -66, -61, -57, -53, -48, -44, -40, -35, -31, -26, -22, -17, -13, -8, -4, 0, 4, 8, 13, 17, 22,
    26, 31, 35, 40, 44, 48, 53, 57, 61, 66, 70, 74, 79, 83, 87, 91, 95, 100, 104, 108, 112, 116, 120, 124,
    127, 131, 135, 139, 143, 146, 150, 154, 157, 161, 164, 167, 171, 174, 177, 181, 184, 187, 190, 193, 196,
    198, 201, 204, 207, 209, 212, 214, 217, 219, 221, 223, 226, 228, 230, 232, 233, 235, 237, 238, 240, 242,
    243, 244, 246, 247, 248, 249, 250, 251, 252, 252, 253, 254, 254, 255, 255, 255, 255, 255, 256, 255, 255,
    255, 255, 255, 254, 254, 253, 252, 252, 251, 250, 249, 248, 247, 246, 244, 243, 242, 240, 238, 237, 235,
    233, 232, 230, 228, 226, 223, 221, 219, 217, 214, 212, 209, 207, 204, 201, 198, 196, 193, 190, 187, 184,
    181, 177, 174, 171, 167, 164, 161, 157, 154, 150, 146, 143, 139, 135, 131, 128, 124, 120, 116, 112, 108,
    104, 100, 95, 91, 87, 83, 79, 74, 70, 66, 61, 57, 53, 48, 44, 40, 35, 31, 26, 22, 17, 13, 8, 4, 0, -4,
    -8, -13, -17, -22, -26, -31, -35, -40, -44, -48, -53, -57, -61, -66, -70, -74, -79, -83, -87, -91, -95,
    -100, -104, -108, -112, -116, -120, -124, -128, -131, -135, -139, -143, -146, -150, -154, -157, -161, -164,
    -167, -171, -174, -177, -181, -184, -187, -190, -193, -196, -198, -201, -204, -207, -209, -212, -214, -217,
    -219, -221, -223, -226, -228, -230, -232, -233, -235, -237, -238, -240, -242, -243, -244, -246, -247, -248,
    -249, -250, -251, -252, -252, -253, -254, -254, -255, -255, -255, -255, -255
]
```

### 12.2 Fast Square Root Function

```pseudocode
// Optimized integer square root using lookup table
// Uses binary search through magnitude ranges for O(1) performance
function fastSqrt(x):
    if x >= 65536:
        if x >= 16777216:
            if x >= 268435456:
                if x >= 1073741824:
                    return TABLE[x / 16777216] * 256
                else:
                    return TABLE[x / 4194304] * 128
            else:
                if x >= 67108864:
                    return TABLE[x / 1048576] * 64
                else:
                    return TABLE[x / 262144] * 32
        else:
            if x >= 1048576:
                if x >= 4194304:
                    return TABLE[x / 65536] * 16
                else:
                    return TABLE[x / 16384] * 8
            else:
                if x >= 262144:
                    return TABLE[x / 4096] * 4
                else:
                    return TABLE[x / 1024] * 2
    else:
        if x >= 256:
            if x >= 4096:
                if x >= 16384:
                    return TABLE[x / 256]
                else:
                    return TABLE[x / 64] / 2
            else:
                if x >= 1024:
                    return TABLE[x / 16] / 4
                else:
                    return TABLE[x / 4] / 8
        else:
            if x >= 0:
                return TABLE[x] / 16

    return -1  // Error case
```

### 12.3 Angle Calculation from Components

```pseudocode
// Calculate direction angle (0-359) from X and Y deltas
function getAngleFromComponents(x, y):
    return validateDirection360(getAngleFromComponentsMaths(x, y))

// Internal angle calculation using COMPONENT lookup table
function getAngleFromComponentsMaths(x, y):
    if abs(x) <= abs(y):
        // Y component is dominant
        if y == 0:
            y = 1  // Prevent division by zero

        x = x * 256
        temp = x / y

        if temp < 0:
            temp = -temp
        if temp > 255:
            temp = 255

        if y < 0:  // Moving up (negative Y)
            if x > 0:
                return COMPONENT[temp]          // Quadrant: 0-45 degrees
            else:
                return 360 - COMPONENT[temp]    // Quadrant: 315-360 degrees
        else:  // Moving down (positive Y)
            if x > 0:
                return 180 - COMPONENT[temp]    // Quadrant: 135-180 degrees
            else:
                return 180 + COMPONENT[temp]    // Quadrant: 180-225 degrees
    else:
        // X component is dominant
        if x == 0:
            x = 1  // Prevent division by zero

        y = y * 256
        temp = y / x

        if temp < 0:
            temp = -temp
        if temp > 255:
            temp = 255

        if y < 0:  // Moving up (negative Y)
            if x > 0:
                return 90 - COMPONENT[temp]     // Quadrant: 45-90 degrees
            else:
                return 270 + COMPONENT[temp]    // Quadrant: 270-315 degrees
        else:  // Moving down (positive Y)
            if x > 0:
                return 90 + COMPONENT[temp]     // Quadrant: 90-135 degrees
            else:
                return 270 - COMPONENT[temp]    // Quadrant: 225-270 degrees
```

### 12.4 Velocity Lookup Functions

```pseudocode
// Get X velocity component for a given direction (0-359 degrees)
function getBaseVelX(direction):
    return BASE_VEL_X[validateDirection360(direction)]

// Get Y velocity component for a given direction (0-359 degrees)
function getBaseVelY(direction):
    return BASE_VEL_Y[validateDirection360(direction)]
```

### 12.5 Checksum Seed Iterator (XOR-Shift PRNG)

```pseudocode
// XOR-shift pseudo-random number generator for deterministic checksums
// Both client and server use the same algorithm to stay in sync
function iterateSeed(seed):
    if seed == 0:
        seed = -1  // Handle zero case

    seed2 = bitLeft(seed, 13)   // seed << 13
    seed = seed XOR seed2

    seed2 = bitRight(seed, 17)  // seed >> 17
    seed = seed XOR seed2

    seed2 = bitLeft(seed, 5)    // seed << 5
    seed = seed XOR seed2

    return seed

function bitLeft(n, s):
    return n << s

function bitRight(n, s):
    return n >> s
```

### 12.6 Snowball Flight Path Calculation

```pseudocode
// Calculate flight parameters for a thrown snowball
// Returns: [direction, timeToLive, parabolaOffset]
function calculateFlightPath(userX, userY, targetX, targetY, trajectory):
    output = [0, 0, 0]

    // Convert tile positions to world coordinates
    startWorldX = tileToWorld(userX)
    startWorldY = tileToWorld(userY)
    targetWorldX = tileToWorld(targetX)
    targetWorldY = tileToWorld(targetY)

    // Calculate direction vector (scaled down by 200 for precision)
    deltaX = (targetWorldX - startWorldX) / 200
    deltaY = (targetWorldY - startWorldY) / 200

    // Get direction angle from components
    output[0] = getAngleFromComponents(deltaX, deltaY)

    // Calculate time-to-live based on trajectory type
    if trajectory == 1:
        // Quick/flat throw - fixed TTL
        output[1] = 13
    else:
        // Calculate distance to target
        distanceSquared = (deltaX * deltaX) + (deltaY * deltaY)
        distanceToTarget = fastSqrt(distanceSquared) * 200

        // TTL proportional to distance
        output[1] = distanceToTarget / 2000

    // Parabola offset is half of TTL (peak at midpoint)
    output[2] = output[1] / 2

    return output
```

### 12.7 Usage in Snowball Movement

```pseudocode
// How velocity tables are used in snowball frame movement
function calculateSnowballFrameMovement():
    timeToLive--

    // Get velocity components from lookup tables
    // BASE_VELOCITY_MULTIPLIER = 2000, VELOCITY_DIVISOR = 255
    deltaX = (getBaseVelX(direction) * 2000) / 255
    deltaY = (getBaseVelY(direction) * 2000) / 255

    // Update position
    locH = locH + deltaX
    locV = locV + deltaY

    // Calculate parabolic height
    distanceFromPeak = timeToLive - parabolaOffset

    switch trajectory:
        case 0:  // Quick throw (flat arc)
            if timeToLive > 3:
                distanceFromPeak = 3 - parabolaOffset
            heightMultiplier = 4
            baseHeight = 4000
        case 1:  // Medium lob
            heightMultiplier = 10
            baseHeight = 3000
        case 2:  // High arc
            heightMultiplier = 100
            baseHeight = 3000

    // Parabola formula: height = base + multiplier * (peak² - current²)
    height = baseHeight + heightMultiplier * (parabolaOffset² - distanceFromPeak²)
```

---

## 13. Checksum Synchronization (Detailed)

### 12.1 Purpose

The client and server independently simulate the game. Checksums verify they're in sync.

### 12.2 Checksum Calculator

```pseudocode
class SnowWarChecksumCalculator:

    function calculate(turn):
        // Start with seeded value based on turn number
        checksum = iterateSeed(turn)

        // Add contribution from each game object
        for each machine in game.machines:
            checksum += machine.getChecksumContribution()

        for each player in game.activePlayers:
            checksum += player.avatar.getChecksumContribution()

        for each ball in snowballs:
            checksum += ball.getChecksumContribution()

        return checksum


    // XOR-shift pseudo-random number generator
    function iterateSeed(seed):
        if seed == 0:
            seed = -1

        seed = seed XOR (seed << 13)
        seed = seed XOR (seed >> 17)
        seed = seed XOR (seed << 5)

        return seed
```

### 12.3 Why Deferred Event Processing?

The system processes collisions in two phases:

1. **Detection phase** (during subturns): Detect hits, record as `DelayedEvent`
2. **Application phase** (after checksum): Apply damage, state changes, award points

```pseudocode
// WHY: Checksum must be calculated based on state BEFORE hits are applied
// Otherwise client and server checksums won't match

// WRONG ORDER:
for each subturn:
    detectCollision()
    applyDamage()        // State changes here
calculateChecksum()      // Checksum sees post-damage state

// CORRECT ORDER:
for each subturn:
    detectCollision()
    recordDelayedEvent() // Just record, don't apply
calculateChecksum()      // Checksum sees pre-damage state
applyDelayedEvents()     // Now apply damage
```

---

## 13. Packet Handling

### 13.1 Incoming Packets

| Header | Name | Purpose |
|--------|------|---------|
| | `INITIATECREATEGAME` | Create new game instance |
| | `INITIATEJOINGAME` | Join existing game |
| | `STARTGAME` | Creator starts the match |
| | `GAMEEVENT` | In-game actions (walk, throw, etc.) |
| | `REQUESTFULLGAMESTATUS` | Client requests full sync |
| | `LEAVEGAME` | Player leaves game |
| | `GAMERESTART` | Player clicks "play again" |

### 13.2 GAMEEVENT Routing

```pseudocode
class GAMEEVENT_Handler:

    function handle(player, reader):
        eventType = reader.readInt()

        gamePlayer = player.roomUser.gamePlayer
        game = GameManager.getGameById(gamePlayer.gameId)

        if game.gameType == SNOWWAR:
            SnowWarMessageHandler.handleMessage(eventType, reader, game, gamePlayer)
```

### 13.3 SnowWar Event Types

```pseudocode
enum SnowWarEvent:
    WALK = 0
    THROW_AT_PERSON = 1
    THROW_AT_LOCATION = 2
    CREATE_SNOWBALL = 3
```

### 13.4 Throw Location Handler

```pseudocode
class SnowWarThrowLocationHandler:

    function handle(reader, game, player):
        attr = SnowWarPlayers.get(player)

        // Validate state
        if not attr.isWalkable():
            return
        if attr.lastThrowTime + THROW_COOLDOWN_MS > currentTimeMillis():
            return
        if attr.snowballCount <= 0:
            return

        // Read target
        worldX = reader.readInt()
        worldY = reader.readInt()
        trajectory = reader.readInt()  // 1 or 2

        if trajectory != 1 and trajectory != 2:
            return

        // Stop walking to throw
        if attr.isWalking:
            SnowWarAvatarObject.getAvatar(player).stopWalking()

        // Create snowball
        objectId = game.generateObjectId()
        targetX = convertToGameCoordinate(worldX)
        targetY = convertToGameCoordinate(worldY)

        snowball = new SnowWarSnowballObject(
            objectId,
            game,
            player,
            attr.currentPosition.x,
            attr.currentPosition.y,
            targetX,
            targetY,
            trajectory
        )

        game.updateTask.addSnowball(snowball)
        attr.snowballCount--
        attr.lastThrowTime = currentTimeMillis()

        // Update facing direction
        currentWorldX = tileToWorld(attr.currentPosition.x)
        currentWorldY = tileToWorld(attr.currentPosition.y)
        direction = getAngleFromComponents(worldX - currentWorldX, worldY - currentWorldY)
        attr.rotation = direction360To8(direction)

        // Queue events
        convertedTargetX = convertToWorldCoordinate(targetX)
        convertedTargetY = convertToWorldCoordinate(targetY)

        game.updateTask.queueEvent(new ThrowEvent(player.objectId, convertedTargetX, convertedTargetY, trajectory))
        game.updateTask.queueEvent(new LaunchSnowballEvent(objectId, player.objectId, convertedTargetX, convertedTargetY, trajectory))
```

### 13.5 Outgoing Packets

| Header | Name | Content |
|--------|------|---------|
| 243 | `FULLGAMESTATUS` | Complete game state (objects, turns, checksum) |
| 244 | `GAMESTATUS` | Incremental update (turns, checksum) |
| | `GAMEINSTANCE` | Game info for lobby list |
| | `GAMESTART` | Game timer started |
| | `GAMEEND` | Final scores |
| | `GAMERESET` | Game restarting |

### 13.6 GAMESTATUS Packet Format

```pseudocode
class GAMESTATUS_Packet:

    function compose(response):
        response.writeInt(currentTurn)
        response.writeInt(currentChecksum)
        response.writeInt(turnEventsList.length)  // Always 5

        for each subturnEvents in turnEventsList:
            response.writeInt(subturnEvents.length)

            for each event in subturnEvents:
                event.serialize(response)
```

### 13.7 Event Serialization

```pseudocode
class AvatarMoveEvent:
    objectId = 0
    targetWorldX = 0
    targetWorldY = 0

    function serialize(response):
        response.writeInt(EVENT_TYPE_AVATAR_MOVE)  // 2
        response.writeInt(objectId)
        response.writeInt(targetWorldX)
        response.writeInt(targetWorldY)


class LaunchSnowballEvent:
    objectId = 0
    throwerId = 0
    targetX = 0
    targetY = 0
    trajectory = 0

    function serialize(response):
        response.writeInt(EVENT_TYPE_THROW)  // 4
        response.writeInt(objectId)
        response.writeInt(throwerId)
        response.writeInt(targetX)
        response.writeInt(targetY)
        response.writeInt(trajectory)


class HitEvent:
    throwerId = 0
    targetId = 0
    hitDirection = 0

    function serialize(response):
        response.writeInt(EVENT_TYPE_HIT)  // 5
        response.writeInt(throwerId)
        response.writeInt(targetId)
        response.writeInt(hitDirection)


class StunEvent:
    targetId = 0
    throwerId = 0
    knockbackDirection = 0

    function serialize(response):
        response.writeInt(EVENT_TYPE_STUN)  // 9
        response.writeInt(targetId)
        response.writeInt(throwerId)
        response.writeInt(knockbackDirection)
```

---

## 14. Scoring and Game End

### 14.1 Score Calculation

Scores accumulate during gameplay:

```pseudocode
// On hit
SnowWarPlayers.get(thrower).score += HIT_SCORE  // +1

// On stun (knock out)
SnowWarPlayers.get(thrower).score += STUN_SCORE  // +5
```

### 14.2 Score Calculator

```pseudocode
class SnowWarScoreCalculator:

    function calculatePlayerScore(player):
        return player.score  // Already accumulated during play

    function calculateTeamScore(team):
        total = 0
        for each player in team.players:
            total += player.score
        return total
```

### 14.3 Game Finish Flow

```pseudocode
class SnowWarGame:

    function finishGame():
        // Transfer SnowWar attributes to GamePlayer
        for each player in getActivePlayers():
            attr = SnowWarPlayers.get(player)
            player.score = attr.score
            SnowWarPlayers.remove(player)

        // Calculate team scores
        for each team in teams:
            team.calculateScore()

        // Send results
        broadcast(new GAMEEND(gameType, teams))

        // Start restart countdown
        startRestartTimer()
```

---

## 15. Common Pitfalls

### 15.1 Thread Safety

- Use thread-safe collections for lists modified during iteration
- Use synchronized blocks when accessing the snowball list
- Use atomic operations for counters modified from multiple threads

```pseudocode
// Thread-safe snowball list
snowballs = SynchronizedList()

function addSnowball(ball):
    synchronized(snowballs):
        snowballs.add(ball)
```

### 15.2 Checksum Timing

**Critical**: Calculate checksums BEFORE applying state changes:

```pseudocode
// WRONG
machine.addSnowball()
checksum = calculate()

// CORRECT
checksum = calculate()
machine.addSnowball()  // Apply after checksum
```

### 15.3 Coordinate Conversion

Always convert coordinates consistently:

```pseudocode
// Client sends world coordinates
worldX = reader.readInt()
tileX = convertToGameCoordinate(worldX)

// Server sends world coordinates to client
response.writeInt(convertToWorldCoordinate(tileX))
```

### 15.4 Activity State Machine

Ensure proper state transitions:

```
NORMAL ──[create snowball]──► CREATING ──[timer=0]──► NORMAL (snowball++)
NORMAL ──[hit when HP=0]──► STUNNED ──[timer=0]──► INVINCIBLE ──[timer=0]──► NORMAL
```

---

## 16. Adaptation Notes

### 16.1 For Different Packet Encodings

If your server uses different serialization (e.g., Shockwave vs Flash):

```pseudocode
function serialize(response):
    // Change writeInt/writeString to your encoding
```

### 16.2 For Different Threading Models

If your server doesn't use a scheduled executor:

```pseudocode
function startGameTask():
    // Use your server's task scheduling mechanism
```

### 16.3 Free-for-All Mode

SnowWar supports 1-team mode (everyone vs everyone):

```pseudocode
function isOpponent(p1, p2):
    if p1.userId == p2.userId:
        return false
    if teamAmount == 1:
        return true  // FFA mode
    return p1.teamId != p2.teamId
```

---

## 17. Testing Checklist

- [ ] Lobby shows game list correctly
- [ ] Game creation with different map/team/length options works
- [ ] Players can join teams and switch teams
- [ ] Game starts when creator presses start (with minimum players)
- [ ] Players spawn at correct positions
- [ ] Movement works with smooth interpolation
- [ ] Snowball throwing works with all 3 trajectories
- [ ] Collision detection hits targets correctly
- [ ] Damage reduces health, stun occurs at 0 HP
- [ ] Stun -> Invincibility -> Normal state flow works
- [ ] Snowball machines generate and dispense snowballs
- [ ] Scores accumulate correctly
- [ ] Game ends when timer expires
- [ ] Results screen shows correct scores
- [ ] Restart flow works for players who click "play again"
- [ ] Spectator mode works
- [ ] Client checksums match server checksums (no desync)

---

## 18. File Structure Summary

```
snowwar/
├── SnowWarGame                     # Main game class
├── SnowWarPlayers                  # Attribute registry
├── SnowWarScoreCalculator          # Scoring logic
├── SnowWarMapsManager              # Map loading
├── SnowWarDelayedEvent             # Deferred event wrapper
├── SnowWarDelayedEventType         # Event type enum
│
├── tasks/
│   ├── SnowWarGameTask             # Main game loop
│   └── SnowWarChecksumCalculator
│
├── objects/
│   ├── SnowWarGameObject           # Base class
│   ├── SnowWarAvatarObject         # Player representation
│   ├── SnowWarSnowballObject       # Projectile
│   └── SnowWarMachineObject        # Dispenser
│
├── events/
│   ├── SnowWarAvatarMoveEvent
│   ├── SnowWarThrowEvent
│   ├── SnowWarLaunchSnowballEvent
│   ├── SnowWarHitEvent
│   ├── SnowWarStunEvent
│   ├── SnowWarCreateSnowballEvent
│   ├── SnowWarMachineAddSnowballEvent
│   ├── SnowWarMachineMoveSnowballsEvent
│   └── SnowWarDeleteObjectEvent
│
├── mapping/
│   ├── SnowWarMap                  # Map data structure
│   ├── SnowWarTile                 # Tile definition
│   ├── SnowWarItem                 # Map item (obstacles)
│   └── SnowWarPathfinder           # Movement validation
│
├── messages/
│   ├── SnowWarMessageHandler       # Event dispatcher
│   └── incoming/
│       ├── SnowWarWalkMessage
│       ├── SnowWarThrowLocationMessage
│       ├── SnowWarAttackPlayerMessage
│       └── SnowWarCreateSnowballMessage
│
└── util/
    ├── SnowWarConstants            # Game constants
    ├── SnowWarAttributes           # Player state
    ├── SnowWarActivityState        # State enum
    ├── SnowWarEvent                # Input event enum
    ├── SnowWarMessage              # Handler interface
    ├── SnowWarMath                 # Math utilities
    ├── SnowWarTrajectory           # Flight path calc
    └── SnowWarItemProperties       # Item collision data
```

---

This guide provides the architectural foundation for implementing SnowWar. The key principles are:

1. **Tick-based simulation** with sub-turns for smooth movement
2. **Checksum synchronization** for client/server validation
3. **Deferred event processing** to maintain sync integrity
4. **Clear separation** between game logic, networking, and state management

Start with the core game loop and basic movement, then add projectiles and collision detection, and finally implement scoring and game flow.
