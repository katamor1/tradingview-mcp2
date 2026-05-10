# 運用・検証仕様

この文書は TradingView MCP の起動方法、運用上の環境変数、仕様書化時の検証方針をまとめます。外部 API のライブ値は検証対象にしません。

## Runtime Requirements

`pyproject.toml` 上の要件は次の通りです。

- Python: `>=3.10`
- package name: `tradingview-mcp-server`
- console script: `tradingview-mcp`
- runtime dependencies: `mcp[cli]`, `tradingview-ta`, `tradingview-screener`, `feedparser`

ローカル開発では、このリポジトリに `.venv` が存在する場合、次の Python を利用できます。

```powershell
.\.venv\Scripts\python.exe
```

## Installation

公開パッケージとして利用する場合:

```bash
pip install tradingview-mcp-server
```

リポジトリから実行する場合:

```bash
uv sync
uv run tradingview-mcp
```

Windows でローカル checkout を Claude Desktop などへ登録する場合は、仮想環境の Python と `server.py` を直接指定する構成も使えます。

```json
{
  "mcpServers": {
    "tradingview-mcp-local": {
      "command": "C:\\Users\\YourUsername\\tradingview-mcp\\.venv\\Scripts\\python.exe",
      "args": ["C:\\Users\\YourUsername\\tradingview-mcp\\src\\tradingview_mcp\\server.py"],
      "cwd": "C:\\Users\\YourUsername\\tradingview-mcp"
    }
  }
}
```

## CLI

現行 CLI は次の形です。

```bash
tradingview-mcp [stdio|streamable-http] [--host HOST] [--port PORT]
```

### `stdio`

既定 transport です。MCP クライアントが child process として起動し、標準入出力で通信します。

```bash
tradingview-mcp
tradingview-mcp stdio
```

`stdio` では `--host` と `--port` は実質的に使われません。

### `streamable-http`

HTTP transport で起動します。

```bash
tradingview-mcp streamable-http --host 127.0.0.1 --port 8000
```

`--host` の既定値は `HOST` 環境変数、未設定なら `127.0.0.1` です。`--port` の既定値は `PORT` 環境変数、未設定なら `8000` です。

## Environment Variables

### MCP server

| 変数 | 既定 | 用途 |
| --- | --- | --- |
| `HOST` | `127.0.0.1` | `streamable-http` 起動時の host 既定値。 |
| `PORT` | `8000` | `streamable-http` 起動時の port 既定値。 |
| `DEBUG_MCP` | 未設定 | 設定時、cwd、argv、server file path を stderr に出力。 |

### Proxy

Yahoo Finance、Reddit などの HTTP 取得では `proxy_manager.py` の opener を利用します。proxy 情報が揃っていない場合は通常接続へ fallback します。

| 変数 | 既定 | 用途 |
| --- | --- | --- |
| `PROXY_ENABLED` | `true` | `false` にすると proxy 無効扱い。 |
| `PROXY_HOST` | `p.webshare.io` | proxy host。 |
| `PROXY_PORT` | `80` | proxy port。 |
| `PROXY_USERNAME_PREFIX` | 空 | 必須。未設定なら proxy 未設定扱い。 |
| `PROXY_PASSWORD` | 空 | 必須。未設定なら proxy 未設定扱い。 |
| `PROXY_SESSION_MIN` | `1` | sticky session ID の最小値。 |
| `PROXY_SESSION_MAX` | `250` | sticky session ID の最大値。 |

`python-dotenv` が import できる場合は `.env` も読み込まれます。ただし runtime dependency には含まれていないため、運用上は環境変数を主としてください。

## Validation Policy

この仕様書化では、次の方針で検証します。

- 公開 tool/resource の件数と名称は AST で `server.py` から抽出する。
- 既存 unit tests を実行する。
- Markdown 内の相対リンクと参照ファイルの存在を確認する。
- TradingView、Yahoo Finance、Reddit、RSS へのライブアクセスは実施しない。

ライブアクセスを避ける理由は、仕様書の目的が「現行コードの説明」であり、外部データの一時的な成功/失敗に仕様の正しさを依存させないためです。

## AST Extraction Command

PowerShell で tool/resource 一覧を再確認するコマンド例です。

