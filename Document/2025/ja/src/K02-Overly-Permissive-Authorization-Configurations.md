---

layout: col-sidebar
title: "K02: 過度に許可を与える認可設定 (Overly Permissive Authorization Configurations)"
---

## 概要

Kubernetes は、ユーザーやサービスがクラスタオブジェクトを操作することを認可するための、一連の豊富で柔軟なオプションを提供します。その柔軟性により、Kubernetes リソースへの不正アクセスを許す、過剰なアクセスのリスクを生じます。

## 説明

Kubernetes の認可設定を検討する際には考慮すべき点がいくつかあります。

### よくある RBAC の誤設定

RBAC 設定ミスが一般的に発生する箇所はいくつかあります。詳細なガイダンスについては [Kubernetes Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) を参照してください。

- **cluster-admin の使用。** Kubernetes の cluster-admin ClusterRole は、シークレットなどの機密データを含む、すべてのクラスタリソースへの無制限のアクセスを提供します。ユーザーにこのロールの使用を許可すると、クラスタ内のリソースの誤った削除や改変のリスクも生じます。
- **ビルトインの Kubernetes ClusterRole の使用。** Kubernetes は `view` や `edit` などのいくつかのビルトインの ClusterRole を提供しています。それらはビルトインであるため、これらのオプションを使用する誘惑にかられますが、ユーザーのニーズに合わない非常に広範なパーミッションを提供する恐れがあります。
- **シークレットへの LIST および WATCH 権限。** ユーザーにシークレットオブジェクトの GET を許可すると、それらが漏洩する可能性があることは明らかですが、シークレットへの LIST や WATCH の権限を許可しても、ユーザーがそれらのシークレットの内容を閲覧できることに注意することも重要です。
- **Kubelet API アクセスを提供する Node/Proxy 権限。** Kubernetes で Node オブジェクトの Proxy サブリソースへの権限を持つユーザーは、監査ログ記録や Admission Control 要件をバイパスして Kubelet API に直接接続できます。そのため、そのアクセスを付与する際には注意が必要です。
- **ESCALATE, BIND, IMPERSONATE パーミッションの使用。** escalate, bind, impersonate といった動詞を使用すると、ユーザーは権限昇格の保護をバイパスできる可能性があります。escalate 動詞は、ユーザーが既に持っている権利よりも多くを持つ ClusterRole を作成できます。bind パーミッションは、ユーザーがまだバインドされていない ClusterRole をバインドできます。impersonate ユーザーは、ユーザーがクラスタ内の他のユーザーになりすますことができます。
- **Namespace 上での PATCH。** Namespace への PATCH パーミッションを持つユーザーは Namespace ラベルを更新できます。Pod Security Admission が使用されているクラスタでは、これはユーザーが管理者が意図したものより寛容なポリシーの Namespace を設定できる可能性があります。NetworkPolicy が使用されているクラスタでは、管理者が許可することを意図していないサービスへのアクセスを間接的に許可するラベルを、ユーザーが設定できる可能性があります。
- **サードパーティのシャドウ RBAC。** サードパーティの Helm チャート、Operator、外部マニフェストのデプロイメントを通じて「シャドウ RBAC」をもたらすリスクは重大ですがしばしば見落とされがちです。導入直後の機能性を確保するために、コミュニティが管理する多くのパッケージはデフォルトで過剰な権限設定になっています。ClusterRole/ClusterRoleBinding の暗黙的なプロビジョニングや、API 動詞やリソースに対するワイルドカード `*` の過剰使用といったものです。これは被害範囲の拡大をもたらし、侵害されたアプリケーションがクラスタ全体にわたる広範な発見や管理権限を継承してしまいます。さらに、多くのチャートは ServiceAccount トークンの自動マウンティングを無効化できておらず、ラテラルムーブメントの即時的な経路を提供します。シェルアクセスを獲得した攻撃者は、たとえアプリケーション自体が機能要件を持たないとしても、これらの認証済みクレデンシャルを活用して API サーバーにクエリできます。
- **ClusterRole Aggregation に注意。** サードパーティ Operator は、独自の ClusterRole に特定のラベルを追加することで、ビルトインろーる (admin など) を密かに拡張し、実質的に管理者権限を付与できます。


### 複数のオーソライザ

Kubernetes は、複数の異なる認可モードがすべてのクラスタに存在できます。ほとんどのクラスタでは RBAC が認可に使用されますが、マネージド Kubernetes クラスタでは Webhook 認可もよく使用されます。

ユーザーやサービスがクラスタに対して持つ権限は、設定された認可メカニズムによって提供される権限を統合したものになります。そのため、クラスタの認可態勢をレビューする際には、設定されているすべてのオプションをチェックすることが重要です。

もう一つの重要な点は、`kubectl auth can-i --list` のようなビルトインの Kubernetes 機能は、非 RBAC 認可システムで提供された権限を考慮しないことです。そのため、Kubernetes パーミッションの効果的な監査にはクラスタ内の各オーソライザごとに行う必要があります。

## 防止方法

The main principle that should be followed when designing authorization in Kubernetes clusters is to use a "least privileges" approach. Excessive general privileges should be avoided both for interactive users and services accessing the Kubernetes API.

For administrators, consider carefully using the impersonation system in Kubernetes to provide a lower privileged main account which can escalate its privileges when needed for specific operations as this will help avoid accidental damage to Kubernetes objects. Note that the impersonation system should be used with care only allowing escalation to specific resource names and not to all users.

In as much as possible, rights should be provided through a single authorization method and not via a mix of different authorizers as it is difficult to audit Kubernetes permissions across multiple authorization systems.

While the principle of least privilege must be the baseline for all kubernetes authorization, operational cases such as emergency debugging of a production incident or performing a sensitive one off change necessitate temporary elevated access. To manage these risks , organizations(the cluster admin) should move beyond the static assignment toward a process driven architecture. Granting privileged access should never be an adhoc manual task. It must be a process oriented workflow where every change is documented. In addition to Kubernetes audit logs, the access grant and related changes should be recorded and traceable outside the cluster as well, typically through GitOps (e.g.,. pull requests and an immutable change history).

For a more resilient security posture and to reduce clusters attack surface, organizations should implement Just In Time (JIT) Access, with tools that create and expire ClusterRoleBindings and RoleBindings on demand. Simple implementations can use short-lived kubeconfig files with OIDC token expiry. This approach effectively eliminates standing privileges by providing ephemeral on demand access. JIT access replaces broad permanent privileges with dynamic time bound authorizations that automatically expire after a specific time. By leveraging granular approval workflows such as multi party verification or context aware auto approvals, JIT ensures that elevated rights are precisely aligned with actual operational needs. This dynamic approach provides a clear, comprehensive audit trail, making it easier to document exactly who had access to what, when, and why.

## 参考情報

- [Role Based Access Control Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [CIS Benchmark for Kubernetes](https://www.cisecurity.org/benchmark/kubernetes) - Section 5.1
