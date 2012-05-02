# マシンリソース・レプリケーション・冗長化構成
## 必要なマシンリソースの見積もり
### ディスク容量
TextFieldが主体のデータで、かつschema.xml において stored="true" （元のデータをSolrのインデックス内に保持する）と指定している場合、Solrを稼働させるホストではおおむね** 元データの6倍 ** のディスク容量を見積もっておくと安全です。

これは、 schema.xml における TextField型のフィールドのインデキシングの設定で一般的な設定を行った場合
、インデックスは元データの約2倍の容量を消費することと、optimize時に元のインデックスと同じサイズの追加のディスク容量が一時的に必要になるためです。 

    元データのサイズ: N 

    stored=“true“設定による元データの保持に必要な容量: N
    インデックス自体に必要な容量: 2N
    optimizeで一時的に必要になる容量: (N+2N)*2=6N

デフォルトの stored="false" だと約4倍で済みますが、インデックスにデータの追加・更新がある場合も考えて余裕をもって見積もるのが安全です。

### ネットワーク
レプリケーション構成をとった場合、masterがoptimizeを行ったあと最初のレプリケーションは、インデックス全体のネットワーク越しのコピーとなります。インデックスサイズが大きい場合、かなりのネットワーク帯域を消費します。optimize後のレプリケーションを短時間で終わらせるためにも、Solrのmasterとslaveはネットワーク的に出来るだけ近くに配置した方がよいです。

## ベンチマークソフトによる性能の予測

Solrのベンチマークソフトとして[SolrMeter](http://code.google.com/p/solrmeter/)というJava Swing製のツールが利用できます。詳細は割愛します。

### 注意すべき点

SolrMeterを使ってベンチマークをとる場合、ベンチマーク用の検索クエリのバリエーションは、本番環境を想定して十分大きくする必要があります。そうしないと検索結果のデータがディスクキャッシュやSolr自体のLRUキャッシュに載ってしまい、ベンチマークの精度が出ません。

## レプリケーション
データインポート時のレスポンス悪化によるユーザー体験への影響を最小限にするため、可能であれば master/slave構成（レプリケーション構成）をとり、ユーザーの検索リクエストへのレスポンスはおもにslaveが担当することが望ましいです。はてなでは[SolrReplication - Solr Wiki](http://wiki.apache.org/solr/SolrReplication) の [enable/disable master/slave in a node](http://wiki.apache.org/solr/SolrReplication#enable.2BAC8-disable_master.2BAC8-slave_in_a_node)の例をほぼそのまま利用し、起動時のパラメータを以下のようにすることでmaster/slaveの役割動的に設定するようにしています。

### solrconfig.xml 内の/replicationリクエストハンドラ設定箇所

    <requestHandler name="/replication" class="solr.ReplicationHandler" >
      <lst name="master">
        <str name="enable">${enable.master:false}</str>
        <str name="replicateAfter">commit</str>
        <str name="replicateAfter">startup</str>
        <str name="confFiles">schema.xml,dataimport.properties</str>
      </lst>
      <lst name="slave">
        <str name="enable">${enable.slave:false}</str>
        <str name="masterUrl">http://${master.host:localhost}:8983/solr/${solr.core.name}/replication</str>
        <str name="pollInterval">00:00:60</str>
      </lst>
    </requestHandler>

solrconfig.xmlでこう設定しておくと、以下のように HTTPリクエスト時に ** property. ** から始まるパラメータを渡すか、[solrcore.properties という設定ファイルを配置する](http://wiki.apache.org/solr/SolrConfigXml#System_property_substitution)ことで、coreの作成時に solrconfig.xml に変数を渡すことが出来ます。

        curl "http://solrhost02:8983/solr/admin/cores?
          action=CREATE&name=blog_entry&property.enable.slave=true&property.master.host=solrhost01"

このようにすることで、ホストの役割をリモートから動的に決定できます。これは、以降で説明する、Solrの検索サービスを冗長化するときに重要となります。

##  Solrの冗長化構成
バックアップ用ホストを用意してSolrのサービスを冗長化する場合、

1. 普段はホットスタンバイとしてmasterのインデックスを同期しておく
2. 障害が起きた場合、そのホストの役割に応じて、masterにもslaveにもなれる
 
という仕様を満たす必要があります。前述の通り、Solrにはcoreを作成するときに、パラメータ応じてmaster/slaveの役割を切り替える仕組みがあるので、これを利用します。

Solr自体はデーモン化の仕組みも、フェイルオーバーの仕組みを提供していないため、はてなでは daemontools によるデーモン化と keepalived によるフェイルオーバーを組み合わせて運用しています。

### keepalived用failoverスクリプト 
バックアップ用ホストにkeepalivedをインストールした上で、このようなfailoverスクリプトを配置することで、master,slaveのどちらがダウンした場合もバックアップ用ホストが適切な役割で昇格する仕組みになっています。
    #!/bin/bash
    
    # Usage
    # -t SERVICE -c CORE                ... promote the solr to MASTER of <SERVICE>/<CORE>
    # -t SERVICE -c CORE -s MASTER_HOST ... promote the solr to SLAVE of <MASTER_HOST> with <SERVICE>/<CORE>

    BASEDIR="/home/httpd/apps/Hatena-Solr-Admin/releases/solr"

    usage()
    {
    cat <<EOF
    usage: $0 options
    OPTIONS:
      -t SERVICE (required)
      -c CORE (required)
      -s MASTER_HOST
      -m
    EOF
    }
    
    promote_slave()
    {
        echo -e "Promote the solr to SLAVE: $MASTER_HOST of $SERVICE/$CORE"
        curl "http://localhost:8983/solr/admin/cores?action=CREATE&name=$CORE&instanceDir=$BASEDIR/$SERVICE/$CORE&property.enable.slave=true&property.master.host=$MASTER_HOST"
    }

    promote_master()
    {
        echo -e "Promote the solr to MASTER of $SERVICE/$CORE"
        curl "http://localhost:8983/solr/admin/cores?action=CREATE&name=$CORE&instanceDir=$BASEDIR/$SERVICE/$CORE&property.enable.master=true"
    }
    
    while getopts "ht:c:s:m" OPTION; do
        case $OPTION in
    
            h)
                usage
                exit 1
                ;;
            t)
                SERVICE=$OPTARG
                ;;
            c)
                CORE=$OPTARG
                ;;
            s)
                MASTER_HOST=$OPTARG
                promote_slave
                exit 0
                ;;
            m)
                promote_master
                exit 0
                ;;
            ?)
                echo -e "Invalid option"
                usage
                exit 1
                ;;
        esac
    done
