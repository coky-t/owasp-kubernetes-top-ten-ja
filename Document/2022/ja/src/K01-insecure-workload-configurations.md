## 概要

Kubernetes におけるワークロードのセキュリティコンテキストは高度に設定できてしまうため、深刻なセキュリティの設定ミスが組織のワークロードやクラスタ全体に広がる可能性があります。Redhat が実施した [2021 Kubernetes Security Survey](https://www.redhat.com/en/resources/kubernetes-adoption-security-market-trends-2021-overview) によると、回答者の 60% 近くが過去 12 か月間に Kubernetes 環境での設定ミスのインシデントを経験しています。

## 説明

Kubernetes マニフェストには特定のワークロードの信頼性、セキュリティ、スケーラビリティに影響を与える可能性のあるさまざまな設定が多く含まれています。これらの設定は継続的に監査し修正する必要があります。影響の大きいマニフェスト設定の例をいくつか以下に示します。

**アプリケーションプロセスは root として実行すべきではない:** コンテナ内のプロセスを `root` ユーザーで実行することは多くのクラスタに共通する設定ミスです。ワークロードによっては `root` が絶対に必要となることもありますが、可能であれば避けるべきです。コンテナが侵害された場合、攻撃者はルートレベルの権限を持ち、悪意のあるプロセスを開始するなど、システム上の他のユーザーが許可されていないアクションが可能になります。

```
apiVersion: v1  
kind: Pod  
metadata:  
  name: root-user
spec:  
  containers:  
	...
  securityContext:  
    #root user
    runAsUser: 0
	#non-root user
	runAsUser: 5554	
```


**読み取り専用ファイルシステムを使用すべきである:** Kubernetes ノード上で侵害されたコンテナの影響を制限するために、可能な限り読み取り専用のファイルシステムを利用することをお勧めします。これにより悪意のあるプロセスやアプリケーションがホストシステムに書き戻すことを防ぎます。読み取り専用のファイルシステムはコンテナのブレイクアウトを防ぐための重要なコンポーネントです。

```
apiVersion: v1  
kind: Pod  
metadata:  
  name: read-only-fs
spec:  
  containers:  

  securityContext:  
	#read-only fs explicitally defined
    readOnlyRootFilesystem: true
```


**特権コンテナを禁止すべきである**: Kubernetes 内でコンテナを `privileged` に設定すると、コンテナ自体がホスト自体の他のリソースやカーネル機能にアクセスできます。特権コンテナはビルトインのコンテナ分離メカニズムの多くを完全に取り除いてしまうため危険です。

```
apiVersion: v1  
kind: Pod  
metadata:  
  name: priviliged-pod
spec:  
  containers:  
	...
  securityContext:  
    #priviliged 
    privileged: true
	#non-priviliged 
	priviliged: false
```

## 防止方法

大規模で分散した Kubernetes 環境全体でセキュアな設定を維持することは、困難な作業となる可能性があります。多くのセキュリティ設定はマニフェスト自体の `securityContext` に設定されることが多いのですが、他の場所で検出される設定ミスも多くあります。設定ミスを防ぐには、まず実行時とコード内の両方で検出しなければなりません。

Open Policy Agent などのツールは一般的な設定ミスを検出するポリシーエンジンとして使用できます。また Kubernetes 要の CIS ベンチマークも設定ミスを発見するための出発点として使用できます。


### 攻撃シナリオの例



### 参考資料

CIS Benchmarks for Kubernetes: [https://www.cisecurity.org/benchmark/kubernetes](https://www.cisecurity.org/benchmark/kubernetes)

Open Policy Agent: [https://github.com/open-policy-agent/opa](https://github.com/open-policy-agent/opa)

Pod Security Standards: [https://kubernetes.io/docs/concepts/security/pod-security-standards/](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
