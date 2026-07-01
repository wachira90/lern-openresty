# Redis Cache Authen

การจับคู่ OpenResty กับ **Redis** เป็นหนึ่งในท่ามาตรฐานที่ทรงพลังมาก เพราะทั้งคู่ทำงานบนหน่วยความจำ (RAM) จึงตอบสนองได้ในระดับมิลลิวินาที (Milliseconds)

เพื่อให้ทำตามได้จริง ก่อนอื่นคุณต้องมี Redis รันอยู่ในเครื่องก่อนนะ (ถ้าใช้ Ubuntu ติดตั้งง่ายๆ ด้วย `sudo apt-get install redis-server`)

OpenResty มีไลบรารี `lua-resty-redis` ติดตั้งมาให้แล้ว เรามาดูวิธีเขียนโค้ดสำหรับทั้ง 2 Use Case กันเลย

---

### 1. การทำระบบ Caching ระดับเซิร์ฟเวอร์ (Server-Level Caching)

หลักการคือ: เมื่อมี Request เข้ามา Nginx จะวิ่งไปถาม Redis ก่อน ถ้ามีข้อมูล (Cache Hit) จะตอบกลับทันทีโดยไม่กวน Backend แต่ถ้าไม่มี (Cache Miss) ถึงจะไปดึงข้อมูลใหม่มาเก็บลง Cache

ให้เพิ่มบล็อก `location /profile` ในไฟล์ `conf/nginx.conf` ของคุณ:

```nginx
        location /profile {
            default_type application/json;
            
            content_by_lua_block {
                -- 1. เรียกใช้ไลบรารี Redis
                local redis = require "resty.redis"
                local red = redis:new()
                
                -- ตั้งเวลา Timeout (มิลลิวินาที)
                red:set_timeouts(1000, 1000, 1000) 
                
                -- เชื่อมต่อ Redis (ค่าเริ่มต้นคือ localhost พอร์ต 6379)
                local ok, err = red:connect("127.0.0.1", 6379)
                if not ok then
                    ngx.say('{"error": "ไม่สามารถเชื่อมต่อ Redis ได้: ' .. err .. '"}')
                    return
                end

                -- 2. รับค่า id จาก URL เช่น /profile?id=5
                local user_id = ngx.var.arg_id or "1"
                local cache_key = "user_profile:" .. user_id

                -- 3. ลองดึงข้อมูลจาก Redis
                local res, err = red:get(cache_key)

                if res == ngx.null then
                    -- กรณีที่ 1: Cache Miss (ไม่มีข้อมูลใน Redis)
                    -- ในระบบจริง ตรงนี้คุณอาจจะต้องยิงไปหา Database หรือ Backend API
                    -- แต่เพื่อการทดสอบ เราจะจำลองข้อมูลขึ้นมา
                    local mock_data = '{"id": ' .. user_id .. ', "name": "User_' .. user_id .. '", "source": "Database"}'
                    
                    -- บันทึกข้อมูลลง Redis และตั้งเวลาหมดอายุ (TTL) เป็น 60 วินาที
                    red:set(cache_key, mock_data)
                    red:expire(cache_key, 60)
                    
                    ngx.say(mock_data)
                else
                    -- กรณีที่ 2: Cache Hit (มีข้อมูลใน Redis)
                    -- เราแอบเปลี่ยนคำว่า source เพื่อให้เห็นชัดว่าดึงมาจาก Cache
                    local cached_data = string.gsub(res, "Database", "Redis Cache")
                    ngx.say(cached_data)
                end

                -- 4. คืน Connection กลับเข้า Pool (สำคัญมาก เพื่อไม่ให้ Connection เต็ม)
                red:set_keepalive(10000, 100)
            }
        }

```

**วิธีทดสอบ:**

* ยิงครั้งแรก: `curl http://localhost:8080/profile?id=99` (จะได้ผลลัพธ์ `"source": "Database"`)
* ยิงครั้งที่สองภายใน 60 วินาที: คุณจะได้ `"source": "Redis Cache"` ซึ่งตอบกลับมาไวมาก!

