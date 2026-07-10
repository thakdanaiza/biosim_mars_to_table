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

## Meal, Ingredient, and Process Extension

ส่วนนี้คือแนวทางเพิ่มระบบอาหารรายมื้อจากไฟล์ `Format Requriement Information.xlsx` เข้ากับ BioSim โดยแยกเป็น 3 ชั้นหลัก:

```txt
Ingredient Stock
  -> Meal Schedule
  -> Process Consumption
  -> BioSim Stores / Logs / Dashboard
```

เป้าหมายคือทำให้ระบบรู้ว่าในแต่ละ sol มีมื้อเช้า เที่ยง เย็น ใช้เมนูอะไร ต้องใช้วัตถุดิบเท่าไร และกระบวนการทำอาหารต้องใช้น้ำ ไฟ เวลา crew และสร้าง waste เท่าไร

### 1. Ingredient Stock

`Ingredient Stock` คือคลังวัตถุดิบแยกชนิด ไม่ใช่ `Food_Store` รวมแบบ BioSim เดิม เช่น:

```txt
Lettuce_Stock
Tomato_Stock
Bell_Pepper_Stock
Olive_Oil_Stock
Vinegar_Stock
Dried_Herbs_Stock
```

แหล่งข้อมูลตั้งต้นมาจาก workbook:

- sheet `Resource`: รายชื่อวัตถุดิบต่อเมนู ปริมาณ และหน่วย
- sheet `Storage`: วิธีเก็บ อุณหภูมิ shelf life และ hardware
- sheet `Crop Information`: ถ้า ingredient มาจาก greenhouse ให้ map ไป crop type

ตัวอย่าง data shape ที่แนะนำ:

```json
{
  "ingredientId": "lettuce",
  "name": "Lettuce",
  "origin": "Greenhouse",
  "unit": "g",
  "initialAmount": 0,
  "storage": {
    "method": "Refrigerated",
    "temperatureC": "0-4",
    "relativeHumidityPercent": "95-100"
  },
  "cropType": "LETTUCE"
}
```

ขั้นตอนเพิ่ม ingredient ใหม่:

1. เพิ่มแถวใน sheet `Resource` ว่าเมนูใช้วัตถุดิบอะไร ปริมาณเท่าไร
2. เพิ่มข้อมูลเก็บรักษาใน sheet `Storage`
3. ถ้าเป็นวัตถุดิบที่ปลูกเอง ให้เพิ่มหรือ map ข้อมูลใน sheet `Crop Information`
4. สร้าง/อัปเดต data file เช่น `meal_plan.json` หรือ `ingredients.json`
5. ให้ dashboard อ่าน stock ปัจจุบันและแสดงว่า stock พอสำหรับมื้อต่อไปหรือไม่

การทำงานที่ควรเกิดตอน simulation รัน:

```txt
Crop harvested
  -> Biomass_Store / Ingredient Stock เพิ่มขึ้น

Earth supply loaded
  -> Ingredient Stock เพิ่มขึ้น

Meal prepared
  -> Ingredient Stock ลดลงตาม recipe
```

หมายเหตุ: BioSim core ตอนนี้มี `Food_Store` และ `Biomass_Store` แต่ยังไม่มี ingredient stock แยกชนิด ดังนั้นระยะแรกสามารถทำเป็น data layer ใน Version3 ก่อน แล้วค่อยต่อเป็น Java module ภายหลัง

### 2. Meal Schedule

`Meal Schedule` คือแผนว่า sol ไหน เวลาไหน ต้องทำเมนูอะไร เช่น:

```txt
Sol 1 Breakfast -> Oatmeal
Sol 1 Lunch     -> Garden Salad
Sol 1 Dinner    -> Rice and Beans
Sol 2 Breakfast -> ...
```

แหล่งข้อมูลตั้งต้นมาจาก workbook:

- sheet `Process`: คอลัมน์ `Sol Day(s) Served`, `Meal Type`, `Meal Name`
- sheet `Resource`: ingredients ต่อ `Meal ID`

ตัวอย่าง data shape ที่แนะนำ:

```json
{
  "sol": 1,
  "mealType": "Lunch",
  "mealId": "M00",
  "mealName": "Garden Salad",
  "servedAtTick": 720
}
```

หลักการแปลงเวลา:

- BioSim ใช้ tick เป็นเวลาจำลอง
- ถ้า `tickLength="1"` และคิด 1 tick = 1 นาที สามารถกำหนดประมาณนี้:

```txt
Breakfast = Sol start + 480 ticks   # 08:00
Lunch     = Sol start + 720 ticks   # 12:00
Dinner    = Sol start + 1080 ticks  # 18:00
```

ตัวอย่าง logic:

