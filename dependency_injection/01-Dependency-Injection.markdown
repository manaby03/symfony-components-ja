Dependency Injectionとは？
=========================

PHPの世界ではDependency Injectionはまだ広まった概念ではありませんので、
この章では平易なPHPコードのみを使いDependency Injectionを紹介します。

いくつかの具体的な実例を通して、Dependency Injectionが解決しようとする問題と
開発者にもたらす恩恵を
あきらかにします。

>**TIP**
>Dependency Injectionに慣れているのであれば、
>この章をスキップしても支障なく次の章を読み始めることができます。

Dependency Injectionはもっともシンプルなデザインパターンであり、
おそらくDependency Injectionを既に使っているでしょう。しかしながら、
うまく説明するのがもっとも困難なものの1つです。これはおそらく
Dependency Injectionの紹介で使われる無意味な例が一因と考えられます。
この章では、PHPの世界で合うベターな実例を見出すように心掛けました。
PHPは主にWeb開発で使われる言語なので、
Webの例を使うことにします。

問題
----

HTTPプロトコルがステートレスであることを克服するために、Webアプリケーションは
Webリクエストの間にユーザーの情報を保存する方法を提供します。これはcookieを使う、
もしくはよりベターなPHPのセッションメカニズムを使うことで
簡単に実現されます:

    [php]
    $_SESSION['language'] = 'fr';

上記のコードは`language`セッション変数でユーザー言語を保存します。
ですので、同じユーザーのその後のすべてのリクエストに関して、`language`は
グローバルな`$_SESSION`配列で利用可能です:

    [php]
    $user_language = $_SESSION['language'];

Dependency Injectionはオブジェクト指向の世界でのみ意味をなすので、
PHPのセッションメカニズムをラップする`SessionStorage`クラスを用意してみましょう:

    [php]
    class SessionStorage
    {
      function __construct($cookieName = 'PHP_SESS_ID')
      {
        session_name($cookieName);
        session_start();
      }

      function set($key, $value)
      {
        $_SESSION[$key] = $value;
      }

      function get($key)
      {
        return $_SESSION[$key];
      }

      // ...
    }

... そして`User`クラスはユーザーの高度なレベルの素晴らしいインターフェイスを提供します:

    [php]
    class User
    {
      protected $storage;

      function __construct()
      {
        $this->storage = new SessionStorage();
      }

      function setLanguage($language)
      {
        $this->storage->set('language', $language);
      }

      function getLanguage()
      {
        return $this->storage->get('language');
      }

      // ...
    }

これらのクラスは十分にシンプルで`User`クラスを使うのも
より楽です:

    [php]
    $user = new User();
    $user->setLanguage('fr');
    $language = $user->getLanguage();

すべては良い状態です...より柔軟性を求めるまでは。例えば
セッションcookieの名前を変更したい場合は？いくつかのありうる
方法は次の通りです:

  * `SessionStorage`コンストラクタで`User`クラスのセッションを
    決め打ちする:

        [php]
        class User
        {
          function __construct()
          {
            $this->storage = new SessionStorage('SESSION_ID');
          }

          // ...
        }

  * `User`クラス外部の定数を定義する:

        [php]
        class User
        {
          function __construct()
          {
            $this->storage = new SessionStorage(STORAGE_SESSION_NAME);
          }

          // ...
        }

        define('STORAGE_SESSION_NAME', 'SESSION_ID');

  * `User`コンストラクタの引数としてセッションの名前を追加する:

        [php]
        class User
        {
          function __construct($sessionName)
          {
            $this->storage = new SessionStorage($sessionName);
          }

          // ...
        }

        $user = new User('SESSION_ID');

  * ストレージクラス用のオプションの配列を追加する:

        [php]
        class User
        {
          function __construct($storageOptions)
          {
            $this->storage = new SessionStorage($storageOptions['session_name']);
          }

          // ...
        }

        $user = new User(array('session_name' => 'SESSION_ID'));

