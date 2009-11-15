速さの必要性
============

この本の最初の5章で、このシンプルで便利なデザインパターンの背後にある
主要な概念を段階的に紹介してきました。
Symfony 2のコアコンポーネントの1つになる
軽量のPHPコンテナの実装も話しました。

XMLとYAML設定ファイルの紹介で、
コンテナ自身のパフォーマンスに少し疑問を持つようになったかもしれません。
サービスが遅延ロードの場合でも、リクエストごとに一連のXMLもしくはYAMLファイルを読み込み
イントロスペクションでオブジェクトを作成するのはPHPではあまり効率的ではありません。
コンテナはほぼアプリケーションの土台なので、速さが大きな問題になるからです。

一方で、サービスと設定を記述するのにXMLもしくはYAMLを使うのは
とても強力で柔軟です:

    [php]
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
            <argument key="port">true</argument>
          </argument>
        </service>
        <service id="mailer" class="%mailer.class%">
          <call method="setDefaultTransport">
            <argument type="service" id="mail.transport" />
          </call>
        </service>
      </services>
    </container>

しかし、一方で、この本の第2章で見たように、サービスコンテナをプレーンな
PHPクラスとして定義することでフルスピードが得られます:

    [php]
    class Container extends sfServiceContainer
    {
      static protected $shared = array();

      protected function getMailTransportService()
      {
        return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
          'auth'     => 'login',
          'username' => $this['mailer.username'],
          'password' => $this['mailer.password'],
          'ssl'      => 'ssl',
          'port'     => 465,
        ));
      }

      protected function getMailerService()
      {
        if (isset(self::$shared['mailer']))
        {
          return self::$shared['mailer'];
        }

        $class = $this['mailer.class'];

        $mailer = new $class();
        $mailer->setDefaultTransport($this->getMailTransportService());

        return self::$shared['mailer'] = $mailer;
      }
    }

設定変数のおかげで柔軟性を提供するための最小限のことを行い、またとても速いです。

両方の良いところを一度につかえ内でしょうか？これはとてもシンプルです。Symfony
Dependency Injectionコンポーネントは別の組み込みのダンパー: **PHP
ダンパー**を提供します。このダンパーはサービスコンテナをプレーンなPHPコードに変換できます。
これは最初に手で書いたコードを生成できます。

説明を簡潔にするために再度`Zend_Mail`を使ってみましょう。
以前の章で作成したXMLの定義ファイルを使います:

    [php]
    $sc = new sfServiceContainerBuilder();

    $loader = new sfServiceContainerLoaderFileXml($sc);
    $loader->load('/somewhere/container.xml');

    $dumper = new sfServiceContainerDumperPhp($sc);

    $code = $dumper->dump(array('class' => 'Container'));

    file_put_contents('/somewhere/container.php', $code);

他のダンパーに関して、`sfServiceContainerDumperPhp`クラスは
コンストラクタの最初の引数としてコンテナを受け取ります。
`dump()`メソッドはオプションの配列を受け取り、
これらの1つは生成するクラスの名前です。

生成されたコードは次の通りです:

    [php]
    class Container extends sfServiceContainer
    {
      protected $shared = array();

      public function __construct()
      {
        parent::__construct($this->getDefaultParameters());
      }

      protected function getMailTransportService()
      {
        $instance = new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
          'auth' => 'login',
          'username' => $this->getParameter('mailer.username'),
          'password' => $this->getParameter('mailer.password'),
          'ssl' => 'ssl',
          'port' => 465
        ));

        return $instance;
      }

      protected function getMailerService()
      {
        if (isset($this->shared['mailer'])) return $this->shared['mailer'];

        $class = $this->getParameter('mailer.class');
        $instance = new $class();
        $instance->setDefaultTransport($this->getMailTransportService());

        return $this->shared['mailer'] = $instance;
      }

      protected function getDefaultParameters()
      {
        return array (
          'mailer.username' => 'foo',
          'mailer.password' => 'bar',
          'mailer.class' => 'Zend_Mail',
        );
      }
    }

ダンパーによって生成されたコードをよく見ると、 
以前手で書いたコードとよく似ていることがわかります。

