# Mars to Table BioSim

Repository นี้รวม BioSim core, Open MCT viewer, และชุดหน้าเว็บสำหรับผู้ใช้ฝั่ง plant/mission dashboard เพื่อทดลอง mission life support และ crop mission สำหรับงาน Mars-to-table / bioregenerative life support.

## Repository Layout

```txt
.
├── biosim/                         # BioSim Java server และ simulation core
│   ├── bin/start-biosim-server      # script สำหรับเปิด REST/WebSocket server
│   ├── configuration/               # ตัวอย่าง .biosim configuration
│   └── biosim-python-user/          # user-facing dashboards และ prototype tools
├── openmct-biosim/                  # Open MCT plugin สำหรับดู telemetry จาก BioSim
├── openmct/                         # Open MCT source/dependency workspace
├── Format Requriement Information.xlsx
└── Life Support Baseline Values and Assumptions Document.pdf
```

## Current System Summary

BioSim core มีระบบพื้นฐานเหล่านี้อยู่แล้ว:

- ปลูกพืชผ่าน `BiomassPS` และ `shelf`
- เก็บเกี่ยวอัตโนมัติเมื่อ crop ถึง harvest interval ผ่าน `autoHarvestAndReplant="true"`
- เก็บผลผลิตพืชเป็น biomass ใน `Biomass_Store`
- มี `Food_Store` สำหรับอาหารรวม
- crew กินอาหารจาก `Food_Store`
- มี `FoodProcessor` ใน Java core สำหรับแปลง `Biomass_Store -> Food_Store` แต่ template ของ Version3 ยังไม่ได้เปิดใช้

สิ่งที่ยังไม่มีใน core ตอนนี้:

- recipe รายเมนู
- schedule เช้า/เที่ยง/เย็น
- ingredient stock แยกชนิด เช่น lettuce, tomato, olive oil
- cooking/prep process ที่หักน้ำ ไฟ เวลา และ waste ตามเมนู

ไฟล์ `Format Requriement Information.xlsx` มี structure สำหรับต่อยอดส่วนนี้แล้ว ได้แก่ `Resource`, `Crop Information`, `Storage`, `Crop Performance`, และ `Process`.

## Prerequisites

ติดตั้งเครื่องมือเหล่านี้ก่อน:

- Java Development Kit 21+
- Maven
- Node.js และ npm สำหรับ `openmct-biosim`
- Python 3 สำหรับเปิด static web server ของ Version3

## Setup BioSim Server

จาก root repo:

```bash
cd biosim
mvn clean package
bin/start-biosim-server
```

ตรวจว่า server เปิดแล้ว:

```txt
http://localhost:8009/api/simulation
```

ถ้าถูกต้องจะเห็นประมาณนี้:

```json
{"simulations":[]}
```

ถ้าต้องการเก็บผล simulation ราย tick ลงไฟล์ ให้เปิดด้วย:

```bash
bin/start-biosim-server --writeTicks
```

ผลจะถูกเก็บที่:

```txt
biosim/logs/sim_<simId>/config.xml
biosim/logs/sim_<simId>/ticks.jsonl
```

และดูย้อนหลังได้ผ่าน:

```txt
http://localhost:8009/api/simulation/<simId>/log
```

## Setup Open MCT Viewer

เปิด terminal ใหม่จาก root repo:

```bash
cd openmct-biosim
npm install
npm start
```

เปิด:

```txt
http://localhost:9091/
```

ใน Open MCT จะมี object สำหรับ BioSim simulations และ telemetry graph ต่าง ๆ

## Setup Version3 Mission Builder

เปิด terminal ใหม่จาก root repo:

```bash
cd biosim/biosim-python-user/Version3
python3 -m http.server 8080
```

เปิด:

```txt
http://localhost:8080/mission_builder.html
```

ขั้นตอนใช้งาน:

1. เลือก crew information
2. เลือก `Crop Mission` หรือ `Baseline (No Crop)`
3. เลือก crop type
4. กด `Create Mission`
5. ระบบจะส่ง XML ไปที่ BioSim server และได้ `simId`
6. กด `Open Dashboard` เพื่อดู telemetry สด

Dashboard ใช้ WebSocket:

```txt
ws://localhost:8009/ws/simulation/<simId>
```

## Version3 Output Flow

ภาพรวมการไหลของข้อมูล:

```txt
mission_builder.html
  -> loads template.biosim
  -> edits crew/crop fields
  -> POST /api/simulation/start
  -> receives simId
  -> dashboard3.html?simId=<id>
  -> WebSocket live telemetry
```

โดย default ผลอยู่ใน memory ของ BioSim server และใน browser dashboard เท่านั้น ถ้าต้องการเก็บผลสำหรับ analysis ให้ใช้ `--writeTicks`.

## Crop Support

BioSim core เดิมรองรับ crop หลัก เช่น wheat, lettuce, soybean, tomato, potato ฯลฯ และในงานนี้เพิ่มกลุ่ม BVAD-derived salad crops เข้าไปแล้ว เช่น:

- Cabbage
- Carrot
- Chard
- Celery
- Green Onion
- Onion
- Pea
- Pepper
- Radish
- Red Beet
- Snap Bean
- Spinach
- Strawberry

ดูรายละเอียดฝั่ง user dashboard ได้ใน [biosim/biosim-python-user/README.md](biosim/biosim-python-user/README.md).

## Useful URLs

```txt
BioSim API:          http://localhost:8009/api/simulation
Open MCT:            http://localhost:9091/
Version3 Builder:    http://localhost:8080/mission_builder.html
Version3 Dashboard:  http://localhost:8080/dashboard3.html?simId=<id>
```

## Git Notes

ก่อน push แนะนำเช็กสถานะ:

```bash
git status
```

โดยทั่วไปไม่ควร commit:

- virtual environment เช่น `venv/`
- generated build output เช่น `target/`
- simulation logs ถ้าไม่ได้ตั้งใจเก็บเป็น artifact

แต่ควร commit:

- Java source ที่เพิ่ม crop/model
- `.biosim` template ที่ตั้งใจแก้
- README / user documentation
- dashboard HTML หรือ data files ที่ใช้รัน Version3
