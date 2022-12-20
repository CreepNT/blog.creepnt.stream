+++
author = "CreepNT"
title = "A look at the R&C 2 Wupash skip..."
date = "2022-09-20"
description = "A case of unforeseen consequences."
+++

# Introduction
In the original Ratchet & Clank games for PlayStation™ 2 (and their HD remasters for PS3™/PS Vita™), the player explores multiple levels and can travel between them. There are a few programatical ways the player is moved from one level to another (for example, at the end of a cutscene), but most of the time, it is up to the player to do it by entering Ratchet's ship, browsing a list of all the unlocked coordinates, selecting a planet and taking off.

On September 19, 2022, community member VollZX found a way to skip a level in Ratchet & Clank 2 with a simple but well-timed keypress. In Ratchet & Clank 2, the third level the player is supposed to visit is the Maktar Nebula, right after Planet Oozla. However, when taking off from Oozla while selecting Maktar, the player is instead taken to Wupash Nebula, a space level. This level isn't exactly popular among speedrunners, for multiple reasons: completing the level optimally in NG+ can be tricky, and it's quite a long fight in Any%; all of this in addition to being a space level.

The bug was explained to me as the following: by pressing Down and Triange at "the right time", [Voll's ship took off towards Maktar from Oozla instead of going towards the Nebula](https://clips.twitch.tv/HyperDistinctLouseArgieB8-ydhBWkGAEdpGcmmS). On first glance, I proposed a (wrong) hypothesis as to what may be happening, but decided to look into it more closely out of curiosity. This blogpost will walk you through how I found the root cause of this quite exotic bug.

### Setup
For reverse engineering, my tool of choice is the open-source [Ghidra](https://ghidra-sre.org/) suite. It's a pretty powerful alternative to IDA, and while it's not as good on certain points, it also has some pretty major advantages (e.g. not costing a fuckton of money, having a working [Vita ELF loader](https://github.com/CreepNT/VitaLoader), ...).

For live debugging, I would usually have used [RCHooker](https://github.com/CreepNT/RCHooker) with well-placed hooks and `printf`s everwhere. However, it's a frankly terrible setup that brings not much more than just pain. It just so happens that I recently came into possession of a Vita devkit and figured out how to allow the debugger to attach to *any* process (even `SceShell` 😊), so I decided I would use that instead. Besides a few quirks, it worked flawlessly and was a much better solution than RCHooker.

# The Bug
## First clues
From previous reversing, I had found a function (that I called `go_to_level`) in R&C3 that looked like a good culprit. I managed to find its counterpart in R&C2 thanks to some debugging strings, placed a breakpoint and performed a regular takeoff; and indeed it got called. The caller, `FUN_81A3B046()`, was more intresting because it had two code pathes: if `levelId` (its first and only argument) was different from `CURRENT_LEVEL_ID` (a global variable), it would perform a call to `go_to_level`; if they were equal, something else was performed. 

I decided to moved my breakpoint to this function (with a `r0 != 1` condition, a.k.a. `levelId != LEVEL_OOZLA` since all takeoffs would be performed from Oozla) and try out the trick. Lo' and behold, after a few attempts, the debugger pauses and resuming flies me to Maktar. Just what we expected.

## Going up the stack
Thanks to the debugger (🙏), I also had call stacks so walking up the caller chain was **much better** than with RCHooker (where you can only dump `lr` and walk up one function at a time). The only caller Ghidra found was the one I had in my call stack, which is cool because it meant it must be a direct call (Ghidra usually fails to establish caller-callee relationships when indirect calls are perfomed). I had named this function `mode_doChange`, a name borrowed from the symboled Deadlocked build that was (re)discovered a few months back, so I understood it must have been implicated in the game mode mechanism. 

### Game modes
The game mode is an important part of the R&C engine. It consists of a few variables, the most important one being `gameMode` which holds the current game mode (duh!). Usually, changing game mode is performed by calling `mode_requestChange()` which sets `pendingGameMode` to the provided game mode, and some generic arguments (`rmc_arg0`/`rmc_arg1`). Later on, `mode_doChange` is called and performs actions based on the globals; first, it copies `gameMode` to `lastGameMode` (**last** stands for **previous** here), then it's mostly two `switch`es looking at `gameMode` and `pendingGameMode`. The code path we were hitting in this case is the following: `gameMode = GAME_MODE_PAUSE` (the ship menu), `pendingGameMode = GAME_MODE_SPACE` (takeoff / get-in / get-out animations) and `rmc_arg0 != 5 && rmg_arg0 != 6`. This leads to `rmc_arg1` being passed as the `levelId` for `FUN_81A3B046()`.

![Screenshot of the decompiled routine](/rac2_decomp_output.png)

If you hooked `FUN_81A3B046()` and exited the ship "normally", you'd see that `levelId` was always identical to `CURRENT_LEVEL_ID` and thus nothing particaly would happen. But when performing the bug, we'd get `LEVEL_MAKTAR`. I conducted a few tests: replacing the `0x2` with a `0x1` (`MAKTAR` -> `OOZLA`) resulted in the ship not taking off, and replacing it with `0x15` (`LEVEL_INSOMNIAC_MUSEUM`) made me fly towards it instead of Maktar. This gave me a rough overview of the bug: **`rmc_arg1` is `0x2` but it should be `0x1`**. Again, thanks to the debugger, I knew that `mode_doChange` was called by `FUN_819CC878()`, but it's a big function with mostly irrelevant stuff and it didn't answer my question: ***who writes 2 to `rmc_arg1` and why?***.

## Why is it 2?
The easiest solution to figure this out is to place a data change breakpoint on `rmc_arg1`. Because it's used a lot may be used by more than this, I added a condition `rmc_arg1 == 2` to reduce false positives, and performed the sequence again. The only function that touches it is `mode_pop`, which writes its second argument to `rmc_arg1`. I quickly glanced the call stack (`FUN_819CC878()` - main game loop -> `FUN_81A20E14()` - executed if current game mode is `GAME_MODE_PAUSE` -> `mode_pop`) and I noticed that a global (we'll call it `g_previewedLevelId`) was used for the second argument to `mode_pop`... You guessed it, another trail to follow. I set a breakpoint on that global and found that only one routine touches it  (`FUN_819CD7A6()`, which writes its first argument to `g_previewedLevelId`). 

This function is only called from three sites:

 1. When the ship menu appears, the call site is `0x81A1EE91`. This is part of a routine called when entering `GAME_MODE_PAUSE` and passes the current level ID to `FUN_819CD7A6()`, so it cannot be our culprit.

 1. When exiting the ship menu, the call site is `0x81A23C05`. This is part of what I assume to be the routine holding the bulk of the menu's logic, and also passes the current level ID to `FUN_819CD7A6()`, so it cannot be our culprit as well.

 1. When pressing Up/Down (i.e. changing which planet I highlight), the call site is `0x81A23DDB`, a bit further in the same routine as above. **The value passed to `FUN_819CD7A6()` is obtained through pointer dereferences**. I didn't bother RE'ing that part because all that mattered is that **this was the only caller with an argument that could be different** so it **had** to be our culprit.

I added logging breakpoint on each of the routines to see in which order they were called and used the menu normally (that is, entering the ship, moving up and down a few times, and exiting the ship with Triangle). This resulted in the functions being called in the order I was expecting: #1 once, #3 as many times as I changed highlighted planet and #2 once.
I then performed the bug and noticed the pattern was not respected: **caller #3 (Up/Down press handler) was executed even though we were exiting the menu**. This seemed even weirder because if you looked up the call chain, all those calls ended up coming from `FUN_81A20E14()`, which I noted earlier runs only when the game mode is `GAME_MODE_PAUSE`!

## Is this Space or Pause?
By now I had a rough overview of the problem but this seemingly mismatching gamemode puzzled me a lot. I used logging breakpoints again to see when the game mode was changed and also added one to know when my Triangle press was detected. Both of these loggers also print the "rendered frames counter" as a timestamp.

The resulting log was... unexpected:
```
TRIANGLE PRESSED @ 419425
SWITCH TO GM 6   @ 419428
```

*🤔 The game mode isn't changed directly?*

Yes. It turns out that game mode changes can be delayed by a few frames. This isn't something you'd catch from the naked eye - could ***you*** notice something is off by a frame in a 60FPS game?

However, this has very important implications. It means that *the `GAME_MODE_PAUSE` handler still runs after requesting to close the menu*, or in other words, that *we can change `g_previewedLevelId` after it is reset but before it is passed to a routine that can initiate the takeoff*. This type of bug is called a race condition because successful exploitation relies on winning a race (in our case, pressing a button at the right moment).

# The Aftermath
For speedrunners, this bug is obviously a blessing, considering it's a quite important timesave that can be achieved relatively easily. Wupash can also be a harsh level to master and isn't very exciting, so I'm confident not many people will regret it no longer being visited.

As for the R&C codebase, I don't know if this bug was accidentally found and fixed, or if was removed merely due to changes in the codebase, but it is *no longer present in R&C3* (as far as I'm aware - don't rub this in my face if a magic skip trick is found in R&C3 some day 😢). As I noted in the introduction, Insomniac had similar "planet redirection" schemes in R&C3.

They are only used for two levels:
 * `Starship Phoenix`
   * After completing Qwark's Hideout, Ratchet is called by Sasha because the Starship Phoenix is under attack. This `Starship Phoenix Rescue` level, with ID `6`, is separate from the regular `Starship Phoenix`, which has ID `3`. Since Ratchet can leave the "Save the Phoenix" mission, the game has to be able to redirect going to level `3` to level `6` under certain circumstances.

 * `Holostar Studios`
   * During the first visit to Holostar Studios, you play as Clank in the `Holostar Studio 43` level, with ID `23`, a separate level from the "normal" `Holostar Studios` level, which has ID `13`. Because the player initiates flying towards the level, however, there again needs to be some redirection scheme.

In R&C3, this is all handled inside `go_to_level` by checking the requested planet ID and changing it if needed. Compared to R&C2, this is a much saner scheme: since this function always ends up called on a level change, this ensures that no matter where the request comes from, nothing can bypass the redirection. As a bonus, this is also not being vulnerable to the race condition.

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
    if (g_previewedLevelId == 0 /* Aranos 1 - some failsafe? */ || BYTE_821C6F7D != 0 /* Wupash finished? */ || g_previewedLevelId != LEVEL_MAKTAR /* not going to Maktar - no redirection needed */) {
        /* ... */
    } else {
        //Overwrite rmc_arg1 with WUPASH_NEBULA to change the destination level
        mode_requestChange(GAME_MODE_SPACE, RMC_SWITCH, 1, LEVEL_WUPASH_NEBULA, NULL);
    }
    /* ... */
}
```