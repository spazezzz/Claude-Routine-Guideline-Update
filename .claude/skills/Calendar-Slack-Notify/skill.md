# Calendar-Slack-Notify

> **Runtime target:** Claude Remote Routine (web sandbox)
> **Constraint:** No Bash / shell / git CLI — all I/O via Google Calendar connector and Slack connector only.

## Invocation

```
/Calendar-Slack-Notify
```

---

## วัตถุประสงค์

ตรวจสอบ Google Calendar ว่ามี Event ในวันถัดไปหรือไม่ ถ้ามี ให้ส่งการแจ้งเตือนสรุป Event ผ่าน Slack

---

## Execution Flow

### Step 1 — คำนวณช่วงเวลาวันถัดไป

1. อ่านวันที่และเวลาปัจจุบัน (timezone: **Asia/Bangkok, UTC+7**)
2. คำนวณวันถัดไป (tomorrow):
   - `timeMin` = เที่ยงคืน 00:00:00 ของวันถัดไป (ISO 8601, UTC)
   - `timeMax` = 23:59:59 ของวันถัดไป (ISO 8601, UTC)
3. บันทึก `DATE_TOMORROW` ในรูปแบบ `DD/MM/YYYY` (ภาษาไทย: วัน/เดือน/ปี พ.ศ. ก็ได้)

### Step 2 — ดึง Event จาก Google Calendar

ใช้ **Google Calendar connector** (MCP) เพื่อดึง Event:

- เรียก `list_events` พร้อม parameter:
  - `timeMin`: เที่ยงคืน 00:00:00 ของพรุ่งนี้ (UTC)
  - `timeMax`: 23:59:59 ของพรุ่งนี้ (UTC)
  - `singleEvents`: true
  - `orderBy`: startTime

**ถ้า connector ไม่พร้อมใช้งาน:**
- หยุดการทำงานทันที
- บันทึก log: `[ERROR] Google Calendar connector unavailable at <ISO timestamp>. Routine aborted.`
- ไม่ต้องดำเนินการต่อ

### Step 3 — ตรวจสอบผลลัพธ์

- **ถ้าไม่มี Event (รายการว่าง):**
  - หยุดการทำงาน — ไม่ต้องทำอะไรต่อ
  - บันทึก log: `[INFO] No events found for <DATE_TOMORROW>. Routine completed with no action.`

- **ถ้ามี Event อย่างน้อย 1 รายการ:**
  - ดำเนินการต่อไปยัง Step 4

### Step 4 — สร้างข้อความแจ้งเตือน

จัดรูปแบบข้อความสรุป Event สำหรับส่งผ่าน Slack:

```
📅 กำหนดการพรุ่งนี้ — <DATE_TOMORROW>

<วนซ้ำสำหรับแต่ละ Event>
🕐 <เวลาเริ่ม>–<เวลาสิ้นสุด> | <ชื่อ Event>
   📍 <สถานที่ หรือ ลิงก์ประชุม (ถ้ามี)>
   👥 <จำนวนผู้เข้าร่วม (ถ้ามี)>
</วนซ้ำ>

รวม <N> กำหนดการ
```

กฎการจัดรูปแบบ:
- แสดงเวลาในรูปแบบ `HH:MM` timezone Asia/Bangkok
- ถ้า Event เป็น All-day ให้แสดง `ทั้งวัน` แทนช่วงเวลา
- ถ้าไม่มีสถานที่หรือลิงก์ประชุม ให้ข้ามบรรทัด 📍
- ถ้าไม่มีผู้เข้าร่วม ให้ข้ามบรรทัด 👥
- จำกัดสูงสุด 10 Event แรก ถ้ามีมากกว่านั้น ให้เพิ่มข้อความ `... และอีก <M> รายการ`

### Step 5 — ส่งแจ้งเตือนผ่าน Slack

ใช้ **Slack connector** (MCP) เพื่อส่งข้อความ:

- เรียก `slack_send_message` พร้อมข้อความจาก Step 4
- Channel / recipient: กำหนดจาก environment variable `SLACK_CHANNEL` (เช่น `#general`, `@username`)

**การจัดการข้อผิดพลาด:**
- สำเร็จ → บันทึก log: `[INFO] Slack notification sent successfully for <DATE_TOMORROW>.`
- ล้มเหลว → บันทึก log: `[ERROR] Slack send failed: <error message>. Do not retry silently.`
- ไม่ retry อัตโนมัติ

---

## Environment Variables

| Variable | Required | คำอธิบาย |
|----------|----------|-----------|
| `SLACK_CHANNEL` | Yes | Channel หรือ User ที่ต้องการส่งแจ้งเตือน เช่น `#daily-reminders` หรือ `@username` |
| `GOOGLE_CALENDAR_ID` | No | Calendar ID ที่ต้องการตรวจสอบ ถ้าไม่ระบุ ใช้ primary calendar |

---

## Timezone

ทุก date/time string ใช้ **Asia/Bangkok (UTC+7)**

เมื่อคำนวณ "วันถัดไป":
1. อ่านเวลา UTC ปัจจุบันจาก runtime context
2. บวก 7 ชั่วโมง → ได้เวลา Bangkok
3. เพิ่ม 1 วัน → ได้วันถัดไปใน timezone Bangkok
4. แปลงกลับเป็น UTC สำหรับ timeMin / timeMax ที่ส่งให้ Calendar API

---

## Guardrails Summary

| กฎ | พฤติกรรม |
|----|----------|
| Google Calendar connector ไม่พร้อม | หยุดทันที, log error, ไม่ส่ง Slack |
| ไม่มี Event ในวันถัดไป | หยุด, log info, ไม่ส่ง Slack |
| `SLACK_CHANNEL` ไม่ได้ตั้งค่า | log error, หยุด — ไม่ส่งไปยัง channel ที่ไม่ได้กำหนด |
| Slack ส่งไม่สำเร็จ | log error, ไม่ retry |
| มี Event มากกว่า 10 รายการ | แสดงแค่ 10 รายการแรก พร้อมแจ้งจำนวนที่เหลือ |

---

## ตัวอย่าง Slack Message

```
📅 กำหนดการพรุ่งนี้ — 04/05/2026

🕐 09:00–10:00 | Weekly Team Standup
   📍 https://meet.google.com/abc-defg-hij
   👥 8 คน

🕐 13:00–14:00 | 1-on-1 กับ Manager
   👥 2 คน

🕐 ทั้งวัน | วันหยุดธนาคาร

รวม 3 กำหนดการ
```

---

## วิธีตั้ง Routine บน Claude Web

1. เปิด **Routines → New Routine**
2. ตั้ง **Skill**: `Calendar-Slack-Notify`
3. ตั้ง **Schedule**: `0 20 * * *` (ทุกคืน 20:00 Bangkok time — แจ้งเตือนล่วงหน้า 1 วัน)
4. ตั้ง **Timezone**: `Asia/Bangkok`
5. ตั้ง **Environment Variables**: `SLACK_CHANNEL`, `GOOGLE_CALENDAR_ID` (optional)
6. ตรวจสอบว่า Google Calendar connector และ Slack connector เชื่อมต่อแล้วใน **Settings → Connectors**
7. บันทึกและเปิดใช้งาน Routine

### ทดสอบ Manual Run

1. กด **Run now** จาก Routine dashboard
2. ตรวจสอบ log ว่า Step ใดทำงานถึง
3. ถ้ามี Event พรุ่งนี้ ตรวจสอบว่าได้รับข้อความใน Slack channel ที่กำหนด
