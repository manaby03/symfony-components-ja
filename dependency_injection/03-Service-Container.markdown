Symfony Service Container
=========================

これまで、一般的な概念を説明してきました。
今後の章で話す実装の理解を深めるために
最初の章は大切でした。
Symfony Service Containerコンポーネントの実装に飛び込みましょう。

Symfony Dependency Injection Containerは
`sfServiceContainer`という名前のクラスによって管理されます。
これは以前の記事で話した基本機能を実装する
とても軽量なクラスです。

Symfonyの説明では、*サービス(service)とはコンテナによって管理されるオブジェクト*です。
前の章の`Zend_Mail`の例では、2つのサービス: `mailer`
サービスと`mail_transport`サービスがあります:

    [php]
    class Container
    {
      static protected $shared = array();

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

Symfonyの`sfServiceContainer`クラスを継承する`Container`を作成すれば、
少しコードを簡略化できます:

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

これは多くありませんが、強力でクリーンなインターフェイスを提供してくれます。
行った主要な変更は次の通りです:

 * メソッドの名前の接尾辞として`Service`を追加しました。慣習によって、
   サービスは`get`という接頭辞と`Service`という接辞尾を持つメソッド
   によって定義されます。それぞれの識別子は一意的な識別子を持ち、
   接頭辞と接辞尾を伴わないアンダースコアバージョンのメソッド名です。
   `getMailTransportService()`メソッドを定義すれば、
   `mail_transport`という名前のメソッドが定義されます。

 * メソッドの可視性はprotectedです。
   コンテナからサービスを読み取る方法をすぐに見ることになります。

 * パラメータの値を取得するためにコンテナを配列として使うことができます
   (`$this['mailer.class']`)。

>**TIP**
>サービスの識別子は一意的で、文字、
>数値、アンダースコア、とドットで構成されなければなりません。ドットは
>コンテナの範囲で"名前空間"を定義するのに便利です(例えば`mail.mailer`と
>`mail.transport`).

新しいコンテナクラスの使い方を見てみましょう:

    [php]
    require_once '/PATH/TO/sfServiceContainerAutoloader.php';
    sfServiceContainerAutoloader::register();

    $sc = new Container(array(
      'mailer.username' => 'foo',
      'mailer.password' => 'bar',
      'mailer.class'    => 'Zend_Mail',
    ));

    $mailer = $sc->mailer;

`Container`クラスは`sfServiceContainer`クラスを継承するので、
よりクリーンなインターフェイスを楽しむことができます:

  * サービスは一律のインターフェイスでアクセスできます:

        [php]
        if ($sc->hasService('mailer'))
        {
          $mailer = $sc->getService('mailer');
        }

        $sc->setService('mailer', $mailer);

  * ショートカットとして、サービスはクラスプロパティの表記法を通しても
    アクセスできます:

        [php]
        if (isset($sc->mailer))
        {
          $mailer = $sc->mailer;
        }

        $sc->mailer = $mailer;

  * パラメータは一律のインターフェイスを通してアクセスできます:

        [php]
        if (!$sc->hasParameter('mailer_class'))
        {
          $sc->setParameter('mailer_class', 'Zend_Mail');
        }

        echo $sc->getParameter('mailer_class');

        // コンテナのすべてのパラメータをオーバーライドする
        $sc->setParameters($parameters);

        // パラメータを追加する
        $sc->addParameters($parameters);

  * ショートカットとして、コンテナを利用して配列のように
    パラメータにアクセスすることもできます:

        [php]
        if (!isset($sc['mailer.class']))
        {
          $sc['mailer.class'] = 'Zend_Mail';
        }

        $mailerClass = $sc['mailer.class'];

  * コンテナのすべてのサービスをイテレートすることもできます:

        [php]
        foreach ($sc as $id => $service)
        {
          echo sprintf("Service %s is an instance of %s.\n", $id, get_class($service));
        }

管理するサービスがごくわずかであるとき`sfServiceContainer`クラスがとても便利です; 
自分自身でたくさんの基礎作業をして、たくさんのコードが重複する場合もです。
しかし、管理するサービスの数が少量ではなくなった場合
サービスを記述するのによりベターな方法が必要です。

これが、大抵の場合、`sfServiceContainer`クラスを直接使わない理由です。
Symfony Dependency Injection Containerの実装の土台なので
説明にいくぶんか時間をかけるほど大切です。
