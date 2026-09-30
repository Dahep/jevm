# jevm — synthesized design package (arena Phase E)

This is the base candidate (C) with grafts from A and B folded in by hand, adjudications applied, and the Synthesis decision section filled. It is the source for docs/design.md in the jevm repo.

## Problem

jevm replaces a scrapped bash CLI (`bin/jev`) and a third-party Node MCP server (`@jkudish/jev-mcp@0.11.0`) with one Go project wrapping OpenCode Zen's System One endpoint (model `jev-1.13`). Three consumers share one core: a human in a shell, opencode agents (via Bash and via MCP tools), and future Go apps importing the module. Constraints that shape it: the wire contract is fixed by Zen/TypeSafe (`POST https://opencode.ai/zen/v1/systemone`, `{model, state, questions}`, noul/choice/score, errors 401/422/429/529, 64k/32k token budgets, text-only state) so all design freedom lives in normalization and policy, not protocol; the three consumers must see **one** answer shape, so normalization happens exactly once in the SDK; the judged party (an LLM agent) must not be able to lower its own pass thresholds; distribution is mise's `github:` backend, which selects release assets by name; billing is jev-1.13 only (~$0.042/1M input, output free, ≈$0.00002/typical call); keys are per-consumer (opencode supplies its own from its auth.json, apps pass theirs, the CLI reads `JEV_API_KEY`).

## Usage (caller's view)

### README quickstart

```markdown
# jevm — System One decisions for your shell, your agents, and your Go code

One binary, three ways in: CLI verbs (`jev <verb>`), an MCP server (`jev serve`),
and a Go SDK (this module's root package).

## Install
    mise use -g github:Dahep/jevm@1.2.3      # dotfiles: scripts/setup-jevm.sh pins this
    export JEV_API_KEY=...                   # your OpenCode Zen key (CLI/apps)

## First call
    $ jev noul --state "Deploy finished 14:02, 3 warnings, exit 0" "Did the deploy succeed?"
    {"schema":1,"answers":[{"id":"noul","type":"noul","value":0.93,
      "probabilities":{"true":0.93,"false":0.07},"confidence":null,"action":"auto"}],
     "usage":{"input_tokens":41,"estimated_usd":0.0000017}}

## Gate an action
    $ jev gate --state-file task.md --claim "tests pass" --strict && gh pr create --fill
    # exit 0 = every row auto · exit 4 = something needs review · exit 1 = bad key/request
```

### Call site 1 — human gating a commit

```console
$ jev gate --state "$(git diff --cached)" \
    --claim "implements #123" --claim "tests included" --strict \
  && gh pr create --fill
```

`--strict` turns the verdict into an exit code so the shell enforces; the envelope on stdout is the audit trail. On 429/529 the call still exits 0 with `"action":"escalate"` rows — fail open to review, never crash the script, never wave bad work through.

### Call site 2 — opencode agent via Bash (canonical invocation, AGENTS.md-liftable)

```bash
jev verify --state-file evidence.md --claim "no secrets in diff" --claim "tests pass"
# act on .answers[].action: auto → proceed · review → double-check · escalate → ask the human
```

### Call site 3 — opencode agent via MCP (no shell)

`jev_gate { "state": "...", "claims": ["tests pass"] }` returns the same envelope as structured content. Wired once in `opencode.json`:

```json
"mcp": { "jev": {
  "type": "local",
  "command": ["bash", "-c", "JEV_API_KEY=$(jq -r '.opencode.key // empty' \"$HOME/.local/share/opencode/auth.json\") exec \"$HOME/.local/share/mise/shims/jev\" serve"],
  "enabled": true
} }
```

(opencode supplies its own key from its own store — per-consumer keys; `OPENCODE_API_KEY` is not exported in this environment. Alternative if the user later exports it: `["mise","exec","--","jev","serve"]` + `"environment": {"JEV_API_KEY": "{env:OPENCODE_API_KEY}"}`.)

