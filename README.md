# Kong AI Gateway 2.0 ガードレール デモ環境

Kong AI Summit 2026（2026-11-19）の講演「安全なAI活用を支える AIガードレール実践アーキテクチャ」で使用する、約 3 分のデモ環境です。

アプリを改修せず、**Kong AI Gateway 2.0 の AI Policy だけで**次の 4 つが効くことを、1 本のリクエスト経路で示します。

| # | シナリオ | 使う機能 | 期待する結果 |
| --- | --- | --- | --- |
| ① | 認証・認可 | AI Consumer（api-key）＋ ACL | キーなし 401、権限外 403、許可 200 |
| ② | クレデンシャル・PII 保護 | AI Prompt Guard ／ AI PII Sanitizer | キーは遮断（400）、個人情報はマスクして通過 |
| ③ | 外部 Guardrail 連携 | AI AWS Guardrails → Amazon Bedrock Guardrails | 履歴書の注入文を検知して 403 ＋理由 |
| ④ | トークンの可視化と上限 | Prometheus Policy ／ AI Rate Limiting Advanced | Grafana に Consumer 別トークン数・コスト、上限超過で 429 |

## 構成

```mermaid
flowchart LR
  C[クライアント<br>curl / demo.sh] -->|apikey| K

  subgraph K[Kong AI Gateway 2.0 データプレーン]
    direction TB
    A[認証・認可<br>key-auth + ACL] --> P[クレデンシャル遮断<br>AI Prompt Guard]
    P --> S[PII マスク<br>AI PII Sanitizer]
    S --> G[外部 Guardrail<br>AI AWS Guardrails]
    G --> R[トークン上限<br>AI Rate Limiting Advanced]
  end

  S <--> PII[PII サービス<br>kong/ai-pii-service]
  G <--> BG[Amazon Bedrock<br>Guardrails]
  R --> LLM[Amazon Bedrock<br>Nova Lite]
  K -->|/metrics| PR[Prometheus] --> GR[Grafana OSS]
  CP[Konnect<br>AI Gateway コントロールプレーン] -. 設定配信 .-> K
```

- 設定は Konnect の AI Gateway コントロールプレーンで管理し、`kongctl` の宣言的 YAML（`ai-gateway.yaml`）で反映します。
- データプレーン・PII サービス・Prometheus・Grafana はローカルの Docker Compose で動かします。
- LLM は Amazon Bedrock の **Amazon Nova Lite**（東京リージョン `ap-northeast-1`）です。

## ディレクトリ構成

```text
kong-ai-demo/
├── .env.example        # 環境変数のひな形（これをコピーして .env を作る）
├── .env                # 実際の値（Git 管理外）
├── .gitignore
├── ai-gateway.yaml     # kongctl の宣言的設定（AI Gateway・Model・Consumer・Policy）
├── docker-compose.yaml # データプレーン・PII サービス・Prometheus・Grafana
├── demo.sh             # 当日実行するデモスクリプト
├── certs/              # データプレーン証明書（Git 管理外）
├── prometheus/
│   └── prometheus.yml
└── grafana/
    └── provisioning/datasources/prometheus.yml
```

## 前提

| 区分 | 必要なもの |
| --- | --- |
| Kong Konnect | AI Gateway 2.x が使える組織と、パーソナルアクセストークン（PAT） |
| AWS | 東京リージョンで Nova Lite を有効化、Bedrock Guardrail、デモ専用 IAM ユーザー（下記） |
| ローカル | Docker（Compose v2）、`kongctl` v1.21.0 以降、`openssl`、`curl`、`jq`、AWS CLI v2 |

### AWS 側で用意するもの

