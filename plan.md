# compromised-packages 実装計画

## 概要

npm/PyPI等のパッケージエコシステムで発見された汚染（マルウェア混入）パッケージの一覧を管理・配信するリポジトリ。Claude CodeのPreToolUse hookと連携し、`npm install` / `pip install` 等の実行前に汚染パッケージを検知・ブロックする。

## 背景

2026年に入り、axios (npm), litellm (PyPI), telnyx (PyPI) など正規パッケージのメンテナーアカウント乗っ取りによる汚染が多発。従来のタイポスクワットだけでなく、正規パッケージ自体が侵害されるケースが増加している。

主要インシデント（2026年）:
- **axios** npm v1.14.1, v0.30.4 — メンテナーアカウント乗っ取り、RATドロッパー (2026/03/30)
- **litellm** PyPI v1.82.7, v1.82.8 — TeamPCPカスケード攻撃 (2026/03/24)
- **telnyx** PyPI v4.87.1, v4.87.2 — WAVステガノグラフィー (2026/03/27)
- **@dydxprotocol/v4-client-js** npm v3.4.1等 — 暗号通貨ウォレットスティーラー (2026/01/27)

## リポジトリ構成（完成形）

```
compromised-packages/
├── compromised-packages.json        # 汚染パッケージ一覧（メインデータ）
├── scripts/
│   └── update-compromised-list.sh   # 週次更新スクリプト
├── com.compromised-packages.weekly-update.plist  # macOS launchdジョブ定義
├── index.html                       # GitHub Pages用ビューア
├── .nojekyll                        # Pages用
├── logs/                            # 実行ログ（gitignore対象）
├── plan.md                          # この実装計画
├── README.md
├── .gitignore
└── .claude/
    └── agents/
        └── security-research.md     # 汚染パッケージ補完調査エージェント
```

ユーザースコープ（リポジトリ外、Claude Code hook）:

```
~/.claude/
├── settings.json                    # PreToolUse hook設定
└── security/
    └── check-package-safety.sh      # hookスクリプト（本リポジトリのJSONを参照）

~/Library/LaunchAgents/
└── com.compromised-packages.weekly-update.plist  # シンボリックリンク
```

---

## TODO

### Phase 1: データ基盤

- [ ] **1-1. compromised-packages.json 作成**
  - 2026年の既知汚染パッケージを初期データとして登録
  - テスト用エントリ（lodash@4.17.21）も含める（テスト完了後に削除）
  - スキーマ:
    ```json
    {
      "last_updated": "YYYY-MM-DD",
      "sources": ["openssf/malicious-packages", "github-advisory-api", "claude-research"],
      "packages": [
        {
          "name": "パッケージ名",
          "ecosystem": "npm|pypi",
          "affected_versions": ["x.y.z"],
          "date_discovered": "YYYY-MM-DD",
          "severity": "critical|high|medium",
          "source": "データソース",
          "description": "概要"
        }
      ]
    }
    ```

- [ ] **1-2. .gitignore 作成**
  - `logs/` を除外

- [ ] **1-3. README.md 更新**
  - プロジェクト概要、データフォーマット、利用方法を記載

### Phase 2: Claude Code hookによるインストール前チェック

- [ ] **2-1. check-package-safety.sh 作成** → `~/.claude/security/`
  - stdinからPreToolUse hookのJSON入力を読み取り
  - `tool_input.command` からインストールコマンドを検知
  - 対象: `npm install`, `yarn add`, `pnpm add`, `pip install`, `pip3 install`, `uv pip install`, `uv add`
  - パッケージ名とバージョンを抽出
  - `~/Desktop/program/compromised-packages/compromised-packages.json` と照合
  - バージョン指定あり + affected_versions一致 → `permissionDecision: "deny"`
  - バージョン指定なし + パッケージ名一致 → `permissionDecision: "ask"`（確認を促す）
  - 一致なし → exit 0（許可）

- [ ] **2-2. settings.json にPreToolUse hook追加** → `~/.claude/settings.json`
  - 既存のNotification/Stop hookに加えてPreToolUse hookを追加:
    ```json
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/security/check-package-safety.sh",
            "timeout": 5
          }
        ]
      }
    ]
    ```

