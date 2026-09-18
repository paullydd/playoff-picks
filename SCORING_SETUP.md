# Playoff Picks — Scoring System (as built)

The Google Sheet behind this app already has a full scoring engine built in.
This documents how it actually works, so future changes don't have to be
reverse-engineered from formulas.

**Multiplier rule:** every round a player's team survives, that team's
*next* round of points doubles again — 1x (Wild Card) → 2x (Divisional) →
4x (Conference Championship) → 8x (Super Bowl). A team that loses stops
generating points entirely (it's out of the playoffs).

## Tabs

- **Picks** — one row per entrant, written by the Apps Script behind
  `index.html`. Columns: `name, QB, qbteam, RB1, rb1team, RB2, rb2team, WR1,
  wr1team, WR2, wrteam, WR3, wr3team, TE, teteam, submittedat, Round`.
- **RosterState** — one row per entrant per slot (7 rows per person).
  Computed from `Picks`. Columns: `Name, Position, Player, Team, StartRound,
  StartRoundIndex, Alive, Streak, Multiplier, RoundPoints, ScoreThisRound`.
  - `Alive` = TRUE unless the player's team was eliminated in an earlier
    round than `CurrentRound` (looked up from `TeamStatus`).
  - `Streak` = `CurrentRound!CurrentRoundIndex - StartRoundIndex` while
    alive, else 0 (i.e. rounds survived so far).
  - `Multiplier` = `2^Streak`.
  - `RoundPoints` = the player's raw PPR points for whatever round is
    currently active (looked up from `PlayerPointsByRound` via
    `CurrentRound`).
  - `ScoreThisRound` = `RoundPoints * Multiplier`.
- **NeedsRepick** — `=FILTER(RosterState!A:D, RosterState!G:G=FALSE)`.
  Auto-lists anyone whose player just got eliminated, so you know who needs
  a replacement pick.
- **RoundConfig** — static reference table: `Round, RoundIndex,
  BaseMultiplier` (WC=0/1, DIV=1/2, CONF=2/4, SB=3/8). Mirrors the `2^Streak`
  math for reference; not itself read by the live formulas.
- **CurrentRound** — the one cell you update each week: `CurrentRound`
  (dropdown: WC/DIV/CONF/SB) and `CurrentRoundIndex` (0–3). This is what
  advances the whole sheet to the next round.
- **TeamStatus** — manual entry after each round: `Team, EliminatedRound,
  EliminatedRoundIndex`. Leave a team's row blank while it's still alive;
  fill in the round it lost once it's out.
- **PlayerTeamMap** — static `Player, Team` lookup, derived from the drafted
  player pool.
- **NeededPlayers** — QA tab confirming every drafted player has a row in
  `PlayerRoundStats` (`Status` = "In PlayerStats" or blank if missing).
- **PlayerRoundStats** — manual entry per player per round: `Round, Player,
  PassYds, PassTD, Int, RushYds, RushTD, Rec, RecYds, RecTD, Fumbles,
  PPRPoints (formula), Key (formula)`.
  - `PPRPoints` = `PassYds/25 + PassTD*4 - Int*2 + RushYds/10 + RushTD*6 +
    Rec*1 + RecYds/10 + RecTD*6 - Fumbles*2`, rounded to 2 decimals.
  - `Key` = `Round & "|" & Player` — a lookup key used elsewhere.
- **PlayerPointsByRound** — one row per player, pivoted from
  `PlayerRoundStats`: `Player, WC_PPR, DIV_PPR, CONF_PPR, SB_PPR`.
- **Leaderboard** — final standings: `Rank, Medal, Name, Total PPR`, with
  medal emoji for the top 3.
- **PublicLeaderboard** — same as `Leaderboard`, a shareable copy (`Total`
  instead of `Total PPR`) that doesn't expose anyone's actual picks.

## Weekly workflow

1. After a round's games finish, go to `TeamStatus` and fill in
   `EliminatedRound`/`EliminatedRoundIndex` for any team that lost.
2. Add that round's raw stats for every player who played, as new rows in
   `PlayerRoundStats`.
3. Once you're ready to move to the next round, update `CurrentRound` /
   `CurrentRoundIndex` on the `CurrentRound` tab.
4. `RosterState`, `Leaderboard`, and `PublicLeaderboard` update themselves —
   no other steps needed.

## Known cleanup done

- `RosterState!L1:O10` had an orphaned, erroring `ARRAYFORMULA(QUERY(...))`
  left over from earlier iteration — cleared, it wasn't referenced by
  anything else.

## Known quirks (cosmetic, not functional)

- `Picks` column K is headered `wrteam` rather than `wr2team` — harmless,
  since every formula reads it by column letter, not header text.
