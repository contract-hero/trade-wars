# Can a third-party desktop game use the player's Claude and ChatGPT subscriptions?

Resolves wayfinder ticket [#44](https://github.com/alilloig/trade-wars/issues/44).
All sources were read on **2026-10-19**. Every claim carries a numbered source; the list is in the [Sources](#sources) section. Anything marked **UNVERIFIED** could not be confirmed against a primary source.

## 1. Verdict

| Provider | Go / no-go | Mechanism that is allowed | What is not allowed |
| --- | --- | --- | --- |
| **Anthropic** | **NO-GO** for subscription OAuth as the game's auth path. **GO** for the API key path, which Max subscribers can fund with included monthly API credits. | Player creates a Console API key and pastes it into the game. Max 5x / Max 20x plans include $100 / $200 of monthly API credits that cover the Claude API [S9]. | The game may not offer claude.ai login, route requests through Pro/Max credentials, or store Claude session tokens [S1][S2]. |
| **OpenAI** | **CONDITIONAL GO** through "Sign in with ChatGPT" with plan usage, on the open-source / locally hosted path. | OAuth flow with scope `chatgpt.tokens.use.direct`, then `POST https://api.openai.com/v1/responses` with `store:false`, `stream:true`, bearer access token. Plus and Pro plans are eligible [S15][S17][S18][S19]. | Spawning `codex app-server` with the user's CLI login inside a product: "App-server authentication has never been permitted for commercial or hosted services" [S21]. A paid or remotely hosted app must fill the interest form first [S17]. |

Trade Wars is public and GPL-3.0 licensed (checked with `gh repo view alilloig/trade-wars` on 2026-10-19), and the model calls run on the player's machine. This matches the OpenAI open-source path.

## 2. Scope and method

The question: a desktop game client wants to call a model many times per hour on the player's own AI subscription, the way T3 Code drives Claude Code and Codex on the user's plans.

Method: read Anthropic and OpenAI terms, product docs and support articles; read the T3 Code source at commit `07d38b8d17167c8d17fba1184dbe13a307fada74` (2026-10-18); read the OpenCode repository for the enforcement timeline; read the live pricing pages for the fallback cost.

## 3. Anthropic

### 3.1 What the terms and docs say

**Consumer Terms of Service** (effective October 8, 2025) [S3]:

- Section 2: "You may not share your Account login information, Anthropic API key, or Account credentials with anyone else."
- Section 3, item 7: access "through automated or non-human means" is prohibited, with an exception for access through an Anthropic API key or where Anthropic otherwise explicitly permits it.
- Section 11: the subscriber "will not use our Services for any commercial or business purposes."
- "Our Commercial Terms of Service govern your use of any Anthropic API key."

**Claude Code: Legal and compliance** [S1], section "Authentication and credential use":

> OAuth authentication is intended exclusively for purchasers of Claude Free, Pro, Max, Team, and Enterprise subscription plans and is designed to support ordinary use of Claude Code and other native Anthropic applications.

> Developers building products or services that interact with Claude's capabilities, including those using the Agent SDK, should use API key authentication through Claude Console or a supported cloud provider. Anthropic does not permit third-party developers to offer Claude.ai login into their own applications, or to route requests through Free, Pro, or Max plan credentials on behalf of their users. Moreover, developers may not collect, store, or intermediate Claude.ai credentials or session tokens — sign-in to a Claude account must complete through Anthropic's own flow.

> Nor does it prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription, including where a platform hosts Claude Code as described under *Can customers offer Claude Code in their products?* above.

> Anthropic reserves the right to take measures to enforce these restrictions and may do so without prior notice.

The same page, section "Acceptable use": "Advertised usage limits for Pro and Max plans assume ordinary, individual usage of Claude Code and the Agent SDK." [S1]

The same page, section "Can customers offer Claude Code in their products?": a product that runs Claude Code must not modify the binary, must not remove any built-in authentication method, and "may not pay for, resell, or intermediate Claude usage on their end users' behalf." Each end user authenticates with their own API key, subscription credentials, or cloud credential, under the Commercial Terms [S1].

**Agent SDK overview** [S2]:

> Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK. Use the API key authentication methods described in the Quickstart instead.

The Agent SDK is governed by the Commercial Terms "including when you use it to power products and services that you make available to your own customers and end users" [S2].

**Headless mode docs** [S4]: `--bare` "is the recommended mode for scripted and SDK calls, and will become the default for `-p` in a future release." In bare mode "Claude Code never reads OAuth credentials or the system keychain." The example says: "Set `ANTHROPIC_API_KEY` before running it, because bare mode doesn't use your subscription login."

**Authentication docs** [S5]: `claude setup-token` prints a one-year OAuth token for `CLAUDE_CODE_OAUTH_TOKEN`. "This token authenticates with your Claude subscription and requires a Pro, Max, Team, or Enterprise plan." Subscription OAuth from `/login` is credential source 7 of 7 in the precedence list; `ANTHROPIC_API_KEY` is source 3. "Bare mode does not read `CLAUDE_CODE_OAUTH_TOKEN`."

**Support article "Use the Claude Agent SDK with your Claude plan"** [S8]:

- Update June 15, 2026: "We've paused the previously-announced changes to Claude Agent SDK usage." "Claude Agent SDK, claude -p, and third-party app usage still draw from your subscription limits."
- Update October 7, 2026: "Claude Max and Team plans now include monthly API credits, which cover the Claude Agent SDK, claude -p, the Claude API, and Claude Managed Agents." "You can still use the Claude Agent SDK, claude -p, and third-party apps with your subscription limits."

Reading of these together: the subscriber may run the SDK, `claude -p`, and third-party apps on their plan. The developer may not build the product around claude.ai login or plan credentials. The only sanctioned overlap is the end user who signs in to the unmodified `claude` binary themselves.

### 3.2 Enforcement timeline against third-party harnesses

| Date | Event | Source type |
| --- | --- | --- |
| 2026-01-09 | OpenCode users report Claude Max access stopped working with an error (issue "Broken Claude Max", anomalyco/opencode #7410, opened 2026-01-09). The maintainer wrote "it's possible they blocked opencode specifically." Closed "not planned" on 2026-03-29. | Primary (GitHub) [S10] |
| 2026-02 (press reports 2026-02-19) | Claude Code docs gain the "Authentication and credential use" section quoted above. | Live docs are primary [S1]; the date is **UNVERIFIED** (secondary press only [S12][S13]; Wayback fetch failed). |
| 2026-03-19 | OpenCode PR #18186 "anthropic legal requests", opened and merged 2026-03-19: "Remove anthropic references per legal requests", removes the `opencode-anthropic-auth` plugin, the `claude-code-20250219` beta header, and the Pro/Max OAuth docs. | Primary (GitHub) [S11] |
| 2026-04-04 | Anthropic email to subscribers (quoted by TechCrunch, 2026-04-04): subscribers "no longer be able to use your Claude subscription limits for third-party harnesses including OpenClaw"; the policy "applies to all third-party harnesses and will be rolled out to more shortly." Boris Cherny: "subscriptions weren't built for the usage patterns of these third-party tools." | Secondary (TechCrunch quoting the email) [S14]. The one-time credit reported elsewhere is **UNVERIFIED**. |
| 2026-06-15 | Anthropic pauses the announced Agent SDK change; SDK, `claude -p`, and third-party apps keep drawing from subscription limits. | Primary (support article) [S8] |
| 2026-10-07 | Max and Team plans gain monthly API credits that cover the Claude API, Agent SDK, `claude -p`, and Managed Agents. | Primary (support article) [S8][S9] |

### 3.3 How T3 Code authenticates to Claude

T3 Code (github.com/pingdotgg/t3code) is public, MIT licensed, and reads at commit `07d38b8d` (2026-10-18) [S22]:

- Dependency: `"@anthropic-ai/claude-agent-sdk": "^0.3.276"`.
- `docs/user/providers-claude.md`: "T3 Code uses Claude Code's login and configuration." Setup is `claude auth login` run by the user; a second account uses `CLAUDE_CONFIG_DIR=~/.claude_personal claude auth login`.
- `apps/server/src/provider/ClaudeProvider.ts`: calls the Agent SDK with `pathToClaudeCodeExecutable` set to the user's installed `claude` binary and passes the user's environment through. It reports `subscriptionType` (Pro, Max 5x, Max 20x, Team, Enterprise) or `apiKeySource` for display. An `apiKey` auth method exists for `ANTHROPIC_API_KEY` / `ANTHROPIC_AUTH_TOKEN` installs.
- T3 Code never performs the Claude login and never stores Claude tokens. The README says: "Works with your subscriptions on Claude Code, Codex, ..."; "If they're set up on your computer, T3 Code can control them."

This is the "end user signs in to the unmodified Claude Code binary" carve-out [S1]. T3 Code is a coding harness; the carve-out text names "ordinary use of Claude Code." Whether T3 Code holds a "previously approved" status under the Agent SDK note [S2] is **UNVERIFIED**.

### 3.4 Why this is a no-go for a game loop

1. The Agent SDK note forbids offering "claude.ai login or rate limits for their products" [S2]. A game advertised as "free with your Claude plan" offers plan rate limits for a product.
2. The acceptable-use text ties Pro/Max limits to "ordinary, individual usage of Claude Code and the Agent SDK" [S1]. A decision loop that calls the model many times per hour is the pattern Anthropic named "outsized strain" in April 2026 [S14]. The June pause [S8] is a pause, not a policy reversal; the legal page keeps the enforcement clause [S1].
3. The recommended programmatic mode, `--bare`, does not read subscription credentials [S4]. Subscription auth for scripted use is on a deprecating path.
4. Each `claude -p` call is a full Claude Code turn, with Claude Code's system prompt and tools. That consumes the 5-hour and weekly windows far faster than a 1,500-token API call. Anthropic publishes no message counts for Pro or Max [S6][S7].

### 3.5 The sanctioned "costless for Max subscribers" path

Monthly API credits [S9]:

- Max 5x: $100 per month. Max 20x: $200 per month. Team: $20 per Standard seat, $100 per Premium seat, pooled, capped at $500 per month. Free, Pro, and Enterprise: not eligible.
- Cover "The Claude API (Messages and Message Batches APIs), the Console Playground, Claude Managed Agents, and the Claude Agent SDK."
- Do not cover interactive Claude Code, "extra usage," or Bedrock / Vertex / Foundry.
- Do not roll over: "Unused credits expire at the end of each billing cycle."
- Activation: the subscriber links one Claude Console organization in claude.ai Settings > Billing. Credits land in that organization's balance. "Usage is never charged to your Claude plan."

With a Console API key from the linked organization, a Max 5x subscriber has $100 per month of Claude API usage. At the per-call cost in section 6, that is about 570,000 Haiku 5.5 decisions per month. Pro subscribers have no credit and pay API rates.

Usage bundles are a separate, prepaid product on Pro, Max, and Team: $50 for $45, $250 for $200, $1,000 for $700. The balance "works across Claude, Claude Code, Claude Cowork, and third-party products" [S30]. The article does not define "third-party products."

## 4. OpenAI

### 4.1 What the docs say

**Codex authentication** [S15]: "Sign in with ChatGPT for subscription access." "API key authentication supports local Codex workflows." "Use API key authentication for programmatic Codex CLI workflows, such as CI/CD jobs." The page has no sentence that restricts ChatGPT sign-in to OpenAI's own apps.

**Codex non-interactive mode** [S16]: "`codex exec` reuses saved CLI authentication by default." "API keys are the right default for automation because they are simpler to provision and rotate." ChatGPT-managed login in automation: "Use this path only if you specifically need to run as your Codex account." "Treat `~/.codex/auth.json` like a password: it contains access tokens."

**Codex app-server** [S21], section "Auth endpoints":

> If you've built a local or open-source application using Codex app-server authentication, you can continue using it, [...] we recommend migrating to Sign in with ChatGPT so users have greater control over and visibility into their usage.

> App-server authentication has never been permitted for commercial or hosted services.

Section "Initialization": a new integration "intended for enterprise use" should "contact OpenAI to get it added to a known clients list." The protocol exposes `apiKey`, `chatgpt` (Codex runs the OAuth flow), `chatgptDeviceCode`, and experimental `chatgptAuthTokens` for "host apps that already own the user's ChatGPT auth lifecycle." The WebSocket transport "aren't supported for production workloads."

**Sign in with ChatGPT (SIWC)**:

- Quickstart [S17]: "ChatGPT plan usage is available to all open-source partners and selected private clients." "Sign in with ChatGPT is currently available to selected commercial partners through a limited trial." Plan usage "doesn't grant access to the user's API key or conversations." Open-source clients get OAuth credentials "for eligible Responses API requests" with no client secret.
- Plan usage overview [S18]: for "open-source and locally hosted apps." A "paid or remotely hosted app" must complete the interest form. Each host persists an opaque `ext_agent_host_id`; "Use an opaque value, not an email, user ID, or other user-identifying value." Send inference with `store: false` and `stream: true`.
- Request a client ID [S20]: "Sign in with ChatGPT is currently offered to a select group of commercial partners." "To join the waitlist, complete the Sign in with ChatGPT interest form."
- Models and inference [S19]: endpoint `POST https://api.openai.com/v1/responses`, `Authorization: Bearer <ACCESS_TOKEN>`. "Use a model available to the signed-in account"; list them from `https://api.openai.com/v1/models` and keep `visibility == "list"`. Examples use `gpt-6.1-sol`. "do not point it at ChatGPT's `backend-api` endpoints." Mid-stream usage errors: `subscription_sharing_usage_limit_exceeded`, `subscription_sharing_usage_unavailable`.
- Preview limitations [S23]: rejected or ignored request fields include `max_output_tokens`, `temperature`, `top_p`, `metadata`, `previous_response_id` over HTTP, and explicit system-message items (use `instructions`). Function and custom tools work when grouped in namespaces or passed via `additional_tools`. Unsupported: image generation, file search, Code Interpreter, hosted MCP, `tool_search`, audio input.
- SIWC with Codex app-server [S24]: the app sets `ACCESS_TOKEN` in the app-server child environment and starts it with a provider config `env_key="ACCESS_TOKEN"`, `wire_api="responses"`, `requires_openai_auth=false`. "No separate Codex sign-in is required." The app refreshes the token and restarts app-server.
- User-facing page [S25]: "eligible ChatGPT Plus and Pro subscribers can also choose to use their ChatGPT plan for AI requests." The user grants a "Use your ChatGPT plan" permission, can disconnect in ChatGPT Settings > Security and login, and can set a per-app weekly cap from 10% to 100% of weekly plan usage. "An app's limit is a cap, not a separate allowance or a reserved portion of your plan."

**Is `gpt-6.1-sol` available this way?** Yes on Plus and Pro. The Codex models page [S26]: the GPT-6.1 Sol launch rollout "includes Plus, Pro, Business, Enterprise, and Edu"; Free and Go are not included at launch. The pricing page [S27]: Plus lists "GPT-6.1 Sol and GPT-6 Luna"; GPT-6.1 Sol is in Work and Codex, not Chat. The Codex SDK page [S28] uses `gpt-6.1-sol` as its example model, "which must be available to your signed-in account." For SIWC, the app must confirm the model from `/v1/models` at run time [S19]; an explicit SIWC model catalog is **UNVERIFIED**.

### 4.2 How T3 Code authenticates to Codex

T3 Code offers two Codex paths [S22]:

1. "Use an existing Codex login": the user runs `codex login`; T3 Code spawns `codex app-server` with that `CODEX_HOME`. This is the app-server auth path that OpenAI now tolerates for existing local / open-source apps and asks to migrate [S21].
2. "Connect with ChatGPT": `apps/server/src/provider/CodexChatGptAuth.ts` implements SIWC. Discovery at `https://auth.openai.com/.well-known/openid-configuration`; `client_id` defaults to `dynamic_agent_client`; the request carries `ext_agent_host_id`; `redirect_uri` is `http://127.0.0.1:<port>/auth/callback`; scope is `openid profile email offline_access resource.invoke chatgpt.tokens.use.direct`. `REQUIRED_SCOPE = "chatgpt.tokens.use.direct"`; without it the UI says "Token sharing is disabled. Sign in with ChatGPT and enable token sharing." `apps/server/src/provider/Drivers/CodexManagedProvider.ts` passes the token as `ACCESS_TOKEN` into the managed app-server process, which is the documented SIWC app-server pattern [S24].

T3 Code is the working reference implementation of the OpenAI path this game would use.

### 4.3 Conditions for a go

- The model calls must run on the player's machine. A game server that calls OpenAI with the player's token is a "remotely hosted app" and needs the interest form [S18].
- Use SIWC, not `codex app-server` with the CLI login. The app-server CLI-login path is "never permitted for commercial or hosted services" and is legacy-tolerated for local open-source apps only [S21].
- Treat the feature as preview. The game cannot send `max_output_tokens` or `temperature` on plan-usage requests [S23]. A 50-token decision must be capped by prompt design and structured output, not by the request field.
- Handle `subscription_sharing_usage_limit_exceeded` and the per-app weekly cap as normal game states [S19][S25].

## 5. Usage caps and rate limits for a game loop

| Plan | What the provider publishes | Reading for a loop at ~60 calls/hour |
| --- | --- | --- |
| Claude Pro / Max (subscription) | 5-hour rolling session limit plus a weekly limit; Max has a separate weekly Fable limit; Claude and Claude Code share one pool. No message or token counts are published. "Pro gives you at least 5x more usage per 5-hour session than Free"; Max is 5x or 20x Pro [S6][S7][S29]. | Not measurable from public numbers. Each `claude -p` turn carries the Claude Code system prompt, so a loop burns the window faster than a bare API call. Not recommended (section 3.4). |
| Claude API key, Start tier | Haiku 5.5: 1,000 RPM, 2,000,000 input tokens/min, 400,000 output tokens/min. Monthly spend cap $500 (Start), $1,000 (Build), $200,000 (Scale). Cached input tokens do not count toward ITPM [S31]. New organizations may start in an Evaluation tier with lower limits [S31]. | 60 calls/hour is 1 call/minute: far inside the limits. |
| ChatGPT Plus (SIWC plan usage) | Codex local messages per five-hour period, Plus: GPT-6.1 Sol 15-160, GPT-6 Luna 350-3,000, GPT-6 Astra 5-45. "These estimates are not fixed message limits." "Weekly limits may also apply." [S27] Per-app weekly cap 10-100% set by the user [S25]. | 300 calls per 5 hours exceeds the Plus GPT-6.1 Sol estimate. GPT-6 Luna (350-3,000) fits. Whether a raw `/v1/responses` call equals one "Codex local message" is **UNVERIFIED**. |
| ChatGPT Pro (SIWC plan usage) | "Pro plans currently have no five-hour limit." "Weekly limits may also apply." [S27] | Fits. Weekly cap is the only constraint. |
| OpenAI API key, Build tier | Sol family: 5,000 RPM, 1,000,000 TPM; Luna: 5,000 RPM, 2,000,000 TPM. Build requires $5 in credit purchases; Launch ($100) raises Sol to 10,000 RPM / 4,000,000 TPM [S32]. | Far inside the limits. |

## 6. Fallback: BYO API key, per-call cost

Decision shape: 1,500 input tokens, 50 output tokens. Prices are per million tokens, standard tier, read 2026-10-19.

| Model | Input | Output | Cached input | Cost per call, no cache | Cost per call, 1,200 of 1,500 input tokens cached | 60 calls/hour, no cache |
| --- | --- | --- | --- | --- | --- | --- |
| Claude Haiku 5.5 (prompts up to 100K tokens) [S33] | $0.10 | $0.50 | $0.01 | **$0.000175** | $0.000067 | $0.0105 / hour |
| gpt-6.1-sol [S34] | $2.00 | $10.00 | $0.10 | **$0.0035** plus reasoning tokens | $0.0012 plus reasoning tokens | $0.21 / hour |
| gpt-6-luna [S34] | $0.10 | $0.50 | $0.01 | $0.000175 | $0.000067 | $0.0105 / hour |

Notes:

- Arithmetic, Haiku 5.5: 1,500 x $0.10 / 1,000,000 = $0.00015; 50 x $0.50 / 1,000,000 = $0.000025; sum $0.000175.
- Arithmetic, gpt-6.1-sol: 1,500 x $2.00 / 1,000,000 = $0.003; 50 x $10.00 / 1,000,000 = $0.0005; sum $0.0035. OpenAI bills reasoning tokens at the output rate; the SIWC preview rejects `max_output_tokens` [S23], so the API-key path is the only one where output can be capped.
- Haiku 5.5 charges $0.50 input / $2.50 output for prompts over 100,000 tokens [S33]; a 1,500-token decision never reaches that tier. The Haiku 5.5 tool-use system prompt adds 286 tokens when `tools` is present [S33].
- Claude Batch API halves the price (Haiku 5.5 $0.05 / $0.25) [S33]; OpenAI Batch and Flex halve gpt-6.1-sol to $1.00 / $5.00 [S34]. Batch is not suitable for a real-time game loop.
- 100 hours of play at 60 calls/hour: Haiku 5.5 about $1.05; gpt-6.1-sol about $21 before reasoning tokens; gpt-6-luna about $1.05.
- A Max 5x subscriber's $100 monthly API credit [S9] covers about 570,000 Haiku 5.5 decisions per month.

## 7. Recommendation

1. Build the BYO API key path first and make it the documented default for both providers. It is the only path both vendors sanction for a third-party product, and the per-decision cost is below $0.0002 on Haiku 5.5 or gpt-6-luna.
2. For Anthropic, present Max monthly API credits as the "costless for subscribers" story. Do not offer claude.ai login, do not read `~/.claude` credentials, and do not spawn `claude -p` on the player's subscription as a product feature.
3. For OpenAI, implement Sign in with ChatGPT plan usage on the open-source path, following T3 Code's `CodexChatGptAuth.ts` and the SIWC docs. Keep inference on the player's machine. Show the per-app weekly cap and the usage-limit errors in the UI.
4. Prefer the cheap tier on each provider for the decision loop: Haiku 5.5 and gpt-6-luna. Reserve gpt-6.1-sol or Sonnet 5.5 for rare, high-value decisions.
5. Re-read [S1], [S2], [S8], [S21], and [S23] before each release. Both vendors changed these pages several times in 2026.

## 8. UNVERIFIED items

- Exact date the Claude Code legal page gained the "Authentication and credential use" section (press says 2026-02-19; Wayback fetch failed).
- The April 2026 one-time credit equal to one month's subscription (reported by secondary sources; TechCrunch did not mention it).
- Whether T3 Code holds a "previously approved" status with Anthropic.
- ChatGPT Plus and Pro monthly prices (openai.com and help.openai.com returned HTTP 403 to every fetch, including a browser user agent).
- OpenAI consumer Terms of Use clause text (same 403; a search snippet shows "Automatically or programmatically extract data or Output" under "What you cannot do" and "Our Business Terms govern use of ChatGPT Enterprise, our APIs...", effective January 1, 2026).
- An explicit catalog of models reachable through SIWC plan usage; the docs say to list `/v1/models` at run time.
- Whether one `/v1/responses` call equals one "Codex local message" in the Plus 5-hour estimates.
- Whether Haiku 5.5 is selectable in Claude Code on Pro and Max subscriptions; the model-config page says Haiku models "are always available and can't be disabled" in the organization-restriction context, and does not list per-plan models [S35].

## Sources

All read on 2026-10-19.

- [S1] Claude Code docs, Legal and compliance: https://code.claude.com/docs/en/legal-and-compliance
- [S2] Claude Code docs, Agent SDK overview: https://code.claude.com/docs/en/agent-sdk/overview
- [S3] Anthropic Consumer Terms of Service (effective October 8, 2025): https://www.anthropic.com/legal/consumer-terms
- [S4] Claude Code docs, Run Claude Code programmatically (headless, `--bare`): https://code.claude.com/docs/en/headless
- [S5] Claude Code docs, Authentication (`setup-token`, precedence): https://code.claude.com/docs/en/authentication
- [S6] Claude support, Use Claude Code with your Pro or Max plan: https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan
- [S7] Claude support, What is the Max plan?: https://support.claude.com/en/articles/11049741-what-is-the-max-plan
- [S8] Claude support, Use the Claude Agent SDK with your Claude plan (updates 2026-06-15 and 2026-10-07): https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan
- [S9] Claude support, Monthly API credits for Max and Team plans: https://support.claude.com/en/articles/17154008-monthly-api-credits-for-max-and-team-plans
- [S10] anomalyco/opencode issue #7410 "Broken Claude Max" (opened 2026-01-09): https://github.com/anomalyco/opencode/issues/7410
- [S11] anomalyco/opencode PR #18186 "anthropic legal requests" (merged 2026-03-19): https://github.com/anomalyco/opencode/pull/18186
- [S12] Secondary: WinBuzzer, 2026-02-19, "Anthropic Bans Claude Subscription OAuth in Third-Party Apps": https://winbuzzer.com/2026/02/19/anthropic-bans-claude-subscription-oauth-in-third-party-apps-xcxwbn/
- [S13] Secondary: AlternativeTo, February 2026: https://alternativeto.net/news/2026/2/anthropic-officially-bans-using-subscription-authentication-for-third-party-claude-use
- [S14] Secondary: TechCrunch, Anthony Ha, 2026-04-04, quoting the Anthropic email and Boris Cherny: https://techcrunch.com/?p=3109429
- [S15] OpenAI Codex docs, Authentication: https://learn.chatgpt.com/docs/auth (redirect target of https://developers.openai.com/codex/auth)
- [S16] OpenAI Codex docs, Non-interactive mode: https://learn.chatgpt.com/docs/non-interactive-mode.md
- [S17] OpenAI, Sign in with ChatGPT quickstart: https://developers.openai.com/siwc/quickstart
- [S18] OpenAI, ChatGPT plan usage overview (open source): https://developers.openai.com/siwc/token-sharing-open-source
- [S19] OpenAI, SIWC Models and inference: https://developers.openai.com/siwc/token-sharing-open-source/models-and-inference
- [S20] OpenAI, Request a client ID: https://developers.openai.com/siwc/request-client-id
- [S21] OpenAI Codex docs, App server: https://learn.chatgpt.com/docs/app-server.md
- [S22] T3 Code source, commit 07d38b8d17167c8d17fba1184dbe13a307fada74 (2026-10-18), MIT: https://github.com/pingdotgg/t3code ; files `README.md`, `docs/user/providers-claude.md`, `docs/user/providers-codex.md`, `apps/server/src/provider/ClaudeProvider.ts`, `apps/server/src/provider/CodexChatGptAuth.ts`, `apps/server/src/provider/Drivers/CodexManagedProvider.ts`
- [S23] OpenAI, SIWC Preview limitations: https://developers.openai.com/siwc/token-sharing-open-source/preview-limitations
- [S24] OpenAI, SIWC with Codex app-server: https://developers.openai.com/siwc/token-sharing-open-source/codex-app-server
- [S25] OpenAI Codex docs, Sign in with ChatGPT (user-facing): https://learn.chatgpt.com/docs/sign-in-with-chatgpt
- [S26] OpenAI Codex docs, Models: https://learn.chatgpt.com/docs/models.md
- [S27] OpenAI Codex docs, Pricing and plan limits: https://learn.chatgpt.com/docs/pricing (redirect target of https://developers.openai.com/codex/pricing)
- [S28] OpenAI Codex docs, Codex SDK: https://learn.chatgpt.com/docs/codex-sdk.md
- [S29] Anthropic, claude.com pricing: https://claude.com/pricing
- [S30] Claude support, Buy usage bundles: https://support.claude.com/en/articles/14246112-buy-usage-bundles
- [S31] Claude API docs, Rate limits: https://platform.claude.com/docs/en/api/rate-limits
- [S32] OpenAI API docs, Rate limits: https://developers.openai.com/api/docs/guides/rate-limits
- [S33] Claude API docs, Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- [S34] OpenAI API docs, Pricing: https://developers.openai.com/api/docs/pricing
- [S35] Claude Code docs, Model configuration: https://code.claude.com/docs/en/model-config
- [S36] anthropics/claude-agent-sdk-python issue #559 (community thread, no Anthropic reply; closed by author 2026-02-16): https://github.com/anthropics/claude-agent-sdk-python/issues/559
