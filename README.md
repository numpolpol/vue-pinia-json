# Multi-language JSON Editor (Vue 3 + Pinia)

## คำอธิบาย

เว็บแอปนี้ช่วยให้คุณสามารถอัปโหลดไฟล์ JSON หลายภาษา (หลายไฟล์ที่มีโครงสร้างเดียวกัน) เพื่อแก้ไขข้อมูลทุกฟิลด์ (รวมถึงข้อมูลซ้อนและ array) ได้พร้อมกันในหน้าต่างเดียว สามารถเปรียบเทียบและแก้ไขค่าทุกภาษาได้อย่างสะดวก และดาวน์โหลดไฟล์ที่แก้ไขแล้วกลับไปใช้งาน

- รองรับการอัปโหลดไฟล์ JSON หลายไฟล์พร้อมกัน (เช่น th.json, en.json)
- แสดงข้อมูลแต่ละไฟล์แบบ side-by-side
- แก้ไขค่าทุกฟิลด์ (string, number, boolean) ทั้งแบบปกติ ซ้อน และ array
- รองรับ collapse/expand object/array ทุกระดับ
- มีปุ่ม "Paste" สำหรับวางค่าหลายภาษาในแต่ละฟิลด์
  - สามารถคัดลอกค่าจาก Google Sheet หรือโปรแกรมตารางอื่น ๆ แล้ววางลงในแต่ละฟิลด์ได้ทันที (รองรับการแยกค่าด้วย tab, space, หรือ newline)
- ถ้าค่าทุกไฟล์เหมือนกันจะแสดงแถวเดียว ถ้าต่างกันจะแสดงทุกภาษา
- ตรวจสอบและแจ้งเตือนหากไฟล์ JSON format ผิด
- สามารถ export ไฟล์ที่แก้ไขแล้วกลับไปใช้งานได้

## วิธีการใช้งาน

1. **อัปโหลดไฟล์ JSON หลายภาษา**

   - คลิกปุ่มเลือกไฟล์ แล้วเลือกไฟล์ .json หลายไฟล์ (เช่น th.json, en.json)
   - ถ้าไฟล์ไหน format ผิด จะมีข้อความแจ้งเตือนและไม่นำเข้าไฟล์นั้น

2. **คลิก Ready เพื่อเข้าสู่หน้าตารางแก้ไข**

3. **แก้ไขข้อมูล**

   - ตารางจะแสดง key และค่าของแต่ละไฟล์แบบ side-by-side
   - สามารถแก้ไขค่าทุกฟิลด์ได้ทันที
   - กดปุ่ม expand/collapse เพื่อดู/ซ่อนข้อมูลซ้อนหรือ array
   - กดปุ่ม "Paste" เพื่อวางค่าหลายภาษาในแต่ละฟิลด์ (สามารถคัดลอกค่าจาก Google Sheet หรือ Excel แล้ววางได้โดยตรง รองรับการแยกค่าด้วย tab, space, หรือ newline)

4. **Export ไฟล์ที่แก้ไขแล้ว**

   - คลิกปุ่ม Export All เพื่อดาวน์โหลดไฟล์ที่แก้ไขแล้วกลับไปใช้งาน

5. **กลับไปหน้าอัปโหลด**
   - คลิกปุ่ม Back เพื่อเลือกไฟล์ใหม่หรือเริ่มต้นใหม่

---

**หมายเหตุ**

- รองรับเฉพาะไฟล์ JSON ที่มีโครงสร้างเดียวกันเท่านั้น
- หากไฟล์ JSON มี key ซ้ำหรือ array ของ object จะ merge ตาม key เฉพาะ (เช่น code)
- หากพบปัญหาไฟล์ผิด format หรือข้อมูลไม่แสดง ให้ตรวจสอบโครงสร้าง JSON ให้ถูกต้องก่อน

---

This template should help get you started developing with Vue 3 in Vite.

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur).

## Type Support for `.vue` Imports in TS

TypeScript cannot handle type information for `.vue` imports by default, so we replace the `tsc` CLI with `vue-tsc` for type checking. In editors, we need [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) to make the TypeScript language service aware of `.vue` types.

## Customize configuration

See [Vite Configuration Reference](https://vite.dev/config/).

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```
