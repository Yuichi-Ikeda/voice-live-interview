# プロジェクト概要: Voice Live Interview

## 1. プロジェクトの目的

**退職者向けナレッジ継承インタビューシステム**。Azure Voice Live API とアバター技術を活用し、AI インタビュアーが退職者と音声対話を行い、長年の経験・暗黙知・判断の勘所を引き出して後任が再現可能な形で記録する。

## 2. システムアーキテクチャ

```
┌────────────────────────┐         ┌─────────────────────────┐         ┌──────────────────┐
│    Browser (Frontend)  │◄──WS───►│  Python Server (FastAPI)│◄──SDK──►│ Azure Voice Live │
│                        │         │                         │         │     Service      │
│  • マイク音声キャプチャ  │         │  • セッション管理        │         └──────────────────┘
│  • 音声再生             │         │  • Voice Live SDK呼出   │                 │
│  • アバター映像表示      │◄──WebRTC (P2P映像)──────────────────────────────────┘
│  • 設定UI / チャット    │         │  • イベント中継          │
└────────────────────────┘         └─────────────────────────┘
```

### 通信方式

| 経路 | プロトコル | 内容 |
|------|-----------|------|
| ブラウザ ↔ Python サーバー | WebSocket | セッション制御、PCM16 音声送受信、SDP シグナリング中継、チャットメッセージ |
| Python サーバー ↔ Azure | Voice Live SDK (WebSocket内部) | セッション管理、音声転送、イベント処理 |
| ブラウザ ↔ Azure | WebRTC (P2P) | アバター映像の直接ストリーミング |

## 3. 技術スタック

### バックエンド
| 技術 | バージョン/詳細 |
|------|----------------|
| Python | 3.10+ (Docker イメージは 3.12-slim) |
| FastAPI | >= 0.104.0 |
| uvicorn | >= 0.24.0 (ASGI サーバー) |
| azure-ai-voicelive | >= 1.0.0 (Voice Live SDK) |
| azure-identity | DefaultAzureCredential 対応 |
| azure-core | Azure 認証基盤 |
| python-dotenv | 環境変数管理 |

### フロントエンド
| 技術 | 詳細 |
|------|------|
| HTML/CSS/JavaScript | フレームワーク不使用のバニラ実装 |
| Web Audio API | AudioWorklet による 24kHz PCM16 マイク音声キャプチャ |
| WebRTC | アバター映像の P2P ストリーミング |
| MediaSource Extensions | WebSocket モードでの fMP4 映像再生 |

### デプロイ
| 技術 | 詳細 |
|------|------|
| Docker | python:3.12-slim ベース、ポート 3000 |
| 推奨ホスティング | Azure Container Apps |
| コンテナレジストリ | Azure Container Registry |

## 4. ファイル構成

| ファイル | 行数 | 役割 |
|---------|------|------|
| `app.py` | 296行 | FastAPI サーバー本体。WebSocket エンドポイント、セッション管理、静的ファイル配信、ヘルスチェック、設定 API |
| `voice_handler.py` | 795行 | Voice Live SDK セッション管理の中核。音声ストリーミング、アバター WebRTC シグナリング、イベント処理、ファンクションコール |
| `prompt.md` | - | AI インタビュアーのシステムプロンプト（退職者向けナレッジ継承インタビュー指示） |
| `static/index.html` | 507行 | メイン UI ページ。接続設定、会話設定、アバター設定、チャット表示 |
| `static/app.js` | 1504行 | クライアント側ロジック。音声キャプチャ/再生、WebSocket 通信、WebRTC、UI 状態管理 |
| `static/style.css` | 731行 | UI スタイルシート。サイドバーレイアウト、レスポンシブデザイン |
| `requirements.txt` | 8行 | Python 依存パッケージ |
| `Dockerfile` | 12行 | Docker コンテナ設定 |
| `README.md` | - | セットアップ手順・アーキテクチャ説明 |

## 5. 主要機能

### 5.1 音声対話
- ブラウザのマイクから 24kHz PCM16 フォーマットで音声をキャプチャ
- WebSocket 経由で Python サーバーに base64 エンコード済み音声を送信
- Azure Voice Live API が音声認識 → AI 応答生成 → 音声合成を実行
- 応答音声を PCM16 でブラウザに返却し再生

### 5.2 アバター表示
- **WebRTC モード（デフォルト）**: Azure から直接 P2P でアバター映像をブラウザにストリーミング。ICE サーバー情報と SDP シグナリングを Python サーバー経由で中継
- **WebSocket モード**: fMP4 チャンクを WebSocket 経由で受信し、MediaSource Extensions で再生
- プリビルトアバター（15種類）、フォトアバター（30種類）、カスタムアバターに対応
- フォトアバターではズーム、位置、回転、振幅のリアルタイム調整が可能

### 5.3 音声設定
- **Standard Voice**: Azure TTS 音声（DragonHD 含む）および OpenAI 音声
- **Custom Voice**: デプロイメント ID 指定によるカスタム音声
- **Personal Voice**: パーソナル音声モデル対応
- 音声テンプレチャー、速度の調整可能
- デフォルト音声: `ja-jp-nanami:DragonHDLatestNeural`（日本語・七海）