| リソース | 内容 |
| --- | --- |
| モデルアクセス | Amazon Nova Lite（`amazon.nova-lite-v1:0`）を有効化 |
| Bedrock Guardrail | `kong-ai-demo-guardrail`。Standard Tier、クロスリージョン推論（APAC Guardrail v1:0）、Prompt attacks を High、拒否トピック `Evaluation Manipulation`（任意）。バージョン 1 を発行 |
| IAM ユーザー | `kong-ai-demo`。`bedrock:InvokeModel`／`InvokeModelWithResponseStream`（Nova Lite）と `bedrock:ApplyGuardrail`（上記 Guardrail）のみ許可 |
| アクセスキー | 上記ユーザーのキーを 1 組発行し、`.env` に記入 |

## セットアップ

詳しい手順は、別途共有している構築手順書（Step 1〜8）を参照してください。ここでは流れだけを示します。

```bash
# 1. 環境変数を用意する
cp .env.example .env && chmod 600 .env
#    KONNECT_TOKEN・AWS のキー・GUARDRAILS_ID などを記入
#    利用者キーは自動生成できる
sed -i "s/^DEV_KEY=$/DEV_KEY=dev-$(openssl rand -hex 8)/; \
        s/^GUEST_KEY=$/GUEST_KEY=guest-$(openssl rand -hex 8)/; \
        s/^LIMITED_KEY=$/LIMITED_KEY=limited-$(openssl rand -hex 8)/" .env
sed -i "s/^HOST_GID=$/HOST_GID=$(id -g)/" .env

# 2. 環境変数を読み込むコマンドを登録する（初回のみ）
echo "alias demo-env='cd $(pwd) && set -a && source .env && set +a && echo \"demo env loaded\"'" >> ~/.bashrc
source ~/.bashrc && demo-env

# 3. データプレーン証明書を作る
mkdir -p certs
openssl req -new -x509 -nodes -newkey rsa:2048 -days 365 \
  -subj "/CN=kong-ai-demo-dp/C=JP" -keyout certs/data-plane.key -out certs/data-plane.crt
chgrp "$(id -g)" certs/data-plane.key && chmod 640 certs/data-plane.key

# 4. Konnect に反映する（AI Gateway・証明書・Model・Consumer・Policy）
kongctl diff  --mode apply -f ai-gateway.yaml --pat "$KONNECT_TOKEN"
kongctl apply -f ai-gateway.yaml --pat "$KONNECT_TOKEN"
#    出力された接続先・ID を .env の AIGW_CONTROL_PLANE / AIGW_TELEMETRY / AI_GATEWAY_ID に記入し、demo-env を再実行

# 5. dev-user と limited を dev-team グループに所属させる（API または Konnect 画面）
API=https://us.api.konghq.com/v1/ai-gateways/$AI_GATEWAY_ID
for c in dev-user limited; do
  curl -s -X PUT "$API/consumers/$c/consumer-groups" \
    -H "Authorization: Bearer $KONNECT_TOKEN" -H 'Content-Type: application/json' \
    -d '{"consumer_groups":["dev-team"]}'
done

# 6. ローカル環境を起動する
docker compose up -d
kongctl get ai-gateway nodes --gateway-name "Kong AI Summit Demo" --pat "$KONNECT_TOKEN"
```

### Grafana

- URL：http://localhost:3000（`admin` / `demo-admin`）
- データソース：Prometheus（`http://prometheus:9090`、自動登録）
- パネルのクエリ例

```promql
# Consumer 別トークン数（直近 5 分）
sum by (consumer) (increase(kong_ai_llm_tokens_total{consumer!="", token_type="total_tokens"}[5m]))

# Consumer 別の推定コスト（USD、直近 15 分）
sum by (consumer) (increase(kong_ai_llm_cost_total{consumer!=""}[15m]))
```

## デモの実行

```bash
demo-env
./demo.sh
```

`Enter` を押すたびに次のリクエストに進みます。

