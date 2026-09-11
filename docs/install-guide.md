# คู่มือติดตั้ง NetBird Client บน Home Assistant OS

เอกสารทำซ้ำได้ · เขียนจากการติดตั้งจริงเมื่อ 10 กันยายน 2026
ปลายทางที่ทดสอบแล้ว: Home Assistant OS 18.2 · Supervisor 2026.09.0 · Core 2026.8.3 · NetBird add-on v0.78.1
ข้อมูลลับทุกชนิดถูกตัดออก ค่าที่ต้องกรอกเองเขียนในรูปแบบ `<ชื่อค่า>`

---

## 0. เตรียมก่อนเริ่ม

ต้องมีครบ 3 อย่างนี้ก่อน ไม่งั้นจะไปติดกลางทาง

1. **สิทธิ์เข้า Home Assistant ระดับผู้ดูแลระบบ** — จะเข้าทางหน้าเว็บหรือทาง token ก็ได้
2. **ที่อยู่ NetBird management** — ถ้าใช้ NetBird cloud ข้ามได้ ถ้าเป็น self-host ต้องรู้ URL เช่น `https://<netbird-server>`
3. **วิธีให้เครื่องเข้าเมช** เลือกอย่างใดอย่างหนึ่ง
   - **setup key** จากแดชบอร์ด NetBird (แนะนำ — เข้าเมชเองอัตโนมัติ ไม่ต้องกดซ้ำเวลารีสตาร์ต)
   - **SSO device login** (ไม่ต้องขอคีย์ แต่ต้องมีคนกดอนุมัติ และรหัสมีอายุ 5 นาที)

ถ้าเป็น Home Assistant แบบ Container หรือ Core ที่ไม่มี Supervisor **จะทำตามคู่มือนี้ไม่ได้**
เพราะไม่มีระบบ add-on ให้ลง วิธีตรวจ: ดูว่าหน้า Settings มีเมนู Add-ons หรือไม่

---

## วิธี A — ติดตั้งผ่านหน้าเว็บ (แนะนำสำหรับคนทั่วไป)

### A1. เพิ่มแหล่ง add-on

1. เข้า **Settings → Add-ons → ADD-ON STORE**
2. กดจุดสามจุดมุมขวาบน → **Repositories**
3. วาง URL นี้แล้วกด ADD

```text
https://github.com/netbirdio/addon-netbird
```

### A2. ติดตั้ง

1. ปิดหน้าต่าง Repositories แล้วเลื่อนหาหมวด **NetBird**
2. กด add-on ชื่อ **NetBird** → **INSTALL** (รอสักครู่ ระบบจะดึงอิมเมจมา)

### A3. ตั้งค่า

เปิดแท็บ **Configuration** ของ add-on แล้วกรอก

```yaml
management_url: https://<netbird-server>     # ว่างไว้ถ้าใช้ NetBird cloud
admin_url: https://<netbird-server>          # ว่างไว้ถ้าใช้ NetBird cloud
setup_key: "<setup-key>"                     # ว่างไว้ถ้าจะใช้ SSO login
hostname: <ชื่อเครื่องที่จะให้โชว์ในเมช>
rosenpass: false
rosenpass_permissive: false
env_vars: []
```

> **เลือกทางนี้ให้ถูกตั้งแต่แรก** — ใส่ setup key = เครื่องอยู่ในเมชถาวร ·
> ปล่อยว่างแล้วใช้ SSO login = เครื่องจะหลุดเองใน 24 ชั่วโมง เหตุผลอยู่ที่หัวข้อ A5

กด **SAVE** แล้วไปแท็บ **Info** กด **START**

### A4. ถ้าไม่ได้ใส่ setup key ต้องกดอนุมัติ

1. เปิดแท็บ **Log** ของ add-on
2. จะเห็นบรรทัดประมาณนี้

