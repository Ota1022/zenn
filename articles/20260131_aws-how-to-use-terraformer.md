---
title: "Terraformerを使ってAWSのリソースをIaC化する"
emoji: "🏗️"
type: "tech"
topics: ["AWS", "Terraform", "IaC"]
published: true
published_at: 2026-01-31 07:00
---

業務でTerraformerを使って既存のAWSリソースをTerraform管理下に移行する機会がありました。インフラに詳しいメンバーの知見を借りながら進めたのですが、その過程で学んだことが多かったので備忘として残します。

https://github.com/GoogleCloudPlatform/terraformer

## 1. Terraformerとは

クラウド上にTerraformで管理されていないリソースが残っていて困った経験は多くの人にあると思います。

- Terraform導入前に試験的に作ったリソースがそのまま残っている
- 手動でコンソールから作成したリソースが把握しきれていない
- 別チームが管理していたリソースが引き継がれないまま放置されている

こうした既存リソースをTerraform管理下に取り込む際に活用できるのがTerraformerです。

### 概要

Terraformerは、既存のクラウドリソースからTerraformのコード（HCL）とstateファイルをまとめて生成するCLIツールです。GitHubのGoogleCloudPlatformでOSSとして公開されています。

Terraformには`import`コマンドがありますが、リソースを1つずつstateに取り込み、対応するHCLコードを手動で書く必要があります。大量のリソースを一度にIaC化するには現実的ではありません。Terraformerはこの作業を一括で行い、HCLコードとstateファイルをまとめて生成してくれるツールとして開発されました。

AWS以外にもGCP、Azure、Kubernetes、Datadogなど多くのプロバイダに対応しています。Go製のシングルバイナリなので、`brew install`やバイナリのダウンロードだけで導入できます。OSSなので誰でも利用可能です。

### Terraformerの位置づけ

Terraformerは「Terraformコードを自動生成してくれるツール」と思われがちですが、実際には既存リソースの構造をTerraformが読める形に写し取るツールです。

内部的には各クラウドプロバイダのAPIを叩いてリソース情報を取得し、それをTerraformのproviderプラグイン経由でstateファイルに書き込みつつ、stateの内容からHCLコードを逆生成しています。つまりAPIのレスポンスをそのままコードに落としているだけなので、Terraformの書き方としてのベストプラクティス（変数化、リソース間の参照、モジュール構成など）は反映されません。

実際、importが完了した直後に`terraform plan`を実行すると大量の差分が出ます。コンソール上の表現とTerraformの表現のズレ、デフォルト値の扱いの違い、機密情報の有無など、構造的に差分が出ることは避けられないと考えるべきです。

importはIaC化のスタート地点なので、ここからコードを整理しplanが安定する状態に持っていく作業が本番になります。

### 「IaC化」と「Terraform管理」の違い

「IaC化」と「Terraform管理」は似ているようで異なります。IaC化はコードが存在する状態を指しますが、Terraform管理はそのコードで安全に`plan`/`apply`ができ、継続的にインフラを更新・運用できる状態を意味します。

Terraformerでimportしただけの段階はIaC化に近いですが、Terraform管理にはまだ至っていません。差分の解消、不要なリソースの除外、機密情報の分離、state構成の整理などを経て、はじめてTerraformで管理しているといえる状態になります。

この記事ではAWSを対象に、Terraformerを使ってTerraform管理に着地させるまでを扱います。

## 2. Terraformerを扱う前に考えるべきこと