>**NOTE**
>できるかぎり速くするために生成されたコードは
>パラメータとサービスにアクセスするために簡略化記法を使いません。

`sfServiceContainerDumperPhp`ダンパーを使うことで、
サービスを記述して設定するためのXMLもしくはYAMLフォーマットの柔軟性、
と最適化され自動生成されたPHPファイルの速度の両方が
得られます。

もちろん、ほとんどの場合プロジェクトは異なる環境ごとに異なる設定を持つので
もちろん環境もしくはデバッグ設定に基づいて異なるコンテナクラスを生成できます。
リクエストのごく最初にコンテナを動的に構築する方法を説明するPHPコードの小さなスニペットは
次の通りです。デバッグモードではないときに他のすべてのリクエスト用のキャッシュされた
コンテナを使います:

    [php]
    $name = 'Project'.md5($appDir.$isDebug.$environment).'ServiceContainer';
    $file = sys_get_temp_dir().'/'.$name.'.php';

    if (!$isDebug && file_exists($file))
    {
      require_once $file;
      $sc = new $name();
    }
    else
    {
      // サービスコンテナを動的に構築する
      $sc = new sfServiceContainerBuilder();
      $loader = new sfServiceContainerLoaderFileXml($sc);
      $loader->load('/somewhere/container.xml');

      if (!$isDebug)
      {
        $dumper = new sfServiceContainerDumperPhp($sc);

        file_put_contents($file, $dumper->dump(array('class' => $name));
      }
    }

このコードはSymfony 2 Dependency Injection Containerのツアーをまとめています。

この本を閉じる前に、ダンパーの別の素晴らしい機能を見てみましょう。
ダンパーは異なる多くの仕事をこなし、コンポーネントの実装の分離を実演するために、
Graphvizダンパーを実装しました。
何のためでしょうか？サービスと依存関係の可視化を手助けするためです。

最初に、コンテナの使い方を見てみましょう:

    [php]
    $dumper = new sfServiceContainerDumperGraphviz($sc);
    file_put_contents('/somewhere/container.dot', $dumper->dump());

Graphvizダンパーはコンテナの`dot`表現を生成します:

    digraph sc {
      ratio="compress"
      node [fontsize="11" fontname="Myriad" shape="record"];
      edge [fontsize="9" fontname="Myriad" color="grey" arrowhead="open" arrowsize="0.5"];

      node_service_container [label="service_container\nsfServiceContainerBuilder\n", shape=record, fillcolor="#9999ff", style="filled"];
      node_mail_transport [label="mail.transport\nZend_Mail_Transport_Smtp\n", shape=record, fillcolor="#eeeeee", style="dotted"];
      node_mailer [label="mailer\nZend_Mail\n", shape=record, fillcolor="#eeeeee", style="filled"];
      node_mailer -> node_mail_transport [label="setDefaultTransport()" style="dashed"];
    }

この表現は
[dotプログラム](http://graphviz.org/)を使用して画像に変換できます:

    $ dot -Tpng /somewhere/container.dot > /somewhere/container.png

![Zend_MailコンテナのPNG画像](http://fabien.potencier.org/media/articles/di/zend-mail.png)

このシンプルな例に対して、可視化は実際に追加された値を持ちませんが、サービスが増えてくると、
便利で美しい
ものになるでしょう...

>**NOTE**
>Graphvizダンパーの`dump()`メソッドはグラフの出力を調整するための
>異なるたくさんのオプションを受け取ります。デフォルトの値を
>それぞれ見つけるためにソースコードを見てみましょう:
>
>  * `graph`: グラフ全体のデフォルトオプション
>  * `node`: ノードのデフォルトオプション
>  * `edge`: エッジのデフォルトオプション
>  * `node.instance`: オブジェクトインスタンスによって直接定義された
>  サービス用のデフォルトオプション
>  * `node.definition`: サービス定義のインスタンスを通して
>  定義されたサービス用のデフォルトオプション
>  * `node.missing`: 見つからないサービス用のデフォルトオプション

新しいSymfony 2 Templating
Frameworkを使用する仮想的なCMSのグラフは次の通りです:

![Symfony 2 Templating Framework](http://fabien.potencier.org/media/articles/di/template.png)
