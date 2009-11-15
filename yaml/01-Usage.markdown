Symfony YAMLを使う
==================

Symfony YAMLライブラリはとてもシンプルな2つのメインクラスで構成されます: 1つは
YAML文字列は解析し(`sfYamlParser`)、もう一つはPHP配列から
YAML文字列を吐き出します(`sfYamlDumper`)。

これら2つのコアクラスに加えて、メインの`sfYaml`クラスは薄いラッパーとして振る舞い
一般的な一般的な使い方を簡略化します。

YAMLファイルを読み込む
---------------------

`sfYamlParser::parse()`メソッドはYAML文字列を解析し
PHP配列に変換します:

    [php]
    $yaml = new sfYamlParser();
    $value = $yaml->parse(file_get_contents('/path/to/file.yaml'));

解析の間にエラーが起きる場合、エラーの種類とエラーが起きたオリジナルのYAML文字列の行を
示す例外をパーサーは投げます:

    [php]
    try
    {
      $value = $yaml->parse(file_get_contents('/path/to/file.yaml'));
    }
    catch (InvalidArgumentException $e)
    {
      // 解析の間に起きたエラー
      echo "Unable to parse the YAML string: ".$e->getMessage();
    }

>**TIP**
>パーサーはリエントラントなので、異なるYAML文字列をロードするために
>同じパーサーオブジェクトを使うことができます。

YAMLファイルをロードするとき、`sfYaml::load()`ラッパーメソッドを使う方が
良いことがあります:

    [php]
    $loader = sfYaml::load('/path/to/file.yml');

staticな`sfYaml::load()`メソッドはYAML文字列もしくはYAMLを含むファイルを受け取ります。
内部では、`sfYamlParser::parse()`メソッドが呼び出されますが、
おまけが追加されます:

  * これはYAMLファイルをPHPファイルのように実行するので
    YAMLファイルにPHPコマンドを埋め込むことができます。

  * ファイルが解析できない場合、これはエラーメッセージにファイルの名前を自動的に追加し
    アプリケーションが複数のYAMLファイルをロードする際に
    デバッグ作業を簡略化します。

YAMLファイルを書く
-----------------

`sfYamlDumper`はPHP配列からYAML表記を吐き出します:

    [php]
    $array = array('foo' => 'bar', 'bar' => array('foo' => 'bar', 'bar' => 'baz'));

    $dumper = new sfYamlDumper();
    $yaml = $dumper->dump($array);
    file_put_contents('/path/to/file.yaml', $yaml);

>**NOTE**
>もちろん、Symfony YAMLダンパーはリソースをダンプできません。また、
>ダンパーがPHPオブジェクトをダンプできる場合でも、
>アルファ版の機能とみなされます。

1つの配列をダンプすることだけ必要であれば、staticな`sfYaml::dump()`メソッドの
ショートカットを使うことができます:

    [php]
    $yaml = sfYaml::dump($array, $inline);

YAMLフォーマットは配列、拡張版、とインライン版の2種類の表記をサポートします。
デフォルトでは、ダンパーはインライン表記を
使用します:

    [yml]
    { foo: bar, bar: { foo: bar, bar: baz } }

`dump()`メソッドの2番目の引数はYAMLの展開表記から
インライン表記に切り替える出力レベルを
カスタマイズします:

    [php]
    echo $dumper->dump($array, 1);

-

    [yml]
    foo: bar
    bar: { foo: bar, bar: baz }

-

    [php]
    echo $dumper->dump($array, 2);

-

    [yml]
    foo: bar
    bar:
      foo: bar
      bar: baz
