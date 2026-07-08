# ระบบหลังบ้าน ร้านคุณยายเสื้อสวย — คู่มือทำให้ใช้งานได้จริง

เอกสารนี้ไล่ทีละ step จากไฟล์ prototype (mockup__2_.html) ไปจนถึงเว็บที่ใช้งานได้จริงในร้าน มีฐานข้อมูลจริง เก็บรูปได้ และ deploy ขึ้นออนไลน์ — ทั้งหมดใช้ของฟรี

## ภาพรวม stack ที่ใช้

| ส่วน | ใช้อะไร | ทำไม |
|---|---|---|
| ฐานข้อมูล | Supabase (Postgres) | ฟรี, เรียกจาก JS ตรงๆ ไม่ต้องเขียน server |
| เก็บรูปสินค้า | Supabase Storage | มาพร้อมกับ Supabase, ฟรี 1GB |
| Auth (ถ้าต้องการ) | Supabase Auth | กันคนนอกเข้ามาแก้ข้อมูล |
| โค้ด/version | GitHub | ฟรี, ย้อนกลับได้ถ้าพลาด |
| Deploy เว็บ | Vercel หรือ Netlify | ฟรี, ได้ลิงก์ HTTPS ใช้จริงบนมือถือ/แท็บเล็ต |

---

## STEP 1 — สร้างโปรเจกต์ Supabase

1. ไปที่ https://supabase.com → สมัคร (ใช้ GitHub login ได้เลย)
2. กด **New Project** → ตั้งชื่อ เช่น `rankhunyai-pos` → ตั้งรหัสผ่าน database (เก็บไว้ดีๆ) → เลือก region ใกล้ไทย (Singapore)
3. รอ 1-2 นาทีให้ project สร้างเสร็จ

## STEP 2 — สร้างตารางข้อมูล (SQL)

ไปที่เมนู **SQL Editor** ในโปรเจกต์ → รันโค้ดนี้ (คัดลอกไปวางแล้วกด Run):

```sql
-- ตารางเจ้าของสินค้า
create table owners (
  id uuid default gen_random_uuid() primary key,
  name text not null,
  avatar_color text default '#C99A3E',
  created_at timestamptz default now()
);

-- ตารางสินค้า
create table products (
  id uuid default gen_random_uuid() primary key,
  item_code text unique not null,       -- เช่น "0187"
  name text,
  category text,
  size text,
  grade text,                            -- A/B/C
  owner_id uuid references owners(id),
  list_price numeric,
  photo_url text,
  status text default 'available',       -- available / sold
  created_at timestamptz default now()
);

-- ตารางการขาย
create table sales (
  id uuid default gen_random_uuid() primary key,
  product_id uuid references products(id),
  sold_price numeric not null,
  payment_method text,                   -- cash / transfer
  shop_commission numeric,
  owner_payout numeric,
  sold_at timestamptz default now()
);

-- ตารางสรุปยอดต้องจ่ายเจ้าของ
create table payouts (
  id uuid default gen_random_uuid() primary key,
  owner_id uuid references owners(id),
  sale_id uuid references sales(id),
  amount numeric,
  is_paid boolean default false,
  paid_at timestamptz
);

-- ตัวรันเลขรหัสสินค้าอัตโนมัติ กันเลขซ้ำ
create sequence item_code_seq start 1;

-- ฟังก์ชันให้เว็บเรียกขอ "เลขรหัสถัดไป" แบบปลอดภัย (เว็บเรียกผ่าน supabase.rpc)
create or replace function get_next_item_code()
returns text as $$
declare
  next_val bigint;
begin
  next_val := nextval('item_code_seq');
  return lpad(next_val::text, 4, '0');
end;
$$ language plpgsql;
```

> จุดสำคัญ: `item_code_seq` คือตัวที่แก้ปัญหาเดิม (เลขรันเก็บแค่ในเบราว์เซอร์) — database จะเป็นคนออกเลขให้ ไม่ซ้ำแม้เปิดจากหลายเครื่องพร้อมกัน

เพิ่มเจ้าของเริ่มต้น (แก้ชื่อ/สีตามจริง):

```sql
insert into owners (name, avatar_color) values
  ('คุณแม่นิด', '#C99A3E'),
  ('ป้าจิ๋ว', '#5F8B6A'),
  ('น้องเอ', '#C1553B'),
  ('ลุงสมชาย', '#285C55'),
  ('ป้าน้อย', '#7A6FA6');
```

## STEP 3 — สร้างที่เก็บรูป (Storage bucket)

1. เมนู **Storage** → New bucket → ชื่อ `product-photos`
2. ตั้งเป็น **Public bucket** (ให้ดึงรูปมาแสดงในแอปได้ทันทีโดยไม่ต้อง auth)

## STEP 4 — เอา API key มาใช้

เมนู **Settings → API** จะเห็น 2 ค่าที่ต้องใช้:
- `Project URL`
- `anon public key`

เก็บไว้ ใช้ในขั้นต่อไป (เป็นคีย์ฝั่ง public ใส่ในหน้าเว็บได้ ไม่ใช่คีย์ลับ)

