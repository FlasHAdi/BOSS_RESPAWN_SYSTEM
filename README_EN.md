# Boss Respawn System - Boss Respawn System after Maintenance

## Description
Automatic system for saving and respawning important bosses after restart/maintenance.
- Saves boss positions in the database
- Supports multiple channels
- Automatic respawn when the server starts
- GM commands for easy management

## Installation

### 1. Database
Run the SQL file to create the table:

### 2. Compilation
Compile the game using:
```bash
cd game/src
gmake clean
gmake -j4
```

## Configuration

### Adding Bosses to the Tracking List

#### Method 1: Using the GM Command (Recommended)
```
/boss_track_add <vnum>
```
Example for common bosses:
- `/boss_track_add 2493` - Beran-Setaou
- `/boss_track_add 6091` - Razador
- `/boss_track_add 2597` - Charon
- `/boss_track_add 2191` - Giant Turtle
- `/boss_track_add 2492` - General Yonghan

#### Method 2: (Automatic at boot)
Modify in `game/src/input_db.cpp` after LoadBossRespawnData():
```cpp
// Automatically add important bosses to tracking
CMobManager::instance().AddTrackedBossVnum(2493); // Beran-Setaou
CMobManager::instance().AddTrackedBossVnum(6091); // Razador
CMobManager::instance().AddTrackedBossVnum(2597); // Charon
CMobManager::instance().AddTrackedBossVnum(2191); // Giant Turtle
CMobManager::instance().AddTrackedBossVnum(2492); // General Yonghan
// Add other bosses here...
```

## GM Commands

### `/boss_track_add <vnum>`
Adds a boss to the tracking list.
- **GM Level:** IMPLEMENTOR
- **Example:** `/boss_track_add 2493`

### `/boss_track_remove <vnum>`
Removes a boss from the tracking list.
- **GM Level:** IMPLEMENTOR
- **Example:** `/boss_track_remove 2493`

### `/boss_track_list`
Displays the list of all tracked bosses.
- **GM Level:** LOW_WIZARD
- **Example:** `/boss_track_list`

### `/boss_respawn_reload`
Reloads the data from the database and respawns the saved bosses.
- **GM Level:** IMPLEMENTOR
- **Usage:** After an error or for testing
- **Example:** `/boss_respawn_reload`

### `/boss_respawn_clear`
Deletes all respawn data from the database for the current channel.
- **GM Level:** IMPLEMENTOR
- **Warning:** This command permanently deletes the data!
- **Example:** `/boss_respawn_clear`

## Functionality

### When Bosses Spawn
When a boss from the tracking list spawns:
1. The system automatically records the position (X, Y)
2. Saves the map_index and channel
3. Writes to the database
4. **Supports multiple instances** - the same boss can be saved in several different locations
### When Bosses Die
When a boss dies:
1. The system removes the entry from the database
2. Stops tracking for that boss until the next spawn

### When the Server Restarts
When the server boots:
1. All respawn data for the current channel is loaded
2. Bosses are automatically respawned at their saved positions
3. Tracking continues for new bosses


**Important Constraint:** `unique_boss_position` allows the same boss (same VNUM) to be saved in **multiple different locations**. Each unique position (x, y) on the same map and channel is treated as a separate instance.

## Examples of Use

### Example 1: Adding a New Boss
```
GM: /boss_track_add 6091
Server: Boss VNUM 6091 (Razador) added to tracking list.

[Boss spawned in game]
Server Log: BOSS_RESPAWN: Registered boss vnum=6091, vid=150001, map=351, pos=(1234,5678), channel=1
Server Log: BOSS_RESPAWN: Saved boss data vnum=6091...
```

### Example 2: After Server Restart
```
Server Log: BOOT: Loading boss respawn system...
Server Log: BOSS_RESPAWN: Loading boss respawn data from database for channel 1...
Server Log: BOSS_RESPAWN: Loaded boss vnum=6091, map=351, pos=(1234,5678), channel=1
Server Log: BOSS_RESPAWN: Loaded 1 boss(es) for channel 1
Server Log: BOSS_RESPAWN: Starting respawn of 1 boss(es) on channel 1...
Server Log: BOSS_RESPAWN: Respawned boss vnum=6091 at map=351, pos=(1234,5678), channel=1 [NEW VID=150002]
Server Log: BOSS_RESPAWN: Successfully respawned 1/1 boss(es) on channel 1
```

### Example 3: Checking Tracked Bosses
```
GM: /boss_track_list
Server: === Tracked Boss VNUMs ===
Server: - VNUM 2191: Beran-Setaou
Server: - VNUM 2493: Dragon Lacau
Server: - VNUM 6091: Razador
Server: - VNUM 8057: Grotto Orc Boss
Server: Total: 4 boss(es)
```
### Example 4: Multiple Instances of the Same Boss
```
# Spawn the same boss in 3 different locations
GM: /mob 6091   # Location 1
GM: /warp 100 200
GM: /mob 6091   # Location 2
GM: /warp 300 400
GM: /mob 6091   # Location 3

# Check in DB
SELECT mob_vnum, x, y FROM boss_respawn_data WHERE mob_vnum=6091;
+----------+--------+--------+
| mob_vnum | x      | y      |
+----------+--------+--------+
|     6091 | 123450 | 678900 |
|     6091 | 234560 | 789010 |
|     6091 | 345670 | 890120 |
+----------+--------+--------+

# After restarting, all 3 instances will be respawned in their locations
```
## Logging

All system actions are logged in syslog with the prefix `BOSS_RESPAWN`:
- Boss logs
- Database saves
- Boot loads
- Respawns

## Multiple Channel Compatibility

The system operates independently on each channel:
- Each channel saves its own bosses
- At boot, only bosses for that channel are loaded
- GM commands only operate on the current channel

## Important Notes

1. **Performance:** The system uses optimized queries with indexes
2. **Safety:** Data is saved with `ON DUPLICATE KEY UPDATE`
3. **Cleanup:** Data is automatically deleted when bosses die

## Debugging

To enable detailed logging, check syslog:
```bash
tail -f /usr/metin2/log/syslog | grep BOSS_RESPAWN
```

## Extension

To add new features:
1. Modify the `TBossRespawnData` structure in `mob_manager.h`
2. Update the save/load functions in `mob_manager.cpp`
3. Add new columns to the SQL table if necessary

## Troubleshooting

### Bosses do not respawn after restart
- Check if the SQL table exists and has data: `SELECT * FROM boss_respawn_data;`
- Check the syslog for errors
- Run `/boss_respawn_reload` as a test

### Bosses respawn multiple times
- Check for duplicates in the database
- Run `/boss_respawn_clear` and re-add the bosses

### Old data in the database
- Run periodically: `DELETE FROM boss_respawn_data WHERE last_update < DATE_SUB(NOW(), INTERVAL 30 DAY);`

## Support

For problems or questions:
- Check the logging in syslog
- Test with `/boss_respawn_reload`
- Make sure all files have been compiled correctly

---
**Version:** 1.0.0
**Last update:** 2026
