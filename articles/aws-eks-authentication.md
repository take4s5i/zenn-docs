---
title: EKSの認証・認可の仕組み解説
emoji: "🐷"
type: tech
topics:
  - aws
  - eks
  - fargate
  - kubernetes
published: true

---
EKS の認証・認可では *AWS の IAM 認証* と *Kubernetes の認証*という２つの要素が登場するため
少し分かりづらいものになっています。

この記事ではEKSの認証・認可について解説します。

EKSで認証・認可では次の２つの仕組みが存在します。
- ConfigMap aws-auth
- IAM Roles for Service Accounts (IRSA)

それでは見ていきましょう。

# aws-auth
最初は`aws-auth`です。
`aws-auth`はIAMエンティティに対して, Kubernetesの権限を付与するための設定(ConfigMap)です。

次のような場合に利用します。
- 管理者や開発者がkubectlを使ってEKSクラスタにアクセスする
- Code Build等のCIサービスからEKSクラスタにデプロイする
- EKSのコンソールにpod等のKubernetesワークロードを表示する

## 大まかな流れ
![](https://storage.googleapis.com/zenn-user-upload/1upundek9nhlawydlfyd3jyp5u2q)

1. `aws eks update-kubeconfig`を実行
2. `aws eks describe-cluster`が実行され、クラスタのエンドポイント等を取得
3. `~/.kube/config`にクラスタエンドポイントや5のトークン取得コマンドが書き込まれる
4. `kubectl`を実行
5. `aws eks get-token`が実行されクラスタにアクセスするためのトークンを取得
6. トークンを使ってクラスタにアクセス
7. `aws-auth`のConfigMapのマッピング情報を元にk8sの認証・認可を処理

## update-kubeconfig
`aws eks update-kubeconfig`はEKSクラスタに接続するためのkubectlの設定(=`kubeconfig`)を生成してくれる便利コマンドです。

実際にEKSクラスタに接続するための情報、手順は`kubeconfig`に書き込まれますので１度だけ実行すればよいです。

`kubeconfig`には次のような内容が書き込まれます。
`server`, `exec`等の接続に必要な情報が書き込まれていることがわかるかと思います。

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: XXXXXXXXXXXXXXXXXXXXXXXXX
    server: https://XXXXXXXXXXXXXXXXXXXXXXXXX.sk1.ap-northeast-1.eks.amazonaws.com
  name: arn:aws:eks:ap-northeast-1:375137234141:cluster/cluster-name
contexts:
- context:
    cluster: arn:aws:eks:ap-northeast-1:375137234141:cluster/cluster-name
    user: arn:aws:eks:ap-northeast-1:375137234141:cluster/cluster-name
  name: arn:aws:eks:ap-northeast-1:375137234141:cluster/cluster-name
current-context: arn:aws:eks:ap-northeast-1:375137234141:cluster/cluster-name
preferences: {}
users:
- name: arn:aws:eks:ap-northeast-1:375137234141:cluster/cluster-name
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      args:
      - --region
      - ap-northeast-1
      - eks
      - get-token
      - --cluster-name
      - cluster-name
      command: aws
      env:
      - name: AWS_PROFILE
        value: my-aws-profile
```

## aws-auth
EKSではクラスタ作成時に`kube-system`namespaceに`aws-auth`というConfigMapが作成されます.

`aws-auth`は次のようになっています.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - groups:
      - system:masters
      rolearn: arn:aws:iam::123456789012:role/my-role
      username: my-k8s-role
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/my-user
      username: my-k8s-user
      groups:
      - system:masters
```

`aws-auth`にはIAM RoleまたはIAM Userに対して、kubernetes上でどのようなgroupに割り当てるかを指定します.

先程の例で言うと

- `arn:aws:iam::123456789012:role/my-role`というIAM Roleに対して
  - k8sのusernameは`my-k8s-role`
  - k8sのgroupは`system:masters`に所属
- `arn:aws:iam::123456789012:user/my-user`
  - k8sのusernameは`my-k8s-user`
  - k8sのgroupは`system:masters`に所属

という設定がされていることになります。

`aws-auth`の詳細は[こちら](https://github.com/kubernetes-sigs/aws-iam-authenticator#full-configuration-format)が参考になりました。

# IAM Roles for Service Accounts(IRSA)
次にIRSAです。
IAM Roles for Service Accounts(IRSA)はKubernetes上のService Accountに対してIAM Roleを割り当てるための仕組みです。

`eksctl`では`iam service account`とも呼ばれていました。

IRSAは次のような場合に利用します。
- EKSのpodとして稼働しているアプリケーションからAWSサービスを利用する
- [AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller)のようなAWSサービスを利用するアプリケーションをデプロイする場合

## 大まかな流れ
![](https://storage.googleapis.com/zenn-user-upload/78ng3ien2zhvyub1equh136ue1zc)

1. EKSにビルトインされているOIDC ID ProviderをIAMに登録
2. IAM Roleを作成し、OIDC ID Provider経由でログインしたユーザ(=ServiceAccount)に対してRoleの引き受けを許可
3. IAM Roleにポリシーをアタッチし、パーミッションを付与
4. Service Accountを作成し、Podに割り当て
5. PodからSAを通してIAM Roleを引き受け

## OIDC ID Provider
EKSクラスタにはOIDC ID Providerがビルトインされています。

またIAMにはOIDC ID Providerと連携してOIDC上のユーザに対してIAM Roleを割り当てる機能(IDフェデレーション)があります。

これらの仕組みでService Account Tokenを利用してIAMのIDフェデレーションを行うことで、
Service AccountをIAMのアイデンティティとして利用出来るようになります

## Roleの引き受け
次のようなAssumeRolePolicyDocumentを設定することで、Service Account(でIDフェデレーションされたIAM アイデンティティ)
に対してRoleの引き受けを許可します。

これでService AccountがIAM RoleとしてAWSサービスにアクセス出来るようになりました。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com",
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:<my-namespace>:<my-service-account>"
        }
      }
    }
  ]
}
```

## Service Accountの設定
kubernetes側ではService Accountを作成し、Podに割り当てます。

Service Accountに対しては次のような設定が必要になります。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/<IAM_ROLE_NAME>
```

このannotationを設定することで、Podに対して環境変数`AWS_WEB_IDENTITY_TOKEN_FILE`, `AWS_ROLE_ARN`が自動的にinjectされるようになります。

Podの中のアプリケーションはこの環境変数を利用してIAM認証を行います。

aws cliやaws-sdkを利用している場合は勝手にやってくれるので、特に環境変数の存在を意識する必要はありません。

# おわりに
更に詳しい情報やリファレンス等はドキュメントを参照しましょう。
参考になりそうなリンクを貼っておきます。

- [ID プロバイダーとフェデレーション - AWS Identity and Access Management](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles_providers.html)
- [IAM roles for service accounts - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Managing users or IAM roles for your cluster - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)
- [RBAC認可を使用する | Kubernetes](https://kubernetes.io/ja/docs/reference/access-authn-authz/rbac/#rolebinding%E3%81%A8clusterrolebinding)

この記事が理解の一助になれば幸いです。
