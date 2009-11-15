サービスを作成するためにビルダーを使用する
========================================

前の章では、サービスコンテナにより魅力的なインターフェイスを提供するために
`sfServiceContainer`クラスの使い方を学びました。
この章では、純粋なPHPコードでサービスと設定を記述するためにさらに一歩進んで
`sfServiceContainerBuilder`クラスを活用する方法を学びます。


`sfServiceContainerBuilder`クラスは`sfServiceContainer`クラスを継承し
開発者がシンプルなPHPインターフェイスで
サービスを記述できるようにします。

>**SIDEBAR**
>Service Containerインターフェイス
>
>すべてのサービスコンテナクラスは`sfServiceContainerInterface`で定義される
>同じインターフェイスを共有します:
>
>     [php]
>     interface sfServiceContainerInterface
>     {
>       public function setParameters(array $parameters);
>       public function addParameters(array $parameters);
>       public function getParameters();
>       public function getParameter($name);
>       public function setParameter($name, $value);
>       public function hasParameter($name);
>       public function setService($id, $service);
>       public function getService($id);
>       public function hasService($name);
>     }

サービスの記述はサービスの定義を登録することで行われます。
それぞれのサービスの定義はサービス: 使うクラスから
コンストラクタに渡す引数、と一連の他の設定
プロパティ(下記の`sfServiceDefinition`のサイドバーを参照)までを記述します。

`Zend_Mail`の例はすべての決め打ちされたコードを取り除き
代わりにビルダークラスで動的に構築することで簡単に書き換えることができます:

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

`register()`メソッドを呼び出すことでサービスの作成が行われます。
このメソッドはサービス名とクラス名を受け取り、`sfServiceDefinition`インスタンスを
返します。

>**TIP**
>内部ではサービスの定義は
>`sfServiceDefinition`クラスのオブジェクトで表現されます。
>サービスを手動で作成してサービスコンテナの
>`setServiceDefinition()`メソッドを使って直接登録することも可能です。

定義オブジェクトは流れるようなインターフェイスを実装し
サービスを設定するメソッドを提供します。上記の例では、次のメソッドを
使いました:

  * `addArgument()`: サービスのコンストラクタに渡す引数を追加する。

  * `setShared()`: コンテナに対してサービスが一意的でなければならないかどうか
    (デフォルトは`true`)。

  * `addMethodCall()`: サービスが作成された後で呼び出すメソッド。
    2番目の引数はメソッドに渡す引数の配列。

サービスの参照は`sfServiceReference`インスタンスで行われます。
この特別なオブジェクトは参照オブジェクトが作成される際に
実際のサービスに動的に置き換えられます。

登録フェーズの間に、サービスは実際には作成されず、
単なるサービスの説明に関するものです。サービスは実際に連携したいときのみに作成されます。
このことはサービス間の依存関係を気にせずに
任意の順序でサービスを登録できることを意味します。またこのことは
同じ名前でサービスを再登録することで
既存のサービス定義をオーバーライドできることも意味します。
テストを目的にサービスをオーバーライドするための別のシンプルな手段
でもあります。

>**SIDEBAR**
>`sfServiceDefinition`クラス
>
>サービスの作成と設定方法を変更する複数のプロパティを
>持ちます:
>
> * `setConstructor()`: サービスが作成されるときに
> 標準の`new`コンストラクタの代わりに使うstaticメソッドをセットする(
> factoryに便利)。
>
> * `setClass()`: サービスのクラスをセットする。
>
> * `setArguments()`: コンストラクタに渡す複数の引数をセットする(
> もちろん順序は大切です)。
>
> * `addArgument()`: コンストラクタ用に1つの引数を追加する。
>
> * `setMethodCalls()`: サービス作成の後で呼び出すサービスメソッドを
> セットする。これらのメソッドは登録と同じ順序で
> 呼び出されます。
>
> * `addMethodCall()`: サービス作成の後で呼び出すサービスメソッドコールを追加する。
> 必要であれば同じメソッドの呼び出しを
> 何度も追加できます。
>
> * `setFile()`: サービスを作成する前にインクルードするファイルをセットする
> (サービスクラスがオートロードされていない場合に便利です)。
>
> * `setShared()`: コンテナに対してサービスが一意的でなければならないか
> どうか(デフォルトは`true`)。
>
> * `setConfigurator()`: サービスが設定された後で呼び出すPHPのcallable
> をセットする

`sfServiceContainerBuilder`クラスは標準の`sfServiceContainerInterface`インターフェイスを
実装するので、サービスコンテナを使うために
これを変更する必要はありません:

    [php]
    $sc->addParameters(array(
      'mailer.username' => 'foo',
      'mailer.password' => 'bar',
      'mailer.class'    => 'Zend_Mail',
    ));

    $mailer = $sc->mailer;

`sfServiceContainerBuilder`は任意のオブジェクトのインスタンス化と設定が記述できます。
`Zend_Mail`クラスでこれを実演しましたが、
Symfonyの`sfUser`クラスを使用する別の例は次の通りです:

    [php]
    $sc = new sfServiceContainerBuilder(array(
      'storage.class'        => 'sfMySQLSessionStorage',
      'storage.options'      => array('database' => 'session', 'db_table' => 'session'),
      'user.class'           => 'sfUser',
      'user.default_culture' => 'en',
    ));

    $sc->register('dispatcher', 'sfEventDispatcher');

    $sc->
      register('storage', '%storage.class%')->
      addArgument('%storage.options%')
    ;

    $sc->
      register('user', '%user.class%')->
      addArgument(new sfServiceReference('dispatcher'))->
      addArgument(new sfServiceReference('storage'))->
      addArgument(array('default_culture' => '%user.default_culture%'))->
    ;

    $user = $sc->user;

>**NOTE**
>Symfonyの例では、ストレージオブジェクトは引数としてオプションの配列を受け取りますが
>この例では、文字列のプレースホルダーを受け取りました。
>(`addArgument('%storage.options%')`)。コンテナは
>プレースホルダーの値である配列を実際に渡すほど十分に賢いです。

サービスを記述するためにPHPコードを使うのはとてもシンプルで強力です。
これは抽象クラスのインスタンス化とオブジェクトの設定をするために
たくさんのコードを重複せずにコンテナを作成するツールを提供します。
