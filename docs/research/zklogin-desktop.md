# zkLogin for a desktop app: Enoki vs self-hosted, OAuth redirect, session lifecycle, sponsored gas

Research note for issue #46. Written 2026-10-19. Every claim carries a source tag `[Sn]`; the
`Sources` section lists the URL and the date it was read. Claims I could not confirm against a
primary source are marked **UNVERIFIED** inline and collected again at the end.

Scope: a TypeScript desktop app (Electron or Tauri) for Trade Wars that hides the chain from the
player: Google login through zkLogin, gas paid for the player, and an escape hatch so a player can
connect Slush instead.

---

## 0. Recommendation (TL;DR)

**Use Enoki for zkLogin (salt + prover), but do not use the stock `registerEnokiWallets` wallet in
the desktop app. Drive the OAuth flow yourself through the system browser with a loopback redirect,
and call Enoki's HTTP API through `EnokiClient` (`@mysten/enoki/client`). Sponsor gas through a
small backend that holds an Enoki private key. Keep self-hosting (prover + salt service + gas pool)
as the documented exit, not the starting point.**

Why, in one paragraph. Enoki's Sandbox tier is free for development; the first plan that lists
*live* apps is Starter at $69/month ("Unlimited live apps on Dev/Testnet", "Up to 3 live apps on
Mainnet"), and the FAQ describes Sandbox as "no live applications" [S13][S14] — so the price is
free until the game goes live, then $69/month (what "live" means for a Testnet-only hobby
deployment is **UNVERIFIED**, see U2); its managed salt removes the single most dangerous
operational duty of a self-hosted setup (a lost salt is a permanently lost address [S1][S2]); its hosted prover is the
only hosted prover that accepts your own Google client ID on Mainnet [S2][S44]; and the self-hosted
prover needs a 16-core / 16 GB machine [S2]. The trade-offs that the custody ticket must carry:
Enoki's wallet code is browser-only (popup + implicit `id_token` flow) [S22]; Google refuses OAuth
inside embedded webviews [S32][S33] and treats desktop apps as public clients that must use the
authorization-code flow with PKCE and a loopback redirect [S29]; sponsorship always needs a
private key, so a backend exists either way [S17][S14]; and the salt is per Enoki app and not
exported to users, so leaving Enoki later means new addresses for every player [S14] (the API does
return `salt`, see 1.4).

---

## 1. Enoki vs self-hosted salt service + prover

### 1.1 What each side provides

| Concern | Enoki (managed) | Self-hosted |
| --- | --- | --- |
| Salt | Managed, derived, per app; "Enoki does not currently expose the salt to users" [S14]. `GET /v1/zklogin` returns `salt`, `address`, `publicKey` for a JWT [S16][S25]. | You run it: DB mapping `sub -> salt`, or HKDF from a master seed; master seed cannot rotate and `aud` cannot change without changing addresses [S2]. Must be HTTPS with CORS limited to your origins, with backups [S2]. |
| Prover | `POST /v1/zklogin/zkp` with the JWT in a `zklogin-jwt` header [S16][S24]. Rate limit: 2 proof requests per minute per zkLogin user [S14]. | Docker: `mysten/zklogin:prover-stable` + `prover-fe-stable`; zkey `zkLogin-main.zkey` for Mainnet/Testnet, `zkLogin-test.zkey` for Devnet; "at least 16 cores and 16 GB RAM"; default timeout 15 s [S2]. About 3 s per proof on 16 vCPU / 64 GB [S2]. |
| Mysten hosted prover without Enoki | n/a | Devnet/Testnet: "public prover endpoint is available without registration" [S2]. The official demo calls `https://prover-dev.mystenlabs.com/v1` ("a public development prover") with `VITE_NETWORK=testnet` [S5]; which networks that endpoint serves is otherwise undocumented (U12). Mainnet: "requires allowlisting through the Enoki platform" [S2]. A forum report (2026-05-17) says `https://prover.mystenlabs.com/v1` on Testnet rejects non-allowlisted Google client IDs ("The audience ... is not supported") — **UNVERIFIED** user report, no staff reply [S44]. |
| Nonce / maxEpoch | `POST /v1/zklogin/nonce` returns `nonce`, `randomness`, `epoch`, `maxEpoch`, `estimatedExpiration`; `additionalEpochs` default 2, range 0–30 [S16]. | `generateNonce(pk, maxEpoch, randomness)` from `@mysten/sui/zklogin` [S2]. |
| Signing | `EnokiKeypair` wraps ephemeral key + proof [S22]; or `ZkLoginSigner` from `@mysten/sui/zklogin` [S2][S6]. | `ZkLoginSigner` / `getZkLoginSignature` [S2]. |
| Sponsorship | `POST /v1/transaction-blocks/sponsor` then `/sponsor/{digest}`; private key required [S16][S17]. | `MystenLabs/sui-gas-pool` (Rust + Redis) [S42], Shinami [S43], or hand-rolled per [S7]. |
| Networks | `EnokiNetwork = 'mainnet' | 'testnet' | 'devnet'`; API default is `mainnet`, so pass `network: 'testnet'` explicitly [S16][S24]. | Any network you run against. |

### 1.2 Pricing (Enoki, read 2026-10-19) [S13]

| Plan | Price | zkLogin | Sponsored transactions | Live apps |
| --- | --- | --- | --- | --- |
| Sandbox | Free | "Unlimited standard ID providers", 3 seats | not listed | not listed; FAQ: "returned to the Sandbox plan (no live applications)" [S14] |
| Starter | $69/month | 7.5K zkLogin MAU | not listed | "Unlimited live apps on Dev/Testnet", "Up to 3 live apps on Mainnet" |
| Professional | $120/month | 10K zkLogin MAU | 100K sponsored transactions, budget "up to $100 SUI equivalent" | up to 5 on Mainnet (page is inconsistent, also says unlimited) |
| Sponsored bundle | from $100 | — | up to 400,000 transactions, $0.001/transaction overage | — |
| Enterprise | custom | unlimited | included | unlimited |

Credits: "Credits purchased do not expire so long as an active subscription remains in place
(including the Free tier)"; a provisioning fee is "only charged when usage exceeds this allowance"
[S14]. **UNVERIFIED:** whether the free Sandbox plan can sponsor any transactions on Testnet, and
whether a Testnet-only deployment counts as a "live" app that needs Starter; the pricing page
lists no sponsored allowance for Sandbox or Starter [S13].

Rate limits [S14]: public API keys 2 requests/s per IP; private keys 1,000 requests/s per IP;
proof generation 2/min per zkLogin user; sponsored-transaction creation 500/s per team.

### 1.3 Self-hosted cost shape

No price is published; it is your infrastructure. The docs fix the floor: a 16-core / 16 GB
prover box [S2], an always-on HTTPS salt service with backups and CORS [S2], and, if you sponsor,
a gas pool with a Redis instance and a dedicated funded sponsor address [S42][S7]. The prover
request format is identical hosted vs self-hosted, "so you can switch between hosted and
self-hosted without changing your client code" [S2]. Self-hosting also "avoid[s] the hosted
prover allowlisting process" on Mainnet [S2].

### 1.4 How secrets survive inside a distributed desktop binary

- **Enoki public API key.** Designed to ship in client code: "Public keys are meant to be used
  when calling Enoki from a frontend or used in publicly available code" [S15]. Abuse is bounded
  by the public-key rate limit (2 rps/IP) [S14], by the app's `allowedOrigins` (returned by
  `GET /v1/app`) [S16], and by the fact that a JWT with your `aud` is required for `/zklogin`,
  `/zklogin/zkp` [S16]. A public key cannot sponsor: "Sponsoring transactions requires using
  private API keys" [S17]. **UNVERIFIED:** whether Enoki accepts public-key requests that carry no
  browser `Origin` header (Electron main process, Tauri Rust side, or a `file://` renderer). This is
  the first thing the custody ticket must spike; if Enoki rejects them, even login needs a tiny
  proxy.
