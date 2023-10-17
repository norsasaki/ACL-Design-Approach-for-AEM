# ACL-Design-Approach-for-AEM

## AEMのACL設計のよくある失敗

1. 新しい要望が出るたびにACL・グループ設計が複雑化し、工数や不具合の増加につながる。
2. 複数グループを組み合わせたときのテスト工数が膨大に増加する。またはテストし切れず運用後に不具合が発覚する。
3. 設計書の更新まで手が回らず、設計と実装にGAPが生まれる。誰も正しいACL設計を把握出来なくなる。
4. 完全性・冪等性を維持したACLのリリース方法が確立されていない。それゆえ環境毎にACL設定が微妙に異なる。
5. AEMの各標準機能が正常に動作するために、どのようなACLが必要か把握出来ていない。そのため運用後に標準機能が動作しないことが発覚する。
6. テストを手動で実施している。そのため網羅性・テスト品質ともに高くならない。

## ACL設計が難しい理由

* そもそもACL設計は難しく知識と経験が必要
* AEMは不完全な Role-based access control(RBAC)しか提供できていない
  * Built-in Groups が少なすぎ。貧弱すぎ。
  * ビジネスユーザ向けの新しいRoleを定義するために、DatabaseレイヤーのRead/Write/Create/Delete権限から設計しないといけないのはおかしすぎるだろ
  * AEMは柔軟なコンテンツ管理が行える点が強み。一方、認可設計としてはコンテンツ設計をかっちり決めたい。相反するためAEMが認可管理が苦手なのは致し方ない。。。
    * Attribute Based Access Control(ABAC)なら両立可能？？
* 「どの機能を利用できるか」と「どのコンテンツにアクセスできるか」をACLという同じ仕組みで制御する仕様のため、ACL設計をする上で認可モデルの深い理解が求められる
* ACLをコードで管理できない。

## Best Practice for AEM ACL Design

