# AGENT_HANDOFF.md

## Purpose

This file is a handoff for any future agent, AI assistant, or human helper working with sh1he on the ADOFAI no-fail DLL patch.

The goal is to preserve what was learned, avoid repeating failed patches, and give a practical search strategy if a new game version moves the relevant code.

## Quick Summary

- The confirmed working patch is in `scrPlayer.Die(bool, bool, string, bool)`, not `scrController.FailAction(bool, bool, string, bool)`.
- The key IL idea is to force the `noFail` branch to behave as true.
- The immediate `ret` patch in `FailAction` should be avoided because it caused broken behaviour: the balls/planets disappeared while the background stayed visible.

## Scope and Use

This document is for private local modding, debugging, and compatibility work only.

Use the patched DLL for local testing and personal play. Do not use it for online leaderboard submission, competitive ranking, or redistribution without permission.

Date: 2026-06-04

## Goal

Make A Dance of Fire and Ice avoid true fail/death behaviour after a miss, while still using the game's own no-fail behaviour where possible.

Expected result:

- The controlled balls/planets do not disappear after a fail.
- The level continues or behaves like the official no-fail/miss handling path.
- The game does not get stuck with only the background visible.

## Confirmed Working Patch

The working patch was not in:

```text
scrController.FailAction(bool, bool, string, bool)
```

The working patch was in:

```text
scrPlayer.Die(bool, bool, string, bool)
```

dnSpy showed the method around:

```text
@06000699
```

The successful IL patch changed the game's `noFail` check from reading the real field to always behaving as true.

Original IL shape:

```text
call   class scrController ADOBase::get_controller()
ldfld  bool scrController::noFail
brtrue.s TARGET
```

Working patched shape:

```text
call      class scrController ADOBase::get_controller()
pop
ldc.i4.1
brtrue.s TARGET
```

Meaning:

```text
Get the controller, discard it, push true, always take the no-fail branch.
```

This was tested in-game and worked.

## What Did Not Work

At first, `scrController.FailAction(bool, bool, string, bool)` was patched by putting an immediate `ret` at the start of the method.

That was too blunt. The visible result was:

- the balls disappeared,
- the background stayed,
- the fail screen/cleanup was skipped.

Conclusion: `FailAction` is not early enough in this game version. By the time it runs, another method has already started death/fail behaviour.

## Important Version Difference

A friend suggested patching a `noFail` branch inside `scrController.FailAction`.

That may be correct for another version, but in the DLL we actually patched, `scrController.FailAction` did not read `scrController.noFail`.

We confirmed this by searching/analysing:

```text
scrController.noFail : bool
```

In this version, `FailAction` was not listed under `Read By`.

The real useful method was:

```text
scrPlayer.Die(bool, bool, string, bool)
```

## Exact dnSpy Workflow

1. Open the real target DLL in dnSpy:

   ```text
   ADOFAI_Data\Managed\Assembly-CSharp.dll
   ```

2. Search only the selected/fresh DLL, not all loaded assemblies.

   This matters because dnSpy can keep multiple old copies loaded and search results can point to the wrong file.

3. Search for:

   ```text
   noFail
   ```

4. Select:

   ```text
   scrController.noFail : bool
   ```

5. Right-click it and choose:

   ```text
   Analyze
   ```

6. Expand:

   ```text
   Read By
   ```

7. Open:

   ```text
   scrPlayer.Die(bool, bool, string, bool)
   ```

8. Right-click `Die` and choose:

   ```text
   Edit Method Body...
   ```

9. Find the IL sequence:

   ```text
   call   class scrController ADOBase::get_controller()
   ldfld  bool scrController::noFail
   brtrue.s ...
   ```

10. Change the `ldfld` row to:

    ```text
    pop
    ```

    Clear its Operand.

11. Add a new instruction before `brtrue.s`:

    ```text
    ldc.i4.1
    ```

12. Leave the branch target alone unless dnSpy automatically adjusts it.

    In the final working edit, the target displayed as `49` after inserting the new row. That was okay.

13. If dnSpy complains about max stack calculation, check:

    ```text
    Keep Old MaxStack
    ```

14. Click `OK`.

15. Save:

    ```text
    File > Save Module
    ```

16. Put the patched DLL into the real Steam game folder:

    ```text
    ADOFAI_Data\Managed\Assembly-CSharp.dll
    ```

17. Test by intentionally missing in-game.

## What To Watch For

### Multiple DLLs Loaded In dnSpy

This caused major confusion.

If dnSpy has old files like these loaded:

```text
Assembly-CSharp-old-changed.dll
Assembly-CSharp.old-ret-patch.dll
```

search/analyser results may point to the wrong DLL.

Use one of these approaches:

- remove old assemblies from dnSpy,
- restart dnSpy,
- search only `Selected Files`,
- open the real Steam DLL directly.

### Do Not Patch `midspinInfiniteMargin`

We saw:

```text
bool scrController::midspinInfiniteMargin
```

This is not the same as:

```text
bool scrController::noFail
```

Do not edit `midspinInfiniteMargin` unless there is new evidence.

### Do Not Use The Immediate `ret` Patch

Do not simply put `ret` at the start of:

```text
scrController.FailAction(...)
```

That patch caused broken behaviour where the player disappeared but the background stayed.

Avoid this patch unless the goal is only to block the fail screen and you know the earlier death code does not matter.

## If A New Game Version Comes Out

Do not trust exact line numbers or offsets. Start from behaviour.

Recommended search path:

1. Search for:

   ```text
   noFail
   ```

2. Analyze:

   ```text
   scrController.noFail
   ```

3. Inspect:

   ```text
   Read By
   ```

4. Prioritise methods related to player death/failure:

   ```text
   scrPlayer.Die
   scrPlayer.OnDamage
   scrPlayer.CheckPostHoldFail
   ffxKillPlayer.StartEffect
   scrController.FailAction
   ```

5. Look for a branch that uses `noFail` to choose between a death path and a no-fail/miss path.

6. Force that branch to treat `noFail` as true.

Useful IL replacements:

If original:

```text
ldarg.0
ldfld bool SomeClass::noFail
brtrue.s TARGET
```

Possible patch:

```text
ldc.i4.1
brtrue.s TARGET
```

or, if stack balance is easier:

```text
ldarg.0
pop
ldc.i4.1
brtrue.s TARGET
```

If original:

```text
call  ...get_controller()
ldfld bool scrController::noFail
brtrue.s TARGET
```

Patch:

```text
call  ...get_controller()
pop
ldc.i4.1
brtrue.s TARGET
```

## Verification

After saving, place the patched DLL into:

```text
ADOFAI_Data\Managed\Assembly-CSharp.dll
```

Launch the game and intentionally miss.

Expected result:

- The game uses no-fail behaviour.
- The controlled balls/planets do not disappear.
- The game does not get stuck on only the background.

## Final Verified Result

After patching `scrPlayer.Die`, saving the module, and testing in-game, the no-fail patch worked.

The winning logic was:

```text
call ADOBase::get_controller()
pop
ldc.i4.1
brtrue.s ...
```

This forces the game to treat `noFail` as true at the important death decision point.

## Changelog

- 2026-06-04: Documented the working `scrPlayer.Die(bool, bool, string, bool)` no-fail patch.

## Signature

Prepared for sh1he.

Signed:

```text
sh1he's agent - Codex
2026-06-04
```
