---
title: ElasticSearch 與 HDFS 的整合
layout: post
category: elasticsearch hadoop hdfs
tags: elasticsearch hadoop hdfs
comments: true
---

本文主要測試 Elasticsearch 與 HDFS 的整合（snapshot/restore）。

## 1. 先把 ElasticSearch 升級到 2.3.1

```
yum update ElasticSearch
```

## 2. 重新編譯及部署 Storm 相關程式 (optional)

```
cd ~/workspace/es-shaed
vim pom.xml # 將 ElasticSearch 版本改成 2.3.1
mvn clean install

cd ~/workspace/LearnStorm
vim src/main/resources/ApLogAnalyzer.properties # 將 es.shield.enabled 改成 false
mvn eclipse:clean eclipse:eclipse # 重新產生 Eclipse 的 .project, .classpath...等檔案
mvn clean package # 重新編譯打包
storm kill ApLogAnalyzerV1 -w 0
storm jar target/LearnStorm-0.0.1-SNAPSHOT.jar com.pic.ala.ApLogAnalyzer
```

## 3. 在所有節點上，關閉 JSM (Java Security Manager)

修改 `/etc/elasticsearch/elasticsearch.yml`：

```yaml
security.manager.enabled: false
```

## 4. 在所有節點上，安裝 `repository-hdfs` plugin 並重啟 ElasticSearch

```bash
# 下載 2.3.4 版
wget "https://oss.sonatype.org/content/repositories/snapshots/org/elasticsearch/elasticsearch-repository-hdfs/2.3.4.BUILD-SNAPSHOT/elasticsearch-repository-hdfs-2.3.4.BUILD-20160803.071811-27-hadoop2.zip"

/usr/share/elasticsearch/bin/plugin install file:///root/tmp/elasticsearch-repository-hdfs-2.3.4.BUILD-20160803.071811-27-hadoop2.zip

# 或下載 2.3.4 light 版，但是記得去修改 ES_CLASSPATH。
wget "https://oss.sonatype.org/content/repositories/snapshots/org/elasticsearch/elasticsearch-repository-hdfs/2.3.4.BUILD-SNAPSHOT/elasticsearch-repository-hdfs-2.3.4.BUILD-20160803.071811-27-light.zip"

```

## 5. 驗證安裝：

