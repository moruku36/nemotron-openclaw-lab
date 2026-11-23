# Intel NUCでのPoC手順

[ホーム](../README.ja.md) | [English](nuc-poc.md)

NUCのハードウェアの事前確認は済んでいます。ローカル推論の計測とエージェントの一往復は未検証で、導入完了レポートではありません。NVIDIAの4Bモデルカードが対応範囲としているのは英語とプログラミング言語なので、最初は合成した英語の短文を使います。日本語の品質は別に評価します。

## 0. 検証機を確認する

ステージ0でCPUのみのx86-64環境とRAM・ストレージの余裕を確認しました。対象は第11世代Core i5・RAM 32 GBクラスのIntel NUCです。実行前に空きリソースとループバックポートを改めて確認します。承認された手段で導入し、本番設定や認証情報、SSHエージェント転送、共有ワークスペースを持たない非特権の新規環境を使います。公開モデルの取得と推論時の外部送信は別です。クラウドへのサインインやフォールバックは使いません。

## 推奨構成：使い捨ての2コンテナ

OllamaとOpenClawのOCIイメージをダイジェストで固定し、指定モデルとともに先に取得します。オフラインの検証時は、2つの使い捨てコンテナが共有する隔離ネットワーク名前空間でループバックだけを使います。ポート公開、ホストパスやコンテナエンジンのソケットのマウントはしません。一時状態は分離し、CPU・RAM・プロセス数・リクエスト時間を制限し、並列実行はしません。この構成だけではOpenShellによる隔離を検証したことにはなりません。

## 1. ローカル推論を確認する

4BのQ4_K_Mはダウンロード約2.8 GBです。実行時にはコンテキストと処理基盤の追加メモリが必要です。30B-A3Bはトークンごとの有効パラメータが少なくても持つべき重み全体が大きいため、最初は4Bを選びます。CUDAや外付けGPUは前提にせず、`:cloud` タグも使いません。

以下は任意のホスト直接実行による切り分けであり、推奨のOCI構成とは別です。OCIでは事前取得後の隔離した名前空間内で同じ確認をし、外向き通信は再開しません。ホストで実行する場合は11434番が空いているときだけ最初のブロックをNUCで実行し、2つ目を別端末で実行します。ポートを空けるために既存サービスを停止しません。

```bash
OLLAMA_HOST=127.0.0.1:11434 OLLAMA_NO_CLOUD=1 OLLAMA_NUM_PARALLEL=1 ollama serve
```

```bash
ollama pull nemotron-3-nano:4b-q4_K_M
ollama show nemotron-3-nano:4b-q4_K_M
curl --fail-with-body --max-time 180 \
  -H 'Content-Type: application/json' \
  --data '{"model":"nemotron-3-nano:4b-q4_K_M","stream":false,"think":false,"messages":[{"role":"user","content":"Reply with exactly: PONG"}],"options":{"num_ctx":4096,"num_predict":32}}' \
  http://127.0.0.1:11434/api/chat
ollama ps
```

クラウド無効のログ、ループバックだけの待受、モデル読み込み中の `ollama ps` のCPU実行表示を確認します。`message.content` が `PONG` なら合格。初回の応答時間、ロード時間、生成速度、ピークRAMを記録します。必要な場合のみ読み込み済みで比較します。生成速度は `eval_count ÷ (eval_duration / 1e9)`。180秒は今回の上限で性能保証ではありません。タイムアウト、メモリ不足、継続的なスワップで停止し、外部モデルに切り替えません。

## 2. OpenClawエージェントの一往復を確認する

下記の任意のホスト直接実行では、新規の `nemotron-lab` プロファイルを使い、`OPENCLAW_STATE_DIR` や `OPENCLAW_CONFIG_PATH` が上書きしていないことを確認します。書き込み前に設定パスを確認し、既存の認証・チャネル・フォールバックにつながる場合は中断します。`ollama-local` は目印で、APIキーではありません。

```bash
export OLLAMA_API_KEY=ollama-local
openclaw --profile nemotron-lab config file
# 以下を書き込む前に検証用の設定パスであることを確認する。
openclaw --profile nemotron-lab config set models.providers.ollama.baseUrl http://127.0.0.1:11434
openclaw --profile nemotron-lab config validate
openclaw --profile nemotron-lab models list --provider ollama
timeout 180s openclaw --profile nemotron-lab infer model run --local \
  --model ollama/nemotron-3-nano:4b-q4_K_M --prompt 'Reply with exactly: PONG' --json
```

ネイティブURLには `/v1` を付けません。`--local` はGateway不要で、エージェントのツールなしの単発確認です。モデル・空でない応答・ローカル経路を確認します。この単発確認だけでは段階2の合格ではありません。検証専用の状態を使い、合成テキストで実際のOpenClawエージェントの一往復を記録し、目的のローカルモデルに届いたことを確認します。組み込み実行でもよく、Gatewayの常駐は不要です。メッセージ連携・ブラウザ・ファイル操作・コマンド実行・スケジュール・外部フォールバックは無効のままにします。より長いコンテキストも別途測定します。

## 3. NVIDIAのサンドボックスは別工程

Ubuntu 24.04は上流の検証対象ですが、ホスト要件とモデル実行要件は別です。4BはNemoClawの検証済みOllama既定モデル一覧にないため、このCPU NUCでの連携は未検証として扱います。

導入するリリースの手順を確認します。現行版では新規のOllama利用OpenClawサンドボックスにOpenShellプロバイダーを個別に割り当てます。旧版は経路が異なり、ホストのループバックURLや `inference.local` がそのまま通るとは限りません。非WSLオンボーディングでは11435番にトークン付きの全インターフェース待受を作る場合があります。意図した境界に到達範囲を限定できなければオンボーディング前に停止します。ファイアウォールを緩めず、ホストのホーム・認証情報・コンテナエンジンのソケットを渡しません。

オンボーディングで実際の構造化ツール呼び出しを確認してから、無害な合成データだけで検証用ワークスペースとローカル推論先が分離されているか確認します。通常の文章にJSONが出ただけでは合格とせず、失敗したらテキスト推論で止めます。

## 証跡と戻し方

バージョン・モデルのダイジェストと量子化・ピークRAM／スワップ・応答時間・生成速度・待受範囲・段階別の合否を記録し、生ログと機器識別情報は非公開にします。停止するのは今回の使い捨てコンテナ、または検証で前面起動したプロセスだけです。検証用の状態とモデルは本番データと別に扱い、Mac miniは変更しません。

## 参考資料

- [NVIDIA 4B model card](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Nano-4B-GGUF) · [Ollama 4B artifact](https://ollama.com/library/nemotron-3-nano:4b-q4_K_M) · [Ollama local FAQ](https://docs.ollama.com/faq)
- [OpenClaw Ollama provider](https://docs.openclaw.ai/providers/ollama) · [Profile paths](https://docs.openclaw.ai/help/environment) · [Inference CLI](https://docs.openclaw.ai/cli/infer)
- [NemoClaw prerequisites](https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/get-started/prerequisites) · [Local inference options](https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/inference/local-inference/choose-local-inference-server) · [Ollama onboarding](https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/inference/local-inference/set-up-ollama) · [Platform support](https://docs.nvidia.com/nemoclaw/latest/user-guide/openclaw/reference/platform-support)
- [NVIDIA OpenShell architecture](https://docs.nvidia.com/openshell/about/architecture)
