# fluentd

一個分送資料流的中介平台，可以預處理數據後，分送到各自的儲存點（資料庫）

## concept

在k8s中，pod送出的stdout及stderr會記錄在docker log中, fluentd可以用來搜集所有node中的docker log(自定義), 使用共同的pipeline來對log做預處理然後分流（送進不同DB,或是送出alert...等）

- 優點：log集中處理，方便維護
- 缺點：要學習其語法，debug不易


### usage
最常見的方式就是 `source` 收集日誌(e.g. docker log)，然後由串聯的 `filter` 做流式的處理，最後交給 `match` 進行分發

- Source: 所有數據來自哪裡
	- http: 以http的port 收http的訊息
	- forward： 以tcp的port 收tcp的數據包
	- 可以同時進行
- Match: 分發日誌
	- 最好精確匹配
	- 順序很重要
- Filter:對data做加工
	- 一定會有@type: 要加什麼工, e.g:
		- record_transformer: 插入新欄位
		- teams: teams的plugin
		- copy: 複製ｄａｔａ
			- 可以接多個story
	- 有多個plugin可以參考
	
匹配方式：
- a.* -> 匹配`a.b`, `a.c`; 不匹配`a`,`a.b.c`
- a.** -> 匹配 `a`,`a.b`,`a.b.c`
- {a,b} -> 匹配 `a`, `b`
- combine -> a{b,c.**} ...

### fluentd on k8s

通常會是以daemonmset的形式存在在k8s中

## build yaml file for fluentd

以docker-compose為例，把log送進graylog. 

fluentd server:
```
  fluentd-tester:
    build:
      context: .
      dockerfile: fluentd.Dockerfile
    environment:
      FLUENTD_ARGS: --no-supervisor -q
      ENV: dev
      ES_ENDPOINT: http://host.docker.internal:9200
      # SLACK_WEBHOOK:
    volumes:
      - ./input-log:/var/log/input
      - ./fluentd-config:/etc/fluent/config.d
    command: /run.sh
    depends_on:
      - elasticsearch
    networks:
      - test-fluentd-network
```

graylog server:
graylog 會需要elasticsearch及mongodb, mongodb用來記錄graylog的metadata, es是graylog存log的資料庫
```
elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.10
    container_name: graylog-elasticsearch
    restart: unless-stopped
    environment:
      - http.host=0.0.0.0
      - transport.host=localhost
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - network.host=0.0.0.0
    ulimits:
      memlock:
        soft: -1
        hard: -1
    deploy:
      resources:
        limits:
          memory: 1g
    ports:
      - 9200:9200
    networks:
      - test-fluentd-network
  graylog:
    image: graylog/graylog:3.2
    container_name: graylog32-server
    restart: unless-stopped
    environment:
      # - GRAYLOG_PASSWORD_SECRET=somepasswordpepper
      # - GRAYLOG_ROOT_PASSWORD_SHA2=8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
      - GRAYLOG_HTTP_EXTERNAL_URI=http://127.0.0.1:9000/
      - GRAYLOG_MONGODB_URI=mongodb://mongodb:27017/graylog
    depends_on:
      - mongodb
      - elasticsearch
    networks:
      - test-fluentd-network
    ports:
      # Graylog web interface and REST API
      - 9000:9000
      # Syslog TCP
      - 1514:1514
      # Syslog UDP
      - 1514:1514/udp
      # GELF TCP
      - 12201:12201
      # GELF UDP
      - 12201:12201/udp
      # Text
      - 5555:5555
  mongodb:
    image: mongo:3
    container_name: graylog-mongo
    restart: unless-stopped
    networks:
      - test-fluentd-network
```

## issue

1. no patterns matched
```
2021-01-21 11:01:20 +0000 [warn]: no patterns matched tag="kubernetes.....var.log.containers....-86f78646d9-pnplz_...-c158bda01f672a39bedb5a09aa76c3c14920bb08092797ea5bc4e749a24fb7e9.log"
2021-01-21 11:01:20 +0000 [warn]: no patterns matched tag="kubernetes....var.log.containers....-5889b4974f-6j687_...-736cb17924d81d05bc01860d7aebee6b19b2dcc4dfae39747f2be3595986c16e.log"
```
解：下面的match pattern 沒有跟上面給的tag match到

## Reference
- fluentd concept: https://www.itread01.com/content/1549792272.html

