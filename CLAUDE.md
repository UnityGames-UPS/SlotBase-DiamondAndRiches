# Diamond and Riches (SL-DAR) — Research Handoff

Nov 18, 2026 · @Manas

## Status

Research is complete: every rule, payload field and Free Games behaviour has been checked against the paytable and real backend logs. Next is the Unity build.

- **Reference:** Orion Stars "Diamond and Riches". Paytable screenshots are the source of truth for rules; Gemini's video breakdown was only a starting point.
- **Backend:** game id SL-DAR, parse sheet + socket logs captured and verified.
- **Free Games reference code:** another game from the same template family (GameManager, UIManager, SlotView) shows the full Free Games flow.

### Still to do before building

- [ ] Revert the Scatter reelsInstance in the parse sheet to the original `{"0":1,"1":2,"2":2,"3":2,"4":1}` (it was raised to 10 on every reel for testing)
- [ ] Check the end-of-Free-Games moment in the reference video: the WIN box showed 7.50 while the popup showed 0.75 + 1.69
- [ ] Check whether the START button on the trigger popup waits for the player or auto-presses during autoplay
- [ ] Decide with the senior: build on the other game's template (already 5x3 with Free Games) or on Fortune Piggy's

## Game rules (paytable)

5 reels x 3 rows, 15 fixed paylines, line wins pay left to right from reel 1 on consecutive reels. Only the highest win per line pays; all line wins plus the Scatter win are added together.

