<h1 align="center">kafka-lag-dashboard</h1>
<p align="center">
  <strong>Kafka lag을 모니터링하는 확실한 방법</strong>
</p>

Kafka Consumer의 처리시간이 지연되면 topic 내부의 partition lag이 증가합니다. lag 모니터링을 통해 어느 partition이 lag이 증가하고 있는지, 어느 컨슈머가 문제가 있는지 확인하기 위해서는 consumer단위의 metric 모니터링으로는 해결하기 쉽지 않습니다. 그렇기 때문에 카프카 컨슈머 모니터링을 위해서는 burrow와 같은 외부 모니터링 tool 사용을 권장합니다.


<p align="center">
    <img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/kafka_logo.png" width="55px" />
    <img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/telegraf_logo.png" width="50px" /> 
    <img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/es_logo.png" width="50px" /> 
    <img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/grafana_logo.png" width="50px" />
</p>

이 문서에서는 Linkedin에서 제공한 burrow를 사용하여 lag정보를 Elasticsearch로 수집하는 데이터파이프라인을 만들어보고, **Grafana 기반의 consumer단위 lag 모니터링 대시보드** 를 만드는 방법을 알려드리고자 합니다. 또한 lag증가에 따른 Slack alert를 받는 기능도 구현해보도록 하겠습니다. 

