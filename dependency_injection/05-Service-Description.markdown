サービスを記述するためにXMLもしくはYAMLを使う
===========================================

前の章では、`sfServiceContainerBuilder`クラスを使用して
PHPコードでサービスを記述する方法を学びました。
今日は、サービスローダーとダンパーの助けを借りて、
サービスを記述するためにXMLもしくはYAMLを使う方法を学びます。

Symfony Dependency Injectionコンポーネントは**"ローダーオブジェクト"**を使用して
サービスをロードするヘルパークラスを提供します。デフォルトでは、コンポーネントには
2つのローダーが付属しています: XMLファイルをロードする`sfServiceContainerLoaderFileXml`と、
YAMLファイルをロードする`sfServiceContainerLoaderFileYaml`です。

XMLとYAMLの表記法に飛び込む前に、最初に
Symfony Dependency Injectionコンポーネントの別の部分: **"ダンパー
オブジェクト"**を見てみましょう。サービスダンパーはコンテナオブジェクトを受け取り
別のフォーマットに変換します。そしてもちろん、コンポーネントは
XMLとYAMLフォーマット用のダンパーを搭載しています。

XMLフォーマットを導入するために、
`sfServiceContainerDumperXml`ダンパークラスを使用して、
昨日のコンテナサービスを`container.xml`ファイルに変換してみましょう。

`Zend_Mail`サービスを定義するために使ったコードを覚えていますか?

    [php]
    require_once '/PATH/TO/sfServiceContainerAutoloader.php';
    sfServiceContainerAutoloader::register();

    $sc = new sfServiceContainerBuilder();

    $sc->
      register('mail.transport', 'Zend_Mail_Transport_Smtp')->
      addArgument('smtp.gmail.com')->
      addArgument(array(
        'auth'     => 'login',
        'username' => '%mailer.username%',
        'password' => '%mailer.password%',
        'ssl'      => 'ssl',
        'port'     => 465,
      ))->
      setShared(false)
    ;

    $sc->
      register('mailer', '%mailer.class%')->
      addMethodCall('setDefaultTransport', array(new sfServiceReference('mail.transport')))
    ;

このコンテナをXML表記に変換するには、次のコードを使います:

    [php]
    $dumper = new sfServiceContainerDumperXml($sc);

    file_put_contents('/somewhere/container.xml', $dumper->dump());

ダンパークラスのコンストラクタは最初の引数としてサービスコンテナビルダーオブジェクトを受け取り
`dump()`メソッドはコンテナサービスをイントロスペクトして
これらを別の表現に変換します。すべてがうまくゆけば、
`container.xml`ファイルは次のようなXMLスニペットになります:

    [xml]
    <?xml version="1.0" ?>

    <container xmlns="http://symfony-project.org/2.0/container">
      <parameters>
        <parameter key="mailer.username">foo</parameter>
        <parameter key="mailer.password">bar</parameter>
        <parameter key="mailer.class">Zend_Mail</parameter>
      </parameters>
      <services>
        <service id="mail.transport" class="Zend_Mail_Transport_Smtp" shared="false">
          <argument>smtp.gmail.com</argument>
          <argument type="collection">
            <argument key="auth">login</argument>
            <argument key="username">%mailer.username%</argument>
            <argument key="password">%mailer.password%</argument>
            <argument key="ssl">ssl</argument>
            <argument key="port">465</argument>
          </argument>
        </service>
        <service id="mailer" class="%mailer.class%">
          <call method="setDefaultTransport">
            <argument type="service" id="mail.transport" />
          </call>
        </service>
      </services>
    </container>

>**TIP**
>XMLフォーマットは無名サービスをサポートします。無名サービスは
>名前を必要としないサービスで使用コンテキストで直接定義されます。
>特定のスコープの外側で使われないサービスが必要なときに
>とても便利です:
>
>     [xml]
>     <service id="mailer" class="%mailer.class%">
>       <call method="setDefaultTransport">
>         <argument type="service">
>           <service class="Zend_Mail_Transport_Smtp">
>             <argument>smtp.gmail.com</argument>
>             <argument type="collection">
>               <argument key="auth">login</argument>
>               <argument key="username">%mailer.username%</argument>
>               <argument key="password">%mailer.password%</argument>
>               <argument key="ssl">ssl</argument>
>               <argument key="port">465</argument>
>             </argument>
>           </service>
>         </argument>
>       </call>
>     </service>

