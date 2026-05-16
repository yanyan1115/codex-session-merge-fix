# Codex Session Merge Fix

修复 Codex Desktop 在 API 登录和 ChatGPT/GPT 订阅登录之间切换后，历史 session 不能合并显示的问题。

## 现象

你可能会遇到这些情况：

- API 登录能看到一批历史 session，切到 ChatGPT/GPT 订阅登录后只剩另一批。
- ChatGPT/GPT 订阅登录能看到旧记录，但 API 登录历史列表空白。
- 切换登录方式后，只显示刚刚新建的那个 session。
- session 文件还在 `%USERPROFILE%\.codex\sessions`，但 Codex Desktop 侧边栏不显示。

## 根因

很多情况下，session 并没有丢失，而是 `model_provider` 元数据分裂了。

例如，旧 session 写成：

```json
"model_provider":"OpenAI"
```

新 session 写成：

```json
"model_provider":"openai"
```

Codex Desktop 的历史列表会按当前 provider 过滤。于是 `OpenAI` 和 `openai` 可能被当成两套历史，导致 API 登录和订阅登录看到的 session 不一致。

## 推荐方案

统一使用 Codex 内置 OpenAI provider id：

```toml
model_provider = "openai"
```

如果 API 模式需要自定义接口地址，不要自定义 `[model_providers.openai]`，也不要写成 `[model_providers.OpenAI]`。推荐使用：

```toml
model = "gpt-5.5"
model_provider = "openai"
openai_base_url = "https://YOUR_API_BASE_URL"
```

ChatGPT/GPT 订阅登录 profile 推荐使用：

```toml
model = "gpt-5.5"
model_provider = "openai"
```

订阅登录 profile 不要设置 `openai_base_url`。

## 注意事项

- 操作前请完全退出 Codex Desktop。
- 操作前请备份 `.codex` 里的配置和数据库。
- 不要把 token、API key、OAuth 内容贴到 issue、论坛或聊天记录里。
- `openai` 是 Codex 内置保留 provider id，不要用 `[model_providers.openai]` 覆盖它。
- 本教程主要面向 Windows 上的 Codex Desktop。其他系统路径不同，但思路相同。

## 1. 备份

PowerShell:

```powershell
$stamp = Get-Date -Format "yyyyMMdd-HHmmss"
$backup = Join-Path $env:USERPROFILE ".codex\backups\provider-unify-$stamp"
New-Item -ItemType Directory -Force -Path $backup | Out-Null

Copy-Item -LiteralPath "$env:USERPROFILE\.codex\config.toml" -Destination (Join-Path $backup "config.toml") -ErrorAction SilentlyContinue
Copy-Item -LiteralPath "$env:USERPROFILE\.codex\state_5.sqlite" -Destination (Join-Path $backup "state_5.sqlite") -ErrorAction SilentlyContinue
Copy-Item -LiteralPath "$env:USERPROFILE\.codex\session_index.jsonl" -Destination (Join-Path $backup "session_index.jsonl") -ErrorAction SilentlyContinue

if (Test-Path "$env:USERPROFILE\.cc-switch\cc-switch.db") {
  Copy-Item -LiteralPath "$env:USERPROFILE\.cc-switch\cc-switch.db" -Destination (Join-Path $backup "cc-switch.db")
}

$backup
```

## 2. 检查 provider 是否分裂

需要本机能运行 `sqlite3`。

PowerShell:

```powershell
sqlite3 "$env:USERPROFILE\.codex\state_5.sqlite" "select model_provider, hex(model_provider), count(*) from threads group by model_provider, hex(model_provider) order by count(*) desc;"
```

如果看到类似结果，说明 provider 已经分裂：

```text
OpenAI|4F70656E4149|65
openai|6F70656E6169|1
```

理想情况下应该只剩一组：

```text
openai|6F70656E6169|你的 session 总数
```

## 3. 修改 Codex config.toml

打开：

```text
%USERPROFILE%\.codex\config.toml
```

API 登录推荐写法：

```toml
model = "gpt-5.5"
model_provider = "openai"
openai_base_url = "https://YOUR_API_BASE_URL"
```

删除这类自定义 provider 配置：

```toml
[model_providers.OpenAI]
name = "OpenAI"
base_url = "https://YOUR_API_BASE_URL"
wire_api = "responses"
requires_openai_auth = true
```

也不要写：

```toml
[model_providers.openai]
```

因为 `openai` 是内置保留 provider id。

