## 概要
Kubernetes では、シークレットはパスワードやトークンなどの機密データを含む小さなオブジェクトです。認証情報や鍵などの機密データを保存しアクセスする方法にアクセスする必要があります。シークレットは Kubernetes エコシステムにおいて便利な機能ですが、取り扱いには細心の注意が必要です。


## 説明

Kubernetes シークレットは Kubernetes のスタンドアロン API オブジェクトで、小さなオブジェクトを格納するために使われます。他の Kubernetes オブジェクトと同様に作られます。以下は `top-secret` という名前のシークレットを作成する `.yaml` マニフェストです。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: top-secret
data:
  username: bXktdXNlcm5hbWUK
  password: bXktcGFzc3dvcmQK
type: Opaque
```

上記のマニフェスト例の `username` と `password` の値は **base64 エンコード** されており、暗号化されてはいません。このためバージョン管理や他のシステム内のシークレットがチェックされると非常に危険になります。以下ではシークレットが望まない場所に漏洩することを防ぐ方法を探ります。

## 防止方法

**保存時にシークレットを暗号化する[¶](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html#encrypt-secrets-at-rest)**

etcd データベースは一般的に Kubernetes API を介してアクセス可能なあらゆる情報を含んでおり、攻撃者にクラスタの状態に対する重大な可視性を与える可能性があります。

十分にレビューされたバックアップおよび暗号化ソリューションを使用してバックアップを常に暗号化し、可能な限りフルディスク暗号化の使用を検討してください。

Kubernetes は保存時の暗号化をサポートしています。1.7 で導入され、1.13 からベータ版となった機能です。これは etcd の Secret リソースを暗号化し、etcd のバックアップにアクセスした関係者がそれらのシークレットのコンテンツを閲覧できないようにします。この機能は現在ベータ版ですが、バックアップが暗号化されていない場合や攻撃者が etcd に読み取りアクセスした場合に追加レベルの防御を提供します。

**セキュリティ設定ミスに対処する**

シークレットを安全に保護し続けるには、すべてのクラスタにおいて強固な設定から始めることが重要です。脆弱性、イメージセキュリティ、ポリシー適用を配備して、最終的にアプリケーションを侵害から保護する必要があります。

RBAC 設定も同様にロックダウンすべきです。すべての Service Account とエンドユーザーアクセスを最小権限にとどめてください。シークレットへのアクセスについては特にそうしてください。クラスタにインストールされているサードパーティプラグインやソフトウェアの RBAC 設定を常に監査し、Kubernetes シークレットへのアクセスが不必要に許可されていないことを確認します。

**ログ記録と監査を確実に実施する**

Kubernetes クラスタはアクティビティに関する有用なメトリクスを生成し、シークレットへのアクセスを含む悪意のある動作や異常な動作を検出するのに役立ちます。 [Kubernetes Audit](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/) レコードを有効に設定し、そのストレージを一元化してください。

## 攻撃シナリオの例

攻撃者は Kubernetes で動作しているウェブアプリケーションを侵害し、シェルを取得できたとします。彼らは以下のコマンドを実行して Kubernetes シークレットがマウントされていることを確認します。

```bash
ls /var/run/secrets/kubernetes.io/serviceaccount
```

攻撃者は侵害した pod に `kubectl` をインストールし、デフォルトで上記のディレクトリにあるデフォルトサービスアカウントを使用してみます。それから攻撃者はデフォルトサービスアカウントの RBAC アクセスを利用して、内部から Kubernetes API と通信できます。RBAC がどのように設定されているかによって、攻撃者はシークレットを読んだり、クラスタ内に悪意のあるワークロードをデプロイしたりできるかもしれません。

## 参考資料

OWASP Kubernetes Cheatsheet: [https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html#securing-data](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html#securing-data)