XMLサービスローダークラスのおかげでコンテナのロードはとてもシンプルです:

    [php]
    require_once '/PATH/TO/sfServiceContainerAutoloader.php';
    sfServiceContainerAutoloader::register();

    $sc = new sfServiceContainerBuilder();

    $loader = new sfServiceContainerLoaderFileXml($sc);
    $loader->load('/somewhere/container.xml');

ダンパーに関して、ローダーはコンストラクタの最初の引数としてサービスコンテナビルダーを受け取り
`load()`メソッドはファイルを読み込みサービスをコンテナに登録します。
コンテナは普通に使えます。

`sfServiceContainerDumperYaml`クラスを使うようにダンパーのコードを変更すれば、
サービスのYAML表現が手に入ります:

    [php]
    require_once '/PATH/TO/sfYaml.php';

    $dumper = new sfServiceContainerDumperYaml($sc);

    file_put_contents('/somewhere/container.yml', $dumper->dump());

>**NOTE**
>この方法はサービスコンテナのローダーとダンパーのために必要なものとしてsfYAMLコンポーネント
>(`http://svn.symfony-project.com/components/yaml/trunk/`)を
>最初にロードする場合のみ動作します。

前のコンテナは次のようなYAMLで表現されます:

    [yml]
    parameters:
      mailer.username: foo
      mailer.password: bar
      mailer.class:    Zend_Mail

    services:
      mail.transport:
        class:     Zend_Mail_Transport_Smtp
        arguments: [smtp.gmail.com, { auth: login, username: %mailer.username%, password: %mailer.password%, ssl: ssl, port: 465 }]
        shared:    false
      mailer:
        class: %mailer.class%
        calls:
          - [setDefaultTransport, [@mail.transport]]

>**SIDEBAR**
>サービス定義用のベストフォーマットは何か？
>
>XMLフォーマットを使うことでYAMLより優れた利点がいくつかもたらされます:
>
>  * XMLファイルがロードされるとき、
>  組み込みの`services.xsd`ファイルで自動的にバリデートされます;
>
>  * IDEでXMLは自動入力補完されます;
>
>  * XMLフォーマットの処理はYAMLよりも速い;
>
>  * XMLフォーマットは外部依存を持たない(YAMLフォーマットは
>  sfYAMLコンポーネントに依存する)。

もちろんあるフォーマットを別のフォーマットを変更するために
ローダーとダンパーを組み合わせることができます:

    [php]
    // XMLコンテナサービスの定義ファイルをYAMLに変換する
    $sc = new sfServiceContainerBuilder();

    $loader = new sfServiceContainerLoaderFileXml($sc);
    $loader->load('/somewhere/container.xml');

    $dumper = new sfServiceContainerDumperYaml($sc);
    file_put_contents('/somewhere/container.yml', $dumper->dump());

>**TIP**
>この章を短くするために、YAMLもしくはXMLフォーマットの
>すべてのありえる場合の一覧は示しません。しかし既存のコンテナを変換して
>出力を見れば簡単に学べます。付録を見れば、
>フォーマットが詳細に書かれています。

サービスの設定にYAMLもしくはXMLファイルを使うことで
GUIでサービスを設定できるようになります(まだやっていません...)。
しかしこれはより興味深い可能性も開きます。

もっとも重要なことは他の"リソース"をインポートする機能です。
リソースは他の設定ファイルになることができます:

    [xml]
    <container xmlns="http://symfony-project.org/2.0/container">
      <imports>
        <import resource="default.xml" />
      </imports>
      <parameters>
        <!-- These parameters override the ones defined in default.xml -->
      </parameters>
      <services>
        <!-- These service definitions override the ones defined in default.xml -->
      </services>
    </container>

