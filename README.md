# Dangling Pointer Fixes for UE 5.5 on Mac

## Summary
Fixed 85+ dangling pointer errors in the EOSIntegrationKit plugin caused by UE 5.5's stricter `-Werror=dangling-assignment` compiler flag on Mac.

## The Problem

**Error Message:**
```
error: object backing the pointer [FieldName] will be destroyed at the end of the full-expression [-Werror,-Wdangling-assignment]
```

**Root Cause:**
UE 5.5 treats the deprecated `TCHAR_TO_ANSI()` and `TCHAR_TO_UTF8()` macros as fatal errors because they create temporary objects that are destroyed immediately, leaving dangling pointers.

**Bad Code (causes error):**
```cpp
Options.Field = TCHAR_TO_ANSI(*MyString);  // ❌ Temporary destroyed immediately
Options.Field = StringCast<ANSICHAR>(*MyString).Get();  // ❌ Same problem
```

---

## Fix Pattern 1: Simple Field Assignment

**Use Case:** Single field assignment (e.g., `Options.FieldName = ...`)

**Solution:**
```cpp
// Create thread-local storage for the converter
static thread_local TAlignedBytes<sizeof(FTCHARToUTF8), alignof(FTCHARToUTF8)> FieldName_Storage;
FTCHARToUTF8* FieldName_Conv = new (&FieldName_Storage) FTCHARToUTF8(*MyString);
Options.FieldName = FieldName_Conv->Get();
```

**Example Files Fixed:**
- `EIK_Auth_DeletePersistentAuth.cpp:32` - `Options.RefreshToken`
- `EIK_AchievementsSubsystem.cpp:182` - `Options.AchievementId`
- `EIK_PlayerDataStorage_DeleteFile.cpp:27` - `Options.Filename`
- `EIK_PresenceSubsystem.cpp:248` - `Options.RichText`

**Key Points:**
- Each field needs its own uniquely named storage variable
- Use `static thread_local` to ensure thread safety
- Use placement new with `TAlignedBytes` for proper alignment
- Use `FTCHARToUTF8` (NOT `FTCHARToANSI` which doesn't exist in UE 5.5)

---

## Fix Pattern 2: Array Assignment in Loop

**Use Case:** Filling an array of `char*` pointers in a loop

**Bad Code:**
```cpp
char** TempArray = new char*[Count];
for (int i = 0; i < Count; i++)
{
    TempArray[i] = TCHAR_TO_ANSI(*SourceArray[i]);  // ❌ Temporary destroyed
}
Options.Array = TempArray;
```

**Solution:**
```cpp
// Allocate string buffers to hold converted strings
TArray<TArray<char>> StringBuffers;
TArray<const char*> PointerArray;
StringBuffers.Reserve(SourceArray.Num());
PointerArray.Reserve(SourceArray.Num());

for (int i = 0; i < SourceArray.Num(); i++)
{
    FTCHARToUTF8 Converter(*SourceArray[i]);
    TArray<char>& Buffer = StringBuffers.Add_GetRef(TArray<char>());
    Buffer.SetNumUninitialized(Converter.Length() + 1);
    FCStringAnsi::Strcpy(Buffer.GetData(), Converter.Length() + 1, Converter.Get());
    PointerArray.Add(Buffer.GetData());
}

Options.Array = PointerArray.GetData();
```

**Example Files Fixed:**
- `EIK_Achievements_UnlockAchievements.cpp:32` - `Options.AchievementIds`
- `EIK_Connect_QueryExternalAccountMappings.cpp:53` - `ExternalAccountIds`
- `EIK_Stats_QueryStats.cpp:55` - `Options.StatNames`

**Why This Works:**
- `StringBuffers` keeps the character buffers alive for the function duration
- `PointerArray` holds stable pointers to those buffers
- Buffers remain valid until the function exits (after the EOS SDK call)

**Why NOT to use `TArray<FTCHARToUTF8>`:**
- `FTCHARToUTF8` has **deleted copy/move constructors**
- Cannot be stored in a `TArray`
- Compiler error: `call to deleted constructor of 'ElementType'`

---

## What to Avoid

### ❌ Don't Use These (Deprecated in UE 5.5):
```cpp
TCHAR_TO_ANSI(*String)
TCHAR_TO_UTF8(*String)
StringCast<ANSICHAR>(*String).Get()  // Without storage
FTCHARToANSI  // Doesn't exist in UE 5.5
```

### ❌ Don't Try to Store Converters in Arrays:
```cpp
TArray<FTCHARToUTF8> Converters;  // ❌ Won't compile (deleted constructors)
```

### ❌ Don't Create Variable-Length Arrays:
```cpp
static thread_local TAlignedBytes<...> Storage[i];  // ❌ VLA error
```

---

## Quick Reference

| Scenario | Pattern to Use |
|----------|---------------|
| Single field assignment | Pattern 1: Placement new with `TAlignedBytes` |
| Multiple fields (same struct) | Pattern 1: Unique storage for each field |
| Array of strings in loop | Pattern 2: `TArray<TArray<char>>` buffer storage |
| Duplicate field names | Use unique storage names (e.g., `SourceFilename_Storage`, `DestinationFilename_Storage`) |

---

## Files Modified (19 total)

### Simple Assignment Pattern (16 files):
1. `EIK_Auth_DeletePersistentAuth.cpp:32`
2. `EIK_AchievementsSubsystem.cpp:182`
3. `EIK_ConnectSubsystem.cpp:197`
4. `EIK_BlueprintFunctions.cpp:447`
5. `EIK_CreateDeviceId_AsyncFunction.cpp:38`
6. `EIK_FindUserByDisplayName_Async.cpp:34`
7. `EIK_GetPUIDFromEpicId_AsyncFunc.cpp:118`
8. `EIK_PlayerDataStorage_DeleteFile.cpp:27`
9. `EIK_PlayerDataStorage_DuplicateFile.cpp:26,27` (2 fields)
10. `EIK_PlayerDataStorage_QueryFile.cpp:25`
11. `EIK_PlayerDataStorage_ReadFile.cpp:67`
12. `EIK_PlayerDataStorage_WriteFile.cpp:63`
13. `EIK_PresenceSubsystem.cpp:248`
14. `EIK_Sanctions_CreatePlayerSanctionAppeal.cpp:29`

### Array-in-Loop Pattern (3 files):
1. `EIK_Achievements_UnlockAchievements.cpp:32`
2. `EIK_Connect_QueryExternalAccountMappings.cpp:53`
3. `EIK_Stats_QueryStats.cpp:55`

---

## Platform-Specific Notes

- **Mac Only:** This error is treated as fatal on Mac due to stricter Clang warnings
- **Windows:** May compile with warnings but should still be fixed for cross-platform compatibility
- **UE 5.5+:** All platforms will eventually enforce this stricter behavior

