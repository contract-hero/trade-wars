# Phase 0 re-baseline: main vs the mechanics audit

| | |
|---|---|
| Ticket | [#43 Re-audit main against the Phase 0 punch list](https://github.com/alilloig/trade-wars/issues/43) |
| Date | 2026-10-19 |
| Code audited | `main` at `910ebe6` (`implement smart planet allocation with active frontier model`). Commit `c7724aee` named in the ticket is `910ebe6` plus the five `docs/` files (four HTML files and `docs/README.md`) and no Move change (`git diff --stat 910ebe6 c7724aee` touches only `docs/`). |
| Design docs | Not on `main`. They live on `feat/revisit-trade-wars-design` at `c7724aee`: [mechanics-audit.html](https://github.com/alilloig/trade-wars/blob/c7724aee4f485ba2e901c5f632f620e3846085df/docs/mechanics-audit.html), [architecture-spec.html](https://github.com/alilloig/trade-wars/blob/c7724aee4f485ba2e901c5f632f620e3846085df/docs/architecture-spec.html). |
| Toolchain | `sui 1.81.0-bf0c491c17b8`. `sui move build`: OK, 12 warnings. `sui move test`: `Total tests: 0; passed: 0; failed: 0`. |

All file:line references point at `move/sources/*.move` on `main` at `910ebe6`. The audit's line numbers point at the older code (`fd5ef25`), so they no longer match.

## 1. TW-001..TW-009 against main

| Item | Audit severity | Status on main | Where (main) | Reason or repro |
|---|---|---|---|---|
| TW-001 Thorium upgrades are free | Critical | **Fixed** | `planet.move:269-271` | `upgrade_thorium_mine` now burns all three costs (`erb_source.burn_resources`, `lan_source.burn_resources`, `tho_source.burn_resources`) before `upgrade_mine()` at `:272`. The three paths are now symmetric (`:209-211`, `:239-241`, `:269-271`). |
| TW-002 Cross-universe reserve theft | Critical | **Moot** | `element_mine.move:105-118` | Per-universe reservoirs no longer exist. `extract_element` takes `&mut ElementSource<T>`, and only one `ElementSource<T>` per element can exist: `create_source` is `public(package)` (`element_source.move:37`), its only callers are in `create_element_sources` (`trade_wars.move:265`, `:282`, `:299`), and each call does `Option::fill` (`trade_wars.move:277`, `:294`, `:311`), which aborts on a second call. One `TreasuryCap<T>` per element leaves `init` (`erbium.move:41`). The type parameter blocks cross-element substitution. The `source: ID` field (`element_mine.move:23`, written at `:50`) is still never checked; see N-3. |
| TW-003 Anyone mints into any reservoir | High | **Moot** | (deleted) | `refill_universe_source` and `universe_element_source.move` were deleted in `0f93fdc`. No refill path exists. The mint path is now N-1. |
| TW-004 `close_universe` re-opens | High | **Live** | `trade_wars.move:215` | `close_universe` ends with `universe.open_universe(creator_cap)`. `universe::close_universe` (`universe.move:153-156`) has zero callers (grep: definition only). Repro: creator calls `close_universe`; `Universe.open` stays `true`; `join_universe` only checks `universe.open()` (`overseer.move:59`), so joins by object ID continue. |
| TW-005 Public creation flag unenforced | Medium | **Live** | `trade_wars.move:144-178` | `public_start_universe` never reads `public_universe_creation`. The flag is set `false` at `:388` and `:406` and has no setter (grep: `:80`, `:97`, `:225-226`, `:388`, `:406` only). Default price is `0` (`:389`, `:407`). Bloat shape changed: each call now creates one shared `Universe`, `galaxies * systems` dynamic-field entries (`universe.move:305-321`), and one owned `UniverseCreatorCap`. |
| TW-006 Fees permanently locked | Medium | **Live** | `trade_wars.move:160-162` | Payment is joined whole into `universe_creation_fees` (`:162`); no change on overpayment. The field appears only at `:101`, `:162`, `:408`; no withdrawal function exists. |
| TW-007 Accrual bricks a mine | Medium (unverified) | **Moot as written; successor is N-1** | `element_mine.move:111` | The brick needed a reservoir-shortfall abort. `mint_for_extraction` (`element_source.move:50-55`) always succeeds, so no extraction can abort for lack of reserves. The unbounded accrual that caused it still exists and now equals unbounded minting. |
| TW-008 Div-by-zero on `production_factor = 0` | Low (unverified) | **Live, scope now global** | `element_mine.move:93` | `amount_produced` divides by `self.production_factor`. `set_mines_parameters<T>` (`trade_wars.move:361-378`) and `create_mine_configuration_parameters` (`mine_configuration_parameters.move:33-45`) accept `0`. After one extraction syncs the value (`element_mine.move:116`, `:135`), every view (`planet.move:73-88`) and upgrade on that planet aborts. New mines copy the value at creation (`element_mine.move:53`). One global source means one bad value reaches every planet in every universe. |
| TW-009 No bounds on universe dimensions | Low (unverified) | **Live, shape changed** | `universe.move:91-105`, `:302-328` | No validation in `create_universe_info` or `start_universe` (`trade_wars.move:415-451`). `galaxies = 0`: first join aborts `EUniverseFull` (`universe.move:258`). `systems = 0`: table has no key `0`; `borrow_mut` aborts (`:261`). `planets = 0`: `pop_back` on an empty vector aborts (`:265`); the division by `total` at `:284` is unreachable because `:265` aborts first. Large dimensions: creation eagerly adds `galaxies * systems` dynamic fields in one transaction (`:305-321`), which hits Sui's per-transaction new-object limit for large products; the old `255^3` single-vector limit failure became a `255^2` object-count limit failure. |

## 2. Defects introduced by mint-on-demand and the reservoir removal

### N-1 Unbounded minting (new, high)

- `mint_for_extraction` (`element_source.move:50-55`) mints any `amount` from the `TreasuryCap`. No field on `ElementSource<T>` (`element_source.move:19-23`) bounds it: no `max_reserves`, no drip rate, no epoch budget.
- The only caller is `extract_element` (`element_mine.move:111`), which passes `amount_produced(now)` = `(now - last_extraction_time) / 1000 * level / production_factor` (`element_mine.move:92-93`). This grows without limit in idle time and in level.
- The only remaining bound is the `u64` supply overflow abort inside `balance::increase_supply`.
- Repro: let a planet idle for `T` seconds, then call any `upgrade_*_planet_mine`; the three mines mint `T * level / 5` units each at default parameters (`trade_wars.move:39`, `:47`, `:55`).
- Relation to the spec: the spec's emission model is a permissionless lazy drip with `max_reserves`, and upgrade costs recycle into the reservoir (spec, "Sinks & emission"; module map row `element_source, universe_element_source`). Main burns upgrade costs (`planet.move:209-211`, `:239-241`, `:269-271`) and has no drip. See D-1 and D-2.

### N-2 Parameter sync lags, and a production decrease can be avoided (new, medium)

Mines cache `production_factor` and the three upgrade costs (`element_mine.move:28-35`) and re-sync from the global source only in `update_parameters` (`:134-139`).

- (a) `upgrade_*_mine` reads the costs from the cache at `planet.move:195-197` (and `:225-227`, `:255-257`) before any extraction syncs the cache at `:200-206`. The current upgrade always pays the pre-sync price; a parameter change applies one upgrade late per mine.
- (b) Extraction runs only when `store < cost` (`planet.move:199`, `:202`, `:205`). A planet whose stores already cover the cost upgrades without any sync.
- (c) `use_element` (`element_mine.move:121-130`), the sync-on-burn path, has zero callers (grep: definition only). The upgrade paths call `source.burn_resources` directly (`planet.move:209-211`), so a burn never syncs.
- (d) `extract_element` computes `amount_produced` with the cached factor at `:111` and syncs at `:116`. A player who withholds extraction accrues at the rate before the decrease for the whole idle period, and N-1 removes the cap on that amount. With the old reservoirs the damage was bounded by the reservoir size.

### N-3 Source-ID binding is dead code (new, low; latent)

`ElementMine.source: ID` (`element_mine.move:23`) is written once at `:50` and never read. Today the construction proof in TW-002 makes a second source impossible. A package upgrade that adds another `ElementSource<T>` creation path would reopen the TW-002 hole with no assert to catch it. Cheap fix: assert `object::id(source) == self.source` in `extract_element`, or delete the field.

### N-4 Universe creation takes three unused shared objects (new, low)

`start_universe` declares `_erb_source`, `_lan_source`, `_tho_source` (`trade_wars.move:418-420`) and never uses them. Both entry points still require them (`public_start_universe` `:147-149`, `admin_start_universe` `:321-323`), so every universe creation adds the three global sources as transaction inputs without a use.

### Positive side effect

With reservoirs gone, `has_enough_resources` (`planet.move:341-351`) and the extraction at `:200-206` agree: both use `store + amount_produced(now)`, and extraction cannot abort on a shortfall. The audit's simulate-vs-execute gap for upgrades is closed.

### Carry-overs that are not new

- Truncation: `amount_produced` floors twice (`element_mine.move:92-93`) and `extract_element` resets `last_extraction_time` to `now` (`:113`), so the sub-second and sub-factor remainders are lost on every extraction. At level 1 and factor 5, an extraction every 4 seconds yields 0 forever.
- Decimals: economy constants are raw units (`trade_wars.move:39-61`) while all three coins have 9 decimals (`erbium.move:29`, `lanthanum.move:29`, `thorium.move:29`).
- `Move.toml:7` pins `rev = "framework/testnet"`, a moving reference.
- `calculate_travel_distance` is a stub returning 0 (`planet.move:140-143`); `system_size` is written (`planet.move:172`) and never read.
- Two events in the package, both creation-time (`trade_wars.move:130`, `universe.move:128`).
- `coin::create_currency` is deprecated in this toolchain (build warning at `erbium.move:27`, `lanthanum.move:27`, `thorium.move:27`); any redeploy for the decimals decision should use `coin_registry::new_currency_with_otw`.

## 3. What the frontier allocation changed relative to the spec

### What `910ebe6` changed (diff `fd5ef25..910ebe6`, `universe.move`)

- `free_planets: vector<PlanetInfo>` with an O(g*s*p) shuffle became `system_free_planets: Table<u16, vector<u8>>` plus `active_galaxy` / `active_system` (`universe.move:61-66`).
- `join_universe` now gets a random free position inside the active system only (`universe.move:256-274`: assert `:258`, borrow `:261`, shuffle `:264`, pop `:265`).
- After each allocation, the frontier advances to the next system when occupancy reaches `FRONTIER_THRESHOLD_PERCENT = 50` (`:29`, `:277-289`), and advances to the next galaxy at the end of a galaxy (`:292-299`).
- `Universe` lost `store` and is shared inside `create_universe` (`:54`, `:133-135`); `create_universe` returns `(ID, UniverseCreatorCap)` instead of the object (`:115`).

### Against the spec's coordinate->planet index decision

The spec asks for `occupied: Table<u32, ID>`, a `planet_at(coords)` view, and occupation events (spec, "Agentic-readiness layer: Discoverability"; module map row `universe`: "coordinate->planet index; O(1) occupy").

| Spec | Main |
|---|---|
| `occupied: Table<u32, ID>` (coordinate -> Planet ID) | `system_free_planets: Table<u16, vector<u8>>` tracks free positions only. No coordinate -> Planet ID mapping anywhere. |
| `planet_at(coords)` view | Absent. |
| `PlanetOccupied` event | Absent (`event::emit` only at `universe.move:128`). |
| `universe: ID` on `Planet` | Absent (`planet.move:45-63`). `PlanetCap` carries only the planet ID (`:29-32`). |
| u32 key | u16 key via `system_key = galaxy * systems + system` (`universe.move:251-253`). Max value `254 * 255 + 254 = 65024`, so it fits. |
| O(1) occupy | O(p) shuffle on a vector of at most 255 `u8` (`:264`). Bounded, not O(1). |
| Retire eager materialization | Still eager: `galaxies * systems` table entries at creation (`:305-325`). See TW-009. |

### Against the spec's colonize-by-coordinates decision

The spec decides "Colonization: choose free coordinates; cost = colony-ship + NOVA fee proportional to distance; `abandon_planet` returns coords" (spec decision ledger, "Colonization"; "Ships, research, colonies").

- Compatible: a free-coordinate check is `system_free_planets[key].contains(pos)` and removal is `vector::remove` (both O(p), p <= 255). The frontier leaves `planets - ceil(planets / 2)` positions in every passed system, which is the colonization reserve that `CLAUDE.md` Phase 2 asked for ("Systems are NOT fully filled").
- Gaps: no colonize entry exists; no path returns a coordinate (an `abandon` would `push_back` into the system vector, but the frontier never rewinds, so returned positions in passed systems are reachable only by a future colonize); no coordinate -> Planet index for raid targeting.
- Consequence for join capacity: a universe accepts `galaxies * systems * ceil(planets / 2)` joins, then `get_free_planet` aborts `EUniverseFull` (`:258`). The other half of every system is unreachable until a colonize path exists.
- Provenance: the frontier rule comes from `CLAUDE.md` "Phase 2: Smart Planet Allocation", not from the spec's decision ledger. The spec has no clustering decision. Treat the 50% threshold and the galaxy-major order as de facto decisions to confirm (D-4).

## 4. State of the test suite

- `move/tests/trade_wars_tests.move` is 6 lines: `#[test_only] module`, three `use` lines, no test function.
- `sui move test` (sui 1.81.0): `Test result: OK. Total tests: 0; passed: 0; failed: 0`.
- `sui move build`: OK with 12 warnings: 3x deprecated `coin::create_currency` (`erbium.move:27`, `lanthanum.move:27`, `thorium.move:27`); 4x deprecated `vector::empty` (`universe.move:312`, `overseer.move:65`, `:166`, `trade_wars.move:387`); 2x unused parameter in `calculate_travel_distance` (`planet.move:140`); 3x unused function `has_enough_erbium` / `has_enough_lanthanum` / `has_enough_thorium` (`planet.move:355`, `:360`, `:365`). `use_element` (`element_mine.move:121`) is also unused but is `public(package)`, so the compiler does not report it.
- The build also notes that an explicit `Sui` dependency disables automatic framework dependencies (`Move.toml:7`).

## 5. Decisions for the owner (not findings)

Main diverges from the locked spec in three places. A research note presents the fork; it does not pick.

| # | Fork | Spec says | Main does | Options |
|---|---|---|---|---|
| D-1 | Emission bound | Permissionless lazy drip with `max_reserves`; net inflation = drip - burn | Unbounded `mint_for_extraction` (N-1) | (a) Reinstate a bounded emission on the global source (drip rate + cap per element). (b) Move the spec's Phase 1 storage cap into Phase 0: clamp `amount_produced` to `capacity - stored` per planet. (c) Accept unbounded minting until Phase 1 and record it. |
| D-2 | Upgrade cost disposal | Mine and storage upgrades recycle into the reservoir | Burn via `burn_resources` (`planet.move:209-211`) | (a) Keep burn and amend the spec's two-realm diagram and "Sinks & emission". (b) Recycle into a global reserve that the drip draws from first. |
| D-3 | Production tuning scope | "Global supply, per-universe production" (decision ledger, "Element supply"; audit decision 2) | One `mine_parameters` per `ElementSource<T>` (`element_source.move:22`), set by `set_mines_parameters` (`trade_wars.move:361-378`): global supply and global tuning | (a) Accept global tuning and amend the ledger. (b) Add a per-universe parameter overlay that mines read on sync. |
| D-4 | Allocation rule | No clustering decision in the ledger | Frontier at 50% per system, galaxy-major order (`universe.move:29`, `:277-299`) | (a) Confirm as a decision; expose the threshold behind `GameAdminCap`. (b) Revert to the spec's index-only model and leave clustering to Phase 3. |
| D-5 | Coordinate index shape | `occupied: Table<u32, ID>` + `planet_at` | Free-position vectors per system, no ID map | (a) Add `occupied` beside `system_free_planets` in Phase 1. (b) Replace `system_free_planets` with `occupied` and derive free positions by lookup. |

## 6. Re-baselined Phase 0 punch list (in order)

1. **TW-004.** `trade_wars.move:215`: call `universe.close_universe(creator_cap)`. Test: close, then `join_universe` aborts `EUniverseNotOpen` (`overseer.move:21`, `:59`).
2. **TW-005.** Assert `self.public_universe_creation` in `public_start_universe` (`trade_wars.move:144-178`); add an admin setter that updates both `TradeWars` and `TradeWarsInfo` (`:97`, `:80`). Test: public creation aborts while the flag is `false`.
3. **TW-006.** Add an admin `withdraw_universe_creation_fees`; split exact payment or return change at `:160-162`. Test: overpayment returns change; withdrawal moves the balance.
4. **TW-008.** Validate `production_factor > 0` in `create_mine_configuration_parameters` (`mine_configuration_parameters.move:33-45`) with a game error. Test: `set_mines_parameters(0, ...)` aborts.
5. **TW-009.** Validate `galaxies >= 1`, `systems >= 1`, `planets >= 1`, and a tested upper bound on `galaxies * systems` in `create_universe_info` (`universe.move:91-105`) or `start_universe` (`trade_wars.move:415-451`). Test: each zero dimension aborts with the game error; the largest allowed universe creates.
6. **Decide D-1, D-2, D-3.** These set the shape of the N-1 and N-2 fixes. Do not implement an emission bound before the decision.
7. **N-1 / N-2 after the decision.** Implement the chosen emission bound. Make upgrades read costs after sync: sync all three mines from their sources at the top of `upgrade_*_mine`, or route burns through `use_element` (`element_mine.move:121-130`) and delete the direct `burn_resources` calls. Test: a parameter change applies to the next upgrade; an idle planet cannot mint past the bound.
8. **Invariants.** Assert `object::id(source) == self.source` in `extract_element` (`element_mine.move:105-118`) or delete the field (N-3). Add `universe: ID` to `Planet` (`planet.move:45-63`) for discoverability and colonize. Drop the spec's `source.universe == planet.universe` and "bind refill caps" items: both are moot without reservoirs.
9. **Decimals.** Rescale `trade_wars.move:39-61` by `1e9`, or redeploy at 0 decimals with `coin_registry::new_currency_with_otw` (the deprecation at `erbium.move:27`).
10. **Pin `Move.toml:7`** to a tagged framework revision.
11. **Test suite.** Minimum set: join and occupy; frontier advances at 50% and moves to the next galaxy; `EUniverseFull` after `g * s * ceil(p / 2)` joins; each of the three upgrades burns the exact cost and the total supply decreases; close/open round trip; parameter validation; dimension validation; extraction truncation at small intervals.
12. **Cleanup.** Delete `has_enough_erbium` / `has_enough_lanthanum` / `has_enough_thorium` (`planet.move:355-367`) and the unused source parameters of `start_universe` (`trade_wars.move:418-420`, `:147-149`, `:321-323`); replace `vector::empty` with `vector[]`.

Phase 1 items the frontier work touches but does not finish: coordinate -> Planet index, `planet_at`, occupation events, storage caps, remainder-carry accrual.

## Method

Primary source: `move/sources/*.move` and `move/tests/trade_wars_tests.move` on `main` at `910ebe6`, read in full; the diff `fd5ef25..910ebe6` for `universe.move`, `element_mine.move`, `element_source.move`, `planet.move`; the file stats of commits `0f93fdc` and `910ebe6`; grep for callers of `close_universe`, `use_element`, `universe_creation_fees`, `public_universe_creation`, `create_source`, `event::emit`, `system_size`. Design text: `docs/mechanics-audit.html` and `docs/architecture-spec.html` at `c7724aee`, tags stripped. Toolchain output: `sui move build` and `sui move test` run in `move/` on 2026-10-19.