### Call site 4 — future Go app (full call, 9 lines)

```go
client := jevm.New(os.Getenv("JEV_API_KEY")) // Zen endpoint + jev-1.13 defaults
resp, err := client.Evaluate(ctx, jevm.Request{
    State: jevm.StringState(evidence),
    Questions: jevm.Questions{
        "ship": jevm.Noul("Ship this change now?"),
        "tier": jevm.Choice("Which tier?", map[string]string{"community": "…", "pro": "…"}),
    },
})
if err != nil { return err }              // only caller faults: validation, missing key, 401, 422, canceled ctx
if resp.Failed() { return ErrDegraded }   // 429/5xx/5xx after retries → ErrorAnswer rows (fail open)
ship, _ := jevm.AsNoul(resp.Answers["ship"])
```

The library never reads env, never applies thresholds, never picks prompts — apps hand over a key and get typed answers; `jevm.ToEnvelope(resp)` gives the wire shape if wanted.

## Shape

### Module layout

```
github.com/Dahep/jevm          (one module, ONE binary named `jev`; go 1.25+;
                                require github.com/modelcontextprotocol/go-sdk v1.7+)
├── client.go   types.go   envelope.go   errors.go   ← package jevm: the thin SDK (repo root)
├── cmd/jev/main.go                                ← dispatch: verbs | serve | version (stdlib flag only)
├── internal/verbs/     verify screen classify decide compare review gate
│                       noul choice score   policy.go (threshold ladder, defaults table)
├── internal/cli/       flags, state loading, envelope→stdout, exit codes
├── internal/mcpserver/ mcp.AddTool registrations, typed args, error semantics
└── .github/workflows/release.yml
```

Call chains are three hops either way: `cmd/jev → internal/cli → internal/verbs → jevm.Client` and `cmd/jev → internal/mcpserver → internal/verbs → jevm.Client`. CLI and MCP never build requests themselves; `internal/verbs` is the single source of truth for prompts, thresholds, and verdicts.

### SDK types (repo root, `package jevm`) — thin, faithful to the verified contract

```go
type Model string
const ModelJev113 Model = "jev-1.13"          // the ONLY value Evaluate accepts; jev-1.13-free
                                              // unreachable by construction (Validate rejects it,
                                              // and no --model flag exists to undermine it)
const DefaultEndpoint = "https://opencode.ai/zen/v1/systemone"

type Client struct {
    Endpoint string       // "" ⇒ DefaultEndpoint
    APIKey   string       // required; Bearer. Library never reads env or files (boundary).
    Model    Model        // "" ⇒ ModelJev113
    HTTP     *http.Client // optional; default = retry transport (2 retries on 429/529,
                          // Retry-After honored, 250ms→2s backoff, ctx-bounded, 20s timeout)
}
func New(apiKey string, opts ...Option) *Client

// Evaluate returns a non-nil error ONLY for caller faults: request validation, missing
// key, 401, 422, canceled ctx. Availability failures (429/529/5xx/network after retries)
// NEVER return as error — every question gets an ErrorAnswer row (fail open).
func (c *Client) Evaluate(ctx context.Context, req Request) (*Response, error)

type Request struct {
    State     State     // string | JSON object | JSON array — text only (§1.3)
    Questions Questions // map[qid]Question; answers come back under the same ids
}
type State struct{ v any }
func StringState(s string) State
func JSONState(v any) (State, error)          // rejects non-object/array at construction

type Questions map[string]Question
type Question interface{ questionType() string }
type NoulQuestion   struct{ Instructions string; Criteria *NoulCriteria } // optional {true,false}
type ChoiceQuestion struct{ Instructions string; Criteria map[string]string } // ≤255 (constructor rejects)
type ScoreQuestion  struct{ Instructions string; Criteria []string }          // 2–10 (constructor rejects)
func Noul(instructions string) Question
func Choice(instructions string, criteria map[string]string) (Question, error)
func Score(instructions string, levels []string) (Question, error)
// v1: instructions accept string only (contract also allows object/array — deferred; flagged)

type Response struct {
    Model   string              // "jev-1.13.0" as returned by the wire — never hard-asserted
    Answers map[string]Answer   // keyed by question id
    Usage   Usage               // {input_tokens, output_tokens} — faithful
}
type Answer interface{ answerType() string }
type NoulAnswer   struct{ Noul float64 }        // 0.95/0.05 confident; ~0.5 not; no confidence field on wire
type ChoiceAnswer struct{ Choice string; Probabilities map[string]float64; Confidence float64 }
type ScoreAnswer  struct{ Score float64; Legend []string; Probabilities map[string]float64; Confidence float64 }
// ErrorAnswer is synthesized by this SDK only — the wire never produces it. Fail open is a
// TYPE a caller can switch on, not a nil-check or a crash.
type ErrorAnswer struct{ Kind ErrorKind; Message string } // rate_limited|overloaded|server|network
func (r *Response) Failed() bool
func AsNoul(a Answer) (float64, bool)             // typed accessors; no type-switch tax
func AsChoice(a Answer) (ChoiceAnswer, bool)
func AsScore(a Answer) (ScoreAnswer, bool)
```

