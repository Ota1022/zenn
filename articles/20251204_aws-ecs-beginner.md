---
title: "ECSのService ConnectとService Discoveryの違いを理解する"
emoji: "🕸️"
type: "tech"
topics: ["AWS", "ECS"]
published: true
published_at: 2025-12-04 00:00
---

本記事は[若手AWS Leading Engineerを志す者達 Advent Calendar 2025](https://qiita.com/advent-calendar/2025/to-be-japan-aws-jr-champions)の4日目の記事です。

AWS Jr. Champions 2026 を目指すアドカレということで、自分はコンテナやアプリケーション開発に興味があるのでAmazon Elastic Container Service (ECS) を使ったマイクロサービス環境に触れる中でService Connect と Service Discoveryが何をしていて、どう使い分けるべきか理解するために調べたことをまとめてみました。

普段は[スリーシェイク](https://3-shake.com/)という会社でフルスタックエンジニアとしてWebアプリケーション開発に従事しています。会社の方でも[3-shake Advent Calendar 2025](https://qiita.com/advent-calendar/2025/3-shake)で記事を書くのでよければ見ていってください。

## 1. Amazon ECSについて

![Amazon ECS](/images/20251204/Arch_Amazon-Elastic-Container-Service_64.png)

ECSは「**コンテナ化されたアプリケーションを簡単にデプロイ・管理・スケーリングできる、完全マネージド型のコンテナオーケストレーションサービス**」などと説明されるのですが、これだと何をどう管理しているのかイメージしにくいと思います。そこで、まずマイクロサービスアーキテクチャという考え方について振り返ります。

### マイクロサービスアーキテクチャとは

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

このように分割すると以下のようなメリットが得られます。

* **独立したスケーリング**
セール時に注文が急増した時、レコメンドサービスや商品管理サービスに影響を与えずに注文サービスだけをスケールアップできます。

* **責務分離による柔軟な処理分担**
サービスごとに責務を分けることで重い処理を専用のサービスに切り出しやすくなります。例えば注文サービスは注文完了後すぐにユーザーに応答を返し、画像処理やレポート生成は別のサービスで非同期に実行できます。各サービスを独立してスケールできるため、重い処理が他のサービスに影響を与えません。

* **障害の影響範囲の限定**
レコメンドサービスに障害が起きても、注文処理自体は継続できます。サービスが独立しているため、障害の影響範囲を明確に切り分けやすくなります。

* **技術スタックの柔軟性**
フロントエンドはNext.js、レコメンドサービスは機械学習を使うのでPython、コアなバックエンドサービスはパフォーマンスが重要なのでGo、といったように各サービスで最適な技術を選択できます。

### ECSとマイクロサービスの関係

マイクロサービスアーキテクチャの考え方を踏まえてECSの話に戻ります。ECSではこれらのサービスをコンテナとして実行・管理します。これにより以下のようなメリットがあります。

* **環境の独立**：サービスごとに異なる言語やライブラリを使える
* **柔軟なリソース配分**：サービスごとに必要なCPU・メモリを設定できる
* **オーケストレーション**：起動・停止・配置・スケール管理を自動化

冒頭の「コンテナ化されたアプリケーションを簡単にデプロイ・管理・スケーリングできる、完全マネージド型のコンテナオーケストレーションサービス」という説明を噛み砕くと「**小さく分けたサービス(コンテナ)を、自動で良い感じに管理・スケールしてくれるAWSのサービス**」と言えます。

なお、AWSのコンテナオーケストレーションサービスとしては他にAmazon EKS (Elastic Kubernetes Service) もあります。これはKubernetesというオープンソースのオーケストレーションツールをマネージドで提供するサービスです。本記事ではECSに焦点を当てますが、ECSは AWS 独自の仕組みでシンプルに使える点が特徴で、EKSはKubernetesの豊富なエコシステムを活用できる点が特徴です。

## 2. ECSの構成要素

ECSがマイクロサービスを管理するサービスだと理解できたところで、具体的な構成要素を見ていきましょう。

* **Cluster(クラスター)**：コンテナを実行する環境全体の論理的な単位です。
* **Task Definition(タスク定義)**：1つ以上のコンテナをまとめたタスク全体の設計図です。どのコンテナを、どのリソース/ネットワーク/ログ設定で動かすかを指定します。
* **Task(タスク)**：Task Definition を元に起動される実行単位です。タスク内には複数コンテナを含められます。
* **Service(サービス)**：継続的に動かし続けたいアプリケーションを管理する単位です。ECSがタスクを指定数維持し、タスクが落ちたら再起動、負荷に応じてスケール、ロードバランサーとの連携などを自動で行います。

![ECSの構成要素](/images/20251204/ecs-core-component.png)
*出典: AWS ECS Immersion Day*

## 3. マイクロサービス間通信の課題と解決策

マイクロサービスアーキテクチャでは、複数のサービスが互いに通信し合う必要があります。例えば、注文サービスがユーザー管理サービスに認証情報を問い合わせたり、商品管理サービスに在庫を確認したりします。

ここで問題になるのが、**通信先のサービスをどうやって見つけるか**です。

通常、サービス間の通信にはIPアドレスとポート番号が必要ですが、ECS上で動くタスクのIPアドレスは固定ではありません。タスクの再起動、スケールアウト・イン、新しいバージョンのデプロイなど、様々な理由でIPアドレスが変わります。

IPアドレスをコードや設定ファイルに直接書き込んでしまうと、タスクが再起動するたびに設定を更新しなければなりません。この問題を解決するためにAWSではいくつか選択肢があるのですが、今回は**Service Connect**と**Service Discovery**に焦点を当てたいと思います。

### Service Connect

Service Connectは、ECSサービス同士の通信を標準的に扱えるようにするECSの機能です。ポイントは、**Service Connectを有効化したECSサービス同士**でのみ通信できるという点です。LambdaやEC2など他のAWSサービスでは利用できません。

以下の図にある通り、各タスクに**Envoyサイドカー**という補助的なコンテナを自動的に追加することで動作します。Envoyはオープンソースのプロキシロードバランサーで、アプリケーションコンテナからの通信を受け取り、適切な宛先にルーティングする役割を果たしています。

![Service Connectの仕組み](/images/20251204/service-connect.png)
*出典: AWS Builders Flash - Web アプリケーションのアーキテクチャ設計パターン*

アプリケーションコンテナは通信先のIPアドレスを知る必要がなく、Service Connectで接続された他のECSサービスにサービス名だけで通信できます。Envoyサイドカーがリクエスト数やレイテンシなどのメトリクスを自動的にCloudWatch Metricsに収集するため、サービス間通信の可観測性が向上します。

### Service Discovery

Service Discoveryもサービスを名前で見つけられるようにする機能で、ECS専用ではなくEC2、Lambda、Kubernetes(EKS)、オンプレミスなど様々なリソースでも利用できます。内部的にAWS Cloud MapとRoute 53を組み合わせてDNS名で名前解決できるようにしているのですが、VPC側でDNSによる名前解決が使える設定になっている必要があります。

![Service Discoveryの仕組み](/images/20251204/service-discovery.png)
*出典: AWS Builders Flash - Web アプリケーションのアーキテクチャ設計パターン*

Service Discoveryを使うことで、タスクのIPアドレスが変わっても同じDNS名でアクセスできます。また、ECS以外のクライアント(同一VPCにいるLambdaやEC2、別の内部システムなど)からもECSタスクにアクセスしやすくなります。

### 使い分けの考え方

使い分けは通信する側と通信される側の組み合わせで決まります。

ECSサービス同士が通信する場合は**Service Connect**を使います。サービス間通信を統一的に扱え、Envoyによる可観測性の向上やトラフィック制御などの恩恵を受けられます。

一方、ECS以外のサービス(LambdaやEC2など)からECSタスクにアクセスする場合は**Service Discovery**を使います。DNSベースの名前解決により、VPC内のどこからでもアクセスできます。

## 4. Terraformでの定義例

ここからは段階的に Terraform での定義を見ていきます。VPC / Subnet / Security Group / IAM ロールなどの周辺設定は省略し、Service Discovery と Service Connect を有効化する時にどこをどう書き足すかに絞ります。

### 4.1 Task 定義と Service

まずはシンプルな構成として、**Cluster、Task Definition、Service** を仮定します。

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
      image = "nginx:latest"      # 使用するDockerイメージ

      portMappings = [
        {
          containerPort = 80      # コンテナ内で待ち受けるポート
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

### 4.2 Service Connect を追加

リトライ・タイムアウト・トラフィック分散・メトリクス収集などをECS で標準化してまとめて扱いたい場合は Service Connect が便利です。Service Connect を有効化すると、各タスクに **Envoy サイドカー**が追加され、アプリケーションの通信を仲介します。

4.1 の基本構成に対して、以下の変更・追加を行います。

```hcl
# ECS クラスター
# Service Connect のデフォルト名前空間を設定
# ここで指定した名前空間内でサービス同士が通信できるようになる
resource "aws_ecs_cluster" "main" {
  name = "my-cluster"

  service_connect_defaults {
    # Cloud Map 名前空間の ARN を指定(ID ではなく ARN なので注意)
    namespace = aws_service_discovery_private_dns_namespace.main.arn
  }
}

# タスク定義
# Service Connect を使う場合、portMappings に name を付ける必要がある
resource "aws_ecs_task_definition" "app" {
  family                   = "my-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "my-app:latest"

      portMappings = [
        {
          # 重要: この name が Service Connect 側の port_name と一致する必要がある
          # これにより、どのポートを Service Connect 経由で公開するかを特定できる
          name          = "http"
          containerPort = 8080
          protocol      = "tcp"
        }
      ]
    }
  ])
}

# ECS サービス
# service_connect_configuration を追加して Service Connect を有効化
resource "aws_ecs_service" "app" {
  name            = "my-app-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = ["subnet-xxx", "subnet-yyy"]
    security_groups = ["sg-zzz"]
  }

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

上記のコードで注目すべきは、**Service Connect の設定が ECS クラスター・タスク定義・サービスの3箇所に分散している**点です。

- **クラスター**では `namespace` を指定して通信可能な範囲を定義
- **タスク定義**では `portMappings[].name` でどのポートを公開するかを宣言
- **サービス**では `service_connect_configuration` で実際の接続設定を行う

この3層の設定が揃って初めて Service Connect が機能します。特に `portMappings[].name` と `service_connect_configuration.service.port_name` の一致が必須であり、ここが食い違うとサービスが正しく動作しません。

もう一つ重要なのは、`discovery_name` で指定した名前が **自動的に Cloud Map にも登録される**ことです。つまり Service Connect は内部的に Service Discovery の仕組みを利用しており、その上に Envoy による通信制御を載せた構造になっています。Service Connect を使えば Service Discovery も同時に得られるということです。

### 4.3 Service Discovery を追加

「user-service」「order-service」のようなサービス名で通信したい場合は Service Discovery(Cloud Map + Route 53 による DNS 登録)を使います。

4.1 の基本構成に対して、以下を追加します。

```hcl
# Cloud Map 名前空間
# DNS のゾーンのようなもの。"my-app.local" の ".local" 部分を定義
# VPC内のプライベートなDNS名前空間として機能する
resource "aws_service_discovery_private_dns_namespace" "main" {
  name = "local"
  vpc  = aws_vpc.main.id
}

# Cloud Map の Service
# DNS レコードの設定を行う。タスクのIPアドレスを自動で登録・削除する
resource "aws_service_discovery_service" "app" {
  name = "my-app"  # "my-app.local" の "my-app" 部分

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10    # DNS キャッシュの有効期間(秒)。短めにするとIPの変更に早く追従
      type = "A"   # IPv4アドレスを登録
    }

    # 複数のIPアドレスを返す(複数タスクに負荷分散)
    routing_policy = "MULTIVALUE"
  }

  # Route 53 のヘルスチェックではなく、ECS の登録状態に基づく簡易ヘルスチェック
  # failure_threshold = 1 で、1回失敗したら unhealthy とみなす
  health_check_custom_config {
    failure_threshold = 1
  }
}

# ECS サービス(service_registries を追加)
resource "aws_ecs_service" "app" {
  name            = "my-app-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = ["subnet-xxx", "subnet-yyy"]
    security_groups = ["sg-zzz"]
  }

  # Service Discovery との連携設定
  # これを追加するだけで、タスクが起動・停止時に自動でCloud Mapに登録・削除される
  # 結果: "my-app.local" という名前でタスクにアクセスできるようになる
  service_registries {
    registry_arn   = aws_service_discovery_service.app.arn
    container_name = "app"  # タスク定義で指定したコンテナ名
    container_port = 80     # 登録するポート番号
  }
}
```

Service Discovery の構成を見ると、**Cloud Map の名前空間とサービスを定義し、それを ECS サービスの `service_registries` で参照する**という2段階の構造になっています。Service Connect と比べると設定箇所が少なくシンプルです。

ここで重要なのは、`routing_policy = "MULTIVALUE"` と `ttl = 10` の組み合わせです。MULTIVALUE ポリシーでは DNS クエリに対して複数のタスクの IP アドレスが全て返されます。つまり、**負荷分散の責任はクライアント側にある**ということです。これは Service Connect の Envoy が自動的に負荷分散してくれるのとは対照的です。

また、`health_check_custom_config` を使っている点も注目すべきです。これは Route 53 の HTTP/HTTPS ヘルスチェックではなく、**ECS が管理するタスクの状態を直接使う**簡易的な方式です。ECS が「このタスクは起動している」と判断すれば healthy、停止すれば DNS から削除されます。別途ヘルスチェックエンドポイントを用意する必要がなく、ECS のタスク管理に完全に委ねられます。

Service Discovery は Cloud Map + Route 53 という汎用的な仕組みのため、ECS 以外からも利用できます。Service Connect は ECS サービス間専用ですが、Service Discovery は VPC 内のあらゆるリソースが対象です。

## 5. まとめ

ECS におけるマイクロサービス間通信の基本として、Service Discovery と Service Connect の 2 つの仕組みを整理しました。業務の中だけではなかなか触れない領域も含めて調べてみたことで、ECS のネットワークまわりの理解が一歩進んだ気がします。

ECS のサービス間通信にはこのほか、internal ALB を活用したパターンなども存在します。なおAWS App Mesh は2026年9月30日にサービス終了が予定されており、Service Connect への移行が推奨されています。

## 6. 参考文献

* [モノリシックアーキテクチャとマイクロサービスアーキテクチャの違い - AWS](https://aws.amazon.com/jp/compare/the-difference-between-monolithic-and-microservices-architecture/)
* [ECS Immersion Day - Basic](https://catalog.workshops.aws/ecs-immersion-day/en-US/30-basic)
* [Web アプリケーションのアーキテクチャ設計パターン - AWS Builders Flash](https://aws.amazon.com/jp/builders-flash/202409/web-app-architecture-design-pattern/)
* [Migrating from AWS App Mesh to Amazon ECS Service Connect - AWS Blog](https://aws.amazon.com/jp/blogs/containers/migrating-from-aws-app-mesh-to-amazon-ecs-service-connect/)
