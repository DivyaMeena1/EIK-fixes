# Questionnaire Round Control Flow Guide

---

## Overview

The **Questionnaire (Main) Round** has been refactored to match the **Faceoff Round's control flow pattern**. Blueprint now has complete control over:

- Question data and source
- Answer validation logic
- Player turn order
- Round completion conditions
- Strikeout timing

This guide explains the new two-way dialog architecture between GameMode and Blueprint.

---

## Architecture Pattern

### Old Pattern (Notification-Only) ❌

```
GameMode owns everything → Blueprint reacts to notifications
```

- GameMode read `GameState->CurrentQuestion` to validate answers
- GameMode automatically cycled through players
- Blueprint only received notifications (OnAnswerCorrect, OnAnswerIncorrect)
- No Blueprint control over game flow

### New Pattern (Two-Way Control) ✅

```
GameMode asks Blueprint → Blueprint decides → Blueprint calls back → GameMode continues
```

- Blueprint provides question data
- Blueprint validates answers
- Blueprint controls turn order
- Blueprint decides when round ends
- GameMode provides infrastructure and state management

---

## Control Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│ MATCH PHASE CHANGES TO: Questionnaire                              │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│ GameMode: AFamilyFeudGameMode::StartMainRound()                    │
│ • Resets RoundStrikes, RoundPoints, CurrentPlayerIndex to 0        │
│ • Fires deprecated OnMainRoundStarted() for compatibility          │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│ SceneDirector: OnMainRoundStart_ProvideQuestion(PlayingTeam)       │
│ • BlueprintNativeEvent - Blueprint MUST override this              │
│ • Blueprint generates/fetches question data                        │
│ • Blueprint MUST call Server_SetQuestionData() to continue         │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Blueprint Callback: Server_SetQuestionData(Question, AnswerCount)  │
│ • Blueprint provides question text and number of answers           │
│ • Can be from AI, database, hardcoded, etc.                        │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│ GameMode: HandleQuestionDataSet(QuestionText, AnswerCount)         │
│ • Initializes RevealedAnswers array (all false)                    │
│ • Logs question data                                               │
│ • Round is now ready for answers                                   │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│ PLAYER SUBMITS ANSWER                                              │
│ PlayerCharacter → ReceiveMainRoundAnswer(PlayerState, Answer)      │
│ • Validates it's this player's turn                                │
│ • Prevents duplicate answers from same player                      │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│ SceneDirector: OnValidateAnswer(Player, Answer, QuestionText)      │
│ • BlueprintNativeEvent - Blueprint MUST override this              │
│ • Blueprint performs validation (AI, embeddings, string match)     │
│ • Blueprint MUST call Server_ReportAnswerResult() to continue      │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Blueprint Callback: Server_ReportAnswerResult()                    │
│ • Player: Who answered                                             │
│ • Answer: What they said                                           │
│ • bCorrect: Did they match a valid answer?                         │
│ • AnswerIndex: Which answer slot (0-7)                             │
│ • Points: How many points to award                                 │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│ GameMode: HandleAnswerResult(Player, Answer, bCorrect, Index, Pts) │
│ • If CORRECT:                                                       │
│   - Marks RevealedAnswers[Index] = true                            │
│   - Adds Points to RoundPoints                                     │
│   - Fires OnAnswerCorrect() (deprecated event)                     │
│   - Fires OnCheckRoundComplete() for Blueprint to decide next step │
│ • If INCORRECT:                                                     │
│   - Increments RoundStrikes                                        │
│   - Fires OnAnswerIncorrect() (deprecated event)                   │
│   - Fires OnStrike(StrikeCount, MaxStrikes)                        │
│   - Fires OnCheckStrikeout(StrikeCount) for Blueprint to decide    │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
                    ┌───────────────┴───────────────┐
                    │                               │
            ✅ CORRECT ANSWER              ❌ INCORRECT ANSWER
                    │                               │
                    ↓                               ↓
    ┌───────────────────────────────┐   ┌──────────────────────────────┐
    │ OnCheckRoundComplete()        │   │ OnCheckStrikeout(Strikes)    │
    │ Blueprint decides:            │   │ Blueprint decides:           │
    │ • All answers revealed?       │   │ • Reached 3 strikes?         │
    │ • Want to end early?          │   │ • Give second chance?        │
    └───────────────────────────────┘   └──────────────────────────────┘
                    │                               │
        ┌───────────┴───────────┐           ┌──────┴──────┐
        │                       │           │             │
    End Round            Next Player    Continue    Trigger Strikeout
        │                       │           │             │
        ↓                       ↓           ↓             ↓
Server_EndMainRound()  Server_SetNextPlayer()  (Wait)  Server_TriggerStrikeout()
        │                       │                         │
        ↓                       ↓                         ↓
