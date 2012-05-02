# データのインポートとインデキシング
[目次に戻る](../README.md)

Solrに文書をインポートする方法は、[複数提供](http://wiki.apache.org/solr/#Search_and_Indexing)されています。基本的な方法としては、[XML](http://wiki.apache.org/solr/UpdateXmlMessages)、[JSON](http://wiki.apache.org/solr/UpdateJSON)、[CSV](http://wiki.apache.org/solr/UpdateCSV)などの形式で決められたHTTPエンドポイントにHTTP POSTすることで登録できます。また、外部のデータベースなどから直接文書データを取り込むことが出来る [DataImportHandler](http://wiki.apache.org/solr/DataImportHandler) というツールが標準で提供されています。

ここでは実際に使われるケースが多いであろう「DataImportHandlerでJDBCを使ってMySQLからSolrにデータをインデックスする」ときの基本的な手順と、その際のコツについて解説します。

##	必要な設定ファイル

$CORE_HOME は solr.core.instanceDir として指定されたディレクトリを示したものです。solr.xml 内で指定するか、あるいはcoreを作成するときに明示的に指定することができます。

### ファイルの構成

    $CORE_HOME/
      conf/
        solrconfig.xml
        data-config.xml
        schema.xml
      lib/
        (3rd party製のトークナイザやJDBCドライバなどはここに配置するとSolrに認識される)

### solrconfig.xml

キャッシュの量やリクエストハンドラ、レプリケーションの設定など、 Solrの全体的な動作を記述するのが solrconfig.xml です。基本的な使い方をしている間は、あまり設定することはないはずです。

[SolrConfigXml - Solr Wiki](http://wiki.apache.org/solr/SolrConfigXml)

### data-config.xml

DataImportHandlerを使ってインポートする方法を記述する設定ファイルです。例を使って説明します。

テーブルスキーマで db01 というホストの blog_db というデータベースに、以下のスキーマでデータがあるとします。

    CREATE TABLE blog_entry (
      entry_id INT unsigned NOT NULL AUTO_INCREMENT,
      title varbinary(255) NOT NULL DEFAULT '',
      body blob NOT NULL DEFAULT '',
      category_id tinyint unsigned,
      created  timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
      modified timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      PRIMARY KEY (entry_id),
      KEY (created),
      KEY (modified)
    ) ENGINE=InnoDB DEFAULT CHARSET=binary;
    
    CREATE TABLE category (
        category_id TINYINT UNSIGNED AUTO_INCREMENT,
        category_name varbinary(255) NOT NULL DEFAULT ''
    ) ENGINE=InnoDB DEFAULT CHARSET=binary;

これをSolrにインポートするための data-config.xml は以下のようなものです。

    <dataConfig>
      <dataSource
        name="blog-db"
        driver="com.mysql.jdbc.Driver"
        url="jdbc:mysql://db01/blog_db"
        user="nobody"
        password="nobody"
        batchSize="-1"
        useUnicode="true"
        characterEncoding="utf8"
        useOldUTF8Behavior="true”
        readOnly="true" />
      <entity
        name="blog_entry"
        dataSource="blog-db"
        query="
          SELECT 
            entry_id,
            CAST(title AS CHAR) AS title,
            CAST(body AS CHAR) AS body,
            DATE_FORMAT(created,'%Y-%m-%d %H:%i:%S JST') AS created,
            CAST(category_name AS CHAR) AS category
          FROM
            blog_entry
              LEFT JOIN category USING (category_id)
          "
        deltaQuery="
          SELECT
            entry_id
          FROM
            blog_entry
          WHERE
            modified &gt; ${dataimporter.last_index_time}
          "
        deltaImportQuery="
          SELECT
            entry_id,
            CAST(title AS CHAR) AS title,
            CAST(body AS CHAR) AS body,
            DATE_FORMAT(created,'%Y-%m-%d %H:%i:%S JST') AS created,
            CAST(category_name AS CHAR) AS category
          FROM
            blog_entry
              LEFT JOIN category USING (category_id)
          WHERE 
            entry_id='${dataimporter.delta.entry_id}'
          "
        transformer="ClobTransformer,DateFormatTransformer">
        <field column="title" clob="true" />
        <field column="body" clob="true" />
        <field column="category" clob="true" />
        <field column="created" dateTimeFormat="yyyy-MM-dd hh:mm:ss z" />
      </entity>
    </dataConfig>


以下、それぞれの箇所を解説します。

#### MySQLのデータベースからインポートするときの設定
DataImportHandlerはJDBC経由では1行ずつfetchされたものをインデックスすることを想定しています。MySQLのJDBCでこの挙動をbatchSize="-1"と指定する必要があります。

    batchSize="-1"

これは分かりづらいため[FAQでも一番最初に表示されています](http://wiki.apache.org/solr/DataImportHandlerFaq)。


#### インポートクエリ
** query **

初回の全件インポート用のクエリを記述します。

** deltaQuery **

差分インポートをするにあたって、対象となるレコードの主キーを取得するためのクエリを記述します。ここでは、デフォルトで ** ${dataimporter.last_index_time} ** という、** ローカルのタイムゾーンでの前回のインポート開始時刻 **を変数として参照できます。必要に応じて、[独自の変数を定義することも出来ます](http://wiki.apache.org/solr/DataImportHandlerFaq#Is_it_possible_to_use_core_properties_inside_data-config_xml.3F)。

** deltaImportQuery **

deltaQuery で取得した主キーを元に、更新・追加されたentityをインポートするためのクエリを記述します。ここでは、 ** ${dataimporter.delta.(deltaQueryで取得したカラム)} ** の変数が使えます。上の例では ** ${dataimporter.delta.entry_id} ** を使っています。

#### UTF-8文字列のカラムを正しくインデックスするための設定
テーブルのカラムに格納されたUTF-8文字列を正しく扱うために必要な設定です。これを設定しないと、マルチバイト文字を正しく認識されず、文字化けすることがあります。

    useUnicode="true"
    characterEncoding="utf8"
    useOldUTF8Behavior="true”

#### 発行するSQLを参照系のみに限定する

インポート設定のミスによって誤ってテーブルの内容を書き換えてしまうことを防止します。

    readOnly="true"

#### フィールドの値の加工
Transformerという仕組みによって、インポートした後のフィールドの値をインデックスする前に加工することが出来ます。

    <entity name="blog_entry"
    <!-- 中略 -->
    transformer="ClobTransformer,DateFormatTransformer">
    <field column="title" clob="true" />
    <field column="created" dateTimeFormat="yyyy-MM-dd hh:mm:ss z" />

ClobTransformerは field 要素 で clob="true" と設定したフィールドを、byte型の配列から文字列に変換します。これはvarbinary型あるいはblob型のカラムがbyte型の配列としてDataImportHandlerに渡ってくるのを、後の行程で文字列として正しく処理するために必要な設定です。分かりづらいためか[FAQにも記述されています](http://wiki.apache.org/solr/DataImportHandlerFaq#Blob_values_in_my_table_are_added_to_the_Solr_document_as_object_strings_like_B.401f23c5)。

DateFormatTransformerは日付と時刻を含むフィールドを変換します。[Solrが受け付ける日付型のタイムゾーンはUTCのみ](http://lucene.apache.org/solr/api/org/apache/solr/schema/DateField.html)なので、フィールドの値をこの形式に適切に変換するために使います。

##### JavaScriptによるフィールドの加工

SolrをJava 6以上の環境で実行している場合、[既存のTransformer](http://wiki.apache.org/solr/DataImportHandler#Transformer)を利用する他に、[ScriptTransformer](http://wiki.apache.org/solr/DataImportHandler#ScriptTransformer)として data-config.xml 内で任意の処理を行うのtransformerをJavaScriptで定義できます。以下はgzipされた状態で格納されているフィールド(gzipped_body)があったとして、それを伸長し、UTF-8文字列として別なフィールド(body)に格納する例です。

    <dataConfig>
      <dataSource
      … />    
      <script><![CDATA[
        function gunzipBody(row) {
          var body = row.get('gzipped_body');
          var inflater = new java.util.zip.Inflater(true);
          inflater.setInput(body, 10, body.length-10);
          var result = java.lang.reflect.Array.newInstance(java.lang.Byte.TYPE, 1000000);
          var resultLength = inflater.inflate(result);
          inflater.end()
          row.put('body', new java.lang.String(result, 0, resultLength, 'UTF-8'));
          return row;
        }
      ]]></script>
      <document>
      <entity
        …
        transformer="script:gunzipBody">
      </entity>
      </document>
    </dataConfig>

上記の例を見ると分かるとおり、row.get('column_name') で加工前のフィールドの値を取得して加工した後で row.set('column_name') で 新たなフィールドを定義したり、既存のフィールドの値を上書きしたりできます。ScriptTransformerではJavaScriptの文法が使用でき、Javaの標準ライブラリをシームレスに利用できますが、Javaのパッケージや配列の扱いにの一部独特の癖があります。ここで利用可能な特有のイディオムについては[Scripting Java](http://www.mozilla.org/rhino/ScriptingJava.html)を参照して下さい。なお、独自のTransformerは[Javaで記述することも出来ます](http://wiki.apache.org/solr/DIHCustomTransformer)。

### schema.xml

data-config.xml で インポートしたデータをどうインデックスするかを決定するのが schema.xml です。

[SchemaXml - Solr Wiki](http://wiki.apache.org/solr/SchemaXml)

いくつかのポイントについて解説します。

トップレベル要素にスキーマの名前を定義できます。スキーマの名前はSolrの管理画面で表示され、操作対象を区別するために重要な情報となるので、容易に識別可能な名前にした方がよいです。ここでは、[coreレベルで利用可能な変数](http://wiki.apache.org/solr/CoreAdmin#Configuration)ため、coreの名前と同一にすることも出来ます。


    <schema name="${solr.core.name}" version="1.4">

最初から定義されているフィールドのほかに、独自のフィールドも定義できます。以下は、日本語文字列からなるフィールドの値を形態素解析した上でインデックスし、検索クエリに関しても同様なことを行った上で検索するフィールドを定義する例です。

    <types>
      …  
      <!-- A Text field that in Japanese -->
      <fieldType name="text_ja" class="solr.TextField" positionIncrementGap="100">
        <analyzer>
          <tokenizer class="org.atilika.kuromoji.solr.KuromojiTokenizerFactory" mode="search" user-dictionary=""/>
          <filter class="solr.LowerCaseFilterFactory" />
        </analyzer>
      </fieldType>
      …  
    </types>

org.atilika.kuromoji.solr.KuromojiTokenizerFactory というライブラリは [kuromoji - japanese morphological analyzer](http://www.atilika.org/) からダウンロードし、$CORE_HOME/lib 以下に配置すると利用可能になります。なお、[Solr 3.6 以上では Kuromoji Tokenizer が日本語の標準トークナイザーとして採用された](http://www.rondhuit.com/solr%E3%81%AE%E6%97%A5%E6%9C%AC%E8%AA%9E%E5%AF%BE%E5%BF%9C.html)ので、このような作業は必要ありません。


DataImportHandlerなどを経由して外部から取得したフィールドの値とtypes要素以下のフィールドの型の定義をもって、どのようにインデックスするかを決定するのが、fields要素内の設定です。

     <fields>
       …
       <field name="body" type="text_ja" indexed="true" stored="true" termVectors="true" />
       <field name="category" type="string" indexed="true" required="false" multiValued="true" />
       …
     </fields>

#### name
DataImportHandlerのqueryやdeltaImportQuery属性で指定したカラム名を指定することで、値をSolrのフィールドと対応させるための属性です。
#### indexed
フィールドの値をインデックスすることで、検索、ソート、フィルタリングを可能にするかどうかを設定します。デフォルトの値はfalseです。
#### stored
フィールドを解析しインデックスする前のフィールドの値を、Solrに保持するかを設定します。trueにすることで、検索時にヒットした部分を抽出し、さらに検索クエリに含まれる単語をマークアップする[highlighting機能](http://wiki.apache.org/solr/HighlightingParameters)が使えるようになりますが、インデックスサイズは増加します。デフォルトの値はfalseです。
#### termVectors
フィールドの特徴ベクトルを生成するかを指定します。trueにすることで、似ている文書をインデックスから発見する[MoreLikeThis機能](http://wiki.apache.org/solr/MoreLikeThis)の精度が向上しますが、インデックスサイズは増加します。デフォルトの値はfalseです。
#### required
フィールドの値を必須としたいときにtrueに設定します。必ず存在しているべきフィールドに指定することで、もしインポート時にフィールドの値が存在していない場合にエラーにできます。デフォルトの値はfalseです。
#### multiValued
1つの文書に同じフィールドが複数個ひもづく場合、multiValuedを指定します。デフォルトの値はfalseです。

どの機能を使うためにどのオプションを有効にする必要があるかの対応表は、 [FieldOptionsByUseCase - Solr Wiki](http://wiki.apache.org/solr/FieldOptionsByUseCase) にあります。

### solr.xml

Solrが起動したときに最初にロードするcoreの名前と対応する設定ファイルの位置などを設定するのが solr.xml です。