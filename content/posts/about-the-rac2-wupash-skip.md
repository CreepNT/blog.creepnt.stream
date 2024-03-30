+++
author = "CreepNT"
title = "A look at the R&C 2 Wupash skip..."
date = "2022-09-20"
description = "Racers-abused race condition"
+++

# Introduction
In the original Ratchet & Clank games for PlayStation™ 2 (and their HD remasters for PS3™/PS Vita™), the player explores multiple levels and can travel between them. There are a few programatical ways the player is moved from one level to another (for example, at the end of a cutscene), but most of the time, it is up to the player to do it by entering Ratchet's ship, browsing a list of all the unlocked coordinates, selecting a planet and taking off.

On September 19, 2022, community member VollZX found a way to skip a level in Ratchet & Clank 2 with a simple but well-timed keypress. In Ratchet & Clank 2, the third level the player is supposed to visit is the Maktar Nebula, right after Planet Oozla. However, when taking off from Oozla while selecting Maktar, the player is instead taken to Wupash Nebula, a space level. This level isn't exactly popular among speedrunners, for multiple reasons: completing the level optimally in NG+ can be tricky, and it's quite a long fight in Any%; all of this in addition to being a space level.

The bug was explained to me as the following: by pressing Down and Triange at "the right time", [Voll's ship took off towards Maktar from Oozla instead of going towards the Nebula](https://clips.twitch.tv/HyperDistinctLouseArgieB8-ydhBWkGAEdpGcmmS). On first glance, I proposed a (wrong) hypothesis as to what may be happening, but decided to look into it more closely out of curiosity. This blogpost will walk you through how I found the root cause of this quite exotic bug.

### Setup
For reverse engineering, my tool of choice is the open-source [Ghidra](https://ghidra-sre.org/) suite. It's a pretty powerful alternative to IDA, and while it's not as good on certain points, it also has some pretty major advantages (e.g. not costing an arm and a leg, having a working [Vita ELF loader](https://github.com/CreepNT/VitaLoaderRedux), etc).

For live debugging, I used to use [RCHooker](https://github.com/CreepNT/RCHooker) with well-placed hooks and `printf`s everwhere, but it's a quite bad setup that makes work extremly painful. It just so happens that I recently came into possession of a Vita devkit and figured out how to allow the debugger to attach to *any* process (even `SceShell` 😊), so I used that instead. Besides a few quirks, it worked flawlessly and was a much better experience than using RCHooker.

# The Bug
## First clues
From previous reversing, I had found a function (`go_to_level`) in R&C3 that looked like a good culprit. I managed to find its counterpart in R&C2 - thanks to some debugging strings - and confirmed it was indeed involved by placing a breakpoint on it and performing a regular takeoff - it indeed got called. I walked up the stack to the caller, `FUN_81A3B046()`, which accepts a single argument (`new_level_id`) and does the following:

 * if `new_level_id == CURRENT_LEVEL_ID`, some uninteresting code path is invoked
   * Note that `CURRENT_LEVEL_ID` here is a *global variable* that stores... the ID of the current level (duh!)
 * otherwise, it calls `go_to_level`

I decided to move my breakpoint to this function - adding a `new_level_id != LEVEL_OOZLA` condition to reduce unnecessary breaking, since all takeoffs would be performed from Oozla - and tried out the trick. After a few attempts, the debugger pauses in the function, invoked with `new_level_id` equal to `LEVEL_MAKTAR_ARENA`. Resuming execution flies me to Maktar Arena, as expected.

## Going up the stack
Thanks to the debugger again (🙏), I had full call stacks (**much better** than RCHooker, where you can only walk up one function) so I could easily go up the call chain. The caller was also the only one Ghidra found, which is cool because it meant the call must be a direct (Ghidra usually fails to establish caller-callee relationships when indirect calls are perfomed). I had named this function `mode_doChange` - a name borrowed from the symboled Deadlocked build that was (re)discovered a few months back - which gave me a hint about this bug's involvment with the game mode system. 

### Game modes
The game mode system is made of several global variables that affect the game's execution flow, and functions to modify this global state:

 * The `gameMode` global variable contains the current game mode
 * The `pendingGameMode`, `rmc_arg0` and `rmc_arg1` global variables store information necessary for game mode change
 * `mode_requestChange(new_mode, a0, a1)` can be called to change the current game mode
  * It stores `new_mode` in `pendingGameMode`, `a0` in `rmc_arg0` and `a1` in `rmc_arg1`
  * *It may also save the current game mode in `modeStack`, to allow restoring it later on*
 * `mode_doChange()` is called in the main game loop (I guess?) and performs the transition
  * First, it backs up `gameMode` in the `lastGameMode` global variable (**last** stands for **previous** here)
  * Then, it finds a function to execute based on the values in `gameMode`, `pendingGameMode`
  * In some cases, `rmc_arg0` is also examined to find the function

In our case, the following request caused `go_to_level` to be invoked:

 * `gameMode` is `GAME_MODE_PAUSE` (we're in the ship's menu)
 * `pendingGameMode` is `GAME_MODE_SPACE` (takeoff / get-in-ship / get-out-of-ship animations)
 * `rmc_arg0` is neither 5 nor 6

In this case, `FUN_81A3B046(rmc_arg1)` is invoked...

![Screenshot of the decompiled routine](/rac2_decomp_output.png)

If you hooked `FUN_81A3B046()` and exited the ship "normally", you'd see that `new_level_id` (and thus `rmc_arg1`) was always identical to `CURRENT_LEVEL_ID` (i.e., `LEVEL_OOZLA`); as such, the "not switching level" code path was invoked and all is well. However, when doing the trick, `new_level_id` was **not** `LEVEL_OOZLA` but `LEVEL_MAKTAR_ARENA` instead! To be sure that this is what caused the ship to leave for Maktar, I replaced `new_level_id` with `LEVEL_INSOMNIAC_MUSEUM` and the ship did take off towards the Museum.

I now had a rough overview of the bug: in the usual case, when leaving the ship, `rmc_arg1` is equal to `CURRENT_LEVEL_ID`; by doing whatever we were doing, we caused something that made it no longer be equal, and thus affected the game's behaviour. Thus, the questions that remained was *why is `rmc_arg1` different when we do wacky stuff?!*

## Why is it `rmc_arg1` wrong?
The easiest solution to figure this out is to place a data change breakpoint on `rmc_arg1`. The variable is actually accessed a lot, so I added a `rmc_arg1 == LEVEL_MAKTAR_ARENA` condition to reduce false positives, then I perforemd the bug. The debugger stopped in `mode_pop`, which is a function used to restore a game mode previously stored on the `modeStack` (it pops the top of the `modeStack` to `pendingGameMode`). It accepts two arguments that are placed in `rmc_arg0` and `rmc_arg1`. Looking at the call stack (`main_game_loop` (`FUN_819CD7A6`) -> `game_mode_pause_handler` (`FUN_81A20E14()`) -> `mode_pop`), we can see that the second arguments - what will be stored in `rmc_arg1` - comes from a global variable. We will call this variable `g_previewedLevelId` for reasons that will become obvious later on.

I placed a breakpoint on that newly found global, and found it was only being modified by one routine, `ship_menu_set_previewed_planet` (`FUN_819CD7A6()`), which takes a level ID as an argument, writes it to `g_previewedLevelId` and reads the size of `rc2/psp2data/global/maps/maps{_mm}[%d].ps3` where `%d` is replaced by `g_previewedLevelId`.
This function is only called from three places:

 1. When the ship menu appears, the call site is `0x81A1EE91`. This is part of a routine called when entering `GAME_MODE_PAUSE` and passes `CURRENT_LEVEL_ID` to `ship_menu_set_previewed_planet`, so it cannot be what causes `rmc_arg1` to be wrong.

 1. When exiting the ship menu, the call site is `0x81A23C05`. This is part of what I assume to be the routine holding the bulk of the menu's logic, and also passes `CURRENT_LEVEL_ID` to `ship_menu_set_previewed_planet`, so it also cannot be what causes `rmc_arg1` to be wrong.

 1. When pressing Up/Down (i.e. changing the selected planet in the menu), the call site is `0x81A23DDB`, a bit further in the menu logic's routine. **The value passed to `ship_menu_set_previewed_planet` is NOT static, but obtained through pointer dereferences**. I didn't bother reversing the whole thing because all that mattered is that **this was the only caller with an argument that could be different from `CURRENT_LEVEL_ID`**, so it **had** to be the one responsible for the unexpected behaviour.

## "Are we in space, or simply paused?"
Attempting to understand what was going on, I added a breakpoint that would print something whenver hit at each of the call sites, to see in which order they were called:

 * When entering the ship menu, #1 was called
 * #3 was called each time I changed the selected planet
 * Exiting the ship menu caused #2 to be called

I then performed the trick, and something weird occured: **the D-Pad Up/Down press handler (#3) was executed after the 'exiting menu' handler (#2) has been called!** Even weirder is that looking up the call chain, #3 is called by `game_mode_pause_handler` which, as can be guessed from its name, only runs when `gameMode == GAME_MODE_PAUSE`! By now I had a rough overview of the problem but this seemingly mismatching gamemode puzzled me a lot. I used logging breakpoints again to see when the game mode was changed and also added one to know when my Triangle press was detected (i.e., when *exiting menu* handler was called). Both of these loggers also print the "rendered frames counter" as a timestamp.

The resulting log was... unexpected:
```
TRIANGLE PRESSED @ 419425
SWITCH TO GM 6   @ 419428
```

*🤔 The game mode isn't changed directly?*

Apparently, no, it isn't... I have not fully reversed everything involved though, so I don't understand exactly why. However, this has very important implications. It means that *the `GAME_MODE_PAUSE` handler still runs after requesting to close the menu*, or in other words, that *we can change `g_previewedLevelId` after it is reset but before it is passed to the routine using it to decide whether or not to initiate a takeoff*. This type of bug is called a *race condition* because successful exploitation relies on winning a race (in our case, pressing a button at the right moment).

# The Aftermath
For speedrunners, this bug is obviously a blessing, considering it's a quite important timesave that can be achieved relatively easily. Wupash Nebula is also a harsh level to master, and it isn't super entertaining to watch or perform, so I'm confident not many people will regret it no longer being visited.

As for the R&C codebase: as I noted in the introduction, Insomniac had similar "planet redirection" schemes in R&C3, so I thought that it might still work. I attempted to replicate the bug but nothing came out of it. This bug seemingly no longer exists; whether it was found and fixed, or is simply gone as a byproduct of architectural changes is unknown though. In R&C3, the planet redirection schemes are only used for two levels:

 * `Starship Phoenix`
   * After completing Qwark's Hideout, Ratchet is called by Sasha because the Starship Phoenix is under attack. This `Starship Phoenix Rescue` level, with ID `6`, is separate from the regular `Starship Phoenix`, which has ID `3`. Since Ratchet can leave the "Save the Phoenix" mission, the game has to be able to redirect going to level `3` to level `6` under certain circumstances.

 * `Holostar Studios`
   * During the first visit to Holostar Studios, you play as Clank in the `Holostar Studio 43` level, with ID `23`, a separate level from the "normal" `Holostar Studios` level, which has ID `13`. Because the player initiates flying towards the level, however, there again needs to be some redirection scheme.

Both of these are handled inside `go_to_level` by checking the requested planet ID and changing it if needed. Compared to R&C2, this is a much saner scheme: since this function always ends up called on a level change, this ensures that no matter where the request comes from, nothing can bypass the redirection.

The following pseudocode exposes how it works:
```c
//FUN_81102774, go_to_level is just a wrapper that calls this
void go_to_level_internal(int level_id, bool a2) {
    /* ... */
    if (level_id == LEVEL_HOLOSTAR_STUDIOS && !BYTE_81A10BFF) {
        level_id = LEVEL_HOLOSTAR_STUDIO_CLANK;
    } else if (level_id == LEVEL_STARSHIP_PHOENIX && (UINT_81A0F812 & 0x1)) {
        level_id = LEVEL_STARSHIP_PHOENIX_RESCUE;
    }
    /* ... */
    /* trigger level change towards level_id */
    /* ... */
}
```

I wanted to figure out how the Maktar->Wupash redirection was done in R&C2 nonetheless, and since it was **not** via code in `go_to_level`, I had to look somewhere else. Fortunately, I accidentally stumbled across the routine that handles the detailed planet view (the screen with the objectives of a planet where pressing X initiates takeoff).

Here's a simplified and commented version of that code:
```c
if (/* Cross is pressed */) {
    if (g_previewedLevelId == 0 /* Aranos 1 - some failsafe? */
        || BYTE_821C6F7D != 0 /* Wupash finished? */
        || g_previewedLevelId != LEVEL_MAKTAR /* not going to Maktar - no redirection needed */
    ) {
        /* ... */
    } else {
        //Overwrite the game mode change request, with rmc_arg1 with WUPASH_NEBULA, to change the destination level
        //BUG: rmc_arg1 is not guaranteed to still be WUPASH_NEBULA by the time it is used for takeoff!
        mode_requestChange(GAME_MODE_SPACE, RMC_SWITCH, 1, LEVEL_WUPASH_NEBULA, NULL);
    }
    /* ... */
}
```