```bash
# 5.1 在 HDFS 上建立目錄
sudo hdfs dfs -mkdir /user/elasticsearch
sudo hdfs dfs -mkdir /user/elasticsearch/backup
sudo hdfs dfs -chown elasticsearch:hdfs /user/elasticsearch/
sudo hdfs dfs -chown elasticsearch:hdfs /user/elasticsearch/backup

# 5.2 創建 HDFS repository

cat >> /etc/elasticsearch/elasticsearch.yml <<EOF
repositories:
  hdfs:
    uri: "hdfs://hdp-staging-ha"
    path: "/user/elasticsearch/backup"
    load_defaults: "false"
    conf_location: "/etc/hadoop/conf/hdfs-site.xml,/etc/hadoop/conf/core-site.xml"
    concurrent_streams: 5
    compress: "true"
    user: "elasticsearch"
EOF

curl -XPUT 'http://localhost:9200/_snapshot/backup/?pretty' -d '{
  "type": "hdfs",
    "settings": {
			"uri": "hdfs://hdp-staging-ha",
            "path": "/user/elasticsearch/backup/",
            "conf_location": "/etc/hadoop/conf/hdfs-site.xml,/etc/hadoop/conf/core-site.xml",
            "compress": "true",
            "user": "elasticsearch"
    }
}'

cd /usr/share/elasticsearch/plugins/repository-hdfs
mv hadoop-libs hadoop-libs.old
ln -s /usr/hdp/current/hadoop-hdfs-client/lib hadoop-libs
service elasticsearch restart

# 5.3 察看創建的配置：
curl http://localhost:9200/_snapshot/_all/?pretty

# 5.4 刪除所有 snapshots：
curl -XDELETE "localhost:9200/_snapshot/backup/snapshot_1?pretty&wait_for_completion=true"
curl -XDELETE "localhost:9200/_snapshot/backup/snapshot_2?pretty&wait_for_completion=true"
curl -XDELETE "localhost:9200/_snapshot/backup/snapshot_3?pretty&wait_for_completion=true"
curl -XDELETE "localhost:9200/aplog_*-2016.04.25?pretty"

# 5.4 備份全部索引 => snapshot_1
curl -XPUT "localhost:9200/_snapshot/backup/snapshot_1?pretty&wait_for_completion=true"
curl -XPUT "localhost:9200/_snapshot/backup/20161121?pretty&wait_for_completion=true"

# 5.5 刪除某一個索引（aplog_aes3g-2016.04.12）
curl -XDELETE "localhost:9200/aplog_aes3g-2016.04.12?pretty"

# 5.6 再備份一次全部索引 => snapshot_2
curl -XPUT "localhost:9200/_snapshot/backup/snapshot_2?pretty&wait_for_completion=true"

# 5.7 使用 ApLogTest 程式產生新的索引（aplog_*-2016.04.25）：
ApLogTest

# 5.8 再備份一次全部索引 => snapshot_3
curl -XPUT "localhost:9200/_snapshot/backup/snapshot_3?pretty&wait_for_completion=true"

# 5.9 刪除剛剛產生的新索引（aplog_*-2016.04.25）
curl -XDELETE "localhost:9200/aplog_*-2016.04.25?pretty"

# 5.10 以 snapshot_1（包含 aplog_aes3g-2016.04.12）還原
curl -XPOST 'localhost:9200/*/_close?pretty'
curl -XPOST "localhost:9200/_snapshot/backup/snapshot_1/_restore?pretty&wait_for_completion=true"

# 5.11 以 snapshot_2（不包含 aplog_aes3g-2016.04.12） 還原
curl -XPOST 'localhost:9200/*/_close?pretty'
curl -XPOST "localhost:9200/_snapshot/backup/snapshot_2/_restore?pretty&wait_for_completion=true"

# 5.12 以 snapshot_3（包含 aplog_aes3g-2016.04.12）還原
curl -XPOST 'localhost:9200/*/_close?pretty'
curl -XPOST "localhost:9200/_snapshot/backup/snapshot_3/_restore?pretty&wait_for_completion=true"

# 5.13 注意「aplog_aes3g-2016.04.12」與「aplog_*-2016.04.25」是否已經還原？
curl -XPOST 'localhost:9200/*/_open?pretty'

# 5.14 刪除 snapshot 2
curl -XDELETE "localhost:9200/_snapshot/backup/snapshot_2?pretty"

# 5.15 檢視「backup」裡的所有 snapshots
curl 'http://hdpr01wn01:9200/_snapshot/backup/*?pretty'

# 5.16 刪除「repository-hdfs」plugin
/usr/share/elasticsearch/bin/plugin remove repository-hdfs

# 5.17 將某 index 複製到某 index 的方法：
#      將 aes3g 九月份的 index 全部複製到 aplog_aes3g-2016.09
curl -XPOST 'hdpr01wn01:9200/_reindex?pretty' -d '
{
  "source": {
    "index": "aplog_aes3g-2016.09.*",
    "sort": { "logTime": "desc" }
  },
  "dest": {
    "index": "aplog_aes3g-2016.09"
  }
}'

# 5.18 檢查 doc 數量
curl 'hdpr01wn02:9200/aplog_aes3g-2016.09/_count?pretty'
```

## 6. 其他未測試問題


  1. 備份時，某個 node 或某個 shard 掛掉，會發生什麼狀況？


## 7. 參考文件：

  1. [elasticsearch之hadoop插件使用](http://bigbo.github.io/pages/2015/02/28/elasticsearch_hadoop/)
  2. [elasticsearch-hadoop](https://github.com/elastic/elasticsearch-hadoop/tree/master/repository-hdfs)
  3. [elasticsearch-repository-hdfs](https://oss.sonatype.org/content/repositories/snapshots/org/elasticsearch/elasticsearch-repository-hdfs/)
  4. [Installing Plugins](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/installation.html)
