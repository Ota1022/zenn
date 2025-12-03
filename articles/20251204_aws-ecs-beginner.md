---
title: "ECSのService ConnectとService Discoveryの違いを理解する"
emoji: "🕸️"
type: "tech"
topics: ["AWS", "ECS"]
published: true
published_at: 2025-12-04 00:00
---

本記事は[若手AWS Leading Engineerを志す者達 Advent Calendar 2025](https://qiita.com/advent-calendar/2025/to-be-japan-aws-jr-champions)の4日目の記事です。

AWS Jr. Champions 2026 を目指すアドカレということで、業務でAmazon Elastic Container Service (ECS) を使ったマイクロサービス環境に触れる中で、Service Connect と Service Discoveryの違いを理解するために調べたことをまとめました。

普段は[スリーシェイク](https://3-shake.com/)という会社でフルスタックエンジニアとしてWebアプリケーション開発に従事しています。会社の方でも[3-shake Advent Calendar 2025](https://qiita.com/advent-calendar/2025/3-shake)で記事を書くのでよければ見ていってください。

## 1. Amazon ECSについて

![Amazon ECS](/images/20251204/Arch_Amazon-Elastic-Container-Service_64.png)

ECSは「**コンテナ化されたアプリケーションを簡単にデプロイ・管理・スケーリングできる、完全マネージド型のコンテナオーケストレーションサービス**」と説明されるのですが、何をどう管理しているのかイメージしにくいと思います。そこで、まずマイクロサービスアーキテクチャという考え方について振り返ります。

### 1.1 マイクロサービスアーキテクチャとは

前提として**モノリシックアーキテクチャ**という考え方があります。これは全ての機能を1つの大きなアプリケーションとして開発・デプロイする方法です。個人開発などでアプリを作る際はここから始めることが多いと思います。
モノリシックアーキテクチャでエンタープライズや複雑な処理の伴うアプリケーションを開発・運用する場合、以下のような課題が生じる可能性があります。

* 一部の機能を修正しただけでも、全体を再デプロイする必要がある
* アプリケーションの規模が大きくなると、開発やテストが複雑になる
* 特定の機能だけスケールさせることが難しい

これらの課題を解決するのが**マイクロサービスアーキテクチャ**で、アプリケーションを小さな独立したサービスの集まりとして構成します。

![モノリシックとマイクロサービスの比較](/images/20251204/monolith_1-monolith-microservices.70b547e30e30b013051d58a93a6e35e77408a2a8.png)
*出典: AWS - モノリシックアーキテクチャとマイクロサービスアーキテクチャの違い*

例としてECサイトを想定します。マイクロサービスアーキテクチャでは機能ごとに以下のようにアプリケーションを分割できます。

* フロントエンドサービス -> ユーザーインターフェース(Next.js)
* ユーザー管理サービス -> 認証やユーザー情報の管理(Go)
* 商品管理サービス -> 商品カタログや在庫管理(Go)
* 注文サービス -> 注文処理や決済(Go)
* レコメンドサービス -> 商品レコメンデーション(Python + 機械学習)

このように分割すると、以下のようなメリットがあります。

* **独立したスケーリング**
セール時に注文が急増した時、レコメンドサービスや商品管理サービスに影響を与えずに注文サービスだけをスケールアップできます。

* **責務分離による柔軟な処理分担**
サービスごとに責務を分けることで重い処理を専用のサービスに切り出しやすくなります。例えば注文サービスは注文完了後すぐにユーザーに応答を返し、画像処理やレポート生成は別のサービスで非同期に実行できます。各サービスを独立してスケールできるため、重い処理が他のサービスに影響を与えません。

* **障害の影響範囲の限定**
レコメンドサービスに障害が起きても、注文処理自体は継続できます。サービスが独立しているため、障害の影響範囲を明確に切り分けやすくなります。

* **技術スタックの柔軟性**
フロントエンドはNext.js、レコメンドサービスは機械学習を使うのでPython、コアなバックエンドサービスはパフォーマンスが重要なのでGo、といったように各サービスで最適な技術を選択できます。

### 1.2 ECSとマイクロサービスの関係

マイクロサービスアーキテクチャの考え方を踏まえてECSの話に戻ります。ECSではこれらのサービスをコンテナとして実行・管理します。これにより以下のようなメリットがあります。

* **環境の独立**：サービスごとに異なる言語やライブラリを使える
* **柔軟なリソース配分**：サービスごとに必要なCPU・メモリを設定できる
* **オーケストレーション**：起動・停止・配置・スケール管理を自動化

冒頭の「コンテナ化されたアプリケーションを簡単にデプロイ・管理・スケーリングできる、完全マネージド型のコンテナオーケストレーションサービス」という説明を噛み砕くと「**小さく分けたサービス(コンテナ)を、自動で良い感じに管理・スケールしてくれるAWSのサービス**」と言えます。

なお、AWSのコンテナオーケストレーションサービスとしては他にAmazon EKS (Elastic Kubernetes Service) もあります。これはKubernetesというオープンソースのオーケストレーションツールをマネージドで提供するサービスです。本記事ではECSに焦点を当てますが、ECSは AWS 独自の仕組みでシンプルに使える点が特徴で、EKSはKubernetesの豊富なエコシステムを活用できる点が特徴です。

### 1.3 ECSの構成要素

ECSがマイクロサービスを管理するサービスだとわかったところで、構成要素を見ていきましょう。

* **Cluster(クラスター)**：コンテナを実行する環境全体の論理的な単位です。
* **Task Definition(タスク定義)**：1つ以上のコンテナをまとめたタスク全体の設計図です。どのコンテナを、どのリソース/ネットワーク/ログ設定で動かすかを指定します。
* **Task(タスク)**：Task Definition を元に起動される実行単位です。タスク内には複数コンテナを含められます。
* **Service(サービス)**：継続的に動かし続けたいアプリケーションを管理する単位です。ECSがタスクを指定数維持し、タスクが落ちたら再起動、負荷に応じてスケール、ロードバランサーとの連携などを自動で行います。

![ECSの構成要素](/images/20251204/ecs-core-component.png)
*出典: AWS ECS Immersion Day*

## 2. マイクロサービス間通信の課題と解決策

マイクロサービスアーキテクチャでは、複数のサービスが互いに通信し合う必要があります。例えば、注文サービスがユーザー管理サービスに認証情報を問い合わせたり、商品管理サービスに在庫を確認したりします。

ここで問題になるのが、**通信先のサービスを見つける方法**です。

通常、サービス間の通信にはIPアドレスとポート番号が必要ですが、ECS上で動くタスクのIPアドレスは固定ではありません。タスクの再起動、スケールアウト・イン、新しいバージョンのデプロイなど、様々な理由でIPアドレスが変わります。

IPアドレスをコードや設定ファイルに直接書き込んでしまうと、タスクが再起動するたびに設定を更新しなければなりません。この問題を解決するためにAWSではいくつか選択肢があるのですが、今回は**Service Connect**と**Service Discovery**に焦点を当てたいと思います。

### 2.1 Service Connect

Service Connectは、ECSサービス同士の通信を標準的に扱えるようにするECSの機能です。**Service Connectを有効化したECSサービス同士**で、短い名前による接続・可観測性の向上・トラフィック制御などの機能を活用できます。

以下の図にある通り、各タスクに**Envoyサイドカー**という補助的なコンテナを自動的に追加することで動作します。Envoyはオープンソースのプロキシロードバランサーで、アプリケーションコンテナからの通信を受け取り、適切な宛先にルーティングする役割を果たしています。

![Service Connectの仕組み](/images/20251204/service-connect.png)
*出典: AWS Builders Flash - Web アプリケーションのアーキテクチャ設計パターン*

アプリケーションコンテナは通信先のIPアドレスを知る必要がなく、Service Connectで接続された他のECSサービスにサービス名だけで通信できます。この際、**通信は必ず自タスク内の Envoy プロキシを経由**します。アプリケーションが短縮名（例: `my-app:8080`）で接続しようとすると、まずタスク内の Envoy がそのリクエストを受け取り、実際の宛先 IP とポートへルーティングする仕組みです。これは VPC の DNS 設定だけで解決される通常の名前解決とは異なります。

Envoyサイドカーがリクエスト数やレイテンシなどのメトリクスを自動的にCloudWatch Metricsに収集するため、サービス間通信の可観測性が向上します。

ただし、**Service Connect の短縮名による接続や可観測性・トラフィック制御といった機能は、Service Connect を設定した ECS タスク間でのみ有効**です。Lambda や EC2 など ECS 外のクライアントから ECS タスクにアクセスさせたい場合は、Service Connect だけでは不十分で、後述する Service Discovery や ALB などの別の接続手段を用意する必要があります。

### 2.2 Service Discovery

Service Discoveryもサービスを名前で見つけられるようにする機能で、ECS専用ではなくEC2、Lambda、Kubernetes(EKS)、オンプレミスなど様々なリソースでも利用できます。内部的にAWS Cloud MapとRoute 53を組み合わせてDNS名で名前解決できるようにしているのですが、VPC側でDNSによる名前解決が使える設定になっている必要があります。

#### AWS Cloud Map とは

ECS タスク、EC2 インスタンス、Lambda などの場所を登録・管理し、他のサービスがそれらを見つけられるようにする仕組みです。以下の2つの方式でサービス検索を提供します。

* **DNS ベース**: Route 53 のプライベートホストゾーンを使って DNS レコードを自動登録。`my-service.local` のような名前で名前解決できる
* **API ベース**: Cloud Map の API を直接呼び出してサービスを検索。DNS が使えない環境でも利用可能

ECS の Service Discovery は、Cloud Map の DNS ベース方式を利用しています。タスクが起動すると自動的に Cloud Map に登録され、DNS 名で名前解決できるようになります。タスクが停止すると自動的に登録解除されるため、常に現在稼働中のタスクだけにアクセスできます。

![Service Discoveryの仕組み](/images/20251204/service-discovery.png)
*出典: AWS Builders Flash - Web アプリケーションのアーキテクチャ設計パターン*

Service Discoveryを使うことで、タスクのIPアドレスが変わっても同じDNS名でアクセスできます。また、ECS以外のクライアント(同一VPCにいるLambdaやEC2、別の内部システムなど)からもECSタスクにアクセスしやすくなります。

### 2.3 使い分けの考え方

使い分けは通信する側と通信される側の組み合わせで決まります。

ECSサービス同士が通信する場合は**Service Connect**を使います。サービス間通信を統一的に扱え、Envoyによる可観測性の向上やトラフィック制御などの恩恵を受けられます。

一方、ECS以外のサービス(LambdaやEC2など)からECSタスクにアクセスする場合は**Service Discovery**を使います。DNSベースの名前解決により、VPC内のどこからでもアクセスできます。

## 3. Terraformでの定義例

ここからは段階的に Terraform での定義を見ていきます。VPC / Subnet / Security Group / IAM ロールなどの周辺設定は省略し、Service Discovery と Service Connect を有効化する時にどこをどう書き足すかに絞ります。

### 3.1 Task 定義と Service

まずはシンプルな構成として、**Cluster、Task Definition、Service** を定義します。この構成をベースに、Service Connect や Service Discovery を追加していきます。

```hcl
# ECS クラスター
resource "aws_ecs_cluster" "main" {
  name = "my-cluster"
}

# タスク定義
# 使うイメージ、CPU/メモリ、ポート番号などを定義
resource "aws_ecs_task_definition" "app" {
  family                   = "my-app"              # タスク定義の名前(バージョン管理される)
  network_mode             = "awsvpc"              # VPC内でIPアドレスを持つモード
  requires_compatibilities = ["FARGATE"]           # サーバーレスで動かす
  cpu                      = "256"                 # 0.25 vCPU
  memory                   = "512"                 # 512 MB

  # コンテナを起動・停止するための権限
  execution_role_arn = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([
    {
      name  = "app"               # コンテナの名前
      image = "my-app:latest"     # 使用するDockerイメージ

      portMappings = [
        {
          containerPort = 8080    # コンテナ内で待ち受けるポート
          protocol      = "tcp"
        }
      ]
    }
  ])
}

# ECS サービス
# タスクの数を常に監視し、指定した数を維持する
resource "aws_ecs_service" "app" {
  name            = "my-app-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2                              # 常に2個のタスクを起動しておく
  launch_type     = "FARGATE"                      # Fargateで起動

  network_configuration {
    subnets         = ["subnet-xxx", "subnet-yyy"] # タスクを配置するサブネット
    security_groups = ["sg-zzz"]                   # タスクに適用するファイアウォールのルール
  }
}
```

### 3.2 Service Connect を追加する

リトライ・タイムアウト・トラフィック分散・メトリクス収集などをECS で標準化してまとめて扱いたい場合は Service Connect を使います。

#### Cloud Map の namespace 作成

まず、Service Connect で使用する Cloud Map の namespace を作成します。

**namespace**とは、Service Connect が内部で使用するサービスの論理的なグルーピング単位です。開発環境・ステージング環境・本番環境などを分離して管理するために使われます。

Service Connect では、namespace に登録されたサービスは **Service Connect の仕組み内で短い名前（例: `my-app:8080`）による接続が可能**になります。ただし、この接続方式は Service Connect を有効化したECSタスク間で動作するもので、通常の DNS 設定とは異なります。

そのため、**ECS 外のクライアント（Lambda や EC2 など）から Service Connect の短縮名を使って到達することはできません**。ECS 外からアクセスさせたい場合は、後述する Service Discovery や ALB などの別の手段が必要です。

```hcl
# Cloud Map namespace (Service Connect 用)
resource "aws_service_discovery_private_dns_namespace" "sc" {
  name = "sc.local"
  vpc  = aws_vpc.main.id
}
```

**補足**: ここでは `aws_service_discovery_private_dns_namespace` リソースを使っていますが、Service Connect はこの namespace を**論理的なグルーピングとして使用**しており、`sc.local` という名前が Route 53 の通常の DNS レコードとして外部から引けるわけではありません。Service Connect における namespace の種別（private DNS / HTTP など）は、主に Cloud Map の管理上の分類であり、Service Connect の短縮名解決の仕組み自体には直接影響しません。実際の名前解決は Envoy プロキシを通じて Service Connect の仕組み内で行われます。

#### ECS クラスターの変更

クラスターに Service Connect のデフォルトnamespace を設定します。

```hcl
resource "aws_ecs_cluster" "main" {
  name = "my-cluster"

  # Service Connect のデフォルトnamespace を追加
  service_connect_defaults {
    # Cloud Map namespace の ARN を指定
    namespace = aws_service_discovery_private_dns_namespace.sc.arn
  }
}
```

#### タスク定義の変更

portMappings に `name` を追加します。この名前は後で Service Connect の設定で使用します。

```hcl
resource "aws_ecs_task_definition" "app" {
  # 他の設定は同じ

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "my-app:latest"

      portMappings = [
        {
          # Service Connect 側の port_name と一致させる
          name          = "http"
          containerPort = 8080
          protocol      = "tcp"
        }
      ]
    }
  ])
}
```

#### ECS サービスの変更

service_connect_configuration ブロックを追加します。

```hcl
resource "aws_ecs_service" "app" {
  # 他の設定は同じ

  # Service Connect の設定を追加
  service_connect_configuration {
    enabled = true  # Service Connect を有効化。各タスクに自動で Envoy サイドカーが追加される

    # このサービスを Service Connect 経由で他のサービスから呼び出せるようにする
    service {
      # タスク定義の portMappings[].name と一致させる(必須)
      port_name      = "http"
      # Service Connect 内での登録名。Cloud Map にも自動登録される
      discovery_name = "my-app"

      # 他のサービスがこのサービスを呼び出す時に使う名前とポート
      # 例: 他のサービスから "my-app:8080" でアクセスできる
      client_alias {
        dns_name = "my-app"
        port     = 8080
      }
    }
  }
}
```

上記のコードの通り、Service Connect の設定は ECS クラスター・タスク定義・サービスの3箇所に分散しています。

* **クラスター**では `namespace` を指定して、短縮名で到達できる論理的なサービスグループを定義
* **タスク定義**では `portMappings[].name` でどのポートを公開するかを宣言
* **サービス**では `service_connect_configuration` で実際の接続設定を行う

なお、**namespace は Service Connect 内での論理的なグルーピングに過ぎず、ネットワーク通信の可否自体は Security Group や NACL などのネットワーク設定で制御されます**。namespace を分けても、セキュリティ境界が自動的に設定されるわけではありません。

特に `portMappings[].name` と `service_connect_configuration.service.port_name` の一致が必須で、ここが食い違うとサービスが正しく動作しません。

もう一つ重要なのは、`discovery_name` で指定した名前が **自動的に Cloud Map にも登録される**ことです。ただし、この登録は Service Connect 専用の namespace 内で行われるため、**Service Connect を有効化したタスク間でのみ名前解決が可能**です。Lambda や EC2 など、Service Connect を使っていないクライアントからはこの名前でアクセスできません。ECS 以外のサービスからアクセスしたい場合は、別途 Service Discovery の設定が必要になります。

#### Envoy の設定について

Service Connect を有効化すると、**Envoy プロキシが自動的にサイドカーコンテナとして追加されます**。Envoy の設定ファイルを自分で書く必要はありません。

* `enabled = true` にするだけで、ECS が各タスクに Envoy コンテナを自動追加
* `service_connect_configuration` ブロックの内容から、Envoy の設定が自動生成される
* リトライ・タイムアウト・ヘルスチェックなどのデフォルト設定が適用される
* アプリケーションコンテナからの通信が自動的に Envoy を経由するようになる

このようにEnvoy の存在を意識せずに使えるのが Service Connect の特徴です。Terraform の `service_connect_configuration` ブロックだけでマイクロサービス間通信の制御ができます。

### 3.3 Service Discovery を追加する

ECS以外のサービス(LambdaやEC2など)からECSタスクにアクセスしたい場合は Service Discovery(Cloud Map + Route 53 による DNS 登録)を使います。

3.1 の基本構成に対して、以下を追加します。

#### Cloud Map の namespace 作成

Service Discovery 用に別の namespace を作成します。こちらは Route 53 のプライベートホストゾーンとして DNS レコードが登録される純粋な DNS の仕組みです。

```hcl
# Cloud Map namespace (Service Discovery 用)
# "my-app.sd.local" の ".sd.local" 部分を定義
resource "aws_service_discovery_private_dns_namespace" "sd" {
  name = "sd.local"
  vpc  = aws_vpc.main.id
}
```

**注意**: Service Connect 用と Service Discovery 用の namespace は分けて定義しています。Service Connect が内部で Cloud Map を使用しますが、その管理は ECS が自動で行うため、手動で Service Connect 用の Cloud Map サービス（`aws_service_discovery_service`）を定義する必要はありません。一方、Service Discovery では明示的に DNS レコードの設定を行う必要があります。

#### Cloud Map サービスの作成

Service Connect では不要でしたが、DNS レコードの詳細な設定を行う必要があります。

```hcl
# Cloud Map の Service (Service Discovery 用)
resource "aws_service_discovery_service" "app" {
  name = "my-app"  # "my-app.sd.local" の "my-app" 部分

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.sd.id

    dns_records {
      ttl  = 10
      type = "A"
    }

    routing_policy = "MULTIVALUE"
  }

  health_check_custom_config {
    failure_threshold = 1
  }
}
```

#### ECS サービスの変更

Service Connect の `service_connect_configuration` に対し、こちらは `service_registries` という別のブロックを使います。

```hcl
resource "aws_ecs_service" "app" {
  # 他の設定は同じ

  # Service Discovery との連携設定を追加
  # タスクのIPアドレスが自動でDNSに登録される
  service_registries {
    registry_arn   = aws_service_discovery_service.app.arn
    container_name = "app"
    container_port = 8080
  }
}
```

Service Discovery の構成を見ると、**Cloud Map のnamespace とサービスを定義し、それを ECS サービスの `service_registries` で参照する**という構造になっています。

Service Connect との違いは、Service Connect が Envoy というプロキシを自動で追加してくれるのに対し、Service Discovery は DNS による名前解決のみを提供する点です。そのため Service Discovery は ECS 以外のサービス（LambdaやEC2など）からも利用できる汎用的な仕組みとなっています。

## 4. まとめ

ECS におけるマイクロサービス間通信の基本として、Service Connect と Service Discovery の 2 つの仕組みを整理しました。

今回は比較する形で紹介してきましたが、あくまでもこれらは**対立する概念ではなく、併用される関係**にあります。

* **ECS サービス間の通信**: Service Connect を使うことで、Envoy による可観測性やトラフィック制御の恩恵を受けられる
* **ECS 以外からのアクセス**: Lambda や EC2 など ECS 以外のクライアントからアクセスする場合は Service Discovery が必要
* **両方が必要なケース**: 同じ ECS タスクに対して、ECS サービスからは Service Connect 経由で、Lambda からは Service Discovery 経由でアクセスする、といった併用も可能

ECS のサービス間通信にはこのほか、internal ALB を活用したパターンなども存在します。なおAWS App Mesh は2026年9月30日にサービス終了が予定されており、Service Connect への移行が推奨されています（[Migrating from AWS App Mesh to Amazon ECS Service Connect](https://aws.amazon.com/jp/blogs/containers/migrating-from-aws-app-mesh-to-amazon-ecs-service-connect/)）。

今回、業務の中だけではなかなか触れないネットワークまわりの領域も調べたことで、ECS の理解が一歩進んだ気がします。アドカレ以外でもこういったアウトプットは定期的に行いたいですね！

## 5. 参考文献

* [モノリシックアーキテクチャとマイクロサービスアーキテクチャの違い - AWS](https://aws.amazon.com/jp/compare/the-difference-between-monolithic-and-microservices-architecture/)
* [ECS Immersion Day - Basic](https://catalog.workshops.aws/ecs-immersion-day/en-US/30-basic)
* [Web アプリケーションのアーキテクチャ設計パターン - AWS Builders Flash](https://aws.amazon.com/jp/builders-flash/202409/web-app-architecture-design-pattern/)
