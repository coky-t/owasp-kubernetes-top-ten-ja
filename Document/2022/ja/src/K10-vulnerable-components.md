---

layout: col-sidebar
title: "K10: 古くて脆弱な Kubernetes コンポーネント (Vulnerable Components)"
---

## 概要
Kubernetes には脆弱性が存在します。CVE データベース、開示、更新を注意深く追跡し、パッチ管理の計画を立てるのは管理者の責任です。

![Vulnerable Components - Illustration](/assets/images/K10-2022.gif)

## 説明

Kubernetes クラスタは非常に複雑なソフトウェアエコシステムであるため、従来のパッチ管理や脆弱性管理では困難を伴うことがあります。

***ArgoCD CVE***: ArgoCD は一つまたは複数のクラスタに継続的にソフトウェアを配信するために使用される非常に人気のある宣言型 GitOps ツールです。ArgoCD は悪意のあるアクターが悪意のある Kubernetes Helm Chart (YAML) をロードできる [CVE-2022-24348](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-24348) を含むいくつかの CVE を長年にわたって抱えています。ArgoCD はクラスタ内部で動作し、これらのチャートを自動的にデプロイする役割を担っています。この Helm チャートはパーシングの脆弱性を悪用して、API キー、シークレットなどの制限された情報にアクセスします。そしてこのデータは攻撃者が Kubernetes クラスタ内でピボットしたり、機密データをさらに機密性の高いデータをダンプするために使用できます。

***Kubernetes CVE:*** 2021 年 10 月に、人気の Kubernetes ingress `ingress-nginx` に CVE がリリースされました (https://github.com/kubernetes/ingress-nginx/issues/7837) 。これは ingress オブジェクトを作成または更新するアビリティを持つユーザーがクラスタのすべてのシークレットを取得できるものです。これは "custom snippets" と呼ばれるサポート機能を利用しています。この問題は `ingress-nginx` のバージョンをアップグレードするだけでは対処できず、セキュリティチームが大規模に対処するのは困難な状況になっていました。

***Istio CVE:*** Istio の中核となる機能の一つにサービス間の認証や認可の提供があります。2020 年に認証バイパス脆弱性が発見されました ([https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-8595](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-8595)) 。Istio の Istio の Authentication Policy の exact path マッチングロジックを悪用したものです。これにより有効な JWT トークンなしでリソースへの不正なアクセスが可能になりました。攻撃者は保護されたパスの後に `?` や `#` の文字を追加することで JWT バリデーションをすべてバイパスできました。

*Kubernetes の最小バージョン*: 複数のクラスタが異なるクラウドやオンプレミス環境で稼働している場合、それらのクラスタの正確なインベントリを維持し、Kubernetes の最小バージョンへの準拠を確認することが重要です。Terraform などの OSS IaC プラットフォームを使用することで、複数のクラスタにまたがる Kubernetes API のバージョンを監査し、必要に応じてパッチを適用できます。

## 防止方法

Kubernetes クラスタ内で動作しているサードパーティソフトウェアは非常に多いため、脆弱なコンポーネントを排除するために多面的なアプローチをします。

**CVE データベースを追跡する:** 何よりもまず、Kubernetes と関連するコンポーネントを既存の CVE 脆弱性スキャンプロセスから除外することはできません。

**継続的にスキャンする:** OPA Gatekeeper などのツールを使用して、クラスタ内の脆弱なコンポーネントを検出するカスタムルールを作成できます。これらを定期的に実行し、セキュリティ運用チームが追跡すべきです。

**サードパーティの依存関係を最小限に抑える:** 過度に寛容な RBAC、低レベルのカーネルアクセス、過去の脆弱性開示記録について、デプロイメント前にすべてのサードパーティソフトウェアを個別に監査すべきです。

![Vulnerable Components - Mitigations](/assets/images/K10-2022-mitigation.gif)

## 攻撃シナリオの例

CVE を悪用する方法は無数にあります。攻撃者がクラスタ内の任意のコンポーネントと対話して CVE を悪用することができれば、その影響はクラスタ全体の侵害になり得ます。攻撃者がウェブアプリケーションのリモートコード実行脆弱性を利用してクラスタのシェルを取得したと想像してみてください。彼らは隣接するマイクロサービスが Istio ポリシーで保護されているために HTTP 呼び出しができないことに気づきました。攻撃者はリクエストの最後に `?` 文字を追加して JWT バリデーションを完全にバイパスし、次の保護された API にピボットできます。

## 参考資料

ArgoCD CVE Database: [https://www.cvedetails.com/vulnerability-list/vendor_id-11448/product_id-81059/version_id-634305/Linuxfoundation-Argo-cd--.html](https://www.cvedetails.com/vulnerability-list/vendor_id-11448/product_id-81059/version_id-634305/Linuxfoundation-Argo-cd--.html)

CVE Database Keyword “Kubernetes”: [https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=kubernetes](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=kubernetes)

Istio Security Bulletins: [https://istio.io/latest/news/security/](https://istio.io/latest/news/security/)

Kubernetes Security and Disclosure Information: [https://kubernetes.io/docs/reference/issues-security/security/](https://kubernetes.io/docs/reference/issues-security/security/)
