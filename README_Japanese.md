Heroku buildpack: Ruby with SQLite3
===================================

これは、
[heroku-buildpack-ruby](https://github.com/heroku/heroku-buildpack-ruby)
からフォークしたレポジトリーです。  
Heroku 上で SQLite3 を利用できるように変更したものです。

***
**【ご注意】**  
**ヘロク上でSQLite3を利用した場合、更新データの消滅等の現象が発生することをよくご理解いただいた上で、このビルドパックをご利用ください。**
***


ヘロク は SQLite3 の利用を制限している
-----
ヘロク は SQLite3 の利用を推奨していません。
これについては、ヘロクのドキュメント・ページ
[SQLite on Heroku](https://devcenter.heroku.com/articles/sqlite3)
にて解説されています。


ヘロクの Cedarスタック は 
**[ephemeral filesystem](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem)**
（ ephemeral：はかない、短命な）と呼ばれる ファイル・システムになっています。  
ヘロクのアプリケーションの実行単位である dyno では、
起動されると独自の仮想環境（Cedarスタックの場合、Ubuntu 10.04）が構築され、
そこへ、デプロイされた Slug が読み込まれ展開されます。
この環境におけるファイル・システムは、
通常のLinuxのファイル・システムと何も変わりないので、自由に読み書きが行えます。
しかしながら、dyno が終了する際に、この仮想環境はファイル・システムを含めて丸ごと破棄されます。
つまり、その dyno の実行中に更新されたデータはどこにも保存されることなく、
完全に消滅してしまうわけです。

一方、SQLite3 のデータベースは そのアプリケーションから操作できる
ファイル・システム上の１つのファイルにデータを保存します。

つまり、ヘロクの Cedarスタック上で、SQLite3 を利用すると、
データの更新等、まったく問題なく動作しますが、dyno が終了する際に、
それらの更新されたデータは全ては消滅してしまうことになります。

このような理由により、ヘロクは故意に SQLite3 をディプロイできないようにしている、と考えられます。





それでも Heroku上で SQLite3 を利用するメリットは なぜか？
-----
前述のとおり、SQLite3 は、１つのファイルですべてのデータを管理できます。
これは、mySQL や、ヘロクが推奨している PostgreSQL に 比べ、ハンドリングが容易ですし、
データのバックアップもファイルを１つコピーするだけ、と超簡単です。


前項の説明のとおり、SQLite3 は ヘロクの
**[ephemeral filesystem](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem)**
には適しません。
しかしながら、ある特定の用途においては、PostgreSQL等に比べても、より適しているといえます。

その特定の用途とは、
データベースの更新の頻度が比較的少なく、かつ、
そのデータベースの更新が、サイト訪問者によるものではなく、
アプリケーションの作成者自身によってディプロイ(git push) される場合です。

SQLite3 は、１つのファイルでデータを管理できるので、
事前にそのデータベース・ファイルにデータを保存しておき、
そのファイルをアプリケーションと一緒に Gitでアーカイブしてから
デプロイすれば、そのデータを簡単に利用できるようになるわけです。

つまり、ヘロク上でデータベースへの書き込みを必要としないアプリケーションであれば、
データの読み込みや検索については、全く問題ありませんし、
ローカルで更新したデータを ヘロク側へアップロードするのも、
ましてや、データベースのスキームの変更に際しても、
単に、アプリケーションのデプロイ(git push)を、もう一回、やり直すだけとなり、
その他の一切の操作が必要なくなります。

このように、アプリケーションとデータベースを一体化して、そのままのかたちで デプロイできるので、
ローカルとリモート間で データやスキームに矛盾がおこることもなく、
ヘロク上でのアプリケーションの動作確認作業も非常に楽になります。


具体的な用途は、以下のようなものです。




適切なアプリケーションの例
-----
* 郵便番号と地名のデータベース
* 過去１００年にわたる 月間の平均気温と降水量のデータベース
* 更新するのが、アプリケーション作者本人のみによるブログ（記事の更新をローカルで行い、その都度、アプリ全体をデプロイし直す）



不適切なアプリケーションの例
-----
* 例えば、掲示板システムのように、サイトの訪問者により頻繁にデータベースが更新されてゆくアプリケーション。  


***


使い方
-----
まずローカルの開発環境で、データベースにSQLite3を用いた通常のアプリケーションを開発してください。
次に、アプリケーションを SQLite3のデータベース・ファイルと一緒に Git に保存します。
それから、ヘロク へ デプロイ(git push) します。

詳細は以下のとおりです。



### Gemfile
Gemfile ファイルに "sqlite3" の記述を加えてから、"bundle install" を実行します。

    gem "sqlite3"



### config/database.yml
config/database.yml に **development:** と **production:** の両方について
以下のように記述します。

    # SQLite3 configuration

    development:
      adapter: sqlite3
      database: db/mydata.sqlite3

    production:
      adapter: sqlite3
      database: db/mydata.sqlite3



  
### BUILDPACK_URL
デプロイ(git push)  する前に、環境変数 **BUILDPACK_URL** を以下のように設定します。
詳細については
[Using a custom Buildpack](https://devcenter.heroku.com/articles/buildpacks#using-a-custom-buildpack)
をご参照ください。

```sh
$ heroku config:set BUILDPACK_URL=https://github.com/yotsumoto/heroku-buildpack-ruby-with-sqlite3
```



### Git と デプロイ
あとは、通常のヘロクのアプリケーションと同様にして、Git で push してデプロイします。

```sh
$ git init
$ git add .
$ git commit -m "init"
$ git push heroku master
```  


***


技術解説： なぜ、SQLite3のデプロイは失敗するのか？
-----
Ruby や Rails のアプリケーションにおいて、Gemfile に "sqlite3" を追加してから
通常の（カスタムのビルドパックを用いない）方法で デプロイ(git push) しようとすると
失敗してしまい、デプロイが完了しません。

その原因は以下の理由によります。

### **sqlite3.h** と **libsqlite3.so** の欠落
sqlite3 の gem のインストールにおいては、Rubyのみで記述された通常のGemとは異なり、
**ネーティブ・エクステンション(native extension)** と呼ばれるライブラリを作成しようとします。
これは、Gemインストーラーが C言語で作成されたプログラムを コンパイル して リンク することを意味します。  
ところが、ヘロクの Cedarスタックは、C言語のコンパイラーが必要とする **sqlite3.h** というヘッダー・ファイルを持っていません。
そのため、以下のように **sqlite3.h** が見つからない、という旨のエラーが発生し、デプロイが失敗します。

```
Gem::Installer::ExtensionBuildError: ERROR: Failed to build gem native extension.
/tmp/build_d2469b1e-a763-4b87-8342-a3be831993fa/vendor/ruby-2.0.0/bin/ruby extconf.rb
checking for sqlite3.h... no
sqlite3.h is missing. Try 'port install sqlite3 +universal',
'yum install sqlite-devel' or 'apt-get install libsqlite3-dev'
and check your shared library search path (the
location where your sqlite3 shared library is located).
```

さらに、リンクの際に必要となる **libsqlite3.so** という名のライブラリのシンボリックリンクも存在していません。

これは、上記のエラー・メッセージにもあるとおり、Ubuntu 10.04ベースの ヘロクの Cedarスタックに
**[libsqlite3-dev](http://packages.ubuntu.com/lucid/libsqlite3-dev)** 
と呼ばれるパッケージがインストールされていない、ということを示しています。

ヘロクの Cedarスタックにおいて、 SQLite3 gem のインストールを成功させるためには、

* **sqlite3.h** というヘッダー・ファイルが存在しない
* **libsqlite3.so** という名のライブラリのシンボリックリンクが存在しない

という ２つの問題を解決しなければならないことになります。  


さらに、これ以外にもの、問題となる点が別に２つあります。

### **config/database.yml** への上書き
通常のビルドパック
[heroku-buildpack-ruby](https://github.com/heroku/heroku-buildpack-ruby)
は、デプロイ中に、**config/database.yml** ファイルを上書きしてしまう、
という問題点があります。
この新たに上書きされた **config/database.yml** は一種のスクリプトになっていています。
ヘロクでは、システムが提供している PostgreSQL を利用することを前提としていますの、
そのデータベースへの接続情報が記載された 環境変数 **DATABASE_URL** の内容を
新たな dyno の起動時に解析して、
adapter、database、username、password、host、port 等のパラメータを
**config/database.yml** の形式にして、Rails 等が読み込める形に変換します。

もし、SQLite3 gem のインストールに成功したとしても、
**config/database.yml**が上書きされれしまったのでは、
SQLite3 に関する情報は、Rails には届かないことになります。
よって、この **config/database.yml** への上書き を阻止する必要があります。



### 自動的に **heroku-postgresql:hobby-dev addon** がインストールされる
通常のビルドパック
[heroku-buildpack-ruby](https://github.com/heroku/heroku-buildpack-ruby)
において、アプリケーションが Rails だと判断されると、
自動的に **heroku-postgresql:hobby-dev addon** がインストールされるように設定しています。
そして、このアドオンがインストールされるに伴い、
[DATABASE_URL config var now set automatically when provisioning Postgres add-on](https://devcenter.heroku.com/changelog-items/438)
にあるとおり、
環境変数 **DATABASE_URL** も自動的に設定されることになります。

また、通常の
[heroku-buildpack-ruby](https://github.com/heroku/heroku-buildpack-ruby)
で作成された Slug には、新たな dyno の起動に動作環境をチェックする gem が自動的にインストールされるようです。
この gem が、環境変数 **DATABASE_URL** の存在を確認すると、
その内容と、それを接続するための PostgreSQL の gem (pg) がインストールされているかも確認するようです。
その際、もし、PostgreSQL の gem がインストールされていない場合は、
```
/app/vendor/bundle/ruby/2.0.0/gems/bundler-1.5.2/lib/bundler/rubygems_integration.rb:240:in `block in replace_gem': Please install the postgresql adapter: `gem install activerecord-postgresql-adapter` (pg is not part of the bundle. Add it to Gemfile.) (LoadError)
```
のようなエラーがログに表示されて、 dyno が クラッシュしてしまうことになります。

データベースとして、SQLite3 のみを利用しようとしている場合、
通常 PostgreSQL の gem はインストールしないことになりますので、
（ PostgreSQL の gem をインストールしてもよいのかもしれないが、使いもしないgemをインストールするのは無駄なことです。）
この クラッシュを避ける必要があります。
よって、自動的に **heroku-postgresql:hobby-dev addon** がインストールされてしまう
阻止する必要があります。

もし開発中、なんらの理由で、このようにクラッシュするようになってしまった場合は、
```sh
$ heroku config:unset DATABASE_URL
```
のように、環境変数 **DATABASE_URL** を削除してみてください。  


***


変更点
-----
ヘロク上での SQLite3 利用を実現するにあたり、オリジナルの
[heroku-buildpack-ruby](https://github.com/heroku/heroku-buildpack-ruby)
から変更した点は以下の５点です。  


### sqlite3.h の追加
コンパイルに必要な **sqlite3.h** が存在していないので、これを **vendor** ディレクトリに追加しました。

[vendor/sqlite3.h](https://github.com/yotsumoto/heroku-buildpack-ruby-with-sqlite3/blob/master/vendor/sqlite3.h)  


### libsqlite3.so のシンボリックリンクの作成
デプロイ中、bundle が実行される前に、libsqlite3.so から /usr/lib/libsqlite3.so.0.8.6 へ シンボリックリンクを張るようにしました。

[lib/language_pack/ruby.rb](https://github.com/yotsumoto/heroku-buildpack-ruby-with-sqlite3/blob/master/lib/language_pack/ruby.rb)
```ruby
L525      run("ln -s /usr/lib/libsqlite3.so.0.8.6 #{yaml_lib}/libsqlite3.so")                        # for sqlite3   make symbolic link
```  


### sqlite3.h のコピー
デプロイ中、bundle が実行される前に、vendor/sqlite3.h を include ディレクトリにコピーするようにしました。

[lib/language_pack/ruby.rb](https://github.com/yotsumoto/heroku-buildpack-ruby-with-sqlite3/blob/master/lib/language_pack/ruby.rb)
```ruby
L526      run("cp #{File.expand_path( "../../vendor/sqlite3.h", $PROGRAM_NAME )} #{yaml_include}")   # for sqlite3   prepare sqlite3.h
```  



### config/database.yml への上書き防止
config/database.yml への上書きするメソッド **create_database_yml** は 94行目から呼び出されている。
これは、81行目から始まっている **compile** というメソッドの中である。
よって、 config/database.yml への上書きを防止するために、この 94行目をコメントにした。

[lib/language_pack/ruby.rb](https://github.com/yotsumoto/heroku-buildpack-ruby-with-sqlite3/blob/master/lib/language_pack/ruby.rb)
```ruby
L94  #        create_database_yml        # for sqlite3    config/database.yml  should be kept intact
```  


### heroku-postgresql:hobby-dev addon の自動インストールの防止
heroku-postgresql:hobby-dev addon が 自動的にインストールされるのを防止するために、
以下の行をコメントにした。

[lib/language_pack/rails2.rb](https://github.com/yotsumoto/heroku-buildpack-ruby-with-sqlite3/blob/master/lib/language_pack/rails2.rb)
```ruby
L65  #  def add_dev_database_addon                # for sqlite3   prevent from forcing addon 'heroku-postgresql:hobby-dev'
L66  #    ['heroku-postgresql:hobby-dev']
L67  #  end
```  


***

