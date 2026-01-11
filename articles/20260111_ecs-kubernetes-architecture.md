---
title: "Kubernetesのノリで学ぶECSの仕組み"
emoji: "🚢"
type: "tech"
topics: ["aws", "ecs", "kubernetes", "コンテナ"]
published: false
---

## 0. はじめに

### 読者前提

この記事は以下の方を想定しています。

- Kubernetesのアーキテクチャ（desired state / controller loop / scheduler / kubelet など）を理解している
- ECSを触ったことがある（Service / TaskDefinition / Task の用語は知っている）
- ただし EKS は触っていない方も読む想定

### 本稿の範囲

本稿は **「ECS vs Kubernetes（一般）」** として、ECSの仕組みをk8s脳に翻訳することが目的です。

EKSは "AWSでKubernetesを動かす代表例" として名前が出る程度に留め、**EKS特有の運用論（アップグレード、アドオン選定、ベストプラクティス等）には踏み込みません**。

### この記事のゴール（1行で持ち帰ってほしいこと）

**「ECSもKubernetesも、desired state を制御ループ（reconciliation）で現実に寄せて収束させる」**

このシンプルな設計パターンの共通性を理解すれば、k8sで学んだ思考法がそのままECSにも適用できます。

---

## 1. まず"同じ骨格"を確認する（設計パターンの共通項）

KubernetesもECSも、根底にある設計思想は同じです。

### Desired State（宣言的な状態）

ユーザーが「こうあってほしい」という状態を宣言する。

- **Kubernetes**: Deployment YAML の `replicas: 3`
- **ECS**: Service の `desiredCount: 3`

### Reconciliation Loop（制御ループ）

コントローラが現実の状態を監視し、desired stateに収束させ続ける。

- **Kubernetes**: Deployment Controller、ReplicaSet Controller
- **ECS**: ECS Service Scheduler（内部コンポーネント）

### Scheduling（配置決定）

新しいワークロードを「どこに」置くかを決める。

- **Kubernetes**: kube-scheduler
- **ECS**: ECS Placement（capacity provider + placement strategy/constraint）

### Data Plane / Node Agent（実行主体）

実際にコンテナを起動・監視する。

- **Kubernetes**: kubelet（各ノード上）
- **ECS**: ECS Agent（EC2モード）/ 不可視（Fargate）

### ここで宣言：ECSとKubernetesは同型

両者は**異なるAPIと用語を使っているが、制御構造は同型**です。
以降、この対応関係を具体的に見ていきます。

---

## 2. 翻訳辞書：k8s脳→ECS脳の対応表

| Kubernetes | ECS | 備考 |
|------------|-----|------|
| **Deployment / ReplicaSet** | **Service** | desired state（レプリカ数、ロールアウト制御）を管理 |
| **Pod** | **Task** | 実行単位（1つ以上のコンテナ） |
| **PodSpec（コンテナ定義）** | **TaskDefinition** | immutableな設計図（revision管理） |
| **kube-scheduler** | **ECS Placement** | どこに配置するかを決定 |
| **kubelet** | **ECS Agent（EC2）/ 不可視（Fargate）** | コンテナ実行とControl Planeとの同期 |
| **kubectl describe / events** | **Service Events / Task Stopped Reason** | 状態遷移と異常の観測 |
| **Service（ClusterIP/LB）** | **Target Group + awsvpc** | サービス発見とロードバランシング |
| **ServiceAccount + IRSA** | **Task Role + Execution Role** | IAM権限の責務分離 |
| **maxSurge / maxUnavailable** | **maximumPercent / minimumHealthyPercent** | ロールアウト中の並走制御（発想は同じ、言葉は逆） |

この表を頭に入れておけば、ECSのドキュメントを読むときに「あ、これk8sの◯◯だ」とピンときます。

---

## 3. desired state の単位を分解する：Service / TaskDefinition / Deployment

### TaskDefinition は immutable な設計図

TaskDefinitionは**イミュータブル**です。変更するとrevisionが上がります。

```plaintext
my-app:1 → my-app:2 → my-app:3
```

これはk8sのPodSpecに近いですが、k8sと違って**世代がバージョン管理される**点が特徴です。

### Service は desired state の主体