### 5.4 会話制御
- **ターン検出**: Server VAD / Azure Semantic VAD
- **発話終了検出（EOU）**: Semantic Detection 対応（カスケードモデルのみ）
- **ノイズ抑制 / エコーキャンセル**: サーバーサイドで処理
- **フィラーワード除去**: Semantic VAD 時に利用可能
- **割り込み（Barge-in）**: ユーザー発話検出時に応答を中断
- **プロアクティブ応答**: 接続直後に AI 側から挨拶を開始

### 5.5 モデル/エージェント対応
- **Model モード**: GPT Realtime、GPT-4.1 系、GPT-4o 系、Phi4 系など複数モデル選択可
- **Agent モード**: Azure AI Foundry エージェントと連携（Agent ID 指定）
- **Agent V2 モード**: エージェント名指定による連携

### 5.6 ファンクションコール
- AI がツールを呼び出す機能をサポート
- 組み込み関数: `get_time`（現在時刻）、`get_weather`（天気）、`calculate`（計算）

### 5.7 開発者モード
- チャット履歴のテキスト表示
- テキスト入力によるメッセージ送信
- デバッグログの表示

## 6. AI インタビュアーのプロンプト設計

`prompt.md` に定義された「退職者向けナレッジ継承インタビュアー」としての振る舞い：

### 会話フロー
1. **挨拶・目的説明・同意確認**
2. **役割・担当の概要聴取**（いつ/何を/どの範囲まで）
3. **深掘り**（具体例・判断理由・手順・失敗/回避策）
4. **引継ぎメッセージ**（後任への助言、チェックリスト、注意点）
5. **まとめ確認**（要点箇条書き読み上げ → 追加確認）

### 深掘り技法
- **具体化**: いつ/どこで/誰と/何が起きた/何をした
- **判断理由**: 選択肢、選定理由、判断基準・制約・リスク
- **再現**: 手順、資料/ツール、コツ、落とし穴
- **代替案**: 改善案、後任が最初にやるべきこと

### 制約
- 個人情報・機密情報は求めない
- ハラスメント/不正/安全に関する話題は事実整理に留める
- 医療/法律の助言はしない
- 不明瞭な入力は推測せず聞き返す

## 7. WebSocket プロトコル

### フロントエンド → バックエンド

| メッセージタイプ | 説明 |
|-----------------|------|
| `start_session` | セッション開始（設定情報付き） |
| `stop_session` | セッション停止 |
| `audio_chunk` | マイク音声送信（base64 PCM16） |
| `send_text` | テキストメッセージ送信 |
| `avatar_sdp_offer` | アバター WebRTC の SDP オファー転送 |
| `interrupt` | 現在の応答を中断 |
| `update_scene` | フォトアバターのシーン設定をリアルタイム更新 |

### バックエンド → フロントエンド

| メッセージタイプ | 説明 |
|-----------------|------|
| `session_started` | セッション準備完了 |
| `session_error` | エラー通知 |
| `ice_servers` | アバター WebRTC 用 ICE サーバー情報 |
| `avatar_sdp_answer` | アバター WebRTC の SDP アンサー |
| `audio_data` | AI 応答音声（base64 PCM16, 24kHz） |
| `video_data` | アバター映像チャンク（WebSocket モード時） |
| `transcript_delta` / `transcript_done` | ストリーミング/完了トランスクリプト |
| `text_delta` / `text_done` | ストリーミング/完了テキスト応答 |
| `response_created` / `response_done` | 応答ライフサイクル |
| `speech_started` / `speech_stopped` | ユーザー発話検出 |
| `function_call_started` / `function_call_result` | ファンクションコール通知 |

## 8. 認証方式

| 方式 | 用途 | 優先度 |
|------|------|--------|
| Entra Token | Agent モード（Azure AI Foundry 連携） | 最優先 |
| API Key (Subscription Key) | Model モード（標準的な利用） | 2番目 |
| DefaultAzureCredential | マネージド ID 等（フォールバック） | 3番目 |

## 9. 環境変数

| 変数名 | 説明 | デフォルト値 |
|--------|------|-------------|
| `AZURE_VOICELIVE_ENDPOINT` | Azure AI Services エンドポイント | なし |
| `AZURE_VOICELIVE_API_KEY` | サブスクリプションキー | なし |
| `VOICELIVE_MODEL` | 使用モデル | `gpt-realtime` |
| `VOICELIVE_VOICE` | 音声名 | `ja-jp-nanami:DragonHDLatestNeural` |

## 10. 起動方法

### ローカル実行
```bash
pip install -r requirements.txt
python app.py
# → http://localhost:3000
```

### Docker 実行
```bash
docker build -t voice-live-avatar-python .
docker run --rm -p 3000:3000 voice-live-avatar-python
# → http://localhost:3000
```

## 11. アバター利用可能リージョン

Southeast Asia, North Europe, West Europe, Sweden Central, South Central US, East US 2, West US 2

## 12. 前提条件

- Python 3.10+
- Azure アカウント（Azure AI Services リソース）
- Microsoft Foundry リソース（Voice Live 対応リージョン）
- ブラウザ: マイク/WebRTC 対応のモダンブラウザ
