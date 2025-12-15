
# Player Data:
- Modifying player data while the server is running can be dangerous if the player joins/leaves while the data is being modified. To minimize the risk of data corruption, I recommend making a copy of the player data file and replacing the original when you're ready.
- It is generally safe to replace individual player data files while the server is running if the player is offline. Replacing the player data of a player who is online will likely have no effect, and is therefore discouraged.
- If you made a copy of a player data file and started modifying it, but the player in question joined the server while doing it, it's best to wait until they disconnect and restart from scratch.
- DO NOT REMOVE OR RENAME THE PLAYERDATA DIRECTORY WHILE THE SERVER IS RUNNING. The game uses a file object to target the directory (or at least it did when writing this), and removing the directory makes it invalid which makes the server unable to save any playerdata until it's restarted! If you wish wipe player data but also make a backup, make a copy with the archive button, then delete the contents of the playerdata directory.


# Mods:
- Avoid replacing mods directly while the server is running, as this can cause the server to crash with strange classloading exceptions.
- Instead, place them in the update directory, they will then be copied and replace the mod with the same modid on the next server restart.


# Copying from Test Server to Survival:
- As long as you never add mods/datapacks directly to survival (i.e. you always put them on the test server before or at the same time), you should always be able to copy directly from the test server.
- There are some edge cases though:
	- Ledger contains a config specifying that it should use an external database on Survival. If the config file is overwritten from the test server, it will begin to use a local one instead. Realising that the server is using the local database is never fun, and requires a painful migration to sort out.
	- Discord-MC-Chat contains data about the channel it's connected to, overwriting it will require manually setting that up again.
	- Care must be taken when copying world data since you don't want to overwrite the survival world. Copying new dimensions should always be fine though.


# Copying large structures/parts of the world:
- There is no good way to copy larger masses of blocks between worlds (RIP MCEdit you will be missed, Amulet has no UI on Linux... Even then neither of those are very good for the panel since uploading/downloading the world can take a long time especially towards the middle/end of the season).
- If the area is far enough from player-populated areas, copying region files directly is usually a very fast and efficient method.
	- Note that this overrides everything in that region (check F3 in-game).
	- You can check the region folder if the region file already exists. 
		- If it doesn't exist, it's safe to copy.
		- If it does, you should double-check in-game to see if anyone has built anything in it.
			- Remember that each region is quite large, about 512x512 blocks horizontally! They go from world bottom all the way to the height limit (although you probably already knew that).
	- Don't forget that there are 3 folders to keep in mind: 
		- region: Contains all blocks.
		- entities: Contains all entities. If you forget this region file, no entities will be copied.
		- poi: Contains point of interest data, i.e. locations of blocks that should be easy for the game to locate. This is very important for villager farms, nether portal linking, lodestone compasses and lightning rods. It's also possible to add more point of interest types with simple-custom-features which may be used in some datapacks, so I recommend always copying this one to be safe, unless you're certain that no point of interest data is relevant.
- Pasting large structures with WorldEdit can be dangerous.
	- If they take too long, the server watchdog will kill the server.
	- If the structure is too big, server might run out of RAM (killing the server).
	- Don't bother trying to use structure blocks, they'll only make it worse.
- How to make pasting with WorldEdit safer.
	- Always save the server before applying larger structures.
		- Wait a day after a lot of chunks have been loaded, this is to give the server time to process a lot of players generating a lot of chunks all at once. Otherwise the reboot time will be so slow that you'll probably think its frozen.
		- First /save-all			(good first step since `/save-all flush` might take too long if too many chunks have been generated recently, crashing the server).
		- Then /save-all flush		(ensures the world has been saved, meaning if the server crashes the rollback is minimal at best).
	- Disable neighbour updates (has to be done after every restart).
		//perf neighbors off
		//perf update off			(for some reason this one doesn't show up via autocomplete on servers)
	- Paste air blocks separately (or skip them entirely). I recommend always pasting air first to avoid blocks like cactus not being placed properly.
		//paste -m air		(optional)
		//paste -a
	- Paste entities/biomes afterward (especillay if there are a lot of entities):
		//paste -em structure_void	(we need to specify a block to paste, so I just specify that it should paste structure_void, which should not paste anything, feel free to replace with a block of your choice)
		//paste -bm structure_void
	- Restart the server after larger pastes to let server recover RAM (or force-run the garbage collector if you can find a button for that somewhere, I can't).
	- If a structure is super-massive, consider splitting it into smaller chunks that you paste separately.
- WorldEdit likes to break redstone contraptions. Consider pasting structures twice if there are any blocks that require supporting blocks. Structure blocks generally don't have this problem, especially if strict placement is used in 1.21.5+.


# Backups:
- When making backups, it's worth double-checking how much storage space remains (the easiest way to do this is via SSH using the `df` command).
- If you notice there is not much space left, pruning old backups is an easy way to recover a lot of storage.
- Distant Horizons eats up a lot of disk space for the LOD cache, so it might be worthwhile trying to exclude it from the backups if possible.

