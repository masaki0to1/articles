# CloudFormationテンプレート開発のTips: ParameterとSSMParameterStore編

## CFnテンプレートのParemtersセクションについて
---
## **Q. SSM Parameter Store を使うべきか：**
## **結論：**
- どうしてもCFn内でのみ完結したい場合を除き、SSM Parameter Store を積極的に利用するべき。
- その際、Parameters セクションで SSM Parameter Store のパラメータのキーを指定し、!Sub '{{resolve:ssm:${ParamKey}}}' で動的参照するのが最も柔軟性があり変更に強い参照の方法になるので、これを推奨する。

### Tips：
- 基本的に、各種プロパティの可変部分は、不変であることの見極めは困難なので、動的パラメータ化して分離すべき
- そして、パラメータ管理は SSM Parameter Store で一括ですべきであって、各スタックのパラメータタブで１つずつ確認したり、それらをドキュメントでまとめたりすべきでない。（ドキュメントの運用・修正・確認のコストがかかるため）
- パラメータの渡し方は、Paramaters セクションから変数として渡す方法と、SSM Parameter Store から動的参照で渡す方法と、両者を組み合わせる方法が存在する。
    - -> 両者を組み合わせた、パラメータキーを Parameters セクションから '{{resolve:ssm}}' へ渡す方法が最も変更に強くおすすめ。（後述）

- 状態を持たせたり（検証環境、本番環境など）、数字を切り替えたり（時間数、バージョンなど）する場合には、Parameters セクション内で（例えばプルダウンで切り替えるなどして）完結すべきで、追加で SSM Parameter Store で保管する必要はない。
- 逆に、毎回必ず決まった１つのパラメータを渡すといった場合には、（コードとの分離や、変更に強くすることが重要である場合を除き）Parameters セクション内にそのパラメータを用意する必要は必ずしもない。その上で、 SSM Parameter Store で管理すれば、運用時もテンプレートファイルを確認する手間が消え、かつクロススタック参照も容易になるため、利用すべき。このように、Parameters セクション内でのパラメータの必要性と、SSM Parameter Store への保管の必要性は同一ではないことに留意する。
- 値それ自体に価値や重要な意味がある場合や、管理運用時に把握しておきたい場合には、SSM Parameter Store を利用すべき。
- SSM Parameter Store パラメータの渡し方比較まとめ
    - Parameters セクションでパラメータ自体を指定して変数として渡す：
        - 特徴：
            - キーの変更に強い（一括変更可）
            - 値の変更に弱い（バージョン指定のコストが発生する）
        - ユースケース：
            - 各リソース毎に、パラメータの一貫性がある=参照しているリソース全体を同タイミング、同じ値で更新したい場合
        - メリット：
            - Parameters セクションのみの変更でパラメータキーを一括変更できる
        - デメリット：
            - Parameters セクションの記述とそのパラメータ設定の操作が必要
            - バージョン指定が必須となり、自動で最新バージョンを指定してくれない

    - 直接 '{{resolve:ssm}}' を埋め込んで渡す：
        - 特徴：
            - 値の変更に強い（自動でバージョン最新化）
            - キーの変更に弱い（一括変更不可）
            - Parameters セクションの記述が不要
        - ユースケース：
            - 各リソース毎に、パラメータの一貫性がなく、それぞれ柔軟に変更したい場合
        - メリット：
            - Parameters セクションの記述とパラメータ設定操作が不要
            - バージョン未指定により、自動で最新バージョンを指定できる
        - デメリット:
            - パラメータキーを一括変更できない

    - Parameters セクションでパラメータキーを指定して、'{{resolve:ssm}}' へ渡す：
        - 特徴：
            - キーの変更に強い（一括変更可）
            - 値の変更に強い（自動でバージョン最新化）
        - ユースケース：
            - キーの変更を一括で行いたい、かつ、値更新時、バージョンのインクリメントをしたくない場合
        - メリット：
            - パラメータキーを Parameters セクションから一括で変更できる
            - バージョン未指定により、自動で最新バージョンを指定できる
        - デメリット：
            - Parameters セクションの記述とそのパラメータ設定の操作が必要
            - キーを環境によって使い分けるなどをする時には、Parameters セクションの対象パラメータ全ての Defalut の書き換えor再設定が必要
            
->このように、各パラメータに対してそれが、参照されるものか、管理運用されるべきなのか、それ自体に価値があるのかただの記号なのか、といったことを考慮して、CFn テンプレートを開発すべきである。
->その上で、適切なパラメータの渡し方を実装すべき。

## SSM Parameter Store を利用すべき３つの理由
---

### 理由１：運用・開発コストの削減が可能

パラメータ関連の話：
- 運用コスト：
    - CFnテンプレートファイルの特性上、変数の再利用を実現するには、Parameters セクション か SSM Parameter Store を利用するしかなく、前者ではスタックのデプロイの度にパラメータをセットする作業負荷が発生する。
    （最初にテンプレートファイル内でパラメータを SSM Parameter Store に渡す際は、Parameters セクションから渡す必要がある）
    - パラメータ確認の負担が減る（詳細は理由３）
    - コードとパラメータの分離が可能になり、運用が楽になる（詳細は理由３）

- 開発コスト：
    - 複数ファイルにまたがる場合、Parameters セクションの記述を何度もしないといけない。SSM Parameter Store で外だししておけば、動的参照で実現できるので、記述量を削減できる。

