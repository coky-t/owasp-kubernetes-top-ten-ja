---

layout: col-sidebar
title: "K05: 不十分なログ記録と監視 (Inadequate Logging)"
---

## 概要

Kubernetes 環境は多くのさまざまなコンポーネントからさまざまなレベルのログを生成する機能を備えています。ログがキャプチャ、保存、アクティブに監視されていない場合、攻撃者はほとんど検出されないまま脆弱性を悪用する可能性があります。ログ記録や監視が行われないことで、インシデント調査やレスポンス作業にも支障が生じます。

![Inadequate Logging - Illustration](/assets/images/K05-2022.gif)

## 説明

Kubernetes のコンテキストにおける不適切なログ記録はいつでも発生します。

- 認証試行の失敗、機密リソースへのアクセス、Kubernetes リソースの手動削除や変更などの関連イベントがログ記録されていない。
- 実行中のワークロードのログやトレースでは疑わしいアクティビティを監視していない。
- アラートの閾値が設定されていないか、適切にエスカレーションされていない。
- ログが一元的に保存されておらず、改竄から保護されていない。
- ログ記録インフラストラクチャが完全に無効になっている。

## 防止方法

以下のログ記録ソースを有効にし、適切に設定する必要があります。

**Kubernetes 監査ログ: [監査ログ](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)** は API によって行われたアクションを後の解析のために記録する Kubernetes 機能です。監査ログは API サーバー自体で発生したイベントに関連する質問に答えるのに役立ちます。

異常な API コールや不要な API コール、特に認可の失敗についてログで監視していることを確認します (これらのログエントリにはステータスメッセージ "Forbidden" が表示されます) 。認可の失敗は攻撃者が盗んだ認証情報を悪用しようとしていることを意味している可能性があります。

AWS, Azure, GCP などのマネージド Kubernetes プロバイダはクラウドコンソールでこのデータへのアクセスをオプションで提供しており、認可失敗時のアラートを設定できることがあります。

**Kubernetes イベント:** Kubernetes イベントはリソースクォータの超過や保留中のポッドなど、Kubernetes リソースの状態変化やエラー、および任意の情報メッセージを示します。

**アプリケーションとコンテナのログ:** Kubernetes 内部で動作するアプリケーションはセキュリティの観点から有用なログを生成します。これらのログをキャプチャするもっとも簡単な方法は、出力が標準出力 `stdout` と標準エラー `stderr` ストリームに書き込まれるようにすることです。これらのログを永続化するにはさまざまな方法があります。オペレータはログをログファイルに書き込むようにアプリケーションを設定するのが一般的です。ログファイルはサイドカーコンテナで扱われ、一元的に集められ処理されます。

**オペレーティングシステムログ**: Kubernetes ノードを実行している OS によっては、追加のログを処理できることがあります。 `systemd` などのプログラムからのログは `journalctl -u` コマンドを使用して利用できます。

**クラウドプロバイダログ:** AWS EKS, Azure AKS, GCP GKE などのマネージド環境で Kubernetes を運用している場合、利用可能なログ記録ストリームがいくつか見つかります。一例として、[Amazon EKS](https://aws.amazon.com/eks/) には [Authenticator](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html) コンポーネント専用のログストリームが存在します。これらのログは EKS が AWS IAM 認証情報を使用した RBAC 認証に使用するコントロールプレーンコンポーネントを表しており、セキュリティ運用チームにとって豊富なデータソースとなります。

**ネットワークログ:** ネットワークログは Kubernetes 内の複数のレイヤでキャプチャできます。従来のプロキシや nginx や apache などの ingress コンポーネントを使用している場合、標準出力 `stdout` と標準エラー `stderr` パターンを使用して、これらのログをキャプチャして送信し、さらに調査する必要があります。 [eBPF](https://ebpf.io/) などの他のプロジェクトはクラスタ内のセキュリティ観測可能性をさらに強化するため、使用可能なネットワークおよびカーネルログを提供することを目的としています。

上記のように、Kubernetes エコシステム内で利用可能なログ記録メカニズムには事欠きません。堅牢なセキュリティログ記録アーキテクチャは関連するセキュリティイベントをキャプチャするだけでなく、クエリ可能で、長期的で、完全性を維持する方法で一元化される必要があります。

![Inadequate Logging - Mitigations](/assets/images/K05-2022-mitigation.gif)

## 攻撃シナリオの例

Scenario #1: Rouge Insider (異常な数の "削除" イベント)

Scenario #2: サービスアカウントトークンの侵害

## 参考資料

[https://developer.squareup.com/blog/threat-hunting-with-kubernetes-audit-logs/](https://developer.squareup.com/blog/threat-hunting-with-kubernetes-audit-logs/)

[https://kubernetes.io/docs/concepts/cluster-administration/logging/](https://kubernetes.io/docs/concepts/cluster-administration/logging/)

[https://www.cncf.io/blog/2021/12/21/extracting-value-from-the-kubernetes-events-feed/](https://www.cncf.io/blog/2021/12/21/extracting-value-from-the-kubernetes-events-feed/)
