# การทำ **Rate Limiting** 

(จำกัดจำนวนครั้งที่เรียกใช้งาน API) เป็นหนึ่งในฟีเจอร์ที่คุ้มค่าที่สุดของการเปลี่ยนมาใช้ OpenResty เพราะมันจัดการได้เร็วกว่าการไปเขียนดักในฝั่ง Application (เช่น Node.js หรือ PHP) มาก

OpenResty มีไลบรารีที่ชื่อว่า `lua-resty-limit-traffic` ติดตั้งมาให้เป็นมาตรฐานอยู่แล้ว เราสามารถเรียกใช้ได้ทันที มาดูวิธีทำทีละขั้นตอนกัน

---

### ขั้นตอนที่ 1: จองพื้นที่หน่วยความจำ (Shared Memory)

เพื่อให้ OpenResty จดจำได้ว่า IP ไหนยิง Request เข้ามาแล้วกี่ครั้ง เราต้องจองพื้นที่ RAM ให้ Nginx แชร์ข้อมูลกันระหว่าง Worker Process ก่อน

ให้เปิดไฟล์ `conf/nginx.conf` แล้วเพิ่มคำสั่ง `lua_shared_dict` ลงในบล็อก `http` :

```nginx
http {
    # จองพื้นที่ RAM 10MB ตั้งชื่อว่า "my_limit_store" เพื่อเก็บสถิติ IP
    # 10MB เก็บข้อมูล IP ได้หลักแสนไอพี สบายๆ 
    lua_shared_dict my_limit_store 10m; 

    server {
        listen 8080;
        
        # ... (เดี๋ยวเราจะแก้บล็อก location /api ในขั้นตอนต่อไป) ...

```

---

### ขั้นตอนที่ 2: เขียน Lua โค้ดดักจับก่อนทำงานจริง

เราจะใช้คำสั่ง `access_by_lua_block` ซึ่งจะทำงาน **ก่อน** `content_by_lua_block` เพื่อเป็นยามเฝ้าประตูกรองคนก่อนเข้าไปทำงานจริง

แก้ไขบล็อก `location /api` ของคุณให้เป็นแบบนี้:

```nginx
        location /api {
            default_type application/json;

            # --- ยามเฝ้าประตู: ระบบ Rate Limiting ---
            access_by_lua_block {
                -- เรียกใช้ไลบรารี Rate Limit
                local limit_req = require "resty.limit.req"

                -- ตั้งค่า: อนุญาต 2 request/วินาที, และยอมให้เกิน (Burst) ได้อีก 1 request
                local lim, err = limit_req.new("my_limit_store", 2, 1)
                if not lim then
                    ngx.log(ngx.ERR, "สร้าง Object ไม่สำเร็จ: ", err)
                    return ngx.exit(500)
                end

                -- ใช้ IP Address ของผู้ใช้เป็น Key ในการจำแนกคน
                local key = ngx.var.binary_remote_addr
                local delay, err = lim:incoming(key, true)

                -- ถ้า delay ไม่มีค่า แปลว่าโดนปฏิเสธ (ยิงรัวเกินโควต้า)
                if not delay then
                    if err == "rejected" then
                        ngx.status = 429 -- ตั้ง HTTP Status เป็น 429 Too Many Requests
                        ngx.say('{"status": "error", "message": "ใจเย็นๆ ยิงรัวเกินไปแล้ว (Rate limit exceeded)!"}')
                        return ngx.exit(429)
                    end
                    -- กรณีเกิด Error อื่นๆ
                    return ngx.exit(500)
                end

                -- ถ้ายิงมาติดช่วง Burst (โควต้าสำรอง) ให้หน่วงเวลา Request นี้แทนการปฏิเสธ
                if delay >= 0.001 then
                    ngx.sleep(delay)
                end
            }

            # --- โค้ดทำงานปกติ (จะมาถึงตรงนี้ได้ ต้องผ่านยามมาแล้ว) ---
            content_by_lua_block {
                local args = ngx.req.get_uri_args()
                local user_name = args.name or "Guest"
                ngx.say('{"status": "success", "message": "ยินดีต้อนรับ ' .. user_name .. ' สู่ API ของเรา"}')
            }
        }

```

---

### ขั้นตอนที่ 3: รีโหลดและทดสอบยิงรัวๆ

1. บันทึกไฟล์แล้วสั่ง Reload ให้ OpenResty อ่านค่าใหม่:

```bash
openresty -p ~/openresty-test/ -s reload

```

2. **ทดสอบยิง Request:**
เราจะใช้คำสั่ง `for` ใน Terminal เพื่อจำลองการยิง Request รัวๆ 5 ครั้งติดกันโดยไม่มีการหยุดพัก:

```bash
for i in {1..5}; do curl http://localhost:8080/api; echo ""; done

```

**ผลลัพธ์ที่คุณควรจะเห็น:**
ตัวแรกๆ จะผ่าน และตัวหลังๆ จะโดนตัดทิ้งด้วย HTTP 429 !

```json
{"status": "success", "message": "ยินดีต้อนรับ Guest สู่ API ของเรา"}
{"status": "success", "message": "ยินดีต้อนรับ Guest สู่ API ของเรา"}
{"status": "success", "message": "ยินดีต้อนรับ Guest สู่ API ของเรา"}
{"status": "error", "message": "ใจเย็นๆ ยิงรัวเกินไปแล้ว (Rate limit exceeded)!"}
{"status": "error", "message": "ใจเย็นๆ ยิงรัวเกินไปแล้ว (Rate limit exceeded)!"}

```

---

เพียงเท่านี้เซิร์ฟเวอร์ของคุณก็มีเกราะป้องกันการยิง Spam Request เบื้องต้นแล้ว แถมยังทำงานในระดับ C/LuaJIT ซึ่งกินทรัพยากรเครื่องน้อยมากๆ