### Envelope — the unified shape (CLI stdout, MCP structured content, library projection)

```go
type Envelope struct {
    Schema  int              `json:"schema"`   // 1
    Answers []EnvelopeAnswer `json:"answers"`  // sorted by question id — deterministic
    Usage   EnvelopeUsage    `json:"usage"`
}
type EnvelopeAnswer struct {
    ID            string             `json:"id"`
    Type          string             `json:"type"`   // noul|choice|score (+ "error" for fail-open rows)
    Value         any                `json:"value"`  // noul: 0.93 · choice: "pro" · score: 3.4 · error: message
    Probabilities map[string]float64 `json:"probabilities"` // noul: derived {true,false}; else as returned
    Confidence    *float64           `json:"confidence"`    // null for noul (its value IS the confidence)
    Action        string             `json:"action,omitempty"` // verbs only; "" on primitives (blessed addition)
}
type EnvelopeUsage struct {
    InputTokens  int     `json:"input_tokens"`
    EstimatedUSD float64 `json:"estimated_usd"` // input_tokens × $0.042/1M (pinned model ⇒ constant)
}
func ToEnvelope(resp *Response) *Envelope // package jevm; verbs wrap it to fill Action
func (e *Envelope) Answer(id string) *Answer
```

The one deliberate union is `value` (number|string per `type`) at this serialization boundary; helpers `ValueString()`/`ValueFloat()` keep consumers honest. Faithful `output_tokens` is dropped in the compact projection (free output, ~20 tokens of accounting noise).

### Verb layer (internal/verbs — where policy lives)

The ladder is one function; error rows and missing answers always map to `escalate`:

```go
// v ≥ autoAt → "auto"; v < blockAt → "block" (screen); v < escalateAt → "escalate"; else "review".
// Ladder input: choice/score → API-native confidence; noul → probabilities["true"] (its p IS the signal).
func Ladder(v float64, p Policy) Action
```

Seven high-level verbs + three primitives (CLI signature · MCP tool name/args · when-to-use line liftable straight into AGENTS.md):

