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

![Amazon ECS](/images/20251205/Arch_Amazon-Elastic-Container-Service_64.png)

ECSは「**コンテナ化されたアプリケーションを簡単にデプロイ・管理・スケーリングできる、完全マネージド型のコンテナオーケストレーションサービス**」と説明されますが、何をどう管理するのかイメージしにくいと思います。そこで、まずマイクロサービスアーキテクチャという考え方について振り返ります。

### マイクロサービスアーキテクチャとは

前提として、**モノリシックアーキテクチャ**という考え方があります。これは全ての機能を1つの大きなアプリケーションとして開発・デプロイする方法です。個人開発などでアプリを作る際は、ここから始まることが多いのではないでしょうか。
モノリシックアーキテクチャでエンタープライズや複雑な処理の伴うアプリケーションを開発・運用する場合、以下のような課題が生じます。

* 一部の機能を修正しただけでも、全体を再デプロイする必要がある
* アプリケーションの規模が大きくなると、開発やテストが複雑になる
* 特定の機能だけスケールさせることが難しい

これらの課題を解決するのが**マイクロサービスアーキテクチャ**です。アプリケーションを小さな独立したサービスの集まりとして構成します。

![モノリシックとマイクロサービスの比較](/images/20251205/monolith_1-monolith-microservices.70b547e30e30b013051d58a93a6e35e77408a2a8.png)
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

![ECSの構成要素](/images/20251205/ecs-core-component.png)
*出典: AWS ECS Immersion Day*

## 3. マイクロサービス間通信の課題と解決策

マイクロサービスアーキテクチャでは、複数のサービスが互いに通信し合う必要があります。例えば、注文サービスがユーザー管理サービスに認証情報を問い合わせたり、商品管理サービスに在庫を確認したりします。

ここで問題になるのが、**通信先のサービスをどうやって見つけるか**です。

通常、サービス間の通信にはIPアドレスとポート番号が必要です。しかし、ECS上で動くタスクのIPアドレスは固定ではありません。タスクの再起動、スケールアウト・イン、新しいバージョンのデプロイなど、様々な理由でIPアドレスが変わります。

IPアドレスをコードや設定ファイルに直接書き込んでしまうと、タスクが再起動するたびに設定を更新しなければならず現実的ではありません。この問題を解決するためにECSには2つの仕組みがあります。

* **Service Discovery**
* **Service Connect**

それぞれの詳細と使い分けを見ていきます。

### Service Discoveryとは

ECSの**Service Discovery**はサービスを名前で見つけられるようにする機能です。内部的にはAWS Cloud MapとRoute 53を組み合わせてDNS名で名前解決できるようにします。DNSベースの仕組みのため、VPC側でDNSによる名前解決が使える設定になっている必要があります。

![Service Discoveryの仕組み](/images/20251205/service-discovery.png)
*出典: AWS Builders Flash - Web アプリケーションのアーキテクチャ設計パターン*

これにより、タスクのIPアドレスが変わっても同じ名前でアクセスできます。また、ECS以外のクライアント(同一VPCにいるLambdaやEC2、別の内部システムなど)からもECSタスクにアクセスしやすくなります。

Service Discoveryが特に適しているのは、ECS以外のサービス(Lambda、EC2、オンプレミスの接続先など)からECSのタスクに名前でアクセスしたい場合や、伝統的な「DNSでサービスを見つける」モデルで統一したい場合です。

### Service Connectとは

Service Connectは、ECSサービス同士の通信を簡単かつ統一的に扱えるようにする仕組みです。重要なのは、**Service Connectを有効化したECSサービス同士**でのみ通信できるという点です。

Service Connectは、各タスクに**Envoyサイドカー**という補助的なコンテナを自動的に追加することで動作します。アプリケーションコンテナからの通信をEnvoyが受け取り、適切な宛先にルーティングします。

![Service Connectの仕組み](/images/20251205/service-connect.png)
*出典: AWS Builders Flash - Web アプリケーションのアーキテクチャ設計パターン*

アプリケーションコンテナは通信先のIPアドレスを知る必要がなく、Service Connectで接続された他のECSサービスにサービス名だけで通信できます。

### Service Discoveryとの違い

Service DiscoveryはDNSベースの名前解決を提供し、VPC内のどこからでも(LambdaやEC2など)ECSタスクにアクセスできます。

