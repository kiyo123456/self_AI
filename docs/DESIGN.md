# SelfSearch AI — 設計ドキュメント

このファイルはプロジェクトの全設計決定・ユビキタス言語・アーキテクチャを集約した知識ベースです。
実装中に設計の判断に迷ったときは、必ずここに立ち返ること。

---

## 1. プロジェクトの目的

経験・価値観を RDF トリプルで構造化し、SPARQL + Gemini API によって
「気づいていない強みの発見」や「キャリアの推論」を行う自己分析ツール。
就活ポートフォリオとして、技術力・設計思考・拡張性の3点をアピールする。

---

## 2. ユビキタス言語（確定版）

| 用語 | 定義 |
|------|------|
| 経験（Experience） | ある期間・役割・組織における活動の記録。ユーザーが直接入力する唯一の事実データ |
| 価値観（Value） | 意思決定の基準となる信念・優先事項。ユーザーが直接入力する |
| 推論スキル（InferredSkill） | 経験・価値観のトリプルを SPARQL + Gemini API で分析した結果として導出されるスキル。ユーザーは直接入力しない |
| 強み（Strength） | 複数の InferredSkill と経験の関係性から導出される、より抽象度の高い推論の産物 |
| トリプル（Triple） | `<主語, 述語, 目的語>` の形式で表現された知識の最小単位 |
| 推論セッション（InferenceSession） | ユーザーが「推論を実行」し、InferredSkill / Strength が生成されるまでの一連の操作 |
| 分析クエリ（AnalysisQuery） | 推論結果に対して「私の強みは？」などの問いを投げる操作 |

> スキルは事実ではなく推論の産物。「スキル」という言葉をユーザー入力の文脈で使ってはならない。

---

## 3. 境界づけられたコンテキスト（Bounded Context）

### ナレッジグラフ管理コンテキスト（サポートドメイン）

- 責務：経験・価値観を RDF トリプルとして登録・管理する。オントロジーの整合性を守る番人。
- 主要な概念：Experience, Value, Triple
- ユーザーが入力するのはこのコンテキストに対してのみ

### 自己分析コンテキスト（コアドメイン）

- 責務：経験トリプルを材料に SPARQL + Gemini API で推論を実行し、InferredSkill / Strength を生成・保存する
- 主要な概念：InferenceSession, InferredSkill, Strength, AnalysisQuery
- このシステムの本質的価値を生み出す場所

```
ナレッジグラフ管理 ──[SPARQLで経験・価値観トリプルを提供]──▶ 自己分析
                  ◀──[InferredSkillをOxigraphに保存]──────────
```

---

## 4. 全設計決定の一覧

| 決定事項 | 選択 | 理由 |
|----------|------|------|
| ExperienceのID管理 | MySQLで発番・UUIDをRDF URIとして使用 | AggregateのIDはドメインが持つ責務 |
| スキルの入力方法 | ユーザー入力なし。経験から推論で導出（InferredSkill） | スキルは事実でなく推論の産物 |
| 推論結果の保存 | Oxigraphに保存して再利用 | 可視化・説明可能性のため |
| プロンプトの置き場 | 「何を推論するか」→ LlmPort インターフェース（ドメイン層）、「どう指示するか」→ GeminiAdapter（インフラ層） | LLM を差し替えてもドメイン層が変わらない |
| SPARQLクエリの置き場 | インフラ層の OxigraphRepository 内 | SPARQL は Oxigraph という永続化詳細に依存する |
| InferredSkillの再生成タイミング | 非同期ジョブ（Laravel Queue） | 経験更新後にバックグラウンドで再推論しUXを損なわない |

---

## 5. データ管理方針

```
MySQL（エンティティ台帳）
  experience.id = "550e8400-e29b-41d4-a716"  ← ドメインがIDを発番

Oxigraph（関係性ストア）
  <http://selfserch.ai/experience/550e8400> ex:hasRole ex:Engineer .
  <http://selfserch.ai/experience/550e8400> ex:belongsTo ex:POSSE .

  ↓ 推論セッション実行後
  <http://selfserch.ai/inferred-skill/xxx> ex:inferredFrom <.../experience/550e8400> .
  <http://selfserch.ai/inferred-skill/xxx> rdfs:label "チームマネジメント" .
```

---

## 6. ディレクトリ構成（DDDレイヤード）