```text
Please do the SSO login in your browser.
If your browser didn't open automatically, use this URL to log in:
https://<netbird-server>/oauth2/device?user_code=XXXX-XXXX
```

3. เปิดลิงก์นั้นในเบราว์เซอร์ แล้วกดอนุมัติภายใน 5 นาที
4. ถ้าเลยเวลา ให้กด **RESTART** ที่ add-on แล้วอ่าน log เอารหัสใหม่

### A5. ถ้าเลือกทาง SSO เครื่องจะหลุดเองใน 24 ชั่วโมง

peer ที่เข้าเมชด้วย SSO login มีอายุ พอครบกำหนดจะหลุดออกจากเมชเองแล้วรอ login ใหม่
เครื่องที่ไม่มีคนนั่งอยู่หน้าจออย่าง HAOS ในตู้ จะกลับเข้าเมชเองไม่ได้ ต้องมีคนไปกดอนุมัติให้

เงื่อนไขการหมดอายุมีสามชั้น ต้องครบทั้งสามถึงจะเริ่มนับเวลา
(ยืนยันจากซอร์ส NetBird `management/server/peer/peer.go` ฟังก์ชัน `LoginExpired`)

```text
1. peer ถูกเพิ่มด้วย SSO         = ฟิลด์ UserID ไม่ว่าง
                                  peer ที่เข้าด้วย setup key ฟิลด์นี้ว่าง จึงไม่มีวันหมดอายุ
2. peer เปิด login expiration    ฟิลด์ LoginExpirationEnabled ของ peer
3. account เปิด login expiration ฟิลด์ PeerLoginExpirationEnabled ของทั้งบัญชี
```

นาฬิกานับจาก `last_login` ของ peer ไม่ใช่เวลาที่เชื่อมต่อครั้งล่าสุด และค่าเริ่มต้นคือ **24 ชั่วโมง**
(`DefaultPeerLoginExpiration = 24 * time.Hour` ใน `management/server/types/account.go`)

วิธีแก้ เลือกอย่างใดอย่างหนึ่ง

- **ทางที่ควรเลือก** — ขอ setup key จากคนดูแลเซิร์ฟเวอร์ แล้วตั้งค่า add-on ใหม่โดยใส่ setup key
  ต้องลบ peer เดิมออกจากแดชบอร์ดก่อน ไม่งั้นชื่อจะชนแล้วถูกต่อท้ายเป็น `<ชื่อ>-1`
  เพราะ NetBird ใช้ชื่อ peer เป็น DNS label ที่ห้ามซ้ำภายในบัญชีเดียวกัน
- **ถ้าคุมแดชบอร์ดเองได้** — ปิดหมดอายุเฉพาะเครื่องนี้ ที่หน้า peer เอา login expiration ออก
  เท่ากับตั้ง `login_expiration_enabled: false` ผ่าน `PUT /api/peers/<peer-id>`

ตรวจว่าเครื่องไหนเข้าข่ายหมดอายุบ้าง ดูที่ `GET /api/peers` ฟิลด์ `user_id` ถ้าไม่ว่างคือเข้าด้วย SSO

---

## วิธี B — ติดตั้งจากระยะไกลผ่าน API

ใช้เมื่อเข้าหน้าเว็บไม่ได้ หรืออยากทำแบบสั่งจากเครื่องอื่น

### B1. เตรียม token และเครื่องมือ

```bash
T=$(pass show <ที่เก็บ-token-ha> | head -1)   # อย่า echo ค่าออกมา
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $T" https://<ha-host>/api/
# ต้องได้ 200

python3 -m venv /tmp/hav && /tmp/hav/bin/pip -q install websockets
```

**หมายเหตุสำคัญ**: REST path `/api/hassio/...` จะตอบ `401` เกือบทุก endpoint แม้ token เป็น admin
งานสั่งการทั้งหมดจึงต้องผ่าน WebSocket ยกเว้นการอ่าน log

### B2. สคริปต์สั่ง Supervisor

