---

layout: col-sidebar
title: "K01: 安全でないワークロード設定 (Insecure Workload Configurations)"
---

## 概要

Kubernetes におけるワークロードのセキュリティコンテキストは高度に設定できてしまうため、深刻なセキュリティの設定ミスが組織のワークロードやクラスタ全体に広がる可能性があります。
Redhat が実施した [Kubernetes adoption, security, and market trends report 2022](https://www.redhat.com/en/resources/kubernetes-adoption-security-market-trends-overview) によると、回答者の 53% 近くが過去 12 か月間に Kubernetes 環境での設定ミスのインシデントを経験しています。







![Insecure Workload Configuration - Illustration](../../../assets/images/K01-2022.gif)


## 説明

Kubernetes マニフェストには特定のワークロードの信頼性、セキュリティ、スケーラビリティに影響を与える可能性のあるさまざまな設定が多く含まれています。
これらの設定は継続的に監査し修正する必要があります。影響の大きいマニフェスト設定の例をいくつか以下に示します。



**アプリケーションプロセスは root として実行すべきではない:** コンテナ内のプロセスを `root` ユーザーで実行することは多くのクラスタに共通する設定ミスです。
ワークロードによっては `root` が絶対に必要となることもありますが、可能であれば避けるべきです。コンテナが侵害された場合、攻撃者はルートレベルの権限を持ち、悪意のあるプロセスを開始するなど、システム上の他のユーザーが許可されていないアクションが可能になります。





```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: root-user
spec:  
  containers:
  ...
  securityContext:  
    #root user:
    runAsUser: 0
    #non-root user:
    runAsUser: 5554
```

**読み取り専用ファイルシステムを使用すべきである:** Kubernetes ノード上で侵害されたコンテナの影響を制限するために、可能な限り読み取り専用のファイルシステムを利用することをお勧めします。
これにより悪意のあるプロセスやアプリケーションがホストシステムに書き戻すことを防ぎます。読み取り専用のファイルシステムはコンテナのブレイクアウトを防ぐための重要なコンポーネントです。




```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: read-only-fs
spec:  
  containers:  
  ...
  securityContext:  
    #read-only fs explicitly defined
    readOnlyRootFilesystem: true
```

**特権コンテナを禁止すべきである**: Kubernetes 内でコンテナを `privileged` に設定すると、コンテナがホストの他のリソースやカーネル機能にアクセスできます。
root で実行しているワークロードを特権コンテナと組み合わせると、ユーザーがホストへ完全にアクセスできるため壊滅的となる可能性があります。
ただし、非 root ユーザーで実行している場合、これは制限されます。特権コンテナはビルトインのコンテナ分離メカニズムの多くを完全に取り除いてしまうため危険です。





```yaml
apiVersion: v1  
kind: Pod  
metadata:  
  name: privileged-pod
spec:  
  containers:  
  ...
  securityContext:  
    #priviliged 
    privileged: true
    #non-privileged 
    privileged: false
```

**リソース制約を強制すべきである**: デフォルトでは、コンテナは Kubernetes クラスタ上の無制限のコンピュータリソースで実行します。
CPU リクエストと制限は Pod 内の個々のコンテナに帰属できます。
コンテナは CPU 制限を指定しない場合、コンテナが消費できる CPU リソースに上限がないことを意味します。
この柔軟性は好都合なこともありますが、コンテナがホスティングノード上で利用可能なすべての CPU リソースを利用する可能性があるため、クリプトマイニングなどの潜在的なリソース乱用のリスクももたらします。





```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-limit-pod
spec:
  containers:
  ...
    resources:
      limits:
        cpu: "0.5" # 0.5 CPU cores
        memory: "512Mi" # 512 Megabytes of memory
      requests:
        cpu: "0.2" # 0.2 CPU cores
        memory: "256Mi" # 256 Megabytes of memory
```

## 防止方法

大規模で分散した Kubernetes 環境全体でセキュアな設定を維持することは、困難な作業となる可能性があります。
多くのセキュリティ設定はマニフェスト自体の `securityContext` に設定されることが多いのですが、他の場所で検出される設定ミスも多くあります。
設定ミスを防ぐには、まず実行時とコード内の両方で検出しなければなりません。
アプリケーションを以下のように適用できます。



1. 非 root ユーザーで実行する
2. 非特権モードで実行する
3. AllowPrivilegeEscalation: False を設定して、子プロセスが親プロセスより高い権限を取得できないようにする
4. LimitRange を設定して、ネームスペース内の該当するオブジェクトの種類ごとにリソース割り当てを制限する



Open Policy Agent などのツールは一般的な設定ミスを検出するポリシーエンジンとして使用できます。
また Kubernetes 要の CIS ベンチマークも設定ミスを発見するための出発点として使用できます。


![Insecure Workload Configuration - Mitigations](../../../assets/images/K01-2022-mitigation.gif)


## 攻撃シナリオの例

TODO

## 参考資料

CIS Benchmarks for Kubernetes:
[https://www.cisecurity.org/benchmark/kubernetes](https://www.cisecurity.org/benchmark/kubernetes)

Open Policy Agent:
[https://github.com/open-policy-agent/opa](https://github.com/open-policy-agent/opa)

Pod Security Standards:
[https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
