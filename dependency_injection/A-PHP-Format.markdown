付録A - PHPフォーマット
======================

この付録ではサービスを記述するために使われるPHPフォーマットを説明します。

### フォーマット

PHPでサービスを記述するには、
`sfServiceContainerBuilder`クラスを使うことができます:

    [php]
    $container = new sfServiceContainerBuilder();

### プレースホルダー

大抵のキーと値はプレースホルダーを使うことができます。プレースホルダーは
`%`記号で囲まれた文字列で、実行時に対応するパラメータの値に
動的に置き換えられます。

サービス
--------

それぞれのサービスは`sfServiceDefinition`オブジェクトで記述されます。
コンテナにサービスを追加するのは
`setServiceDefinition()`メソッドを使うことで可能です。これはサービスの名前と
`sfServiceDefinition`インスタンスを受け取ります:

    [php]
    $definition = new sfServiceDefinition('FooClass');
    $container->setServiceDefinition('foo', $definition);

`sfServiceContainerBuilder`はサービスを動的に登録するための
流れるようなインターフェイスもサポートします:

    [php]
    $container->register('foo', 'FooClass');

`register()`メソッドは2番目の引数として渡したクラスの名前に基づいて
`sfServiceDefinition`オブジェクトを返します。
ですので2つの例は厳密に同等です。

今後は、具体例に対して流れるようなインターフェイスを使うことにします。

### 共有サービス

デフォルトで、サービスは共有されます。コンテナから
特定のサービスを取得するたびに、同じインスタンスが返されます:

    [php]
    $container->getService('foo') === $container->getService('foo')

サービスを取得するたびに新しいインスタンスを取得したい場合、
非共有なものとして宣言する必要があります:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->setShared(false)
    ;

`foo`サービスを取得するたびに、
このインスタンスが手に入ります:

    [php]
    $container->getService('foo') !== $container->getService('foo')

### コンストラクタ

コンテナがサービスを作成するとき、
PHPの`new`演算子が使われます。クラスコンストラクタの新しいインスタンスが
staticメソッドで作成しなければならないのでクラスコンストラクタが
protectedである場合(例えばクラスが例えばfactoryもしくはsingletonである)、
`setConstructor()`メソッドを使用して記述します:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->setConstructor('getInstance')
    ;

### ファイル

デフォルトでは、サービスコンテナはサービス作成用のオートロード機能に依存します。
しかしクラスがオートロードされない場合、`setFile()`メソッドを通して
クラスのインスタンスを作成する直前に
コンテナに必要なファイルを提供する必要があります:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->setFile('/path/to/FooClass.php')
    ;

パスにもプレースホルダーを使うことができます:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->setFile('/path/to/%foo.class_file%.php')
    ;

### 引数

引数にサービスのクラスコンストラクタを渡す必要がある場合、
`addArgument()`メソッドでこれらを定義できます:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->addArgument('foo')
      ->addArgument(array(1, 2))
    ;

>**NOTE**
>引数を登録する順序はコンストラクタで定義されたものと
>同じでなければなりません。

それぞれの引数はプレースホルダーを使うことができます。

サービスがコンストラクタに注入する別のサービスを必要とする場合、
`sfServiceReference`クラスのインスタンスを渡します:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->addArgument(new sfServiceReference('bar'))
    ;

>**NOTE**
>1つの呼び出しですべての引数を定義するために`setArguments()`メソッドを
>使うこともできます。これはユニークな引数としてコンストラクタに渡す
>引数の配列を受け取ります。

### configurator

サービスをインスタンス化した後で、コンテナは
configuratorを呼び出すことができます。configuratorは
さらにサービスを設定することができるようになるcallableです。callableは引数として
サービスのインスタンスに渡されます。

configuratorは関数になることができます:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->setConfigurator('configure')
    ;

もしくは既存のサービスで呼び出されるメソッド:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->setConfigurator(array(new sfServiceReference('baz'), 'configure'))
    ;

もしくはstaticメソッド:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->setConfigurator(array('BazClass', 'configureStatic'))
    ;

### メソッド呼び出し

`sfServiceDefinition`クラスは`addMethodCall()`メソッドを通して
メソッドインジェクションもサポートします:

    [php]
    $container
      ->register('foo', 'FooClass')
      ->addMethodCall('configure', array('foo'))
      ->addMethodCall(array('BazClass', 'configureStatic'))
      ->addMethodCall(array('BazClass', 'configureStatic'))
    ;

最初の引数はサービスで呼び出すメソッドで、
2番目の引数は呼び出す際に使う引数の配列です。
