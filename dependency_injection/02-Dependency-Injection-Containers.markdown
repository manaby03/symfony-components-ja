Dependency Injectionコンテナ
============================

Dependency Injection Containerの世界に飛び込む前に、
思い切った発言で始めましょう:

  *大抵の場合、Dependency Injectionから恩恵を得るために
  Dependency Injection Containerは必要ありません*.

たくさんの依存関係があるたくさんの異なるオブジェクトを管理する必要がある場合、
Dependency Injection Containerは本当に役立つでしょう(例えば
フレームワークを考えてみてください)。

最初の章の例を覚えているのであれば、`User`オブジェクトの作成において
先に`SessionStorage`オブジェクトの作成が必要でした。たいしたことではありませんが、それでも
必要なオブジェクトを作成する前に必要なすべての依存性を
知らなければなりません:

    [php]
    $storage = new SessionStorage('SESSION_ID');
    $user = new User($storage);

次の章では、とりわけDependency Injection Containerコンポーネント
について話します。
実装がSymfonyに縛られていないことを明らかにしたいので、この本ではZend
Frameworkの例も使います。

Zend Frameworkの`Mail`ライブラリは、Eメールの管理を楽にし、デフォルトではメール送信用にPHP
`mail()`関数を使いますが、これは本当に柔軟ではありません。
ありがたいことに、トランスポートオブジェクトを提供することで
この振る舞いを変更するのはきわめて簡単です。
次のコードスニペットはGmailのアカウントを使用してEメールを送信する
`Zend_Mail`オブジェクトを作成する方法を示します:

    [php]
    $transport = new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
      'auth'     => 'login',
      'username' => 'foo',
      'password' => 'bar',
      'ssl'      => 'ssl',
      'port'     => 465,
    ));

    $mailer = new Zend_Mail();
    $mailer->setDefaultTransport($transport);

>**NOTE**
>この本を短くするために、シンプルな例を使います。もちろん、
>これらのシンプルな例では、コンテナを用意する意味はありません。
>コンテナによって管理されるオブジェクトのコレクション
>のごく一部と考えてください。

Dependency Injection Containerはオブジェクトをインスタンス化して設定する方法を知っている
オブジェクトです。そしてこの仕事をできるようにするには、
コンストラクタの引数とオブジェクトの間のリレーションを知る必要があります。

コンテナをハードコードした`Zend_Mail`のシンプルな例は次の通りです:

    [php]
    class Container
    {
      public function getMailTransport()
      {
        return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
          'auth'     => 'login',
          'username' => 'foo',
          'password' => 'bar',
          'ssl'      => 'ssl',
          'port'     => 465,
        ));
      }

      public function getMailer()
      {
        $mailer = new Zend_Mail();
        $mailer->setDefaultTransport($this->getMailTransport());

        return $mailer;
      }
    }

コンテナクラスの使い方はシンプルです:

    [php]
    $container = new Container();
    $mailer = $container->getMailer();

コンテナを使うとき、メーラーオブジェクトを求めるだけで、
これを作る方法を知る必要はもはやありません; メーラーのインスタンスの作り方に
関するすべての知識はコンテナに埋め込まれます。
`getMailTransport()`の呼び出しのおかげで、
メーラートランスポートの依存性はコンテナによって自動的に注入されます。
コンテナのすべてなパワーは
この単純な呼び出しにあります！

しかし、賢明な読者の方は問題にお気づきのことでしょう。コンテナ自身が
すべてをハードコードしていることを！ですので、コンテナを本当に便利なものにするために
1つの上のステップに行って混合状態にパラメータを追加する必要があります:

    [php]
    class Container
    {
      protected $parameters = array();

      public function __construct(array $parameters = array())
      {
        $this->parameters = $parameters;
      }

      public function getMailTransport()
      {
        return new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
          'auth'     => 'login',
          'username' => $this->parameters['mailer.username'],
          'password' => $this->parameters['mailer.password'],
          'ssl'      => 'ssl',
          'port'     => 465,
        ));
      }

      public function getMailer()
      {
        $mailer = new Zend_Mail();
        $mailer->setDefaultTransport($this->getMailTransport());

        return $mailer;
      }
    }

コンテナのコンストラクタにいくつかのパラメータを渡すことで
Googleのユーザー名とパスワードを変更するのは簡単です:

    [php]
    $container = new Container(array(
      'mailer.username' => 'foo',
      'mailer.password' => 'bar',
    ));
    $mailer = $container->getMailer();

テスト用にメーラークラスを変更する必要がある場合、オブジェクトのクラス名をパラメータとして
渡すこともできます:

    [php]
    class Container
    {
      // ...

      public function getMailer()
      {
        $class = $this->parameters['mailer.class'];

        $mailer = new $class();
        $mailer->setDefaultTransport($this->getMailTransport());

        return $mailer;
      }
    }

    $container = new Container(array(
      'mailer.username' => 'foo',
      'mailer.password' => 'bar',
      'mailer.class'    => 'Zend_Mail',
    ));
    $mailer = $container->getMailer();

大事なことですが、メーラーを取得するたびに、新しいインスタンスは必要ありません。
ですので、同じオブジェクトを常に返すようにコンテナを
変更できます:

    [php]
    class Container
    {
      static protected $shared = array();

      // ...

      public function getMailer()
      {
        if (isset(self::$shared['mailer']))
        {
          return self::$shared['mailer'];
        }

        $class = $this->parameters['mailer.class'];

        $mailer = new $class();
        $mailer->setDefaultTransport($this->getMailTransport());

        return self::$shared['mailer'] = $mailer;
      }
    }

staticな`$shared`プロパティを導入することで、
`getMailer()`メソッドを呼び出すたびに、最初の呼び出し用に作成されたオブジェクトが返されます。

これによってDependency Injection Containerによって実装される必要のある基本機能を完成させます。
Dependency Injection Containerはインスタンスから設定までオブジェクトを管理します。
オブジェクト自身はコンテナによって管理されていることおよびコンテナを知りません。
これがコンテナがPHPオブジェクトを管理できる理由です。
依存関係用にオブジェクトがDependency Injectionを使う場合がベターですが、
必須ではありません。

もちろん、コンテナクラスの作成と維持を手作業で行うのはすぐに悪夢になります。
しかしコンテナを便利するための要件は最小限なので、
これを実装するのは簡単です。
