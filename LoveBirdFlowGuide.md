# Love Bird Game Flow Guide

## Server RPCs (Blueprint Calls These)

Call these functions from Blueprint to control the game flow:

```cpp
// 1. Start the show intro
SceneDirector->Server_StartShow()

// 2. Discover all contestants (finds AFermataCharacter with tag 'contestant')
SceneDirector->Server_DiscoverContestants()

// 3. Start pleading phase (resets index to 0, starts first contestant)
SceneDirector->Server_StartPleadingPhase()

// 4. Mark current contestant's plea as finished
SceneDirector->Server_FinishContestantPlea(Contestant)

// 5. Start next contestant's plea (advances index, starts next)
SceneDirector->Server_StartNextContestantPlea()

// 6. Start elimination phase
SceneDirector->Server_StartElimination()

// 7. Give feather to contestant (safe from elimination)
SceneDirector->Server_GiveFeather(Contestant)

// 8. Start vulture snatch phase
SceneDirector->Server_StartVulturePhase()

// 9. Close the show
SceneDirector->Server_CloseShow()
```

## Blueprint Events (Blueprint Overrides These)

Override these events in your Blueprint to react to game flow:

```cpp
// Phase 1: Introduction
OnShowIntroduction()

// Phase 2: Pleading
OnContestantsDiscovered(DiscoveredContestants, TotalCount)
OnContestantPleaStarted(Contestant, ContestantIndex)
OnContestantPleaFinished(Contestant)
OnAllContestantPleasFinished()

// Phase 3: Elimination
OnEliminationStarted()
OnFeatherGiven(Contestant, FeatherNumber)
OnFeatherReceived(Contestant, FeatherNumber)
OnEliminationFinal(RemainingContestants, RemovedContestants)

// Phase 4: Vultures
OnVulturesSnatch(TargetContestants)
OnContestantSnatched(Contestant)
OnVulturesFinished()

// Phase 5: Closing
OnShowClosing()
OnShowComplete()
```

## State Properties (Blueprint Reads These)

Access these replicated properties from SceneDirector:

```cpp
ELoveBirdPhase CurrentPhase                        // Intro, Pleading, Elimination, Vultures, Closing
TArray<AFermataCharacter*> Contestants             // All discovered contestants
TArray<AFermataCharacter*> SafeContestants         // Contestants who received feathers
TArray<AFermataCharacter*> EliminatedContestants   // Contestants being eliminated
int32 FeathersGivenCount                           // Number of feathers given (0-2)
int32 CurrentContestantIndex                       // Index of current contestant pleading
```

## Example Flow

```
Blueprint Event Graph:

1. OnShowIntroduction()
   └─> [Play intro animations]
   └─> Call: Server_DiscoverContestants()

2. OnContestantsDiscovered(Contestants, Count)
   └─> [Verify all contestants ready]
   └─> Call: Server_StartPleadingPhase()

3. OnContestantPleaStarted(Contestant, Index)
   └─> [Play plea dialogue/animation]
   └─> [Wait for completion]
   └─> Call: Server_FinishContestantPlea(Contestant)

4. OnContestantPleaFinished(Contestant)
   └─> [Transition animation]
   └─> If more contestants: Call Server_StartNextContestantPlea()
   └─> Else: Wait for OnAllContestantPleasFinished()

5. OnAllContestantPleasFinished()
   └─> [All pleas done]
   └─> Call: Server_StartElimination()

6. OnEliminationStarted()
   └─> [Show elimination UI]
   └─> [Wait for player choice]
   └─> Call: Server_GiveFeather(ChosenContestant) // Do this twice

7. OnFeatherGiven(Contestant, FeatherNumber)
   └─> [Play celebration VFX]

8. OnEliminationFinal(SafeContestants, EliminatedContestants)
   └─> [Show final results]
   └─> Call: Server_StartVulturePhase()

9. OnVulturesSnatch(TargetContestants)
   └─> [Trigger vulture animations]

10. OnContestantSnatched(Contestant) // Called for each eliminated contestant
    └─> [Individual snatch VFX]

11. OnVulturesFinished()
    └─> [Cleanup vulture sequence]
    └─> Call: Server_CloseShow()

12. OnShowClosing()
    └─> [Play outro dialogue]
    └─> [Optional: Call OnShowComplete when ready]
```

## Key Rules

- **C++ only fires events** - Blueprint decides when to progress
- **Server_GiveFeather must be called exactly twice** - After 2nd call, OnEliminationFinal fires
- **All contestants must have tag 'contestant'** - Server_DiscoverContestants finds them by tag
