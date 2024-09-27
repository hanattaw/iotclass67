# Analyze and make aggregations.
>> อธิบาย ส่วนวิเคราห์ข้อมูล สามส่วนนี้ ใช้หลักการอย่างไร ทำแล้วได้อะไร

## Aggregate Metrics By Sensor Processor
โปรเซสเซอร์ Aggregate Metrics By Sensor Processor ทำหน้าที่รวมข้อมูลจากเซ็นเซอร์โดยอิงตาม Sensor ID โดยใช้หน้าต่างเวลาแบบหมุน (sliding window) ขนาด 5 นาที เพื่อคำนวณค่าเฉลี่ยของพารามิเตอร์ต่างๆ เช่น อุณหภูมิ ความชื้น ความดันอากาศ และความส่องสว่าง
ในกระบวนการนี้ โปรเซสเซอร์จะสร้าง schema model ที่เหมาะสม และจำเป็นต้องกำหนดค่า SerDe (Serializer/Deserializer) ที่ถูกต้องเพื่อแปลงข้อมูลในรูปแบบที่สามารถจัดเก็บและดึงข้อมูลจาก Kafka ได้อย่างมีประสิทธิภาพ
```java
@Component
public class AggregateMetricsBySensorProcessor {

    private static final Logger logger = LoggerFactory.getLogger(AggregateMetricsBySensorProcessor.class);

    private final static int WINDOW_SIZE_IN_MINUTES = 5;
    private final static String WINDOW_STORE_NAME = "aggregate-metrics-by-sensor-tmp";

    /**
     * Agg Metrics Sensor Topic Output
     */
    @Value("${kafka.topic.aggregate-metrics-sensor}")
    private String aggMetricsSensorOutput;

    /**
     *
     * @param stream
     */
    public void process(KStream<SensorKeyDTO, SensorDataDTO> stream) {
        buildAggregateMetricsBySensor(stream)
                .to(aggMetricsSensorOutput, Produced.with(String(), new SensorAggregateMetricsSensorSerde()));
    }

    /**
     * Build Aggregate Metrics By Sensor Stream
     *
     * @param stream
     * @return
     */
    private KStream<String, SensorAggregateSensorMetricsDTO> buildAggregateMetricsBySensor(KStream<SensorKeyDTO, SensorDataDTO> stream) {
        return stream
                .map((key, val) -> new KeyValue<>(val.getId(), val))
                .groupByKey(Grouped.with(String(), new SensorDataSerde()))
                .windowedBy(TimeWindows.of(Duration.ofMinutes(WINDOW_SIZE_IN_MINUTES)).grace(Duration.ofMillis(0)))
                .aggregate(SensorAggregateSensorMetricsDTO::new,
                        (String k, SensorDataDTO v, SensorAggregateSensorMetricsDTO va) -> aggregateData(v, va),
                         buildWindowPersistentStore()
                )
                .suppress(Suppressed.untilWindowCloses(unbounded()))
                .toStream()
                .map((key, value) -> KeyValue.pair(key.key(), value));
    }

    /**
     * Build Window Persistent Store
     *
     * @return
     */
    private Materialized<String, SensorAggregateSensorMetricsDTO, WindowStore<Bytes, byte[]>> buildWindowPersistentStore() {
        return Materialized
                .<String, SensorAggregateSensorMetricsDTO, WindowStore<Bytes, byte[]>>as(WINDOW_STORE_NAME)
                .withKeySerde(String())
                .withValueSerde(new SensorAggregateMetricsSensorSerde());
    }

    /**
     * Aggregate Data
     *
     * @param v
     * @param va
     * @return
     */
    private SensorAggregateSensorMetricsDTO aggregateData(final SensorDataDTO v, final SensorAggregateSensorMetricsDTO va) {
        // Sensor Data
        va.setId(v.getId());
        // Sensor Data
        va.setId(v.getId());
        va.setName(v.getName());
        // Start Agg
        if (va.getStartAgg() == null) {
            final Date startAggAt = new Date();
            va.setStartAgg(startAggAt);
            va.setStartAggTm(startAggAt.getTime());
        }
        va.setCountMeasures(va.getCountMeasures() + 1);
        // Temperature
        va.setSumTemperature(va.getSumTemperature() + v.getPayload().getTemperature());
        va.setAvgTemperature(va.getSumTemperature() / va.getCountMeasures()); // Humidity
        // Humidity
        va.setSumHumidity(va.getSumHumidity() + v.getPayload().getHumidity());
        va.setAvgHumidity(va.getSumHumidity() / va.getCountMeasures()); // Luminosity
        // Luminosity
        va.setSumLuminosity(va.getSumLuminosity() + v.getPayload().getLuminosity());
        va.setAvgLuminosity(va.getSumLuminosity() / va.getCountMeasures()); // Pressure
        // Pressure
        va.setSumPressure(va.getSumPressure() + v.getPayload().getPressure());
        va.setAvgPressure(va.getSumPressure() / va.getCountMeasures());

        // End Agg
        final Date endAggAt = new Date();
        va.setEndAgg(endAggAt);
        va.setEndAggTm(endAggAt.getTime());
        return va;
    }

}
```
เมื่อหน้าต่างเวลาปิดลง (เมื่อครบ 5 นาที) ระบบจะสร้างบันทึกข้อมูลที่ประกอบไปด้วยการวัดผลรวมประมาณ 300 ค่า สำหรับเซ็นเซอร์แต่ละตัว โดยคำนวณค่าเฉลี่ยของแต่ละพารามิเตอร์ จากนั้นข้อมูลเหล่านี้จะถูกบันทึกลงในหัวข้อ iot-aggregate-metrics-by-sensor บน Kafka เพื่อใช้ในกระบวนการต่อไป
## Aggregate Metrics By Place Processor
โปรเซสเซอร์ Aggregate Metrics By Place Processor ทำหน้าที่รวมข้อมูลในลักษณะเดียวกับ Aggregate Metrics By Sensor Processor แต่ในกรณีนี้ การคำนวณค่าเฉลี่ยจะอ้างอิงจาก สถานที่ (Place ID) แทนที่จะเป็นเซ็นเซอร์แต่ละตัว
```java
@Component
public class AggregateMetricsByPlaceProcessor {

    private static final Logger logger = LoggerFactory.getLogger(AggregateMetricsByPlaceProcessor.class);

    private final static int WINDOW_SIZE_IN_MINUTES = 5;
    private final static String WINDOW_STORE_NAME = "aggregate-metrics-by-place-tmp";

    /**
     * Agg Metrics Place Topic Output
     */
    @Value("${kafka.topic.aggregate-metrics-place}")
    private String aggMetricsPlaceOutput;

    /**
     *
     * @param stream
     */
    public void process(KStream<SensorKeyDTO, SensorDataDTO> stream) {
        buildAggregateMetrics(stream)
                .to(aggMetricsPlaceOutput, Produced.with(String(), new SensorAggregateMetricsPlaceSerde()));
    }

    /**
     * Build Aggregate Metrics Stream
     *
     * @param stream
     * @return
     */
    private KStream<String, SensorAggregatePlaceMetricsDTO> buildAggregateMetrics(KStream<SensorKeyDTO, SensorDataDTO> stream) {
        return stream
                .map((key, val) -> new KeyValue<>(val.getPlaceId(), val))
                .groupByKey(Grouped.with(String(), new SensorDataSerde()))
                .windowedBy(TimeWindows.of(Duration.ofMinutes(WINDOW_SIZE_IN_MINUTES)).grace(Duration.ofMillis(0)))
                .aggregate(SensorAggregatePlaceMetricsDTO::new,
                        (String k, SensorDataDTO v, SensorAggregatePlaceMetricsDTO va) -> aggregateData(v, va),
                        buildWindowPersistentStore()
                )
                .suppress(Suppressed.untilWindowCloses(unbounded()))
                .toStream()
                .map((key, value) -> KeyValue.pair(key.key(), value));
    }

    /**
     * Build Window Persistent Store
     *
     * @return
     */
    private Materialized<String, SensorAggregatePlaceMetricsDTO, WindowStore<Bytes, byte[]>> buildWindowPersistentStore() {
        return Materialized
                .<String, SensorAggregatePlaceMetricsDTO, WindowStore<Bytes, byte[]>>as(WINDOW_STORE_NAME)
                .withKeySerde(String())
                .withValueSerde(new SensorAggregateMetricsPlaceSerde());
    }

    /**
     * Aggregate Data
     *
     * @param v
     * @param va
     * @return
     */
    private SensorAggregatePlaceMetricsDTO aggregateData(final SensorDataDTO v, final SensorAggregatePlaceMetricsDTO va) {
        va.setPlaceId(v.getId());
        // Start Agg
        if (va.getStartAgg() == null) {
            final Date startAggAt = new Date();
            va.setStartAgg(startAggAt);
            va.setStartAggTm(startAggAt.getTime());
        }
        va.setCountMeasures(va.getCountMeasures() + 1);
        // Temperature
        va.setSumTemperature(va.getSumTemperature() + v.getPayload().getTemperature());
        va.setAvgTemperature(va.getSumTemperature() / va.getCountMeasures()); // Humidity
        // Humidity
        va.setSumHumidity(va.getSumHumidity() + v.getPayload().getHumidity());
        va.setAvgHumidity(va.getSumHumidity() / va.getCountMeasures()); // Luminosity
        // Luminosity
        va.setSumLuminosity(va.getSumLuminosity() + v.getPayload().getLuminosity());
        va.setAvgLuminosity(va.getSumLuminosity() / va.getCountMeasures()); // Pressure
        // Pressure
        va.setSumPressure(va.getSumPressure() + v.getPayload().getPressure());
        va.setAvgPressure(va.getSumPressure() / va.getCountMeasures());

        // End Agg
        final Date endAggAt = new Date();
        va.setEndAgg(endAggAt);
        va.setEndAggTm(endAggAt.getTime());
        return va;
    }

}
```
โปรเซสเซอร์จะสร้าง schema model ที่แตกต่างกัน ซึ่งมีการกำหนดค่า SerDe (Serializer/Deserializer) ที่เหมาะสม เพื่อให้สามารถบันทึกข้อมูลในรูปแบบ serialized และจัดเก็บลงในหัวข้อ iot-aggregate-metrics-place บน Kafka ได้อย่างถูกต้อง

