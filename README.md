# serene-pub-gguf-list

The curated list of recommended GGUF models for [Serene Pub](https://github.com/doolijb/serene-pub).

Serene Pub fetches [`recommended.yaml`](recommended.yaml) at runtime to populate the **Recommended Models** sections of its Ollama and KoboldCpp managers, so users can find and pull a model that fits their hardware without going repo-hunting on Hugging Face.

This list lives in its own repository so it can be updated as new models are released, without shipping a new build of the app.

## How it is consumed

The app fetches this file directly from `main`:

```
https://raw.githubusercontent.com/doolijb/serene-pub-gguf-list/main/recommended.yaml
```

Two code paths read it, and they behave differently:

| | Ollama manager | KoboldCpp manager |
|---|---|---|
| Fields used | all of them | `name`, `pull`, `recommended_vram`, `parameter_size`, `description` |
| Caching | none | 1 hour, in memory |
| Hugging Face lookup | no | yes — each entry is re-resolved against the HF API |
| Description shown | the one in this file | HF's own, falling back to this one |

**Changes are live as soon as they land on `main`.** There is no release step. A KoboldCpp user may wait up to an hour for the cache to expire.

### The KoboldCpp resolution step

The KoboldCpp path does not download from `pull` directly. It searches the Hugging Face API for the `name`, takes the top few results by download count, and picks the first public, non-gated repo that has at least one usable quant file.

A file counts as a usable quant only if the **last hyphen-separated segment** of its name matches `/^(Q|IQ|BF|F)\d/i`:

| Filename | Last segment | Usable? |
|---|---|---|
| `Model-Q4_K_M.gguf` | `Q4_K_M` | yes |
| `Model.i1-Q4_K_M.gguf` | `Q4_K_M` | yes |
| `Model-Q4_K_M-imat.gguf` | `IMAT` | **no** |
| `Model_q4_0-it.gguf` | `IT` | **no** |

**If nothing resolves, the entry silently disappears from the KoboldCpp list** — no error, no log. It will still appear under Ollama. This has bitten the list before: the long-standing `Lewdiculous/L3-8B-Stheno-v3.2-GGUF-IQ-Imatrix` entry was invisible to every KoboldCpp user because all of its files end in `-imat.gguf`. Prefer a publisher with conventional filenames when one is available.

Because resolution goes through a download-sorted search, an entry may resolve to a popular mirror rather than the repo you named. That is harmless — the model still appears, and Ollama still pulls from the repo in `pull`.

## Schema

```yaml
models:
  - name: unsloth/Qwen3.5-9B-GGUF                      # HF repo id, no hf.co/ prefix
    pull: hf.co/unsloth/Qwen3.5-9B-GGUF:Q4_K_M         # Ollama pull ref: hf.co/<name>:<QUANT_TAG>
    size: 5.68                                         # GB on disk for that specific quant
    recommended_vram: 7                                # whole GB; drives the tier badge
    tags: [roleplay, utility, vision]                  # inline list, see vocabulary below
    details:
      parameter_size: 9B
      quantization_level: Q4_K_M
      modified_at: "2026-02"                           # quoted YYYY-MM, the model's release
      description: "Two or three sentences."           # single line, no double quotes inside
```

Entries are sorted ascending by `recommended_vram`.

`recommended_vram` should leave headroom above `size` for the KV cache — roughly 1–3 GB depending on the model's context window.

### Tier badges

The UI derives its tier label purely from `recommended_vram`. The `# NGB VRAM Tier` comments in the file are for human curators only, so keep them consistent with this mapping:

| `recommended_vram` | Badge |
|---|---|
| ≤ 3 | Ultra Budget |
| ≤ 6 | Budget |
| ≤ 10 | Mainstream |
| ≤ 16 | High-End |
| > 16 | Enthusiast |

### Tags

`tags` is descriptive metadata for filtering. Stick to this vocabulary so filtering stays predictable:

- **Role** — `roleplay`, `storytelling`, `utility`, `structured-output`, `summarization`, `extraction`
- **Modality** — `vision`, `audio`, `tool-calling`, `long-context`
- **Reasoning** — `thinking-default-on`, `thinking-default-off`, `thinking-capable`, `no-thinking`
- **Build** — `abliterated`, `uncensored`, `imatrix`, `qat`, `legacy`

`long-context` means 128k tokens or more.

The reasoning tags are the important ones and the distinction is deliberate:

- `no-thinking` — the model has no reasoning mode at all
- `thinking-default-off` — it has one, but does not use it by default
- `thinking-default-on` — it reasons unless explicitly told not to
- `thinking-capable` — reasoning-derived, exact default behaviour unverified

This matters because Serene Pub cannot disable thinking on every backend. Ollama disables it by default and KoboldCpp exposes a toggle, but llama.cpp, OpenAI-compatible and LM Studio connections have no way to turn it off. A `thinking-default-on` model on one of those backends will emit reasoning traces into character replies, and will break the JSON extraction pipelines outright.

Verify reasoning behaviour from the base model rather than assuming — roleplay finetunes are sometimes built on reasoning bases. Leave a tag off rather than guess.

## Editing rules

`recommended.yaml` is **not** parsed with a YAML library. Both consumers use hand-rolled line-by-line string parsing, so valid YAML is necessary but not sufficient:

1. **`description` must be one physical line.** No `>` or `|` block scalars, no wrapping. A wrapped line is silently dropped.
2. **No `"` inside a description.** Double quotes are stripped globally from the value.
3. **No trailing `#` comments on a value line** — they get swallowed into the value.
4. **Keep `tags` inline** (`tags: [a, b]`). A block list would put `- item` lines inside the entry body, and entry boundaries are detected by looking for lines starting with `- name:`.
5. **Only the documented keys are read.** Anything else — including `tags` today — is ignored by the app. Adding a field the app acts on requires a matching change in Serene Pub.
6. **The `pull` tag must match a real file** in the repo. Note that `mradermacher` imatrix repos need an `i1-` prefix, e.g. `i1-Q4_K_M`.

## Curation criteria

- **Roleplay and creative writing first.** This is what the app is for. Uncensored and NSFW-capable entries are welcome; describe them factually rather than advertising the capability.
- **Utility models are a distinct class.** Serene Pub routes summarization, character extraction and narrative-graph building to separately configurable connections, and roleplay finetunes do badly at them — drifting into prose instead of emitting JSON. Utility entries should be strong at instruction-following and safe on the reasoning front. Lead their `description` with `Utility model -` so they are distinguishable in the UI, which has no tag filtering yet.
- **Cover the hardware range.** Roughly 3 GB through 24 GB, with no large gaps between tiers.
- **Prefer active, high-traffic repos.** Check monthly downloads before adding. Entries below a few hundred downloads a month are usually dead weight.
- **One quant per model,** chosen so `size` fits comfortably under `recommended_vram`. `Q4_K_M` is the default choice; `Q5_K_M` or QAT builds are reasonable when quality matters more than footprint.
- **Prune as you add.** The list is a recommendation, not an archive. Models superseded by a newer generation should go.

## Before opening a pull request

- Confirm the file still parses: `python3 -c "import yaml; yaml.safe_load(open('recommended.yaml'))"`
- Confirm every `description` is a single line with no inner double quotes
- Confirm entries are still sorted by `recommended_vram`, and that each `size` is comfortably below it
- Confirm each `name` resolves to a public, ungated HF repo whose filenames survive the quant check described above
- Confirm the `pull` tag matches an actual file in the repo, and pull it once to be sure

## License

MIT — see [LICENSE](LICENSE).
