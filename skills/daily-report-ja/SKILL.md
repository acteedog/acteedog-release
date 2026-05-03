---
name: daily-report-ja
description: acteedog MCPからアクティビティを取得して日報を生成・保存するスキル。「日報」「daily report」「今日の振り返り」「昨日やったこと」など、日次の活動レポート作成に関するリクエストで使用する。日報の作成、保存、アクティビティの取得・確認にも対応する。
---

# 日報生成スキル

acteedog MCPからアクティビティデータを取得し、構造化された日報をmarkdown形式で生成・保存する。

## ワークフロー

### 1. 対象日の特定

ユーザーのリクエストから対象日を特定する。指定がなければ前日（昨日）をデフォルトとする。

### 2. アクティビティの取得

`get_activities` で対象日のデータを取得する。

- データが存在し件数が十分な場合 → ステップ3へ
- データが不足または存在しない場合 → `trigger_activity_collection` でジョブをトリガーし、`get_job_status` で全ジョブの完了を確認してから再取得する

#### ジョブ完了待ちのパターン

```
1. trigger_activity_collection で全コネクタのジョブIDを取得
2. 15秒程度待ってから全ジョブのステータスを並列で確認
3. 未完了のジョブがあれば再度待機→確認を繰り返す
4. 全ジョブ完了後、get_activities で再取得
```

### 3. データ分析

アクティビティデータは非常に大きくなりうる（数万〜数十万バイト）ため、**必ずコード実行（Bash + python3）を用いて分析する**。直接コンテキストに読み込まない。

#### `get_activities` のレスポンスデータ構造

`get_activities` はキャッシュファイルのパスを返す。ファイル内のJSONは以下の構造：

```jsonc
{
  "start_date": "2026-04-10",
  "end_date": "2026-04-10",
  "total_activities": 82,
  "groups": [              // コンテキスト単位でグルーピングされたアクティビティ
    {
      "activity_count": 11,
      "context_chain": [   // source → channel/repo → thread/PR の階層（要素がnullの場合あり）
        {
          "id": "slack:source",
          "name": "slack:source",          // optional
          "description": "Activity source from Slack",  // optional, null の場合あり
          "url": "https://yourorgname.slack.com"             // optional
        },
        { "id": "slack:channel:CXXXXXXXX", "url": "https://yourorgname.slack.com/archives/CXXXXXXXX" },
        { "id": "slack:thread:CXXXXXXXX:123456.789", "description": "スレッドの先頭メッセージ", "url": "..." }
      ],
      "activities": [      // 個別のアクティビティ
        {
          "description": "発言内容やアクション",  // null の場合あり
          "timestamp": "2026-04-10T06:36:28.542649030+00:00",
          "url": "https://yourorgname.slack.com/archives/..."
        }
      ],
      "relations": {       // ※リストではなくdict
        "activities": { "outbound": [], "referenced_by": [] },
        "contexts": {
          "outbound": [    // このグループから参照している外部コンテキスト
            {
              "connector_id": "github",
              "id": "github:pull_request:yourorgname/reponame:123",
              "resource_type": "pull_request",
              "title": "PR#123: Fix issue with user authentication",
              "url": "https://github.com/yourorgname/reponame/pull/123"
            }
          ],
          "referenced_by": []  // 他のコンテキストからの参照
        }
      }
    }
  ]
}
```

#### 分析の手順

**Step 1: 全体構造の把握**

```python
import json
from collections import Counter

with open(FILE_PATH) as f:
    data = json.load(f)

groups = data['groups']
print(f"Total activities: {data['total_activities']}, Total groups: {len(groups)}")

# ソース別件数
sources = Counter()
for g in groups:
    chain = g.get('context_chain', [])
    if chain and chain[0]:
        sources[chain[0].get('id', '?').split(':')[0]] += g.get('activity_count', 0)
for src, cnt in sources.most_common():
    print(f"  {src}: {cnt}")

# グループ一覧（アクティビティ数降順）
for i, g in sorted(enumerate(groups), key=lambda x: x[1].get('activity_count', 0), reverse=True):
    chain = [c.get('name', c.get('id', '?')) for c in g.get('context_chain', []) if c]
    desc = (g['activities'][0].get('description') or '')[:100] if g.get('activities') else ''
    print(f"[{i}] ({g.get('activity_count',0)}) {' → '.join(chain)}")
    if desc:
        print(f"    {desc}")
```

