# IoT Docker compose
<!-- > ให้นำไฟล์ docker-compose.yaml มาอธิบายว่า แต่ละส่วนคืออะไร โดยใช้การ comment ในไฟล์ docker-compose.yaml -->
### docker-compose.yaml
```yaml
services:
    # zookeeper เป็นระบบจัดการและประสานงานแบบกระจาย (distributed coordination service) โดยใช้ในระบบที่มีความซับซ้อนและต้องการความเสถียรสูง เช่น Apache Kafka, Hadoop, และ Hbase
    zookeeper:
        image: confluentinc/cp-zookeeper
        container_name: zookeeper
        restart: unless-stopped
        volumes:
        - zookeeper-data:/var/lib/zookeeper/data
        - zookeeper-log:/var/lib/zookeeper/log
        environment:
        ZOOKEEPER_CLIENT_PORT: 2181
        ZOOKEEPER_LOG4J_ROOT_LOGLEVEL: INFO
        ZOOKEEPER_LOG4J_PROP: INFO,ROLLINGFILE
        ZOOKEEPER_LOG_MAXFILESIZE: 10MB
        ZOOKEEPER_LOG_MAXBACKUPINDEX: 10
        ZOOKEEPER_SNAP_COUNT: 10
        ZOOKEEPER_AUTOPURGE_SNAP_RETAIN_COUNT: 10
        ZOOKEEPER_AUTOPURGE_PURGE_INTERVAL: 3
    
    # Kafka คือระบบจัดคิวข้อความแบบกระจาย (distributed message queue) ที่สามารถรับส่งข้อมูลในปริมาณมากได้อย่างมีประสิทธิภาพ Kafka เหมาะสำหรับระบบที่ต้องการความสามารถในการส่งข้อมูลระหว่างแอปพลิเคชันหลายตัวในรูปแบบที่เสถียรและปรับขนาดได้
    kafka:
        image: confluentinc/cp-kafka
        container_name: kafka
        volumes:
        - kafka-data:/var/lib/kafka
        restart: unless-stopped
        environment:
        # Required. Instructs Kafka how to get in touch with ZooKeeper.
        KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
        KAFKA_NUM_PARTITIONS: 1
        KAFKA_COMPRESSION_TYPE: gzip
        # Required when running in a single-node cluster, as we are. We would be able to take the default if we had
        # three or more nodes in the cluster.
        KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
        KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
        KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
        # Required. Kafka will publish this address to ZooKeeper so clients know
        # how to get in touch with Kafka. "PLAINTEXT" indicates that no authentication
        # mechanism will be used.
        KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
        KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
        links:
        - zookeeper

    # kafka-rest-proxy เป็นเครื่องมือหรืออินเทอร์เฟซที่ช่วยให้ผู้ใช้สามารถสื่อสารกับ Kafka brokers ผ่าน HTTP API แทนที่จะใช้ Kafka client โดยตรง ซึ่งทำให้การเชื่อมต่อกับ Kafka ง่ายขึ้นสำหรับแอปพลิเคชันหรือบริการที่ไม่รองรับ Kafka client libraries หรือไม่ได้เขียนด้วยภาษาที่ Kafka รองรับโดยตรง
    kafka-rest-proxy:
        image: confluentinc/cp-kafka-rest:latest
        container_name: kafka-rest-proxy
        environment:
        # Specifies the ZooKeeper connection string. This service connects
        # to ZooKeeper so that it can broadcast its endpoints as well as
        # react to the dynamic topology of the Kafka cluster.
        KAFKA_REST_ZOOKEEPER_CONNECT: zookeeper:2181
        # The address on which Kafka REST will listen for API requests.
        KAFKA_REST_LISTENERS: http://0.0.0.0:8082/
        # Required. This is the hostname used to generate absolute URLs in responses.
        # It defaults to the Java canonical hostname for the container, which might
        # not be resolvable in a Docker environment.
        KAFKA_REST_HOST_NAME: kafka-rest-proxy
        # The list of Kafka brokers to connect to. This is only used for bootstrapping,
        # the addresses provided here are used to initially connect to the cluster,
        # after which the cluster will dynamically change. Thanks, ZooKeeper!
        KAFKA_REST_BOOTSTRAP_SERVERS: kafka:9092
        # Kafka REST relies upon Kafka, ZooKeeper
        # This will instruct docker to wait until those services are up
        # before attempting to start Kafka REST.
        restart: unless-stopped
        ports:
        - "9999:8082"
        depends_on:
        - zookeeper
        - kafka

    # kafka-connect เป็นเครื่องมือที่ทำหน้าที่เชื่อมต่อและแปลงข้อมูลระหว่างระบบต้นทาง (source systems) และระบบปลายทาง (target systems) โดยไม่ต้องสร้าง producer และ consumer ขึ้นมาเอง Kafka Connect ช่วยลดความซับซ้อนในการทำงานกับข้อมูลจำนวนมากที่อยู่ใน Kafka
    kafka-connect:
        image: confluentinc/cp-kafka-connect:latest
        hostname: kafka-connect
        container_name: kafka-connect
        environment:
        # Required.
        # The list of Kafka brokers to connect to. This is only used for bootstrapping,
        # the addresses provided here are used to initially connect to the cluster,
        # after which the cluster can dynamically change. Thanks, ZooKeeper!
        CONNECT_BOOTSTRAP_SERVERS: "kafka:9092"
        # Required. A unique string that identifies the Connect cluster group this worker belongs to.
        CONNECT_GROUP_ID: kafka-connect-group
        # Connect will actually use Kafka topics as a datastore for configuration and other data. #meta
        # Required. The name of the topic where connector and task configuration data are stored.
        CONNECT_CONFIG_STORAGE_TOPIC: kafka-connect-meta-configs
        # Required. The name of the topic where connector and task configuration offsets are stored.
        CONNECT_OFFSET_STORAGE_TOPIC: kafka-connect-meta-offsets
        # Required. The name of the topic where connector and task configuration status updates are stored.
        CONNECT_STATUS_STORAGE_TOPIC: kafka-connect-meta-status
        # Required. Converter class for key Connect data. This controls the format of the
        # data that will be written to Kafka for source connectors or read from Kafka for sink connectors.
        CONNECT_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
        # Required. Converter class for value Connect data. This controls the format of the
        # data that will be written to Kafka for source connectors or read from Kafka for sink connectors.
        CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
        # Required. The hostname that will be given out to other workers to connect to.
        CONNECT_REST_ADVERTISED_HOST_NAME: "kafka-connect"
        CONNECT_REST_PORT: 8083
        # The next three are required when running in a single-node cluster, as we are.
        # We would be able to take the default (of 3) if we had three or more nodes in the cluster.
        CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: "1"
        CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: "1"
        CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: "1"
        #Connectos path
        CONNECT_PLUGIN_PATH: "/usr/share/java,/data/connectors/"
        CONNECT_LOG4J_ROOT_LOGLEVEL: "INFO"
        restart: unless-stopped
        volumes:
        - ./kafka_connect/data:/data
        command: 
        - bash 
        - -c 
        - |
            echo "Launching Kafka Connect worker"
            /etc/confluent/docker/run & 
            #
            echo "Waiting for Kafka Connect to start listening on http://$$CONNECT_REST_ADVERTISED_HOST_NAME:$$CONNECT_REST_PORT/connectors ⏳"
            while [ $$(curl -s -o /dev/null -w %{http_code} http://$$CONNECT_REST_ADVERTISED_HOST_NAME:$$CONNECT_REST_PORT/connectors) -ne 200 ] ; do 
            echo -e $$(date) " Kafka Connect listener HTTP state: " $$(curl -s -o /dev/null -w %{http_code} http://$$CONNECT_REST_ADVERTISED_HOST_NAME:$$CONNECT_REST_PORT/connectors) " (waiting for 200)"
            sleep 5 
            done
            nc -vz $$CONNECT_REST_ADVERTISED_HOST_NAME $$CONNECT_REST_PORT
            echo -e "\n--\n+> Creating Kafka Connect MongoDB sink Current PATH ($$PWD)"
            /data/scripts/create_mongo_sink.sh 
            echo -e "\n--\n+> Creating MQTT Source Connect Current PATH ($$PWD)"
            /data/scripts/create_mqtt_source.sh
            echo -e "\n--\n+> Creating Kafka Connect Prometheus sink Current PATH ($$PWD)"
            /data/scripts/create_prometheus_sink.sh
            sleep infinity
        # kafka-connect relies upon Kafka and ZooKeeper.
        # This will instruct docker to wait until those services are up
        # before attempting to start kafka-connect.
        depends_on:
        - zookeeper
        - kafka
    
    # mosquitto เป็นซอฟต์แวร์โอเพ่นซอร์สที่ทำหน้าที่เป็น MQTT broker ซึ่งเป็นโปรโตคอลที่ใช้ในการสื่อสารระหว่างอุปกรณ์ IoT ด้วยการออกแบบที่มีประสิทธิภาพและใช้ทรัพยากรน้อย ทำให้ Eclipse Mosquitto เหมาะสำหรับงานที่เกี่ยวกับ IoT
    mosquitto:
        image: eclipse-mosquitto:latest
        hostname: mosquitto
        container_name: mosquitto
        restart: unless-stopped
        ports:
        - "1883:1883"
        - "9001:9001"
        volumes:
        - ./mosquitto/config:/mosquitto/config
        - ./mosquitto/data:/mosquitto/data
        - ./mosquitto/log:/mosquitto/log
    
    # mongo เป็นฐานข้อมูล NoSQL ที่ได้รับความนิยมซึ่งออกแบบมาให้จัดเก็บข้อมูลในรูปแบบที่ยืดหยุ่น MongoDB เหมาะสำหรับการจัดการข้อมูลปริมาณมาก โดยนักพัฒนาสามารถปรับแต่งการทำงานของระบบเพื่อสร้างแอปพลิเคชันที่ทันสมัยและขับเคลื่อนด้วยข้อมูลได้อย่างง่ายดาย
    mongo:
        image: mongo:4.4.20
        container_name: mongo
        env_file:
        - .env
        restart: unless-stopped
        environment:
        - MONGO_INITDB_ROOT_USERNAME=${MONGO_ROOT_USER}
        - MONGO_INITDB_ROOT_PASSWORD=${MONGO_ROOT_PASSWORD}
        - MONGO_INITDB_DATABASE=${MONGO_DB}

    # grafana เป็นเครื่องมือสร้างแดชบอร์ดที่สามารถแสดงข้อมูลจากเมตริกต่าง ๆ ในรูปแบบกราฟแบบเรียลไทม์ Grafana รองรับการดึงข้อมูลจากแหล่งข้อมูลยอดนิยม เช่น Prometheus, InfluxDB, Elasticsearch, AWS CloudWatch
    grafana:
        image: grafana/grafana:latest-ubuntu
        container_name: grafana
        user: '0'
        volumes:
        - ./grafana/data:/var/lib/grafana
        - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
        - ./grafana/datasources:/etc/grafana/provisioning/datasources
        - ./grafana/data/plugins:/var/lib/grafana/plugins

        environment:
        - GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin}
        - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
        - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-worldmap-panel,grafana-piechart-panel
        - GF_USERS_ALLOW_SIGN_UP=false
        - GF_SECURITY_ANGULAR_SUPPORT_ENABLED=True
        - GF_FEATURE_TOGGLES_ANGULARDEPRECATIONUI=False
        restart: unless-stopped
        links:
        - prometheus
        ports:
        - '8085:3000'

    # prometheus เป็นระบบมอนิเตอร์ริ่งและเตือนภัยแบบโอเพ่นซอร์สที่ออกแบบมาเพื่อรวบรวมและจัดเก็บข้อมูลเมตริกในรูปแบบ time series data เหมาะสำหรับการมอนิเตอร์ระบบและแอปพลิเคชันต่าง ๆ โดยมีการทำงานแบบกระจายและมีความยืดหยุ่นสูง Prometheus สามารถทำงานร่วมกับ Grafana เพื่อสร้างกราฟแสดงข้อมูลแบบเรียลไทม์
    prometheus:
        image: prom/prometheus:latest
        container_name: prometheus
        volumes:
        - ./prometheus/:/etc/prometheus/
        - prometheus_data:/prometheus
        command:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--storage.tsdb.path=/prometheus'
        - '--web.console.libraries=/etc/prometheus/console_libraries'
        - '--web.console.templates=/etc/prometheus/consoles'
        - '--storage.tsdb.retention.time=200h'
        - '--web.enable-lifecycle'
        restart: unless-stopped
        ports:
        - '8086:9090'

    # iot-processor เป็นการประมวลผลข้อมูลที่ได้จากเซ็นเซอร์ ข้อมูลที่เก็บมาอาจจะถูกส่งไปยังระบบประมวลผลหรือเซิร์ฟเวอร์กลาง การประมวลผลอาจรวมถึง
    iot-processor:
        image: ssanchez11/iot_processor:0.0.1-SNAPSHOT
        container_name: iot-processor
        restart: unless-stopped
        ports:
        - '8080:8080'
        depends_on:
        kafka-connect:
            condition: service_started
            restart: true

    # IoT sensor 1 เป็น sensor ที่ถูกจําลองด้วยไมโครเซอร์วิสที่ใช้ใน Spring Boot (ผ่านไลบรารี Eclipse Paho MQTT) ที่ถูกติดตั้งอยู่บนเซิฟเวอร์ โดยจะส่งข้อมูล telemetry ไปยังโบรกเกอร์ Eclipse Mosquitto ข้อมูลที่ถูกจำลองนี้ generate ค่า ทุกอย่างภายใน payload มาจาก Callable โดยจะถูกสร้างขึ้นทุกวินาทีและ มี payload ในรูปแบบที่สร้างขึ้นให้ตรงกัน
    iot_sensor_1:
        image: ssanchez11/iot_sensor:0.0.1-SNAPSHOT
        build:
        context: ./microservices/iot_sensor
        args:
            - MQTT_SERVER=${IOT_SENSOR_1_ID}
        container_name: iot_sensor_1
        restart: unless-stopped
        environment:
        - sensor.id=${IOT_SENSOR_1_ID}
        - sensor.name=${IOT_SENSOR_1_NAME}
        - sensor.place.id=${IOT_SENSOR_1_PLACE_ID}
        - sensor.mqtt.username=${IOT_SENSOR_1_USERNAME}
        - sensor.mqtt.password=${IOT_SENSOR_1_PASSWORD}
        - MQTT_SERVER=${MQTT_SERVER}
        depends_on:
        iot-processor:
            condition: service_started
            restart: true
```
<!-- zookeeper kafka kafka-rest-proxy kafka-connect mosquitto mongo grafana prometheus iot-processor iot_sensor_1 -->
<!-- > อธิบายว่า  หน้าจอที่ 1 ที่ต้องเปิดใช้งาน มีการเปิด service อะไรบ้างใน docker -->
## start-service #0
```bash
sh start_0zookeeper_kafka.sh
```
#### service
* [zookeeper](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#apache-zookeeper)
* [kafka](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#apache-kafka)

> [!NOTE]
> เมื่อรันคำสั่ง รอจนกว่า service kafka zookeeper จะนิ่งถึงจะรัน `start-service #1` ต่อไป


## start-service #1
```bash
sh start_1kafka_service.sh
```
#### service
* [kafka-rest-proxy](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#apache-kafka-rest-proxy)
* [kafka-connect](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#apache-kafka-connect)
* [mosquitto](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#eclipse-mosquitto)
* [mongo](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#mongodb)
* [grafana](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#grafana)
* [prometheus](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#prometheus)

> [!NOTE]
> เมื่อรันคำสั่ง รอจนกว่า terminal จะแสดง 
`kafka-connect Kafka Connect listener HTTP state:  000  (waiting for 200)` ถึงจะรัน `start-service #2` ต่อไป

## start-service #2
```bash
sh start_2iot_processor.sh
```
#### service
* [iot-processor](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#iot-processor)

> [!NOTE]
> รอจนกว่า iot-processor ขึ้น initialize ถึงจะรัน `start-service #3` ต่อไป

> [!WARNING]
> ถ้าขึ้น `iot-processor Shutdown complete` ให้ restart iot-processor 

## start-service #3
```bash
sh start_3iot_sensor.sh
```
#### service
* [iot_sensor](https://github.com/Thanabodin19/iotclass67/blob/main/assignment00/architecture.md#iot-sensor)


<!-- > [!NOTE]
> This is a note.

> [!TIP]
> This is a tip. (Supported since 14 Nov 2023)

> [!IMPORTANT]
> Crutial information comes here.

> [!CAUTION]
> Negative potential consequences of an action. (Supported since 14 Nov 2023)

> [!WARNING]
> Critical content comes here. -->

