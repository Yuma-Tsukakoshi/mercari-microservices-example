# ハンズオン仕様書：新規エンドポイント追加 → Kubernetes にデプロイ（実務想定）

このハンズオンは「実務でよくある、API追加の一連作業」を想定しています。
このリポジトリの構成（gRPC + grpc-gateway + kind + Istio）に合わせて、**新規エンドポイントを追加し、ローカル Kubernetes(kind) にデプロイして動作確認**まで行います。

---

### ゴール

- **`GET /catalog/items/mine`** を追加する
- 認証（JWT）が必要で、**自分（JWT の subject）の出品アイテムだけ**返す
- `make` で **Docker build → kind load → k8s apply** まで行い、`curl` で確認する

---

### 事前知識（このリポジトリの流れ）

- 外部からの HTTP は `services/gateway` が受け、grpc-gateway で `services/catalog` 等へ中継します
- HTTP のルーティングは **`.proto` の `google.api.http` アノテーション**で定義されます（例：`services/catalog/proto/catalog.proto`）
- proto の生成は `make pb`（Docker 経由）または `make gen-proto`（ローカル）で行います
- デプロイは `make <service>`（例：`make catalog` / `make gateway`）で行います

---

### 前提（環境）

- `go`, `docker`, `make`, `curl`, `jq` が使える
- ローカルで kind を動かせる

---

### 例題：/catalog/items/mine（自分の出品一覧）

#### 概要（業務背景）

「マイページに “自分が出品した商品一覧” を表示したい。既存の `/catalog/items` は全件を返すので、**ログインユーザーに紐づくものだけを返す API** を追加したい。」

#### 要件

- **HTTP**
  - **`GET /catalog/items/mine`**
  - Header: `authorization: bearer <ACCESS_TOKEN>`
  - 200: 自分のアイテムのみ返す
  - 401: トークンがない/不正
- **gRPC（CatalogService）**
  - `ListMyItems(ListMyItemsRequest) returns (ListMyItemsResponse)`
  - HTTP アノテーションで `/catalog/items/mine` にマッピングする
- **データ仕様**
  - “自分” は `authorization` の JWT を parse して `sub`（subject）を customer_id とみなす
  - 返す Item の `customer_id` は一致していること

#### 非要件（今回はやらない）

- `ItemService` / `DBService` 側にフィルタ API を増やして効率化（今回は Catalog 側でフィルタしてOK）
- ページングやソート

---

### ステップ0：現状の動作確認（ベースライン）

クラスタが無ければ作ります。

```bash
make cluster
```

疎通（README と同等）：

```bash
curl -s -XPOST -d '{"name":"gopher"}' localhost:30000/auth/signup | jq .
TOKEN=$(curl -s -XPOST -d '{"name":"gopher"}' localhost:30000/auth/signin | jq .access_token -r)
curl -s -XPOST -d '{"title":"Keyboard","price":30000}' -H "authorization: bearer $TOKEN" localhost:30000/catalog/items | jq .
curl -s -XGET -H "authorization: bearer $TOKEN" localhost:30000/catalog/items | jq .
```

---

### ステップ1：API を設計して `.proto` に追加する（課題）

#### 変更対象

- `services/catalog/proto/catalog.proto`

#### やること（課題）

- **`CatalogService` に RPC を1つ追加**する
  - RPC 名：`ListMyItems`
  - HTTP：`GET /catalog/items/mine`
- request/response message を追加する（中身は空でもOK）
  - `message ListMyItemsRequest {}`
  - `message ListMyItemsResponse { repeated Item items = 1; }`

#### 完了条件

- `catalog.proto` に新 RPC と message が増えている

---

### ステップ2：proto 生成（課題）

#### やること（課題）

以下のどちらかで生成します（基本は `make pb` 推奨）。

```bash
make pb
```

または（ローカルに buf / plugins が揃っている場合）：

```bash
make gen-proto
```

#### 期待する変化（例）

- `services/catalog/proto/catalog_grpc.pb.go`
- `services/catalog/proto/catalog.pb.gw.go`