**Step 2: 重要グループの詳細確認**

以下の基準で重要なグループを特定し、詳細を取得する：

- アクティビティ数が多いグループ（議論の深さを示す）
- GitHub PR レビューなど成果物を伴うもの
- 複数コンテキストにまたがる活動

各グループから取得すべき情報：
- コンテキストチェーン（source → channel/repo → thread/PR）
- 各アクティビティの `description`, `timestamp`, `url`
- PRの場合は `context_chain` 内のPRの `description`（PRの目的・変更内容）
- `relations` による関連コンテキスト

```python
important_indices = [...]  # Step 1 の結果から選定

for idx in important_indices:
    g = groups[idx]
    print(f"\n{'='*80}")
    print(f"[Group {idx}] Activities: {g.get('activity_count', 0)}")

    # コンテキストチェーン
    for c in g.get('context_chain', []):
        if c is None:
            continue
        print(f"  - {c.get('id','?')}: {c.get('name', '')}")
        desc = (c.get('description') or '')[:200]
        if desc:
            print(f"    desc: {desc}")
        if c.get('url'):
            print(f"    url: {c['url']}")

    # リレーション（dict構造に注意）
    rels = g.get('relations', {})
    if isinstance(rels, dict):
        for direction in ['outbound', 'referenced_by']:
            items = rels.get('contexts', {}).get(direction, [])
            for r in items:
                print(f"  relation({direction}): {r.get('title','')} - {r.get('url','')}")

    # アクティビティ
    for a in g.get('activities', []):
        if a is None:
            continue
        desc = (a.get('description') or '')[:300]
        print(f"  [{a.get('timestamp','')}] {desc}")
        if a.get('url'):
            print(f"    url: {a['url']}")
```

**Step 3: 横断的な関連性の分析**

- 同じプロジェクトに属するグループをまとめる
- 定例会議（スタンドアップ等）での「やったこと/今日やること」とSlackスレッドの活動内容を突合
- カレンダーイベントと対応するSlackスレッドの紐付け

### 4. 日報の生成

以下のフォーマットで生成する。

```markdown
## yyyy-mm-dd

### 主要な成果

- **○○の××を推進**: 具体的な成果サマリ（[リンク](URL)）
- ...（3〜5個）

### プロジェクト・活動詳細

#### ○○プロジェクト

プロジェクト全体での活動のサマリ。

- **具体的な活動名**（[スレッド](URL) / [PR](URL)）
  - 自分がどのような発言・貢献をしたかの具体的な記述
  - 技術的な観点での貢献内容

#### 横断・その他

- **活動名**（[リンク](URL)）: 具体的な内容
```

#### 記載の粒度に関する重要な方針

日報の価値は「何をしたか」ではなく「どのように貢献したか」にある。

- **悪い例**: 「PR#123をレビューした」「Slackで議論した」
- **良い例**: 「PR#123のフィーチャーフラグ制御を確認し、◯◯機能の切り替えロジックが仕様通りであることを検証して承認した」
- **良い例**: 「☓☓のデータ構造との整合性の観点から、△△を持つと構造保証が崩れる技術的制約を指摘した」

具体的に：
- PRレビュー → PRの内容（何のPRか）、レビューでの確認ポイント、指摘事項やコメント内容
- Slack議論 → 自分の発言内容から、どのような論点を提示し、どんな提案をしたか
- モブプログラミング → 自分が共有した知見、参照したコード、気づいた点

#### コンテキスト間の関連性

単独のアクティビティではなく、関連するコンテキストをグルーピングする：
- カレンダーのMTG + そのMTGに対応するSlackスレッド → まとめて記載
- 定例会議での計画共有 + 実際の作業スレッド → 一連の流れとして記載

#### URLの記載

すべての引用元にURLを含める。Slackスレッド、GitHub PR、カレンダーイベントなど、アクティビティの `url` フィールドやコンテキストの `url` フィールドを活用する。

### 5. ユーザー確認と保存

生成した日報をユーザーに提示し、確認を得てから `save_daily_report` で保存する。
