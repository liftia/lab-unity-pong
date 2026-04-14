# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Unity 6 (6000.4.0f1) Pong game built on the Game Systems Cookbook architecture. Event-driven design using ScriptableObject-based pub/sub. Two assemblies: `GameSystems.Core` (reusable framework) and `GameSystems.PaddleBall` (game logic).

## Commands

### Run EditMode tests (CLI, no Unity Editor needed)
```bash
/Applications/Unity/Hub/Editor/6000.4.0f1/Unity.app/Contents/MacOS/Unity \
  -batchmode -nographics \
  -projectPath . \
  -runTests -testPlatform EditMode \
  -testResults /tmp/unity-test-results.xml \
  -logFile /tmp/unity-test.log
```
Exit code 0 = all pass. Results in XML at `/tmp/unity-test-results.xml`.

### Run a single test class
```bash
/Applications/Unity/Hub/Editor/6000.4.0f1/Unity.app/Contents/MacOS/Unity \
  -batchmode -nographics \
  -projectPath . \
  -runTests -testPlatform EditMode \
  -testFilter "ScoreTests" \
  -testResults /tmp/unity-test-results.xml \
  -logFile /tmp/unity-test.log
```

### Open project in Unity
Unity Hub > Add project from disk. Scenes: `Assets/PaddleBall/Scenes/Bootloader_Scene.unity`.

## Architecture

### Event System (Core/EventChannels)
`GenericEventChannelSO<T>` — ScriptableObject-based pub/sub. Concrete channels: `VoidEventChannelSO`, `IntEventChannelSO`, `FloatEventChannelSO`, `BoolEventChannelSO`, `StringEventChannelSO`, `Vector2EventChannelSO`, `Vector3EventChannelSO`, `PlayerIDEventChannelSO`, etc. All game communication flows through these channels — systems never reference each other directly.

### ScriptableObject Data Layer
- **GameDataSO** — central game config (speeds, masses, drag, player IDs, level layout reference)
- **LevelLayoutSO** — level geometry (ball/paddle positions, walls, goals). Supports JSON export/import
- **InputReaderSO** — bridges Unity InputSystem → UnityAction events (`P1Moved`, `P2Moved`, `GameRestarted`)
- **ScoreObjectiveSO** — win condition, extends `ObjectiveSO` from Core/Objectives
- **PlayerIDSO** — player identity tokens

### Game Flow
`Bootloader_Scene` → `GameSetup.Initialize()` spawns level from `LevelLayoutSO` → `GameManager` orchestrates via event subscriptions (goal scored → score update → objective check → win screen) → `ScoreManager` maintains scores and broadcasts to UI/objectives.

### Assembly Structure
| Assembly | Path | Dependencies |
|----------|------|-------------|
| `GameSystems.Core` | `Assets/Core/` | None |
| `GameSystems.PaddleBall` | `Assets/PaddleBall/Scripts/` | Core, InputSystem, TextMeshPro |
| `EditModeTests` | `Assets/Tests/EditMode/` | Core, PaddleBall, NUnit |

### Key Patterns
- **NullRefChecker** — reflection-based validation of `[SerializeField]` fields at SO enable time. Fields marked `[Optional]` are skipped. Uses `Debug.LogError` for missing assignments.
- **Objective System** — `ObjectiveManager` tracks multiple `ObjectiveSO` instances, broadcasts completion via event channel when all met.
- **UIManager** — stack-based screen navigation with history (push/pop views).
- **Initialize pattern** — explicit `Initialize()` methods over `OnEnable()` when dependencies needed.

## Tests

Tests live in `Assets/Tests/EditMode/` (NUnit, Editor-only). Test namespace: `PaddleBall.Tests`.

Current test classes: `ScoreTests`, `EventChannelTests`, `GameDataTests`, `ScoreObjectiveTests`.

Use `Assert.That` constraint model (not legacy `Assert.AreEqual`). When testing SOs that run `NullRefChecker` on enable, wrap creation with `LogAssert.ignoreFailingMessages` to suppress expected errors.

## Controls

Player 1: W/S — Player 2: Arrow Up/Down — Restart: per InputSystem action map.

## Dependencies

Key packages: Unity InputSystem 1.19.0, URP 17.4.0, UI Toolkit 2.0.0, Test Framework 1.6.0, TextMeshPro, Timeline 1.8.11, AI Navigation 2.0.11.