HandleMainRoundEnd()   HandleNextPlayerSet()   HandleStrikeout()
        │                       │                         │
        ↓                       ↓                         ↓
CompleteMainRound()    Wait for answer         StartStealOpportunity()
        │                       │                         │
        ↓                       │                         ↓
 Results Phase          ← Loop back           Steal Voting Phase
```

---

## Blueprint Implementation Guide

### Step 1: Override OnMainRoundStart_ProvideQuestion

This event fires when the match phase transitions to Questionnaire. Blueprint must provide question data.

**Event Signature:**
```cpp
UFUNCTION(BlueprintNativeEvent, Category="Feud|MainRound|Control")
void OnMainRoundStart_ProvideQuestion(EFFTeam PlayingTeam);
```

**Blueprint Example:**
```
Event OnMainRoundStart_ProvideQuestion
    ↓
Generate Question (AI, Database, Hardcoded)
    ↓
Server_SetQuestionData("What's the best pet?", 8)  // 8 possible answers
```

**What to Do:**
1. Generate or fetch question data from any source:
   - Call AI (ChatGPT, Claude, etc.)
   - Query database
   - Use hardcoded questions
   - Parse from JSON/CSV
2. Determine how many valid answers exist (1-8)
3. **MUST call `Server_SetQuestionData(QuestionText, AnswerCount)`**

**Parameters for Server_SetQuestionData:**
- `QuestionText` (string) - The question to display (optional, can be empty)
- `AnswerCount` (int32) - Number of valid survey answers (required)

---

### Step 2: Override OnValidateAnswer

This event fires every time a player submits an answer. Blueprint must validate it.

**Event Signature:**
```cpp
UFUNCTION(BlueprintNativeEvent, Category="Feud|MainRound|Control")
void OnValidateAnswer(AFermataPlayerState* Player, const FString& Answer, const FString& QuestionText);
```

**Blueprint Example:**
```
Event OnValidateAnswer
    Player: TeamAPlayer1
    Answer: "dog"
    QuestionText: "What's the best pet?"
    ↓
Check Answer (AI, Embeddings, String Match)
    ↓
[If Match Found]
    Server_ReportAnswerResult(Player, "dog", true, 0, 45)
    // true = correct, 0 = first answer slot, 45 points
    ↓
[If No Match]
    Server_ReportAnswerResult(Player, "dinosaur", false, -1, 0)
    // false = incorrect, -1 = no slot, 0 points
```

**What to Do:**
1. Validate the player's answer against your question data:
   - Use AI/LLM for semantic matching
   - Use Inworld Text Embeddings for similarity
   - Use string matching with fuzzy logic
   - Use exact string comparison
2. Determine if correct and which answer slot it matches
3. **MUST call `Server_ReportAnswerResult()`**

**Parameters for Server_ReportAnswerResult:**
- `Player` (AFermataPlayerState*) - Who answered
- `Answer` (string) - What they said (for logging/display)
- `bCorrect` (bool) - Did they match a valid answer?
- `AnswerIndex` (int32) - Which answer slot (0-7) or -1 if incorrect
- `Points` (int32) - How many points to award (if correct)

---

### Step 3: Override OnCheckRoundComplete

This event fires after every **correct** answer. Blueprint decides whether to end the round or continue.

**Event Signature:**
```cpp
UFUNCTION(BlueprintNativeEvent, Category="Feud|MainRound|Control")
void OnCheckRoundComplete();
```

**Blueprint Example:**
```
Event OnCheckRoundComplete
    ↓
Check RevealedAnswers Array
    ↓
[If All 8 Answers Revealed]
    Server_EndMainRound(true)  // true = all answers revealed
    ↓
[If Only Top 3 Revealed, Want to Continue]
    Get Next Player in Lineup
    Server_SetNextPlayer(NextPlayerState)
    ↓
[If Want to End Early]
    Server_EndMainRound(false)  // false = early end
```

**What to Do:**
1. Check `GameState->RevealedAnswers` array to see what's been found
2. Decide if round should end:
   - All answers revealed → Call `Server_EndMainRound(true)`
   - Want to end early → Call `Server_EndMainRound(false)`
   - Want to continue → Call `Server_SetNextPlayer(NextPS)`

**Parameters for Server_EndMainRound:**
- `bAllAnswersRevealed` (bool) - Did they find everything?

**Parameters for Server_SetNextPlayer:**
- `NextPlayer` (AFermataPlayerState*) - Who answers next

---

### Step 4: Override OnCheckStrikeout

This event fires after every **incorrect** answer. Blueprint decides when to trigger strikeout.

**Event Signature:**
```cpp
UFUNCTION(BlueprintNativeEvent, Category="Feud|MainRound|Control")
void OnCheckStrikeout(int32 CurrentStrikes);
```

**Blueprint Example:**
```
Event OnCheckStrikeout
    CurrentStrikes: 3
    ↓
