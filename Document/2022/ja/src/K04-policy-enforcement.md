## 概要

セキュリティポリシーを複数のクラスタ、クラウドに分散し施行すると、セキュリティチームはすぐにリスク許容度を管理できなくなります。設定ミスを一元的に検出、修正、防止できないと、クラスタが危険にさらされる可能性があります。

## 説明
Kubernetes のポリシー施行はソフトウェアデリバリライフサイクルを通じていくつかの場所で行うことができ、また行うべきです。ポリシー施行によりセキュリティおよびコンプライアンスチームはマルチクラスタ／マルチクラウドインフラストラクチャ全体にガバナンス、コンプライアンス、セキュリティ要件を適用できます。


施行ポリシーの例:

*信頼できないレジストリからのイメージを拒否すること:* 特定のクラスタで不正なイメージが実行されるのを防ぐために、イメージレジストリを明示的に許可するブロッキングアドミッションコントロールポリシーを配布することをお勧めします。 open-policy-agent と ubuntu にマッチしないレジストリからのイメージを使用するすべてのワークロードをブロックする OPA Gatekeeper Rego ポリシーの例は以下のとおりです。

```
# Allowed repos
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "sbx"
      - "prd"
  parameters:
    repos:
      - "open-policy-agent"
      - "ubuntu"
```

## 防止方法

ワークロードの設定ミスを検出するだけでは十分ではありません。チームは設定ミスのある Kubernetes オブジェクトをアドミッション時にブロックするという保証を必要としています。これは一般的に Kubernetes API 自体の [Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) によって処理されます。Kubernetes API 自体の一部として Pod Security Standards と呼ばれるビルトイン機能が存在し、クラスタ自体の [Pod Security Admission Controller](https://kubernetes.io/docs/concepts/security/pod-security-admission/) の一部としてポリシーを施行します。Privileged, Baseline, Restricted の三つのモードが用意されています。

Open Policy Agent Gatekeeper, Kyverno, Kubewarden などの他の OSS プロジェクトも同様に、クラスタ上で設定ミスのある pod がスケジュールされることを防ぐポリシー施行機能を提供しています。

## 攻撃シナリオの例
例 #1: コンテナブレイクアウト 1 ライナー

Kubernetes API に対して以下のコマンドを実行すると、高い権限があるコンテナを実行する非常に特殊な pod を作成します。まず `"hostPID": true` を見ます。これはコンテナの最も基本的な分離を解除し、すべてのプロセスがホスト上にあるかのように見ることができるようにするものです。`nsenter` コマンドはホストの `mount` 名前空間にある `pid 1` が動作している別の `mount` 名前空間に切り替わります。最後に、ワークロードが `privileged` であることを確認し、パーミッションエラーを防ぐことができます。流行しています。コンテナブレイクアウトの [ツイート](https://twitter.com/mauilion/status/1129468485480751104) です！

```
 kubectl run r00t --restart=Never -ti --rm --image lol \
	 --overrides '{"spec":{"hostPID": true, 
	 "containers":[{"name":"1","image":"alpine", 
	 "command":["nsenter","--mount=/proc/1/ns/mnt","--","/bin/bash"], 
     "stdin": true,"tty":true,"imagePullPolicy":"IfNotPresent", 
     "securityContext":{"privileged":true}}]}}' \
/
```

## 参考資料
OPA Gatekeeper: [https://github.com/open-policy-agent/gatekeeper](https://github.com/open-policy-agent/gatekeeper)

Pod Security Admission Controller: [https://kubernetes.io/docs/concepts/security/pod-security-admission/](https://kubernetes.io/docs/concepts/security/pod-security-admission/)

Kyverno: [https://kyverno.io/](https://kyverno.io/)