これらすべての代替方法はきわめて悪いものです。`User`クラスでセッション名を決め打ちしても
本当に問題を解決したことにはなりません。後で気が変わった際に
`User`クラスを簡単に変更できないからです。定数を使うのも悪いアイディアです。
`User`クラスが設定される定数に依存するからです。
セッション名を引数もしくはオプションの配列として渡す方法はおそらく最良のソリューションですが、
まだ不吉な臭いがします。この方法では`User`コンストラクタの引数をオブジェクトに関係ないもので
ごちゃごちゃにしてしまうからです。

しかしまだ簡単には解決できない別の問題があります: `SessionStorage`クラスを変更する方法は？
例えば、テスト作業を楽にするためにこれをモックオブジェクトに置き換える、
もしくはデータベースのテーブルもしくはメモリにセッションを保存するためなどです。
これは現在の実装では
`User`クラスを変更する以外実現不可能です。

ソリューション
-------------

Dependency Injectionに入ります。*`User`クラス内部で`SessionStorage`オブジェクトを
作成する代わりに、`SessionStorage`オブジェクトを
`User`オブジェクトのコンストラクタの引数として注入してみましょう*:

    [php]
    class User
    {
      function __construct($storage)
      {
        $this->storage = $storage;
      }

      // ...
    }

これがDependency Injectionです。これ以上のことは何もありません！
最初に`SessionStorage`オブジェクトを作る必要があるので
`User`クラスを使うには少し多くのことが関わるようになりました:

    [php]
    $storage = new SessionStorage('SESSION_ID');
    $user = new User($storage);

これで、セッションストレージオブジェクトを設定するのはとてもシンプルで、
セッションストレージクラスを置き換えるのも非常に簡単です。
関心の分離のおかげで`User`クラスを変更せずにすべてが可能です。

[Pico Container website](http://www.picocontainer.org/injection.html)
はDependency Injectionを次のように説明しています:

  "Dependency Injectionにおいてコンポーネントに対して
  コンストラクタ、メソッドを通してもしくは直接フィールドに依存性が渡されます。"

>**TIP**
>他のデザインパターンと同じように、Dependency Injection
>にはアンチパターンもあります。
>[Pico ContainerのWebサイト](http://www.picocontainer.org/)
>はそれらの一部を説明しています。

Dependency Injectionはコンストラクタインジェクションだけに限られるわけではありません:

  * コンストラクタインジェクション:

        [php]
        class User
        {
          function __construct($storage)
          {
            $this->storage = $storage;
          }

          // ...
        }

  * セッターインジェクション:

        [php]
        class User
        {
          function setSessionStorage($storage)
          {
            $this->storage = $storage;
          }

          // ...
        }

  * プロパティインジェクション:

        [php]
        class User
        {
          public $sessionStorage;
        }

        $user->sessionStorage = $storage;

経験則として、このチュートリアルの例のようにコンストラクタインジェクションは必須の依存にベストです,
キャッシュオブジェクトのように、
セッターインジェクションはオプションの依存にベストです。

こんにちでは、疎結合されたコンポーネントのセットを提供するために
大抵のモダンなPHPフレームワークはDependency Injectionを多いに利用します:

    [php]
    // symfony: コンストラクタインジェクションの例
    $dispatcher = new sfEventDispatcher();
    $storage = new sfMySQLSessionStorage(array('database' => 'session', 'db_table' => 'session'));
    $user = new sfUser($dispatcher, $storage, array('default_culture' => 'en'));

    // Zend Framework: セッターインジェクションの例
    $transport = new Zend_Mail_Transport_Smtp('smtp.gmail.com', array(
      'auth'     => 'login',
      'username' => 'foo',
      'password' => 'bar',
      'ssl'      => 'ssl',
      'port'     => 465,
    ));

    $mailer = new Zend_Mail();
    $mailer->setDefaultTransport($transport);
