# 🚀 n8n + PostgreSQL + Ngrok (Docker Setup)

ระบบนี้เป็นการติดตั้ง **[n8n](https://n8n.io)** (workflow automation tool) ร่วมกับ **PostgreSQL** และ **Ngrok Tunnel**  
เพื่อให้สามารถใช้งาน n8n ได้ทั้งในเครื่องและเชื่อมต่อจากภายนอกได้อย่างปลอดภัยผ่าน HTTPS

---

## 🧱 โครงสร้างระบบ

```
📁 project-root/
 ├── .env                # เก็บค่าตัวแปรลับต่างๆ เช่น รหัสผ่าน DB และ Encryption Key
 ├── docker-compose.yml  # กำหนด services ทั้งหมด
 ├── postgres_data/      # โฟลเดอร์เก็บข้อมูล PostgreSQL (จะถูกสร้างอัตโนมัติ)
 └── n8n_data/           # โฟลเดอร์เก็บข้อมูลการตั้งค่าและ workflow ของ n8n
```

---

## ⚙️ การตั้งค่าไฟล์ `.env`

สร้างไฟล์ `.env` ที่ root ของโปรเจ็กต์ แล้วใส่ค่าตามด้านล่างนี้:

```bash
# PostgreSQL Credentials
POSTGRES_DB=n8n
POSTGRES_USER=admin
POSTGRES_PASSWORD=pg@12345

# n8n Encryption Key (สำคัญมาก ห้ามทำหาย)
# สร้างคีย์สุ่มยาวๆ ได้จาก: openssl rand -hex 32
N8N_ENCRYPTION_KEY=0123456789abcdef0123456789abcdef

# Timezone Settings
GENERIC_TIMEZONE=Asia/Bangkok
TZ=Asia/Bangkok

# ngrok Settings
# สมัคร ngrok ฟรีได้ที่: https://dashboard.ngrok.com/signup
NGROK_AUTHTOKEN=ngrok-authtoken-here
```

---

## 🐳 การตั้งค่า Docker Compose

```yaml
networks:
  n8n-network:
    driver: bridge

services:

  # PostgreSQL
  postgres:
    image: postgres:16
    container_name: n8n_postgres
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    networks:
      - n8n-network

  # n8n
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n_main
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${TZ}
      - N8N_HOST=localhost
    volumes:
      - ./n8n_data:/home/node/.n8n
    networks:
      - n8n-network
    depends_on:
      - postgres

  # Ngrok (สำหรับเปิด HTTPS tunnel)
  ngrok:
    image: ngrok/ngrok:latest
    container_name: n8n_ngrok_tunnel
    restart: unless-stopped
    environment:
      - NGROK_AUTHTOKEN=${NGROK_AUTHTOKEN}
    command: http n8n:5678
    ports:
      - "4040:4040"
    networks:
      - n8n-network
    depends_on:
      - n8n
```

---

## ▶️ วิธีเริ่มต้นใช้งาน

1. ติดตั้ง **Docker** และ **Docker Compose** ให้พร้อม  
   👉 [ดาวน์โหลด Docker Desktop](https://www.docker.com/products/docker-desktop/)

2. สร้างไฟล์ `.env` ตามตัวอย่างด้านบน

3. เปิด Terminal แล้วรันคำสั่ง:
   ```bash
   docker compose up -d
   ```

4. ตรวจสอบสถานะ container ทั้งหมด:
   ```bash
   docker ps
   ```

5. เข้าใช้งาน n8n ได้ที่:  
   🔗 [http://localhost:5678](http://localhost:5678)

6. หากต้องการเชื่อมต่อจากภายนอก (เช่น มือถือหรือระบบอื่น):
   - เปิดหน้า **Ngrok Web UI** ที่ [http://localhost:4040](http://localhost:4040)
   - จะเห็น URL เช่น `https://abcd1234.ngrok.io`  
     ใช้ลิงก์นี้เข้าถึง n8n จากภายนอกได้ทันที ✅

---

## 🔐 หมายเหตุสำคัญ

- อย่าลืม **เก็บค่า `N8N_ENCRYPTION_KEY` ไว้อย่างปลอดภัย**  
  เพราะใช้เข้ารหัส Credential ของ Workflow — หากหายจะไม่สามารถถอดรหัสค่าได้อีก
- แนะนำให้ **สำรองข้อมูลใน `n8n_data` และ `postgres_data`** เป็นระยะ
- ถ้าต้องการอัปเดต n8n ให้รัน:
  ```bash
  docker compose pull n8n
  docker compose up -d
  ```

---

## 🧩 คำสั่งที่มีประโยชน์

| คำสั่ง | คำอธิบาย |
|--------|------------|
| `docker compose up -d` | เริ่มระบบทั้งหมดแบบเบื้องหลัง |
| `docker compose down` | ปิดระบบทั้งหมด |
| `docker compose logs -f n8n` | ดู log แบบ real-time ของ n8n |
| `docker exec -it n8n_main bash` | เข้า shell ภายใน container n8n |
| `docker exec -it n8n_postgres psql -U admin -d n8n` | เข้า PostgreSQL console |

---

## 🧠 แหล่งข้อมูลเพิ่มเติม

- 🌐 [n8n Documentation](https://docs.n8n.io)
- 🐳 [Docker Hub – n8n](https://hub.docker.com/r/n8nio/n8n)
- 🌉 [Ngrok Dashboard](https://dashboard.ngrok.com)
