## 概要
[ロールベースのアクセス制御](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (Role-Based Access Control, RBAC) は Kubernetes の主要な認可メカニズムで、リソースに対するパーミッションを担っています。これらのパーミッションは動詞 (get, create, delete など) とリソース (pods, services, nodes など) を組み合わせ、名前空間やクラスタスコープにできます。クライアントが実施したいアクションに応じて適切なデフォルトの責任分担を備えている、すぐに使えるロールのセットを提供しています。最小権限の適用で RBAC を設定することは後述の理由により簡単ではありません。

## 説明
RBAC は適切に設定された場合、Kubernetes の非常に強力なセキュリティ施行メカニズムですが、侵害が発生した場合にはすぐにクラスタの大きなリスクとなり、被害範囲が拡大する可能性があります。以下は RBAC の設定ミスの例です。

*`cluster-admin` の不要な使用*

Service Account, User, Group などの subject が `cluster-admin` と呼ばれるビルトインの Kubernetes "supreuser" にアクセスできる場合、クラスタ内のあらゆるリソースに対してあらゆるアクションを実施できます。このレベルのパーミッションはクラスタ全体のすべてのリソースを完全に制御することを許可する `ClusterRoleBinding` で使用されると特に危険です。また `cluster-admin` は `RoleBinding` で使用できますが、これも重大な危険をもたらす可能性があります。

以下はある有名な OSS Kubernetes 開発プラットフォームの RBAC 設定です。これは `default` サービスアカウントにバインドされている非常に危険な `ClusterRoleBinding` を示しています。なぜこれが危険なのでしょうか？これは `default` 名前空間にあるすべての Pod に非常に強力な `cluster-admin` 権限を付与します。デフォルト名前空間の pod が侵害された場合 (リモートコード実行を考えてみてください) 、攻撃者がサービスになりすましてクラスタ全体を侵害することは簡単なことなのです。

```jsx

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
 name: redacted-rbac
subjects:
 - kind: ServiceAccount
   name: default
   namespace: default
roleRef:
 kind: ClusterRole
 name: cluster-admin
 apiGroup: rbac.authorization.k8s.io
```

## 防止方法

攻撃者が RBAC 設定を悪用するリスクを減らすには、設定を継続的に分析し、最小権限の原則が常に適用されていることを確認することが重要です。推奨事項をいくつか以下に示します。

- エンドユーザーによるクラスタへの直接アクセスを可能な限り減らす
- サービスアカウントトークンをクラスタ外で使用しない
- デフォルトサービスアカウントトークンを自動的にマウントすることを避ける
- インストールされているサードパーティコンポーネントに含まれる RBAC を監査する
- 一元管理されたポリシーをデプロイし、リスクのある RBAC パーミッションを検出およびブロックする
- `RoleBindings` を利用するには、クラスタ全体の RBAC ポリシーではなく、特定の名前空間にパーミッションの範囲を制限する
- Kubernetes ドキュメントにある公式の [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) に従う

## 攻撃シナリオの例
プラットフォームエンジニアリングチームがプライベート Kubernetes OSS クラスタ内にクラスタ観測ツールをインストールしたとします。このツールはトラフィックをデバッグおよび分析するためのウェブ UI が含まれています。UI は含まれているサービスマニフェストを通じてインターネットに公開されます。type: LoadBalancer を使用しており、AWS ALB ロードバランサを **パブリック** IP アドレスで起動します。

この架空のツールは以下の RBAC 設定を使用しています。

```jsx
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: default-sa-namespace-admin
  namespace: prd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: system:serviceaccount:prd:default
```

攻撃者はオープンなウェブ UI を見つけ、クラスタ内の実行中のコンテナ上でシェルを取得できます。`prd` 名前空間のデフォルトサービスアカウントトークンがウェブ UI で使用されており、攻撃者はそれになりすまして Kubernetes API を呼び出し、`kube-system` 名前空間の `describe secrets` などの特権アクションを実施できます。これは `roleRef` によってそのサービスアカウントにクラスタ全体でビルトイン権限の `admin` が与えられているためです。

## 参考資料

Kubernetes RBAC: [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

RBAC Police Scanner: [https://github.com/PaloAltoNetworks/rbac-police](https://github.com/PaloAltoNetworks/rbac-police)

Kubernetes RBAC Good Practices: [https://kubernetes.io/docs/concepts/security/rbac-good-practices/](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