- [ ] **2-3. hookの動作テスト**
  - テスト用汚染エントリ: lodash@4.17.21
  - テストケース:
    1. `npm install lodash@4.17.21` → deny（ブロック）されること
    2. `npm install lodash` (バージョン指定なし) → ask（確認プロンプト）が出ること
    3. `npm install express` → 通常通り許可されること
  - テスト完了後、lodashエントリを compromised-packages.json から削除

### Phase 3: 週次自動更新

- [ ] **3-1. update-compromised-list.sh 作成** → `scripts/`
  - 2段構成で汚染リストを更新:
    - **Step 1: 公開データソースからの自動取得**
      - OpenSSF Malicious Packages (`github.com/ossf/malicious-packages`)
        - shallow clone → `~/.claude/security/malicious-packages/`
        - 週次で `git pull` → OSV形式JSONをパースして npm/pypi の正規パッケージ侵害を抽出
      - GitHub Advisory API (`GET /advisories?type=malware&ecosystem=npm`)
        - GitHubトークン利用（5,000 req/h）
      - OSV.dev API (`POST api.osv.dev/v1/query`)
        - 認証不要、MAL-プレフィックスでマルウェアフィルタ
    - **Step 2: Claude CLIによる補完調査**
      - `@security-research` エージェントでWeb検索
      - 速報で未DB登録のものを補完
    - **Step 3: マージ・コミット・プッシュ**
      - 既存リストと重複排除しつつマージ
      - `last_updated` を更新
      - git commit & push

- [ ] **3-2. security-research エージェント作成** → `.claude/agents/security-research.md`
  - 検索対象: Socket.dev, Snyk, BleepingComputer, The Hacker News, Aikido Intel
  - 対象エコシステム: npm, PyPI, RubyGems, Maven
  - 既存 compromised-packages.json を読み込み、未登録のもののみ追加
  - JSON形式で出力

- [ ] **3-3. launchd ジョブ作成** → `com.compromised-packages.weekly-update.plist`
  - スケジュール: 毎週月曜 09:00
  - 実行スクリプト: `~/Desktop/program/compromised-packages/scripts/update-compromised-list.sh`
  - ログ出力: `~/Desktop/program/compromised-packages/logs/`
  - 既存の `claude-code-releases` の daily-update.plist と同じパターン

- [ ] **3-4. launchd ジョブ登録**
  - `~/Library/LaunchAgents/` にシンボリックリンク作成
  - `launchctl load` で登録
  - `launchctl list | grep compromised` で確認

### Phase 4: GitHub Pages公開

- [ ] **4-1. index.html 作成**
  - compromised-packages.json をブラウザで閲覧できるシンプルなビューア
  - フィルタ機能（ecosystem, severity）

- [ ] **4-2. .nojekyll 作成**

- [ ] **4-3. GitHub Pages有効化**
  - `gh api` でPages設定（mainブランチ / rootディレクトリ）

---

## データソース詳細

| ソース | URL | 認証 | 形式 | 特徴 |
|--------|-----|------|------|------|
| OpenSSF Malicious Packages | `github.com/ossf/malicious-packages` | 不要 | OSV JSON | 15,000件超、マルウェア特化 |
| GitHub Advisory API | `api.github.com/advisories?type=malware` | GitHubトークン | JSON | npmマルウェアに強い |
| OSV.dev API | `api.osv.dev/v1/query` | 不要 | JSON | MAL-プレフィックスでフィルタ |
| Aikido Intel | `intel.aikido.dev` | 不要 | AGPL公開 | 126,000件、14+エコシステム |

## 検証方法

1. hook: lodash@4.17.21 のテスト用エントリで deny/ask/allow の3パターン確認
2. 週次更新: `update-compromised-list.sh` を手動実行 → JSON更新・commit確認
3. launchd: `launchctl list | grep compromised-packages` で登録確認
4. Pages: ブラウザで `https://namikis.github.io/compromised-packages/` にアクセス確認