`imports`セクションは他のリソースが評価される前にインクルードされる
必要のあるリソースの一覧を示します。デフォルトでは、現在のファイルへの相対パスで
ファイルが捜索されますが、
ローダーの2番目の引数として探すパスの配列を渡すこともできます:

    [php]
    $loader = new sfServiceContainerLoaderFileXml($sc, array('/another/path'));
    $loader->load('/somewhere/container.xml');

リソースをロードできる`class`を定義することで、
XMLの定義ファイルにYAMLの定義ファイルを埋め込むこともできます:

    [xml]
    <container xmlns="http://symfony-project.org/2.0/container">
      <imports>
        <import resource="default.yml" class="sfServiceContainerLoaderFileYaml" />
      </imports>
    </container>

そしてもちろん、YAMLフォーマットにも同じことがあてはまります:

    [yml]
    imports:
      - { resource: default.xml, class: sfServiceContainerLoaderFileXml }

`import`ファシリティはサービスの定義ファイルを編成するための柔軟な方法を提供します。
定義ファイルを共有して再利用するのも素晴らしい方法です。
最初の章で紹介したWebセッションの例について話しましょう。
テスト環境でWebセッションを使うとき、おそらくセッションストレージオブジェクト
はモックである必要があります; 一方で、ロードバランスされたいくつかのWebサーバーがある場合、
本番環境ではセッションをMySQLのようなデータベースに保存する必要があります。
環境に基づいて異なる設定を用意する1つの方法は
異なる複数の設定ファイルを作成して
必要に応じてインポートします:

    [xml]
    <!-- in /framework/config/default/session.xml -->
    <container xmlns="http://symfony-project.org/2.0/container">
      <parameters>
        <parameter key="session.class">sfSessionStorage</parameter>
      </parameters>

      <!-- service definitions go here -->
    </container>

    <!-- in /project/config/session_test.xml -->
    <container xmlns="http://symfony-project.org/2.0/container">
      <imports>
        <import resource="session.xml" />
      </imports>

      <parameters>
        <parameter key="session.class">sfSessionTestStorage</parameter>
      </parameters>
    </container>

    <!-- in /project/config/session_prod.xml -->
    <container xmlns="http://symfony-project.org/2.0/container">
      <imports>
        <import resource="session.xml" />
      </imports>

      <parameters>
        <parameter key="session.class">sfMySQLSessionStorage</parameter>
      </parameters>
    </container>

正しい設定を使うのはささいなことです:

    [php]
    $loader = new sfServiceContainerLoaderFileXml($sc, array(
      '/framework/config/default/',
      '/project/config/',
    ));
    $loader->load('/somewhere/session_'.$environment.'.xml');

XMLはもっとも読みやすい設定フォーマットではないので、
設定を定義するのにXMLを使うのを嫌がる人を耳にしますが、
Symfonyの背景知識があれば、すべてのファイルをYAMLフォーマットで書くことができます。
設定からサービスの定義を分離することもできます。
他のファイルからインポートできるので、`services.xml`ファイルでサービスを定義し、
`parameters.xml`ファイルで関連設定を保存できます。
YAMLファイルでパラメータも定義できます(`parameters.yml`)。
最後に、標準のINIファイルからパラメータを読み込むことができる
組み込みのINIローダーがあります:

    [xml]
    <!-- in /project/config/session_test.xml -->
    <container xmlns="http://symfony-project.org/2.0/container">
      <imports>
        <import resource="config.ini" class="sfServiceContainerLoaderFileIni" />
      </imports>
    </container>

    <!-- /project/config/config.ini -->
    [parameters]
    session.class = sfSessionTestStorage

>**NOTE**
>INIファイルでサービスを定義することはできません; パラメータのみが定義され
>解析されます。

これらの例はコンテナのローダーとダンパーの機能の表面をふれただけですが、
PHPフォーマットを越えるXMLとYAMLフォーマットの力の概要がよくわかっていただければ幸いです。
複数の設定ファイルをロードする必要のある
コンテナのパフォーマンスに疑問がある方は
次の章ですっきりすると思います。