## Aggregate Metrics time series
โปรเซสเซอร์ Aggregate Metrics Time Series Processor มีจุดประสงค์ในการแปลงข้อมูลให้เหมาะสมกับการใช้งานใน Prometheus โดยใช้ชื่อเซ็นเซอร์, รหัสเซ็นเซอร์ และตัวระบุสถานที่เป็น มิติข้อมูล (dimensions) ในการจัดการข้อมูลเหล่านี้
```java
@Component
public class MetricsTimeSeriesProcessor {

    private static final Logger logger = LoggerFactory.getLogger(MetricsTimeSeriesProcessor.class);

    private final static String SENSOR_TIME_SERIE_NAME = "sample_sensor_metric";
    private final static String SENSOR_TIME_SERIE_TYPE = "sensor";

    /**
     * Metrics Time Series
     */
    @Value("${kafka.topic.metrics-time-series}")
    private String metricTimeSeriesOutput;

    /**
     *
     * @param stream
     */
    public void process(KStream<SensorKeyDTO, SensorDataDTO> stream) {
        stream
                .map((key, val) -> new KeyValue<>(val.getId(), buildSensorTimeSerieMetric(val)))
                .to(metricTimeSeriesOutput, Produced.with(String(), new SensorTimeSerieMetricSerde()));
    }

    /**
     * Build Sensor Time Serie Meter
     *
     * @param sensorData
     * @return
     */
    private SensorTimeSerieMetricDTO buildSensorTimeSerieMetric(final SensorDataDTO sensorData) {
        return SensorTimeSerieMetricDTO.builder()
                .name(SENSOR_TIME_SERIE_NAME)
                .timestamp(new Date().getTime())
                .type(SENSOR_TIME_SERIE_TYPE)
                .dimensions(SensorTimeSerieMetricDimensionsDTO.builder()
                        .placeId(sensorData.getPlaceId())
                        .sensorId(sensorData.getId())
                        .sensorName(sensorData.getName())
                        .build())
                .values(SensorTimeSerieMetricValuesDTO.builder()
                        .humidity((double) sensorData.getPayload().getHumidity())
                        .luminosity((double) sensorData.getPayload().getLuminosity())
                        .pressure((double) sensorData.getPayload().getPressure())
                        .temperature((double) sensorData.getPayload().getTemperature())
                        .build())
                .build();
    }

}
```
ข้อมูลที่ได้จะอยู่ในรูปแบบที่สามารถนำไปแสดงผลเป็น กราฟลำดับเวลา (time-series) บน Grafana ได้ โดยบันทึกเหล่านี้จะถูกจัดเก็บในหัวข้อ iot-metrics-time-series ซึ่ง Prometheus จะเข้ามาดึงข้อมูลจากหัวข้อนี้เพื่อนำไปใช้ในการตรวจสอบและวิเคราะห์เมตริกต่างๆ ในระบบ