---

### 2. การเช็คสิทธิ์ (Authentication) ด้วย Redis

หลักการคือ: ลูกค้าส่ง Token มาทาง HTTP Headers (`Authorization: Bearer <token>`) จากนั้น OpenResty (Nginx) จะเอา Token นี้ไปเช็คใน Redis ว่ามีอยู่จริงและยังไม่หมดอายุใช่หรือไม่ **ก่อน** ที่จะอนุญาตให้เข้าถึง API

ให้เพิ่มบล็อก `location /secure-api` ลงไป:

```nginx
        location /secure-api {
            default_type application/json;

            # ใช้ access_by_lua_block เพื่อทำหน้าที่เป็น "ยาม" ตรวจบัตร
            access_by_lua_block {
                -- 1. ดึง Headers มาดู
                local headers = ngx.req.get_headers()
                local auth_header = headers["Authorization"]
                
                -- ถ้าไม่ได้แนบ Header มา ให้เตะออกด้วย HTTP 401
                if not auth_header then
                    ngx.status = 401
                    ngx.say('{"error": "Unauthorized: Missing Token"}')
                    return ngx.exit(401)
                end

                -- 2. ตัดคำว่า "Bearer " ออก เพื่อเอาแค่ตัว Token (7 ตัวอักษร + 1 space)
                local token = string.sub(auth_header, 8)

                -- 3. เชื่อมต่อ Redis
                local redis = require "resty.redis"
                local red = redis:new()
                red:connect("127.0.0.1", 6379)
                
                -- 4. เช็คว่า Token นี้มีใน Redis ไหม (สมมติว่าเราเก็บ key เป็น "session:<token>")
                local user_id, err = red:get("session:" .. token)
                
                if user_id == ngx.null or not user_id then
                    ngx.status = 401
                    ngx.say('{"error": "Unauthorized: Invalid or Expired Token"}')
                    return ngx.exit(401)
                end

                -- 5. ถ้ายืนยันตัวตนผ่าน! เราสามารถฝัง user_id เข้าไปใน Header 
                -- ส่งต่อให้ Backend ของจริง (ถ้ามี) เอาไปใช้งานต่อได้เลย
                ngx.req.set_header("X-User-Id", user_id)
                
                -- คืน Connection เข้า Pool
                red:set_keepalive(10000, 100)
            }

            # โค้ดนี้จะทำงานก็ต่อเมื่อผ่าน access_by_lua_block มาได้เท่านั้น
            content_by_lua_block {
                -- ดึง User ID ที่เราเพิ่งฝังไว้ใน Header ออกมาแสดง
                local user_id = ngx.req.get_headers()["X-User-Id"]
                ngx.say('{"status": "success", "message": "Access Granted!", "user_id": "' .. user_id .. '"}')
            }
        }

```

**วิธีทดสอบ:**

1. จำลองการ Login โดยสร้าง Token ปลอมๆ ใน Redis ผ่าน Terminal ก่อน:

```bash
redis-cli SET session:mysecrettoken "User_12345"

```

2. รีโหลด OpenResty (`openresty -p ~/openresty-test/ -s reload`)
3. ทดสอบเข้า API แบบไม่มี Token (จะโดนปฏิเสธ):

```bash
curl http://localhost:8080/secure-api

```

4. ทดสอบเข้า API แบบมี Token ที่ถูกต้อง:

```bash
curl -H "Authorization: Bearer mysecrettoken" http://localhost:8080/secure-api

```

*(คุณควรจะเห็นข้อความ Access Granted พร้อมกับ user_id ที่ดึงมาจาก Redis)*

---

ตอนนี้เซิร์ฟเวอร์ของคุณสามารถใช้ Redis จัดการทั้ง Cache และ Authentication ดักที่หน้าบ้านได้แล้ว ซึ่งจะช่วยลดภาระของ Database และ Backend หลักไปได้มหาศาล