- **Enoki private API key.** Backend only; "it is your responsibility as the developer to ensure
  that the endpoint sponsoring transactions is appropriately access restricted" [S14].
- **Google OAuth client.** Use the **Desktop app** client type: "Desktop apps are public clients
  and cannot keep secrets confidential"; "Always use PKCE (Proof Key for Code Exchange) when
  requesting authorization codes"; `client_secret` is optional at the token endpoint [S29]. The
  client ID is public by design; it is also an input to the address (`aud`) [S1].
- **Enoki salt lock-in.** One wallet per `clientID`; if the `clientID` "changes, users can no
  longer access their wallets or the assets inside them" [S14]. The `GET /v1/zklogin` response
  does include `salt` [S16] and the deprecated `EnokiFlow` reads it [S25], so a developer-side
  export for migration is technically reachable. **UNVERIFIED:** whether Enoki's terms permit that
  and whether `salt` is always populated. Enoki Connect does not change this: it lets other apps
  use an Enoki app's accounts through a hosted callback on `*.connect.enoki.mystenlabs.com`,
  "Custom zkLogin configurations are not supported" — portability inside Enoki only [S21].

---

## 2. The OAuth redirect flow in a desktop app

### 2.1 What Google allows for desktop

- Client type: "Select the Desktop app application type" [S29].
- Redirect: "Loopback IP address (macOS, Linux, Windows desktop)" (`http://127.0.0.1:port` or
  `http://[::1]:port`) — "this is the recommended mechanism for obtaining the authorization code";
  "start an HTTP listener on a random available port"; `localhost` is allowed but "may cause issues
  with client firewalls" [S29].
