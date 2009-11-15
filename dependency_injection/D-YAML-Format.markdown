付録D - YAMLフォーマット
=======================

この付録ではパラメータをサービスを記述するために使われる
YAMLフォーマットを説明します。

### フォーマット

YAMLファイルはXMLのようにバリデートできません。ですのでこれらを書く際には
注意深く行うことが必要です。

YAMLファイルは3つの主要なエントリを定義できます:

  * `parameters`: コンテナパラメータを定義する
  * `services`:   コンテナサービスを定義する
  * `imports`:    ファイルを解析する前にインポートするファイルを定義する

### プレースホルダー

たいていのキーと値はプレースホルダーを使うことができます。プレースホルダーは
`%`記号で囲まれた文字列で、実行時に対応するパラメータの値に
動的に置き換えられます。

### 優先ルール

YAMLリソースをロードするとき、サービスの定義は現在定義されているものを
オーバーライドします。

しかしパラメータに関しては、現在のものでオーバーライドされます。これによって
コンテナコンストラクタに渡されるパラメータにおいてロードされた優先順位よりも
優先させることができます。

    [php]
    $container = new sfServiceContainerBuilder(array('foo' => 'bar'));
    $loader = new sfServiceContainerLoaderFileYaml($container);
    $loader->load('services.yml');

上記の例では、ロードされるリソースが`foo`パラメータを定義する場合でも、
ビルダーコンストラクタで定義されたように値は'bar'のままです。

パラメータ
----------

YAML配列で定義されます。すべてのYAMLルールが適用されます:

    [yml]
    parameters:
      foo: bar
      values:
        - true
        - false
        - 0
        - 1000.3

パラメータの値はプレースホルダーを格納できます:

    [yml]
    parameters:
      foo: bar
      bar: %foo%
      baz: The placeholders can be %foo% embedded in a string

上記のYAMLスニペットは次のPHPコードと同等です:

    [php]
    array('foo' => true, 'bar' => true, 'baz' => 'The placeholders can be true embedded in a string')

`%`は2つ重ねることでエスケープできます:

    [yml]
    parameters:
      foo: The string has no placeholder... %%foo

サービス
--------

ハッシュを作成することでサービスを定義します。キーはサービスの
一意的な識別子を表します:

    [yml]
    services:
      foo: { class: FooClass }

`class`エントリはサービスを定義するための最小限の必須項目です。

### 属性

`service`エントリは次の属性をサポートします:

  * `class`: サービスのクラス名(必須で、プレースホルダーが使用可能)。

  * `shared`: サービスを共有するかどうか

  * `constructor`: サービスのインスタンスを作成するために呼び出す
    staticメソッドのコンストラクタ。

  * `file`: サービスをインスタンス化する前に必要な
    ファイルの絶対パス(プレースホルダーが使用可能)。

  * `arguments`: コンストラクタに渡す引数(順序が
    重要で、値はプレースホルダーを使うことができます)。

    引数はYAML配列表記を使うことで定義されます。
    `@`記号を使用することでサービスへの参照もサポートされます:

        [yml]
        services:
          foo: { class: FooClass, arguments: [foo, @bar] }
          bar: { class: BarClass }

  * `configurator`: インスタンス化後でサービスを設定するために
    呼び出すcallable(callableは引数としてサービスのインスタンスに
    渡されます)。

    configuratorは関数になります:

        [yml]
        foo: { class: FooClass, configurator: configure }

    もしくは既存のサービスに呼び出されたメソッド:

        [yml]
        foo: { class: FooClass, configurator: [@baz, configure] }

    もしくはstaticメソッド:

        [yml]
        foo: { class: FooClass, configurator: [BazClass, configure] }

  * `calls`: インスタンス化後にサービスインスタンス上で呼び出す
    サービスメソッドの配列。それぞれのメソッド呼び出しは配列、最初の要素はメソッドの名前で
    2番目の要素はそのメソッドに渡す引数の配列です。

エントリを最大限使う例は次の通りです:

    [yml]
      foo:
        class: FooClass
        constructor: getInstance
        shared: false
        file: %path%/foo.php
        arguments: [foo, @foo, [true, false]]
        configurator: [@baz, configure]
        calls:
          - [ setBar, [ foo, @foo, [true, false] ] ]

インポート
----------

YAMLファイルが解析される前に、最初にコンポーネントは
`imports`エントリの元で定義されたインポートリソースを読み込みます:

    [yml]
    imports:
      - { resource: services.yml }

多くのリソースをインポートする場合、これらが定義された順序で解釈されます。
1つのリソースは以前定義したパラメータとサービスをオーバーライドできるので、
順序が大切です。

リソースが相対パスである場合、最初にリソースが
現在のYAMLファイルと同じディレクトリでリソースが捜索されます。見つからなければ、ローダーコンストラクタの2番目の引数に渡されるパスが次々と
捜索されます。

デフォルトでは、現在のローダーと同じものが使われますが、
`class`属性を定義することで他のローダークラスも使うことができます:

    [yml]
    imports:
      - { resource: services.yml }
      - { resource: "../ini/parameters.ini", class: sfServiceContainerLoaderFileIni }

オリジナルのローダーと同じパスが新しいローダーに渡されます。

パラメータはコンポーネントによってシンプルなキー/値の組として見なされるので値がオーバーライドされるとき
、値は新しい値に置き換えられます。

例えば、次の2つのYAMLファイルがある場合:

    [yml]
    file1.yml
    parameters:
      complex: [true, false]

    file2.yml
    parameters:
      complex: foo

この順番で`file1.yml`と`file2.yml`をロードするとき、complexの値
は"foo"になります。
