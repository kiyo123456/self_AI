# SelfSearch AI — Claude Code 向けプロジェクト概要

## プロダクト理念

「あなたを測る尺度は、ひとつじゃない」

RDF × AI で強み・弱み・現在地を可視化する自己分析ツール。
詳細は `docs/VISION.md` を参照すること。

---

## 設計ドキュメント

- **プロダクト理念・原体験・問題定義・MVP** → `docs/VISION.md`
- **DDD設計・ユビキタス言語・アーキテクチャ・全設計決定** → `docs/DESIGN.md`
- **技術的判断ログ** → `docs/DECISION_LOG.md`
- **面接準備** → `docs/INTERVIEW_PREP.md`

設計・実装の話題が出たときは、必ず `docs/DESIGN.md` と `docs/VISION.md` を参照すること。

---

## 技術スタック

- バックエンド: Laravel 11（PHP 8.3）+ Laravel Blade
- RDFストア: Oxigraph（Docker）
- RDFライブラリ: EasyRdf（PHP）
- LLM: Gemini API（GeminiAdapter としてインフラ層に実装）
- DB: MySQL（エンティティのID管理）
- キュー: Laravel Queue（非同期推論ジョブ）

---

## AI協働スタンス

このプロジェクトは就活ポートフォリオであり、**ユーザーが設計判断を自分の言葉で説明できること**を最優先とする。

- 答えをそのまま出力しない。思考を促すフィードバックを返す
- Domain層・Application層はユーザー自身が書く（設計の意図が詰まっているため）
- Infrastructure層・UI層はAIが実装してよい
- 「なぜそう設計したか」を言語化できるようサポートする
