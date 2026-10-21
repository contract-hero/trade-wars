# Trade Wars

An agentic trade game on Sui. Players run planets that mine elements, trade or raid for the elements they lack, and sell surplus to the Company for NOVA. This glossary grows one wayfinder ticket at a time; terms below were fixed by the emission-model and markets-view decisions of 2026-10-20.

## Language

### Elements and production

**Element**:
One of the in-game substances (Erbium, Lanthanum, Thorium) that mines produce and construction consumes.
_Avoid_: resource, token (for the substance itself)

**Mine**:
The part of a planet that produces one element at a rate set by its level.

**Production**:
The amount of an element a mine generates per unit of time.

**Extraction**:
The settlement of a mine's pending production into the planet's store. New element supply comes into existence only here.

**Emission**:
The creation of new element supply. It happens only at extraction and is bounded per planet by the storage cap.
_Avoid_: drip, minting (as a game concept)

**Idle accrual**:
Production a mine has generated that the planet has not extracted yet. It can never exceed the storage cap.

**Store**:
A planet's on-hand holding of one element.

**Storage cap**:
The most a planet's store can hold of one element. Production halts at the cap. It is the element emission bound.
_Avoid_: reservoir, max reserves

**Storage level**:
A planet's per-element storage tier, which sets its storage cap.

**Cumulative production**:
The total an element mine has ever extracted. Together with planet age it weights a planet's share of NOVA emission and its raid immunity.

**Planet age**:
The time since a planet was created.

**Burn**:
Permanent destruction of elements. Every spend on a planet and every sale to the Company burns.
_Avoid_: recycle

### Tuning

**Mine parameters**:
The global per-element base settings for production and upgrade costs, shared by every universe.

**Speed multiplier**:
A per-universe factor fixed at creation that scales mine production for every planet in that universe. Storage caps are not scaled.
_Avoid_: economy speed, server speed

### Markets

**Markets view**:
The client panel that shows Company prices, the public NOVA/SUI market and the player's balances, and hosts the swap.

**Company price**:
The NOVA the Company pays per unit of an element at this moment, set by its controller.
_Avoid_: exchange rate, market price (for the Company's side)

**Cash out**:
A player's swap of NOVA for SUI on the public NOVA/SUI market.
_Avoid_: withdraw, sell NOVA

**Buy in**:
A player's swap of SUI for NOVA on the public market, to accelerate play.
_Avoid_: deposit, top up

**Human gate**:
An action the player must take in person. The agent may recommend it and can never execute it. Cash out and buy in sit behind it.

## Retired terms

**Drip** (retired 2026-10-20):
The former permissionless refill of per-universe reservoirs. There is no drip; the storage cap bounds emission.

**Reservoir** (retired 2026-10-20):
The former per-universe element pool. Elements exist only in planet stores and wallets.

**Cash-out cut** (retired 2026-10-20):
The former treasury cut on a cash out. There is none: NOVA is a plain coin on a public market, so no cut can be enforced.
