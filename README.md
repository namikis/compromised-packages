# compromised-packages

npm / PyPI 等のパッケージエコシステムで発見された汚染（マルウェア混入）パッケージの一覧を管理・配信するリポジトリ。

## Pages

- ビューア: https://namikis.github.io/compromised-packages/
- JSON: https://namikis.github.io/compromised-packages/compromised-packages.json

## 用途

Claude Code の PreToolUse hook と連携し、`npm install` / `pip install` 等の実行前に汚染パッケージを検知・ブロックする。

## データフォーマット

```json
{
  "name": "パッケージ名",
  "ecosystem": "npm|pypi|...",
  "affected_versions": ["x.y.z"],
  "date_discovered": "YYYY-MM-DD",
  "severity": "critical|high|medium",
  "source": "データソース",
  "description": "概要"
}
```

## データソース

- [OpenSSF Malicious Packages](https://github.com/ossf/malicious-packages)
- [GitHub Advisory Database](https://github.com/advisories?query=type%3Amalware)
- [OSV.dev](https://osv.dev/)