[If Strikes >= 3]
    Play Strikeout Animation
    Wait for Animation Complete
    Server_TriggerStrikeout(TeamA)  // TeamA struck out
    ↓
[If Strikes < 3]
    // Do nothing, round continues
```

**What to Do:**
1. Check `CurrentStrikes` parameter
2. Decide if it's time for strikeout:
   - Standard rules: 3 strikes → Call `Server_TriggerStrikeout(FailedTeam)`
   - Custom rules: Could trigger earlier/later
   - Could give second chances based on game state
3. If not triggering strikeout, do nothing (round continues)

**Parameters for Server_TriggerStrikeout:**
- `FailedTeam` (EFFTeam) - Which team struck out

---

## GameState Properties (UI Only)

These properties are **deprecated for game logic** but still replicate for UI display:

```cpp
/** DEPRECATED FOR LOGIC: Current question (UI reference only) */
UPROPERTY(ReplicatedUsing=OnRep_CurrentQuestion, BlueprintReadOnly)
FSurveyQuestion CurrentQuestion;

/** DEPRECATED FOR LOGIC: Strike count (UI reference only) */
UPROPERTY(Replicated, BlueprintReadOnly)
int32 RoundStrikes = 0;

/** DEPRECATED FOR LOGIC: Which answers revealed (UI reference only) */
UPROPERTY(Replicated, BlueprintReadOnly)
TArray<bool> RevealedAnswers;

/** DEPRECATED FOR LOGIC: Accumulated points (UI reference only) */
UPROPERTY(Replicated, BlueprintReadOnly)
int32 RoundPoints = 0;
```

**Important:**
- Blueprint can READ these for UI updates
- Blueprint should NOT use these for validation logic
- Blueprint owns the question data source
- GameMode updates these for backwards compatibility only

---

## Complete Blueprint Implementation Example

```blueprint
// ====================================================================
// STEP 1: Provide Question Data
// ====================================================================
Event OnMainRoundStart_ProvideQuestion (PlayingTeam)
    ↓
Ask ChatGPT: "Generate a Family Feud question with 8 answers"
    ↓
Parse JSON Response → Store in custom Blueprint variable
    ↓
Server_SetQuestionData(QuestionText, 8)


// ====================================================================
// STEP 2: Validate Answer
// ====================================================================
Event OnValidateAnswer (Player, Answer, QuestionText)
    ↓
FOR EACH stored answer in custom Blueprint variable:
    ↓
    Use Inworld Text Embedding to compare similarity
    ↓
    [If Similarity > 0.85]
        Server_ReportAnswerResult(Player, Answer, TRUE, AnswerIndex, Points)
        RETURN
    ↓
[If No Match Found]
    Server_ReportAnswerResult(Player, Answer, FALSE, -1, 0)


// ====================================================================
// STEP 3: Check Round Complete
// ====================================================================
Event OnCheckRoundComplete ()
    ↓
Count TRUE in GameState->RevealedAnswers
    ↓
[If Count == 8]  // All answers found
    Server_EndMainRound(TRUE)
    ↓
[Else]  // More answers remain
    Get Next Player from Lineup
    Server_SetNextPlayer(NextPlayer)


// ====================================================================
// STEP 4: Check Strikeout
// ====================================================================
Event OnCheckStrikeout (CurrentStrikes)
    ↓
[If CurrentStrikes >= 3]
    Play Steve Harvey Reaction Animation
    Wait 2 seconds
    Server_TriggerStrikeout(CurrentTeam)
    ↓
[Else]
    // Do nothing, keep playing
```

---

## Migration from Old Pattern

If you have existing Blueprint code using the old pattern, here's how to migrate:

### Old Code (Notification Pattern)
```blueprint
// GameMode auto-validated answers
Event OnAnswerCorrect (Answer, Points)
    Play celebration animation
    Update UI

Event OnAnswerIncorrect (Answer)
    Play buzzer sound
    Update UI
```

### New Code (Control Pattern)
```blueprint
// Blueprint validates answers
Event OnValidateAnswer (Player, Answer, QuestionText)
    ↓
    Validate answer with your logic
    ↓
    Server_ReportAnswerResult(Player, Answer, bCorrect, Index, Points)

// Deprecated events STILL FIRE for backwards compatibility
Event OnAnswerCorrect (Answer, Points)
    Play celebration animation
    Update UI

Event OnAnswerIncorrect (Answer)
    Play buzzer sound
    Update UI
