
# Pulsar ジオレプリケーション

<!-- TOC depthFrom:2 depthTo:3 withLinks:1 updateOnSave:1 orderedList:0 -->

- [レプリケーションモデル](#レプリケーションモデル)
- [レプリケーション設定](#レプリケーション設定)
	- [プロパティへの権限付与](#プロパティへの権限付与)
	- [グローバルネームスペースの作成](#グローバルネームスペースの作成)
	- [グローバルトピックの利用](#グローバルトピックの利用)

<!-- /TOC -->

## レプリケーションモデル

複数のクラスタを設定されたプロパティはそれらのクラスタ間においてレプリケーションを有効にできます。レプリケーションはネームスペースレベルで利用者が直接管理します。グローバルネームスペース内のすべてのトピックがレプリケートされます。利用者はグローバルネームスペースを作成し、2つあるいはそれ以上のクラスタ間でレプリケートされるよう設定します。そのネームスペース内のトピックに発行されたすべてのメッセージは、設定したすべてのクラスタにレプリケートされます。

メッセージはまずローカルクラスタに永続化され、そして非同期にリモートクラスタに転送されます。

通常時、接続の問題がない場合、メッセージはローカルのConsumerに配信されると同時に即座にレプリケートされます。一般的にend-to-endの配送にかかるレイテンシはリモートの地域間のネットワーク*RTT* (Round Trip Time; 往復遅延時間) によって定義されます。

アプリケーションはリモートクラスタが到達不可能 (例. ネットワークが分離されている状況) であっても、ProducerとConsumerをどのクラスタにも作ることができます。

サブスクリプションはそれが作成されたクラスタ内に閉じており、クラスタ間で転送されることはありません。

![Replication Diagram](../../img/GeoReplication.png)

上の例ではトピックは***Cluster-A***, ***Cluster-B***, ***Cluster-C***という3つのクラスタにレプリケートされます。

任意のクラスタにproduceされたすべてのメッセージはすべてのクラスタのすべてのサブスクリプションに配送されます。この例では、***C1***と***C2***の両方は***P1***, ***P2***, ***P3***に発行されたすべてのメッセージを受け取ります。Producerごとのメッセージの順序は保証されます。

## レプリケーション設定

### プロパティへの権限付与

クラスタ間のレプリケーションを行うため、テナントにクラスタを利用する権限が必要です。この権限はプロパティ作成時に付与するか、後で更新できます。

作成時、利用したいクラスタを指定します:
```shell
$ bin/pulsar-admin properties create my-property    \
        --admin-roles my-admin-role                 \
        --allowed-clusters us-west,us-east,us-cent
```

既存のプロパティの権限を更新する際は、`create`の代わりに`update`を使用します。

### グローバルネームスペースの作成

レプリケーションを利用するためには*グローバル*トピック、すなわちグローバルネームスペースに属し、特定のクラスタに紐付けられていないトピックが必要となります。

グローバルネームスペースは***global***仮想クラスタの中に作成される必要があります:

```shell
$ bin/pulsar-admin namespaces create my-property/global/my-namespace
```

最初はネームスペースはどのクラスタにも紐付けられていません。

```shell
$ bin/pulsar-admin namespaces set-clusters
                          my-property/global/my-namespace \
                          --clusters us-west,us-east,us-cent
```

ネームスペースのレプリケーション先クラスタは進行中のトラフィックを中断することなく、いつでも変更が可能です。設定が変更されると即座にすべてのクラスタのレプリケーションチャンネルはセットアップあるいは停止されます。

### グローバルトピックの利用

この時点でProducerまたはConsumerを単に作成すれば、自動的にグローバルトピックを作成できます。

一般的に各アプリケーションはローカルクラスタの`serviceUrl`を使用します。

#### 選択的レプリケーション

デフォルトでは、メッセージはネームスペースに設定されているすべてのクラスタにレプリケートされます。メッセージのレプリケーションリストを指定することでレプリケーション先を制限できます。するとメッセージは一部のクラスタのみにレプリケートされます。

Java APIでは、これは`MessageBuilder`において行われます:
```java
...
List<String> restrictReplicationTo = new ArrayList<>();
restrictReplicationTo.add("us-west");
restrictReplicationTo.add("us-east");

Message message = MessageBuilder.create()
              .setContent("my-payload".getBytes())
              .setReplicationClusters(restrictReplicationTo)
              .build();

producer.send(message);
```

#### トピックの統計情報

グローバルトピックの統計情報は通常のトピックと同じAPIとツールによって取得できます:

```shell
$ bin/pulsar-admin persistent stats persistent://my-property/global/my-namespace/my-topic
```

各クラスタは入力/出力のレプリケーションのレートおよびバックログを含む自身の統計情報をレポートします。

#### グローバルトピックの削除

複数の地域に存在するグローバルトピックを直接削除はできません。削除の処理は自動的に行われるトピックのガベージコレクションに依存します。

Pulsarではトピックがもう利用されていないとき、すなわちProducerまたはConsumerが一つも接続しておらず、***かつ***サブスクリプションが存在しないときに自動的に削除されます。

グローバルトピックにおいて、各地域はフォールトトレラントな仕組みを使用して、そのトピックをいつローカルで削除するのが安全かを判断します。

グローバルトピックを削除するために、そのトピックのすべてのProducerとConsumerを閉じ、すべてのレプリケーション先クラスタにおけるローカルなサブスクリプションを削除します。Pulsarがシステム内に有効なサブスクリプションが残っていないと判断すると、そのトピックをガベージコレクトします。