- Custom URI scheme: "Custom URI schemes are no longer supported due to the risk of app
  impersonation" [S29]. The page's explicit platform note names Android and Chrome apps; the
  sentence above is unqualified. Treat custom schemes (`tradewars://`) as unavailable for the
  Google step and keep loopback.
- Manual copy/paste (OOB): "no longer supported" [S29].
- Flow: "Set the parameter value to `code` for installed applications" [S29]. The implicit flow
  (`response_type=id_token`) is documented for "a client-side application (typically a JavaScript
  app running in the browser)" [S30]. **UNVERIFIED:** whether a Desktop-type client accepts
  `response_type=id_token`; it is undocumented, so do not design on it.
- The ID token comes from the token endpoint: the token response's `id_token` "is only returned
  if your request included an identity scope, such as `openid`" [S29]; scope "must begin with the
  `openid` value" [S30].
- `nonce` is a required authentication-request parameter and the ID token's `nonce` claim is
  "The value of the `nonce` supplied by your app in the authentication request" [S30]. This is the
  zkLogin binding: the nonce must equal
  `ToBase64URL(Poseidon_BN254([ext_eph_pk/2^128, ext_eph_pk%2^128, max_epoch, jwt_randomness])[len-20..])`
  [S1]; the proof "confirms the nonce is derived correctly" [S1].
- Embedded browsers are blocked: "Google Account sign-ins from all embedded frameworks will be
  blocked starting on January 4, 2021 ... This block affects CEF-based apps and other non-supported
  browsers" [S32]; loading the OAuth endpoint in an embedded webview yields `disallowed_useragent`
  after 2021-09-30 [S33]. Electron is not named; Electron windows are Chromium-embedded, so plan
  for the block and open the system browser. Google's own doc lists `WKWebView` on macOS as
  disallowed [S29].

### 2.2 Why the stock Enoki wallet does not fit a desktop app

`EnokiWallet` (`@mysten/enoki` 1.3.0, commit `26f6fdfae9`, 2026-10-13) [S26]:

- opens `window.open('about:blank', '_blank', windowFeatures)` and polls `popup.location.hash`
  every 16 ms [S22];
- builds the Google URL with `response_type: 'id_token'` and `redirect_uri` defaulting to
  `window.location.href.split('#')[0]` [S22];
- uses PKCE only for the `playtron` provider (`pkceFlowProviders`) [S22];
- generates the ephemeral key with `WebCryptoSigner.generate()` and stores it with `idb-keyval`
  in IndexedDB; the session (`maxEpoch`, `randomness`, `expiresAt`, `jwt`, `proof`) is encrypted
  with `createDefaultEncryption()` keyed by the **public API key** [S22][S23];
- the docs confirm: "the Enoki SDK will handle the OAuth flow in a pop-up window" [S19].

A desktop renderer has no `https` origin to register as an authorized JavaScript origin, the
popup is an embedded Chromium window (2.1), and the implicit flow is not the documented desktop
flow (2.1). `EnokiFlow` offers `createAuthorizationURL({ redirectUrl })` and
`handleAuthCallback(hash)` but is marked `@deprecated Use registerEnokiWallets instead` and also
hardcodes `response_type: 'id_token'` [S25]. The Enoki FAQ already anticipates this: "In mobile
apps, the impact on user experience is that a web-based authentication flow is triggered in all
cases rather than a native authentication" [S14].

### 2.3 The desktop flow that the sources support

1. Generate an ephemeral Ed25519 key pair in the app [S2].
2. `EnokiClient.createZkLoginNonce({ network: 'testnet', ephemeralPublicKey, additionalEpochs })`
   → `nonce`, `randomness`, `maxEpoch`, `estimatedExpiration` [S16][S24]. (Self-hosted:
   `generateNonce` + `generateRandomness` + `maxEpoch = epoch + N` [S2].)
3. Start a loopback HTTP listener on `127.0.0.1:<random port>`; generate PKCE verifier/challenge;
   open `https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=http://127.0.0.1:<port>&response_type=code&scope=openid&nonce=<nonce>&code_challenge=...&code_challenge_method=S256&state=...`
   in the **system browser** [S29][S30]. Electron: `shell.openExternal`; Tauri: opener plugin
   `openUrl` (default permissions allow `http(s)://`) [S41].
4. Receive `code` on the loopback listener, verify `state`, exchange at
   `https://oauth2.googleapis.com/token` with `code_verifier`; read `id_token` [S29].
5. `EnokiClient.getZkLogin({ jwt })` → `address`, `salt`, `publicKey` [S16]. (Self-hosted:
   your salt service, then `jwtToAddress(jwt, salt, false)` [S2].)
6. `EnokiClient.createZkLoginZkp({ network, jwt, maxEpoch, randomness, ephemeralPublicKey })` →
   proof inputs [S16][S24]. (Self-hosted: `POST $PROVER_URL/v1` [S2].)
