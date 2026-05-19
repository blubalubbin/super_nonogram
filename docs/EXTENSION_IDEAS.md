# Extension Ideas

A menu of realistic next directions for Super Nonogram, with rough
effort sizing, who'd care, and which files would change. The final
section picks one as a recommended first step and sketches the
implementation.

For background on the existing architecture, see
[`ARCHITECTURE.md`](./ARCHITECTURE.md).

## Sizing

- **S** — a focused weekend; one or two new files, no schema work.
- **M** — touches several existing files and adds new tests; might need
  a small data-model change.
- **L** — meaningful refactor or new infra (cloud, multi-color rendering,
  etc.).

## Candidate ideas

### 1. Daily puzzle + streak  *(S)*
A "Daily" entry on the home screen. Today's seed is derived from the
date (`yyyymmdd`), so every player gets the same board, and a streak
counter persists across days. Builds entirely on top of the existing
`ClassicGameMode` seed plumbing in `lib/api/classic_puzzles.dart` and
the `stows` persistence in `lib/data/stows.dart`. Touches:
`lib/data/game_mode.dart`, `lib/data/stows.dart`, `lib/pages/title_page.dart`,
`lib/main.dart` (new route).

### 2. Hint system  *(S–M)*
Let the user spend a hint to reveal one correct cell, one full row, or
one full column. Rate-limit hints with a daily cap or per-puzzle count.
Touches `lib/board/board.dart` (the live state) and reads the answer
labels from `lib/board/board_labels.dart`. Persist the hint budget via
`stows`.

### 3. Undo / redo  *(M)*
Wrap every tile mutation in `lib/board/board.dart` with a command
object, keep a bounded history list, expose ⌘Z / ⌘⇧Z and on-screen
buttons. Useful for pan-fill mistakes. No data model changes — purely
in-memory.

### 4. Timer + best-time leaderboard  *(M)*
Time each solve and push the result to a `games_services` leaderboard.
Currently only achievements are wired; leaderboards would mean adding
leaderboard IDs to `lib/games_services/achievement_ids.dart` and
plumbing them through `games_services_helper.dart`. A local "best time
per level" can land first without the cloud step.

### 5. Custom puzzle editor + share codes  *(M)*
A second mode on the board widget where tapping authors the answer
instead of solving. Serialise via the existing `.ngb` format
(`lib/board/ngb.dart`), then encode to a short Base64 string that
deep-links into the app (`/play?code=…`). Adds an `Editor` page and one
new GoRouter route. Great for community shareables.

### 6. Colour nonograms  *(L)*
Real picross variants support multiple colours per cell. Today,
`TileState` is `empty | selected | crossed` (`lib/board/tile_state.dart`),
so this is a meaningful refactor: change the state to a colour index,
update `board_labels.dart` to emit per-colour runs, and teach the
renderers in `lib/board/render_objects/` to draw colour swatches in the
clues. Worth it eventually but expensive to land.

### 7. Sound effects  *(S)*
There's already a TODO in `lib/util/sonic_controller.dart` for audio.
Bundle a small set of click / win / cross sound assets, add an
`audioEnabled` stow next to `hyperlegibleFont` in `lib/data/stows.dart`,
and trigger them from the same call sites as the existing haptics.

### 8. Pluggable image sources  *(M)*
Right now Pixabay is hard-wired in `lib/api/api.dart`. Extract an
`ImageSource` interface, keep the Pixabay implementation, and add
alternatives — Unsplash, Openverse, or a "pick from photo library"
source backed by Flutter's `image_picker`. Pays off the existing
single-source coupling.

### 9. Stats screen  *(S)*
A new page that reads from `stows` (and any new counters we add) to
show puzzles solved, average solve time, daily-puzzle streak, fastest
solve per board size, etc. Pure read-side; no game logic changes.

### 10. Cloud save sync  *(L)*
Sync `stows` state across devices via iCloud Key-Value Store on iOS and
Drive App Data on Android, with conflict resolution by `updatedAt`
timestamps. The biggest infra lift — needs entitlements, new packages,
and a careful merge strategy. Probably only worth it after Stats and
Daily exist to give the sync something to carry.

## Recommended first extension: **Daily puzzle + streak**

Why this one first:

- **Smallest blast radius.** It piggy-backs on `ClassicGameMode`'s
  existing seeded generator — no new puzzle logic.
- **Reuses existing persistence.** Streak counters fit straight into
  `lib/data/stows.dart` next to `currentLevel`.
- **High retention payoff.** Daily puzzles are the standard pattern for
  bringing players back tomorrow (NYT Connections, Wordle, etc.).
- **Unlocks downstream features.** Once a `Daily` route and streak
  state exist, the Stats screen, hint budget, and leaderboard ideas
  all have something concrete to hang off.

### Implementation sketch

**1. Add the game mode.** In `lib/data/game_mode.dart`, either add a
`DailyGameMode` variant or simply construct `ClassicGameMode` with a
date-derived seed at the call site. Inline is fine for v1:

```dart
int dailySeed(DateTime date) =>
    date.year * 10000 + date.month * 100 + date.day;
```

**2. Add persisted state.** In `lib/data/stows.dart`, add three new
stows alongside `currentLevel`:

- `dailyStreak: int` — current consecutive-day streak.
- `lastDailySolvedDate: String` (ISO `yyyy-MM-dd`) — the date of the
  last solved daily, used to detect break / continue / already-solved.
- `dailyBestStreak: int` — for the eventual Stats screen.

**3. Wire the route.** In `lib/main.dart`, add:

```dart
GoRoute(
  path: '/daily',
  builder: (_, __) => PlayPage(ClassicGameMode(dailySeed(DateTime.now()))),
),
```

**4. UI entry point.** Add a prominent "Today's puzzle" button on
`lib/pages/title_page.dart` that shows the current streak and whether
today is already done.

**5. Solve hook.** On the win path inside `PlayPage` (or the existing
solved callback in `lib/board/board.dart`), check whether the active
mode is the daily and:

- If `lastDailySolvedDate` is yesterday → `dailyStreak += 1`.
- If it's any earlier date → `dailyStreak = 1`.
- If it's already today → no-op (replays don't double-count).
- Update `dailyBestStreak` if appropriate.

**6. Tests.** Mirror `test/level_to_board_test.dart`: assert that
`dailySeed(date)` is deterministic, that the same date produces the
same board through the existing classic generator, and that the streak
state machine handles the three transition cases.

**7. Out of scope for v1.** No notifications, no cloud sync, no
calendar UI — those can come later. Ship the loop first.
