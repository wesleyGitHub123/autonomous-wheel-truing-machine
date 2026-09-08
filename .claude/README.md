# Claude Code entry points for this repo

Two launchers share one repo, one set of committed agents (`.claude/agents/`) and one
settings file (`.claude/settings.json`). The committed agent files carry only model
aliases — `haiku`, `sonnet`, `inherit` — so the same files serve both entries; which
model an alias resolves to is decided by the launching environment, never by the repo.

| role | `model:` in the agent file | first-party serves | OpenRouter serves |
|---|---|---|---|
| firmware-scout | `haiku` | Claude Haiku | z-ai/glm-5.3-flash |
| firmware-worker | `sonnet` | Claude Sonnet | z-ai/glm-5.3-flash |
| spec-reviewer | `inherit` | the session model | z-ai/glm-5.3 |
| bench-verifier | `haiku` | Claude Haiku | z-ai/glm-5.3-flash |

## First-party (Anthropic subscription)

    claude --model sonnet

Always pass `--model`. A bare launch inherits the default saved in
`~/.claude/settings.json`, and that file has held OpenRouter-only slugs before — its
global `model` key was removed for this reason on 2026-09-08. Do not save an OpenRouter
slug there as a global default again; use the session-only save when switching models
from an OpenRouter session.

## OpenRouter (direct environment launch)

The canonical OpenRouter entry is a plain `claude` with the environment exported in the
shell — no wrapper. The key is read from what `ori login` saved
(`~/.ori/credentials.json`); `OPENROUTER_API_KEY` in the environment overrides it. The
key is never embedded in this repo.

PowerShell:

    $key = (Get-Content "$env:USERPROFILE\.ori\credentials.json" | ConvertFrom-Json).key
    $env:ANTHROPIC_BASE_URL = "https://openrouter.ai/api"
    $env:ANTHROPIC_AUTH_TOKEN = $key
    Remove-Item Env:ANTHROPIC_API_KEY -ErrorAction SilentlyContinue
    $env:CLAUDE_CODE_MAX_CONTEXT_TOKENS = "1000000"
    $env:ANTHROPIC_DEFAULT_HAIKU_MODEL = "z-ai/glm-5.3-flash"
    $env:ANTHROPIC_DEFAULT_SONNET_MODEL = "z-ai/glm-5.3-flash"
    claude --model z-ai/glm-5.3

Git Bash:

    key="$(node -e 'let d="";process.stdin.on("data",c=>d+=c).on("end",()=>{try{process.stdout.write((JSON.parse(d).key)||"")}catch(e){}})' < ~/.ori/credentials.json)"
    export ANTHROPIC_BASE_URL="https://openrouter.ai/api"
    export ANTHROPIC_AUTH_TOKEN="$key"
    unset ANTHROPIC_API_KEY
    export CLAUDE_CODE_MAX_CONTEXT_TOKENS=1000000
    export ANTHROPIC_DEFAULT_HAIKU_MODEL="z-ai/glm-5.3-flash"
    export ANTHROPIC_DEFAULT_SONNET_MODEL="z-ai/glm-5.3-flash"
    claude --model z-ai/glm-5.3

Notes:

- `ANTHROPIC_API_KEY` must be unset — with it present, Claude Code sends `x-api-key`
  instead of the `Authorization: Bearer` header OpenRouter wants.
- The worker tier is the one knob: for gnarly work, export
  `ANTHROPIC_DEFAULT_SONNET_MODEL="z-ai/glm-5.3"` instead. That is a shell-level change;
  the repo never changes.
- `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1` (what `ori`'s own launcher sets) is
  only needed to make `/model` list gateway models; the explicit `--model` above and the
  alias remaps both resolve without it (gate-verified 2026-09-08).
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` is optional hygiene; it reduces
  non-essential requests.
- The `ori` tools are not the canonical Claude Code entry. `ori.exe`'s launcher diverges
  from this documented environment — it sets gateway model discovery but not the alias
  remaps, so committed agents silently resolve to the wrong tier.
  `~/.local/bin/ori-claude` still exists and works, but nothing in this repo depends on
  it.

## The model gate

Aliases resolving through `ANTHROPIC_DEFAULT_*_MODEL` is the load-bearing assumption of
the table above. It is proven empirically, not assumed — third-party gateways are
officially unsupported by the Claude Code docs. Re-run this whenever agents error on
model after a Claude Code update:

1. The probes live permanently at `~/.claude/agents/probe-{haiku,sonnet,inherit}.md` —
   trivial one-line briefs carrying `model: haiku` / `sonnet` / `inherit`.
2. Launch the OpenRouter environment above, from `Truing Repo` root.
3. Ask Claude to spawn each probe with a trivial prompt.
4. Read the OpenRouter dashboard activity log — expect `z-ai/glm-5.3-flash`,
   `z-ai/glm-5.3-flash`, `z-ai/glm-5.3` respectively. A probe's behavioral self-report
   is not evidence; the dashboard is.
5. Repeat from first-party (`claude --model sonnet`): the same probe files must simply
   launch with no model errors.

Gate results:

- 2026-09-08, full gate, both entries: OpenRouter direct environment — probe-haiku →
  `z-ai/glm-5.3-flash`, probe-sonnet → `z-ai/glm-5.3-flash`, probe-inherit →
  `z-ai/glm-5.3`, each read from the OpenRouter dashboard, not from a probe's
  self-report. Gateway model discovery was not exported and was not required. First-party
  `claude --model sonnet`: probe-haiku and probe-sonnet launched with no model errors.
  Verified on this machine against the Claude Code version current on 2026-09-08; the
  docs' position that third-party gateways are unsupported still stands — re-run this
  gate after a Claude Code update if agents error on model.
- 2026-09-08, planning session: the committed spec-reviewer (`model: sonnet`, launched
  under the ori.exe gateway environment with no alias remaps) launched and ran without
  a model error — the served slug was not identified.