7. Sign with `EnokiKeypair` or `ZkLoginSigner({ ephemeralSigner, maxEpoch, inputs })` [S22][S2].

Custom URL scheme remains useful for *non-Google* deep links (Electron
`app.setAsDefaultProtocolClient` + `open-url` on macOS / `second-instance` on Windows and Linux,
with the scheme declared in `Info.plist` at build time [S35]; Tauri deep-link plugin, desktop
schemes under `plugins > deep-link > desktop.schemes`, "only triggered for installed applications
on desktop", macOS testable only from `/Applications` [S39]). It is not the Google redirect.

### 2.4 The `aud` design fork (do not resolve here)

The address is `f(sub, iss, aud, user_salt)` [S1], and Enoki "creates one wallet for a given user,
per `clientID`" [S14]. A Desktop-type Google client and the existing Web-type client for `web/`
have different client IDs, therefore different addresses for the same Google account. Options:

- **A. Separate addresses per surface** (desktop client ID ≠ web client ID). Simple, follows
  Google's client-type rules exactly. Players who use both the website and the app have two
  empires.
- **B. One shared client ID.** Google allows `http://localhost:8080`-style redirect URIs on Web
  clients "for testing" and exempts localhost from the HTTPS rule [S31]; the Enoki FAQ says "you
  need to share the `clientID` across each of these surfaces" [S14]. **UNVERIFIED:** whether the
  token endpoint accepts a Web-type client with PKCE and no `client_secret`, and whether Google
  tolerates a dynamic loopback port on a Web client (ports are not addressed in [S31]).

The custody ticket must pick one before it fixes the Google client type.

---

## 3. Ephemeral key + ZK proof lifecycle

### 3.1 Facts

- The address is permanent; the **session** is the ephemeral key pair plus `max_epoch`. "When the
  epoch passes `max_epoch`, the user logs in again, generates a new ephemeral key pair and
  zero-knowledge proof, and continues using the same address" [S1].
- "A proof is valid until the ephemeral key pair expires (when the current epoch exceeds
  `max_epoch`). Cache and reuse the proof across the session" [S1]; "You can reuse the proof to
  sign any number of transactions until the current epoch crosses `maxEpoch`" [S2].
- Epochs are "about 24 hours on both Mainnet and Testnet"; "Epochs on Devnet are 1 hour" [S9].
- Enoki: `additionalEpochs` default `2`, allowed `0–30` [S16]. With 24 h epochs, the default
  session lasts from the login moment to the end of epoch `current + 2`, i.e. between about 48 h
  and 72 h; the ceiling is about 30–31 days. The Enoki wallet stores `expiresAt =
  estimatedExpiration` (ms) and triggers a new login when `Date.now() > session.expiresAt` or the
  JWT is missing [S22][S23].
- The Sui docs' example uses `maxEpoch = epoch + 2` [S2]; the demo page says "scope `maxEpoch`
  to a short session" [S5].
- JWT `exp`/`iat` are not used by zkLogin; "Expiry is controlled by the `nonce`, which embeds
  `max_epoch`" [S1]. A Google ID token that expires in one hour does not shorten the session; the
  JWT is consumed by the prover once per session.
- `InvalidSignature` from the network means the key expired or the nonce does not match the
  signing key [S2].

### 3.2 Refresh

No silent extension exists. `max_epoch` is hashed into the nonce [S1], and a refreshed ID token
"SHOULD NOT have a nonce Claim ... if it is present, its value MUST be the same as in the ID Token
issued at the time of the original authentication" [S34]. A new session therefore needs a new
nonce, which needs a new authorization request to Google. Whether Google completes that request
without user interaction (`prompt=none` with a live Google session in the system browser) is
**UNVERIFIED** and still opens a browser tab. Budget for the Enoki proof limit of 2 requests per
minute per user when retrying [S14].

### 3.3 What "play while the app runs" and "play while away" can mean

- **While the app runs:** the ephemeral key and cached proof sign locally; "Unlike other wallets,
  signing does not require confirmation to approve the transaction" [S20]. Any game action can be a
  transaction without a prompt until `maxEpoch`. This is the model the custody ticket should
  build.
- **While away (app closed):** the only artefacts that can sign are the ephemeral private key and
  the proof. "If both the ephemeral private key and zero-knowledge proof are compromised, an
  attacker can sign transactions on the user's behalf" [S2]. Handing both to a server to act
  while the player is away is, by the protocol's own definition, full delegation of the account
  for up to `maxEpoch`. The docs say "Never persist them in an insecure location on any platform"
  [S2]. If offline play is wanted, it must be a *Move-level* delegation (a capability object or
  an agent address the game contract honors) or sponsor-proposed transactions the player signs on
  return [S7]. That is a game-design decision outside this ticket.

### 3.4 Where the session lives on disk