## 4. 统一历史记录里的 model_provider

下面脚本会把 SQLite 和 JSONL session 元数据里的 `OpenAI` 统一成 `openai`。

PowerShell:

```powershell
@'
import json
import os
import re
import sqlite3
from pathlib import Path

home = Path(os.environ["USERPROFILE"])
state_db = home / ".codex" / "state_5.sqlite"
sessions_dir = home / ".codex" / "sessions"

with sqlite3.connect(state_db) as con:
    con.execute("update threads set model_provider='openai' where model_provider='OpenAI' collate binary")
    con.commit()

pattern = re.compile(r'("model_provider"\s*:\s*)"OpenAI"')
changed = 0

for path in sessions_dir.rglob("*.jsonl"):
    text = path.read_text(encoding="utf-8", errors="replace")
    if '"model_provider"' not in text:
        continue
    new_text = pattern.sub(r'\1"openai"', text)
    if new_text != text:
        path.write_text(new_text, encoding="utf-8", newline="")
        changed += 1

print(f"Updated JSONL files: {changed}")
'@ | python -
```

## 5. 如果你使用 cc-switch

如果你用 cc-switch 或类似工具管理 Codex 配置，需要保证 API profile 和订阅 profile 也使用同一个 provider id。

API profile 示例：

```toml
model = "gpt-5.5"
model_provider = "openai"
openai_base_url = "https://YOUR_API_BASE_URL"
model_reasoning_effort = "high"
```

ChatGPT/GPT 订阅 profile 示例：

```toml
model = "gpt-5.5"
model_provider = "openai"
model_reasoning_effort = "high"
```

重点：

- 两个 profile 都使用 `model_provider = "openai"`。
- 只有 API profile 设置 `openai_base_url`。
- 订阅 profile 不设置 `openai_base_url`。

## 6. 验证

检查 SQLite provider 分布：

```powershell
sqlite3 "$env:USERPROFILE\.codex\state_5.sqlite" "select model_provider, hex(model_provider), count(*) from threads group by model_provider, hex(model_provider);"
```

检查数据库完整性：

```powershell
sqlite3 "$env:USERPROFILE\.codex\state_5.sqlite" "pragma integrity_check;"
```

应该输出：

```text
ok
```

也可以结构化检查 JSONL 的 `session_meta.payload.model_provider`：

```powershell
@'
import json
import os
from collections import Counter
from pathlib import Path

home = Path(os.environ["USERPROFILE"])
counts = Counter()

for path in (home / ".codex" / "sessions").rglob("*.jsonl"):
    with path.open("r", encoding="utf-8", errors="replace") as file:
        for line in file:
            if '"session_meta"' not in line:
                continue
            obj = json.loads(line)
            counts[obj.get("payload", {}).get("model_provider")] += 1
            break

print(counts)
'@ | python -
```

理想结果类似：

```text
Counter({'openai': 65})
```

## 7. 重启 Codex Desktop

完全退出 Codex Desktop，再重新打开。

如果 app-server 进程还在后台，旧配置可能会被缓存。重启后再切换 API 登录和 ChatGPT/GPT 订阅登录，历史列表应该显示同一套 session。

## 给 AI 的快速诊断提示

如果你想让 AI 帮你排查，可以直接发这段：

```text
请检查 Codex Desktop 历史 session 是否因为 model_provider 大小写不一致而分裂。
重点检查：
1. %USERPROFILE%\.codex\config.toml 里的 model_provider / openai_base_url
2. %USERPROFILE%\.codex\state_5.sqlite 的 threads.model_provider 分布
3. %USERPROFILE%\.codex\sessions 下 JSONL 的 session_meta.payload.model_provider
4. 如果使用 cc-switch，检查 API profile 和订阅 profile 是否都使用 model_provider = "openai"

目标：把 API 登录、订阅登录、SQLite、JSONL 的 provider id 统一成小写 openai。
API profile 可使用 openai_base_url，订阅 profile 不要设置 openai_base_url。
操作前先备份。
```

## 一句话总结

把 API 登录、ChatGPT/GPT 订阅登录、SQLite 历史记录、JSONL session 元数据里的 provider id 全部统一成小写 `openai`。API 模式通过 `openai_base_url` 指向自定义接口，订阅模式不设置 base URL。

## 致谢

感谢 Codex/GPT 协助定位 `model_provider` 分裂问题、验证修复路径并整理这份教程。

感谢 Claude 参与前期排查和交接，为后续定位提供了线索。