| ステップ | 期待する結果 |
| --- | --- |
| ① 1-1 キーなし | `HTTP 401` |
| ① 1-2 guest | `HTTP 403` |
| ① 1-3 dev-user | `HTTP 200` |
| ② 2-1 クレデンシャル | `HTTP 400`（`prompt pattern is blocked`） |
| ② 2-2 PII マスク | 応答内の名前・メール・電話番号がプレースホルダに置換 |
| ③ 3-1 履歴書の注入文 | `HTTP 403`（`GUARDRAIL_BLOCKED` と理由） |
| ④ 4-1 | Grafana で Consumer 別トークン数を表示 |
| ④ 4-2 limited 連続リクエスト | 数回の `HTTP 200` の後に `HTTP 429` |

4-2 のトークン上限（300 トークン／分）は 1 分ごとにリセットされます。リハーサル直後に本番を行う場合は 1 分以上空けてください。

## 環境変数（`.env`）

| 変数 | 内容 |
| --- | --- |
| `KONNECT_TOKEN` | Konnect の PAT |
| `AI_GATEWAY_ID` | AI Gateway の ID（`kongctl get ai-gateway` で取得） |
| `AIGW_CONTROL_PLANE` / `AIGW_TELEMETRY` | データプレーンの接続先（同上） |
| `HOST_GID` | WSL ユーザーのグループ ID（`id -g`）。証明書の鍵を読ませるために使用 |
| `AWS_REGION` | `ap-northeast-1` |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | IAM ユーザー `kong-ai-demo` のアクセスキー |
| `GUARDRAILS_ID` / `GUARDRAILS_VERSION` | Bedrock Guardrail の ID とバージョン |
| `DEV_KEY` / `GUEST_KEY` / `LIMITED_KEY` | デモ用 AI Consumer の API キー |
| `AWS_PAGER` | 空にすると AWS CLI の結果がページャーなしで表示される（任意） |

## 既知の注意点

- **PII サービスは英語版（`-en`）を使う**：Kong は英語での解析を依頼するため、`-ja` イメージだと 500 になります。日本語の個人情報は外部 Guardrail で補う前提です。
- **Prometheus の Consumer 別集計**：Prometheus Policy に `per_consumer: true` が必要です。無いと `consumer=""` になります。
- **メトリクス名**：`kong_ai_llm_tokens_total` のように `kong_` が付きます。
- **入力トークン数**：Bedrock 経由では `prompt_tokens` がリクエストあたり 1 と記録される事象を確認しています。グラフは `total_tokens` か `completion_tokens` で見せるのが無難です。
- **AWS CLI の結果が `:` で止まる**：ページャーの表示です。`--no-cli-pager` を付けるか、`.env` に `AWS_PAGER=` を追加してください。

## 費用の目安

リハーサルを含めて AWS の料金は 1 ドル未満の見込みです（Nova Lite：東京で入力 $0.072・出力 $0.288 / 100 万トークン、Guardrails：$0.15 / 1,000 テキストユニット）。固定費はありません。

## 後片付け

```bash
docker compose down -v
kongctl delete -f ai-gateway.yaml --pat "$KONNECT_TOKEN"
```

- IAM ユーザー `kong-ai-demo`（アクセスキーごと）を削除
- Bedrock Guardrail `kong-ai-demo-guardrail` を削除
- Konnect の PAT を無効化

## セキュリティ

- `.env` と `certs/` は絶対にコミットしないでください（`.gitignore` 済み）。
- 誤って push した場合は、ファイルを消すだけでは履歴に残ります。直ちに AWS のアクセスキーと Konnect の PAT を無効化してください。
- `demo.sh` の `AKIAIOSFODNN7EXAMPLE` は AWS 公式のダミーキーです。

## 参考

- [Kong AI Gateway](https://developer.konghq.com/ai-gateway/)
- [AI Gateway 2.x concepts](https://developer.konghq.com/ai-gateway/ai-gateway-v2-concepts/)
- [Manage AI Gateway with kongctl](https://developer.konghq.com/ai-gateway/kongctl/)
- [AI Gateway Policies](https://developer.konghq.com/ai-gateway/policies/)
- [Bedrock provider（Kong）](https://developer.konghq.com/ai-gateway/ai-providers/bedrock/)
- [Amazon Bedrock Guardrails supported languages](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-supported-languages.html)
