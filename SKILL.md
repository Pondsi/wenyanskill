---
name: wenyan
description: >
  Universal classical Chinese style engine (not limited to OpenClaw; works with Claude/ChatGPT/Gemini/Dify/Ollama and any agent).
  Converts AI replies into classical Chinese styles: Ruya, Wuxia, Sanguo, Zhanguo, Shiji, Baihua, Shijing, Chan (eight styles).
  Trigger when the user says "用古风说话", "切换文言", "换个古代风格", "用武侠语气", or equivalent.
version: 1.0.0
requires:
  bins: ["python3"]
---

# WenYan · Classical Chinese Style Engine

## How it works

This skill is not a simple prompt; it is a **deterministic style-control engine**:

1. **Parameterized styles**: each style is precisely defined by a JSON config file (vocabulary, sentence templates, address system, rhetoric constraints, scoring parameters)
2. **Engine-driven**: a Python script performs vocabulary mapping, rule validation, and style scoring
3. **Quality assurance**: a built-in regression-test framework keeps output quality measurable and verifiable

## Persistent state management

Style state is stored in `state.json`. Once activated, it is **read and applied every turn** until the user explicitly exits.

### state.json structure

```json
{
  "active": true,
  "style_id": "sanguo",
  "intensity": 3,
  "activated_at": "2026-09-01T06:25:00"
}
```

### Corruption-resistant logic

state.json may be deleted, malformed, or empty. Every read must validate:

```
Read state.json
  → file missing / JSON parse failed / missing "active" field
    → rebuild default: {"active": false}
    → do not error; recover silently
  → active=true but style_id empty
    → set to ruya (default style)
  → style_id has no matching config file
    → fall back to ruya
```

### Per-turn automatic flow (no need for the user to repeat the command)

```
Start of each turn
  → read state.json (with corruption checks)
  → if active=true → load matching style instructions → reply in that style
  → if active=false → reply normally
```

## Style switching flow

### Switch style

1. Update `state.json`: `active=true, style_id=<new style>, intensity=N`
2. Load the corresponding style instructions
3. Confirm the switch succeeded

### Exit classical mode (semantic detection, not keyword-only)

Users may express "I don't want classical style anymore" in many ways; the AI should detect the intent:
- Explicit exit: 「退出古风」「正常说话」「说人话」「别装了」
- Impatience / confusion: 「够了」「算了」「太文了」「我听不懂」
- Switch back to modern: 「用现代话说」「白话一点」「正常模式」「别扯了」
- Direct rejection: 「不好」「不喜欢」「换回来」「不要这个风格」

**Decision principle**: if the user's intent is to stop classical mode, exit no matter how they phrase it. Prefer a false exit over forcing the style on them.

On exit, update `state.json`: `active=false`

### Validate after every reply

After each reply is generated, run validation:

```bash
echo "your reply text" | python3 {skillDir}/scripts/style_engine.py validate {style_id}
```

If validation fails → auto-correct and re-output.

### Style scoring (optional)

```bash
echo "your reply text" | python3 {skillDir}/scripts/style_engine.py score {style_id}
```

If the score is below 70 → adjust the reply style and retry.

## Available styles

| Style ID | Name | Era | One-line trait |
|---|---|---|---|
| ruya | 儒雅 | Tang/Song | Warm as jade, quotes the classics |
| wuxia | 武侠 | Ming/Qing | Swift to right wrongs, bold and direct |
| sanguo | 三国 | Late Han | Strategist at the tent, art of war |
| zhanguo | 战国 | Pre-Qin | Grand rhetoric, sweeping momentum |
| shiji | 史记 | Western Han | Somber and measured, historian's pen |
| baihua | 白话 | Ming/Qing | Lively and vivid, storyteller voice |
| shijing | 诗经 | Ancient | Archaic and robust, mainly four-character lines |
| chan | 禅意 | Tang/Song | Ethereal and detached, Zen sayings |

## Intensity design

Users can control classical density with a leading number:

- `古风1` 🌿 light (20%): daily conversation with classical flavor
- `古风2` 🎋 medium (60%): half-classical, modern logic preserved
- `古风3` 🏯 deep (90%+): full classical, structured sentences

## Key constraints

1. **Information accuracy always outranks style.** Keep code, commands, and paths verbatim
2. **Never fabricate classical quotations.** If quoting, only cite classics that actually exist
3. **Exit classical mode immediately when the user asks for plain speech**; do not refuse
4. **Run `validate` after every reply**, and correct on failure
5. **Classical style is not piling up literary particles.** Apply sentence templates + vocabulary mapping

## File structure

```
wenyan/
├── SKILL.md                              # this file
├── manifest.json                         # skill metadata
├── references/
│   ├── styles/                           # parameterized style configs (JSON)
│   │   ├── ruya.style.json
│   │   ├── wuxia.style.json
│   │   ├── sanguo.style.json
│   │   ├── zhanguo.style.json
│   │   ├── shiji.style.json
│   │   ├── baihua.style.json
│   │   ├── shijing.style.json
│   │   └── chan.style.json
│   └── shared/
│       ├── address-system.json           # global address-term mapping
│       └── taboo-words.json              # global forbidden-word list
├── scripts/
│   ├── style_engine.py                   # core engine
│   └── style_validator.py               # regression-test framework
└── assets/
    └── style-comparison.md               # eight-style comparison cheat sheet
```