```
laravel-app/
├── app/
│   ├── Domain/
│   │   ├── KnowledgeGraph/              # サポートドメイン ★自分で書く
│   │   │   ├── Models/Experience.php
│   │   │   ├── Models/Value.php
│   │   │   ├── ValueObjects/Triple.php
│   │   │   └── Repositories/ITripleRepository.php
│   │   └── SelfAnalysis/                # コアドメイン ★自分で書く
│   │       ├── Models/InferenceSession.php
│   │       ├── ValueObjects/InferredSkill.php
│   │       ├── ValueObjects/Strength.php
│   │       └── Ports/LlmPort.php
│   ├── Application/                     # ユースケース層 ★自分で書く
│   │   ├── KnowledgeGraph/
│   │   │   └── RegisterExperienceUseCase.php
│   │   └── SelfAnalysis/
│   │       └── RunInferenceUseCase.php
│   ├── Infrastructure/                  # AIに任せてよい
│   │   ├── Rdf/OxigraphRepository.php
│   │   └── Llm/GeminiAdapter.php
│   └── Http/Controllers/
├── resources/views/                     # Blade UI（AIに任せてよい）
└── docker-compose.yml
```

### 実装分担の方針

| 層 | 担当 | 理由 |
|----|------|------|
| Domain層 | **自分で書く** | 設計の意図が全て詰まっている。ここを書けば面接で全ての判断を説明できる |
| Application層 | **自分で書く** | ユースケースの責務を体で覚えることが設計思考の訓練になる |
| Infrastructure層 | AIに任せる | SPARQLの構文・Gemini APIの呼び方は技術詳細。設計の説明で問われない |
| UI（Blade） | AIに任せる | HTMLの書き方はDDDのアピールポイントではない |

---

## 7. アーキテクチャ全体像

```
[Blade UI]
  経験入力フォーム
  推論実行ボタン
  分析クエリフォーム
       ↓
[Application層]
  RegisterExperienceUseCase  ← 経験をMySQLに保存 → Oxigraphにトリプルを書き込む
  RunInferenceUseCase        ← 非同期ジョブで推論を実行
  RunAnalysisUseCase         ← InferredSkillに問いを立てる
       ↓
[Domain層]
  Experience / Value         ← 事実データ（ユーザー入力）
  Triple                     ← RDFの最小単位（バリューオブジェクト）
  InferenceSession           ← 推論の実行単位
  InferredSkill / Strength   ← 推論の産物（バリューオブジェクト）
  LlmPort                    ← LLMへの依存をドメインが定義するインターフェース
  ITripleRepository          ← RDFストアへの依存を抽象化するインターフェース
       ↓
[Infrastructure層]
  OxigraphRepository         ← ITripleRepositoryの実装（SPARQL）
  GeminiAdapter              ← LlmPortの実装（Gemini API）
  Laravel Queue              ← 非同期ジョブのランナー
  MySQL                      ← エンティティのID台帳
```

---

## 8. 就活での説明ストーリー

> 「経験・価値観を単なるテキストではなく RDF トリプルで構造化することで、
> LLM が関係性を辿りながら推論できる設計にしました。
> スキルはユーザーが入力するのではなく、経験から推論で導出する設計にしています。
> これにより『私の強みは何か』という問いに対して、根拠となるトリプルとともに答えを生成できます。
> 将来的には外部の職種定義 LOD（Wikidata 等）と owl:sameAs で統合し、
> より客観的なキャリアマッチングに発展させる構想があります。」

アピールできる3点：
- **技術力**：RDF・SPARQL・DDDを正しく理解・実装している
- **設計思考**：なぜその技術・構成を選んだか説明できる
- **拡張性**：将来の発展を見据えた設計になっている

---

## 9. ロードマップ（MVP）

```
Phase 1：Laravelプロジェクト新規作成 + DDDディレクトリ構成セットアップ
Phase 2：docker-compose.yml で Oxigraph + MySQL + Laravel 環境構築
Phase 3：ドメインモデル実装（Triple, Experience, Value, LlmPort, ITripleRepository）
Phase 4：インフラ実装（OxigraphRepository, GeminiAdapter, Laravel Queue）
Phase 5：ユースケース実装（RegisterExperienceUseCase, RunInferenceUseCase）
Phase 6：Blade UI 実装（経験入力 → 推論実行 → 結果表示）
```
