## 概要
Kubernetes の認証は多くの形態を取り、非常に柔軟です。このように高度に設定可能であることを重視しているため、Kubernetes は多くのさまざまな環境で動作しますが、クラスタやクラウドセキュリティ体勢に関して課題もあります。

![Broken Authentication - Illustration](/assets/images/K06-2022.gif)

## 説明
Kubernetes API にアクセスするために必要なエンティティがいくつかあります。認証はこれらのリクエストの最初のハードルです。Kubernetes API への認証は HTTP リクエストを介して行われ、認証方式はクラスタごとに異なります。リクエストが認証できない場合、HTTP ステータス 401 で拒否されます。

![Kubernetes Authentication](/assets/images/kubernetes-auth.png)

入手元: [https://kubernetes.io/docs/concepts/security/controlling-access/](https://kubernetes.io/docs/concepts/security/controlling-access/)

Kubernetes API への認証が必要なさまざまなタイプの subject について詳しく見ていきましょう。

**人間の認証** 

人はさまざまな理由で Kubernetes と対話する必要があります。ステージングクラスタで実行中のアプリケーションをデバッグする開発者や、新しいインフラを構築しテストするプラットフォームエンジニアなど。OpenID Connect (OIDC)、証明書、クラウド IAM、さらには ServiceAccount トークンなど、人間としてクラスタに認証するために利用できる方法がいくつかあります。これらの中にはより堅牢なセキュリティを提供するものもあります。以下の防止方法のセクションで説明します。

**サービスアカウントの認証** 

サービスアカウント (Service account, SA) トークンは RBAC で適切に設定された場合、認証メカニズムとして Kubernetes API に提示できます。SA はシンプルな認証メカニズムで、一般的にはクラスタの *内部* からのコンテナから API への認証のために予約されています。

## 防止方法
***エンドユーザー認証に証明書を使用することは避ける:*** 証明書は Kubernetes API への認証に使用すると便利ですが、細心の注意を払って使用すべきです。現時点では、API には証明書を失効させる方法がないため、秘密鍵マテリアルの侵害や漏洩があった場合、クラスタの鍵を更新するために奔走することになります。また証明書は設定、署名、配布するのも面倒です。証明書は "Break Glass" 認証メカニズムとして使用できますが、プライマリ認証には使用してはいけません。

***独自の認証を作ってはいけない:*** 暗号と同様に、必要でないときに新しいものを作るべきではありません。サポートされ広く採用されているものを使用します。

**可能な場合は MFA を強制する:** 選ばれた認証メカニズムに関係なく、人間に二番目の認証方式 (通常は OIDC の一部) を提供するように強制します。

***クラスタ外からサービスアカウントトークンを使用しない:*** SA はグループにバインドできず、有効期限もありません。クラスタの外部から長期間有効な SA を使用すると、クラスタが重大なリスクにさらされます。

***ユーザーと外部サービスには短期間有効なトークンを使用して認証する:*** すべての認証トークンは許容範囲内で短期間有効であるべきです。このようにすれば、認証情報が漏洩した場合でも、アカウントを侵害するために必要な時間内にリプレイできない可能性があります。

![Broken Authentication - Mitigations](/assets/images/K06-2022-mitigation.gif)

## 攻撃シナリオの例

***偶発的な Git 流出:*** ある開発者が自分のラップトップから誤って `.kubeconfig` をチェックしてしまいました。このファイルには稼働中のクラスタの Kubernetes 認証情報が入っていました。ある人が GitHub をスキャンしてその認証情報を見つけ、ターゲット API (残念ながらインターネット上にあります) にリプレイします。クラスタは証明書を使用して認証するように設定されているため、流出したファイルにはターゲットクラスタへの認証に成功するために必要なすべての情報が含まれていました。

## 参考資料

Tremlo Blog Post: [https://www.tremolosecurity.com/post/what-the-nsa-and-cisa-left-out-of-their-kubernetes-hardening-guide](https://www.tremolosecurity.com/post/what-the-nsa-and-cisa-left-out-of-their-kubernetes-hardening-guide)

Kubernetes Authentication: [https://kubernetes.io/docs/concepts/security/controlling-access/](https://kubernetes.io/docs/concepts/security/controlling-access/)
