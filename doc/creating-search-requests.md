# 検索リクエストの作成とレスポンスの形式
Solrのインデックスに対する検索リクエストは、Javaで直接呼び出すほか、ネットワーク越しにHTTPリクエストでも行うことが出来ます。ここでは、よく使われるであろう後者の方式のみ解説します。

    $ curl 'http://localhost:8983/solr/hoge/select?q=body:hoge'
    
Solrがサポートする検索リクエストのAPIは、大きく分けて2種類、細かく分けると3種類あり、開発者が用途によって使い分けられるようになっています。順を追って説明します。- [Lucene Query Parser](http://wiki.apache.org/solr/SolrQuerySyntax#lucene)- [DisMax Query Parser](http://wiki.apache.org/solr/DisMaxQParserPlugin)- [Extended DisMax Query Parser](http://wiki.apache.org/solr/ExtendedDisMax)
## Lucene Query Parser
Solrで利用できるもっとも基本的なクエリパーザーがLucene Query Parserです。SolrはLuceneをベースにしている関係で、Luceneのそれと上位互換性があります。フィールドごとにいわゆるブーリアン検索が出来ます。
    ?q=title:(hoge OR fuga)

複数フィールドにまたがって検索したい場合は明示的に指定します。    
    ?q=title:はてな OR body:はてな### Lucene Query Parserの挙動デフォルトでは、検索クエリが文書と同じようにトークナイズされるため、片方でもマッチすれば文書自体はマッチしたと見なされます。
    
    ?q=body:山田太郎 # 山田 だけでも 太郎 だけでもマッチする

“キーワードがこの順で連続して現れる” 文書だけ検索したい場合はフレーズクエリを使うとよいです。検索ワードを""で囲みます。

    ?q=body:”山田太郎” # 山田太郎 だけにマッチ

詳細な仕様は、[Apache Lucene - Query Parser Syntax](http://lucene.apache.org/core/old_versioned_docs/versions/3_5_0/queryparsersyntax.html) を参照して下さい。

### Lucene Query Parser のメリット・デメリット
#### メリット
- クエリに文書がマッチする条件を厳密に伝えられる
- ブーリアン検索は一般的なウェブ検索エンジンで使える検索クエリとして慣れ親しんでいる人が多く、理解しやすい#### デメリット
- 検索したいフィールドが複数あり、検索クエリが複数語から成る場合に、検索クエリの組み立てが複雑になる。具体的には、2語のAND検索を2つフィールドで網羅的に行う場合、以下のようになります。
    q=(title:はてな AND title:しなもん) OR       (body:はてな AND body:しなもん) OR       (title:はてな AND body:しなもん) OR       (title:しなもん AND body:はてな) 
- 文書のスコアリングも基本的に組み込みのTF・IDFのみ

##### 解決策- schema.xml で CopyFieldを使い、検索したいTextFieldを一つにまとめてインデックスする
この方法でも可能ですが、DisMax Query Parserにおいてフィールドごとにマッチしたときの重みをカスタマイズする方法（後述）は使えなくなります。- DisMax Query Parser や Extended DisMax Parser などの拡張クエリを使う検索対象文書が多く、文書のスコアリング（検索結果のランキング）が重要な場合、こちらがおすすめです。

## DisMax Query Parser

Lucene Query Parserでカバーしきれないユースケースを満たすためにSolrで独自に開発されたクエリパーザーです。** &defType=dismax ** というパラメータを指定することでデフォルトのLucene Query Parserに代わってDisMax Query Parserを利用するモードになります。

1. 各フィールドをそれぞれ(Disjunction)検索して
2. TF・IDFスコアがもっとも高い(Max)フィールドを
3. その文書のスコアとして表現する

のが、基本的な動作です。
### DisMax Query Parserの使いどころ
#### 多数のテキストフィールドを横断的に検索する

    q=はてな しなもん&defType=dismax&qf=title+body

BooleanQueryの例よりシンプルに書けるのが分かると思います。

**qf** パラメータを指定すると、他のテキストフィールドよりも重視したいなフィールドがある場合、個別に重みを指定することが出来ます。以下はタイトルでのマッチを本文よりも重視する(タイトルの方にweightを設定する)例です。

    q=はてな しなもん&defType=dismax&qf=title^2+body

#### テキストフィールド以外のフィールドを文書のスコアリングに使う

**boost** パラメータを使うことで、ソーシャルボタンのShare数や、ブログエントリの日付の新しさ、テキストの内容以外のメタデータを文書のスコアリングに使えるようになります。ここで使える関数などは[FunctionQuery - Solr Wiki](http://wiki.apache.org/solr/FunctionQuery)を参照して下さい。

    q=はてな しなもん&defType=dismax&qf=title^2 body&boost=log(share_count)

#### 単語の出現位置を考慮してスコアリングする
複数のキーワードで検索した場合、とくにキーワード感の近さなどはスコアリングに考慮されないのですが、これだと長い文書のまったく関係ないコンテキストでたまたまマッチしてしまったりして都合が悪いです。この問題を解決するために、複数のキーワードで検索したときに検索対象の文書中のマッチしたキーワード間の距離に関して

- どのぐらい近ければ"文書が検索クエリにマッチした"と見なすか
- どのぐらい近ければ"キーワード同士が近くにある"と見なすか

を考慮するオプションがあります。

**qs** パラメータを指定して、複数のキーワードで検索したとき、それらの距離が600 positions 以下の場合のみ検索クエリにマッチしたことにする。

    q=はてな しなもん&defType=edismax&qf=title^2 body&qs=600

**ps** パラメータで複数のキーワードで検索したとき、各フィールドでそれらの距離が20 positions 以下の場合に検索クエリにとくに関連する文書と見なして、各フィールドの追加の重みを**pf** パラメータで設定する。

    q=はてな しなもん&defType=edismax&qf=title^2 body&pf=title^4 body^2&ps=20

## Extended DisMax Query Parser

DisMax Query Parser よりもさらに詳細に文書のマッチやスコアリングをコントロールできる [Extended DisMax Query Parser]() があります。**defType=edismax** と指定することで、文書のマッチとスコアリングに関するいくつかの追加パラメータが指定可能になりますが、詳細は割愛します。 [ExtendedDisMax - Solr Wiki](http://wiki.apache.org/solr/ExtendedDisMax) を参照して下さい。

## レスポンスの形式
レスポンス形式はデフォルトのXMLのほか、JSON, Ruby, CSVなどが選べます。**wt** パラメータで指定します。

    $ curl 'http://localhost:8983/solr/hoge/select?q=body:hoge&wt=json'

また、必要に応じて自分でレスポンス形式を定義することも出来ます。詳しくは[QueryResponseWriter - Solr Wiki](http://wiki.apache.org/solr/QueryResponseWriter)を参照して下さい。