ECS Serviceは以下を管理します。

- **desiredCount**（何個動かすか）
- **使用するTaskDefinition**（どのrevisionを使うか）
- **deployment configuration**（ロールアウトの制約）
- **placement strategy / constraint**（どこに置くか）

### Deployment（ECS用語）は「世代の並走→収束」を表す

ECS Service内部には **`deployments`** という状態オブジェクトが存在します（複数形であることに注意）。

```plaintext
┌─ ECS Service ─────────────────────┐
│ desiredCount: 3                   │
│                                   │
│ deployments:                      │
│  - PRIMARY (my-app:2)  → 3 tasks  │  ← 新しいrevision
│  - ACTIVE  (my-app:1)  → 0 tasks  │  ← 古いrevisionが収束完了
└───────────────────────────────────┘
```

ロールアウト中は**PRIMARY（新）とACTIVE（旧）が並走**し、徐々に新しい方にシフトします。
これはk8sのReplicaSetの並走（古いRSのreplicas減、新しいRS増）と同じ考え方です。

---

## 4. ECSのreconciliation loop：何を入力に、何を出力するのか

### 入力（観測する状態）

ECS Service Schedulerは以下を入力として受け取ります。

- **deployments の desired / running / pending**
- **Task の state**（RUNNING / PENDING / STOPPED）
- **（LB連携時の）ヘルスチェック結果**
- **Capacity Provider の空き状況**（Fargate / EC2）

### 出力（制御アクション）

- **RunTask**（新しいTaskを起動）
- **StopTask**（古いTaskを停止）
- **待機**（制約を満たすまで何もしない）

### deployment config の制約

ロールアウト中の並走数と健全性を制御するパラメータ。

| パラメータ | 意味 | k8sでの類似概念 |
|-----------|------|----------------|
| **minimumHealthyPercent** | 最低何%を健全に保つか | 100 - maxUnavailable |
| **maximumPercent** | 最大何%まで並走を許すか | 100 + maxSurge |

例：`minimumHealthyPercent: 50, maximumPercent: 200, desiredCount: 4`

```plaintext
┌────────────────────────────────────┐
│ 最低 2 (50%) は常に健全             │
│ 最大 8 (200%) まで並走可能          │
│                                    │
│ → 古い4つを残したまま新4つ起動OK   │
│ → 新が健全になったら古を停止        │
└────────────────────────────────────┘
```

このループは**外から観測できる事実（task state, service events）に基づいて推測する**しかありませんが、動作パターンは上記の制約に従います。

---

## 5. スケジューリング：kube-scheduler相当をECSでどう捉えるか

ECSのスケジューリングは**二段階**で考えると整理しやすいです。

### ① どのキャパシティを使うか

- **Fargate**: ノード管理不要、タスク単位で起動
- **EC2 (Capacity Provider)**: クラスタ内のEC2インスタンス
- **External (ECS Anywhere)**: オンプレミス等

これはk8sの「どのノードプールに配置するか」に相当しますが、ECSでは**Capacity Provider**という抽象で表現されます。

### ② その中でどこに置くか（Placement）

**Placement Strategy**:

- `spread`: AZ/インスタンスにばらまく
- `binpack`: リソース効率を優先して詰め込む
- `random`: ランダム

**Placement Constraint**:

- `memberOf`: 特定の属性を持つインスタンスに限定（例：`attribute:ecs.instance-type =~ t3.*`）

### Fargateの場合

ノード選択が抽象化され、制約は**ネットワーク（サブネット）とリソース要求**の形で表現されます。
kubeletやノード側の制約（taint/toleration等）は意識する必要がありません。

---

## 6. Data Plane と責務境界：ECS agent / Fargate の見え方

### EC2モード：ECS Agent が見える

```plaintext
┌─ Control Plane (ECS) ─┐
│  Service Scheduler     │
└────────┬───────────────┘
         │ (API)
┌────────▼───────────────┐
│  ECS Agent (EC2上)     │  ← kubeletに相当
│  - Task状態の報告      │
│  - コンテナ起動/停止   │
│  - Docker/containerdと通信 │
└────────────────────────┘
```

EC2上の**ECS Agent**はControl Planeと同期し、タスクの起動/停止を担います。
これはkubeletと同じ責務です。