| Verb | CLI signature (canonical) | MCP tool + args | When to use |
|---|---|---|---|
| **verify** | `jev verify --state S\|--state-file F --claim C... [--auto-at .9 --escalate-at .6]` | `verify(state string, claims []string)` | Check concrete claims against evidence before you report them done. |
| **screen** | `jev screen --state ... [--intent S] [--block-at .75 --auto-at .85]` | `screen(state, intent?)` | Gate fetched/untrusted content before reading it into context; block ⇒ discard. |
| **classify** | `jev classify --state ... --label name[:desc]... [--auto-at .85 --escalate-at .5]` | `classify(state, labels [{name,desc}])` | Route a state into exactly one of your labeled buckets. |
| **decide** | `jev decide --state ... --question Q --option o[:desc]... [--no-escape] [--auto-at .8]` | `decide(state, question, options [{name,desc}], no_escape?)` | Pick one named option for a bounded decision; appends an `escalate` escape hatch unless `--no-escape`. |
| **compare** | `jev compare --state ... --question Q --item i[:desc]... [--auto-at .8]` | `compare(state, question, items [{name,desc}])` | Pick the better of 2–3 alternatives against one criterion. |
| **review** | `jev review --state ... --rubric "name: weak \| ok \| solid"... [--auto-at .85]` | `review(state, rubrics [{name,levels[]}])` | Score a diff/output against explicit rubric levels before accepting it. |
| **gate** | `jev gate --state ... --claim C... [--rubric ...] [--auto-at .95]` | `gate(state, claims []string, rubrics?)` | Final done-check: completion claims + quality rubric in one call; stricter than review. |
| **noul** | `jev noul --state ... "instructions"` | `noul(state, instructions)` | Raw yes/no when no verb fits. |
| **choice** | `jev choice --state ... "instructions" --option o[:desc]...` | `choice(state, instructions, options)` | Raw multiple choice when no verb fits. |
| **score** | `jev score --state ... "instructions" --level L... (2–10)` | `score(state, instructions, levels)` | Raw rubric score when no verb fits. |

Internal composition (single source of truth): `verify` = one noul per claim; `screen` = three positively-named nouls (`clean`/`substantive`/`relevant` — high is good everywhere, so one ladder serves all rows); `classify`/`decide`/`compare` = one choice (decide additionally appends an `escalate` option); `review` = one score per rubric; `gate` = one noul per claim + a quality score rubric.

**Exit codes:** `0` envelope emitted (fail-open error rows included — actions say `escalate`); `1` upstream hard error (401/422 or unrecoverable, message on stderr, no envelope); `2` flag/usage error; `3` internal/runtime failure; `4` `--strict` and some row not `auto`. Verdicts travel in the envelope; `--strict` is the explicit opt-in that turns "all rows auto" into a scriptable truth.

**No implicit stdin**: `--state-file -` selects stdin explicitly; the CLI never blocks on a tty.

### MCP server (`jev serve` — internal/mcpserver)

```go
server := mcp.NewServer(&mcp.Implementation{Name: "jev", Version: version}, nil)
type VerifyArgs struct {
    State  string   `json:"state"  jsonschema:"evidence/state to judge (text; JSON may be passed as text)"`
    Claims []string `json:"claims" jsonschema:"claims to verify, one entry per claim"`
}
mcp.AddTool(server, &mcp.Tool{
    Name: "verify",
    Description: "Verify concrete claims against evidence. Use before reporting work done " +
                 "or repeating a source's claims. Returns per-claim noul + action auto/review/escalate.",
}, func(ctx context.Context, req *mcp.CallToolRequest, a VerifyArgs) (*mcp.CallToolResult, jevm.Envelope, error) {
    return nil, verbs.Verify(ctx, client, a), nil   // envelope as structured content
})
// … screen, classify, decide, compare, review, gate, noul, choice, score
return server.Run(ctx, &mcp.StdioTransport{})
```

- Handler signature and `&mcp.StdioTransport{}` are the shapes verified against the go-sdk README (v1.7+).
- **Tool names are bare** — opencode prefixes tools with the server name, so server `jev` + tool `verify` surfaces as `jev_verify` (no double-prefixing; matches the ecosystem convention and the `jev*` permission glob).
- **Threshold stance (decided):** MCP tool args carry NO threshold knobs. Thresholds are control-plane policy set once at server start (`jev serve --auto-at .85 --block-at .5 --escalate-at .3`, defaults baked per verb); the MCP client is the LLM being judged, and a model that can loosen its own gate inverts the gate. CLI flags apply raw because the shell caller is the operator.
- Hard errors (401/422) surface as `IsError` tool results with actionable text; missing `JEV_API_KEY` at server startup is a startup-visible error, not a silent fail-open.