```powershell
$env:PYTHONIOENCODING='utf-8'
@'
import ast
from pathlib import Path

p = Path('src/tradingview_mcp/server.py')
mod = ast.parse(p.read_text(encoding='utf-8'))
counts = {'mcp.tool': 0, 'mcp.resource': 0}

for node in mod.body:
    if not isinstance(node, ast.FunctionDef):
        continue
    decorators = []
    for deco in node.decorator_list:
        func = deco.func if isinstance(deco, ast.Call) else deco
        if isinstance(func, ast.Attribute) and isinstance(func.value, ast.Name):
            decorators.append(f'{func.value.id}.{func.attr}')
    for decorator in decorators:
        if decorator not in counts:
            continue
        counts[decorator] += 1
        print(f'{decorator}\t{node.lineno}\t{node.name}')

print(counts)
'@ | .\.venv\Scripts\python.exe -
```

期待される件数は、現行実装では `mcp.tool: 27`, `mcp.resource: 1` です。

## Test Command

既存テストは pytest で実行します。

```powershell
.\.venv\Scripts\python.exe -m pytest
```

現行リポジトリでは `tests/unit/test_validators.py` が `sanitize_timeframe()` の正常系と fallback を検証しています。外部 API にアクセスする統合テストは、この仕様書化の検証対象外です。

ローカル `.venv` に pytest が入っていない場合、pytest 実行は `No module named pytest` で失敗します。その場合は検証結果に「pytest 未導入」と明記し、仕様書化の補助確認として次の直接 assert を実行します。この補助確認は pytest の代替ではなく、既存 validator 期待値の最小確認です。

```powershell
$env:PYTHONPATH='src'
@'
from tradingview_mcp.core.utils.validators import sanitize_timeframe

assert sanitize_timeframe('1d') == '1D'
assert sanitize_timeframe('1w') == '1W'
assert sanitize_timeframe('1m') == '1M'
assert sanitize_timeframe(' 1D ') == '1D'
assert sanitize_timeframe(' 1W ') == '1W'
assert sanitize_timeframe(' 1M ') == '1M'
assert sanitize_timeframe('5m') == '5m'
assert sanitize_timeframe('15m') == '15m'
assert sanitize_timeframe('1h') == '1h'
assert sanitize_timeframe('4h') == '4h'
assert sanitize_timeframe('invalid', '15m') == '15m'
print('validator assertions passed')
'@ | .\.venv\Scripts\python.exe -
```

## Markdown Link Check

仕様書内の相対リンクは `docs/specification/` 配下に閉じています。追加・変更時は次の観点を確認します。

- `README.md` から `mcp-tools.md`, `internal-design.md`, `operations-and-validation.md` へリンクできること。
- 各仕様書から相互参照している Markdown が存在すること。
- repo root の `README.md` から `docs/specification/README.md` へリンクできること。

PowerShell で存在確認する例:

```powershell
Test-Path docs\specification\README.md
Test-Path docs\specification\mcp-tools.md
Test-Path docs\specification\internal-design.md
Test-Path docs\specification\operations-and-validation.md
```

より厳密に見る場合は、Markdown の相対リンクを抽出し、各 target が存在することを確認します。

```powershell
$env:PYTHONIOENCODING='utf-8'
@'
import re
from pathlib import Path

files = [Path('README.md'), *sorted(Path('docs/specification').glob('*.md'))]
missing = []

for file in files:
    text = file.read_text(encoding='utf-8')
    for match in re.finditer(r'!?.*?\[([^\]]+)\]\(([^)]+)\)', text):
        target = match.group(2).strip()
        if target.startswith(('http://', 'https://', 'mailto:', '#')) or '://' in target:
            continue
        clean = target.split('#', 1)[0]
        if clean and not (file.parent / clean).resolve().exists():
            missing.append((file.as_posix(), target))

if missing:
    raise SystemExit(f'missing markdown links: {missing!r}')
print('relative markdown links exist')
'@ | .\.venv\Scripts\python.exe -
```

## Known Validation Limits

- TradingView の screener 結果、指標名、対応 symbol は外部サービスに依存します。
- Yahoo Finance Chart API の market state、price、52 week high/low は取得時点で変わります。
- Reddit JSON API は subreddit の投稿状況、rate limit、検索仕様に左右されます。
- RSS フィードは provider 側の配信有無や item 形式に左右されます。
- バックテストは Yahoo Finance の OHLCV と現行 strategy 実装に基づく教育目的の結果です。将来の収益性を示しません。
- 現行 tests は validation helper に限られており、MCP tool 全体の回帰テストを網羅していません。