- **Minimum match:** 3 of a kind for most symbols; 2 of a kind also pays for Car, Boat, 9 and Scatter.
- **Line pay** = paytable value x line bet. **Scatter pay** = paytable value x total bet (verified).
- **Wild (private jet):** substitutes for everything except Scatter, x1 (never doubles, unlike Fortune Piggy's 2xWild), no pay of its own.
- **Scatter (blonde man):** pays anywhere; 3 or more start Free Games (15 spins, x3 on all wins). 6 Scatters on screen pay as 5.
- Ignore the paytable's "15 WAYS TO WIN" footer: it is generic provider text and fits none of the 15 lines.

| Symbol (id)                | 5x  | 4x  | 3x | 2x |
|----------------------------|-----|-----|----|----|
| Car (1), Boat (2)          | 750 | 125 | 25 | 2  |
| Watch (3)                  | 500 | 100 | 20 |    |
| Diamond (4), Cash (5)      | 250 | 75  | 15 |    |
| A (6), K (7)               | 150 | 40  | 10 |    |
| Q (8)                      | 125 | 30  | 5  |    |
| J (9), 10 (10)             | 100 | 25  | 5  |    |
| 9 (11)                     | 100 | 25  | 5  | 2  |
| Scatter (12), x total bet  | 50  | 20  | 5  | 2  |

### Paylines

Row per reel 1-5; 0 = top, 1 = middle, 2 = bottom. The backend's lineIndex is 0-based, so line 1 = index 0.

| Line | Rows      | Line | Rows      | Line | Rows      |
|------|-----------|------|-----------|------|-----------|
| 1    | 1,1,1,1,1 | 6    | 1,0,0,0,1 | 11   | 1,2,1,0,1 |
| 2    | 0,0,0,0,0 | 7    | 1,2,2,2,1 | 12   | 0,1,1,1,0 |
| 3    | 2,2,2,2,2 | 8    | 0,0,1,2,2 | 13   | 2,1,1,1,2 |
| 4    | 0,1,2,1,0 | 9    | 2,2,1,0,0 | 14   | 0,1,0,1,0 |
| 5    | 2,1,0,1,2 | 10   | 1,0,1,2,1 | 15   | 2,1,2,1,2 |

Lines 10 and 11 originally differed in the backend (reel 4 was 0 and 2). The senior chose the paytable shapes; the backend fixed it on 2026-11-18 and the live init now matches.

## Backend: parse sheet and init

The parse sheet and the live game:init agree; bets are per line, so total bet = bet x 15.

- **Game:** SL-DAR, matrix x 5 / y 3, bets `[1, 2, 5, 10, 20, 50, 100]` per line, `platformJackpot: true`.
- **Symbols:** Wild (0) and Scatter (12) have `useWildSub: false`; Scatter is the only `isBonusSymbol`. Wild payouts is empty.
- **Free Games config:** `{enabled: true, retrigger: true, multiplier: 3, spinsCount: 15, maxFreeSpins: 15, triggerCount: 3, retriggerSpinsCount: 1}`. In practice retriggers never happen (the 15 cap blocks them); senior's call is to follow the backend.
- **game:init** carries `gameData {lines, bets, totalLines: 15}`, `features.freeGames`, `uiData.paylines.symbols[]` (multiplier ordered 5x down to 2x, description = group), `player.balance`, `jackpotData.values` (mini/minor/major/grand as strings). `jackpot:sync` events match Fortune Piggy.
- **Spin request:** `{type: "SPIN", payload: {betIndex, spins: 100}}`; `spins` is probably unnecessary, as on Fortune Piggy.

### Reel strip counts

reelsInstance, reels 1-5. Reel lengths 82 / 86 / 86 / 86 / 82.

| Symbol             | Counts                 |
|--------------------|------------------------|
| Wild               | 2 / 3 / 3 / 3 / 2      |
| Car, Boat          | 3 each reel            |
| Watch              | 4 each reel            |
| Diamond, Cash      | 5 each reel            |
| A, K               | 7 each reel            |
| Q                  | 8 each reel            |
| J                  | 9 / 10 / 10 / 10 / 9   |
| 10                 | 13 / 14 / 14 / 14 / 13 |
| 9                  | 15 each reel           |
| Scatter (original) | 1 / 2 / 2 / 2 / 1      |

Rough odds from these counts: 2 Scatters about 1 in 38 spins, Free Games about 1 in 640 spins. For testing, Scatter was set to 10 on every reel (about 1 in 5).

## Result payload and verified behaviour

The result shape differs from Fortune Piggy, so new data-model classes are needed; every win, Scatter pay and balance in the captured logs was checked by script and is correct.

```
{ success, id: "ResultData",
  matrix: [[...5 ids...], x3 rows],          // TOP-LEVEL, matrix[row][col], string ids
  payload: {
    betAmountPerLine, totalBetAmount,         // totalBetAmount = 0 on free spins
    winAmount, linesWinAmount, scatterWinAmount, grandTotalWin, netReturnRatio,
    lineWins: [{ lineIndex, positions: [{position:[row,col]}], pattern,
                 symbolId (string), symbolName, payout, matchCount }],
    scatterWin?: { count, payout, positions: [{position:[row,col]}] },   // only when a Scatter wins
    freeGames?: { triggered, retriggered, active, spinsRemaining,
                  totalSpins, totalWinCash, multiplier },             // only around Free Games
    isFreeSpinTriggered, isFreeSpinActive, activeLines, outBalance },
  player: { balance } }
```

| Behaviour                                                         | Verified by                                     |
|-------------------------------------------------------------------|-------------------------------------------------|
| Partial lines pay; positions lists only the matched cells, pattern the full line | 3x and 4x wins on the 295-win spin |
| Wild substitutes and its cells are included in positions          | Wild on reel 4 turned 3x 10 into 4x 10          |
| Line payout is a money amount, show it directly                   | 2x 9 at 2 per line paid payout: 4               |
| Scatter pays x total bet                                          | 2 Scatters = 30 at bet 15; 3 Scatters = 150 at bet 30 |
| winAmount = lines + scatter                                       | 8 + 30 = 38                                     |
| Balance = previous - total bet + win                              | Back-to-back spins, 6565.88 to 6588.88          |
| netReturnRatio = win / total bet                                  | 295 / 15 = 19.67                                |

Positions arrive as `[row, col]` objects; Fortune Piggy sent plain `[[row,col]]` and the reference SlotView expects a flat `row*5+col`.

## Free Games

The backend drives everything (counts, x3, end of round); the client only shows it, locks controls and runs the transitions.

### Backend behaviour (verified on a full 15-spin round)

- Trigger spin is a normal paid spin: `freeGames.triggered: true, active: false, spinsRemaining: 15`.
- Free spins cost 0, spinsRemaining counts 14 down to 0, x3 applies to line AND Scatter wins, and payout already includes the x3.
- `totalWinCash` = free-spin wins only (excludes the trigger spin's win).
- The last free spin already has `active: false`. Detect the end with `spinsRemaining == 0`, not `active`.
- Bet stays locked at the triggering bet. No retriggers happen.

### Target UI (reference screenshots)

1. **Base game:** "Diamonds And Riches" logo top centre; WIN box shows "GOOD LUCK !" when there is no win; SPIN says "HOLD FOR AUTO".
2. **Trigger:** Scatters light up, the rest dim; neon "15 FREE GAMES" popup over the reels; the spin button becomes START; WIN box shows the trigger win. Menu/settings and bet +/- grey out until the end.
3. **Fade in:** screen fades to black and back; the logo is replaced by a neon "FREE GAMES N" header (N = spins remaining); different frame and background art; spin button shows pink STOP; WIN box keeps the trigger win.
4. **End:** header "FREE GAMES 0"; popup with GAME WIN (trigger spin win), FEATURE WIN (totalWinCash) and TOTAL WIN; spin button becomes TAKE; then fade back to the base game.

### Reference code flow (other game, same template family)

1. After win animations, if `freeSpinData.isTriggered` and not already in Free Games: animate all Scatters (3.5 s), then `StartFreeSpins()` sets `isInFreeSpins`, blocks spins until the popup is closed, switches music, pauses autoplay and saves its remaining count.
2. `UIManager.OnFreeSpinsStarted` shows the popup (auto-closes after 5 s); closing it hides the logo, shows the counter and calls `StartFirstFreeSpin()`.
3. Each free spin: no balance check or deduction, Stop ignored, counter updated as soon as the server replies, next spin starts automatically after 0.3 s.
4. When the round is over: `EndFreeSpins()` shows the total popup, fades back, re-enables controls and resumes autoplay.

| Reference field       | SL-DAR field                                   |
|-----------------------|------------------------------------------------|
| isTriggered           | freeGames.triggered                            |
| spinsAwarded / total  | freeGames.totalSpins                           |
| serverSpinsRemaining  | freeGames.spinsRemaining                       |
| serverSpinsUsed       | totalSpins - spinsRemaining (not sent)         |
| serverTotalRoundWin   | freeGames.totalWinCash                         |
| isRoundOver           | isInFreeSpins && spinsRemaining == 0           |

### What must differ from the reference code

| Reference game              | Diamond and Riches                   |
|-----------------------------|--------------------------------------|
| Settings stays usable       | Disable settings too                 |
| Fade only at the end        | Fade at start and end                |
| "FREE GAME 3 OF 15"         | "FREE GAMES 13" (remaining)          |
| Take button on the popup    | START / TAKE on the spin button      |
| End popup shows the total only | Game Win + Feature Win + Total Win |
| WIN box resets to 0         | Keeps the trigger win                |
| Same frame                  | Swap frame and background            |

## Build notes

The core loop (sockets, spin flow, bet/balance, autoplay, turbo, jackpot) carries over from Fortune Piggy; the new work is the 5x3 grid, 15-line partial-win highlights, new data models and the Free Games mode.

| Area        | Fortune Piggy          | Diamond and Riches                           |
|-------------|------------------------|----------------------------------------------|
| Grid        | 3x3                    | 5x3                                          |
| Lines       | 3 straight             | 15 shaped, 2-5 in a row from reel 1          |
| Total bet   | bet x 3                | bet x 15                                     |
| Symbols     | 8, incl. Blank         | 13, no Blank                                 |
| Wild        | 2xWild, doubles        | x1                                           |
| Scatter     | Pig, pays by count     | Pays 2-5 x total bet + starts Free Games     |
| Group pays  | Any Bars, Any Sevens   | None                                         |
| Free Games  | None                   | 15 spins, x3                                 |

### Gotchas found in the reference code

- Nothing explicitly disables the bet buttons once Free Games start, and `IncreaseBet`/`DecreaseBet` only check idle and autoplay. Disable bet controls when Free Games start and add an `isInFreeSpins` guard in both methods.
- `GetTotalPay` uses `creditDivisor` with a default of 25; it must be 15.
- Reference SlotView is already 5x3 (7 images per reel, visible indices 2-4) and has two-phase win presentation (all wins, then line by line; phase 2 skipped in Free Games and autoplay). It does not draw the coloured payline, which the reference game does.
- Filler symbols use `Random.Range(0, 10)`; update for SL-DAR ids.
- Ignore the reference game's uSpin, moneyBag and wheel code.

## Logs and sources used

- Paytable screenshots (Wild/Scatter, symbol pays, Free Game, Win Lines, rules) and 5 Free Games UI screenshots
- Parse sheet SL-DAR + game:init, losing, winning, 2-Scatter, higher-bet and full Free Games result logs
- Reference game scripts: GameManager.cs, UIManager.cs, SlotView.cs

## Rules

- Minimal changes only
- Trace all references before deleting or renaming anything
- One small step at a time
- Never edit shared code without asking
