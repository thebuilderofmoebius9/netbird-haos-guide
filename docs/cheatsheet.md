# HAOS + NetBird สูตรโกง

> คำสั่งที่ใช้จริงวันที่ 2026-09-10 — คุม Home Assistant OS จากระยะไกล, ลง NetBird client, ตรวจ DNS, แก้ LocalSend
> ค่าลับทั้งหมด (token / รหัสผ่าน / setup key) ถูกตัดออก อ่านผ่าน `pass` เท่านั้น

---

## 🔑 ดึง token โดยไม่ให้ค่าหลุดจอ

```bash
T=$(pass show <ha-token-path> | head -1)          # ค่าอยู่ในตัวแปร ไม่ echo ออกมา
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $T" https://<ha-host>/api/
```

`200` = token ใช้ได้ · `401` = ตาย/ผิดสิทธิ์

## 🧩 คุม Supervisor ผ่าน WebSocket (ทางเดียวที่เวิร์ก)

REST `/api/hassio/...` ตอบ `401` เกือบทุก path แม้เป็น admin — ต้องยิงผ่าน WebSocket

```python
# pip install websockets
import asyncio, json, os, websockets
async def main():
    async with websockets.connect("wss://<ha-host>/api/websocket", max_size=None) as ws:
        await ws.recv()
        await ws.send(json.dumps({"type": "auth", "access_token": os.environ["T"]}))
        await ws.recv()
        await ws.send(json.dumps({"id": 1, "type": "supervisor/api",
                                  "endpoint": "/addons", "method": "get"}))
        print(json.loads(await ws.recv()))
asyncio.run(main())
```

endpoint ที่ใช้บ่อย: `/addons` · `/store/addons` · `/store/repositories` · `/dns/info` · `/network/info` · `/addons/<slug>/info` · `/addons/<slug>/options` · `/addons/<slug>/restart`

## 📜 อ่าน log ของ add-on (อันนี้ต้อง REST)

```bash
curl -s -H "Authorization: Bearer $T" \
  "https://<ha-host>/api/hassio/addons/<slug>/logs?lines=200"
```

WebSocket อ่าน log ไม่ได้ เพราะ log เป็น text ไม่ใช่ JSON → เจอ `unknown_error`

## 🌐 ลง NetBird client บน HAOS

```text
1) เพิ่ม repo    POST /store/repositories  {"repository": "https://github.com/netbirdio/addon-netbird"}
2) ติดตั้ง       POST /store/addons/<slug>/install
3) ตั้งค่า       POST /addons/<slug>/options   (ต้องส่งครบทุกคีย์ รวม env_vars: [])
4) รีสตาร์ต      POST /addons/<slug>/restart
5) อ่าน log     เอา URL device login มากดอนุมัติ (อายุ 5 นาที)
```

options ที่ใช้จริง (ไม่มี setup key = ใช้ SSO login แทน)

```json
{"admin_url":"https://<netbird-server>",
 "management_url":"https://<netbird-server>",
 "setup_key":"","hostname":"<hostname>",
 "rosenpass":false,"rosenpass_permissive":false,"env_vars":[]}
```

## 🔍 ตรวจว่าเข้าเมชจริงไหม

```bash
netbird status -d | grep -E "netbird|Status:|Peers count"
ping -c 3 <netbird-ip>        # NetBird IP ของ <hostname>
```

## 🧪 ตรวจว่า NetBird ไปแย่ง DNS หรือเปล่า

```bash
# 1. ดูใน log ว่ามันทำอะไรกับ resolv.conf
curl -s -H "Authorization: Bearer $T" \
  "https://<ha-host>/api/hassio/addons/<slug>/logs?lines=200" | grep -iE "dns|resolv"

# 2. ดู DNS ระดับ Supervisor / การ์ดเน็ต   → endpoint /dns/info และ /network/info

# 3. ทดสอบของจริง: บังคับ HA ดึงข้อมูลออกเน็ตแล้วดูว่าเวลาขยับไหม
curl -s -X POST -H "Authorization: Bearer $T" -H "Content-Type: application/json" \
  -d '{"entity_id":"weather.forecast_home"}' \
  https://<ha-host>/api/services/homeassistant/update_entity
curl -s -H "Authorization: Bearer $T" \
  https://<ha-host>/api/states/weather.forecast_home
```

## 📡 LocalSend / Tailscale

```bash
tailscale status | grep -i main            # เอา IP ปัจจุบัน อย่าจำ IP เก่า
nc -z -w3 <tailscale-ip> 53317 && echo "พอร์ตเปิด"
```

## ⚡ ลัด

```text
ทำอะไร                          คำสั่ง
─────────────────────────────────────────────────────────────────
เช็ค token HA                    curl -H "Authorization: Bearer $T" .../api/
ดูรายชื่อ add-on                 ws supervisor/api → GET /addons
ดู log add-on                    curl .../api/hassio/addons/<slug>/logs?lines=200
ดู DNS ของ HAOS                  ws supervisor/api → GET /dns/info
ดู peer ในเมช                    netbird status -d
ทดสอบ DNS ใช้งานได้จริง          POST /api/services/homeassistant/update_entity
```

## ⚠️ trap ที่เจอจริง

```text
trap                                             วิธีเลี่ยง
──────────────────────────────────────────────────────────────────────────────────
/api/hassio/* ตอบ 401 แม้ token เป็น admin        ใช้ WebSocket supervisor/api แทน
POST options แล้วขึ้น Missing option 'env_vars'   ส่งครบทุกคีย์ในสคีมา รวมค่าว่าง []
ws อ่าน log แล้วได้ unknown_error                  log เป็น text → ใช้ REST อ่านแทน
device code อายุ 5 นาทีแล้วหมด                     รีสตาร์ต add-on แล้วดึงโค้ดใหม่จาก log
ยิง LocalSend ไป IP เดิมแล้วไม่ติด                 เช็ค IP ปัจจุบันก่อนเสมอ อย่าเชื่อค่าที่จำไว้
สรุปว่า "เน็ตขาด" ทั้งที่ยิงผิด IP                  แยกให้ออกว่าปลายทางผิด กับ เครือข่ายพัง
```

---

🤖 เขียนโดย Atom Oracle — Atomic Cosmos (AI) ให้ Axe · 2026-09-10
