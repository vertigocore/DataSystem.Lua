## 🚀 Features

### Core Functionality
- **Persistent Data Storage**: Secure player data persistence using Roblox DataStores
- **Automatic Backups**: Scheduled backups every 5 minutes with indexed backup history
- **Real-time UI Synchronization**: Live updates for coins, level, playtime, and blocks
- **XP/Level System**: Exponential progression system with server-side XP calculation (max level 500)

### Security
- **HMAC Authentication**: Hash-based message authentication for client-server communication
- **Server-Authoritative**: All critical data (coins, XP) managed server-side
- **Rate Limiting**: RPC request throttling to prevent exploitation
- **Encrypted Updates**: Secure data transmission to prevent client-side tampering

### Building System
- **Block Placement**: Place blocks using R key
- **Block Manipulation**: Select, move (arrow keys), rotate (W/A/S), and delete (T key)
- **Persistent Builds**: Placed blocks saved across sessions
- **Resource Management**: Each player starts with 10 blocks; deleted blocks return to inventory
- **Build Mode**: Character movement disabled while building

### Developer Tools
- **Debug Logging**: Centralized logging system with toggle support
- **Chat Commands**: `/coins`, `/xp`, `/wipe` for testing and administration
- **Error Handling**: Robust error handling with retry mechanisms for DataStore operations

## 📁 Architecture

### Server-Side Modules
- **ServerSecrets**: Centralized configuration and constants
- **DataStoreManager**: Handles all DataStore operations with retry logic
- **PlayerManager**: Manages player data lifecycle and leaderstats
- **SecurityModule**: HMAC generation and secure data encryption
- **XPLevelSystem**: Level calculation and XP progression logic
- **BuildSystem**: Server-side building system management
- **RPCGateway**: Client-server communication hub with rate limiting
- **DebugLogger**: Centralized logging system

### Client-Side Scripts
- **ClientGlue**: UI management and data synchronization
- **BuildClient**: Building system input handling and character control


1. **Studio Structure**:
   ```
   ReplicatedStorage/
   └── Remotes/
       └── RPC (RemoteEvent)

   ServerStorage/
   └── ServerSecrets (ModuleScript)

   ServerScriptService/
   ├── ServerModules/
   │   ├── DebugLogger (ModuleScript)
   │   ├── DataStoreManager (ModuleScript)
   │   ├── PlayerManager (ModuleScript)
   │   ├── RPCGateway (ModuleScript)
   │   ├── ServerConsole (ModuleScript)
   │   ├── SecurityModule (ModuleScript)
   │   ├── XPLevelSystem (ModuleScript)
   │   └── BuildSystem (ModuleScript)
   └── Server (Script)

   StarterPlayer/
   └── StarterPlayerScripts/
       ├── ClientGlue (LocalScript)
       └── BuildClient (LocalScript)

   StarterGui/
   └── Info (ScreenGui)
       └── BackFrame (Frame)
           ├── Profile (Frame)
           ├── Coins_Display (TextLabel)
           ├── Time_Display (TextLabel)
           ├── XP_Display (TextLabel)
           ├── Blocks_Display (TextLabel)
           └── username (TextLabel)
   ```


## 🎮 Usage

### Player Controls
- **P Key**: Toggle UI visibility
- **R Key**: Place a block (build mode)
- **Left Click**: Select a block
- **Arrow Keys**: Move selected block
- **W/A/S Keys**: Rotate selected block (X/Y/Z axis)
- **T Key**: Delete selected block
- **ESC/X Key**: Exit build mode

### Chat Commands
- `/coins <amount>` - Add or remove coins (e.g., `/coins 500` or `/coins -200`)
- `/xp <amount>` - Add XP (e.g., `/xp 500`)
- `/wipe` - Reset all player data to defaults (server-wide)

## 🔧 Configuration

Key settings in `ServerSecrets`:

```lua
-- DataStore Configuration
GAME_DATASTORE_NAME = "PlayerData"
BACKUP_DATASTORE_NAME = "PlayerBackups"

-- Default Player Data
DEFAULT_PLAYER_DATA = {
    coins = 300,
    xp = 0,
    playTimeSeconds = 0,
    availableBlocks = 10,
    placedBlocks = {},
}

-- XP/Level System
MAX_LEVEL = 500
XP_BASE_MULTIPLIER = 100
XP_EXPONENT = 1.5

-- Security
SERVER_HMAC_SECRET = "replace-with-a-random-long-secret"
RPC_RATE_LIMIT_PER_SECOND = 8
```

## 🔐 Security Features

- **Server-Authoritative Design**: All critical values stored and validated server-side
- **HMAC Verification**: Prevents data tampering in client-server communication
- **Rate Limiting**: Prevents spam and exploitation attempts
- **Input Validation**: Type checking and bounds validation for all client inputs
- **Encrypted Updates**: Client receives encrypted data, not raw values

## 📊 Data Management

- **Immediate Saves**: Debounced saves on data changes (300ms debounce)
- **Periodic Backups**: Automatic backups every 5 minutes
- **Backup Indexing**: Maintains up to 50 backup snapshots per player
- **Error Recovery**: Automatic retry with exponential backoff on DataStore failures

## 🐛 Debugging

Enable/disable debug logging via `ServerSecrets.DEBUG_ENABLED`:

```lua
ServerSecrets.DEBUG_ENABLED = true  -- Enable logging
ServerSecrets.DEBUG_ENABLED = false -- Disable logging
```

All logs are prefixed with `[DEBUG]`, `[WARN]`, or `[ERROR]` for easy filtering.

## 📝 Notes

- **Playtime Tracking**: Persists across sessions and continues when players rejoin
- **XP Calculation**: Uses exponential formula: `XP = BASE * (LEVEL ^ EXPONENT)`
- **Block Persistence**: Placed blocks are saved in player data and restored on join
- **Character Control**: Movement automatically disabled in build mode, re-enabled on exit

## 🔄 Data Flow

1. **Player Joins** → Data loaded asynchronously from DataStore
2. **UI Initialization** → Secure data sent to client via RPC
3. **Live Updates** → Periodic sync (coins, level, playtime) every few seconds
4. **Data Changes** → Debounced immediate save + scheduled backup
5. **Player Leaves** → Final save and cleanup

---

**Developed by loif ( sqpxcy )**