が更新される（生成物は基本編集しない）。

---

### ステップ3：Catalog の実装を追加する（課題）

#### 変更対象

- `services/catalog/grpc/server.go`

#### やること（課題）

- `ListMyItems(ctx, req)` を実装する
  - `authorization` が無ければ 401 相当（gRPC なら `codes.Unauthenticated` 推奨）
  - JWT を parse して `sub` を取り出し、`customer_id` として扱う
  - `s.itemClient.ListItems(...)` を呼び、返ってきた items を **customer_id == sub** でフィルタする
  - `ListMyItemsResponse` として返す

#### 実装ヒント

- token 取得は `CreateItem` が参考になります（`AuthFromMD(ctx, "bearer")`）
- 変換（mapping）は `ListItems` / `CreateItem` の戻り組み立てが参考になります

#### 完了条件

- `go build ./...` が通る（生成後の状態で）

---

### ステップ4：Gateway 側の考慮点（確認）

このリポジトリの gateway は `RegisterCatalogServiceHandlerClient` で Catalog の HTTP ハンドラを登録しています。
そのため、**Catalog の `.proto` を増やしたら、gateway も再ビルド & 再デプロイ**が必要です（同じ Go module 内で生成物を import しているため）。

---

### ステップ5：Kubernetes にデプロイ（課題）

#### やること（課題）

変更を反映するため、少なくとも以下をデプロイします。

```bash
make catalog
make gateway
```

（proto を変更した場合、**生成 → ビルド → デプロイ**の順になるよう注意）

#### 完了条件

- Pod が更新されて Ready になる

```bash
./script/kubectl get pods -n catalog
./script/kubectl get pods -n gateway
```

---

### ステップ6：動作確認（受け入れテスト）

#### シナリオ

- ユーザーAで2件出品
- ユーザーBで1件出品
- Aの `GET /catalog/items/mine` は **Aの2件だけ**返す

#### コマンド例

```bash
# user A
curl -s -XPOST -d '{"name":"user-a"}' localhost:30000/auth/signup | jq .
TOKEN_A=$(curl -s -XPOST -d '{"name":"user-a"}' localhost:30000/auth/signin | jq .access_token -r)
curl -s -XPOST -d '{"title":"A-1","price":100}' -H "authorization: bearer $TOKEN_A" localhost:30000/catalog/items | jq .
curl -s -XPOST -d '{"title":"A-2","price":200}' -H "authorization: bearer $TOKEN_A" localhost:30000/catalog/items | jq .

# user B
curl -s -XPOST -d '{"name":"user-b"}' localhost:30000/auth/signup | jq .
TOKEN_B=$(curl -s -XPOST -d '{"name":"user-b"}' localhost:30000/auth/signin | jq .access_token -r)
curl -s -XPOST -d '{"title":"B-1","price":300}' -H "authorization: bearer $TOKEN_B" localhost:30000/catalog/items | jq .

# new endpoint
curl -s -XGET -H "authorization: bearer $TOKEN_A" localhost:30000/catalog/items/mine | jq .
curl -s -XGET -H "authorization: bearer $TOKEN_B" localhost:30000/catalog/items/mine | jq .
```

#### 期待結果（例）

- `TOKEN_A` で叩いた結果に `A-1`, `A-2` は含まれる
- `TOKEN_A` の結果に `B-1` は含まれない（逆も同様）

#### エラー系

```bash
curl -i -s -XGET localhost:30000/catalog/items/mine | head
```

- 401（もしくはそれ相当）が返る（実装のエラーコード設計に依存）

---

### 追加チャレンジ（任意）

- **チャレンジA**: `GET /catalog/items/mine?min_price=...` を追加し、min_price 以上に絞り込む
  - proto に query param を追加し、server 側でフィルタする
- **チャレンジB**: Catalog 側フィルタではなく、`ItemService` と `DBService` に「customer_id 指定の List」を追加して効率化する
  - 影響範囲が広いので、実務寄り（proto変更が複数サービスに波及する）を体験できる

---

### 片付け（必要なら）

```bash
make clean
```


