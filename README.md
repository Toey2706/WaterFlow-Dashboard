# WaterFlow Dashboard

ส่วน dashboard ของอุปกรณ์วัดการใช้น้ำ ESP32-S3 โดยทีม **ครูบาช่วยด้วย** ประกอบด้วย dashboard บน ThingsBoard Cloud และสำเนาฟังก์ชันแจ้งเตือนรุ่นแรก

โค้ดฉบับล่าสุดของ water-alert, water-status และแท็บน้ำในแอปอยู่ที่ https://github.com/Toey2706/Dorm-care

- dashboard แบบสาธารณะ (ดูได้อย่างเดียว): https://thingsboard.cloud/dashboard/d93b85d0-e41c-11f1-b86a-c14319f12de9?publicId=495be850-e42c-11f1-b8a1-9db681a717b8
- เฟิร์มแวร์: https://github.com/Toey2706/WaterFlow-Firmware

## การทำงานร่วมกัน

```
YF-S201 ──> ESP32-S3 ──MQTT──> ThingsBoard Cloud ──> dashboard (เกจ ปริมาณรวม กราฟ ปุ่มล้างค่า)
                │                      └──> แท็บน้ำในแอปผู้ดูแล (ผ่านฟังก์ชัน water-status)
                │
                └──HTTPS (น้ำไหลต่อเนื่องครบ LEAK_SECONDS)──> ฟังก์ชัน water-alert บน Supabase
                                                                ├─> การ์ดเตือนทาง LINE ถึงผู้พักที่ผูกห้อง
                                                                ├─> แจ้งเตือนเข้าแอป Android ของผู้ดูแล
                                                                └─> บันทึกลงรายการแจ้งเตือนในแท็บน้ำ
```

การแจ้งเตือนมีไว้บอกเท่านั้น ระบบไม่เปิดใบแจ้งซ่อมเอง ผู้พักเป็นคนตัดสินใจกด "แจ้งซ่อม" จากการ์ดใน LINE แล้วเข้าสู่ขั้นตอนแจ้งซ่อมตามปกติ

## Dashboard

`thingsboard/water_flow_monitor.json` คือ dashboard ที่ export จาก ThingsBoard Cloud ถ้าจะใช้กับบัญชีอื่น

1. สร้างอุปกรณ์ แล้วคัดลอก access token ไปใส่ใน `secrets.h` ของเฟิร์มแวร์
2. ไปที่ Dashboards > Add dashboard > Import dashboard แล้วเลือกไฟล์ JSON
3. เปิดแต่ละ widget แล้วเลือกอุปกรณ์ของคุณเป็นแหล่งข้อมูล

ค่าที่เฟิร์มแวร์ส่งมา: `flow_lpm`, `total_l`, `flowing`, `rssi`, `flow_seconds`, `leak`
ปุ่ม Reset total เรียก RPC `resetTotal` เฟิร์มแวร์รับคำสั่ง `testAlert` ด้วย ใช้ส่งแจ้งเตือนทันทีหนึ่งครั้งตอนสาธิต

ถ้าจะให้ลิงก์สาธารณะแสดงข้อมูล ต้องเอาทั้ง dashboard และอุปกรณ์ไปไว้ใน entity group ที่ตั้งเป็นสาธารณะ

## water-alert

โฟลเดอร์ `water-alert/` เป็นสำเนารุ่นแรกของ Supabase Edge Function ตัวจริงอยู่ที่ `supabase/functions/water-alert` ใน repo Dorm-care ซึ่งใช้โค้ด LINE และ Firebase ร่วมกับส่วนอื่น

ตัวอย่างคำขอ

```
POST https://<project>.supabase.co/functions/v1/water-alert
x-device-key: <WATER_DEVICE_KEY>
Content-Type: application/json

{"room":"A1-01","minutes":10,"flow_lpm":3.2,"total_l":41.5}
```

ผลตอบกลับ: `200` พร้อมจำนวนข้อความ LINE และแจ้งเตือน Android ที่ส่งได้, `401` คีย์อุปกรณ์ผิด, `404` ห้องนี้ไม่ได้ติดอุปกรณ์, `400` ตัวเลขไม่ถูกต้อง

การตั้งค่าใน Supabase

1. เพิ่ม secret `WATER_DEVICE_KEY` (Edge Functions > Secrets) ใช้ค่าเดียวกับใน `secrets.h` ของเฟิร์มแวร์ ใส่เฉพาะตัวคีย์ ไม่ต้องมีเครื่องหมายคำพูด
2. deploy ฟังก์ชัน `water-alert` โดยปิด Verify JWT เพราะฟังก์ชันตรวจคีย์อุปกรณ์เอง

เทสต์รันด้วย Node จาก `supabase/functions` ใน Dorm-care: `node --test`
