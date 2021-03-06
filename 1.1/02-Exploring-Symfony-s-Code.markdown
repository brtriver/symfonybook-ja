第2章 - symfonyのコードを探求する
=================================

一見すると、symfony駆動のアプリケーションの背後に存在する(巨大で複雑な)コードにやる気をくじかれてしまうかもしれません。symfonyは多くのディレクトリとスクリプトで構成され、PHPクラスファイルとHTMLファイルが混在しており、ファイルのなかには両方の言語が含まれるものもあります。クラスのリファレンスがアプリケーションのフォルダーのなかでは見つからず、ディレクトリの深さが6レベルに達していることも見ることになります。しかしながら、ひとたびこの見た目の複雑のすべての背後にある理由を理解したのであれば、symfonyのアプリケーション構造がごく自然であり別のものに置き換えようとは思わなくなるでしょう。この章ではおびえた感情を説明でとり除きます。

MVCパターン
-----------

symfonyは3つのレベルから構成される、MVCアーキテクチャ(Model-View-Controller architecture)として知られるWebの古典的なデザインパターンに基づいています:

  * モデル(Model)はアプリケーションが影響を与える情報、ビジネスロジックを表します。
  * ビュー(View)はモデルをユーザーとのインタラクションに最適なWebページにレンダリングします。
  * コントローラー(Controller)はユーザーのアクションに対応しモデルもしくはビューの変更を適切に行います。

図2-1はMVCパターンを図示しています。

MVCアーキテクチャはビジネスロジック(モデル)とプレゼンテーションを分離するので、結果として維持管理しやすくなります。たとえば、アプリケーションが標準的なWebブラウザーと携帯端末の両方で動く場合、新しいビューだけが必要です; オリジナルのコントローラーとモデルはそのままにできます。コントローラーはモデルとビューからリクエスト(HTTP、コンソールモード、メールなど)のために使われるプロトコルの詳細を隠すための助けを行います。そしてモデルはデータのロジックの抽象化を行い、ビューとアクションを、たとえば、アプリケーションによって使われるデータベースの種類から独立したものにします。

図2-1 - MVCパターン

