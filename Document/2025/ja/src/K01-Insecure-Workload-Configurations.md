---

layout: col-sidebar
title: "K01: 安全でないワークロード設定 (Insecure Workload Configurations)"
---

## 概要

デフォルトでは、Kubernetes クラスタで実行しているワークロードは強力なセキュリティ態勢を持たず、攻撃者がアプリケーションを侵害し、その侵害を他のワークロードやクラスタのコントロールプレーンにまで拡大することが容易になる可能性があります。

クラスタ全体のセキュリティを改善するには、`securityContext` 設定を実装し、その他の堅牢化対策を適用することが不可欠です。  
説明  
デフォルトでは Kubernetes と基盤となるコンテナランタイムはシンプルなデフォルト設定を提供しており、アプリケーション開発者はクラスタ上でワークロードを実行することが簡単になります。しかし、多くの箇所でこれらのデフォルト設定は本番環境向けには堅牢化されていません。以下ではデフォルト値を変更する必要がある主な箇所をいくつか示します。

- **root として実行**。コンテナを root ユーザー (UID 0) として実行することは危険です。より広範なカーネルコードパスへのアクセスなど、そのユーザーは Linux ホスト上で特別な権限を持ち、コンテナのブレイクアウトのリスクを高めます。
- **デフォルトの Linux 機能**。コンテナランタイムはすべてのコンテナに全体的な root 権限の一部である一連の Linux 機能を提供します。これらの機能の中には危険なものがあり、攻撃者がクラスタ内の他のコンテナやシステムを侵害しようとするのを容易にする可能性があります。
- **サービスアクセストークン**。各 Pod はデフォルトでサービスアクセストークンがマウントされ、Kubernetes API へのアクセスを提供します。これにより、攻撃者はクラスタに関するより多くの情報を取得できる可能性があり、設定が誤っている場合には、クラスタの侵害につながる可能性があります。
- **seccomp フィルタの削除**。containerd などの主要なコンテナランタイムは、コンテナのブレイクアウトのリスクを軽減するのに役立つ seccomp フィルタを追加しますが、Kubernetes はデフォルトで制限なしの seccomp モードになっており、ランタイムの組み込みフィルタは明示的に設定しない限り適用されません。
- **リソース制限の欠如**。Kubernetes はデフォルトで CPU/メモリ制限を適用しません。侵害されたコンテナはノードのリソースを枯渇し、同じ場所に配置されたワークロードに対してサービス拒否を引き起こす可能性があります。

危険なデフォルトに加えて、可能な限り避けるべきワークロードセキュリティ設定がいくつかあります。

- **特権付け**。特権付けられたコンテナは基盤となるホストに脱出できるため、この設定は絶対に必要な場合を除き使用すべきではありません。
- **ホスト名前空間**。コンテナは名前空間を使用して基盤となるホストからの分離を提供しており、その保護を解除するとコンテナの分離を弱めます。ワークロードは、その機能が明示的に必要な場合を除き、ホスト名前空間を使用すべきではありません。

## 防止方法

Kubernetes ワークロードの設定をデフォルト値から改善する方法はいくつかあります。まず見るべきはセキュリティコンテキストセクションですが、一部の設定は Pod またはコンテナの一般的な仕様にもあります。

ポッドレベルまたは個々のコンテナレベルで設定できるオプションが数多くあります。これらはすべて重要ですが、クラスタで実行するワークロードのセキュリティを大幅に改善するため、すべてのワークロードが設定を検討すべき主要なオプションについて説明します。

### 非ルートとして実行

ワークロードセキュリティの最も重要な変更の一つは、アプリケーションがホスト上でルートユーザーとして実行しないようにすることです。
ルートユーザーとして実行すると、コンテナ内で動作するアプリケーションを侵害する攻撃者が、基盤となるクラスタノードへブレイクアウトすることが非常に容易になります。

これを実現できるいくつかの異なるアプローチがあります。一つ目として pod 内のすべてのコンテナに対して `runAsUser` と `runAsGroup` を指定できます。この例では、この pod 内のコンテナにあるすべてのプロセスが UID 1000 および GID 3000 として動作します。

```yaml
 securityContext:
    runAsUser: 1000
    runAsGroup: 3000
```

Kubernetes 1.33 で利用可能なもう一つのアプローチはユーザー名前空間を使用することです。コンテナ内ではルートで実行していても、ホスト上ではコンテナがルートとして動作しないように確保します。これは `hostUsers` 設定で指定できます。

```yaml
spec:
  hostUsers: false
```


### ケイパビリティのドロップ

デフォルトでは、コンテナには特権操作のための一連のケイパビリティを付与しています。これらのケイパビリティを削除することでコンテナブレイクアウトのリスクを低減し、ほとんどのアプリケーションはそれらを必要としていません。

```yaml
      securityContext:
        allowPrivilegeEscalation: false
        privileged: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

### readOnlyRootFilesystem の有効化

Most malware and exploit payloads require writing to /tmp or /bin. Forcing a read-only root FS makes it significantly harder for an attacker to maintain persistence or install tools. If applications need to write logs or temp files should use `emptyDir` volumes instead of writing to the container layer.

### allowPrivilegeEscalation の無効化

The ‍`allowPrivilegeEscalation: false` setting enforces the `no_new_privs` kernel flag, which prevents a process from gaining more privileges than its parent. In a standard Linux environment, an attacker can exploit files with the SUID bit to temporarily elevate their effective UID to root, even if the container started as a non privileged user. By disabling this escalation, you ensure that the kernel blocks any attempts to bypass user restrictions, effectively neutralizing a primary path for local privilege escalation and reducing the risk of a full container breakout

Not mounting service account tokens

Unless your application needs to communicate with the Kubernetes API, you should configure the pod not to mount a service account token.

```yaml
    spec:
      automountServiceAccountToken: false
```

### Seccomp フィルタの有効化

Re-enabling the container runtime's seccomp filter will improve the overall security of the workload and reduce the risk of container breakout.

```yaml
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```


## 参考情報

 - [Kubernetes Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
 - [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
 - [CIS Benchmark for Kubernetes] - Sections 5.1 and 5.2
