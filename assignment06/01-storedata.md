# Store data.
## Kafka to MongoDB 

### Overview
ขั้นตอนนี้มีจุดประสงค์ในการย้ายข้อมูลที่ประมวลผลแล้วจากหัวข้อ Kafka ไปยังคอลเลกชันที่เกี่ยวข้องใน MongoDB เพื่อให้สามารถดูและวิเคราะห์ข้อมูลผ่านเครื่องมือ เช่น MongoDB-Express ได้ในภายหลัง

ได้มีการตั้งค่า **MongoDBSinkConnector** จำนวน 3 อินสแตนซ์ โดยแต่ละอินสแตนซ์จะดึงข้อมูลจากหัวข้อ Kafka และนำข้อมูลไปจัดเก็บในคอลเลกชันที่กำหนดไว้ใน MongoDB ดังนี้:

1. ย้ายระเบียนต้นฉบับจากหัวข้อ `iot-frames` ไปยังคอลเล็กชัน `iot_frames` ในฐานข้อมูล `iot`
2. ย้ายข้อมูลจากหัวข้อ `iot-aggregate-metrics-sensor` ไปยังคอลเล็กชัน `iot_aggregate_metrics_sensor`
3. ย้ายข้อมูลรวมตามสถานที่จากหัวข้อ `iot-aggregate-metrics-place` ไปยังคอลเล็กชัน `iot_aggregate_metrics_place`

### Configuration

#### 1. MongoDB Sink Connector for `iot-frames`
ตัวเชื่อมต่อนี้จะย้ายข้อมูลจากหัวข้อ `iot-frames` ไปยังคอลเล็กชัน `iot_frames` ใน MongoDB โดยใช้ข้อมูลในรูปแบบ JSON และไม่จำเป็นต้องใช้ schema (เช่น Avro หรือ JSON-Schema) เพื่อลดความซับซ้อนของกระบวนการ

```json
{
   "name":"iot-frames-mongodb-sink",
   "config":{
      "connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
      "tasks.max":1,
      "topics":"iot-frames",
      "connection.uri":"mongodb://devroot:devroot@mongo:27017",
      "database":"iot",
      "collection":"iot_frames",
      "value.converter": "org.apache.kafka.connect.json.JsonConverter",
      "key.converter": "org.apache.kafka.connect.json.JsonConverter",
      "value.converter.schemas.enable": false,
      "key.converter.schemas.enable":false
   }
}
```
#### 2. MongoDB Sink Connector for `iot-aggregate-metrics-sensor`
ตัวเชื่อมต่อนี้จะย้ายข้อมูลเมตริกที่รวมตามเซ็นเซอร์จากหัวข้อ `iot-aggregate-metrics-sensor` ไปยังคอลเล็กชัน `iot_aggregate_metrics_sensor` ใน MongoDB
```json
{
   "name":"iot-aggregate-metrics-sensor-mongodb-sink",
   "config":{
      "connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
      "tasks.max":1,
      "topics":"iot-aggregate-metrics-sensor",
      "connection.uri":"mongodb://devroot:devroot@mongo:27017",
      "database":"iot",
      "collection":"iot_aggregate_metrics_sensor",
      "value.converter": "org.apache.kafka.connect.json.JsonConverter",
      "key.converter": "org.apache.kafka.connect.storage.StringConverter",
      "value.converter.schemas.enable": false,
      "key.converter.schemas.enable": false
   }
}
```
#### 3. MongoDB Sink Connector for `iot-aggregate-metrics-place`
ตัวเชื่อมต่อนี้จะย้ายข้อมูลเมตริกที่รวมตามสถานที่จากหัวข้อ `iot-aggregate-metrics-place` ไปยังคอลเล็กชัน `iot_aggregate_metrics_place`
```json
{
   "name":"iot-aggregate-metrics-place-mongodb-sink",
   "config":{
      "connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
      "tasks.max":1,
      "topics":"iot-aggregate-metrics-place",
      "connection.uri":"mongodb://devroot:devroot@mongo:27017",
      "database":"iot",
      "collection":"iot_aggregate_metrics_place",
      "value.converter": "org.apache.kafka.connect.json.JsonConverter",
      "key.converter": "org.apache.kafka.connect.storage.StringConverter",
      "value.converter.schemas.enable": false,
      "key.converter.schemas.enable": false
   }
}
```

## Kafka to Prometheus 
Prometheus ซึ่งจำเป็นต้องกำหนดค่าตัวเชื่อมต่อที่สามารถย้ายข้อมูลจาก หัวข้อ `iot-metric-time-series` ไปยังฐานข้อมูลได้ Prometheus จะได้รับข้อมูลผ่านระบบการสำรวจข้อมูล ดังนั้นตัวเชื่อมต่อนี้จึงเปิดใช้งานเซิร์ฟเวอร์ HTTP ที่ Prometheus สามารถค้นหาข้อมูลได้

ตัวเชื่อมต่อ `Kafka Connect Prometheus Metrics Sink` ช่วยให้ข้อมูลนี้พร้อมใช้งานสำหรับจุดสิ้นสุดที่ถูกขูดโดยเซิร์ฟเวอร์ Prometheus ตัวเชื่อมต่อยอมรับโครงสร้างและ JSON แบบไม่มีโครงร่างเป็นค่าของระเบียน Kafka

```json
{
  "name" : "prometheus-connector-sink",
  "config" : {
   "topics":"iot-metrics-time-series",
   "connector.class" : "io.confluent.connect.prometheus.PrometheusMetricsSinkConnector",
   "tasks.max" : "1",
   "confluent.topic.bootstrap.servers":"kafka:9092",
   "prometheus.scrape.url": "http://0.0.0.0:8084/iot-metrics-time-series",
   "prometheus.listener.url": "http://0.0.0.0:8084/iot-metrics-time-series",
   "value.converter": "org.apache.kafka.connect.json.JsonConverter",
   "key.converter": "org.apache.kafka.connect.json.JsonConverter",
   "value.converter.schemas.enable": false,
   "key.converter.schemas.enable":false,
   "reporter.bootstrap.servers": "kafka:9092",
   "reporter.result.topic.replication.factor": "1",
   "reporter.error.topic.replication.factor": "1",
   "behavior.on.error": "log"
  }
}
```
