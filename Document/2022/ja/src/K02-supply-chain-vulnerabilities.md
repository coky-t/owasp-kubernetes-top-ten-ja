## 概要

コンテナは開発ライフサイクルのサプライチェーンのさまざまな段階において多くの形態をとり、それぞれが個別のセキュリティ課題を提示しています。単一のコンテナだけでも何百ものサードパーティコンポーネントや依存関係を持つことがあり、各段階での起源の信頼性を確保することは非常に困難です。これらの課題にはイメージの完全性、イメージの構成、既知のソフトウェア脆弱性などがありますが、これらに限定されるものではありません。

![Supply Chain Vulnerabilities - Illustration](/assets/images/K02-2022.gif)

## 説明

**イメージの完全性:** [Solarwinds 事件](https://www.businessinsider.com/solarwinds-hack-explained-government-agencies-cyber-security-2020-12) やさまざまな [汚染されたサードパーティパッケージ](https://therecord.media/malware-found-in-npm-package-with-millions-of-weekly-downloads/) などの出来事によりソフトウェアの来歴がメディアで最近大きな関心を集めています。これらのサプライチェーンのリスクは Kubernetes 内部での実行時だけでなく、コンテナビルドサイクルのさまざまな場面で表面化する可能性があります。コンテナイメージの内容に関する記録システムが存在しない場合、予期しないコンテナがクラスタで実行される可能性があります。

**イメージの構成:** コンテナイメージはレイヤーで構成されており、それぞれのレイヤーがセキュリティに影響を及ぼす可能性があります。適切に構築されたコンテナイメージは攻撃対象領域を減らすだけでなく、デプロイメント効率を向上させることができます。余計なソフトウェアが含まれたイメージは権限昇格や既知の脆弱性の悪用に利用される可能性があります。

**既知のソフトウェア脆弱性**: サードパーティパッケージを多用しているため、多くのコンテナイメージは信頼できる環境に取り込んで実行するには本質的に危険です。例えば、イメージ内の特定のレイヤーに既知のエクスプロイトの影響を受けやすいバージョンの OpenSSL が含まれている場合、それが複数のワークロードに伝播し、知らぬ間にクラスタ全体が危機にさらされる可能性があります。

## 防止方法

**イメージの完全性:** コンテナイメージは製作者から使用者に渡される一連のソフトウェア成果物およびメタデータと考えることができます。このハンドオフは開発者の IDE から直接 Kubernetes クラスタに渡すような単純なものから、多段階の専用 CI/CD ワークフローのような複雑なものまであります。ソフトウェアの完全性は各フェーズを通じて [in-toto](https://in-toto.io/) [attestations](https://github.com/in-toto/attestation) を使用して検証する必要があります。これによりビルドパイプラインの [SLSA](https://slsa.dev) レベルも向上します。SLSA レベルが高いほど、より耐性のあるビルドパイプラインであることを示します。

**ソフトウェア部品表 (Software Bill of Materials, SBOM)**: SBOM は特定のソフトウェア成果物に含まれるソフトウェアパッケージ、ライセンス、ライブラリのリストを提供し、他のセキュリティチェックの出発点として使用されるべきものです。SBOM 生成のもっとも一般的なオープンスタンダードは [CycloneDX](https://cyclonedx.org/) と [SPDX](https://spdx.dev/) の二つです。

**イメージの署名**: DevOps ワークフロー全体の各ステップは攻撃や予期せぬ事態をまねく可能性があります。製作者と使用者はサプライチェーンの各ステップで暗号鍵ペアを使用して成果物に署名し検証することで、成果物自体の改竄を検出します。オープンソースの [Cosign](https://github.com/sigstore/cosign) プロジェクトはコンテナイメージの検証を目的としたオープンソースプロジェクトです。

**イメージの構成:** コンテナイメージは最小限の OS パッケージと依存関係を使用して作成し、ワークロードが侵害された場合の攻撃対象領域を減らす必要があります。 [Distroless](https://github.com/GoogleContainerTools/distroless) や [Scratch](https://hub.docker.com/_/scratch) などの代替ベースイメージを利用すると、セキュリティ態勢を改善するだけでなく、脆弱性スキャナが生成するノイズを大幅に削減できます。distroless イメージを使用すると、イメージサイズも縮小し、最終的に CI/CD ビルドの高速化につながります。また、ペースイメージに最新のセキュリティパッチを適用しておくことも重要です。 [Docker Slim](https://github.com/docker-slim/docker-slim) のようなツールはパフォーマンスとセキュリティ上の理由からイメージのフットプリントを最適化するために利用できます。

**既知のソフトウェア脆弱性:** イメージの脆弱性スキャンはコンテナイメージの既知の脆弱性を列挙することを目的としており、防御の第一線として使用すべきものです。特定のレイヤでビルドされたイメージを探すだけで、脆弱性のあるすべてのアップストリームソフトウェアを特定することができます。イメージは迅速にパッチを適用すべきであり、脆弱性を含むレイヤを置き換え、最新の修正済みパッケージを使用してリビルドする必要があります。 [Clair](https://github.com/coreos/clair) や [trivy](https://github.com/aquasecurity/trivy) などのオープンソースツールは CVE などの既知の脆弱性についてコンテナイメージを静的に解析するため、可能な限り開発サイクルの早い段階で使用すべきです。

**ポリシーの適用:** Kubernetes の [admission controls](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) や [Open Policy Agent](https://www.openpolicyagent.org/) や [Kyverno](https://kyverno.io) などのポリシーエンジンで未承認のイメージを使用できないようにし、以下のようなワークロードイメージを拒否します。

- 脆弱性についてスキャンされていない
- 明示的に許可されていないベースイメージを使用している
- 承認済み SBOM を含んでいない
- 信頼できないレジストリを元としている

![Supply Chain Vulnerabilities - Mitigations](/assets/images/K02-2022-mitigation.gif)

## 攻撃シナリオの例

例 #1: 侵害された CI/CD パイプライン

ほとんどのチームは何らかの形で自動化を使用し、コンテナイメージをビルドして中央レジストリにプッシュします。その後、オブジェクトマニフェストで定義されたとおりに、Kubernetes からプルされます。このビルドシステムが侵害され、ビルドの一部として悪意のあるパッケージが注入された場合、Kubernetes はクラスタにイメージをプルしてそれを実行することになります。マルウェアが実行されたり、暗号通貨マイナーがインストールされたり、バックドアが仕込まれる可能性があります。

## 参考資料
Admission Controllers: [https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)

Co-Sign: [https://github.com/sigstore/cosign](https://github.com/sigstore/cosign)

CycloneDX: [https://owasp.org/www-project-cyclonedx/](https://owasp.org/www-project-cyclonedx/)

Docker Slim: [https://github.com/docker-slim/docker-slim](https://github.com/docker-slim/docker-slim)

Open Policy Agent: [https://www.openpolicyagent.org/](https://www.openpolicyagent.org/)

in-toto: [https://in-toto.io](https://in-toto.io)

SLSA: [https://slsa.dev](https://slsa.dev)