## STEP 5 — ต่อไฟล์ HTML เข้ากับ Supabase

ใส่ใน `<head>` ของ mockup__2_.html:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script>
  const supabase = window.supabase.createClient(
    'https://YOUR_PROJECT.supabase.co',
    'YOUR_ANON_PUBLIC_KEY'
  );
</script>
```

## STEP 6 — แก้ทีละหน้า (เรียงตามความสำคัญ)

ทำทีละหน้า ทดสอบให้ผ่านก่อนค่อยไปหน้าถัดไป:

1. **หน้าเพิ่มสินค้า** — เปลี่ยนปุ่ม "สร้างรหัส" ให้ดึงเลขจาก `item_code_seq`, เปลี่ยนกล่องถ่ายรูปเป็น `<input type="file" capture="environment">` แล้วอัปโหลดเข้า `product-photos`, ปุ่ม "บันทึกสินค้า" ให้ insert เข้าตาราง `products`
2. **หน้าขายสินค้า** — ดึงรายการสินค้าที่ `status = 'available'` มาแสดงแทนของ hardcode, ปุ่ม "ยืนยันการขาย" ให้ insert เข้า `sales` + `payouts` และ update สินค้าเป็น `sold`
3. **หน้าสรุปเจ้าของ** — query รวมยอดจากตาราง `payouts` ตาม `owner_id`, ปุ่ม "จ่ายแล้ว" ให้ update `is_paid`
4. **หน้าแดชบอร์ด** — query สรุปยอดขายวันนี้/เดือนนี้จากตาราง `sales`

> แนะนำ: ทำทีละหน้า commit ขึ้น GitHub ทีละครั้ง อย่าแก้ทุกหน้าพร้อมกันแล้วค่อยทดสอบ จะหา bug ยาก

## STEP 7 — เก็บโค้ดขึ้น GitHub

```bash
git init
git add .
git commit -m "initial POS prototype"
```

สร้าง repo ใหม่บน https://github.com → ทำตามคำสั่งที่ GitHub ให้มา (`git remote add origin ...` แล้ว `git push`)

## STEP 8 — Deploy ขึ้นออนไลน์ (ฟรี)

**วิธีที่ง่ายที่สุด (ไม่ต้องมี GitHub ก็ได้):**
1. ไปที่ https://app.netlify.com/drop
2. ลากไฟล์ HTML (ทั้งโฟลเดอร์) ไปวาง → ได้ลิงก์ทันที

**วิธีที่ดีกว่าถ้าจะแก้โค้ดต่อเรื่อยๆ:**
1. ไปที่ https://vercel.com หรือ https://netlify.com → login ด้วย GitHub
2. เลือก **Import Project** → เลือก repo ที่ push ไว้
3. กด Deploy — ทุกครั้งที่ push โค้ดใหม่ขึ้น GitHub เว็บจะ deploy ให้อัตโนมัติ

## STEP 9 — ทดสอบบนอุปกรณ์จริงที่จะใช้ในร้าน

เปิดลิงก์ที่ deploy แล้วบนมือถือ/แท็บเล็ตที่จะวางไว้หน้าร้านจริง ทดสอบ:
- [ ] เพิ่มสินค้า + ถ่ายรูปจากกล้องจริง
- [ ] ขายสินค้า แล้วเช็คว่าข้อมูลไปโผล่ใน Supabase table
- [ ] ปิด-เปิดเบราว์เซอร์ใหม่ ข้อมูลต้องยังอยู่ (ต่างจากตอนเป็น prototype ที่ข้อมูลหายตอนรีเฟรช)
- [ ] เปิดจาก 2 เครื่องพร้อมกัน เลขรหัสสินค้าต้องไม่ซ้ำกัน

## (ทางเลือกเสริม) กันคนนอกเข้าระบบ

ถ้าอยากล็อกไม่ให้คนแปลกหน้าเปิดลิงก์แล้วแก้ข้อมูลร้านได้:
- ใช้ **Supabase Auth** ทำหน้า login ง่ายๆ (email + password) ก่อนเข้าหน้าแอป
- หรือถ้าเอาแบบง่ายสุดสำหรับใช้ในครอบครัว: ตั้งรหัสผ่านเดียวเช็คฝั่ง JS ก่อนเข้าแอป (ป้องกันคนเดินผ่านมากดเล่นได้ระดับหนึ่ง แต่ไม่ปลอดภัยเท่า Auth จริง)

---

## สรุปลำดับทำงาน

```
Supabase (สร้าง project + ตาราง + storage)
        ↓
แก้โค้ด HTML ทีละหน้า ต่อกับ Supabase
        ↓
ทดสอบบนเครื่อง local (เปิดไฟล์ในเบราว์เซอร์)
        ↓
push ขึ้น GitHub
        ↓
deploy ผ่าน Vercel/Netlify
        ↓
ทดสอบบนมือถือ/แท็บเล็ตจริงที่จะใช้ในร้าน
```
