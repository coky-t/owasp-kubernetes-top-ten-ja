## 概要
[ロールベースのアクセス制御](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (Role-Based Access Control, RBAC) は Kubernetes の主要な認可メカニズムで、リソースに対するパーミッションを担っています。これらのパーミッションは動詞 (get, create, delete など) とリソース (pods, services, nodes など) を組み合わせ、名前空間やクラスタスコープにできます。クライアントが実施したいアクションに応じて適切なデフォルトの責任分担を備えている、すぐに使えるロールのセットを提供しています。最小権限の適用で RBAC を設定することは後述の理由により簡単ではありません。

![Overly Permissive RBAC - Illustration](/assets/images/K03-2022.gif)

## 説明
RBAC は適切に設定された場合、Kubernetes の非常に強力なセキュリティ施行メカニズムですが、侵害が発生した場合にはすぐにクラスタの大きなリスクとなり、被害範囲が拡大する可能性があります。以下は RBAC の設定ミスの例です。

### `cluster-admin` の不要な使用

Service Account, User, Group などの subject が `cluster-admin` と呼ばれるビルトインの Kubernetes "supreuser" にアクセスできる場合、クラスタ内のあらゆるリソースに対してあらゆるアクションを実施できます。このレベルのパーミッションはクラスタ全体のすべてのリソースを完全に制御することを許可する `ClusterRoleBinding` で使用されると特に危険です。また `cluster-admin` は `RoleBinding` で使用できますが、これも重大な危険をもたらす可能性があります。

以下はある有名な OSS Kubernetes 開発プラットフォームの RBAC 設定です。これは `default` サービスアカウントにバインドされている非常に危険な `ClusterRoleBinding` を示しています。なぜこれが危険なのでしょうか？これは `default` 名前空間にあるすべての Pod に非常に強力な `cluster-admin` 権限を付与します。デフォルト名前空間の pod が侵害された場合 (リモートコード実行を考えてみてください) 、攻撃者がサービスになりすましてクラスタ全体を侵害することは簡単なことなのです。

```yaml
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

#### 防止方法

攻撃者が RBAC 設定を悪用するリスクを減らすには、設定を継続的に分析し、最小権限の原則が常に適用されていることを確認することが重要です。推奨事項をいくつか以下に示します。

- エンドユーザーによるクラスタへの直接アクセスを可能な限り減らす
- サービスアカウントトークンをクラスタ外で使用しない
- デフォルトサービスアカウントトークンを自動的にマウントすることを避ける
- インストールされているサードパーティコンポーネントに含まれる RBAC を監査する
- 一元管理されたポリシーをデプロイし、リスクのある RBAC パーミッションを検出およびブロックする
- `RoleBindings` を利用するには、クラスタ全体の RBAC ポリシーではなく、特定の名前空間にパーミッションの範囲を制限する
- Kubernetes ドキュメントにある公式の [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) に従う

#### 攻撃シナリオの例
プラットフォームエンジニアリングチームがプライベート Kubernetes OSS クラスタ内にクラスタ観測ツールをインストールしたとします。このツールはトラフィックをデバッグおよび分析するためのウェブ UI が含まれています。UI は含まれているサービスマニフェストを通じてインターネットに公開されます。type: LoadBalancer を使用しており、AWS ALB ロードバランサを **パブリック** IP アドレスで起動します。

この架空のツールは以下の RBAC 設定を使用しています。

```yaml
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

#### 参考資料

Kubernetes RBAC: <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>

RBAC Police Scanner: <https://github.com/PaloAltoNetworks/rbac-police>

Kubernetes RBAC Good Practices: <https://kubernetes.io/docs/concepts/security/rbac-good-practices/>

### `LIST` パーミッションの不要な使用

リストのレスポンスにはそれらの名前だけではなくすべてのアイテムが完全に含まれています。 `LIST` パーミッションを持つアカウントは API から特定のアイテムを取得することはできませんが、リストする際にすべてのアイテムを完全に取得できます。

kubectl はオブジェクト名のみを表示するような選択となるようにデフォルトでこれを隠していますが、それらのオブジェクトのすべての属性を持っています。



#### 防止方法
`LIST` パーミッションはそのアカウントにそのリソースのすべてを `GET` することを許可している場合にのみ付与してください。

#### 攻撃シナリオの例

```bash

# Create example A, which can only list secrets in the default namespace
# It does not have the GET permission
kubectl create serviceaccount only-list-secrets-sa
kubectl create role only-list-secrets-role --verb=list --resource=secrets
kubectl create rolebinding only-list-secrets-default-ns --role=only-list-secrets-role --serviceaccount=default:only-list-secrets-sa
# Now to impersonate that service account
kubectl proxy &
# Create a secret to get
kubectl create secret generic abc --from-literal=secretAuthToken=verySecure123
# Prove we cannot get that secret
curl http://127.0.0.1:8001/api/v1/namespaces/default/secrets/abc -H "Authorization: Bearer $(kubectl -n default get secrets -ojson | jq '.items[]| select(.metadata.annotations."kubernetes.io/service-account.name"=="only-list-secrets-sa")| .data.token' | tr -d '"' | base64 -d)"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
  },
  "status": "Failure",
  "message": "secrets \"abc\" is forbidden: User \"system:serviceaccount:default:only-list-secrets-sa\" cannot get resource \"secrets\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "name": "abc",
    "kind": "secrets"
  },
  "code": 403
}
# Now to get all secrets in the default namespace, despite not having "get" permission
curl http://127.0.0.1:8001/api/v1/namespaces/default/secrets?limit=500 -H "Authorization: Bearer $(kubectl -n default get secrets -ojson | jq '.items[]| select(.metadata.annotations."kubernetes.io/service-account.name"=="only-list-secrets-sa")| .data.token' | tr -d '"' | base64 -d)"
{
  "kind": "SecretList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/default/secrets",
    "resourceVersion": "17718246"
  },
  "items": [
  REDACTED : REDACTED
  ]
}
# Cleanup
kubectl delete serviceaccount only-list-secrets-sa
kubectl delete role only-list-secrets-role 
kubectl delete rolebinding only-list-secrets-default-ns 
kubectl delete secret abc
# Kill backgrounded kubectl proxy
kill "%$(jobs | grep "kubectl proxy" | cut -d [ -f 2| cut -d ] -f 1)"
```

#### 参考資料
Why list is a scary permission on k8s: <https://tales.fromprod.com/2022/202/Why-Listing-Is-Scary_On-K8s.html>
Kubernetes security recommendations for developers: <https://kubernetes.io/docs/concepts/configuration/secret/#security-recommendations-for-developers>

### `WATCH`パーミッションの不要な使用

ウォッチのレスポンスには更新された際にそれらの名前だけではなくすべてのアイテムが完全に含まれています。 `WATCH` パーミッションを持つアカウントは API から特定の値を取得したり、すべてのアイテムをリストすることはできませんが、ウォッチ呼び出し中にはすべてのアイテムを完全に取得し、ウォッチが中断されなければ新しいアイテムもすべて取得します。


#### 防止方法
`WATCH` パーミッションはそのアカウントにそのリソースのすべてを `GET` および `LIST` することを許可している場合にのみ付与してください。

![Overly Permissive RBAC - Mitigations](/assets/images/K03-2022-mitigation.gif)

#### 攻撃シナリオの例
```bash

# Create example A, which can only watch secrets in the default namespace
# It does not have the GET permission
kubectl create serviceaccount only-watch-secrets-sa
kubectl create role only-watch-secrets-role --verb=watch --resource=secrets
kubectl create rolebinding only-watch-secrets-default-ns --role=only-watch-secrets-role --serviceaccount=default:only-watch-secrets-sa
# Now to impersonate that service account
kubectl proxy &
# Create a secret to get
kubectl create secret generic  abcd  --from-literal=secretPassword=verySecure
# Prove we cannot get that secret
curl http://127.0.0.1:8001/api/v1/namespaces/default/secrets/abcd -H "Authorization: Bearer $(kubectl -n default get secrets -ojson | jq '.items[]| select(.metadata.annotations."kubernetes.io/service-account.name"=="only-watch-secrets-sa")| .data.token' | tr -d '"' | base64 -d)"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
  },
  "status": "Failure",
  "message": "secrets \"abc\" is forbidden: User \"system:serviceaccount:default:only-watch-secrets-sa\" cannot get resource \"secrets\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "name": "abcd",
    "kind": "secrets"
  },
  "code": 403
}

# Prove we cannot list the secrets either
curl http://127.0.0.1:8001/api/v1/namespaces/default/secrets?limit=500 -H "Authorization: Bearer $(kubectl -n default get secrets -ojson | jq '.items[]| select(.metadata.annotations."kubernetes.io/service-account.name"=="only-watch-secrets-sa")| .data.token' | tr -d '"' | base64 -d)"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "secrets is forbidden: User \"system:serviceaccount:default:only-watch-secrets-sa\" cannot list resource \"secrets\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "secrets"
  },
  "code": 403
}

# Now to get all secrets in the default namespace, despite not having "get" permission
curl http://127.0.0.1:8001/api/v1/namespaces/default/secrets?watch=true -H "Authorization: Bearer $(kubectl -n default get secrets -ojson | jq '.items[]| select(.metadata.annotations."kubernetes.io/service-account.name"=="only-watch-secrets-sa")| .data.token' | tr -d '"' | base64 -d)"

{
  "type": "ADDED",
  "object": {
    "kind": "Secret",
    "apiVersion": "v1",
    "metadata": {
      "name": "abcd",
      "namespace": "default",
      "selfLink": "/api/v1/namespaces/default/secrets/abcd",
      "uid": "725c84ee-8dc7-41ef-a03e-193225e228b2",
      "resourceVersion": "1903164",
      "creationTimestamp": "2022-09-09T13:39:43Z",
      "managedFields": [
        {
          "manager": "kubectl-create",
          "operation": "Update",
          "apiVersion": "v1",
          "time": "2022-09-09T13:39:43Z",
          "fieldsType": "FieldsV1",
          "fieldsV1": {
            "f:data": {
              ".": {},
              "f:secretPassword": {}
            },
            "f:type": {}
          }
        }
      ]
    },
    "data": {
      "secretPassword": "dmVyeVNlY3VyZQ=="
    },
    "type": "Opaque"
  }
}
REDACTED OTHER SECRETS
# crtl+c to stop curl as this http request will continue

# Proving that we got the full secret
echo "dmVyeVNlY3VyZQ==" | base64 -d
verySecure

# Cleanup
kubectl delete serviceaccount only-watch-secrets-sa
kubectl delete role only-watch-secrets-role 
kubectl delete rolebinding only-watch-secrets-default-ns --role=only-list-secrets-role --serviceaccount=default:only-list-secrets-sa
kubectl delete secret abcd
# Kill backgrounded kubectl proxy
kill "%$(jobs | grep "kubectl proxy" | cut -d [ -f 2| cut -d ] -f 1)"
```

#### 参考資料
Kubernetes security recommendations for developers: <https://kubernetes.io/docs/concepts/configuration/secret/#security-recommendations-for-developers>
