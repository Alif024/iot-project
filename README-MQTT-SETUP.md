# คู่มือการแก้ปัญหา HiveMQ และ Node-RED

## ปัญหาที่พบ
HiveMQ กับ Node-RED เชื่อมต่อกันไม่ได้

## สาเหตุที่เป็นไปได้
1. HiveMQ ยังไม่พร้อมรับการเชื่อมต่อเมื่อ Node-RED เริ่มทำงาน
2. การตั้งค่า MQTT listener ไม่ถูกต้อง
3. Network configuration ระหว่าง containers ไม่ถูกต้อง

## วิธีแก้ไข

### 1. ตรวจสอบและรีสตาร์ท Containers

```bash
# หยุด containers ทั้งหมด
docker-compose down

# ลบ volumes เก่า (ถ้าจำเป็น)
docker volume rm code_project_hivemq-data

# สร้างและเริ่ม containers ใหม่
docker-compose up -d

# ตรวจสอบ logs
docker logs hivemq
docker logs nodered
```

### 2. ตรวจสอบสถานะ HiveMQ

ดู logs ของ HiveMQ ว่า MQTT listener เปิดแล้วหรือยัง:

```bash
docker logs hivemq | findstr "Started TCP Listener"
```

คุณควรเห็นข้อความแบบนี้:
```
INFO  - Started TCP Listener on address 0.0.0.0 and on port 1883
```

### 3. ทดสอบการเชื่อมต่อ MQTT

#### ทดสอบด้วย Web Interface
1. เปิดไฟล์ `test-mqtt.html` ใน browser
2. URL: `http://localhost:8081/test-mqtt.html`
3. กด "เชื่อมต่อ"
4. ดู log ว่าเชื่อมต่อสำเร็จหรือไม่

#### ทดสอบด้วย MQTT Explorer (แนะนำ)
1. ดาวน์โหลด [MQTT Explorer](http://mqtt-explorer.com/)
2. เชื่อมต่อไปที่:
   - Host: `localhost`
   - Port: `1883`
   - Protocol: `mqtt://`

#### ทดสอบด้วย Command Line (mosquitto-clients)
```bash
# Subscribe
docker exec -it hivemq mosquitto_sub -h localhost -p 1883 -t "test/topic"

# Publish
docker exec -it hivemq mosquitto_pub -h localhost -p 1883 -t "test/topic" -m "Hello MQTT"
```

### 4. ตรวจสอบการตั้งค่า Node-RED

1. เปิด Node-RED: `http://localhost:1880`
2. ตรวจสอบ MQTT node configuration:
   - Server: `hivemq` (ชื่อ container, ไม่ใช่ localhost)
   - Port: `1883`
   - Client ID: กำหนดเป็นชื่อที่ไม่ซ้ำ

### 5. การแก้ไขปัญหาเพิ่มเติม

#### ถ้า Node-RED ยังเชื่อมต่อไม่ได้
1. ลบและสร้าง MQTT broker node ใหม่ใน Node-RED
2. ตรวจสอบว่าใช้ hostname `hivemq` ไม่ใช่ `localhost`
3. Deploy ใหม่

#### ถ้า HiveMQ ไม่เริ่มทำงาน
1. ตรวจสอบ config.xml มีข้อผิดพลาดหรือไม่:
```bash
docker exec -it hivemq cat /opt/hivemq/conf/config.xml
```

2. ตรวจสอบ permissions ของ volume:
```bash
docker exec -it hivemq ls -la /opt/hivemq/conf/
```

### 6. การเพิ่ม WebSocket Support (สำหรับ Web Apps)

HiveMQ พร้อมใช้งาน WebSocket บน port 8080 แล้ว:
- WebSocket URL: `ws://localhost:8080/mqtt`
- ใช้สำหรับ web applications ที่ต้องการเชื่อมต่อ MQTT

### 7. Port ที่ใช้งาน

| Service | Internal Port | External Port | Description |
|---------|---------------|---------------|-------------|
| HiveMQ MQTT | 1883 | 1883 | MQTT Protocol |
| HiveMQ WebSocket | 8080 | 8080 | MQTT over WebSocket |
| Node-RED | 1880 | 1880 | Node-RED UI |
| Nginx | 80 | 8081 | Static Files |

### 8. ทดสอบด้วย curl (API)

ทดสอบ Node-RED API:
```bash
curl -X POST http://localhost:1880/api/publish ^
  -H "Content-Type: application/json" ^
  -d "{\"message\": \"Hello from curl\", \"value\": 123}"
```

### 9. Common Issues และวิธีแก้

#### Issue: "Connection refused"
```bash
# แก้ไข: ตรวจสอบว่า HiveMQ พร้อมแล้ว
docker exec -it hivemq netstat -tulpn | findstr 1883
```

#### Issue: "Client identifier rejected"
- เปลี่ยน Client ID ใน Node-RED เป็นชื่อที่ไม่ซ้ำกัน

#### Issue: "Too many connections" (Trial license = 25 max)
```bash
# ดูจำนวน connections ปัจจุบัน
docker exec -it hivemq hivemq-cli shell
```

### 10. Health Check

Docker Compose มี health check สำหรับ HiveMQ แล้ว:
```bash
# ตรวจสอบสถานะ health
docker inspect hivemq | findstr Health
```

### 11. การ Debug

เปิด debug logs ใน Node-RED:
1. ไปที่ Settings (มุมบนขวา)
2. เลือก "User Settings"
3. เพิ่ม:
```json
{
    "logging": {
        "console": {
            "level": "debug"
        }
    }
}
```

## สรุป

หลังจากทำตามขั้นตอนนี้แล้ว:
1. HiveMQ จะมี health check เพื่อให้ Node-RED รอจนกว่าจะพร้อม
2. MQTT listener จะตั้งค่าให้รับ connections จากทุก interface (0.0.0.0)
3. มีไฟล์ทดสอบ (test-mqtt.html) สำหรับ verify การทำงาน

หากยังมีปัญหา ให้ตรวจสอบ logs อย่างละเอียด:
```bash
docker logs hivemq -f
docker logs nodered -f
```