궁금한점이나 고칠점이 있다면 repository의 [issues](https://github.com/AndersonChoi/kafka-lag-dashboard/issues)에 남겨주시거나 fork 후 PR 부탁드립니다. 만약 이 문서가 도움되셨다면 :star: 를 눌러주세요.

> 이 repository에 대한 라이센스는 없습니다. 누구든지 사용하여 카프카 lag모니터링에 도움이 되길 바랍니다.

## 준비물

Partition lag을 모니터링하고 alert를 받기 위해 아래와 같은 기술 stack을 사용합니다. 아래 기술 stack은 무료로 사용 가능하면서도 필요한기능들(alert, pipeline 등)을 효과적으로 적용가능하기 때문에 고르게 되었습니다. 필요에 따라 기술을 대체할 수 있으므로, 요구사항에 따라 바꿔 사용하셔도 좋습니다. (예를 들어, Grafana → Elasticsearch X-Pack) 

- Burrow([link](https://github.com/linkedin/Burrow)) : linkedin에서 공개한 opensource lag monitoring application입니다. rest api를 통해 lag 정보를 전달받을 수 있습니다.
- Telegraf([link](https://github.com/influxdata/telegraf)) : 데이터의 수집 및 전달에 특화된 agent입니다. configuration설정을 통해 burrow의 데이터를 ES에 넣는 역할을 합니다.
- Elasticsearch([link](https://github.com/elastic/elasticsearch)) : kafka lag 데이터를 저장하는 역할을 합니다. 
- Grafana([link](https://github.com/linkedin/Burrow)) : ES의 데이터를 시각화하고 threshold를 설정하여 slack alert를 보낼 수 있는 강력한 대시보드 tool입니다.

> 상기 준비물의 설치 방법 및 사용방법은 이 문서에서 다루지 않습니다. 각 기술의 링크를 참고하여 설치 부탁드립니다.

## 설정

### 1) Burrow 설정

Burrow는 kafka 클러스터와 연동되어 lag정보를 rest api를 통해 조회할 수 있습니다. Burrow 그 자체로도 alert를 보낼 수 있는 기능이 제공되지만 장기적으로 봤을 때 데이터가 쌓이지 않기 때문에 alert이전 내용에 대한 분석이 매우 어렵습니다. 그러므로 burrow설정에서는 기본적인 kafka와의 연동만 하도록 합니다.

만약 kafka가 SASL 등의 보안정책을 가지고 있거나 추가 설정이 필요하신 분은 [이 링크](https://github.com/linkedin/Burrow/wiki/Configuration)를 참고하시기 바랍니다.

**burrow.toml**

```
[zookeeper]
servers=["주키퍼01:2181","주키퍼02:2181","주키퍼03:2181"] 
timeout=6
root-path="/burrow"

[cluster.live]
class-name="kafka"
servers=["카프카01:9092","카프카02:9092","카프카03:9092"]
topic-refresh=120
offset-refresh=30

[consumer.live]
class-name="kafka"
cluster="live"
servers=["카프카01:9092","카프카02:9092","카프카03:9092"]
group-blacklist="^(console-consumer-|quick-).*$"
group-whitelist=""

[consumer.live_zk]
class-name="kafka_zk"
cluster="live"
servers=["주키퍼01:2181","주키퍼02:2181","주키퍼03:2181"] 
zookeeper-timeout=30
group-blacklist="^(console-consumer-|quick-).*$"
group-whitelist=""

[httpserver.default]
address=":8000"
```

### 2) Telegraf 설정

Telegraf는 agent application으로서 burrow의 rest api데이터를 일정주기로 ES에 전달하는 역할을 합니다.

**telegraf.conf**

```
[[inputs.burrow]]
  servers = ["http://버로우:8000"]
  topics_exclude = [ "__consumer_offsets" ]
  groups_exclude = ["console-*"]

[[outputs.elasticsearch]]
  urls = [ "http://엘라스틱서치:9200" ] 
  timeout = "5s"
  enable_sniffer = false
  health_check_interval = "10s"
  index_name = "burrow-%Y.%m.%d" 
  manage_template = false
```

### 3) Elasticsearch 설정

Elasticsearch에 수집된 document들의 index pattern을 정의해야합니다. Kibana를 통해 아래와 같이 패턴을 정의할 수 있습니다.

<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/burrow_index_pattern_setting.png" width="70%" height="30%" title="es 설정" alt="es 설정"></img>
</p>

### 4) Grafana 설정

Elasticsearch와 연동을 위해서 아래와 같이 Datasource 연동이 필요합니다.

**Grafana Datasorce 추가**
<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/garfana_datastore_setting.png" width="70%" title="그라파나 datastore 추가" alt="그라파나 datastore 추가"></img>
</p>

**Datasource에 Elasticsearch, index pattern 정보 추가**

<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/burrow_es_datastore_setting.png" width="50%" title="datastore에 es추가" alt="datastore에 es추가"></img>
</p>

## 결과물


### Kafka lag 조회

위 설정까지 끝내셨다면 lag을 모니터링할 준비가 완료되었습니다. 이제 Grafana 대시보드에서 그래프를 만들어서 Elasticsearch의 데이터를 기반으로 그래프를 그려보겠습니다.

**그래프 설정**
<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/grafana_graph_setting.png" width="70%" height="30%" title="그래프그리기 위한 query" alt="그래프그리기 위한 query"></img>
</p>

burrow에서 수집한 데이터 중 measure_ment_name이 **burrow_partition** 인것만 대상으로 lag을 수집합니다. topic, groupId, partition 별로 group by로 묶어주시면 topic, groupId, partition별 lag추이를 확인할 수 있습니다.

**그래프 확인**
<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/grafana_graph.png" width="70%" height="30%" title="그래프 스크린샷" alt="그래프 스크린샷"></img>
</p>

### Kakfa lag slack alert 설정

Grafana에는 alert기능이 준비되어 있습니다. Alert는 Slack뿐만아니라 Line, Telegram, webhook, email 등 다양한 방식의 alert를 제공하고 있으므로 필요에 따라 사용하고 싶은 alert를 사용하셔도 좋습니다.

**Slack url경로 추가**
<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/grafana_setting_slack_alert.png" width="70%" height="30%" title="grafana에 slack token 적용" alt="grafana에 slack token 적용"></img>
</p>

Slack과 연동하기 위해서는 url을 발급받아야 합니다. webhook url을 발급받는 방법은 [이 링크](https://api.slack.com/messaging/webhooks)를 확인해주세요.

**그래프 alert 설정**
<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/grafana_setting_alert_graph.png" width="70%" height="30%" title="그래프 알람 설정" alt="그래프 알람 설정"></img>
</p>

Alert를 받고 싶은 그래프에서 Alert configuration을 추가합니다. 위 쿼리는 1분마다, 지난 60초간 lag의 평균값이 10 이상일 경우에 Slack 메시지를 보내도록 설정하였습니다. 필요에 따라 설정을 변경하여 적용하면 lag 모니터링시 유용합니다.

**이상징후 발생**
<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/grafana_graph_alert.png" width="70%" height="30%" title="그래프 이상감지" alt="그래프 이상감지"></img>
</p>

**Slack 메시지**
<p align="center">
<img src="https://raw.githubusercontent.com/AndersonChoi/kafka-lag-dashboard/master/images/slack_alert_message.png" width="70%" height="30%" title="slack 알람 스크린샷" alt="slack 알람 스크린샷"></img>
</p>

