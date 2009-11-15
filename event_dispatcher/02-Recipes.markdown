レシピ
======

この章ではイベントディスパッチャの一般的な使い方の一覧を示します。

Event Dispatcherオブジェクトを単独で渡す
---------------------------------------

`sfEventDispatcher`クラスを見てみると、このクラスがSingleton(
staticな`getInstance()`メソッドが存在しない)として振る舞わないことがわかります。
単独のPHPリクエストで平行する複数のイベントディスパッチャを用意したいことがあるので
わざとこのようにしています。
しかしこのことはイベントに接続もしくはイベントを通知する
必要のあるオブジェクトにディスパッチャを渡す方法が必要であることも意味します。

ベストプラクティスはイベントディスパッチャオブジェクトをあなたのオブジェクトに注入することです。
別名でDependency Injectionといいます。

コンストラクタインジェクションを使うことができます:

    [php]
    class Foo
    {
      protected $dispatcher = null;

      public function __construct(sfEventDispatcher $dispatcher)
      {
        $this->dispatcher = $dispatcher;
      }
    }

もしくはセッターインジェクションも:

    [php]
    class Foo
    {
      protected $dispatcher = null;

      public function setEventDispatcher(sfEventDispatcher $dispatcher)
      {
        $this->dispatcher = $dispatcher;
      }
    }

2つの方法のどちらかを選ぶのは好みの問題でしかありません。
コンストラクションの時点でオブジェクトを十分に初期化されるので、
筆者はコンストラクタインジェクションを好む傾向にあります。
しかし長い依存関係のリストがある場合、とりわけオプションの依存関係に対して、
セッターインジェクションの方がうまくゆきます。

>**TIP**
>上記の2つの例で行ったようにDependency Injectionを使う場合、
>これらのオブジェクトを洗練された方法で管理するために
>Symfony Dependency Injectionコンテナを簡単に使うことができます。

メソッド呼び出しの後で何かを行う
------------------------------

メソッドが呼び出される直前もしくは直後に何かを行いたい場合、
メソッドの始めもしくは終わりにイベントをそれぞれ通知することができます:

    [php]
    class Foo
    {
      // ...

      public function send($foo, $bar)
      {
        // メソッドの前で何かを行う
        $event = new sfEvent($this, 'foo.do_before_send', array('foo' => $foo, 'bar' => $bar));
        $this->dispatcher->notify($event);

        // 実際のメソッドの実装は次の通り
        // $ret = ...;

        // メソッドの後で何かを行う
        $event = new sfEvent($this, 'foo.do_after_send', array('ret' => $ret));
        $this->dispatcher->notify($event);

        return $ret;
      }
    }

クラスにメソッドを追加する
-------------------------

複数のクラスが別のクラスにメソッドを追加できるようにするには、
次のように拡張したいクラスで`__call()`マジックメソッドを定義できます:

    [php]
    class Foo
    {
      // ...

      public function __call($method, $arguments)
      {
        // 'foo.method_is_not_found'という名前のイベントを作り
        // メソッドの名前とこのメソッドに渡す引数を渡す
        $event = new sfEvent($this, 'foo.method_is_not_found', array('method' => $method, 'arguments' => $arguments));

        // 1つのリスナーが$methodを実装できるようになるまですべてのリスナーを呼び出す
        $this->dispatcher->notifyUntil($event);

        // リスナーがイベントを処理できない？メソッドが存在しない
        if (!$event->isProcessed())
        {
          throw new sfException(sprintf('Call to undefined method %s::%s.', get_class($this), $method));
        }

        // リスナーの戻り値を返す
        return $event->getReturnValue();
      }
    }

それから、リスナーをホストするクラスを作成します:

    [php]
    class Bar
    {
      public function addBarMethodToFoo(sfEvent $event)
      {
        // 'bar'メソッドへの呼び出しにのみ応答したい場合
        if ('bar' != $event['method'])
        {
          // この未知のメソッドを考慮する別のリスナーに機会を提供する
          return false;
        }

        // subjectオブジェクト(fooインスタンス)
        $foo = $event->getSubject();

        // barメソッドの引数
        $arguments = $event['parameters'];

        // do something
        // ...

        // 戻り値をセットする
        $event->setReturnValue($someValue);

        // tell the world that you have processed the event
        return true;
      }
    }

最終的に、`Foo`クラスに新しい`bar`メソッドを追加します:

    [php]
    $dispatcher->connect('foo.method_is_not_found', array($bar, 'addBarMethodToFoo'));


引数を修正する
-------------

サードパーティのクラスがメソッドが実行される直前にメソッドに渡された引数を
修正できるようにしたい場合、メソッドの始めに`filter`
イベントを追加します:

    [php]
    class Foo
    {
      // ...

      public function render($template, $arguments = array())
      {
        // 引数をフィルタリングする
        $event = new sfEvent($this, 'foo.filter_arguments');
        $this->dispatcher->filter($event, $arguments);

        // フィルタリングされた引数を取得する
        $arguments = $event->getReturnValue();

        // ここからメソッドを始める
      }
    }

フィルタの例は次の通りです:

    [php]
    class Bar
    {
      public function filterFooArguments(sfEvent $event, $arguments)
      {
        $arguments['processed'] = true;

        return $arguments;
      }
    }