クロススタック参照の話：
- 運用コスト：
    - パラメータとしては扱わないが、クロススタック参照をして値を再利用するというケースにおいても、その実現方法には、Export によるものと SSM Parameter Store によるものの２つがあるが、前者ではスタック間に強い依存関係が発生し、動的に変更することができない。
    - 一方、SSM Parameter Store では、動的参照ができる。（※ 以前までは、SSM Parameter Store のパラメータを更新した場合は、CFn テンプレートファイル内のresolve:ssm/ParamName/Version の箇所の Version を毎回手動で上書きする必要があったが、アップデートで Version を指定しなければ、最新 Version を自動で参照するように変更されたので、実質デメリットがなくなった）
- 開発コスト：
    - クロススタック参照をするテンプレートファイル群を開発していると、!ImportValue 時に、参照先ファイルの Export の Name を確認する手間が往々にしてあるが、（もちろん、プレフィックスに ${AWS::StackName} を入れ、その後にシンプルな単語を繋げ、またその記法を統一するなどでいくらか楽になるが、）SSM Parameter Store を利用すれば、階層的に管理されたパラメータを逐次、コンソール画面１つで確認しながら開発ができる。他ファイルの特定行を目視するよりも断然楽になる。

### 理由２：データのセキュリティ向上が可能

機密情報管理の話：
- SSM Parameter Store でタイプを [SecureString] とすることでKMSキーで暗号化して保管することができる
    - SecureString は、CFn テンプレートからのデプロイに未対応であることに注意する（参照は可）
- SSM Parameter Store から Secret Manager シークレットを参照することもできる

### 理由３：パラメータとコードの分離管理が可能

パラメータ一元管理機能の話：
- SSM Parameter Store のコンソール画面で簡単にどのパラメータがどんな値となっているかを一括確認できる（CFn の Parameters セクションのみで管理すると、スタック毎にパラメータタブを見る労力が発生する）
- Tag 付けにより、どのシステムで利用されているパラメータかを簡単に管理できる

システムの独立性の話：
- システムのコードとパラメータが分離されるため、コードを修正することなく、SSM Parameter Store で完結してパラメータ値の変更ができ、システムの独立性を向上できる

## CFn Parameters セクションでのバグ?メモ
---
- Type: AWS::SSM::Parameter::Name を指定しても、バリデーションが効かない
- しかし、スタックが CREATE_IN_PROGRESS から全く進まなくなる
- 本来なら、バリデーションチェックにより、スタックをデプロイできないはずなのに、デプロイしようとして進んでしまって、CREATE_IN_PROGRESS から全く進まなくなるのだと思う
- 例：下記の場合に発生

        SsmKeyParameter:
            Description: Specify the SSM Parameter Key.
            Type: AWS::SSM::Parameter::Name
            Default: dir/parameterkeypath

        UseKeyResource:
            Properties:
                Key: '{{resolve:ssm:${SsmKeyParameter}}}'

- これを下記のようにすれば、ちゃんと動くはずだが、

        SsmKeyParameter:
            Description: Specify the SSM Parameter Key.
            Type: AWS::SSM::Parameter::Name
            Default: /dir/parameterkeypath

        UseKeyResource:
            Properties:
                Key: '{{resolve:ssm:${SsmKeyParameter}}}'

- ここで言いたいのは、本来バリデーションを効かせるのは、SSMパラメータキーの入力ミスに気づくためのはずだが、バリデーションが場合によっては機能せず、むしろスタックの進行が固まったままになり余計に面倒であるということである
- また、キーを指定しない為に空欄にしても、パラメータ値が存在しないというエラーが出るため、キーを指定しない運用もできない
- -> なので、Parameter の Type は String にしておくのが無難
- -> ただし、必ず正しいパラメータキーを指定させたい場合には、有効なので活用する（先頭の/を打ち損じた場合にスタックの進行が固まる現象以外は便利）

## CFn の Parameters セクションで動的にパラメータを管理しようとして困ったこと
--- 
- Mappings や Parameters セクションを更新して、変更セットを作成しようとしても「変更はありません」となり、失敗する
    - つまり、例えば、Organizations のメンバーアカウントが新しく追加される度に、SSOアカウント割り当て用のテンプレートのアカウントについてのパラメータを更新しようとしても変更セットによる運用はできないということになる


- Outputs の Export でクロススタック参照すると、参照先を削除してからでないと下記のエラーが出て、参照元の Export した値を変更することができず不便（正確には、出力する Arn や Id などが変わらなければ、エラーは出ない）
    
        Export test cannot be updated as it is in use by test
    
    - その為、特別な理由がなければ、SSM Parameter Store を使うのが良さそう

### スタックの粒度の考え方

例：IAM Identity Center の SSO Assignment を記述したテンプレートをデプロイ

アカウント割り当て作業１つをスタックの粒度とした場合：
- Parameters セクションで Type を AWS::SSM::Parameter::Value<> として割り当て内容を参照する

    - メリット：
        - スタックのライフサイクルと対象アカウントへの割り当てが一致するため、コンソールでの操作で割り当て削除については、完結できる
        -  コードは変更せず、パラメータの操作でのみ挙動を変更する運用ができるので、検証環境と本番環境とで冪等性を最大限に保ちたい場合に有利
        - 次に示す例含めて、これ例が最も冪等性が高い

    - デメリット：
        - 対象のスタックが、アカウント数や、ユーザー数/グループ数、許可セットの数に比例して増加するため、その管理運用コストも爆発的に増加する
        - アカウントが増える度に、デプロイ時のパラメータ選択といった運用コストが増加する

アカウント割り当て作業全てを１つのスタックに追記していく場合：

- ＜resolve:ssm:***ParamName***＞で割り当て内容を参照する
    
    - メリット：
        - スタックが１つにまとまるので、スタックの管理コストが下がる

    - デメリット：
        - コードを追記していくため、冪等性が保てない

