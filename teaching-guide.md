# 📘 คู่มือปฏิบัติการ: ติดตั้ง CrowdSec ตรวจจับ SSH Brute Force บน AlmaLinux

> **สำหรับผู้เริ่มต้น** — เป็นสื่อการสอนแบบทำตามทีละขั้น (Step-by-step)
> ทุกคำสั่งมีคำอธิบายว่าทำอะไรและทำไมต้องทำ พร้อม Code Block ที่คุณสามารถ **Copy (กดปุ่ม 📋)** แล้วนำไปวางใน Terminal ได้ทันที

---

## 🔎 ภาพรวมระบบ (อ่านก่อนเริ่ม)

สิ่งที่เราจะสร้างคือระบบที่ช่วย **จับ IP ที่พยายาม Login SSH ผิดซ้ำ ๆ แล้วบล็อกอัตโนมัติ พร้อมแจ้งเตือนไปยัง Discord**

ระบบประกอบด้วย 4 ส่วนหลัก:

| ส่วน | ทำหน้าที่ |
| ---- | --------- |
| 1️⃣ **CrowdSec Engine** | อ่าน Log และวิเคราะห์พฤติกรรมผิดปกติ |
| 2️⃣ **SSH Collection** | ตรวจจับเหตุการณ์ที่เกี่ยวกับ SSH |
| 3️⃣ **Firewall Bouncer** | นำผลอนุมัติไปบล็อก IP ผ่าน Firewall |
| 4️⃣ **Discord Notification** | แจ้งเตือนผู้ดูแลระบบทันที |

### แผนผังลำดับการทำงาน

```
ผู้โจมตี Login SSH ผิดหลายครั้ง
         ↓
      SSHD   ← โปรแกรม SSH
         ↓
    SSH Log   ← บันทึก Log
         ↓
  CrowdSec Engine   ← อ่าน Log
         ↓
   SSH Collection    ← วิเคราะห์
         ↓
      Alert          ← พบเหตุการณ์
         ↓
    Decision         ← สั่ง Ban IP
         ↓
  Firewall Bouncer   ← ไปบล็อกที่ Firewall
         ↓
    Firewall Block   ← บล็อกจริง
         ↓
  Discord Notification ← แจ้งเตือนผู้ดูแล
```

---

# 🟦 ตอนที่ 1: ติดตั้ง CrowdSec Security Engine

ในตอนนี้เราจะติดตั้งตัว "สมอง" ของระบบ นั่นคือ CrowdSec Engine ทำหน้าที่อ่าน Log และวิเคราะห์พฤติกรรม

## 📌 ขั้นที่ 1.1 — เพิ่ม Repository ของ CrowdSec

ก่อนติดตั้ง เราต้องเพิ่มแหล่งซอฟต์แวร์ (Repository) ของ CrowdSec เข้าไปในระบบก่อน

```bash
curl -s https://install.crowdsec.net | sudo sh
```

> **ทำไมต้องทำ?** คำสั่งนี้จะดาวน์โหลดสคริปต์ติดตั้ง Repository ซึ่งทำให้ระบบรู้ว่าจะหาซอฟต์แวร์ CrowdSec ได้จากที่ไหน

## 📌 ขั้นที่ 1.2 — ติดตั้ง CrowdSec

```bash
sudo dnf install crowdsec -y
```

> **ทำไมต้องทำ?** คำสั่ง `dnf install` จะติดตั้งตัว CrowdSec Engine เข้ามาในระบบ
> **หมายเหตุ:** `-y` แปลว่า ยืนยันการติดตั้งอัตโนมัติ (ไม่ต้องกด Y ทุกครั้ง)

## 📌 ขั้นที่ 1.3 — เปิดใช้งาน CrowdSec

ติดตั้งเสร็จแล้วต้องสั่งให้บริการ (Service) เริ่มทำงานและเปิดใช้งานอัตโนมัติตอนบูตเครื่อง

```bash
sudo systemctl enable --now crowdsec
```

> **อธิบายคำสั่ง:**
> - `enable` = ให้เปิดใช้งานอัตโนมัติเมื่อเครื่องบูต
> - `--now` = เริ่มทำงานทันทีตอนนี้เลย

## 📌 ขั้นที่ 1.4 — ตรวจสอบว่าทำงานได้จริง

```bash
sudo systemctl status crowdsec
```

**ผลลัพธ์ที่ควรเห็น:**

```text
active (running)
```

### ✅ ตรวจสอบความสำเร็จ

ถ้าเห็นข้อความ `active (running)` แปลว่า:
- ยินดีด้วย! คราวนี่ CrowdSec ทำงานเรียบร้อยแล้ว
- ถ้าเห็น `inactive` หรือ `failed` ให้ย้อนกลับไปตรวจสอบขั้นตอนก่อนหน้า