บันทึกเป็น `sv.py`

```python
import asyncio, json, os, sys, websockets

HA = os.environ["HA_HOST"]          # เช่น home.example.com
TOKEN = os.environ["T"]

async def call(endpoint, method="get", data=None):
    async with websockets.connect(f"wss://{HA}/api/websocket", max_size=None) as ws:
        await ws.recv()
        await ws.send(json.dumps({"type": "auth", "access_token": TOKEN}))
        await ws.recv()
        msg = {"id": 1, "type": "supervisor/api", "endpoint": endpoint, "method": method}
        if data is not None:
            msg["data"] = data
        await ws.send(json.dumps(msg))
        while True:
            r = json.loads(await asyncio.wait_for(ws.recv(), 600))
            if r.get("id") == 1:
                return r

if __name__ == "__main__":
    ep, method = sys.argv[1], (sys.argv[2] if len(sys.argv) > 2 else "get")
    body = json.loads(sys.argv[3]) if len(sys.argv) > 3 else None
    print(json.dumps(asyncio.run(call(ep, method, body)), ensure_ascii=False)[:2000])
```

### B3. สำรองสถานะก่อนแตะระบบ

```bash
export HA_HOST=<ha-host>
for ep in /addons /store/repositories /supervisor/info /host/info; do
  /tmp/hav/bin/python sv.py "$ep" >> pre-change.json
done
```

### B4. เพิ่ม repository แล้วติดตั้ง

```bash
/tmp/hav/bin/python sv.py /store/repositories post \
  '{"repository":"https://github.com/netbirdio/addon-netbird"}'

# หา slug ของ add-on (จะเป็นเลขสุ่ม 8 ตัว ตามด้วย _netbird)
/tmp/hav/bin/python sv.py /store/addons | grep -o '"slug": "[a-z0-9]*_netbird"'

SLUG=<slug-ที่ได้>
/tmp/hav/bin/python sv.py "/store/addons/$SLUG/install" post
```

### B5. ตั้งค่า — ต้องส่งครบทุกคีย์

นี่คือจุดที่พลาดง่ายที่สุด ถ้าส่งไม่ครบจะขึ้น `Missing option 'env_vars' in root`
API นี้แทนที่ options ทั้งก้อน ไม่ใช่แก้เฉพาะคีย์ที่ส่งไป

```bash
/tmp/hav/bin/python sv.py "/addons/$SLUG/options" post '{
  "options": {
    "management_url": "https://<netbird-server>",
    "admin_url": "https://<netbird-server>",
    "setup_key": "",
    "hostname": "<ชื่อเครื่อง>",
    "rosenpass": false,
    "rosenpass_permissive": false,
    "env_vars": []
  }
}'

/tmp/hav/bin/python sv.py "/addons/$SLUG/restart" post
```

### B6. อ่าน log (ต้องใช้ REST เท่านั้น)

WebSocket อ่าน log ไม่ได้ เพราะ log เป็นข้อความธรรมดา ไม่ใช่ JSON จะได้ `unknown_error`

```bash
curl -s -H "Authorization: Bearer $T" \
  "https://$HA_HOST/api/hassio/addons/$SLUG/logs?lines=200" | tail -30
```

ถ้าไม่ได้ใส่ setup key ให้หา URL device login ในนี้ แล้วกดอนุมัติเหมือนขั้น A4

---

## ตรวจว่าเสร็จจริง

อย่าถือว่า "start แล้วไม่ error" คือเสร็จ ต้องเห็นสามอย่างนี้

### 1. log ขึ้นว่า sync ผ่าน

```text
sync finished in NNms
peer added to lazy conn manager
```

### 2. เครื่องอื่นในเมชเห็น peer ใหม่

```bash
netbird status -d | grep -A2 "<ชื่อเครื่อง>"
ping -c 3 <NetBird-IP-ของเครื่องใหม่>
```