- Electron: `safeStorage` encrypts with the OS store (Keychain on macOS; DPAPI on Windows, "only
  a user with the same logon credential ... can typically decrypt the data"; kwallet/libsecret on
  Linux). On Linux with no secret store the backend is `basic_text` and items "will be unprotected
  as they are encrypted via hardcoded plaintext password"; detect with
  `getSelectedStorageBackend()` [S37].
- Tauri: Stronghold plugin, vault at `${appDataDir}/vault.hold`, key derived from a password
  through argon2 with a `salt.txt` in the app data dir; "The password hash must contain exactly
  32 bytes" [S40].
- The Enoki web wallet's own storage (IndexedDB, encryption keyed by the public API key [S23]) is
  obfuscation, not protection; do not copy that pattern to desktop.

---

## 4. Sponsored gas

### 4.1 Protocol

A sponsored transaction has a `GasData.owner` different from `sender`; both parties sign the
entire `TransactionData` including `GasData`, in any order [S7]. The user cannot sign before the
gas is attached, so the sponsor must build (or wrap) the transaction first — the "user proposed"
pattern: user sends `GasLessTransactionData`, sponsor attaches gas and signs, user signs, anyone
submits [S7]. Risks: equivocation if a gas coin or a user input is reused in flight (use dedicated
gas coin pools); censorship if the sponsor relays, so "submit transactions directly to a full
node" [S7]. Production hardening list: per-user rate limits, allowlist Move call targets, cap gas
budget, dedicated sponsor address, separate Testnet/Mainnet sponsors [S7].

### 4.2 Enoki sponsorship

- Private key only; runs from a backend [S17][S15]. Flow: client builds
  `txb.build({ client, onlyTransactionKind: true })`; backend `POST /v1/transaction-blocks/sponsor`
  with `network`, `transactionBlockKindBytes`, and either the `zklogin-jwt` header or `sender`
  ("Include this parameter if not including the `zklogin-jwt` header") → `{ bytes, digest }`;
  client signs `bytes`; backend `POST /v1/transaction-blocks/sponsor/{digest}` with `signature`
  → executed `digest` [S16][S17][S24].