---

# 🟦 ตอนที่ 2: ติดตั้ง Collection สำหรับ SSH

Collection เหมือน "โมดูล" ที่บอก CrowdSec ว่าควรมองหาอะไรใน Log SSH

## 📌 ขั้นที่ 2.1 — ติดตั้ง SSH Collection

```bash
sudo cscli collections install crowdsecurity/sshd
```

> **ทำไมต้องทำ?** Collection `crowdsecurity/sshd` มีกฎ (Scenario) สำหรับตรวจจับการพยายาม Login ผิดซ้ำ ๆ ผ่าน SSH โดยเฉพาะ

## 📌 ขั้นที่ 2.2 — ตรวจสอบว่า Collection ถูกติดตั้ง

```bash
sudo cscli collections list
```

**ผลลัพธ์ที่ควรเห็น** (ควรมีรายการดังนี้):

```text
crowdsecurity/sshd
```

---

# 🟦 ตอนที่ 3: ตรวจสอบว่า CrowdSec อ่าน Log ได้

แค่ติดตั้ง Collection ยังไม่พอ เราต้องมั่นใจว่า CrowdSec **อ่าน Log ของ SSH ได้จริง** (เรียกว่า Acquisition)

## 📌 ขั้นที่ 3.1 — ดูสถิติการอ่าน Log

```bash
sudo cscli metrics show acquisition
```

หรือดูภาพรวมทั้งหมด:

```bash
sudo cscli metrics
```

### ✅ ตรวจสอบความสำเร็จ

ตรวจสอบว่ามี:
- **Log Source ของ SSH** ปรากฏ
- **จำนวน Log ที่อ่าน** เพิ่มขึ้นเรื่อย ๆ

> ถ้าเห็น Log Source ของ SSH และจำนวนเพิ่มขึ้น แปลว่าระบบ Acquisition ทำงานถูกต้อง

---

# 🟦 ตอนที่ 4: ตรวจสอบ Alert และ Decision

## 📌 ขั้นที่ 4.1 — ดู Decision (IP ที่ถูกสั่ง Ban)

```bash
sudo cscli decisions list
```

> Decision = "คำสั่ง" ที่ CrowdSec ตั้งไว้ เช่น ให้ Ban IP นี้

> ⚠️ **ข้อควรรู้:** ในตอนนี้ CrowdSec Engine **แค่สร้าง Decision** แต่ยังไม่ได้บล็อก IP จริง ที่ Firewall ต้องรอการติดตั้ง **Bouncer** ในตอนที่ 7

## 📌 ขั้นที่ 4.2 — ดู Alert (เหตุการณ์ที่ตรวจพบ)

```bash
sudo cscli alerts list
```

> Alert = "บันทึก" เหตุการณ์ที่ CrowdSec พบว่าผิดปกติ

---

# 🟦 ตอนที่ 5: ทดสอบการตรวจจับ SSH Brute Force

ตอนนี้เราจะทดลองว่า CrowdSec จับ IP ที่พยายาม Login ผิด ได้จริงหรือไม่

## 📌 ขั้นที่ 5.1 — ทดสอบจากเครื่องอื่น

**จากเครื่อง/โทรศัพท์อื่น** ให้ลองเข้าสู่ระบบ SSH โดยใส่รหัสผ่านผิดหลาย ๆ ครั้ง:

```bash
ssh testuser@IP_ALMALINUX
```

> ใส่ `testuser` หรือ username ใดก็ได้ แล้วกรอกรหัสผ่านผิดติดต่อกันหลายครั้ง (10+ ครั้ง) 

> 💡 **เคล็ดลับ:** ถ้าไม่มีเครื่องอื่นทดสอบจริง อาจใช้คำสั่ง brute-force จำลอง แต่แนะนำให้ใช้วิธี Login จริงเพื่อให้เห็นผลชัดเจน

## 📌 ขั้นที่ 5.2 — กลับมาตรวจสอบ Alert

กลับมาที่เครื่อง AlmaLinux แล้วดู Alert:

```bash
sudo cscli alerts list
```

## 📌 ขั้นที่ 5.3 — ตรวจสอบ Decision

```bash
sudo cscli decisions list
```

### ✅ ตรวจสอบความสำเร็จ

ถ้าตรวจจับได้ จะเห็น **Alert** และ **Decision** ของ IP ที่มีพฤติกรรมน่าสงสัยปรากฏขึ้น

---

# 🟦 ตอนที่ 6: การปลดบล็อก IP

ถ้าอยากปลด IP ออกจาก Ban (เช่น IP ที่เป็นของตัวเองโดนแบนพลาด)

## 📌 ขั้นที่ 6.1 — ปลดบล็อก IP ที่ระบุ

```bash
sudo cscli decisions delete --ip <IP_ที่ต้องการปลดบล็อก>
```

