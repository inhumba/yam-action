# yam-action

GitHub Action ที่ส่งรายงานช่องโหว่ (CVE) และรายการ dependency ของ repo ไปที่ป้อมยาม (Yam) ระบบดูแลเว็บของ inhumba

- สแกน lockfile ทุกชุดใน repo (`pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `composer.lock`) ด้วย [osv-scanner](https://github.com/google/osv-scanner) v2.7.1 ที่ตรวจ SHA-256 ก่อนรัน
- ส่งแค่ชื่อ / เวอร์ชันของ package, ผลสแกน, runtime และ framework ไม่ส่งเนื้อหาไฟล์, `.env` หรือค่า secret ใดๆ
- ทุกคำขอเซ็น HMAC-SHA256 ด้วย secret ของแอปนั้น และตรวจลายเซ็นของคำตอบ
- ใช้ได้เฉพาะแอปที่ลงทะเบียนในป้อมยามแล้ว

## ตั้งค่าใน repo

ออก secret จากแท็บ CI ของแอปในป้อมยาม แล้วตั้งที่ Settings → Secrets and variables → Actions

| ชื่อ | ประเภท | ค่า |
| --- | --- | --- |
| `YAM_SECRET` | Secret | secret ที่แสดงครั้งเดียวในป้อมยาม (ห้ามใช้ org secret ร่วมกับ repo อื่น) |
| `YAM_SITE_ID` | Variable | site id ของแอป |
| `YAM_PROD_BRANCH` | Variable | ใส่เมื่อ branch ที่ขึ้น PROD ไม่ใช่ default branch |
| `YAM_ENV_MAP` | Variable | ใส่เมื่อมี UAT เช่น `main=prod,develop=uat` |

## `.github/workflows/yam.yml`

```yaml
name: yam
on:
  push:
    branches: [main] # branch ที่ deploy เช่น [main, develop]
  schedule:
    - cron: "17 20 * * *" # 03:17 น. เวลาไทย
  workflow_dispatch:
permissions:
  contents: read
concurrency:
  group: yam-${{ github.ref }}
  cancel-in-progress: true
jobs:
  report:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event_name == 'schedule' && vars.YAM_PROD_BRANCH || github.ref }}
      - uses: inhumba/yam-action@v1
        with:
          site-id: ${{ vars.YAM_SITE_ID }}
          secret: ${{ secrets.YAM_SECRET }}
          portal: <ที่อยู่ agent ของป้อมยาม>
          env-map: ${{ vars.YAM_ENV_MAP }}
```

- `yam.yml` ต้องอยู่บน default branch ด้วย GitHub รัน schedule และแสดงปุ่ม Run workflow จากไฟล์บน default branch เท่านั้น
- `YAM_ENV_MAP` ซ้ายเป็นชื่อ branch ขวาเป็น `prod` หรือ `uat` เช่น branch `production` กับ `uat` → `production=prod,uat=uat`
- branch ที่ไม่อยู่ใน `YAM_ENV_MAP` (หรือไม่ใช่ default branch เมื่อไม่ตั้ง) จะไม่ส่งรายงานและไม่ล้ม
- ต้องมี lockfile ใน repo (`pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `composer.lock`) ถ้าไม่มี log จะเตือนว่าไม่ได้ตรวจ dependency
- เจอช่องโหว่ไม่ทำให้ job ล้ม job ล้มเมื่อส่งรายงานไม่สำเร็จ ข้อความใน log บอกวิธีแก้
- ผลสรุปอยู่ใน job summary

ต้นฉบับของ action นี้อยู่ใน repo ส่วนตัวของ Yam ไฟล์ใน repo นี้สร้างจากสคริปต์ release ห้ามแก้ตรงนี้

## YamScan

ถ้าเห็น User-Agent `YamScan/1` ใน log ของเว็บ: เป็นตัวตรวจของทีมที่ดูแลเว็บนั้น (ระบบ Yam ของ inhumba) รันวันละครั้งจาก GitHub Actions

- ขอด้วย GET อย่างเดียว ทีละคำขอ ไม่กี่สิบ path ต่อวัน เพื่อดูว่ามีไฟล์ที่ไม่ควรเปิดให้คนนอกอ่าน (เช่น `.env`, `.git`, log) หลุดออกมาไหม
- ไม่ส่ง POST ไม่ลองรหัสผ่าน ไม่เก็บเนื้อหาที่ดึงมา
- ถ้าต้องการให้หยุดตรวจเว็บของคุณ ติดต่อผู้ดูแลเว็บของคุณ หรือเปิด issue ใน repo นี้

