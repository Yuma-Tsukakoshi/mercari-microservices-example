# ディレクトリ構成
- serviceディレクトリ内にマイクロサービスがそれぞれ格納されている

- bin/ の役割
  - このリポジトリに同梱している“実行バイナリ置き場”（例: kubectl, kind, istioctl）
  - 開発者ごとの環境差（PATHやバージョン違い）を減らし、同じコマンドで実行できるようにする
  - READMEの make cluster 実行時などに使われる（./script/kubectl などのラッパースクリプト経由）

- pkg/ の役割
  - Goの慣習に則り、「他のサービス/パッケージから再利用される共通ライブラリ」を格納
  - 例: pkg/grpc/*（interceptorやcontext周り）、pkg/logger/*
  - 各サービス共通の土台コードを置く
  - サービス固有の実装は基本ここには入れず、services/* 側に置く

- platform/ の役割
  - アプリを動かすための“基盤コンポーネント”や運用資材を配置
  - 例: 
    - platform/db/: DBサービスの実装 + gRPC + Dockerfile + k8s manifest
    - platform/ingress-nginx/, platform/jaeger/, platform/kiali/: ingress / tracing / observability 系のk8s定義
  - 業務機能マイクロサービス（services/）とは別で、周辺インフラ一式を格納

- tools/ の役割
  - Goの“toolsモジュール”のためのディレクトリで、protocのプラグイン等をgo modでバージョン管理する
  - 例: tools/tools.go で protoc-gen-grpc-gateway をimportし、ツール依存をtools/go.modに閉じ込めて再現性を担保
  - アプリ本体の依存（go.mod）とは分離して管理

# 各サービスの役割


## Gateway
- Kubernetesクラスタの外部と接続する唯一のマイクロサービスであり、プロキシの役割を果たします。
- Authorityサービスから取得できる公開鍵を使って、アクセスリクエストのアクセストークン（JWT）を検証し、認証を行います。
- JSON形式のリクエストをgRPCのProtocol Buffers形式へと変換（トランスコード）します。

## Authority
- 顧客アプリケーションのためにアクセストークン（JWT）を発行する役割を持つサービスです。
- 他のマイクロサービスがアクセストークンの署名を検証できるように、公開鍵のgRPCエンドポイントを提供します。

## Catalog
- CustomerサービスおよびItemサービスからデータを集約し、API利用者が扱いやすい形で提供するマイクロサービスです。
- BFF（Backend For Frontend）的な役割を担っています。

## Customer
- 顧客情報をデータベースに保存し、APIを通じてその情報を提供するマイクロサービスです。

## Item
- 商品情報をデータベースに保存し、APIとして提供するマイクロサービスです。


# make clusterの実行で何が行われるか

1. kindを使ってローカル環境にKubernetesクラスタを立ち上げます。
2. 立ち上げたKubernetesクラスタにIstioをインストールします。
3. /servicesディレクトリ内に存在する各マイクロサービスのDockerイメージをビルドします。
4. ビルドした各マイクロサービスをKubernetesクラスタ上にデプロイします。

この流れで、ローカルKubernetes環境上にIstioと全マイクロサービスが動く状態になります。

---


# make pb の実行で何が行われるか（Protocol Buffers 生成）

`make pb` は、各サービスの `.proto` から **GoのgRPCスタブ / HTTP Gateway向けコード** を生成します。
ローカル環境差分を減らすため、基本は **Docker上のGo環境**で実行されます。

## 1) `pb` ターゲット（Dockerコンテナで `gen-proto` を実行）

Makefile上は以下の流れです：

- `docker run` で `golang:1.19-bullseye` を起動
- リポジトリをコンテナへマウントして、コンテナ内で `make gen-proto` を実行

（ホスト側の `go` / `buf` / `protoc` の状態に依存しないようにするため）

## 2) `gen-proto` ターゲット（依存ツールの準備 → buf generate）

`gen-proto` は以下に依存します（必要なら `bin/` に取得/ビルドされます）：

- `buf`（`bin/buf`）
  - Buf CLI。`buf generate` を実行するためのコマンド。
- `protoc-gen-go`（`bin/protoc-gen-go`）
  - `.proto` からGoのメッセージ型（`*.pb.go`）を生成するプラグイン。
- `protoc-gen-go-grpc`（`bin/protoc-gen-go-grpc`）
  - `.proto` からGoのgRPCスタブ（`*_grpc.pb.go`）を生成するプラグイン。
- `protoc-gen-grpc-gateway`（`bin/protoc-gen-grpc-gateway`）
  - gRPC-Gateway用コード生成プラグイン。`tools/` 配下のtools moduleを使って `go build` されます。

ツールが揃ったら、以下を実行します：

- `buf generate --path ./services/ --path ./platform/db/`
  - `services/*/proto/*.proto` と `platform/db/proto/*.proto` を対象に生成
  - 生成物は各 `proto/` 配下に `*.pb.go`, `*_grpc.pb.go`, `*.pb.gw.go` などとして出力されます

## 補足（よく見るWARN）

- `buf.build/beta/googleapis` のdeprecated警告が出ることがありますが、現状は **生成自体は成功**します（将来的にBuf設定更新が必要になる可能性はあります）。