### Distribution and dotfiles wiring

**One binary, `jev`.** (Adjudicated: candidate C proposed naming it `jevm`; rejected — `jev` is the settled name, matches the `jev*` permission glob, and the `bin/jev` collision is self-resolving because the same dotfiles PR deletes the bash script.) One binary = one asset per platform, one attestation chain, zero CLI/MCP version skew; the MCP server is cheap to carry (go-sdk is the only dependency; stdlib `flag`, no cobra).

Release workflow sketch (`.github/workflows/release.yml`, on `v*` tags): matrix `linux/amd64`, `linux/arm64`, `darwin/arm64`; `CGO_ENABLED=0 go build -trimpath -ldflags "-s -w -X main.version=${TAG#v}" -o jev ./cmd/jev`; tar as `jev_<ver>_<os>_<arch>.tar.gz` (binary `jev` at archive root — each token is a mise autodetection signal); `sha256sum` → `jev_<ver>_checksums.txt`; `actions/attest-build-provenance` per artifact (mise verifies attestations by default); publish with the release.

**Dotfiles (separate repo, pure config):** `scripts/setup-jevm.sh` is idempotent — `mise use -g "github:Dahep/jevm@${JEVM_VERSION:-<pin>}"` then assert `jev version` prints the pin (verify end-state, not exit code). opencode.json mcp block as in Call site 3 (shims path is version-stable across pin bumps; verify shim stability at implementation). `bin/jev`, `bin/jev-mcp`, and the old `mcp.jev` + `"jev*": "allow"` blocks are removed in the same dotfiles PR that lands setup-jevm.sh; the `jev*` permission returns when the own server ships.

### Invariants and where they live

- **Never `jev-1.13-free`** → `Validate` rejects every `Model` but `ModelJev113`; no `--model` flag exists.
- **Fail open on 429/529/5xx-class** → `ErrorAnswer` rows in typed `Response.Answers`; `Evaluate`'s doc contract forbids returning availability failures as `error`; the verb ladder forces error rows to `escalate`; retries bounded (2, Retry-After honored, ctx-bounded).
- **Policy in one place** → `internal/verbs` owns the only threshold ladder; CLI and MCP are two thin shells over it; the root package cannot drift into policy.
- **One answer shape** → `ToEnvelope` is the only wire→envelope projection; rows sorted by id; same input + model version ⇒ byte-identical stdout ordering.
- **Per-consumer keys** → the SDK takes `APIKey` as a value; the CLI reads `JEV_API_KEY`/`--key`; opencode's mcp command supplies its own from its auth.json; no keyfile code exists anywhere.

## Synthesis decision

**Base: Candidate C** (glm-5.3-flash runner) — chosen by the cross-judge (kimi-k3) and concurred by the orchestrator after full reads of all three. C was the only candidate that writes down where each invariant lives (Validate owns the model pin, `ErrorAnswer` + the Evaluate contract own fail-open, `internal/verbs` owns policy, sorted rows own determinism), making it the safest base for a future maintainer; its illegal-states-unrepresentable types mean extensions go through constructors that reject bad shapes.

**Grafted from Candidate A (kimi-k3):**
1. Serve-time threshold flags on `jev serve` as the operator control plane — replaces C's tighten-only MCP arg clamping entirely; zero agent-facing threshold surface.
2. The go-sdk handler/transport shapes actually verified against the go-sdk README (`func(ctx, *mcp.CallToolRequest, Args)`, `&mcp.StdioTransport{}`) — C's `ServerSession` param and `mcp.NewStdioTransport()` were unverified variants.
3. `jev version` subcommand and `Envelope.Answer(id)` lookup helper.
4. The find/extract/audit rejection rationale (each is a one-line composition of kept verbs) — preserved as doc so the verb set doesn't re-grow speculatively.