```txt
currentSol = floor(ticksGoneBy / 1440) + 1
minuteInSol = ticksGoneBy % 1440

if currentSol == schedule.sol
and minuteInSol == servedAtMinute
then trigger meal process
```

ขั้นตอนเพิ่ม meal schedule ใหม่:

1. เพิ่ม `Meal ID`, `Meal Name`, `Sol Day(s) Served`, `Meal Type` ใน sheet `Process`
2. เพิ่ม ingredient ของเมนูนั้นใน sheet `Resource`
3. export หรือแปลงเป็น `meal_schedule.json`
4. dashboard หรือ scheduler อ่าน schedule แล้ว trigger process เมื่อถึง tick

### 3. Process Consumption

`Process Consumption` คือการหัก resource ตอนทำอาหาร เช่น วัตถุดิบ น้ำ ไฟ เวลา crew และ waste

แหล่งข้อมูลตั้งต้นมาจาก workbook:

- sheet `Process`: `Power Required`, `Water Required`, `Prep Time`, `Cook Time`, `Cleaning Resources`, `Waste Type`, `Waste Amount`
- sheet `Resource`: amount ของ ingredient ต่อเมนู

ตัวอย่าง Garden Salad จาก workbook:

```txt
Lettuce 100 g
Tomato 80 g
Bell Pepper 50 g
Olive Oil 15 mL
Vinegar 10 mL
Dried Herbs 2 g

Water Required: 0.5 L ต่อ fresh ingredient
Power Required: 0.0 kWh
Waste: Organic Waste 20 g ต่อ fresh ingredient
```

ตัวอย่าง data shape ที่แนะนำ:

```json
{
  "mealId": "M00",
  "mealName": "Garden Salad",
  "ingredients": [
    {"ingredientId": "lettuce", "amount": 100, "unit": "g"},
    {"ingredientId": "tomato", "amount": 80, "unit": "g"},
    {"ingredientId": "bell_pepper", "amount": 50, "unit": "g"}
  ],
  "process": {
    "powerKWh": 0.0,
    "waterL": 1.5,
    "prepTimeMin": 10,
    "cookTimeMin": 0,
    "cleaningTimeMin": 13,
    "waste": [
      {"type": "Organic Waste", "amount": 60, "unit": "g"}
    ]
  }
}
```

ขั้นตอนการทำงานตอนถึงมื้ออาหาร:

```txt
1. ตรวจว่า Ingredient Stock มีวัตถุดิบพอไหม
2. ถ้าไม่พอ ให้แจ้ง meal readiness = NOT READY
3. ถ้าพอ ให้หัก ingredient ตาม recipe
4. หัก Potable_Water_Store ตาม process water
5. หัก General_Power_Store ตาม process power
6. เพิ่ม Dry_Waste_Store หรือ Organic Waste ตาม waste amount
7. บันทึก meal event ลง log/dashboard
8. ถ้าต้องการให้ crew ได้พลังงาน ให้เพิ่มหรือถือว่า consume จาก Food_Store ตาม meal calories
```

การต่อกับ BioSim ทำได้ 2 ระดับ:

ระดับแรก ทำใน Version3/dashboard:

- อ่าน `meal_plan.json`
- แสดง current/next meal
- คำนวณ stock, water, power, waste แบบ side calculation
- ยังไม่หัก resource จริงจาก Java core

ระดับสอง ทำเป็น BioSim-integrated:

- เพิ่ม Java module เช่น `MealProcessor`
- module นี้เป็น `FoodConsumer`, `PotableWaterConsumer`, `PowerConsumer`, `DryWasteProducer`
- เมื่อถึง scheduled tick ให้ module หัก resource จาก stores จริง
- expose state ผ่าน REST/WebSocket เพื่อให้ dashboard แสดงผล

### Suggested File Structure

สำหรับเริ่มทำใน Version3 แนะนำเพิ่มไฟล์ data แยกก่อน:

```txt
biosim-python-user/Version3/data/
├── ingredients.json
├── recipes.json
├── meal_schedule.json
└── process_requirements.json
```

แล้วค่อยให้ `dashboard3.html` หรือไฟล์ JS ใหม่อ่าน data เหล่านี้

### End-to-End Example

```txt
Sol 1, 12:00 Lunch
  -> Schedule says M00 Garden Salad
  -> Recipe needs lettuce, tomato, bell pepper, olive oil, vinegar, herbs
  -> Ingredient Stock is checked
  -> Process consumes 1.5 L potable water and 0.0 kWh power
  -> Organic waste is added
  -> Dashboard logs "Garden Salad prepared"
```

ภาพรวมปลายทาง:

```txt
Crop Production
  -> Harvest
  -> Ingredient Stock
  -> Meal Schedule
  -> Process Consumption
  -> Crew Food / Waste / Water / Power telemetry
```

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
