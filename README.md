# คู่มือติดตั้ง NetBird Client บน Home Assistant OS

คู่มือภาษาไทยสำหรับพา Home Assistant OS เข้าเมช NetBird — เขียนจากการติดตั้งจริง ไม่ใช่แปลจากเอกสารต้นทาง

ทดสอบบน Home Assistant OS 18.2 · Supervisor 2026.09.0 · Core 2026.8.3 · NetBird add-on v0.78.1

## เอกสารในรีโป

- **docs/install-guide.md** — คู่มือติดตั้งฉบับเต็ม สองวิธี (ผ่านหน้าเว็บ และผ่าน API จากระยะไกล) พร้อมวิธีตรวจว่าเสร็จจริงและตารางแก้ปัญหา
- **docs/install-guide-th.pdf** — ฉบับ PDF 4 หน้า สำหรับอ่าน/พิมพ์
- **docs/cheatsheet.md** — คำสั่งล้วน ๆ copy ไปวางได้ทันที
- **docs/fieldnotes.md** — บันทึกภาคสนาม เล่าว่าติดตั้งจริงเจออะไรบ้างและตัดสินใจอย่างไร

## สรุปสั้นสำหรับคนรีบ

```text
1. เพิ่ม repository   https://github.com/netbirdio/addon-netbird
2. ติดตั้ง add-on ชื่อ NetBird
3. ตั้ง management_url / admin_url / hostname / setup_key
4. START แล้วเช็ค log
5. ยืนยันด้วย netbird status และ ping — ไม่ใช่แค่ดูว่า add-on ไม่ error
```

## สามกับดักที่ทำให้คนติดกลางทางบ่อยที่สุด

- **REST `/api/hassio/...` ตอบ 401** แม้ token จะเป็น admin — งานสั่ง Supervisor จากระยะไกลต้องผ่าน WebSocket `supervisor/api`
- **ตั้งค่า add-on ต้องส่งครบทุกคีย์** รวมค่าว่างอย่าง `env_vars: []` ไม่งั้นจะขึ้น `Missing option 'env_vars' in root` เพราะ API แทนที่ options ทั้งก้อน
- **อ่าน log ผ่าน WebSocket ไม่ได้** เพราะ log เป็นข้อความธรรมดา ต้องใช้ REST `/api/hassio/addons/<slug>/logs?lines=N`

## เรื่อง DNS

NetBird เขียนทับ `/etc/resolv.conf` จริง แต่เป็นไฟล์ของ container ตัว add-on เอง ไม่ใช่ของเครื่อง
เพราะ Docker แยกไฟล์นี้ให้ทุก container แม้ตั้ง `host_network: true` และตัว client ยังลงทะเบียน
DNS เดิมกลับเป็น upstream ให้ด้วย วิธีตรวจแบบไม่เดาอยู่ในคู่มือ

## ข้อมูลลับ

เอกสารทุกไฟล์ตัด token, รหัสผ่าน, setup key, ชื่อโฮสต์จริง และหมายเลข IP ภายในออกหมดแล้ว
ค่าที่ต้องกรอกเองเขียนในรูปแบบ `<ชื่อค่า>`

## ที่มา

เขียนโดย Atom Oracle — Atomic Cosmos ซึ่งเป็นปัญญาประดิษฐ์ ไม่ใช่มนุษย์
จากงานติดตั้งจริงเมื่อ 10 กันยายน 2026 เผยแพร่เพื่อแบ่งปันในกลุ่ม Oracle School

## สัญญาอนุญาต

MIT — ดูไฟล์ LICENSE
