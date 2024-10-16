# Ingest and store real-time data from IoT sensors 
<!-- >> อธิบาย 3 ส่วนนี้ สร้างมาได้อย่างไร -->

## iot-sensor-1
<!-- >> คืออะไร  -->
IoT sensor 1 เป็น sensor ที่ถูกจําลองด้วยไมโครเซอร์วิสที่ใช้ใน Spring Boot (ผ่านไลบรารี Eclipse Paho MQTT) ที่ถูกติดตั้งอยู่บนเซิฟเวอร์ โดยจะส่งข้อมูล telemetry ไปยังโบรกเกอร์ Eclipse Mosquitto ข้อมูลที่ถูกจำลองนี้ generate ค่า ทุกอย่างภายใน payload มาจาก Callable โดยจะถูกสร้างขึ้นทุกวินาทีและ มี payload ในรูปแบบที่สร้างขึ้นให้ตรงกัน 

## iot-sensor-2
<!-- >> คืออะไร -->
เป็น sensor ที่ถูกจําลองด้วยไมโครเซอร์วิสที่ใช้ใน Spring Boot (ผ่านไลบรารี Eclipse Paho MQTT)เช่นเดียวกันกับ sensor 1  เพียงแต่ติดตั้งอยู่ในเครื่องของคนในทีม

### Edit file pom.xml 
#### iot-sensor-1 และ iot-sensor-2 ต้องทำตรงนี้
เข้าไปแก้ไข้ที่ `iot_event_streaming_architecture/microservices/iot_sensor/pom.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.dreamsoftware</groupId>
    <artifactId>iotframesingest</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>iot_sensor</name>
    <description>Simulate IoT Sensor</description>

    <properties>
        <java.version>1.8</java.version>
        <eclipse-paho-mqtt.version>1.2.0</eclipse-paho-mqtt.version>
        <jackson-bind.version>2.9.8</jackson-bind.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>org.eclipse.paho</groupId>
            <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
            <version>${eclipse-paho-mqtt.version}</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson-bind.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <finalName>iot_sensor</finalName>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>1.18.12</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>

            <plugin>
                    <groupId>com.spotify</groupId>
                    <artifactId>dockerfile-maven-plugin</artifactId>
                    <version>1.4.6</version>
                    <dependencies>
                        <dependency>
                            <groupId>com.github.jnr</groupId>
                            <artifactId>jnr-unixsocket</artifactId>
                            <version>0.38.14</version>
                        </dependency>
                    </dependencies>
            </plugin>
        </plugins>
    </build>

</project>

```
หลังจากนั้นเข้าไปยังโฟลเดอร์ และ `complie` 
```bash
cd iot_event_streaming_architecture/microservices/iot_sensor

# Use maven complie iot sensor for arm
docker run -it --rm --name my-maven-project -v "$(pwd)":/usr/src/mymaven -w /usr/src/mymaven arm64v8/maven:3.8-jdk-8 mvn clean install

# Use maven complie iot sensor for x86
docker run -it --rm --name my-maven-project -v "$(pwd)":/usr/src/mymaven -w /usr/src/mymaven maven:3.8-openjdk-8 mvn clean install
```

## iot-sensor-3-10
<!-- >> คืออะไร  -->
เซ็นเซอร์ 3 จะเป็นค่าจริงจาก sensor ที่เป็นการนำค่าที่อ่านได้จาก Cucumber RS ของกลุ่มตนเอง แตกต่างจาก iot-sensor-1, iot-sensor-2 ที่เป็นการ Mock-up แล้วส่งผ่าน MQTT เพื่อนำมาแสดง และ Payload ต่างๆก็ถูกตั้งให้อยู่ใน format เดียวกัน แล้ว เพื่อการรำไปใช้ต่อได้ในทุกๆ sensor 
Sensor 4 - 10 นั้น จะเป็นค่าจริงเช่นเดียวกัน แต่จะเป็นค่าจาก จาก sensor ของกลุ่มอื่น ที่เป็นการนำค่าที่อ่านได้จาก Cucumber RS ของแต่ละกลุ่ม ที่ส่งผ่าน MQTT มาให้เช่นเดียวกัน เพื่อนำมาแสดง และ Payload ต่างๆก็ถูกตั้งให้อยู่ใน format เดียวกัน แล้ว เพื่อการแสดงผ่าน Cucumber แบบแสดงผลพร้อมกัน 10 sensor