ตัวอย่าง (ปลด IP `192.168.1.100`):

```bash
sudo cscli decisions delete --ip 192.168.1.100
```

## 📌 ขั้นที่ 6.2 — ลบ Decision ทั้งหมด

```bash
sudo cscli decisions delete --all
```

---

# 🟦 ตอนที่ 7: ติดตั้ง Firewall Bouncer

**ทำไมต้องมี Bouncer?** เพราะ CrowdSec Engine แค่สร้าง Decision แต่**ไม่บล็อก IP เอง** Bouncer จะคอยนำ Decision ไปสั่ง Firewall (nftables) ให้บล็อกจริง

## 📌 ขั้นที่ 7.1 — ติดตั้ง Bouncer

สำหรับระบบที่ใช้ nftables (เป็นค่าเริ่มต้นของ AlmaLinux):

```bash
sudo dnf install -y crowdsec-firewall-bouncer-nftables
```

## 📌 ขั้นที่ 7.2 — เปิดใช้งาน Bouncer

```bash
sudo systemctl enable --now crowdsec-firewall-bouncer
```

## 📌 ขั้นที่ 7.3 — ตรวจสอบสถานะ

```bash
sudo systemctl status crowdsec-firewall-bouncer
```

**ผลลัพธ์ที่ควรเห็น:**

```text
active (running)
```

## 📌 ขั้นที่ 7.4 — ตรวจสอบว่า Bouncer เชื่อมต่อกับ CrowdSec

```bash
sudo cscli bouncers list
```

---

# 🟦 ตอนที่ 8: ตั้งค่า Discord Notification

ตั้งค่าให้ CrowdSec ส่งข้อความแจ้งเตือนไปยัง Discord ทุกครั้งที่ Ban IP

## 📌 ขั้นที่ 8.1 — เปิดไฟล์ Notification

```bash
sudo nano /etc/crowdsec/notifications/http.yaml
```

## 📌 ขั้นที่ 8.2 — เพิ่ม Configuration

ให้เพิ่มเนื้อหาต่อไปนี้ลงในไฟล์ (แทนที่เนื้อหาเดิมหรือต่อท้าย):

```yaml
type: http

name: discord_default

log_level: info

format: |
  {
    "content": "🚨 **CrowdSec Alert** 🚨\n**Blocked IP:** {{range .}}{{.Source.IP}}{{end}}\n**Reason:** {{range .}}{{.Scenario}}{{end}}"
  }

timeout: 5s

method: POST

url: "https://discord.com/api/webhooks/YOUR_WEBHOOK_URL_HERE"

headers:
  Content-Type: application/json
```

## 📌 ขั้นที่ 8.3 — เปลี่ยน Webhook URL

แทนที่ข้อความ:

```text
YOUR_WEBHOOK_URL_HERE
```

ด้วย **Discord Webhook URL** ของช่อง ที่ต้องการรับการแจ้งเตือน (สร้างจาก Discord ➜ Server Settings ➜ Integrations ➜ Webhooks)

---

# 🟦 ตอนที่ 9: เชื่อม Notification เข้ากับ Profile

เราต้องบอก CrowdSec ว่า "เมื่อเกิดเหตุการณ์ ให้ทำ processing แล้วแจ้งเตือนผ่าน discord_default"

## 📌 ขั้นที่ 9.1 — เปิดไฟล์ Profile

```bash
sudo nano /etc/crowdsec/profiles.yaml
```

## 📌 ขั้นที่ 9.2 — เพิ่ม/แก้ไข Profile

```yaml
name: default_ip_remediation

filters:
  - Alert.Remediation == true && Alert.GetScope() == "Ip"

decisions:
  - type: ban
    duration: 4h

notifications:
  - discord_default

on_success: break
```

> ⚠️ **สำคัญมาก:** ชื่อ `discord_default` ในไฟล์นี้ **ต้องตรงกับ** ชื่อที่ตั้งไว้ในไฟล์ `/etc/crowdsec/notifications/http.yaml` เท่านั้นมันจึงจะเชื่อมกันได้

---

# 🟦 ตอนที่ 10: ตรวจสอบและรีสตาร์ท

## 📌 ขั้นที่ 10.1 — ตรวจสอบ Configuration

ก่อน restart ให้ตรวจว่าการตั้งค่าถูกต้อง (ไม่มี Error):

```bash
sudo crowdsec -t
```

## 📌 ขั้นที่ 10.2 — Restart CrowdSec

ถ้าไม่มี Error ให้รีสตาร์ทเพื่อให้การตั้งค่ามีผล:

```bash
sudo systemctl restart crowdsec
```

## 📌 ขั้นที่ 10.3 — ตรวจสอบสถานะอีกครั้ง

```bash
sudo systemctl status crowdsec
```

---