### Fargate：ノード側が不可視

```plaintext
┌─ Control Plane (ECS) ─┐
│  Service Scheduler     │
└────────┬───────────────┘
         │ (API)
┌────────▼───────────────┐
│  Fargate（ブラックボックス）│  ← ノード側に介入できない
│  - タスク単位で起動    │
│  - インフラ管理不要    │
└────────────────────────┘
```

Fargateは**ノード側の部品が完全に隠蔽**されます。
ユーザーが観測できるのは**タスクの状態・理由・イベント**のみです。

### k8sとの違い

- **k8s**: ノード側の部品（kubelet / CNI / CSI）が露出し、カスタマイズ可能
- **ECS**: タスク単位の状態/理由/イベントが主語。ノード側は（Fargateでは）ブラックボックス

---

## 7. awsvpc を中心にネットワークを語る（タスク=ネットワーク境界）

### タスクがENIを持つという設計

ECSの**awsvpcモード**では、**1 Task = 1 ENI**です。

```plaintext
┌─ VPC ─────────────────────────────┐
│                                   │
│  ┌─ Task (ENI) ─┐                │
│  │  IP: 10.0.1.5  │  ← セキュリティグループを直接適用 │
│  │  Container A   │                │
│  │  Container B   │                │
│  └────────────────┘                │
│                                   │
└───────────────────────────────────┘
```

### これが意味すること

- **セキュリティグループ**: タスクレベルで直接適用（PodにSGを当てるイメージ）
- **到達性**: タスクIPがVPC内で直接ルーティング可能
- **サービス発見**: Cloud Map / Target GroupがタスクIPを直接参照
- **ロードバランシング**: ALB/NLBがタスクIPに直接トラフィックを送る

### k8sのPod IPとの近さと違い

- **近さ**: どちらも「ワークロード単位でIPを持つ」
- **違い**: ECSはAWSネットワーク制約（ENI上限、サブネット設計）が**直に効く**
  - ENI数上限はインスタンスタイプ依存
  - Fargateは1タスク=1ENIが強制

---

## 8. IAM：execution role と task role を"責務分離"として理解する

ECSには2種類のIAMロールがあります。

### Execution Role（起動のため）

タスク**起動時**にECSが必要とする権限。

- ECRからイメージをpull
- CloudWatch Logsにログを送信
- Secrets Managerから環境変数を注入

**誰が使うか**: ECS（Control Plane側）

### Task Role（実行中のアプリのため）

タスク**実行中**にアプリケーションがAWS APIを叩く権限。

- S3からファイルを取得
- DynamoDBにデータを書き込み
- SQSからメッセージを取得

**誰が使うか**: コンテナ内のアプリケーション

### k8sのServiceAccount / IRSA（一般概念）との対比

| Kubernetes | ECS | 責務 |
|------------|-----|------|
| **Node IAM Role（暗黙）** | **Execution Role** | 起動・インフラ操作 |
| **ServiceAccount + IRSA** | **Task Role** | アプリがAWS APIを叩く |

**注意**: EKSでのIRSA運用論（OIDC設定、Podへの注入方法等）は本稿のスコープ外です。
ここでは「責務分離の考え方が同じ」という点だけ押さえてください。

---

## 9. 観測：kubectl describe の代替としての "ECSの真実ソース"

k8sで `kubectl describe pod` や `kubectl get events` を見る習慣がある人向けに、ECSでの観測優先順位を示します。

### 優先順位（制御ループの判断を追う）

1. **Service Events**
   - `ecs describe-services` の `events` フィールド
   - 「なぜタスクが起動/停止したか」の判断理由が書かれている
   - 例：`(service my-service) has reached a steady state.`

2. **Deployments（Service内部の状態）**
   - PRIMARY / ACTIVE の並走状況
   - `desiredCount` / `runningCount` / `pendingCount`

3. **Task Stopped Reason / Exit Code**
   - `ecs describe-tasks` の `stoppedReason` と `containers[].exitCode`
   - 例：`Essential container in task exited`

4. **Logs / Metrics**
   - CloudWatch Logs（コンテナの標準出力）
   - CloudWatch Metrics（CPU/Memory使用率）