1. [NetcentricのAC Tool](https://github.com/Netcentric/accesscontroltool)を使用する。
2. [Best Practices](https://github.com/Netcentric/accesscontroltool/blob/develop/docs/BestPractices.md)に従う。
3. フォルダ／ノード設計時にACL設計も同時に行い、シンプルなACL設計になるようにフォルダ／ノード構造を設計する。
4. 複雑なACL設計を避けられるように要件定義・設計をする。複雑なACLが求められる要件はそのデメリットをしっかり伝え、必須か否かクライアントによく検討してもらう

## What's AC Tool

* The AC tool provide the simple way to manage the specification and deployment of complex Access Control Lists in AEM.

## Features of AC Tool

* easy-to-read Yaml configuration file format
* run mode support
* automatic installation with install hook
* cleans obsolete ACL entries when configuration is changed
* ACLs can be exported
* management of user's key stores and the global trust store
* stores history of changes
* ensured order of ACLs
* built-in expression language to reduce rule duplication
See [the slide of adaptTo() 2016](https://adapt.to/2016/presentations/adaptto2016-ac-tool-jochen-koschorke-roland-gruber.pdf) for details



## How can AC tool resolve the problems?

### ACL・グループ設計の複雑化を避けるには？

[Best Practices](https://github.com/Netcentric/accesscontroltool/blob/develop/docs/BestPractices.md)に従うことが最も大切。そうすることで、柔軟で見通しの良い設計が行える。
その中でも特に重要なTipsをリストアップする。

* Use fragment groups for functional aspects and content access
* Always use Allow statements. Avoid using a Deny statement
* Consider access rights when designing you content structure

### 複数グループを組み合わせた時の不具合を防ぐには？

グループAに`deny`ルール、グループBに`allow`ルールが設定がされており、グループA、Bの両方にユーザが所属した場合にACLが競合し期待する動作にならないことが問題となる。
※一番最後に設定されたACLが勝つ
By default ACEs with denies are sorted up to the top of the list, this follows the best practice to order denies always before allows

競合を避けるためには
* `deny`ルールはトップ階層のノードにのみ定義する
* 下層階層には`allow`ルールのみを定義し、`deny`ルール使わない


#### 実装例

下記設計の実装を考えてみる

![](./img/acl-design-sample.drawio.svg)


* 普通にACL設定すると子ノードに継承されてしまう。
* そのため`allow`のみで実装するには子ノードに継承されないACL設定方法が必要
* 子ノードに継承させずに親ノードにのみACL設定をするためは以下のイディオムを使う

```
# The `deny` rules should be defined in fragment-restrict-for-everyone
- path: /content
  permission: deny
  actions: 
  privileges: jcr:all
  repGlob: 

- path: /content
  permission: allow
  actions: read,modify,create,delete
  privileges: 
  repGlob: ""  # matches node /foo only (no descendants, not even properties)

- path: /content
  permission: allow
  actions: read,modify,create,delete
  privileges: 
  repGlob: /jcr:*
```

* このイディオムを使って緑色ページ用のルールを定義した場合のサンプルConfig
* 他のConfigはこちらを参照
  * ユーザがページ追加できず保守性が低いので、実際のプロジェクトではもっとACLが実装しやすいノード階層に設計するべき。

```
       - FOR path IN [/content/we-retail, /content/we-retail/A1]:
           - path: ${path}
             permission: allow
             actions: 
             privileges: jcr:read
             repGlob: ""
 
           - path: ${path}
             permission: allow
             actions: 
             privileges: jcr:read
             repGlob: /jcr:*
```

### 設計書の陳腐化を防ぐアイデア

AC Toolを使えばACLをYAMLとして定義できる
そしてYAMLにはコメントアウトを記述できるので、ACL設計に対してコメントを残すことで、YAMLファイルでACL設計書を代用することができる。

![Alt text](./img/comment-in-yaml.png)


### 完全性・冪等性を維持したACLのリリース方法

* AC Toolを使うと、ACLをコードとして管理し、Javaのコードと同じようにGit管理および資源移送が可能になる。
* また完全性・冪等性も備えるので、安心してACLをリリースすることができる。

以前はACS CommonsのACL Packagerというツールが主流であったが、完全性・冪等性を備えておらず、ACL PackagerでACLを移送すると知らず知らずのうちに、環境毎に差が出ることが多かった。

### 標準機能に必要なACLを適切に把握するアイデア

* AEMはいくつかのBuilt-in Groupを提供しており、ビジネス要件に応じて、これらのグループを拡張・または参考にすると良い。
* AC Toolを用いると、AEM内の全グループのACLをYAML形式で出力ができるので、Built-in Groupに定義されているACLを参照することで、ACL設計の抜け漏れを減らすことができる。
  * 例えば、ページ公開するにはページ自体の`Replicate`権限に加えて、テンプレート、ポリシーの`Replicate`権限も必要

### ACLの効率的なテスト方法

* テストコードを書き網羅的なACLテストを行う
  * CRUD操作は`curl`などで行える
  * マニュアルテストは補助として実施する。
* use [access-control-validator](https://github.com/Netcentric/access-control-validator)













## the Best Practice is great, but...

でも実際にBest Practiceにしたがって設計するとグループが多くなり複雑になりやすい。
そのためもっとシンプルな設計案を紹介したい

## I suggest that the more simplified design

![](./img/simplified-approach.drawio.svg)

### More simplified design #1

Best Practice では、各機能に対応したFunctional Fragmentグループを作成し、ロールグループに必要な機能のみをアサインべきだと言っている。
しかし、これをプロジェクト毎に設計・実装するのは大変だし、本来はAEM製品側がFunctional Fragmentグループを提供すべきだが、現実そうはなっていない。

幸いにも、多くのAEMプロジェクトで求められるロールは下記の３種類かその変化系だけだ。
* 編集者
* 公開者
* 承認者
  
そしてこれらのロールは content-authors, dam-users, workflow-usersなどのbuit-inグループを組み合わせることで十分に表現できる。
そのためFunctional Fragmentは作らず、built-inグループをUser Role fragmentグループの代用とすれば設計にかかるコストと時間を省くことができる。

### Useful Built-in Groups

再利用性の高いBuilt-in Groupを下記表にまとめる。

| Built-in Groups         | Description                                                                                                                                                                        |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| contributor             | Basic privileges that allow the user to write content (as in, functionality only).                                                                                                 |
| content-authors         | Group responsible for content editing. Requires read, modify, create, and delete permissions.                                                                                      |
| dam-users               | Out-of-the-box reference group for a typical AEM Assets user.                                                                                                                      |
| projects-administrators |                                                                                                                                                                                    |
| projects-users          |                                                                                                                                                                                    |
| user-administrators     | Authorizes user administration, that is, the right to create users and groups.                                                                                                     |
| workflow-administrators |                                                                                                                                                                                    |
| workflow-editors        | Group that is allowed to create and modify workflow models.                                                                                                                        |
| workflow-users          | A user participating in a workflow must be a member of group workflow-users. Gives the user full access to: /etc/workflow/instances so that they can update the workflow instance. |

Ref: [User Administration and Security](https://experienceleague.adobe.com/docs/experience-manager-65/administering/security/security.html?lang=en)


しかし、built-inグループはアクセス許可を与えたくないノードへのアクセス権を既に持っている場合がある。
例えばcontent-authorsは/content配下の読み取り・書き込み権限を持っているが、/content配下の特定のサイト以外のアクセス権は付与したくない、という要件は非常にポピュラーだ。
その場合は`fragment-restrict-for-everyone`に`/content`を`deny`するルールを定義することでBuilt-in Groupの持つACLをリセットすることができる。

下図に示すとおり、ACツールはACLの常に一番下に新しいルールを追加する。そしてACLが競合した場合、リストの下に位置するルールが優先される。
この特徴を利用することで、CSSリセットのようにBuilt-in Groupに定義されているACLをリセットすることができる。
具体的には下図の場合、`content-authors`と`fragment-restrict-for-everyone`ともに`/content`に対してのACLが定義されている。
`content-authors`と`fragment-restrict-for-everyone`の両方に所属するとACLが競合するが、`fragment-restrict-for-everyone`のルールがよりACLの下の位置に定義されているので優先される。

![](./img/priority-of-acl.png)


Built-in Groupsの持つACLをリセットできるのはわかった。では、その状態から任意の`allow`ルールを与えるにはどうすれば良いか。
これはとても簡単で、必要な`allow`ルールをAC Toolsで定義するだけで良い。

ACツールはYAMLファイルの中で定義されている`deny`ルールをまず登録し、その後に`allow`ルールを登録する。
先述の通り、リストの下に位置するルールが優先されるので、`allow`ルールが優先される。

### Built-in Groupsを必ず使用しないといけないケース

More simplified design #1にはもう一つメリットがある。
AEMはACL以外に"特定のグループに所属しているかどうか"を条件に認可制御する場合がある。
例えば、下記のUIは `workflow-users` に所属する場合のみ表示される。
そのため Language Copy を正しく使えるようにするためには `workflow-users` に所属する必要がある。

![](./img/sample-ui-managed-by-groups.png)


こうした点を踏まえてえも、fragment groupを設計するよりも、Built-in Group を活用する方法を確立する方が有用だと考える。

```
# /libs/cq/translation/cloudservices/rendercondition/isWorkflowUser/isWorkflowUser.jsp
 if (checkUserGroup(resolver, userSession, workflow_users)) {
     return true;
 } 
```

Ref: [Render Condition](https://developer.adobe.com/experience-manager/reference-materials/6-5/granite-ui/api/jcr_root/libs/granite/ui/docs/server/rendercondition.html#)

## More simplified design #2

Best Practice では、Read/write access to contents はcontent groupによって提供されるべきだと言っている。
基本的に同意するが、下記の点でより実用的なアプローチがあると考える。

* コンテンツに対してどのアクセス権を与えるかはビジネス要件で決まる。しかし、ロールとコンテンツに応じて求められる権限は細かく変わるため、Read/Writeという観点でfragmentグループを作成すると、グループ数が非常に多くなることが懸念される。
* コンテンツを公開することができるかどうかは直感的には機能観点だが、AEM内部ではACLの１つとして制御される。そのため、公開権限自体はfragment groupではなく、content groupで管理する方が見通しが良い

そこでロールに１対１対応するようにcontent groupを作り、そのグループでそのロールに必要なコンテンツに対するACLを全て管理する方が見通しが良い。
また、公開権限についてもcontent group で管理する方が良い。

## ケーススタディ | 要件

下記要件のACL・グループ設計を考えてみる。

* マルチサイト(We-Retail)
* 本社がlanguage-masterのコンテンツを管理し、Live CopyおよびLanguage Copyを使って各国・言語へ展開する
* 支社は自分が管轄するリージョンサイトのコンテンツを管理する。
* ページ公開には必ず承認者の商品が必要
* ページの編集者はページの作成・更新が可能
* ページの公開者はページの作成・更新・削除および公開が可能

## ケーススタディ | Global Fragments

Global Fragmentは以下の２グループを定義する。

* fragment-restrict-for-everyone
  + `Deny`ルールのみを定義する。
  + `Deny`ルールはこのグループ以外では原則定義してはならない。
  + 下記のユーザコンテンツ領域へのアクセス禁止するACLを定義
    - /content, /content/experience-fragments, /content/projects, /content/dam, 
    - /content/dam/projects, /content/dam/collections, /content/cq:tags, /conf
  + Built-in Groupに定義されたACLで上書きが必要なものもここで定義
* fragment-basic-allow
  + 全サイト、全グループ共通で必要な`Allow`ルールを定義
  + 主に、親ノードへのアクセスは許可するが、子ノードへのアクセス権限を与えず、親ノードのみのアクセス権を与える時のルールを記載
    - [White listing nodes](https://github.com/Netcentric/accesscontroltool/blob/develop/docs/BestPractices.md#white-listing-nodes)

## ケーススタディ | Content Fragments

Content Fragmentsは下記の方針で作成する。

* language-mastersおよび各リージョンサイトの管理者は異なるため、リージョン毎にContent グループを分ける
* さらに各リージョン毎に編集者、公開者、承認者というロールを分けて作成する。
  + content-${sitePrefix}-${country}-for-${role}
* 各ロールで付与するACLは下記
  + acl_read権限を与えないと
    | role      | priviledge                         |
    | --------- | ---------------------------------- |
    | editor    | read,modify,create                 |
    | publisher | read,modify,create,delete |
    | approver  | read,modify,create,delete |

* ケーススタディのサンプルコンフィグはこちらを参照