# 🟦 ตอนที่ 11: ทดสอบระบบทั้งหมด

## 📌 ขั้นที่ 11.1 — ทดสอบ Discord Notification

ทดสอบว่า Webhook ทำงานได้จริง:

```bash
sudo cscli notifications test discord_default
```

> ถ้าตั้งค่าถูกต้อง คุณควรได้รับข้อความแจ้งเตือนใน Discord

## 📌 ขั้นที่ 11.2 — ทดสอบ Firewall Bouncer

สร้าง Decision จำลอง เพื่อดูว่า Bouncer บล็อก IP ผ่าน Firewall จริงหรือไม่:

```bash
sudo cscli decisions add --ip 1.1.1.1 --type ban
```

## 📌 ขั้นที่ 11.3 — ตรวจสอบ Decision ถูกสร้าง

```bash
sudo cscli decisions list
```

## 📌 ขั้นที่ 11.4 — ตรวจสอบ Firewall Rules

```bash
sudo nft list ruleset
```

> ตรวจสอบว่ามี Rule ที่เกี่ยวข้องกับ CrowdSec ปรากฏใน Firewall

## 📌 ขั้นที่ 11.5 — ลบ IP ที่ใช้ทดสอบ

เมื่อทดสอบเสร็จ ให้ลบ Decision ทดสอบทิ้ง (กันอันตราย):

```bash
sudo cscli decisions delete --ip 1.1.1.1
```

## 📌 ขั้นที่ 11.6 — ตรวจสอบ Bouncer

```bash
sudo cscli bouncers list
```

> ถ้า Bouncer ทำงานปกติ จะพบรายการ Firewall Bouncer ในระบบ

---

# 🧰 ชุดคำสั่งสำคัญสำหรับตรวจสอบระบบ (เก็บไว้ใช้บ่อย)

```bash
# ตรวจสอบสถานะ CrowdSec
sudo systemctl status crowdsec

# ตรวจสอบ Collection
sudo cscli collections list

# ตรวจสอบ Log Acquisition
sudo cscli metrics show acquisition

# ดู Alert
sudo cscli alerts list

# ดู IP ที่ถูก Ban
sudo cscli decisions list

# ดู Bouncer
sudo cscli bouncers list

# ตรวจสอบ Firewall
sudo nft list ruleset
```

---

# 📋 สรุปหน้าที่ของแต่ละส่วนประกอบ

| ส่วนประกอบ | หน้าที่ |
| ---------- | ------- |
| CrowdSec Engine | อ่าน Log และตรวจจับพฤติกรรมผิดปกติ |
| SSH Collection | วิเคราะห์เหตุการณ์ที่เกี่ยวข้องกับ SSH |
| Alert | แจ้งว่า CrowdSec ตรวจพบเหตุการณ์ผิดปกติ |
| Decision | กำหนดการดำเนินการ เช่น Ban IP |
| Firewall Bouncer | นำ Decision ไปบล็อก IP ผ่าน Firewall |
| Discord Notification | ส่งการแจ้งเตือนไปยัง Discord |
| nftables | Firewall ที่ใช้บล็อกการเชื่อมต่อ |

---

# 🎯 สรุปสุดท้าย

เมื่อติดตั้งครบทุกตอน ระบบของคุณจะทำงานอัตโนมัติตามลำดับ:

**SSH Log → CrowdSec ตรวจจับ → Alert → Decision → Firewall Bouncer บล็อก IP → Discord แจ้งเตือนผู้ดูแลระบบ**

ซึ่งเป็นโครงสร้างหลักของระบบป้องกัน **SSH Brute Force** ด้วย CrowdSec บน AlmaLinux 🎉

---

## ✅ Checklist ส่งงาน (เวิร์กช็อป)

ใช้ช่องนี้ตรวจสอบว่าแต่ละตอนทำสำเร็จหรือไม่:

- [ ] **ตอนที่ 1** CrowdSec Engine ติดตั้งและทำงาน (`active (running)`)
- [ ] **ตอนที่ 2** ติดตั้ง `crowdsecurity/sshd` Collection แล้ว
- [ ] **ตอนที่ 3** CrowdSec อ่าน SSH Log ได้ (จำนวน Log เพิ่มขึ้น)
- [ ] **ตอนที่ 5** ทดสอบ SSH Brute Force แล้วพบ Alert + Decision
- [ ] **ตอนที่ 6** รู้วิธีปลดบล็อก IP
- [ ] **ตอนที่ 7** Firewall Bouncer ทำงาน และเชื่อมต่อสำเร็จ
- [ ] **ตอนที่ 8-9** ตั้งค่า Discord Webhook และ Profile เรียบร้อย
- [ ] **ตอนที่ 11** ทดสอบ Discord แจ้งเตือนสำเร็จ + Firewall มี Rule
