---
title: "ECSのService ConnectとService Discoveryの違いを理解する"
emoji: "🕸️"
type: "tech"
topics: ["AWS", "ECS"]
published: false
---

本記事は[若手AWS Leading Engineerを志す者達 Advent Calendar 2025](https://qiita.com/advent-calendar/2025/to-be-japan-aws-jr-champions)の4日目の記事です。

普段は[スリーシェイク](https://3-shake.com/)という会社で主にWebアプリケーションのバックエンド開発に従事しています。今回はAmazon Elastic Container Service (ECS) を使ったマイクロサービス環境に触れる中でService Connect と Service Discoveryが何を解決していて、どう使い分けるのかを理解するために調べたことをまとめました。

AWS Jr. Champions 2026 を目指すアドカレということもあり、コンテナやアプリケーション開発に馴染みのない人にもわかるように解説したいと思います。

会社の方でも[3-shake Advent Calendar 2025](https://qiita.com/advent-calendar/2025/3-shake)で記事を書くのでよければ見ていってください。

## 1. Amazon ECSについて

![Amazon ECS](/images/20251204/Arch_Amazon-Elastic-Container-Service_64.png)

ECSは「**コンテナ化されたアプリケーションを簡単にデプロイ・管理・スケーリングできる、完全マネージド型のコンテナオーケストレーションサービス**」と説明されますが、何をどう管理するのかイメージしにくいと思います。そこで、まずマイクロサービスアーキテクチャという考え方について振り返ります。

### マイクロサービスアーキテクチャとは

前提として、**モノリシックアーキテクチャ**という考え方があります。これは全ての機能を1つの大きなアプリケーションとして開発・デプロイする方法です。個人開発などでアプリを作る際は、ここから始まることが多いのではないでしょうか。
モノリシックアーキテクチャでエンタープライズや複雑な処理の伴うアプリケーションを開発・運用する場合、以下のような課題が生じます。

* 一部の機能を修正しただけでも、全体を再デプロイする必要がある
* アプリケーションの規模が大きくなると、開発やテストが複雑になる
* 特定の機能だけスケールさせることが難しい

これらの課題を解決するのが**マイクロサービスアーキテクチャ**です。アプリケーションを小さな独立したサービスの集まりとして構成します。

![モノリシックとマイクロサービスの比較](/images/20251204/monolith_1-monolith-microservices.70b547e30e30b013051d58a93a6e35e77408a2a8.png)
*出典: AWS - モノリシックアーキテクチャとマイクロサービスアーキテクチャの違い*

例えばECサイトの場合、

* フロントエンドサービス：ユーザーインターフェース(Next.js)
* ユーザー管理サービス：認証やユーザー情報の管理(Go)
* 商品管理サービス：商品カタログや在庫管理(Go)
* 注文サービス：注文処理や決済(Go)
* レコメンドサービス：商品レコメンデーション(Python + 機械学習)

このように機能ごとに分割することで以下のようなメリットが得られます。

**1. 独立したスケーリング**
セール時に注文が急増しても注文サービスだけをスケールアップできます。レコメンドサービスや商品管理サービスは影響を受けません。

**2. 責務分離による柔軟な処理分担**
サービスごとに責務を分けることで重い処理を専用のサービスに切り出しやすくなります。例えば注文サービスは注文完了後すぐにユーザーに応答を返し、画像処理やレポート生成は別のサービスで非同期に実行できます。各サービスを独立してスケールできるため、重い処理が他のサービスに影響を与えません。

**3. 障害の影響範囲の限定**
レコメンドサービスに障害が起きても、注文処理自体は継続できます。モノリシックでは一部の機能の障害がシステム全体に影響することがありました。

**4. 技術スタックの柔軟性**
フロントエンドはNext.js、レコメンドサービスは機械学習を使うのでPython、コアなバックエンドサービスはパフォーマンスが重要なのでGo、といったように各サービスで最適な技術を選択できます。

### ECSとマイクロサービスの関係

マイクロサービスアーキテクチャでは、それぞれのサービスが固有の処理を担当します。ECSは、これらのサービスを「コンテナ」として実行し、以下のような管理を担います。

* **環境の独立性**：サービスごとに異なる言語やライブラリを使える
* **柔軟なリソース配分**：サービスごとに必要なCPU・メモリを設定できる
* **オーケストレーション**：起動・停止・配置・スケール管理を自動化

冒頭の「コンテナ化されたアプリケーションを簡単にデプロイ・管理・スケーリングできる、完全マネージド型のコンテナオーケストレーションサービス」という説明を噛み砕くと「**小さく分けたサービス(コンテナ)を、自動で良い感じに管理・スケールしてくれるAWSのサービス**」と言えます。

## 2. ECSの構成要素

ECSがマイクロサービスを管理するサービスだと理解できたところで、具体的な構成要素を見ていきましょう。

* **Cluster(クラスター)**：コンテナを実行する環境全体の論理的な単位です。
* **Task Definition(タスク定義)**：1つ以上のコンテナをまとめたタスク全体の設計図です。どのコンテナを、どのリソース/ネットワーク/ログ設定で動かすかを指定します。
* **Task(タスク)**：Task Definition を元に起動される実行単位です。タスク内には複数コンテナを含められます。
* **Service(サービス)**：継続的に動かし続けたいアプリケーション(Webサービスやマイクロサービスなど)を管理する単位です。ECSがタスクを指定数維持し、タスクが落ちたら再起動、負荷に応じてスケール、ロードバランサーとの連携などを自動で行います。

![ECSの構成要素](/images/20251204/ecs-core-component.png)
*出典: AWS ECS Immersion Day*

## 3. マイクロサービス間通信の課題と解決策

マイクロサービスアーキテクチャでは、複数のサービスが互いに通信し合う必要があります。例えば、注文サービスがユーザー管理サービスに認証情報を問い合わせたり、商品管理サービスに在庫を確認したりします。

ここで問題になるのが、**通信先のサービスをどうやって見つけるか**です。

通常、サービス間の通信にはIPアドレスとポート番号が必要です。しかし、ECS上で動くタスクのIPアドレスは固定ではありません。タスクの再起動、スケールアウト・イン、新しいバージョンのデプロイなど、様々な理由でIPアドレスが変わります。

IPアドレスをコードや設定ファイルに直接書き込んでしまうと、タスクが再起動するたびに設定を更新しなければならず現実的ではありません。この問題を解決するためにECSには2つの仕組みがあります。


### Service Discovery

ECSの**Service Discovery**はサービスを名前で見つけられるようにする機能です。内部的にはAWS Cloud MapとRoute 53を組み合わせてDNS名で名前解決できるようにします。DNSベースの仕組みのため、VPC側でDNSによる名前解決が使える設定になっている必要があります。

![Service Discoveryの仕組み](/images/20251204/service-discovery.png)
*出典: AWS Builders Flash - Web アプリケーションのアーキテクチャ設計パターン*

これにより、タスクのIPアドレスが変わっても同じ名前でアクセスできます。また、ECS以外のクライアント(同一VPCにいるLambdaやEC2、別の内部システムなど)からもECSタスクにアクセスしやすくなります。

Service Discoveryが特に適しているのは、ECS以外のサービス(Lambda、EC2、オンプレミスの接続先など)からECSのタスクに名前でアクセスしたい場合や、伝統的な「DNSでサービスを見つける」モデルで統一したい場合です。

### Service Connect

Service Connectは、ECSサービス同士の通信を簡単かつ統一的に扱えるようにする仕組みです。重要なのは、**Service Connectを有効化したECSサービス同士**でのみ通信できるという点です。

Service Connectは、各タスクに**Envoyサイドカー**という補助的なコンテナを自動的に追加することで動作します。アプリケーションコンテナからの通信をEnvoyが受け取り、適切な宛先にルーティングします。

![Service Connectの仕組み](/images/20251204/service-connect.png)
*出典: AWS Builders Flash - Web アプリケーションのアーキテクチャ設計パターン*

アプリケーションコンテナは通信先のIPアドレスを知る必要がなく、Service Connectで接続された他のECSサービスにサービス名だけで通信できます。

### Service Discoveryとの違い

**Service Discovery**はDNSベースの名前解決を提供し、VPC内のどこからでも(Lambda、EC2など)ECSタスクにアクセスできます。

**Service Connect**はECSサービス同士の通信に特化しており、Envoyサイドカーを使って通信を仲介します。リクエスト数やレイテンシなどのメトリクスを自動的にCloudWatch Metricsに収集するため、サービス間通信の可観測性が向上します。

### 使い分けの考え方

使い分けは通信する側と通信される側の組み合わせで決まります。

ECSサービス同士が通信する場合は**Service Connect**を使います。サービス間通信を統一的に扱え、Envoyによる可観測性の向上やトラフィック制御などの恩恵を受けられます。

一方、ECS以外のサービス(LambdaやEC2など)からECSタスクにアクセスする場合は**Service Discovery**を使います。DNSベースの名前解決により、VPC内のどこからでもアクセスできます。

## 4. Terraformでの定義例

ここからは段階的に Terraform での定義を見ていきます。**VPC / Subnet / Security Group / IAM ロールなどの周辺設定は省略**し、「Service Discovery と Service Connect を有効化する時にどこをどう書き足すか」に絞ります。

### 4.1 最小構成 (Task 定義と Service)

まずは最もシンプルな構成として、**Cluster、Task Definition、Service** の 3 つがあれば ECS は動きます。

```hcl
# ECS クラスター
resource "aws_ecs_cluster" "main" {
  name = "my-cluster"
}

# タスク定義(どんなコンテナを動かすか)
resource "aws_ecs_task_definition" "app" {
  family                   = "my-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"

  execution_role_arn = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "nginx:latest"

      portMappings = [
        {
          containerPort = 80
          protocol      = "tcp"
        }
      ]
    }
  ])
}

# ECS サービス(タスクを何個維持するか)
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
}
```

IAM ロール / ログ / VPC などの定義は省略しています。

この状態では、サービス間通信をする場合「どのタスクの IP に向けて呼ぶか」を呼び出し側で何らかの方法で解決する必要があります(IP は動的に変わるため、固定値をコードや設定に書くのは現実的ではありません)。

### 4.2 Service Discovery を追加 (サービス名でアクセス)

**「user-service」「order-service」のようなサービス名で通信したい**場合は Service Discovery(Cloud Map + Route 53 による DNS 登録)を使います。

追加するのは主に 2 つです。

1. **名前空間(Namespace)**：`service-name.local` の `.local` 部分
2. **ECS Service の `service_registries`**：この設定により、タスク起動/終了に追従して Cloud Map に登録/解除されます

```hcl
# Cloud Map 名前空間(DNS のゾーンのようなもの)
resource "aws_service_discovery_private_dns_namespace" "main" {
  name = "local"
  vpc  = aws_vpc.main.id
}

# Cloud Map の Service(DNS レコードの設定など)
resource "aws_service_discovery_service" "app" {
  name = "my-app"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10
      type = "A"
    }

    routing_policy = "MULTIVALUE"
  }

  # Route 53 のヘルスチェックではなく、ECS の登録状態に基づく簡易ヘルスチェック
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

  # これを追加するだけで、タスクが Cloud Map に登録され DNS で解決できるようになる
  service_registries {
    registry_arn   = aws_service_discovery_service.app.arn
    container_name = "app"
    container_port = 80
  }
}
```

これで **`my-app.local`** という DNS 名で到達できるようになります(タスクの増減・入れ替えにも追従します)。

### 4.3 Service Connect を追加(ECS サービス間通信を統一)

**リトライ・タイムアウト・トラフィック分散・メトリクス収集**などをECS で標準化してまとめて扱いたい場合は Service Connect が便利です。Service Connect を有効化すると、各タスクに **Envoy サイドカー**(補助コンテナ)が自動で追加され、アプリケーションの通信を仲介します。

ここで重要なポイントは 2 つです。

1. **`service_connect_configuration.enabled = true`** で有効化する
2. **`port_name` の一致**：Task Definition の `portMappings[].name` と ECS Service 側の `service.port_name` を 必ず一致させる

```hcl
# クラスターに Service Connect のデフォルト名前空間を設定
# namespace には Cloud Map 名前空間の "ARN" を指定する
resource "aws_ecs_cluster" "main" {
  name = "my-cluster"

  service_connect_defaults {
    namespace = aws_service_discovery_private_dns_namespace.main.arn
  }
}

# タスク定義(portMappings に name を追加：これが port_name と一致する必要がある)
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
          name          = "http"   #  Service Connect 側の port_name と一致
          containerPort = 8080
          protocol      = "tcp"
        }
      ]
    }
  ])
}

# ECS サービス(service_connect_configuration を追加)
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
    enabled = true

    # この service ブロックが「Service Connect 経由で公開するサービス」を定義する
    service {
      port_name      = "http"   # Task Definition の portMappings[].name と一致
      discovery_name = "my-app" # Service Connect 内での登録名(Cloud Map 側に連動)

      # 呼び出し側が使う入口(DNS 名とポート)
      client_alias {
        dns_name = "my-app"
        port     = 8080
      }
    }
  }
}
```

これで、同じ名前空間(例：`local`)に属する Service Connect 対応サービスからは **`my-app:8080`** のように名前で呼べるようになります。また、Envoy が通信を仲介することで、サービス間通信の運用(統一設定・可観測性の向上など)をしやすくなります。

## 5. まとめ

本記事では、ECS におけるマイクロサービス間通信の基本として、Service Discovery と Service Connect の 2 つの仕組みを整理しました。

業務の中だけではなかなか触れない領域も含めて調べてみたことで、ECS のネットワークまわりの理解が一歩進んだ気がします。

ECS のサービス間通信にはこのほか、internal ALB を活用したパターンや App Mesh によるサービスメッシュ構成なども存在します。これらについてもまた別の機会に掘り下げていければと思います。

## 6. 参考文献

* [モノリシックアーキテクチャとマイクロサービスアーキテクチャの違い - AWS](https://aws.amazon.com/jp/compare/the-difference-between-monolithic-and-microservices-architecture/)
* [ECS Immersion Day - Basic](https://catalog.workshops.aws/ecs-immersion-day/en-US/30-basic)
* [Web アプリケーションのアーキテクチャ設計パターン - AWS Builders Flash](https://aws.amazon.com/jp/builders-flash/202409/web-app-architecture-design-pattern/)