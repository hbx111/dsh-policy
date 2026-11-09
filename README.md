# dsh-policy

A [DeepSeek Harness](https://github.com/deepseek-ai) plugin with two independent jobs:

1. **In-stream refusal detection & auto-retry** — if the model starts its answer with a refusal, drop that attempt and silently re-issue the request.
2. **Policy block injection** *(optional, off by default)* — append a policy text to the outgoing request as a `user` message at the tail.

The retry piece is the headline feature: it keeps streaming intact, which naive proxies can't do.

> 中文简介：在 dsh 客户端侧对模型响应做流内判拒并自动重发。只预读开头一两句，被拒那次不会吐给上层，所以正常回复的流式打字几乎不受影响。

## Why this is not just a proxy

A reverse proxy sees HTTP text only. To detect a refusal it must buffer the whole answer, wait for completion, then decide — which kills streaming. A client plugin receives **structured chunks**, so it can decide after the first few `text-delta`s and discard the rest before the user ever sees them.

```
plugin:  llm/stream (waterfall)  →  peek first N chars  →  refusal?  →  re-issue & swallow
proxy:   full response buffered  →  judge at the end    →  no streaming at all
```

## Features

- **Cheap peek**: only the first `peekChars` (default 200) characters are buffered. Normal answers stream through with a barely noticeable delay.
- **Reasoning-aware**: `reasoning-delta` blocks don't count as "output started". A model may think for thousands of characters before refusing — that case is still caught.
- **Tool-call aware**: if the first block is a tool call, it's not a refusal; no retry.
- **Upstream filter aware**: `content_filter` / `guardrail_intervened` / `risk control` terminations are treated as terminal. Retrying those just gets cut again and burns tokens, so the plugin stops and annotates the failure instead.
- **Bilingual refusal patterns**: matches `Sorry, I can't…`, `抱歉，我不能提供…`, `我无法写这类…`, and Chinese *post-posed* negation such as `继母与继子这类家庭亲属关系的性内容我不写`.
- **Guarded against false positives**: technical phrasing like `这个函数我不写注释` or `这个类的构造器不能写 const` does not trigger a retry.
- **Zero dependencies**: Node built-ins only.

## Install

Drop the plugin directory next to your dsh profile and register it, or install it as a profile bundle (recommended):

```yaml
# profile/cordis.patch.yml — only needed if you are NOT installing as a bundle
- insert:
    - id: dsh-policy
      name: "dsh-policy"
```

As a bundle, `cordis.patch.yml` inside this package already contains that line — don't duplicate it in the profile.

Files are read from `~/.dsh/policy/`:

```
~/.dsh/policy/
├── config.json      # switches (see below)
├── policy-sys.txt   # text injected when mode != off
└── policy.log       # decision log, capped at 256 KiB
```

## Configuration

`~/.dsh/policy/config.json`

```json
{
  "mode": "off",
  "onlyAgentLoop": true,
  "log": true,
  "retry": {
    "enabled": true,
    "maxRetries": 2,
    "peekChars": 200,
    "peekMs": 1200,
    "minTextChars": 24,
    "textStallMs": 1500,
    "maxWaitMs": 10000,
    "backoffMs": 350
  }
}
```

| Key | Default | Meaning |
| --- | --- | --- |
| `mode` | `off` | Policy injection target: `off` \| `user` \| `system` \| `both`. `user` is the only position reliably delivered to the model. **Retry is independent of this switch.** |
| `onlyAgentLoop` | `true` | Inject only on requests that carry tool definitions; leave title-generation and other internal calls alone. |
| `log` | `true` | Append decisions to `policy.log`. |
| `retry.enabled` | `false` | Master switch for refusal retry. |
| `retry.maxRetries` | `2` | How many times to re-issue after a refusal. |
| `retry.peekChars` | `200` | Characters buffered before deciding. |
| `retry.peekMs` | `1200` | Max wait for the first characters. |
| `retry.minTextChars` | `24` | Below this, keep waiting (a two-character answer is not a signal). |
| `retry.textStallMs` | `1500` | If text stops growing for this long, give up waiting and pass through. |
| `retry.maxWaitMs` | `10000` | Hard cap on the peek window. |
| `retry.backoffMs` | `350` | Delay before re-issuing. |

The plugin re-reads `config.json` when its mtime changes, so switches take effect without a restart. **Changing `index.js` itself does require restarting dsh.**

## How detection works

`looksRefusalHead(text)` takes the first 200 characters and tests them against a set of patterns. They fall into four groups:

1. **Leading apology + inability** — `抱歉，我不能提供…`, `Sorry, I can't…`
2. **Leading inability + action** — `我不能写…`, `我无法生成…`
3. **Demonstrative + inability** — `这个我不能写…`, `这类内容我无法生成…`
4. **Post-posed negation** — `…这类内容的我不写`, `这条线我换掉了`

Group 4 exists because Chinese frequently puts the negation *after* the topic. A rule set that only anchors on `抱歉` / `我不能` at position 0 silently misses those.

Patterns are deliberately narrow: `(?=$|[。！？!?，,、；;—…])`-style end anchors and a whitelist of object words (`这类`, `题材`, `内容`, `关系`) keep technical prose out.

## Tests

```bash
node test-retry.mjs
```

16 cases, no network, no dsh process needed — every case drives a fake chunk stream through the real listener:

```
PASS 判定：开口就拒的形状命中
PASS 判定：正常内容不误判
PASS 第一次被拒 → 丢弃并重发，用户只看到第二次
PASS 先推理很久、正文只蹦出两个字就拒绝 → 仍要抓到
PASS 正文起头就断流 → 不挂死，按断流放行
PASS 一直拒 → 用完次数后把最后一次原样交出
PASS 开头是工具调用 → 不重试
PASS 上游 content_filter 切断 → 不重试，失败信息带注
...
16 项通过
```

## Limitations

- Retry is a **statistical** fix. If the model refuses every time for a given input, retrying twice just costs tokens; the last attempt is returned unchanged.
- Detection is prefix-based. A refusal that appears after several paragraphs of normal text is not caught.
- The peek window adds latency proportional to `peekMs` on slow first tokens — lower it if you care more about time-to-first-character than detection accuracy.

## License

MIT
