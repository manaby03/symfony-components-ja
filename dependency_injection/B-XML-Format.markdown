付録B - XMLフォーマット
======================

この付録ではパラメータをサービスを記述するために
XMLフォーマットを説明します。

### フォーマット

XMLファイルはコンポーネントに解析される前に
[XSD](http://github.com/fabpot/dependency-injection/blob/0d2d88248920eee0e6f382791ae6d075d1ef19d3/lib/services.xsd)
ファイルによって常にバリデートされます。

最小限度の妥当なXMLファイルは次のようになります:

    [xml]
    <?xml version="1.0" ?>

    <container xmlns="http://symfony-project.org/2.0/container">
    </container>

パラメータとサービスはメインの`<container>`タグの元で記述されます。
ファイル内ですべての要素に接頭辞をつけないようにする
`xmlns`属性の使い方に注目してください。

公式の名前空間は`http://symfony-project.org/2.0/container`です。

`<container>`タグの元で、3つの妥当なタグのどれかを使うことができます:

  * `<parameters>`: はコンテナパラメータを定義する
  * `<services>`:   はコンテナサービスを定義する
  * `<imports>`:    ファイルを解析する前にインポートするファイルを定義する

それぞれのタイプのタグは最大で1つ使うことができます。それぞれのタイプは
次のセクションで説明します。

### プレースホルダー

大抵のキーと値はプレースホルダーを使うことができます。プレースホルダーは`%`記号で囲まれた文字列で、
実行時に対応するパラメータの値に動的に
置き換えられます。

### 優先ルール

XMLリソースをロードするとき、サービスの定義は現在のものを
オーバーライドします。

しかしパラメータに関して、これらは現在のものでオーバーライドされます。
ロードされたルールより優先させるためにコンテナコンストラクタに
パラメータを渡すことができます。

    [php]
    $container = new sfServiceContainerBuilder(array('foo' => 'bar'));
    $loader = new sfServiceContainerLoaderFileXml($container);
    $loader->load('services.xml');

上記の例では、ロードされたリソースが`foo`パラメータを定義する場合でも、
値はビルダーコンストラクタで定義された'bar'のままになります。

パラメータ
----------

それぞれのパラメータは`<parameter>`タグを使用することでパラメータを定義しなければなりません:

    [xml]
    <parameters>
      <parameter>a string</parameter>
    </parameters>

デフォルトでは、パラメータは(PHP配列として)PHPによって生成されたキーを持ちます。
以前のXMLは次のPHP配列と同等です:

    [php]
    array('a string')

ハッシュを作りたい場合、`key`属性を使うことで
独自のキーを定義できます:

    [xml]
    <parameters>
      <parameter key="foo">a string</parameter>
    </parameters>

上記のコードは次のPHPコードと同等です:

    [php]
    array('foo' => 'a string')

`type`属性を`collection`に変更することで配列も定義できます:

    [xml]
    <parameters>
      <parameter key="values" type="collection">
        <parameter>foo</parameter>
        <parameter>bar</parameter>
      </parameter>
    </parameters>

もちろん配列を入れ子にすることができます。

XMLパラメータを解析するとき、コンポーネントは値をキャストすることも
自動的に行います:

  * `true`、`on`、`false`、と`off`文字列は`true`
    と`false`に変換される

  * `null`の文字列はPHPの`null`の値に変換される

  * 数値を表す文字列はPHPの数値に変換される
    (integer, octal notation, and hexadecimal notations are supported)

`string`型を使うことで特殊な値を文字列として
解釈されるように強制できます:

    [xml]
    <parameters>
      <parameter key="foo">true</parameter>
      <parameter key="bar" type="string">true</parameter>
    </parameters>

以前の例では、the converted PHP array will be similar to the
following one:

    [php]
    array('foo' => true, 'bar' => 'true')

パラメータの値はプレースホルダーを含むことができます:

    [xml]
    <parameters>
      <parameter key="foo">true</parameter>
      <parameter key="bar">%foo%</parameter>
      <parameter key="baz">The placeholders can be %foo% embedded in a string</parameter>
    </parameters>

上述のXMLスニペットは次のPHPコードと同等です:

    [php]
    array('foo' => true, 'bar' => true, 'baz' => 'The placeholders can be true embedded in a string')

`%`は2つ重ねることでエスケープできます:

    [xml]
    <parameters>
      <parameter key="foo">The string has no placeholder... %%foo</parameter>
    </parameters>

サービス
--------

サービスは`<service>`タグによって定義されます:

    [xml]
    <services>
      <service id="foo" class="FooClass" />
    </services>

`id`と`class`属性はサービスを定義するための最小の必須項目です。

### 属性

`<service>`タグは次の属性をサポートします:

  * `id`: サービスの一意的な識別子(必須)

  * `class`: サービスのクラスの名前(必須で、プレースホルダーが可)

  * `shared`: サービスを共有するかどうか

  * `constructor`: サービスのインスタンスを作成するために呼び出すコンストラクタである
    staticメソッド

### タグ

メインの`<service>`タグの元で、サービスを設定するためにいくつかの他のオプションタグを
使うことができます:

  * `<file>`: サービスをインスタンス化する前に必要な
    ファイルの絶対パス(プレースホルダーが使用可)。

  * `<argument>`: コンストラクタに渡す引数(順序が
    重要で、値はプレースホルダーが可)。

    引数はパラメータとして同じ表記を使うことができます。It also supports
    one more type, `service`、これは別のサービスを参照するために使うことができます。
    `service`のタイプを使う場合、`id`属性も定義しなければなりません。
    これは参照したいサービスの一意的な識別子です:

        [xml]
        <argument>foo</argument>
        <argument type="service" id="foo" />
        <argument type="collection">
          <argument>true</argument>
          <argument>false</argument>
        </argument>

  * `<configurator>`: インスタンス化後でサービスを設定するために
    呼び出すcallable(callableは引数としてサービスの
    インスタンスに渡されます)。

    configuratorは関数になります:

        [xml]
        <configurator function="configure" />

    既存のサービスに呼び出されるメソッド:

        [xml]
        <configurator service="baz" method="configure" />

    もしくはstaticメソッド:

        [xml]
        <configurator class="BazClass" method="configureStatic" />

  * `<call>`: インスタンス化した後でサービスに呼び出すメソッド
    (いくつかの`<call>`タグを定義できます)。

    メソッドは`method`属性で定義し、そのメソッドに渡す引数は
    `<argument>`タグで定義できます。

もっとも多くの属性を使う例は次の通りです:

    [xml]
    <service id="bar" class="FooClass" shared="true" constructor="getInstance">
      <file>%path%/foo.php</file>
      <argument>foo</argument>
      <argument type="service" id="foo" />
      <argument type="collection">
        <argument>true</argument>
        <argument>false</argument>
      </argument>
      <configurator function="configure" />
      <call method="setBar" />
      <call method="setBar">
        <argument>foo</argument>
        <argument type="service" id="foo" />
        <argument type="collection">
          <argument>true</argument>
          <argument>false</argument>
        </argument>
      </call>
    </service>

### 無名サービス

無名サービスを定義することもできます。これは別のサービスコンストラクタ、
configuratorもしくはメソッド呼び出しに渡すことができるサービスを作る際に便利です。
スコープの外側からアクセスする必要がない場合、
一意的な`id`を定義し、メインの名前空間を散らかす必要は
ありません。

無名サービスの定義は`service`タイプの`argument`を定義するだけです。
`id`を定義しませんが、引数として`<service>`の定義を提供します:

    [xml]
    <services>
      <service id="foo" class="FooClass">
        <argument type="service">
          <service class="BarClass" />
        </argument>
      </service>
    </services>

`argument`、および`service`属性のどちらも`id`を定義していないことに
注目してください。

インポート
----------

XMLファイルが解析される前に、最初にコンポーネントは
`<imports>`タグの元で定義されたインポートリソースを読み込みます:

    [xml]
    <imports>
      <import resource="services.xml" />
    </imports>

多くのリソースをインポートする場合、これらが定義された順序で解釈されます。
1つのリソースは以前定義したパラメータとサービスをオーバーライドできるので、
順序は大切です。

リソースが相対パスである場合、最初にリソースはXMLファイルと同じディレクトリで捜索されます。
見つからない場合、パスはローダーコンストラクタに渡され
2番目の引数が次から次に
捜索されます。

デフォルトでは、現在のローダーと同じものが使われますが、
`class`属性を定義することで他のローダークラスを使うこともできます:

    [xml]
    <imports>
      <import resource="services.xml" />
      <import resource="../ini/parameters.ini" class="sfServiceContainerLoaderFileIni" />
    </imports>

オリジナルのローダーと同じパスは新しいものに渡されます。

コンポーネントはパラメータをシンプルなキー/値の組とみなすので、
値がオーバーライドされる際に、値は新しい値に置き換えられます。

例えば、次の2つのXMLファイルがある場合:

    [xml]
    file1.xml
    <parameter key="complex" type="collection">
      <parameter>true</parameter>
      <parameter>false</parameter>
    </parameter>

    file2.xml
    <parameter key="complex">foo</parameter>

`file1.xml`と`file2.xml`がこの順序でロードされるとき、complexの値は
"foo"になります。