- Guards: `allowedMoveCallTargets` ("List of permitted Move targets the sponsored user's
  transactions can call") and `allowedAddresses` [S16]; "Both private and public API keys require
  setting up the Allowed Move Call Targets" [S14]. The Solitaire example restricts the sponsor
  endpoint to its 11 game functions [S46]. For Trade Wars this list is the `overseer::*` entry
  functions.
- Non-zkLogin senders are supported: "Sponsoring user transactions that have a non-zkLogin
  wallet" is a listed backend use case [S14], which covers Slush players (see 5).
- Limits: 500 sponsor creations/s per team [S14]; plan allowances in 1.2; `$0.001` overage per
  transaction on the bundle [S13].

### 4.3 Self-run gas station

- `MystenLabs/sui-gas-station` now redirects to `MystenLabs/sui-gas-pool` (Rust, Apache-2.0,
  pushed 2026-09-08, last commit on the default branch 2026-06-02) [S42]. "Sui Gas Pool is a
  service that powers sponsored transactions on Sui at scale." APIs: `POST /v1/reserve_gas`
  (reservation released after `reserve_duration_secs`) and `POST /v1/execute_tx`; one Redis
  instance per service; sponsor key in KMS or in memory; "must use a dedicated address"; bearer
  token via `GAS_STATION_AUTH`; `daily-gas-usage-cap` [S42]. Network is whatever `fullnode-url`
  points at [S42].
- Shinami Gas Station: "0.002 SUI for each sponsorship", 10% of the budget above 0.02 SUI; no
  free tier for Gas Station [S43].

### 4.4 Player-funded fallback and faucet

A zkLogin address is an ordinary Sui address and can hold its own SUI. On Testnet the player can
use `https://faucet.sui.io/` ("The requests are rate limited"), the Discord faucet, or community
faucets with limits "such as one request per day" [S10]. "There is no faucet for Mainnet" [S10];
on Mainnet the docs point to fiat on-ramps that deliver to the derived zkLogin address [S2]. The
app can therefore fall back to "pay your own gas" by building a normal transaction with the
player as gas owner whenever sponsorship is unavailable; this is the same `ZkLoginSigner` with no
`setGasOwner` [S7][S2].

### 4.5 Verdict

Sponsorship is a backend concern in every option. For a solo developer on Testnet today: Enoki
private key in a minimal backend (one route, authenticated, target allowlist = Trade Wars entry
functions). Spike whether the Sandbox plan sponsors on Testnet (**UNVERIFIED**, 1.2). If it does
not, run `sui-gas-pool` against Testnet with faucet SUI, or let players faucet their own address
(4.4) until Mainnet matters. Keep the player-funded path as the permanent fallback.

---

## 5. Wallet-standard interop (so a player can still use Slush)

- The Wallet Standard discovers wallets with `getWallets().get()`; extension wallets register
  themselves in the page [S8]. Enoki wallets are Wallet-Standard wallets (`EnokiWallet implements
  Wallet`, features `standard:connect`, `sui:signTransaction`, `sui:signAndExecuteTransaction`,
  `sui:signPersonalMessage`) [S22]; the Sui docs say "Once registered, they appear in the
  standard connection UI alongside any other installed wallets" [S6].
- **Slush browser extension:** unavailable in a desktop shell. "Electron does not support
  arbitrary Chrome extensions from the store" and only loads unpacked extensions with a restricted
  API set [S36]. Tauri documents no extension loading at all (**UNVERIFIED** by absence).
- **Slush web wallet** (`@mysten/slush-wallet`, `registerSlushWallet`): opens a popup channel to
  `https://my.slush.app` and keeps its session in `localStorage`; supports `signTransaction`,
  `signAndExecuteTransaction`, `signPersonalMessage`; the connect modal shows the extension if
  installed, otherwise the web app [S28][S47]. In Electron, `window.open` creates a
  `BrowserWindow` and `webContents.setWindowOpenHandler` must allow it [S38]. **UNVERIFIED:**
  whether Slush's own Google login inside that popup survives Google's embedded-browser block
  (2.1), and how Tauri handles `window.open` (no primary source found).
- **Design consequence:** implement the desktop zkLogin flow from 2.3 as a custom `Wallet` class
  and register it through dapp-kit `walletInitializers` [S48], so the game code calls one
  `signAndExecuteTransaction` for Enoki-zkLogin and Slush alike. Sponsor Slush players with the
  backend `sender` parameter [S16][S14].
- Repo note: `web/` pins `@mysten/sui 1.29.1` and `@mysten/dapp-kit 0.16.3`; the APIs cited here
  (`@mysten/dapp-kit-core`, `SuiGrpcClient`, `@mysten/enoki/client`) are newer [S26][S48]. The
  custody ticket inherits an SDK upgrade.

---

## 6. Constraints the key-custody design must respect

1. **Session length.** One login authorizes signing until `maxEpoch`; Enoki default
   `additionalEpochs = 2` (48–72 h with 24 h epochs), hard ceiling 30 epochs [S16][S9]. Pick a
   value and expose "session expires in N hours" in the UI.
2. **No silent refresh.** A new session needs a new nonce and a new Google authorization request
   [S1][S34]. Design the re-login as a normal, expected event, not an error path.
3. **Secrets on disk.** Ephemeral private key + proof + JWT live only in OS-backed storage
   (Electron `safeStorage` with a `basic_text` check; Tauri Stronghold) [S37][S40]; wipe at
   logout and at `maxEpoch`; never log them [S2]. Do not reuse Enoki's IndexedDB pattern [S23].
4. **Nothing secret in the binary.** Ship the Enoki public key and the Google client ID; keep the
   Enoki private key and any gas-pool key in a backend with its own auth [S15][S17][S14][S29].
5. **OAuth transport.** System browser + loopback `127.0.0.1:<random port>` + `code` + PKCE;
   never an embedded window; never a custom scheme for the Google leg [S29][S32][S33].
6. **Client ID = address.** Decide fork 2.4 (separate vs shared `aud`) before creating the Google
   client; the choice is irreversible for existing players [S1][S14].
7. **Network is explicit.** Enoki defaults to `mainnet`; every call passes `network` [S16].
8. **Sponsorship allowlist.** `allowedMoveCallTargets` = Trade Wars entry functions only; set
   per-user rate limits and a gas cap; separate Testnet and Mainnet sponsors [S16][S7].
9. **Player-funded fallback stays.** Any transaction the sponsor refuses must be buildable with
   the player as gas owner [S7].
10. **"Away" play is not a custody feature.** Server-side signing with the player's ephemeral key
    is account delegation [S2]; route offline play through Move design.
11. **Enoki exit path.** Record that addresses are bound to Enoki's per-app salt; a migration
    needs the `salt` from `GET /v1/zklogin` per player, if permitted (**UNVERIFIED**) [S14][S16].
12. **First spike.** Confirm Enoki accepts public-key calls with no browser `Origin`, and confirm
    whether Sandbox sponsors on Testnet [S15][S16][S13].
13. **JWT lifetime vs session lifetime.** The protocol ignores the JWT `exp` claim [S1], but
    Enoki's `zklogin-jwt` header endpoints may not. Request the proof (`/zklogin/zkp`)
    immediately after login (2.3 step 6), and after the Google ID token's own lifetime use the
    `sender` parameter, not the JWT header, when sponsoring [S16]. Whether Enoki rejects an
    expired `exp` is **UNVERIFIED** (U11).

---

## 7. UNVERIFIED items (collected)

| # | Claim | Why it matters |
| --- | --- | --- |
| U1 | Enoki accepts public-key API calls without a browser `Origin` matching `allowedOrigins`. | Decides whether login needs a proxy. |
| U2 | Enoki Sandbox (free) can sponsor on Testnet. | Decides whether a gas pool is needed on day one. |
| U3 | `prover.mystenlabs.com/v1` on Testnet accepts arbitrary Google client IDs (forum says no, 2026-05-17/06-02) [S44]. | Only relevant if Enoki is dropped. |
| U4 | Google Desktop-type client accepts `response_type=id_token`. | Would let the Enoki implicit flow run on desktop; undocumented. |
| U5 | Google Web-type client + PKCE + loopback with dynamic port and no `client_secret`. | Needed for fork 2.4 option B. |
| U6 | Google `prompt=none` re-authorization without user interaction. | Would soften re-login UX. |
| U7 | Enoki terms permit developer-side salt export for migration; `salt` always populated. | Exit path. |
| U8 | Slush web wallet login works inside an Electron/Tauri popup; Tauri `window.open` behaviour. | Slush escape hatch on desktop. |
| U9 | Tauri cannot load browser extensions (no doc found either way). | Slush extension on desktop. |
| U10 | Custom URI schemes are unsupported for Google Desktop clients specifically (page text is unqualified; platform note names Android/Chrome apps) [S29]. | Loopback is safe regardless. |
| U11 | Enoki endpoints that take the `zklogin-jwt` header reject a JWT whose `exp` has passed. | Decides whether the backend sponsors by JWT header or by `sender` after the first hour. |
| U12 | Which network(s) `https://prover-dev.mystenlabs.com/v1` serves; an older docs snippet said Devnet-only, the current demo page only calls it "a public development prover" [S5]. | Only relevant if Enoki is dropped. |

---

## Sources

Local mirror = `~/.claude/sui-pilot/.sui-docs` / `.ts-sdk-docs`, synced from `MystenLabs/sui@main`
and `MystenLabs/ts-sdks@main` on 2026-08-25. Live pages were read on 2026-10-19 unless stated.

- [S1] Sui docs, zkLogin Technical Reference — https://docs.sui.io/sui-stack/zklogin-integration/zklogin (mirror 2026-08-25)
- [S2] Sui docs, zkLogin Integration Guide — https://docs.sui.io/sui-stack/zklogin-integration/integration-guide (mirror 2026-08-25; live page checked 2026-10-19 for prover/salt access text)
- [S3] Sui docs, zkLogin overview — https://docs.sui.io/sui-stack/zklogin-integration (mirror 2026-08-25)
- [S4] Sui docs, Configure OpenID Providers — https://docs.sui.io/sui-stack/zklogin-integration/developer-account (mirror 2026-08-25)
- [S5] Sui docs, zkLogin Demo — https://docs.sui.io/sui-stack/zklogin-integration/zklogin-demo (mirror 2026-08-25)
- [S6] Sui docs, zkLogin Wallets — https://docs.sui.io/onchain-finance/asset-custody/wallets/zk-login-wallets (mirror 2026-08-25)
- [S7] Sui docs, Sponsored Transactions — https://docs.sui.io/develop/transaction-payment/sponsor-txn (mirror 2026-08-25)
- [S8] Sui docs, Wallet Standard — https://docs.sui.io/onchain-finance/asset-custody/wallets/wallet-standard (mirror 2026-08-25)
- [S9] Sui docs, Epochs / Networks — https://docs.sui.io/develop/sui-architecture/epochs and https://docs.sui.io/develop/sui-architecture/networks (mirror 2026-08-25; live search 2026-10-19)
- [S10] Sui docs, Get SUI from Faucet — https://docs.sui.io/getting-started/onboarding/get-coins (mirror 2026-08-25)
- [S13] Enoki pricing — https://enoki.mystenlabs.com/pricing (2026-10-19)
- [S14] Enoki Developer Portal FAQ — https://docs.enoki.mystenlabs.com/faq (2026-10-19; answers parsed from the page payload)
- [S15] Enoki HTTP API overview — https://docs.enoki.mystenlabs.com/http-api (2026-10-19)
- [S16] Enoki OpenAPI reference — https://docs.enoki.mystenlabs.com/http-api/openapi (2026-10-19)
- [S17] Enoki SDK, Sponsored Transactions — https://docs.enoki.mystenlabs.com/ts-sdk/sponsored-transactions (2026-10-19)
- [S18] Enoki SDK, Register Enoki Wallets — https://docs.enoki.mystenlabs.com/ts-sdk/register (2026-10-19)
- [S19] Enoki SDK, Signing in — https://docs.enoki.mystenlabs.com/ts-sdk/sign-in (2026-10-19)
- [S20] Enoki SDK, Signing Transactions — https://docs.enoki.mystenlabs.com/ts-sdk/transactions (2026-10-19)
- [S21] Enoki Connect — https://docs.enoki.mystenlabs.com/enoki-connect (2026-10-19)
- [S22] `@mysten/enoki` wallet source — https://github.com/MystenLabs/ts-sdks/blob/26f6fdfae9/packages/enoki/src/wallet/wallet.ts (commit 2026-10-13, read 2026-10-19)
- [S23] `@mysten/enoki` wallet state — https://github.com/MystenLabs/ts-sdks/blob/26f6fdfae9/packages/enoki/src/wallet/state.ts (2026-10-19)
- [S24] `@mysten/enoki` client — https://github.com/MystenLabs/ts-sdks/blob/26f6fdfae9/packages/enoki/src/EnokiClient/index.ts and `type.ts` (2026-10-19)
- [S25] `@mysten/enoki` `EnokiFlow` (deprecated) — https://github.com/MystenLabs/ts-sdks/blob/26f6fdfae9/packages/enoki/src/EnokiFlow.ts (2026-10-19)
- [S26] `@mysten/enoki` `package.json` 1.3.0 and `CHANGELOG.md` — https://github.com/MystenLabs/ts-sdks/blob/26f6fdfae9/packages/enoki/package.json (2026-10-19)
- [S27] `@mysten/enoki` `register.ts` — https://github.com/MystenLabs/ts-sdks/blob/26f6fdfae9/packages/enoki/src/wallet/register.ts (2026-10-19)
- [S28] `@mysten/slush-wallet` wallet source — https://github.com/MystenLabs/ts-sdks/blob/main/packages/slush-wallet/src/wallet/index.ts (2026-10-19)
- [S29] Google, OAuth 2.0 for Mobile & Desktop Apps — https://developers.google.com/identity/protocols/oauth2/native-app (2026-10-19)
- [S30] Google, OpenID Connect — https://developers.google.com/identity/openid-connect/openid-connect (2026-10-19)
- [S31] Google, OAuth 2.0 for Web Server Applications (redirect URI rules) — https://developers.google.com/identity/protocols/oauth2/web-server (2026-10-19)
- [S32] Google Developers Blog, 2020-08, "Guidance to developers affected by our effort to block less secure browsers and applications" — https://developers.googleblog.com/2020/08/guidance-for-our-effort-to-block-less-secure-browser-and-apps.html (2026-10-19)
- [S33] Google Developers Blog, 2021-06, "Upcoming security changes to Google's OAuth 2.0 authorization endpoint in embedded webviews" — https://developers.googleblog.com/2021/06/upcoming-security-changes-to-googles-oauth-2.0-authorization-endpoint.html (2026-10-19)
- [S34] OpenID Connect Core 1.0, §12.2 Successful Refresh Response — https://openid.net/specs/openid-connect-core-1_0.html (2026-10-19)
- [S35] Electron, `app.setAsDefaultProtocolClient`, `open-url`, `second-instance` — https://www.electronjs.org/docs/latest/api/app (2026-10-19)
- [S36] Electron, Extensions — https://www.electronjs.org/docs/latest/api/extensions (2026-10-19)
- [S37] Electron, `safeStorage` — https://www.electronjs.org/docs/latest/api/safe-storage (2026-10-19)
- [S38] Electron, Opening windows from the renderer — https://www.electronjs.org/docs/latest/api/window-open (2026-10-19)
- [S39] Tauri v2, Deep Linking plugin — https://v2.tauri.app/plugin/deep-linking/ (2026-10-19)
- [S40] Tauri v2, Stronghold plugin — https://v2.tauri.app/plugin/stronghold/ (2026-10-19)
- [S41] Tauri v2, Opener plugin — https://v2.tauri.app/plugin/opener/ (2026-10-19, via search summary)
- [S42] `MystenLabs/sui-gas-station` → `MystenLabs/sui-gas-pool` README and GitHub API metadata — https://github.com/MystenLabs/sui-gas-pool (README read 2026-10-19; `pushed_at` 2026-09-08; last default-branch commit 2026-06-02)
- [S43] Shinami, Sui pricing — https://www.shinami.com/pricing/sui (2026-10-19, via search summary)
- [S44] Sui forum, "Hosted zkLogin prover: Google OAuth audience not supported on Testnet" — https://forums.sui.io/t/hosted-zklogin-prover-google-oauth-audience-not-supported-on-testnet/49289 (post 2026-05-17, reply 2026-06-02; read 2026-10-19)
- [S46] Sui docs, Enoki Solitaire example — https://docs.sui.io/sui-stack/enoki/solitaire (mirror 2026-08-25)
- [S47] Sui TS SDK docs, Slush Wallet App Integration — https://sdk.mystenlabs.com/slush-wallet/dapp (mirror 2026-08-25)
- [S48] Sui TS SDK docs, dApp Kit instance (`walletInitializers`) — https://sdk.mystenlabs.com/dapp-kit/dapp-kit-instance (mirror 2026-08-25)
