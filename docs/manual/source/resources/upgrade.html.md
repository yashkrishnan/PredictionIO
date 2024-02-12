---
title: Upgrade Instructions
---

<!--
Licensed to the Apache Software Foundation (ASF) under one or more
contributor license agreements.  See the NOTICE file distributed with
this work for additional information regarding copyright ownership.
The ASF licenses this file to You under the Apache License, Version 2.0
(the "License"); you may not use this file except in compliance with
the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

This page highlights major changes in each version and upgrade tools.

# How to Upgrade

## Upgrade to 0.14.0

This release adds Elasticsearch 6 support. See [pull request](https://github.com/apache/predictionio/pull/466) for details.
Consequently, you must reindex your data.

1. Access your old cluster to check existing indices

```
$ curl -XGET 'http://localhost:9200/_cat/indices?v'
health status index     uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   pio_event 6BAPz-DfQ2e9bICdVRr03g   5   1       1501            0    321.3kb        321.3kb
yellow open   pio_meta  oxDMU1mGRn-vnXtAjmifSw   5   1          4            0     32.4kb         32.4kb

$ curl -XGET "http://localhost:9200/pio_meta/_search" -d'
{
  "aggs": {
    "typesAgg": {
      "terms": {
        "field": "_type",
        "size": 200
      }
    }
  },
  "size": 0
}'
{"took":3,"timed_out":false,"_shards":{"total":5,"successful":5,"skipped":0,"failed":0},"hits":{"total":4,"max_score":0.0,"hits":[]},"aggregations":{"typesAgg":{"doc_count_error_upper_bound":0,"sum_other_doc_count":0,"buckets":[{"key":"accesskeys","doc_count":1},{"key":"apps","doc_count":1},{"key":"engine_instances","doc_count":1},{"key":"sequences","doc_count":1}]}}}

$ curl -XGET "http://localhost:9200/pio_event/_search" -d'
{
  "aggs": {
    "typesAgg": {
      "terms": {
        "field": "_type",
        "size": 200
      }
    }
  },
  "size": 0
}'
{"took":2,"timed_out":false,"_shards":{"total":5,"successful":5,"skipped":0,"failed":0},"hits":{"total":1501,"max_score":0.0,"hits":[]},"aggregations":{"typesAgg":{"doc_count_error_upper_bound":0,"sum_other_doc_count":0,"buckets":[{"key":"1","doc_count":1501}]}}}
```

2. (Optional) Settings for new indices

If you want to add specific settings associated with each index, we would recommend defining [Index Templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html).

For example,

```
$ curl -H "Content-Type: application/json" -XPUT "http://localhost:9600/_template/pio_meta" -d'
{
  "index_patterns": ["pio_meta_*"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}'
$ curl -H "Content-Type: application/json" -XPUT "http://localhost:9600/_template/pio_event" -d'
{
  "index_patterns": ["pio_event_*"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}'
```

3. [Reindex](https://www.elastic.co/guide/en/elasticsearch/reference/6.0/reindex-upgrade-remote.html)

According to the following conversion table, you run the reindex every index that you need to migrate to your new cluster.

| Old Cluster | New Cluster |
| --------------- | ---------------- |
| index: `pio_meta` type: `accesskeys` | index: `pio_meta_accesskeys` |
| index: `pio_meta` type: `apps` | index: `pio_meta_apps` |
| index: `pio_meta` type: `channels` | index: `pio_meta_channels` |
| index: `pio_meta` type: `engine_instances` | index: `pio_meta_engine_instances` |
| index: `pio_meta` type: `evaluation_instances` | index: `pio_meta_evaluation_instances` |
| index: `pio_meta` type: `sequences` | index: `pio_meta_sequences` |
| index: `pio_event` type: It depends on your use case. (e.g. `1`) | index: pio_event_<old_type> (e.g. `pio_event_1`) |

For example,

```
$ curl -H "Content-Type: application/json" -XPOST "http://localhost:9600/_reindex" -d'
{
  "source": {
    "remote": {
      "host": "http://localhost:9200"
    },
    "index": "pio_meta",
    "type": "accesskeys"
  },
  "dest": {
    "index": "pio_meta_accesskeys"
  }
}'
```

## Upgrade to 0.12.0

In 0.12.0, Elasticsearch 5.x client has been reimplemented as a singleton.
Engine templates directly using Elasticsearch 5.x StorageClient require
update for compatibility. See [pull request]
(https://github.com/apache/predictionio/pull/421) for details.

## Upgrade to 0.11.0

Starting from 0.11.0, PredictionIO no longer bundles any JDBC drivers in the
binary assembly. If your setup is using a JDBC backend and you run into storage
connection errors after an upgrade, please manually install the JDBC driver. If
you use PostgreSQL, you can find instructions
[here](/install/install-sourcecode#pgsql).

## Upgrade to 0.9.2

The Spark dependency has been upgraded to version 1.3.0. All engines must be
rebuilt against it in order to work.

Open and edit `build.sbt` of your engine, and look for these two lines:

```scala
"org.apache.spark" %% "spark-core"    % "1.2.0" % "provided"

"org.apache.spark" %% "spark-mllib"   % "1.2.0" % "provided"
```

Change `1.2.0` to `1.3.0`, and do a clean rebuild by `pio build --clean`. Your
engine should now work with the latest Apache Spark.


### New PEventStore and LEventStore API

In addition, new PEventStore and LEventStore API are introduced so that appName can be used as parameters in engine.json to access Event Store.

NOTE: The following changes are not required for using 0.9.2 but it's recommended to upgrade your engine code as described below because the old API will be deprecated.

#### 1. In **DataSource.scala**:

- remove this line of code:

    ```scala
    import org.apache.predictionio.data.storage.Storage
    ```

    and replace it by

    ```scala
    import org.apache.predictionio.data.store.PEventStore
    ```

- Change `appId: Int` to `appName: String` in DataSourceParams

    For example,

    ```scala
    case class DataSourceParams(appName: String) extends Params
    ```

- remove this line of code: `val eventsDb = Storage.getPEvents()`

- locate where `eventsDb.aggregateProperties()` is used, change it to `PEventStore.aggregateProperties()`:

    For example,

    ```scala

      val usersRDD: RDD[(String, User)] = PEventStore.aggregateProperties( // CHANGED
        appName = dsp.appName, // CHANGED: use appName
        entityType = "user"
      )(sc).map { ... }

    ```

- locate where `eventsDb.find() `is used, change it to `PEventStore.find()`

    For example,

    ```scala

      val viewEventsRDD: RDD[ViewEvent] = PEventStore.find( // CHANGED
        appName = dsp.appName, // CHANGED: use appName
        entityType = Some("user"),
        ...

    ```

#### 2. In **XXXAlgorithm.scala**:

If Storage.getLEvents() is also used in Algorithm (such as ALSAlgorithm of E-Commerce Recommendation template), you also need to do following:

NOTE: If `org.apache.predictionio.data.storage.Storage` is not used at all (such as Recommendation, Similar Product, Classification, Lead Scoring, Product Ranking template), there is no need to change Algorithm and can go to the later **engine.json** section.

- remove `import org.apache.predictionio.data.storage.Storage` and replace it by `import org.apache.predictionio.data.store.LEventStore`
- change `appId` to `appName` in the XXXAlgorithmParams class.
- remove this line of code: `@transient lazy val lEventsDb = Storage.getLEvents()`
- locate where `lEventsDb.findSingleEntity()` is used, change it to `LEventStore.findByEntity()`:

    For example, change following code

    ```scala
      ...
      val seenEvents: Iterator[Event] = lEventsDb.findSingleEntity(
        appId = ap.appId,
        entityType = "user",
        entityId = query.user,
        eventNames = Some(ap.seenEvents),
        targetEntityType = Some(Some("item")),
        // set time limit to avoid super long DB access
        timeout = Duration(200, "millis")
      ) match {
        case Right(x) => x
        case Left(e) => {
          logger.error(s"Error when read seen events: ${e}")
          Iterator[Event]()
        }
      }
    ```

    to

    ```scala
      val seenEvents: Iterator[Event] = try { // CHANGED: try catch block is used
        LEventStore.findByEntity( // CHANGED: new API
          appName = ap.appName, // CHANGED: use appName
          entityType = "user",
          entityId = query.user,
          eventNames = Some(ap.seenEvents),
          targetEntityType = Some(Some("item")),
          // set time limit to avoid super long DB access
          timeout = Duration(200, "millis")
        )
      } catch { // CHANGED: try catch block is used
        case e: scala.concurrent.TimeoutException =>
          logger.error(s"Timeout when read seen events." +
            s" Empty list is used. ${e}")
          Iterator[Event]()
        case e: Exception =>
          logger.error(s"Error when read seen events: ${e}")
          throw e
      }
    ```

    If you are using E-Commerce Recommendation template, please refer to the latest version for other updates related to `LEventStore.findByEntity()`

#### 3. In **engine.json**:

locate where `appId` is used, change it to `appName` and specify the name of the app instead.

For example:

```json
  ...

  "datasource": {
    "params" : {
      "appName": "MyAppName"
    }
  },

```

Note that other components such as `algorithms` may also have `appId` param (e.g. E-Commerce Recommendation template). Remember to change it to `appName` as well.

That's it! You can re-biuld your engine to try it out!

## Upgrade to 0.9.0

0.9.0 has the following new changes:

- The signature of `P2LAlgorithm` and `PAlgorithm`'s `train()` method is changed from

    ```scala
      def train(pd: PD): M
    ```

    to

    ```scala
      def train(sc: SparkContext, pd: PD): M
    ```

    which allows you to access SparkContext inside `train()` with this new parameter `sc`.

- A new SBT build plugin (`pio-build`) is added for engine template


WARNING: If you have existing engine templates running with previous version of PredictionIO, you need to either download the latest templates which are compatible with 0.9.0, or follow the instructions below to modify them.

Follow instructions below to modify existing engine templates to be compatible with PredictionIO 0.9.0:

1. Add a new parameter `sc: SparkContext` in the signature of `train()` method of algorithm in the templates.

    For example, in Recommendation engine template, you will find the following `train()` function in `ALSAlgorithm.scala`

    ```scala
    class ALSAlgorithm(val ap: ALSAlgorithmParams)
      extends P2LAlgorithm[PreparedData, ALSModel, Query, PredictedResult] {

      ...

      def train(data: PreparedData): ALSModel = ...

      ...
    }
    ```

    Simply add the new parameter `sc: SparkContext,` to `train()` function signature:

    ```scala
    class ALSAlgorithm(val ap: ALSAlgorithmParams)
      extends P2LAlgorithm[PreparedData, ALSModel, Query, PredictedResult] {

      ...

      def train(sc: SparkContext, data: PreparedData): ALSModel = ...

      ...
    }
    ```

    You need to add the following import for your algorithm as well if it is not there:

    ```scala
    import org.apache.spark.SparkContext
    ```

2. Modify the file `build.sbt` in your template directory to use `pioVersion.value` as the version of org.apache.predictionio.core dependency:

    Under your template's root directory, you should see a file `build.sbt` which has the following content:

    ```
    libraryDependencies ++= Seq(
      "org.apache.predictionio"    %% "core"          % "0.8.6" % "provided",
      "org.apache.spark" %% "spark-core"    % "1.2.0" % "provided",
      "org.apache.spark" %% "spark-mllib"   % "1.2.0" % "provided")
    ```

    Change the version of `"org.apache.predictionio" && "core"` to `pioVersion.value`:

    ```
    libraryDependencies ++= Seq(
      "org.apache.predictionio"    %% "core"          % pioVersion.value % "provided",
      "org.apache.spark" %% "spark-core"    % "1.2.0" % "provided",
      "org.apache.spark" %% "spark-mllib"   % "1.2.0" % "provided")
    ```

3. Create a new file `pio-build.sbt` in template's **project/** directory with the following content:

    ```
    addSbtPlugin("org.apache.predictionio" % "pio-build" % "0.9.0")
    ```

    Then, you should see the following two files in the **project/** directory:

    ```
    your_template_directory$ ls project/
    assembly.sbt  pio-build.sbt
    ```

4. Create a new file `template.json` file in the engine template's root directory with the following content:

    ```
    {"pio": {"version": { "min": "0.9.0" }}}
    ```

    This is to specify the minium PredictionIO version which the engine can run with.

5. Lastly, you can add `/pio.sbt` into your engine template's `.gitignore`. `pio.sbt` is automatically generated by `pio build`.

That's it! Now you can run `pio build`, `pio train` and `pio deploy` with PredictionIO 0.9.0 in the same way as before!


##Upgrade to 0.8.4

**engine.json** has slightly changed its format in 0.8.4 in order to make engine more flexible. If you are upgrading to 0.8.4, engine.json needs to have the ```params``` field for *datasource*, *preparator*, and *serving*. Here is the sample engine.json from templates/scala-parallel-recommendation-custom-preparator that demonstrate the change for *datasource* (line 7).


```
In 0.8.3
{
  "id": "default",
  "description": "Default settings",
  "engineFactory": "org.template.recommendation.RecommendationEngine",
  "datasource": {
    "appId": 1
  },
  "algorithms": [
    {
      "name": "als",
      "params": {
        "rank": 10,
        "numIterations": 20,
        "lambda": 0.01
      }
    }
  ]
}
```



```
In 0.8.4
{
  "id": "default",
  "description": "Default settings",
  "engineFactory": "org.template.recommendation.RecommendationEngine",
  "datasource": {
    "params" : {
      "appId": 1
    }
  },
  "algorithms": [
    {
      "name": "als",
      "params": {
        "rank": 10,
        "numIterations": 20,
        "lambda": 0.01
      }
    }
  ]
```



##Upgrade from 0.8.2 to 0.8.3

0.8.3 disallows entity types **pio_user** and **pio_item**. These types are used by default for most SDKs. They are deprecated in 0.8.3, and SDKs helper functions have been updated to use **user** and **item** instead.

If you are upgrading to 0.8.3, you can follow these steps to migrate your data.

##### 1. Create a new app

```
$ pio app new <my app name>
```
Please take note of the <new app id> generated for the new app.

##### 2. Run the upgrade command

```
$ pio upgrade 0.8.2 0.8.3 <old app id> <new app id>
```

It will run a script that creates a new app with the new app id and migreate the data to the new app.

##### 3. Update **engine.json** to use the new app id. **Engine.json** is located under your engine project directory.

```
  "datasource": {
    "appId": <new app id>
  },
```

## Schema Changes in 0.8.2

0.8.2 contains HBase and Elasticsearch schema changes from previous versions. If you are upgrading from a pre-0.8.2 version, you need to first clear HBase and ElasticSearch. These will clear out all data
in Elasticsearch and HBase. Please be extra cautious.

DANGER: **ALL EXISTING DATA WILL BE LOST!**


### Clearing Elasticsearch

With Elasticsearch running, do

```
$ curl -X DELETE http://localhost:9200/_all
```

For details see http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/indices-delete-index.html.

### Clearing HBase

```
$ $HBASE_HOME/bin/hbase shell
...
> disable_all 'predictionio.*'
...
> drop_all 'predictionio.*'
...
```

For details see http://wiki.apache.org/hadoop/Hbase/Shell.

## Experimental upgrade tool (Upgrade HBase schema from 0.8.0/0.8.1 to 0.8.2)

Create an app to store the data

```
$ bin/pio app new <my app>
```

Replace by the returned app ID: ( is the original app ID used in 0.8.0/0.8.2.)

```
$ set -a
$ source conf/pio-env.sh
$ set +a
$ sbt/sbt "data/run-main org.apache.predictionio.data.storage.hbase.upgrade.Upgrade <from app ID>" "<to app ID>"
```