### なぜこの順なのか

ECSの制御ループは**Service → Deployments → Tasks**という階層で動きます。
**上位の判断**が下位に影響するため、まず上位（Service Events）から追うことで「なぜこうなったか」が見えます。

これはk8sで「Deployment → ReplicaSet → Pod → Container」と追う思考法と同じです。

---

## 10. まとめ：k8sノリでECSを読むチェックリスト

ECSを触るとき、以下を自問すると整理しやすくなります。

### ✅ desiredはどこ？

→ **Service の `desiredCount`** と **TaskDefinition のrevision**

### ✅ コントローラは誰？

→ **ECS Service Scheduler**（内部コンポーネント、直接触れない）

### ✅ どこで状態が見える？

→ **Service Events → Deployments → Task State**（この順で追う）

### ✅ どこで死んだ？

→ **Task Stopped Reason → Container Exit Code → CloudWatch Logs**

---

## おわりに

KubernetesもECSも、**desired state を制御ループで収束させる**という設計パターンは同じです。

- 用語は違う（Deployment vs Service、Pod vs Task）
- APIは違う（kubectl vs aws ecs）
- でも**制御構造は同型**

k8sで学んだ「状態」「制約」「責務境界」「入力/出力」の思考法は、そのままECSにも適用できます。

ECSを「よくわからないAWSのコンテナサービス」として捉えるのではなく、「k8sと同じ設計パターンで動く、AWS特化型のオーケストレーター」として見ると、理解が一気に進むはずです。

---

## 付録：図解

### 図1: ECS vs k8s の Control Plane / Data Plane

```plaintext
┌─────────────────────────────────────────────────────────┐
│                   Control Plane                         │
├─────────────────────┬───────────────────────────────────┤
│   Kubernetes        │   ECS                             │
│                     │                                   │
│  - API Server       │  - ECS API                        │
│  - etcd             │  - (内部DB、非公開)               │
│  - Scheduler        │  - Service Scheduler + Placement │
│  - Controllers      │  - (内部コントローラ、非公開)     │
└─────────────────────┴───────────────────────────────────┘
           ↓                          ↓
┌─────────────────────────────────────────────────────────┐
│                   Data Plane                            │
├─────────────────────┬───────────────────────────────────┤
│   Kubernetes        │   ECS                             │
│                     │                                   │
│  - kubelet          │  - ECS Agent (EC2)                │
│  - Container Runtime│  - Fargate (不可視)               │
│  - CNI / CSI        │  - awsvpc (ENI)                   │
└─────────────────────┴───────────────────────────────────┘
```

### 図2: ECS Service → Deployments → Tasks（世代並走→収束）

```plaintext
ECS Service
  ├─ desiredCount: 3
  │
  ├─ deployments:
  │   ├─ PRIMARY (my-app:2)  ← 新revision
  │   │   ├─ desired: 3
  │   │   ├─ running: 3
  │   │   └─ pending: 0
  │   │
  │   └─ ACTIVE (my-app:1)   ← 旧revision
  │       ├─ desired: 0      ← 収束完了
  │       ├─ running: 0
  │       └─ pending: 0
  │
  └─ Tasks:
      ├─ Task-A (my-app:2) → RUNNING
      ├─ Task-B (my-app:2) → RUNNING
      └─ Task-C (my-app:2) → RUNNING
```

### 図3: reconciliation loop（inputs/outputs と制約）

```plaintext
┌───────────────────────────────────────────────────────┐
│              ECS Service Scheduler                    │
│                                                       │
│  【入力】                                             │
│   - deployments (desired/running/pending)            │
│   - task state (RUNNING/PENDING/STOPPED)             │
│   - health check 結果                                 │
│   - capacity provider 空き状況                        │
│                                                       │
│  【制約】                                             │
│   - minimumHealthyPercent (最低健全性)               │
│   - maximumPercent (最大並走数)                      │
│                                                       │
│  【判断・出力】                                       │
│   → RunTask (新しいTask起動)                         │
│   → StopTask (古いTask停止)                          │
│   → 待機 (制約を満たすまで何もしない)                │
└───────────────────────────────────────────────────────┘
```

---

**Happy Container Orchestration!** 🚢