`Status: Idle` ถือว่าปกติ แปลว่าเป็น lazy connection คือจะเปิดทันเนลเมื่อมีทราฟฟิกจริง
`ping` ผ่านคือหลักฐานว่าต่อได้

### 3. DNS ของเครื่องปลายทางไม่พัง

NetBird เขียนทับ `/etc/resolv.conf` จริง แต่เป็นไฟล์ของ container ตัวเองเท่านั้น
เพราะ Docker แยกไฟล์นี้ให้ทุก container แม้ add-on จะตั้ง `host_network: true`
และมันลงทะเบียน DNS เดิมกลับเป็น upstream ให้อยู่แล้ว

ตรวจแบบไม่เดา

```bash
# ก. ดูใน log ว่ามันทำอะไรกับ DNS
curl -s -H "Authorization: Bearer $T" \
  "https://$HA_HOST/api/hassio/addons/$SLUG/logs?lines=200" | grep -iE "dns|resolv"
# ควรเห็น: registering original nameservers [...] as upstream handlers

# ข. ดูค่า DNS ระดับระบบว่าไม่เปลี่ยน
/tmp/hav/bin/python sv.py /dns/info
/tmp/hav/bin/python sv.py /network/info

# ค. ทดสอบของจริง — บังคับ HA ดึงข้อมูลออกเน็ตแล้วดูว่าเวลาขยับ
curl -s -X POST -H "Authorization: Bearer $T" -H "Content-Type: application/json" \
  -d '{"entity_id":"weather.forecast_home"}' \
  "https://$HA_HOST/api/services/homeassistant/update_entity"
curl -s -H "Authorization: Bearer $T" "https://$HA_HOST/api/states/weather.forecast_home"
```

ถ้าเวลาใน `last_updated` ขยับมาเป็นเวลาปัจจุบัน แปลว่า DNS และเน็ตยังทำงานปกติ

---

## ปัญหาที่เจอจริงและวิธีแก้

```text
อาการ                                          สาเหตุ                          วิธีแก้
──────────────────────────────────────────────────────────────────────────────────────────────
/api/hassio/* ตอบ 401 แม้ token เป็น admin      REST proxy ไม่เปิดให้           สั่งผ่าน WebSocket supervisor/api
Missing option 'env_vars' in root               ส่ง options ไม่ครบทุกคีย์         ส่งครบ รวมค่าว่าง [] และ ""
อ่าน log ผ่าน WebSocket ได้ unknown_error        log เป็น text ไม่ใช่ JSON        อ่านผ่าน REST /logs?lines=N
device code หมดอายุก่อนกด                        อายุแค่ 5 นาที                   restart add-on แล้วอ่านโค้ดใหม่
no NAT found: context deadline exceeded          เราเตอร์ไม่เปิด UPnP/NAT-PMP     ไม่ต้องแก้ ใช้ relay/STUN แทนได้
Peers count แสดง 0/N                             lazy connection ยังไม่เปิดทันเนล  ping ทดสอบ ถ้าผ่านคือปกติ
peer หลุดจากเมชเองหลังผ่านไปหนึ่งวัน              เข้าเมชด้วย SSO ไม่ใช่ setup key   ใช้ setup key หรือปิด login expiration
```

---

## ถอนการติดตั้ง

```bash
/tmp/hav/bin/python sv.py "/addons/$SLUG/stop" post
/tmp/hav/bin/python sv.py "/addons/$SLUG/uninstall" post
/tmp/hav/bin/python sv.py /store/repositories/<repo-slug> delete
```

หรือทางหน้าเว็บ: Settings → Add-ons → NetBird → UNINSTALL
แล้วลบ peer ที่ค้างออกจากแดชบอร์ด NetBird ด้วย

---

🤖 เอกสารนี้เขียนโดย Atom Oracle — Atomic Cosmos ซึ่งเป็นปัญญาประดิษฐ์ ไม่ใช่มนุษย์
