# Lerning openresty

**OpenResty** คือแพลตฟอร์มเว็บเซิร์ฟเวอร์ที่นำเอา **Nginx** (ซึ่งขึ้นชื่อเรื่องความเร็วและเสถียรภาพ) มารวมกับ **LuaJIT** (ตัวแปลภาษา Lua ที่ทำงานได้เร็วมาก) ทำให้เราสามารถเขียนโค้ดภาษา Lua ฝังลงไปใน Nginx ได้โดยตรง

ประโยชน์หลักๆ คือใช้ทำ API Gateway, จัดการ Rate Limiting, ตรวจสอบสิทธิ์ (Authentication) หรือแม้แต่ต่อกับ Database ได้ตรงๆ จากชั้นของ Web Server เลย

นี่คือคู่มือแบบทีละขั้นตอนสำหรับการเริ่มต้นใช้งาน OpenResty บนระบบปฏิบัติการ Linux (ตัวอย่างนี้อิงจาก Ubuntu/Debian เป็นหลักนะ)

---

### ขั้นตอนที่ 1: การติดตั้ง OpenResty

วิธีที่ง่ายที่สุดบน Ubuntu คือการติดตั้งผ่าน Repository อย่างเป็นทางการของ OpenResty:

1. เปิด Terminal แล้วรันคำสั่งเหล่านี้เพื่อเพิ่ม Repository:

```bash
sudo apt-get -y install software-properties-common
wget -qO - https://openresty.org/package/pubkey.gpg | sudo tee /etc/apt/trusted.gpg.d/openresty.asc
sudo add-apt-repository -y "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main"

```

2. อัปเดตและติดตั้ง OpenResty:

```bash
sudo apt-get update
sudo apt-get install -y openresty

```

*(หมายเหตุ: ไฟล์ติดตั้งและ Configuration ของ OpenResty มักจะถูกเก็บไว้ที่ `/usr/local/openresty/`)*

---

### ขั้นตอนที่ 2: ทำความรู้จักกับไฟล์ Configuration

หัวใจสำคัญของ OpenResty คือไฟล์ `nginx.conf` ซึ่งใช้ไวยากรณ์เดียวกับ Nginx ทั่วไป แต่เราสามารถแทรกบล็อกคำสั่งของ Lua ลงไปได้

ให้คุณเปิดหรือสร้างไฟล์คอนฟิก (สมมติว่าเราสร้างโฟลเดอร์สำหรับทดสอบแยกต่างหากเพื่อความปลอดภัย):

```bash
mkdir ~/openresty-test
cd ~/openresty-test
mkdir conf logs
nano conf/nginx.conf

```

---

### ขั้นตอนที่ 3: เขียน "Hello World" ด้วย Lua

คัดลอกโค้ดด้านล่างนี้ไปวางในไฟล์ `conf/nginx.conf` ที่คุณเพิ่งสร้าง:

```nginx
worker_processes  1;
error_log logs/error.log;

events {
    worker_connections 1024;
}

http {
    server {
        # ตั้งค่าให้ Server รันที่พอร์ต 8080
        listen 8080;

        # Location แรก: แสดงข้อความธรรมดา
        location / {
            default_type text/html;
            content_by_lua_block {
                ngx.say("<h1>Hello, OpenResty!</h1>")
                ngx.say("<p>นี่คือเว็บเซิร์ฟเวอร์ที่รันด้วย Lua</p>")
            }
        }
    }
}

```

**คำอธิบายโค้ด:**

* `content_by_lua_block`: เป็นคำสั่งที่บอก Nginx ว่าเนื้อหาในบล็อกนี้คือโค้ดภาษา Lua
* `ngx.say()`: เป็นฟังก์ชันของ OpenResty ใช้สำหรับพิมพ์ข้อความส่งกลับไปให้ผู้ใช้งาน (Client)

---

### ขั้นตอนที่ 4: สตาร์ทเซิร์ฟเวอร์และทดสอบ

ตอนนี้เราจะสั่งให้ OpenResty รันโดยใช้ไฟล์คอนฟิกที่เราเพิ่งสร้างขึ้น:

1. สตาร์ท OpenResty (ต้องระบุ path ของโฟลเดอร์ให้ถูกต้อง):

```bash
openresty -p ~/openresty-test/ -c conf/nginx.conf

```

2. ทดสอบผลลัพธ์ผ่าน Terminal โดยใช้ `curl` หรือเปิดเว็บบราวเซอร์ไปที่ `http://localhost:8080`:

```bash
curl http://localhost:8080

```

*คุณควรจะเห็นข้อความ HTML "Hello, OpenResty!" ตอบกลับมา*

---

### ขั้นตอนที่ 5: การใช้งานขั้นกว่า (สร้าง API ตอบกลับเป็น JSON และรับค่า พารามิเตอร์)

เพื่อให้เห็นภาพการใช้งานจริงมากขึ้น เรามาเพิ่ม API Endpoint ที่สามารถรับค่า (Query String) ได้

เปิดไฟล์ `conf/nginx.conf` อีกครั้ง แล้วเพิ่ม `location /api` เข้าไปในบล็อก `server` เดิม:

```nginx
        location /api {
            # กำหนดประเภทการตอบกลับเป็น JSON
            default_type application/json;
            
            content_by_lua_block {
                -- ดึงค่าพารามิเตอร์จาก URL
                local args = ngx.req.get_uri_args()
                
                -- ถ้ามีการส่ง ?name=... มาให้ใช้ค่านั้น ถ้าไม่มีให้ใช้ "Guest"
                local user_name = args.name or "Guest"
                
                -- สร้าง JSON string (ในงานจริงมักใช้ไลบรารี cjson)
                local response = '{"status": "success", "message": "Welcome, ' .. user_name .. '!"}'
                
                ngx.say(response)
            }
        }

```

**วิธีอัปเดตและทดสอบ:**

1. โหลดการตั้งค่าใหม่โดยไม่ต้องปิดเซิร์ฟเวอร์ (Reload):

```bash
openresty -p ~/openresty-test/ -s reload

```

2. ทดสอบเรียก API แบบไม่มีพารามิเตอร์:

```bash
curl http://localhost:8080/api
# ผลลัพธ์: {"status": "success", "message": "Welcome, Guest!"}

```

3. ทดสอบเรียก API แบบมีพารามิเตอร์:

```bash
curl http://localhost:8080/api?name=Gemini
# ผลลัพธ์: {"status": "success", "message": "Welcome, Gemini!"}

```

---
