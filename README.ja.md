# Nemotron OpenClaw Lab

[English](README.md) | 日本語

CPUのみのIntel NUCで、NVIDIA Nemotronを使う2台目のOpenClawは動くか。本番のMac miniには手を加えず、外部APIにプロンプトを送らない構成で試します。まずはNemotron 3 Nano 4Bに、OpenClaw経由で短いテキストを1件返させます。

> **ステータス：実機の事前確認済み。** 対象NUCのCPU・RAM・ストレージを確認しました。推論性能、エージェントの一往復、NVIDIAのサンドボックスの隔離は未検証です。

## 構成図

```mermaid
flowchart LR
  OP[検証担当者] --> N1
  subgraph NUC[検証用 Intel NUC]
    subgraph NET[隔離した名前空間：ループバックのみ]
      N1[固定版 OpenClaw コンテナ] -->|ネイティブ API| O[固定版 Ollama コンテナ]
      O --> M[Nemotron 3 Nano 4B Q4_K_M を CPU で推論]
    end
    S[NemoClaw と OpenShell] -.->|別工程| O
  end
  subgraph PROD[別系統の本番 Mac mini]
    K[既存 OpenClaw Kuro は変更しない]
  end
```

OpenClawはエージェントの実行基盤、Nemotronはモデルです。NVIDIA OpenShellはサンドボックスのポリシーを適用し、NVIDIA NemoClawはOpenShell上にOpenClawなどをセットアップするリファレンス構成です。OllamaでNemotronを動かすだけでは、OpenShellやNemoClawは導入されません。

| 段階 | 確認すること | 状態 |
| --- | --- | --- |
| 0. 実機の事前確認 | CPU・RAM・ストレージを確認 | 完了 |
| 1. ローカル推論 | Ollamaで短い応答を取得し、メモリと応答時間を計測 | 計画中 |
| 2. 2台目のOpenClaw | 独立した検証用の設定・状態で、ループバック経由のエージェントの一往復を合成テキストで確認 | 計画中 |
| 3. NVIDIAのサンドボックス | ローカル推論先と構造化ツール呼び出しを検証し、合成データで隔離を確認 | 任意・未検証 |

イメージとモデルの事前取得後、使い捨ての2コンテナは隔離した名前空間のループバックだけを共有します。ポート公開も、ホストパスやコンテナエンジンのソケットのマウントも行いません。本番機・その資格情報・既存連携は対象外です。クラウド推論・個人データ・メッセージ連携・外部公開ポート・本番デプロイ・自動ジョブは使いません。短文が返っても、サンドボックスの隔離や本番運用に適していることの証明にはなりません。

- [NUC検証手順](docs/nuc-poc.ja.md) | [English](docs/nuc-poc.md)
- [アーカイブと変更の経緯](archive/README.md) | [旧プロジェクト](archive/openshell-claude-reviewer/README.ja.md)
- [リポジトリのライセンス](LICENSE)。モデルの重みは同梱しておらず、[NVIDIAの個別ライセンス](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Nano-4B-GGUF)が適用されます。

個人の検証プロジェクトであり、NVIDIAまたはOpenClawの公式配布物ではありません。