**Grafted from Candidate B (gpt-5.6-luna):**
1. Four-way exit-code separation (usage / upstream / internal distinct) — extended with a fifth code (4) for `--strict` verdict failure, which C had overloaded onto 3.
2. Correct `mise exec -- jev serve` syntax as the alternative opencode wiring (C's `["mise","exec","jevm","--","serve"]` was malformed) while keeping C's don't-bake-versioned-paths rationale.
3. The no-implicit-stdin guarantee stated with its rationale (no accidental blocking reads in agent execution).

**Rejections (the high-signal record):**
- A's 8th verb `rerank` — score-per-item is the score primitive; A's own criterion that killed find/extract/audit kills it too.
- A's fail-open-at-frontends split — re-implements policy in two shells and leaves Go apps to re-derive it; C's typed `ErrorAnswer` in the SDK encodes the settled behavior once.
- A's literal `jev_*` tool names — under opencode's server-name prefixing they would surface as `jev_jev_verify`; bare names + server `jev` compose correctly.
- A's `--model` flag — reopens the never-free invariant that Q10 settled; the pin lives in Validate, not in a flag default.
- A's synthesized noul confidence (`max(p,1−p)`) — uncalibrated invention; the envelope keeps `confidence: null` for noul (API-native) and the ladder consumes `probabilities["true"]` instead; revisit only with live calibration data.
- B's policy types in the root SDK package — leaks policy into the thin library Q18 settled; B also never specified per-verb question batteries (the policy core) and self-contradicted gate's verdict representation without flagging it.
- B's stale `jsonschema:"description=…"` tag format and hedged AddTool signatures — replaced by README-verified shapes (graft 2 from A).
- C's binary rename to `jevm` — transitional collision argument, self-resolving; settled name wins.
- C's tighten-only MCP threshold clamp — monotone-correct but advertises knobs to the judged model and doubles schema surface for no evidenced use case.

**Convergence signal:** all three candidates independently converged on: one binary, verb layer shared by CLI and MCP as single source of truth, official go-sdk, thin repo-root SDK, versioned compact envelope, jev-1.13 hard pin, bounded retry on 429/529 only, mise github-backend distribution. That consensus shape ships.

**Dropouts:** none. **Process note:** the rubric file was not materialized at /tmp/arena-jevm/RUBRIC.md before the judge ran (orchestrator omission); the judge reconstructed R1–R6 from the candidates' mutually consistent self-checks — the reconstruction matches the intended rubric, so the verdict stands, but future arenas should write the rubric file first.

## Tradeoffs accepted

- One binary carrying an MCP server it rarely uses, in exchange for a 3-asset release matrix mise autodetects trivially and zero CLI↔MCP version skew.
- Agents unable to tune thresholds at all (no MCP args), in exchange for a control plane the judged model cannot argue with; loosening requires a PR to the baked defaults, which is the right friction.
- `exit 0` on fail-open rows, forcing scripts to inspect `.action`/`--strict`, in exchange for rate limits never crashing an agent run.
- One additive envelope field (`action`, `omitempty`, verbs only) in exchange for policy being observable and jq-composable; schema stays 1 since nothing has shipped.
- `confidence: null` for noul rows (API-faithful) in exchange for not inventing a metric; the ladder reads `probabilities["true"]` instead.
- String-only CLI state (no implicit JSON sniffing) in exchange for a predictable boundary; structured state is a typed `JSONState` call away for Go apps.
- Prompt iteration requires a release (prompts baked per verb) in exchange for versioned, attested, byte-identical behavior across CLI and MCP.
- No client-side token counting (no vendored tokenizer); 64k/32k limits surface as 422s with an error message pointing at oversized state.
- `estimated_usd` from a hardcoded $0.042/1M constant in exchange for zero config; pricing changes are a release, exactness-seekers read `input_tokens`.
- The SDK returning real errors only for caller faults, in exchange for fail-open being a switchable type (`Failed()`, `ErrorAnswer`) rather than a crash.

## Alternatives considered

1. **Two binaries (`jev` CLI + `jevm` MCP)** — rejected: doubles release assets and mise disambiguation work, invites version skew; nothing about stdio MCP needs a separate process image.
2. **Wrapping `@jkudish/jev-mcp` from Go** — rejected: that is the status quo being replaced (node/npx startup, third-party thresholds and tool semantics).
3. **jkudish 12-tool parity** — rejected: find ≈ rerank-with-criterion, extract is a verify/classify battery (Jev returns typed values, never spans), audit ≈ multi-claim verify; surface area costs more agent confusion than it saves calls.
4. **Policy layer in the SDK (`EvaluateAndGate`, thresholds on Client)** — rejected: leaks prompts into every Go app and makes the library untestable without verb fixtures; policy belongs to `internal/verbs`.
5. **OpenCode plugin tool instead of MCP (research §3.2)** — rejected for this iteration: TS file with env plumbing outside pure config, duplicating a core the CLI needs anyway; its unique value (`tool.execute.before` enforcement) is explicitly deferred.
6. **Single raw `evaluate` tool** — rejected: agents need opinionated verbs with taught descriptions; a raw tool pushes prompt engineering back onto every caller.
7. **HTTP daemon for apps** — rejected (Q3): apps get the native Go library; an HTTP layer would re-expose what the SDK already types.

## Open questions and risks

- First live Zen call must confirm the real success envelope (the one shape research could not verify): response `model` string (`jev-1.13.0` vs `jev-1.13`) and answer-map shape — normalization must not hard-assert on the model field.
- Threshold defaults (0.85/0.9/0.95 ladders above) are asserted, not calibrated; expect a tuning pass after a week of real use.
- Screen's `block` is advisory until the deferred plugin hooks land — is AGENTS.md wording enough, and should `screen` docs tell agents to pre-redact secrets (state leaves the machine; not zero-retention per research §1.5)?
- Is `{env:OPENCODE_API_KEY}` interpolation supported inside opencode.json `mcp.environment`, or only in other config surfaces (research §3.1 showed the pattern but the current shell has the var unset anyway — the recommended bash-c wiring avoids the question entirely)?
- Retry budget (2 retries, ~2s worst case) is untuned against real Zen-side Jev limits (undocumented; TypeSafe's 1200 rpm may not apply through the proxy).
- Is `~/.local/share/mise/shims/jev` stable across mise upgrades (believed yes — shims are the standard activation mechanism), or should the wiring fall back to `mise exec`?
- Should `estimated_usd` be rounded (6 dp) in JSON, and does an optional `JEV_PRICE_PER_MTOK` override earn its existence?
- Multi-question partial responses (endpoint returns some ids, omits others): represent omitted rows as ErrorAnswer(`missing`) or hard-error? Lean missing→escalate (fail open), decide at implementation.

## Next implementation step

Bootstrap the repo (`git init`, go.mod, CI skeleton), then write `package jevm` (types.go, client.go, errors.go, envelope.go) against a golden fixture captured from the first live `jev-1.13` call — with fail-open mapping and `ToEnvelope` unit-tested — since every verb, both shells, and all future apps sit on exactly that leaf.

## Appendix A — settled decision ledger (grilling, 2026-09-30)

| # | Decision | Settled answer |
|---|---|---|
| Q1 | Scrap scope | Scrap bash CLI + third-party MCP entirely; keep a CLI; possibly own MCP server |
| Q2 | First-class consumers | User in a shell; agents likely fine on CLI alone (MCP added in Q14) |
| Q3 | App access style | Native library preferred over HTTP endpoint or per-app plugins |
| Q4 | Forced enforcement | Not this iteration (no plugin `tool.execute.before` hooks yet) |
| Q5 | Toolset | Own core 5–8 tools, not jkudish's 12 verbatim |
| Q6 | Day-one coverage | Full owned set from day one |
| Q7 | Key ownership | Own credential handling — resolved by Q24 as per-consumer keys |
| Q8 | 429/5xx behavior | Fail open |
| Q9 | Language | Compiled binary (Go/Rust shortlist) |
| Q10 | Billing | jev-1.13 paid only; never the free SKU |
| Q11 | Process | Design doc first, code after |
| Q12 | Old grammar | noul:/choice:/score: argument grammar scrapped too |
| Q13 | Language pick | Go (official go-sdk, stdlib HTTP/JSON, single static binary) |
| Q14 | MCP timing | Ship own MCP now (go-sdk makes it thin) |
| Q15 | CLI grammar | Verb-first; primitives remain as low-level subcommands |
| Q16 | Output contract | Compact versioned JSON envelope |
| Q17 | Home | Separate project on the personal git account, own release cycles |
| Q18 | SDK weight | Thin typed client package |
| Q19 | opencode.json cleanup | Full removal of old references now |
| Q20 | Concrete app | None yet; generic SDK |
| Q21 | Name + directory | `jevm` at `~/workspace/personal/jevm`, GitHub `Dahep/jevm` |
| Q22 | Module layout | Single Go module = the SDK at repo root; cmd packages for binaries |
| Q23 | Distribution | mise if possible, else plain release binaries (mise `github:` backend makes both true) |
| Q24 | Keys | Per-consumer: CLI env/flag, opencode supplies its own, apps pass theirs |
| Q25 | Agent guidance | Verbs + when-to-use guidance in AGENTS.md when the MCP ships |
| Q26 | Thresholds | Sane baked defaults, flag override; MCP callers get none |
| Q27 | Design doc home | This repo (`docs/design.md`) |
| Q28 | Sequencing | Old wiring removed immediately; agents accept the gap until jevm v1 |

## Appendix B — fact status

**Verified this session (primary sources fetched 2026-09-30):**
- Official Go MCP SDK `github.com/modelcontextprotocol/go-sdk` v1.7+: `mcp.AddTool` with typed Go structs (jsonschema tags become input schemas), `&mcp.StdioTransport{}`, handler `func(ctx, *mcp.CallToolRequest, Args) (*mcp.CallToolResult, Out, error)` (go-sdk README).
- mise `github:Owner/Repo` backend installs release assets directly; asset autodetection scores os/arch/libc/archive-format tokens; GitHub Artifact Attestations verified by default; the older `ubi:` backend is deprecated (mise backends + github-backend docs).
- `OPENCODE_API_KEY` is not exported in the interactive shell (checked live) — the opencode MCP wiring therefore self-supplies the key from opencode's auth.json in the recommended command, rather than relying on `{env:...}` interpolation.

**Carried from the 2026-09-29 research doc (verified there):** the System One request/response contract §1.3, pricing §1.4 (jev-1.13 $0.042/1M input, output free), endpoint liveness probe §1.2, models.dev absence of jev §2.2, plugin/MCP config surfaces §3.1–3.2, REFLEX/LangChain pattern §3.3–3.4.

**Unverified (resolve at implementation):**
- A successful response from the Zen endpoint (needs a live key) — capture a golden fixture on the first real call.
- `{env:VAR}` interpolation support inside `opencode.json` `mcp.environment` (moot under the recommended bash-c wiring).
- Stability of `~/.local/share/mise/shims/jev` across mise upgrades.
- Zen-side Jev rate limits; threshold defaults are uncalibrated pending real usage.