![MVCパターン](http://github.com/masakielastic/symfonybook-ja/raw/master/images/F0201.png "MVCパターン")

### MVC階層化

MVCの利点を理解するために、PHPの基本的なアプリケーションをMVCアーキテクチャのアプリケーションに改造する方法を見てみましょう。blogのアプリケーションにおける投稿の一覧を表示する機能が完璧な例です。

#### ベタ書きのプログラミング

ベタ書きのPHPファイル(flat PHP file)において、データベースのエントリーの一覧表示機能はリスト2-1のようなスクリプトになります。

リスト2-1 - ベタ書きのスクリプト

    [php]
    <?php

    //データベースに接続、選択する
    $link = mysql_connect('localhost', 'myuser', 'mypassword');
    mysql_select_db('blog_db', $link);

    // SQLクエリを実行する
    $result = mysql_query('SELECT date, title FROM post', $link);

    ?>

    <html>
      <head>
        <title>投稿の一覧</title>
      </head>
      <body>
       <h1>投稿の一覧</h1>
       <table>
         <tr><th>日付</th><th>タイトル</th></tr>
    <?php
    // 結果をHTML形式で出力する
    while ($row = mysql_fetch_array($result, MYSQL_ASSOC))
    {
    echo "\t<tr>\n";
    printf("\t\t<td> %s </td>\n", $row['date']);
    printf("\t\t<td> %s </td>\n", $row['title']);
    echo "\t</tr>\n";
    }
    ?>
        </table>
      </body>
    </html>

    <?php

    // 接続を閉じる
    mysql_close($link);

    ?>

素早く書けて、速く実行できますが、維持することは不可能です。このコードの重大な問題はつぎのとおりです:

  * エラーのチェック機能が存在しない(データベースの接続が失敗したらどうする？)
  * HTMLとPHPのコードが混在している。
  * コードがMySQLデータベースに直結している

#### プレゼンテーションを分離する

リスト2-1の`echo`と`printf`の呼び出しがコードを読みづらくしています。現在の構文ではプレゼンテーションを強化するためにHTMLコードを修正する作業は骨が折れます。ですのでコードを2つの部分に分割する方法を採用することにします。最初に、リスト2-2で示されるような、ビジネスロジックをともなう純粋なPHPコードをコントローラースクリプトに移動させます。

リスト2-2 - コントローラー部分を表す`index.php`

    [php]
     // データベースに接続、選択する
     $link = mysql_connect('localhost', 'myuser', 'mypassword');
     mysql_select_db('blog_db', $link);

     // SQLクエリを実行する
     $result = mysql_query('SELECT date, title FROM post', $link);

     // ビューのために配列を充填する
     $posts = array();
     while ($row = mysql_fetch_array($result, MYSQL_ASSOC))
     {
        $posts[] = $row;
     }

     // 接続を閉じる
     mysql_close($link);

     // ビューのスクリプトを読み込む
     require('view.php');

リスト2-3で示されるように、テンプレートのようなPHP構文を含むHTMLコードはビューのスクリプトに保存します。

リスト2-3 - ビューの部分を表す`view.php`

    [php]
    <html>
      <head>
        <title>投稿の一覧</title>
      </head>
      <body>
        <h1>投稿の一覧</h1>
        <table>
          <tr><th>日付</th><th>タイトル</th></tr>
        <?php foreach ($posts as $post): ?>
          <tr>
            <td><?php echo $post['date'] ?></td>
            <td><?php echo $post['title'] ?></td>
          </tr>
        <?php endforeach; ?>
        </table>
      </body>
    </html>

ビューが十分に明解であるのか判断するためのよい経験則は、PHPの知識のないHTMLデザイナーが理解できるように最小量のPHPコードだけが含まれるかです。ビューでもっともよく使われるステートメントは`echo`、`if/endif`、`foreach/endforeach`でだけです。また、HTMLタグを`echo`するPHPコードが存在してはなりません。

すべてのロジックをコントローラースクリプトに移動させます。コントローラー内部にはHTMLが存在しない純粋なPHPコードだけが含まれます。当然のことながら、同じコントローラーが全体的に異なるプレゼンテーション、たとえばPDFファイルもしくはXML構造のなかで再利用できるようにすることを想像すべきです。

#### データの操作機能を分離する

コントローラースクリプトの大部分はデータの操作専用です。しかし、別のコントローラーのために投稿の一覧表示機能が必要な場合、たとえばblog投稿のRSSフィードを出力するには？コードの重複を避けるために、すべてのデータベースのクエリを一ヶ所に保存したいのであればどうしますか？ `post`テーブルが`weblog_post`にリネームされたのでデータモデルを変更することを決定したらどうしますか？MySQLの代わりにPostgreSQLに切り替えたい場合はどうしますか？これらすべてを実現するには、リスト2-4で示されるように、コントローラーからデータ操作のコードを削除し、モデルと呼ばれる、別のスクリプトに設置します。

リスト2-4 - モデルの部分を表す`model.php`

    [php]
    function getAllPosts()
    {
      // データベースに接続し、選択する
      $link = mysql_connect('localhost', 'myuser', 'mypassword');
      mysql_select_db('blog_db', $link);

      // SQLクエリを実行する
      $result = mysql_query('SELECT date, title FROM post', $link);

      // 配列を充填する
      $posts = array();
      while ($row = mysql_fetch_array($result, MYSQL_ASSOC))
      {
         $posts[] = $row;
      }

      // 接続を閉じる
      mysql_close($link);

      return $posts;
    }

改訂したコントローラーはリスト2-5で示されています。

リスト2-5 - 改訂したコントローラーの部分を表す`index.php`

    [php]
    // モデルのコードを読み込む
    require_once('model.php');

    // 投稿のリストを読みとる
    $posts = getAllPosts();

    // ビューをrequireする
    require('view.php');

これでコントローラーは読みやすくなりました。コントローラーの唯一のタスクはモデルからデータを取得してビューに渡すことです。より複雑なアプリケーションにおいて、コントローラーはリクエスト、ユーザーセッション、認証なども扱うことができます。モデルの関数の名前を明確にすることでコントローラーコードのコメントは不要になります。

モデルスクリプトはデータアクセス専用で、それに応じて編成されます。(リクエストパラメーターのような)データレイヤーに依存しないすべてのパラメーターはコントローラーから渡されなければならず、またモデルによって直接アクセスされないようにしなければなりません。モデルの関数は別のコントローラーで簡単に再利用できません。

### MVCを超えるレイヤーの分離

MVCアーキテクチャの原則はコードの性質にしたがってコードを3つのレイヤーに分離することです。データロジックのコードはモデルの範囲に、プレゼンテーションのコードはビューの範囲に、アプリケーションのロジックはコントローラーの範囲に設置されます。

デザインパターンを追加することでコーディング作業がより簡単になります。モデル、ビュー、コントローラーレイヤーをさらに細かく分割できます。

#### データベースの抽象化

モデルレイヤーはデータベースアクセスレイヤーとデータベース抽象化レイヤーに分割できます。この方法では、データアクセス関数はデータベースに依存するクエリのステートメントを使いませんが、ほかの関数がそれら自身でクエリを行います。もしあとでデータベースシステムを変更する場合、データベース抽象化レイヤーの更新だけが必要になります。

リスト2-6はデータベース抽象化レイヤーのサンプルで、続くリスト2-7はMySQL固有のデータアクセスレイヤーの例です。

リスト2-6 - モデル内の抽象化データベース部分

    [php]
    <?php

    function open_connection($host, $user, $password)
    {
      return mysql_connect($host, $user, $password);
    }

    function close_connection($link)
    {
      mysql_close($link);
    }

    function query_database($query, $database, $link)
    {
      mysql_select_db($database, $link);

      return mysql_query($query, $link);
    }

    function fetch_results($result)
    {
      return mysql_fetch_array($result, MYSQL_ASSOC);
    }

リスト2-7 - モデル内のデータアクセス部分

    [php]
    function getAllPosts()
    {
      // データベースに接続する
      $link = open_connection('localhost', 'myuser', 'mypassword');

      // SQLクエリを実行する
      $result = query_database('SELECT date, title FROM post', 'blog_db', $link);

      // 配列を充填する
      $posts = array();
      while ($row = fetch_results($result))
      {
         $posts[] = $row;
      }

      // 接続を閉じる
      close_connection($link);

      return $posts;
    }

(リスト2-7で)データベース依存の関数がデータベースアクセスレイヤーで見つからないことを確認できます。これによって関数はデータベースから独立したものになります。これに加えて、データベースの抽象化レイヤーで作成された関数はデータベースにアクセスする必要があるほかの多くのモデル関数で再利用できます。

>**NOTE**
>リスト2-6とリスト2-7の例はまだ十分に満足のゆくものではなく、データベースを完全に抽象化するための作業が残されています(データベースから独立したクエリビルダーを通してSQLコードを抽象化すること、すべての関数をクラスに移動させることなど)。しかしこの本の目的は手作業ですべてのコードを書く方法を示すことではなく、8章でsymfonyがネイティブですべての抽象化を上手に行うことを見ることになります。

#### ビューの要素

ビューレイヤーもコード分離から恩恵をこうむります。Webページがアプリケーション全体で永続する要素を持つことはよくあります: たとえばページヘッダー、グラフィカルなレイアウト、フッター、グローバルナビゲーションなどです。ページ内部の部分のみが変わります。ビューがレイアウトとテンプレートに分離される理由はそういうわけです。通常、レイアウトはアプリケーションもしくはページのグループに対してグローバルです。テンプレートにはコントローラーによって利用可能になる変数のみが含まれます。これらのコンポーネントを連携させるために少々のロジックが必要で、このビューロジックのレイヤーはビューの名前を保有します。リスト2-8、2-9、2-10で示されるように、これらの原則にしたがって、リスト2-3のビューの部分を3つに分割できます。

リスト2-8 - ビューのテンプレート部分(`mytemplate.php`)

    [php]
    <h1>投稿の一覧</h1>
    <table>
    <tr><th>日付</th><th>タイトル</th></tr>
    <?php foreach ($posts as $post): ?>
      <tr>
        <td><?php echo $post['date'] ?></td>
        <td><?php echo $post['title'] ?></td>
      </tr>
    <?php endforeach; ?>
    </table>

リスト2-9 - ビューのロジック部分

    [php]
    <?php

    $title = '投稿の一覧';
    $posts = getAllPosts();

リスト2-10 - ビューのレイアウト部分

    [php]
    <html>
      <head>
        <title><?php echo $title ?></title>
      </head>
      <body>
        <?php include('mytemplate.php'); ?>
      </body>
    </html>

#### アクションとフロントコントローラー

前の例ではコントローラーはあまり多くのことを行いませんでしたが、実際のWebアプリケーションでは、コントローラーは多くの仕事を担います。この仕事の重要な部分はアプリケーションのすべてのコントローラーに共通です。共通のタスクにはリクエスト処理、セキュリティ処理、アプリケーション設定の読み込み、および似たような雑用作業が含まれます。コントローラーがフロントコントローラーとアクションによく分割されるのはそういうわけです。フロントコントローラーはアプリケーション全体で唯一のもので、アクションは1つのページに固有のコントローラーコードのみを格納します。

フロントコントローラーの大きな利点の一つはアプリケーション全体で唯一のエントリーポイント(入り口)が提供されることです。アプリケーションを閉鎖することを決定した場合、必要な作業はフロントコントローラーのスクリプトを編集することだけです。フロントコントローラーなしのアプリケーションでは、それぞれのコントローラーをオフに切り替えることが必要です。

#### オブジェクト指向

以前のすべての例では手続き型のプログラミングの方法を利用しました。現代のプログラミング言語の手法であるオブジェクト指向プログラミング(OOP)の機能によってプログラミングはさらに簡単なものになります。なぜなら、オブジェクトがロジックをカプセル化し、別のオブジェクトから継承し、クリーンな名前の規約を提供できるからです。

MVCアーキテクチャをオブジェクト指向ではない言語で実装すると名前空間の衝突とコードの重複問題が発生し、全体のコードが読みにくくなります。

オブジェクト指向によって開発者はビューオブジェクト、コントローラーオブジェクト、モデルオブジェクトといったものを扱うことが可能で、以前の例のすべての関数はメソッドに翻訳されます。オブジェクト指向はMVCアーキテクチャのための必須の方法です。

>**TIP**
>オブジェクト指向の文脈でWebアプリケーションのためのデザインパターンについてもっと学びたい場合、Martin Fowler(マーチン・ファウラー)が書いたPatterns of Enterprise Application Architecture(Addison-Wesley, ISBN: 0-32112-742-0 (邦訳は翔泳社より刊行された「エンタープライズ アプリケーションアーキテクチャパターン」)をお読みください。Fowler本のコードの例はJavaもしくはC#ですが、PHP開発者にも読みやすいです。

### symfonyによるMVCの実装方法

少しお待ちください。blogで投稿の一覧を表示する単独のページに対して、必要なコンポーネントはいくつ存在するでしょうか？ 図2-2で説明されているように、つぎのような部品があります:

  * モデルレイヤー 
    * データベースの抽象化 
    * データアクセス 
  * ビューレイヤー 
    * ビュー 
    * テンプレート 
    * レイアウト 
  * コントローラーレイヤー 
    * フロントコントローラー 
    * アクション

7つのスクリプト、全体の多くは新しいページを作るたびに開いて修正するファイルです！しかしながら、symfonyはこれらの作業を簡単にします。MVCアーキテクチャを最大限活用する一方で、symfonyはこのアーキテクチャを早くて苦痛のともなわないアプリケーションの開発方法で実装します。

最初に、アプリケーションのすべてのアクションでフロントコントローラーとレイアウトは共通です。複数のコントローラーとレイアウトを用意するのは可能ですが、必要なものはそれぞれ1つだけです。フロントコントローラーは純粋なMVCロジックのコンポーネントで、symfonyが生成してくれるので書く必要はありません。

ほかのよいお知らせは、データ構造に基づいて、モデルレイヤーのクラスも自動的に生成されることです。これはPropelライブラリの仕事で、クラスのスケルトンとコードの生成方法を提供します。Propelは外部キー制約もしくはデータフィールドを見つけると、データ操作を簡単にする特別なアクセサー(accessor)メソッドとミューテーター(mutator)メソッドを提供します。そしてデータベースの抽象化は全体的に開発者には見えません。なぜなら、Creoleと呼ばれるほかのコンポーネントによって処理されるからです。データベースエンジンを変更することを決めた場合、書き直す必要のあるコードはゼロです。設定パラメーターを1つ変更することだけが必要です。

そして最後のお知らせは、プログラミングを行わずに、ビューロジックをシンプルな設定ファイルに簡単に翻訳できることです。

図2-2 - symfonyのワークフロー

![symfonyのワークフロー](http://github.com/masakielastic/symfonybook-ja/raw/master/images/F0202.png "symfonyのワークフロー")

リスト2-11、2-12、2-13で示されるように、このことは私たちの例で説明された投稿の一覧機能をsymfonyで動かすには3つのファイルだけが必要になることを意味します。

リスト2-11 - `list`アクション(`myproject/apps/myapp/modules/weblog/actions/actions.class.php`)

    [php]
    <?php

    class weblogActions extends sfActions
    {
      public function executeList()
      {
        $this->posts = PostPeer::doSelect(new Criteria());
      }
    }

リスト2-12 - `list`テンプレート(`myproject/apps/myapp/modules/weblog/templates/listSuccess.php`)

    [php]
    <?php slot('title', '投稿の一覧') ?>

    <h1>投稿の一覧</h1>
    <table>
    <tr><th>日付</th><th>タイトル</th></tr>
    <?php foreach ($posts as $post): ?>
      <tr>
        <td><?php echo $post->getDate() ?></td>
        <td><?php echo $post->getTitle() ?></td>
      </tr>
    <?php endforeach; ?>
    </table>

加えて、リスト2-13で示されるように、まだ1つのレイアウトを定義する必要がありますが、これは何度も再利用されます。

リスト2-13 - レイアウト(`myproject/apps/myapp/templates/layout.php`)

    [php]
    <html>
      <head>
        <title><?php include_slot('title') ?></title>
      </head>
      <body>
        <?php echo $sf_content ?>
      </body>
    </html>

これが本当に必要なコードのすべてです。これが以前のリスト2-1で示されたベタ書きのスクリプトとまったく同じページを表示するために必要で正しいコードです。(すべてのコンポーネントを連携させる)残りの部分はsymfonyによって処理されます。行数を数えると、ベタ書きのファイルで書くよりもsymfonyによるMVCアーキテクチャで投稿の一覧機能を作成するほうが時間がかからないもしくはコーディング作業が少なくてすむことが理解できます。そのことだけでなく、symfonyは非常に大きな利点、とりわけきれいなコードの編成、再利用性、柔軟性を提供し、そしてコーディング作業がずっと楽しくなります。おまけに、XHTMLの適合性(conformance)、デバッグ機能、簡単な設定、データベースの抽象化、スマートURLのルーティング、複数の環境、そしてより多くの開発ツールが手に入ります。

### symfonyのコアクラス

symfonyのMVCアーキテクチャはこの本で何度も見ることになるいくつかのクラスを利用します:

  * `sfController`はコントローラークラスです。リクエストを解読してアクションに渡します。 
  * `sfRequest`はすべてのリクエスト要素(パラメーター、Cookie、ヘッダーなど)を保存します。 
  * `sfResponse`はレスポンスヘッダーと内容を格納します。これは最終的にHTMLレスポンスに変換されユーザーに送信されるオブジェクトです。 
  * (`sfContext::getInstance()`によって読みとりされる)コンテキスト(context)はすべてのコアオブジェクトと現在の設定への参照を保存します: どこからでもアクセス可能です。

6章でこれらのオブジェクトを詳しく学びます。

ご覧のとおり、symfonyのすべてのクラスは`sf`をプレフィックスとして使います。テンプレートのコア変数も同様です。これによってあなた独自のクラスと変数の名前の衝突を回避することが可能で、symfonyのコアクラスは親しみやすく認識しやすいものになります。

>**NOTE**
>symfonyで利用されているコーディング規約のなかで、UpperCamelCase(アッパーキャメルケース)はクラスと変数の命名に関する規約です。ただし、2つの例外が存在します: symfonyのコアクラスは小文字の`sf`で始まり、テンプレートで見つかる変数はアンダースコア(`_`)で区切られた構文を使います。

コードの編成
------------

これであなたはsymfonyアプリケーションの異なるコンポーネントを理解しましたが、これらがどのように編成されるのかおそらく疑問に思っているでしょう。symfonyはコードを1つのプロジェクト構造に編成して、プロジェクトファイルを1つの標準ツリー構造に設置します。

### プロジェクトの構造: アプリケーション、モデルとアクション

symfonyにおいて、プロジェクト(project)とは、同じオブジェクトモデルを共有し、指定されたドメイン名のもとで利用できるサービスとオペレーションの集合です。

プロジェクト内部では、オペレーションはアプリケーションに論理的に分類されます。通常は、アプリケーションは同じプロジェクトのほかのアプリケーションから独立して動作できます。多くの場合、プロジェクトには2つのアプリケーションが格納されます: 1つはフロントオフィス用の、もう1つはバックオフィス用のアプリケーションです。これらは同じデータベースを共有します。しかし、それぞれが異なるアプリケーションである小さなサイトを多く含む1つのプロジェクトを用意することもできます。アプリケーション間のハイパーリンクは絶対パスでなければならないことに注意してください。

それぞれのアプリケーションは1つもしくは複数のモジュールの一式(セット)です。通常、1つのモジュール(module)は同じ目的を持つ1つのページもしくはページのグループを表します。たとえば、`home`、`articles`、`help`、`shoppingCart`、`account`モジュールなどを持つことができます

モジュールはアクションを格納します。アクション(action)は1つのモジュールのなかで行うことができるさまざまな行動を表します。たとえば、`shoppingCart`モジュールは`add`(追加する)、`show`(表示する)、`update`(更新する)アクションを持つことができます。一般的にアクションは動詞として記述できます。アクションの扱いかたは古典的なアプリケーションのページの扱いかたとほとんど同じですが、2つのアクションの結果が同じページになることもあります(たとえば、blogでコメントを投稿画面に追加すると新しいコメントが含まれる投稿画面が再表示される)。

>**TIP**
>プロジェクトを始めるのに際してアクションが多くの内容を表しすぎる場合、すべてのアクションを単独のモジュールにまとめるのはとても簡単です。アプリケーションがより複雑になるとき、複数のアクションを異なるモジュールに分類する時期です。1章で説明したように、(ふるまいを保ちながら)構造もしくは読みやすさを改善するためにコードを書き直す作業はリファクタリングと呼ばれ、RAD(ラピッドアプリケーション開発)の原則を適用するときに、この作業を多く行います。

図2-3はプロジェクト/アプリケーション/モジュール/アクション構造のblogプロジェクトのコードの編成のサンプルを示します。しかしプロジェクトの実際のファイルツリー構造は図で示されたセットアップと異なることをご了承してください。

図2-3 - コードの編成の例

![コードの編成の例](http://github.com/masakielastic/symfonybook-ja/raw/master/images/F0203.png "コードの編成の例")

### ファイルのツリー構造

一般的に、すべてのWebプロジェクトはつぎのような同じ種類の内容を共有します:

  * データベースのファイル。MySQL、PostgreSQLなど
  * 静的なファイル(HTML、画像、JavaScriptファイル、スタイルシートなど)
  * サイトのユーザーと管理者がアップロードしたファイル
  * PHPクラスとライブラリ 
  * 外部ライブラリ(サードパーティのスクリプト)
  * バッチファイル(コマンドラインもしくはcron tableで起動するスクリプト) 
  * ログファイル(アプリケーションかつ/もしくはサーバーによって書かれるトレース)
  * 設定ファイル

symfonyはアーキテクチャ上の選択(MVCパターンとプロジェクト/アプリケーション/モジュールの分類)と調和した、論理的な方法でこれらすべての内容を編成するファイルの標準ツリー構造を提供します。すべてのプロジェクト、アプリケーションもしくはモジュールを初期化するときに、このツリー構造は自動的に作成されます。もちろん、あなたの都合にあわせてファイルとディレクトリを再編成する、もしくは顧客の要件を満たすためにこのツリー構造を完全にカスタマイズできます。

#### rootのツリー構造

これらはsymfonyのプロジェクトのrootで見つかるディレクトリです:

    apps/
      frontend/
      backend/
    cache/
    config/
    data/
      sql/
    doc/
    lib/
      model/
    log/
    plugins/
    test/
      bootstrap/
      unit/
      functional/
    web/
      css/
      images/
      js/
      uploads/

テーブル2-1でこれらのディレクトリの内容が説明されています。

テーブル2-1 - ルートディレクトリ

ディレクトリ | 説明
------------ | ------------
`apps/`      | プロジェクトのアプリケーションごとに1つのディレクトリが格納される(よくあるのは`frontend`と`backend`でそれぞれはフロントオフィス用とバックオフィス用)
`cache/`     | 設定のキャッシュバージョン、(有効にした場合)アクションのキャッシュバージョン、そしてプロジェクトのテンプレートが格納される。キャッシュメカニズム(詳細は12章)はWebリクエストへのレスポンスを加速するためにこれらのファイルを使う。それぞれのアプリケーションはここにサブディレクトリを保有し、あらかじめ処理されたPHPとHTMLファイルを保存する。
`config/`    | プロジェクトの一般的な設定が保存される。
`data/`      | ここでは、プロジェクトのデータファイルを保存できる。たとえば、データベーススキーマ、テーブルを作成するSQLファイル、もしくはSQLiteデータベースファイルなど。
`doc/`       | プロジェクトのドキュメントが保存される。あなた独自のドキュメントやPHPdocによって生成されたドキュメントも含む。
`lib/`       | 外部クラスもしくは外部ライブラリ専用。ここでは、アプリケーションのあいだで共有が必要なコードを追加できる。`model/`サブディレクトリはプロジェクトのオブジェクトモデルを保存する(8章で説明)。
`log/`       | symfonyによって直接生成されたアプリケーションのログファイルを保存する。Webサーバーのログファイル、データベースのログファイル、もしくはプロジェクトの任意部分からのログファイルなども保存できる。symfonyはアプリケーション単位と環境単位で1つのログファイルを作成する(ログファイルは16章で説明)。
`plugins/`   | アプリケーションにインストールされたプラグインが保存される(プラグインは17章で説明)
`test/`      | PHPで記述されsymfonyのテストフレームワークと互換性のあるユニットテストと機能テストを保存する(15章で説明)。プロジェクトのセットアップ最中に、symfonyはいくつかの基本的なテストを使用してスタブを自動的に追加する。
`web/`       | 	Webサーバーのroot。インターネットからアクセスできるファイルはこのディレクトリに設置されたもののみ。

#### アプリケーションのツリー構造

すべてのアプリケーションディレクトリのツリー構造は同じです:

    apps/
      [application name]/
        config/
        i18n/
        lib/
        modules/
        templates/
          layout.php

テーブル2-2でアプリケーションのサブディレクトリが説明されています。

テーブル2-2 - アプリケーションのサブディレクトリ

ディレクトリ | 説明
------------ | -----------
`config/`    | たくさんのYAML設定ファイルの一式を保有する。フレームワーク自身で見つかるデフォルトパラメーターとは別に、たいていのアプリケーションの設定が存在する場所。必要な場合、ここでデフォルトパラメーターをさらにオーバーライドできることに注意。5章でアプリケーションの設定について詳しく学ぶ。
`i18n/`      | アプリケーションの国際化のために使われるファイルが格納される。多くはインターフェイスの翻訳ファイルが格納される(13章で国際化機能を扱う)。国際化のために1つのデータベースを選択した場合、このディレクトリを回避できる。
`lib/`       | アプリケーションに固有のクラスとライブラリが格納される。
`modules/`   | アプリケーションの機能を含むすべてのモジュールが含まれる。
`templates/` | アプリケーションのグローバルテンプレートの一覧が表示される。グローバルテンプレートはすべてのモジュールに共有される。デフォルトでは1つの`layout.php`ファイルが保有される。このファイルはモジュールテンプレートが挿入されるメインのレイアウト。

>**NOTE**
>`i18n/`、`lib/`、`modules/`ディレクトリは新しいアプリケーションのために空です。

アプリケーションのクラスは同じプロジェクトのほかのアプリケーションのメソッドもしくはプロパティにアクセスできません。同じプロジェクトの2つのアプリケーションのあいだのハイパーリンクは絶対パスでなければならないことを注意してください。1つのプロジェクトを複数のアプリケーションに分割する方法を選ぶとき、この最後の制約に注意する必要があります。

#### モジュールのツリー構造

それぞれのアプリケーションは1つもしくは複数のモジュールを保有します。それぞれのモジュールは独自のサブディレクトリを`modules`ディレクトリに格納し、このディレクトリの名前はセットアップの最中に選択されます。

つぎのものはモジュールの典型的なツリー構造です:

    apps/
      [アプリケーションの名前]/
        modules/
          [モジュールの名前]/
              actions/
                actions.class.php
              config/
              lib/
              templates/
                indexSuccess.php

テーブル2-3でモジュールのサブディレクトリが説明されています。

テーブル2-3 - モジュールのサブディレクトリ

ディレクトリ  | 説明
------------- | ------------
`actions/`    | 一般的に`action.class.php`と呼ばれる単独のファイルが保存される。モジュールのすべてのアクションをこのファイルに保存できる。異なるファイルで1つのモジュールの異なるアクションを書くこともできる。
`config/`     | モジュール用のローカルパラメーターを持つカスタム設定ファイルを保存できる。
`lib/`        | モジュール固有のクラスとライブラリを保存できる。
`templates/`  | モジュールのアクションに対応するテンプレートが格納される。デフォルトのテンプレートの名前は`indexSuccess.php`で、モジュールのセットアップの間に作成される。

>**NOTE**
>`config/`、と`lib/`ディレクトリは新しいモジュールのために空です。

#### webディレクトリのツリー構造

`web`ディレクトリは一般利用者がアクセスできるファイルのディレクトリで、ごくわずかですが制約が存在します。基本的な命名規約に従うことでデフォルトのふるまいと便利なショートカットが提供されます。`web`ディレクトリ構造の例はつぎのとおりです:

    web/
      css/
      images/
      js/
      uploads/

慣習的には、静的なファイルはテーブル2-4の一覧で示されるディレクトリに配置されます。

テーブル 2-4 - 典型的なwebのサブディレクトリ

ディレクトリ  | 説明
------------- | ---------------------------------
`css/`        | `.css`拡張子を持つスタイルシート
`images/`     | `.jpg`、`.png`もしくは`.gif`形式の画像が保存される
`js/`         | `.js`拡張子を持つJavaScriptファイルが保存される
`uploads/`    | ユーザーがアップロードしたファイルを保存できる。通常このディレクトリには画像が保存されるが、開発サーバーと運用サーバーの同期化がアップロードされた画像に影響を与えないようにimagesディレクトリとは区別される。

>**NOTE**
>デフォルトのツリー構造の維持は大いに推奨されますが、特定のニーズのために修正することは可能です。たとえば、異なるツリー構造とコーディング規約を持つサーバーでプロジェクトを運営できるようにするためなどです。ファイルのツリー構造の修正に関する詳細な情報は19章を参照してください。

共通の手法
----------

symfonyの開発のなかで少数のテクニックが繰り返し使われ、この本とあなた独自のプロジェクトでもこれらをよく目にすることになります。これらはパラメーターホルダー、定数、そしてクラスのオートロードを含みます。

### パラメーターホルダー

symfonyの多くのクラスは1つのパラメーターホルダー(parameter holder)を含みます。これはゲッターとセッターメソッドを持つプロパティをカプセル化するために便利な方法です。たとえば、`sfRequest`クラスは`getParameterHolder()`メソッドを呼び出すことで読みとりできるパラメーターホルダーを保有します。リスト2-14で示されるように、それぞれのパラメーターホルダーは同じ方法でデータを保存します。

リスト2-14 - `sfRequest`パラメーターホルダーを使う

    [php]
    $request->getParameterHolder()->set('foo', 'bar');
    echo $request->getParameterHolder()->get('foo');
     => 'bar'

パラメーターホルダーを使う多くのクラスはget/setオペレーションのために必要なコードを短くするプロキシ(proxy)メソッドを提供します。これは`sfRequest`オブジェクトにもあてはまり、リスト2-15のコードはリスト2-14と同じことができます。

リスト2-15 - `sfRequest`パラメーターホルダーのプロキシメソッドを使う

    [php]
    $request->setParameter('foo', 'bar');
    echo $request->getParameter('foo');
     => 'bar'

パラメーターホルダーのゲッターは2番目の引数をデフォルト値として受けとります。これは条件文で実現できることよりもはるかに簡潔で便利なフォールバックメカニズム(訳注：障害が起きても最低限の機能を維持するメカニズム)を提供します。具体例はリスト2-16をご覧ください。

リスト2-16 - パラメーターホルダーのゲッターのデフォルト値を使う

    [php]
    // 'foobar' パラメーターは定義されていないので、ゲッターは空の値を返す
    echo $request->getParameter('foobar');
     => null

    // デフォルト値はゲッターを条件文に設置することで利用可能
    if ($request->hasParameter('foobar'))
    {
      echo $request->getParameter('foobar');
    }
    else
    {
      echo 'default';
    }
     => default

    // しかしそれに対して2番目のゲッターの引数を使うほうがはるかに速い
    echo $request->getParameter('foobar', 'default');
     => default

symfonyのコアクラスのなかには(`sfNamespacedParameterHolder`クラスのおかげで)名前空間をサポートするものもあります。3番目の引数をセッターもしくはゲッターに指定する場合、それは名前空間として使われ、パラメーターはその名前空間でのみ定義されます。リスト2-17で例が示されています。

リスト2-17 - `sfUser`パラメーターホルダー名前空間を使う

    [php]
    $user->setAttribute('foo', 'bar1');
    $user->setAttribute('foo', 'bar2', 'my/name/space');
    echo $user->getAttribute('foo');
     => 'bar1'
    echo $user->getAttribute('foo', null, 'my/name/space');
     => 'bar2'

もちろん、構文のファシリティを利用するために独自クラスにパラメーターホルダーを追加することもできます。リスト2-18はパラメーターホルダーでクラスを定義する方法を示しています。

リスト2-18 - 独自クラスにパラメーターホルダーを追加する

    [php]
    class MyClass
    {
      protected $parameterHolder = null;

      public function initialize($parameters = array())
      {
        $this->parameterHolder = new sfParameterHolder();
        $this->parameterHolder->add($parameters);
      }

      public function getParameterHolder()
      {
        return $this->parameterHolder;
      }
    }

### 定数

定数は一度定義すると変更できないのでsymfonyでは定数は見つかりません。symfonyは`sfConfig`と呼ばれる独自の設定オブジェクトを使い、これは定数を置き換えます。このオブジェクトはどこからでもパラメーターにアクセスできる静的なメソッドを提供します。リスト2-19は`sfConfig`クラスのメソッドの使いかたを示しています。

リスト2-19 - 定数の代わりに`sfConfig`クラスのメソッドを使う

    [php]
    // PHP定数の代わり
    define('FOO', 'bar');
    echo FOO;

    // symfonyはsfConfigオブジェクトを使う
    sfConfig::set('foo', 'bar');
    echo sfConfig::get('foo');

`sfConfig`メソッドはデフォルト値をサポートし、値を変更するために同じパラメーターで`sfConfig::set()`メソッドを何度も呼び出すことができます。5章で`sfConfig`メソッドの詳細を検討します。

### クラスのオートロード機能

通常の方法では、PHPでクラスを使うもしくはオブジェクトを作るとき、まずはクラスの定義をインクルードする必要があります。

    [php]
    include 'classes/MyClass.php';
    $myObject = new MyClass();

しかし、多くのクラスと深いディレクトリ構造をかかえる大きなプロジェクトにおいて、インクルードするすべてのクラスファイルとそれらのパスを追跡するために多くの時間が必要です。`spl_autoload_register()`関数を提供することで、symfonyは`include`ステートメントを不要にするので、つぎのようにクラスを直接書けます:

    [php]
    $myObject = new MyClass();

symfonyはプロジェクトの`lib/`ディレクトリの1つのなかに存在する`php`拡張子で終わるすべてのファイルのなかの`MyClass`の定義を探します。クラスの定義が見つかったら、ファイルは自動的にインクルードされます。

すべてのクラスを`lib/`ディレクトリに保存する場合、クラスをインクルードしなくてすみます。symfonyプロジェクトが通常`include`ステートメントもしくは`require`ステートメントを必要としない理由はそういうわけです。

>**NOTE**
>パフォーマンスをさらに改善するために、symfonyのオートロード機能は最初のリクエストの間に(内部の設定ファイルで定義された)ディレクトリのリストをスキャンします。symfonyはこれらのディレクトリが格納するすべてのクラスを登録し、PHPファイルに対応するクラス/ファイルを連想配列として保存します。この方法によって、今後のリクエストではディレクトリのスキャンを行う必要はありません。クラスファイルをプロジェクトに追加もしくは移動させるたびに`symfony cache:clear`コマンドを呼び出してキャッシュをクリアする必要がある理由はそういうわけです(開発環境を除いて、クラスが見つからないときsymfonyは一度キャッシュをクリアします)。キャッシュは12章で、オートロード機能の設定は19章で学ぶことになります。

まとめ
----

MVCフレームワーク(Model-View-Controller framework)を利用することで、フレームワークの規約にしたがってコードを分割して編成することが強制されます。プレゼンテーションのコードはビュー(View)に、データの操作コードはモデル(Model)に、リクエストの操作ロジックはコントローラー(Controller)になります。これによってMVCパターンのアプリケーションはとても便利で非常に制限されたものになります。

symfonyはPHP 5で書かれたMVCフレームワークです。その構造はMVCパターンを乗り越えながらも、非常に使いやすいように設計されました。器用で設定が柔軟であるおかげで、symfonyをすべてのWebアプリケーションのプロジェクトに適用できます。

今、あなたはsymfonyの背後に存在する基本理論を理解したので、最初のアプリケーションを開発する準備がほとんど整いました。しかしそのまえに、symfonyをインストールして開発サーバーで動かすことが必要です。
