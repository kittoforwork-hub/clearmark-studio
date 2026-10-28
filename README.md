# ClearMark Studio — Local AI Inpainting

เว็บลบข้อความ/วัตถุจากภาพที่คุณเป็นเจ้าของหรือได้รับอนุญาตให้แก้ไข โดยใช้ AI Inpainting บนเบราว์เซอร์ ไม่ส่งไฟล์ภาพไป API หรือเซิร์ฟเวอร์ประมวลผลภาพ

## มีอะไรในโปรเจกต์นี้

- Static HTML/CSS/JavaScript: deploy ได้โดยตรงบน Cloudflare Pages
- MI-GAN ONNX สำหรับ inpainting (โมเดลระบุใบอนุญาต MIT)
- ใช้ WebGPU ก่อน และลอง WASM/CPU เมื่อ WebGPU เริ่มต้นหรือรัน inference ไม่สำเร็จ
- OCR ผ่าน Tesseract.js เพื่อเสนอข้อความที่น่าสงสัยบริเวณขอบภาพ (heuristic; ไม่ใช่โมเดลตรวจจับลายน้ำโดยตรง)
- ระบาย/คืนพื้นที่มาสก์ด้วยเมาส์ Undo/Redo, preview ก่อน/หลัง และดาวน์โหลด PNG
- ภาพที่อัปโหลดอยู่ใน memory ของเบราว์เซอร์เท่านั้น ไม่มี upload endpoint

## ข้อจำกัดที่ควรทราบ

1. **การตรวจจับอัตโนมัติเป็นเพียงตัวช่วย**: OCR ตรวจอ่านข้อความเป็นหลัก และเลือกข้อความที่อยู่ใกล้ขอบภาพตามกฎ heuristics จึงอาจพลาดโลโก้ล้วน ลายน้ำตรงกลาง ตัวอักษรบาง/จาง หรือเลือกข้อความสินค้ามาเป็นมาสก์ได้ ต้องตรวจสีแดงก่อนกด AI และปรับมาสก์เอง
2. MI-GAN เป็น image inpainting ไม่ใช่ watermark detector และไม่สามารถรับประกันการสร้างพิกเซลเดิมกลับมาได้ โดยเฉพาะข้อความ รุ่นสินค้า ตัวเลขสเกล โลโก้ ขอบโลหะ/ฟันเฟือง และลวดลายละเอียด
3. ครั้งแรกต้องเชื่อมต่ออินเทอร์เน็ตเพื่อดาวน์โหลด ONNX Runtime, Tesseract.js/ชุดภาษาอังกฤษ และโมเดล MI-GAN (~28 MB) จาก CDN/Hugging Face; ไฟล์ภาพต้นฉบับไม่ถูกส่งไปยังแหล่งเหล่านั้น
4. ถ้าเปิดในโหมด WebGPU ไม่ได้ ระบบจะลอง WASM ซึ่งอาจช้าหรือใช้หน่วยความจำสูง โมเดลอาจทำงานไม่ได้บนอุปกรณ์ที่มีหน่วยความจำต่ำ
5. ภาพที่มีด้านยาวเกิน 2048px จะถูกลดขนาดก่อนประมวลผล เพื่อหลีกเลี่ยงหน่วยความจำเต็ม ผลลัพธ์จะมีขนาดเท่ากับภาพที่ประมวลผล ไม่ใช่ภาพต้นฉบับขนาดเต็ม
6. ปุ่มพอดีหน้าจอใช้ปรับการแสดงผลบน canvas เท่านั้น ไม่เพิ่มรายละเอียดภาพ

## Deploy บน GitHub + Cloudflare Pages

### วิธี A — ผ่าน GitHub Desktop (ง่ายสุด)

1. ดาวน์โหลด ZIP โปรเจกต์นี้และแตกไฟล์
2. สร้าง repository ใหม่บน GitHub ชื่อ `clearmark-studio` (ตั้งเป็น Public หรือ Private ได้)
3. เปิด GitHub Desktop → **Add Existing Repository** (หรือสร้าง repository ในโฟลเดอร์นี้) และ publish/push ไฟล์ทั้งหมดขึ้น GitHub
4. เข้า Cloudflare Dashboard → **Workers & Pages** → **Create application** → **Pages** → **Connect to Git**
5. เลือก repository `clearmark-studio`
6. ตั้งค่า build:
   - Production branch: `main`
   - Framework preset: `None`
   - Build command: `exit 0` (หรือปล่อยว่าง หากหน้า UI อนุญาต)
   - Build output directory: `.` (โฟลเดอร์รากของ repository)
7. กด Deploy แล้วเปิด URL `*.pages.dev` ที่ระบบสร้างให้

### วิธี B — ผ่าน Git ใน Command Prompt

แตกไฟล์ ZIP แล้วเปิด terminal ในโฟลเดอร์ `ai-watermark-remover` จากนั้นใช้:

```bash
git init
git add .
git commit -m "Initial ClearMark Studio"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/clearmark-studio.git
git push -u origin main
```

สร้าง repo เปล่าชื่อ `clearmark-studio` บน GitHub ก่อนใช้คำสั่ง และแทน `YOUR-USERNAME` ด้วยชื่อบัญชีจริง จากนั้นเชื่อม repo ใน Cloudflare Pages ตามขั้นตอนด้านบน

## ทดสอบในเครื่อง

เพราะ WebGPU, worker และการโหลดโมเดลควรทำงานบน HTTP(S) อย่าเปิด `index.html` ด้วย `file://` โดยตรง หากติดตั้ง Python แล้วให้เปิด terminal ในโฟลเดอร์โปรเจกต์และรัน:

```bash
python -m http.server 8000
```

จากนั้นเปิด `http://localhost:8000` ใน Chrome หรือ Edge รุ่นใหม่

## ลำดับใช้งาน

1. อัปโหลดไฟล์ JPG/PNG/WebP ไม่เกิน 20 MB
2. กด **ตรวจจับข้อความที่ขอบภาพ** เพื่อให้ OCR เสนอพื้นที่เบื้องต้น หรือใช้แปรงระบายสีแดงเอง
3. ตรวจมาสก์สีแดงให้แน่ใจว่าไม่ครอบคลุมรหัสรุ่น/สเกล/ตัวอักษรที่ต้องรักษา
4. กด **เริ่มลบด้วย AI** แล้วรอโหลดโมเดลครั้งแรก
5. เปรียบเทียบภาพด้านซ้ายและขวา ใช้ Undo/Redo หากต้องกลับสถานะก่อนหน้า
6. ดาวน์โหลดผลลัพธ์ PNG

## Dependencies และใบอนุญาต

- MI-GAN ONNX mirror: [edgetools/migan](https://huggingface.co/edgetools/migan) — model card ระบุ MIT; ดูไฟล์ LICENSE ของแหล่งโมเดล
- ONNX Runtime Web: [onnxruntime-web](https://github.com/microsoft/onnxruntime)
- Tesseract.js: [tesseract.js](https://github.com/naptha/tesseract.js)

โค้ดตัวอย่างนี้ให้มาในสภาพ best-effort โปรดตรวจใบอนุญาตและเงื่อนไขของ dependency/โมเดลที่โหลด ณ เวลานำไปใช้จริง และใช้เพื่อแก้ไขภาพที่คุณเป็นเจ้าของหรือได้รับสิทธิ์เท่านั้น