Service ConnectはECSサービス同士の通信に特化しており、Service Connectを有効化したサービス間でのみ利用できます。Envoyサイドカーを使って通信を仲介し、サービス間通信を標準化します。

## 4. 使い分けの考え方

使い分けは通信する側と通信される側の組み合わせで決まります。

**ECSサービス同士の通信**
Service Connectを使います。サービス間通信を統一的に扱え、運用が容易になります。

**ECS以外(LambdaやEC2など)からECSへの通信**
Service Discoveryを使います。DNSベースの名前解決により、VPC内のどこからでもアクセスできます。

簡単にまとめると、Service ConnectはECSサービス同士の通信に最適化されており、Service DiscoveryはECS以外のクライアントからのアクセスも含めて広く使える、という特徴があります。

迷ったときは、以下の3つの質問で判断できます。

* 通信するのはECSサービス同士？ → Yes: Service Connect
* ECS外(Lambda/EC2など)から名前で呼びたい？ → Yes: Service Discovery
* ECS内の通信設定を統一したい？ → Yes: Service Connect

## 5. Terraformでの定義例

以降のサンプルは最小構成です。VPC・Subnet・セキュリティグループなどの詳細は省略しています。

### 5.1 Service Discovery(DNSで service-name.local を引ける)

Service Discoveryを使うと、`user-service.local` のようなDNS名でサービスにアクセスできます。設定のポイントは **`service_registries` を入れること**。これだけでCloud Mapに登録され、DNSで引けるようになります。

```hcl
resource "aws_service_discovery_private_dns_namespace" "main" {
  name = "local"
  vpc  = aws_vpc.main.id
}

resource "aws_service_discovery_service" "user_service" {
  name = "user-service"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.main.id

    dns_records {
      ttl  = 10
      type = "A"
    }

    routing_policy = "MULTIVALUE"
  }
}

resource "aws_ecs_service" "user_service" {
  name            = "user-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.user_service.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.ecs_tasks.id]
  }

  service_registries {
    registry_arn = aws_service_discovery_service.user_service.arn
  }
}
```

### 5.2 Service Connect(ECSサービス同士の通信をECS側で管理する)

Service Connectを使うと、ECSサービス同士が名前だけで通信できます。各タスクにEnvoyサイドカーが自動で追加され、通信を仲介します。

覚えるポイントは2つです。

**1. `enabled = true` でON**
`service_connect_configuration` で有効化します。

**2. `service.port_name` はタスク定義の `portMappings.name` と一致が必要**
例えば、両方とも `"http"` にします。

```hcl
resource "aws_ecs_cluster" "main" {
  name = "main-cluster"

  service_connect_defaults {
    namespace = aws_service_discovery_private_dns_namespace.main.arn
  }
}

resource "aws_ecs_service" "order_service" {
  name            = "order-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.order_service.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.ecs_tasks.id]
  }

  service_connect_configuration {
    enabled = true

    service {
      port_name      = "http"
      discovery_name = "order-service"

      client_alias {
        port     = 8080
        dns_name = "order-service"
      }
    }
  }
}
```

タスク定義側では、`portMappings` に `name` を指定します。

```hcl
# タスク定義の container_definitions 内の portMappings 部分
portMappings = [
  {
    name          = "http"
    containerPort = 8080
    protocol      = "tcp"
  }
]
```

## 6. まとめ

本記事では、ECSでマイクロサービス間通信を実現する2つの仕組み、Service DiscoveryとService Connectについて解説しました。

普段の業務ではインフラ部分は他の担当者の方に構築していただいた部分に修正を入れたり、馴染みのないところはブラックボックスに進めてしまうことが多いのですが、この機会に理解を深めることが出来てよかったです。

## 参考文献

* [モノリシックアーキテクチャとマイクロサービスアーキテクチャの違い - AWS](https://aws.amazon.com/jp/compare/the-difference-between-monolithic-and-microservices-architecture/)
* [ECS Immersion Day - Basic](https://catalog.workshops.aws/ecs-immersion-day/en-US/30-basic)
* [Web アプリケーションのアーキテクチャ設計パターン - AWS Builders Flash](https://aws.amazon.com/jp/builders-flash/202409/web-app-architecture-design-pattern/)