TerraformerはAWSの全サービスに対応しているわけではありません。[対応サービスの一覧](https://github.com/GoogleCloudPlatform/terraformer/blob/master/docs/aws.md)を事前に確認しておくと、対象外のサービスをimportしようとして手戻りになることを避けられます。

### 既存リソースの前提

既存のAWSリソースには、コードには表れない暗黙の前提が多く含まれています。

- コンソールから手動で設定したセキュリティグループのルール
- デフォルトVPCやデフォルトサブネットに依存した構成
- 手動で付与したIAMポリシーの継ぎ足し
- タグが付いていない、あるいは命名規則が統一されていないリソース

これらはTerraformerでimportしても、コードを見ただけでは意図が読み取れません。importする前に、なぜそのリソースがその状態になっているのかを把握しておくことが重要です。

### Terraform管理に向いているもの、向いていないもの

すべてのリソースをTerraform管理に移行する必要はありません。

**向いているもの**
- VPC、サブネット、セキュリティグループなどのネットワーク系リソース
- IAMロール、ポリシーなどの権限管理系リソース
- ECS、Lambda、RDSなどアプリケーション基盤となるリソース

**向いていないもの**
- 一時的な検証用リソース（削除した方が早い）
- 他のツールがライフサイクルを管理しているリソース（例：CDKやSAMで管理中のもの）
- 頻繁に手動変更が入るリソース（Terraform管理にしても差分が出続ける）

### 事前に決めるべき指針

Terraformerを実行する前に、以下の3点を決めておくと作業がブレにくくなります。

1. **対象**：どのサービス・どのリソースをimport対象にするか
2. **除外**：明示的にTerraform管理から外すものは何か（セキュリティ系の認証情報、バイナリ化されたLambdaのコード、stateファイルを格納しているS3バケットなど）
3. **ゴール**：`plan`の差分ゼロを目指すのか、主要リソースの管理だけで十分とするのか

全リソースの差分ゼロを目指すと作業量が膨れ上がるため、最初から完璧を目指さず、段階的に管理範囲を広げていく方針の方が現実的です。

## 3. Terraformerの実行フロー

Terraformerによる移行作業は、大きく4つのステップに分けて考えると整理しやすいです。

### ステップ① 棚卸し

Terraformerを実行する前に、対象のAWSアカウントにどんなリソースが存在するのかを棚卸ししましょう。

- AWSコンソールやAWS CLIで現状のリソースを洗い出す
- 誰が何のために作ったのか分からないリソースを特定する
- Terraform管理に含めるもの・含めないものを仕分ける

棚卸しを面倒だからといって飛ばすと、不要なリソースまでTerraform管理に入り込み、後から除外する手間が増えます。

### ステップ② import

棚卸しの結果をもとに、Terraformerでimportを実行します。対象のサービスやリージョンを指定してコマンドを実行すると、HCLファイルとstateファイルが生成されます。

```bash
terraformer import aws \
  --resources=vpc,subnet,security_group \
  --regions=ap-northeast-1 \
  --profile=your-profile
```

なお私はカンマ区切りで複数サービスを同時にimportしたところ、一部のリソースが取得されない現象に遭遇しました。同様の報告はGitHub上にもありましたが（[#1886](https://github.com/GoogleCloudPlatform/terraformer/issues/1886)）、原因となるリソースの特定が難しいので、うまくいかない場合はサービスごとに分けて実行するのが無難です。

```bash
# サービスごとに分けて実行する
terraformer import aws --resources=vpc --regions=ap-northeast-1 --profile=your-profile
terraformer import aws --resources=ecs --regions=ap-northeast-1 --profile=your-profile
```

ただし、サービスごとに分けるとimportのたびにstateファイルが別々に生成されるため、後から1つのstateに統合する作業が必要になります。この点についてはステップ③で触れます。

なお、Terraformerはサービス単位（vpc、ecs、iamなど）でimportするのが基本ですが、Terraformプロジェクト側の構成は責務単位（ネットワーク、アプリケーション基盤、監視など）で分けることが多いです。import時点ではサービス単位で実行し、次のステップで責務単位へ再構成する、という2段階で考えると進めやすいです。

この時点で生成されるコードはあくまで素材だと思ってください。

### ステップ③ 統合

生成されたHCLコードを、実際のTerraformプロジェクトの構成に合わせて整理します。ここが最も手間のかかる工程です。

Terraformerが生成するコードには以下のような特徴があり、そのままリポジトリに入れても読めませんしメンテナンスも厳しいです。

- リソース名が`tfer--`プレフィックス付きの自動生成名になる
- すべての属性がベタ書きされる（デフォルト値と同じ値も明示的に出力される）
- IDやARNがハードコードされている
- リソース間の参照がつながらず、IDやARNの直書きになりがち

生成されたコードは「どのリソースが存在するか」「どんな属性を持っているか」を把握するための下書きとして扱い、以下の観点で書き直していきます。

- **命名**：`tfer--sg-0123456789abcdef0`のような名前を`web_app_sg`のように意味のある名前に変更する
- **ファイル分割**：1ファイルにすべてを詰め込まず、責務やリソースの関連性に応じて分ける（例：`network.tf`、`iam.tf`、`ecs.tf`）
- **参照の書き換え**：IDのハードコードを`aws_vpc.main.id`のようなリソース参照に置き換える
- **不要な属性の削除**：デフォルト値と同じ属性や、Terraformでは不要な計算属性を削除する
- **機密情報の確認**：コードやstateに機密情報が含まれていないか確認する

地味な作業ですが、後から他のメンバーがコードを読むときの可読性に直結します。コードの整形やリネームなどはAIの支援である程度楽ができますが、何を残して何を外すか、どう構成するかといった判断は人間が行う必要があります。

なお、移行直後にmodule化を急ぐのは避けた方がいいです。コードの修正のたびにmoduleのインターフェースも変える必要が出て手戻りが増えます。まずはフラットな構成でplanを安定させてから、複数環境で同じパターンを使い回す必要が出てきた段階でmodule化を検討する方が進めやすいです。

#### stateの統合について

Terraformerはサービスごとに`generated/aws/<service>/terraform.tfstate`を生成します。これを本番のTerraformプロジェクトに統合する方法は主に2つあります。

1. **生成されたstateは使わず、コードだけ取り出して`terraform import`でやり直す**
2. **`terraform state mv`で生成されたstateから本番のstateにリソースを移動する**

2の方が効率的に見えますが、`state mv`はリソース名の変更やbackendの違いを考慮する必要があり、操作ミスのリスクもあります。Terraformerが生成するコードはどのみち大幅に書き直すことになるので、コードを整理した上で`terraform import`でやり直す方が結果的に安全です。生成されたstateは「元のリソースIDを確認するための参考資料」として使うくらいの位置づけが良いと思います。

### ステップ④ planの安定化

コードの整理が終わったら、`terraform plan`を実行して差分を確認します。最初は大量の差分が出るのが普通です。

- 差分の内容を1つずつ確認し、実害のない表現差なのか、本当に変更が必要なのかを判断する
- `lifecycle`の`ignore_changes`で制御すべきものを特定する
- stateの操作（`state rm`や`state mv`）が必要なケースに対応する

planの差分がなくなる、もしくは意図した差分だけになった時点で、Terraform管理への移行は完了です。

## 4. importの際に考慮したいポイント

### `resources=*` は避けたい

Terraformerには`--resources=*`で全サービスを一括importするオプションがあります。一見便利ですが、実際に使うと問題が多いです。

- AWSアカウント上のすべてのリソースがimport対象になるため、大量の不要なリソース（デフォルトVPC、使っていないIAMロールなど）まで取り込まれる
- 生成されるファイル数が膨大になり、どこから手をつけていいか分からなくなる
- APIのレートリミットに引っかかってimport自体が途中で失敗することがある

棚卸しで決めた対象サービスを、`--resources=vpc,ecs,iam`のように明示的に指定しましょう。

### グローバルリソースとリージョナルリソース

AWSにはリージョンに紐づくリソース（EC2、RDS、ECSなど）と、グローバルなリソース（IAM、Route 53、CloudFrontなど）があります。Terraformerの`--regions`オプションはリージョナルリソースに対してのみ有効で、グローバルリソースはリージョン指定に関係なくimportされます。

また、CloudFront用のACM証明書のように`us-east-1`での作成が必須なリソースもあります。こうしたリソースをTerraformで扱うにはprovider aliasの設計が必要になるため、import時にどのリージョンに属するリソースなのかを意識しておかないと、後からprovider設定を変更する手間が発生します。

リージョンが異なるリソースの管理方法としては、ディレクトリをリージョンごとに分ける（`regions/ap-northeast-1/`、`regions/global/`など）か、同一ディレクトリ内でprovider aliasを使い分けるかの2択になります。プロジェクトの規模が小さければprovider aliasで十分ですが、リソースが増えてきたらディレクトリ分離の方がstateが肥大化しにくくなります。

```hcl
provider "aws" {
  region = "ap-northeast-1"
}

provider "aws" {
  alias  = "global"
  region = "us-east-1"
}
```

## 5. 機密情報の扱いとTerraform state

### stateに機密情報が入る仕組み

Terraformのstateファイルには、管理対象リソースの全属性が平文で記録されます。つまり、データベースのパスワードやAPIキーなどをTerraformで管理すると、それらがstateにそのまま書き込まれます。

stateをS3などのリモートバックエンドに保存していても、アクセス権限のあるメンバーなら`terraform state pull`で中身を見ることができます。Terraformerでimportした場合、元のリソースに設定されていた機密情報がそのままstateに取り込まれるため、意図せず機密情報が広がるリスクがあります。

### 環境変数、シークレットなどの取り扱い

Terraformerでimportすると特に問題になりやすいのが以下のケースです。

- **Lambda環境変数**：環境変数にAPIキーやDBの接続文字列が入っていると、それがHCLコードとstateの両方に平文で出力される
- **SSM Parameter Store**：`SecureString`で格納しているパラメータの値がstateに記録される
- **Secrets Manager**：シークレットの値そのものがstateに入る

これらはTerraformerが勝手にやっているわけではなく、Terraformの仕様としてstateに全属性が保存されるために起こります。importした後に気づかずリポジトリにstateをコミットしてしまうと、機密情報の漏洩につながります。

対処としては、機密情報はParameter Store（SecureString）やSecrets Managerで管理し直し、Terraform側ではARNや名前で参照するだけにとどめるのが基本です。

### Lambdaのコード管理

Terraformerのimportで生成されるのは主にLambda関数の設定（メモリ、タイムアウト、環境変数、IAMロールなど）であり、コードの供給（zip化、S3へのアップロード、ビルドパイプラインなど）は別途設計が必要です。

チームの運用に合わせてCI/CDパイプラインでビルド→zip化→S3アップロード→Terraformでデプロイという流れにするのか、開発者がローカルでビルドしたものを使うのかなどを決めておかないと、Terraform側の`source_code_hash`が毎回変わってplanが安定しません。

### Terraformで管理しないという選択肢

冒頭でも書きましたがTerraformに全てを取り込む必要はなく、機密情報を多く含むリソースや、コードのライフサイクルがインフラと異なるリソース（Lambdaのコードなど）は、あえてTerraform管理の対象外にする方が安全な場合もあります。

管理しないと決めたリソースは`terraform state rm`でstateから外し、コードからも削除しておけばTerraformは関知しなくなります。importしたからといってすべてを維持する義務はないので、管理コストとリスクのバランスで判断しましょう。

## 6. planの差分との向き合い方

Terraformerでimportした直後に`terraform plan`を実行すると、大量の差分が表示されて心が折れそうになります。しかし、この差分の多くはインフラが壊れているわけではなく、AWSの内部表現とTerraformの表現が一致していないだけです。

例えば、AWSコンソール上では空文字列で設定されている属性が、Terraform側では`null`として扱われるケースがあります。多くは表現の違いだけで、applyしても結果が変わらないことが多いです。ただし、providerの実装によっては表現差に見える差分でもUpdate/Replaceが走ることがあるので、planの出力は精読する必要があります。

### よくある差分のパターン

差分を整理すると、だいたい以下のパターンに分類できます。

- **デフォルト値の差分**：Terraformerが出力した属性がTerraformのデフォルト値と同じで、書いても書かなくても結果が変わらないもの（例：`enable_dns_support = true`はVPCのデフォルト）
- **空文字列 vs null**：AWSが空文字列を返すが、Terraformは`null`を期待するケース
- **順序の違い**：セキュリティグループのルールやIAMポリシーのStatement順序が異なるだけで、内容は同一のもの
- **計算属性の差分**：`arn`や`id`など、AWS側が自動生成する属性がコードに含まれていて差分として表示される
- **実際の変更**：Terraformerが取得できなかった属性や、コード修正によって生じた本物の差分

これらの差分に対するアプローチは、**コードを修正して消す**、**`ignore_changes`で無視する**、**stateから外す**の3つです。すべてをコード修正で解消する必要はなく、差分ごとにどれが適切かを判断していきます。

### コードの修正で消す

デフォルト値と同じ属性を削除する、順序を揃える、不要な計算属性を消すなど、コード側の調整で解消できる差分はまずこれで対処します。差分の大半はこれで片付きます。

### ignore_changesで無視する

Terraformの`lifecycle`ブロックにある`ignore_changes`は、指定した属性の変更をTerraformに無視させる設定です。

例えばECSサービスの`desired_count`はAuto Scalingによって動的に変わります。これをTerraformで固定してしまうと、applyするたびにスケーリングがリセットされてしまいます。

```hcl
resource "aws_ecs_service" "app" {
  # ...

  lifecycle {
    ignore_changes = [
      # Auto Scalingがdesired_countを動的に変更するため
      desired_count,
    ]
  }
}
```

一方で、差分の原因を調べずにとりあえず`ignore_changes`に追加するのは避けるべきです。コード修正で解消できる差分（デフォルト値の明示、順序の修正など）や、セキュリティグループのルールのように変更を検知したい属性には使わない方がいいです。

運用上の注意点として、なぜignoreしているのかコメントで理由を残すこと、`ignore_changes = all`は避けること（全属性を無視するならstateから外すべき）を意識しておくと破綻しにくくなります。

### stateの調整

stateはTerraformがどのリソースを管理しているかの記録です。Terraformerからの移行では、以下のようなケースでstate操作が必要になることがあります。

- **リソース名の変更**：`tfer--`付きの名前を意味のある名前に変えた場合、`state mv`で対応を反映する
- **管理対象からの除外**：Terraform管理に含めないリソースを`state rm`で外す
- **構成変更**：リソースを別ファイルやモジュールに移動した場合、stateのアドレスも`state mv`で合わせる

state操作自体はクラウドを直接変更しませんが、コードとの整合が崩れると次の`apply`で破壊的変更につながります。例えば`state rm`した後にコードを消し忘れると、次の`apply`でリソースが再作成されます。`state mv`の移動先を間違えると別リソース扱いになりrecreateが走ることもあります。操作前に`terraform state pull > backup.tfstate`でバックアップを取り、操作後は必ず`plan`で意図しない差分が出ていないか確認してください。

```bash
# リソース名の変更
terraform state mv aws_security_group.tfer--sg-xxxxx aws_security_group.web_app

# 管理対象から外す
terraform state rm aws_lambda_function.legacy_function
```

## 7. まとめ

Terraformerはimportコマンド一発で既存リソースのHCLとstateが生成されるため、それだけで移行が完了したように見えますが実際にはそこからが本番で、コードの整理、差分の解消、機密情報の分離、state統合といった地道な作業が続きます。

大変ではありますが、完了すればインフラの可視性と再現性が大きく向上します。この記事がこれからTerraformerを使おうとしている方の参考になれば幸いです。