```

**Key Differences:**
1. **Must override new control events** (`OnValidateAnswer`, etc.)
2. **Must call Server RPCs** to report decisions
3. **Old events still work** for animations/UI
4. **Blueprint owns question data** instead of GameMode

---

## Deprecated Functions (Backwards Compatibility)

These functions are marked `UE_DEPRECATED` but still functional:

**SceneDirectorComponent:**
- `OnMainRoundStarted()` - Still fires, use `OnMainRoundStart_ProvideQuestion()` instead
- `OnAnswerCorrect()` - Still fires, use `OnValidateAnswer()` + `Server_ReportAnswerResult()` instead
- `OnAnswerIncorrect()` - Still fires, use `OnValidateAnswer()` + `Server_ReportAnswerResult()` instead
- `OnStrikeout()` - Still fires, use `OnCheckStrikeout()` + `Server_TriggerStrikeout()` instead

**FamilyFeudGameMode:**
- `ValidateAnswer()` - Old C++ string matching, use Blueprint validation instead
- `AdvanceToNextPlayer()` - Old auto-cycling, use `Server_SetNextPlayer()` instead
- `AwardStrike()` - Old strike logic, use `Server_TriggerStrikeout()` instead

---

## Common Pitfalls

### ❌ Forgetting to Call Server RPCs
```blueprint
Event OnValidateAnswer (Player, Answer, QuestionText)
    Validate answer
    // FORGOT TO CALL Server_ReportAnswerResult()
    // Round will hang waiting for Blueprint response!
```

**Fix:** Always call the corresponding Server RPC.

---

### ❌ Using GameState->CurrentQuestion for Logic
```blueprint
Event OnValidateAnswer (Player, Answer, QuestionText)
    FOR EACH Answer in GameState->CurrentQuestion.Answers:  // ❌ WRONG
        Check match
```

**Fix:** Blueprint owns question data. Store it in your own variables.

```blueprint
Event OnMainRoundStart_ProvideQuestion (PlayingTeam)
    Generate question → Store in MyQuestionData variable  // ✅ CORRECT

Event OnValidateAnswer (Player, Answer, QuestionText)
    FOR EACH Answer in MyQuestionData.Answers:  // ✅ CORRECT
        Check match
```

---

### ❌ Not Handling All Code Paths
```blueprint
Event OnCheckRoundComplete ()
    [If All Answers Revealed]
        Server_EndMainRound(TRUE)
    // FORGOT ELSE CASE - Round will hang!
```

**Fix:** Always call either `Server_EndMainRound()` OR `Server_SetNextPlayer()`.

```blueprint
Event OnCheckRoundComplete ()
    [If All Answers Revealed]
        Server_EndMainRound(TRUE)
    [Else]
        Server_SetNextPlayer(NextPlayer)  // ✅ CORRECT
```

---

## Testing Checklist

- [ ] Blueprint overrides `OnMainRoundStart_ProvideQuestion`
- [ ] Blueprint calls `Server_SetQuestionData()` with valid data
- [ ] Blueprint overrides `OnValidateAnswer`
- [ ] Blueprint calls `Server_ReportAnswerResult()` for every answer
- [ ] Blueprint overrides `OnCheckRoundComplete`
- [ ] Blueprint calls either `Server_EndMainRound()` OR `Server_SetNextPlayer()`
- [ ] Blueprint overrides `OnCheckStrikeout`
- [ ] Blueprint calls `Server_TriggerStrikeout()` when appropriate
- [ ] Round doesn't hang waiting for callbacks
- [ ] Points accumulate correctly in `GameState->RoundPoints`
- [ ] Strikes increment correctly in `GameState->RoundStrikes`
- [ ] RevealedAnswers array updates correctly
- [ ] Steal opportunity triggers after 3 strikes
- [ ] Round ends when all answers revealed

---

## File References

**Core Files:**
- `/FamilyFeud/Plugins/FermataCore/Source/FermataCore/Public/Components/SceneDirectorComponent.h` (lines 118-376)
- `/FamilyFeud/Plugins/FermataCore/Source/FermataCore/Private/Components/SceneDirectorComponent.cpp` (lines 129-248)
- `/FamilyFeud/Plugins/FermataCore/Source/FermataCore/Public/FamilyFeud/FamilyFeudGameMode.h` (lines 286-391)
- `/FamilyFeud/Plugins/FermataCore/Source/FermataCore/Private/FamilyFeud/FamilyFeudGameMode.cpp` (lines 1661-1935)

---

## Questions?

This control flow pattern gives Blueprint complete flexibility over:
- Question generation (AI, database, hardcoded)
- Answer validation (semantic, fuzzy, exact)
- Turn order (sequential, dynamic, skill-based)
- Round completion (all answers, early end, time limit)
- Strikeout timing (standard, second chances, custom rules)

The GameMode provides infrastructure while Blueprint owns all game logic decisions.
