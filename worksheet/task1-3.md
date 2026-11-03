# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Chayutphong Soisri

**Student ID / รหัสนิสิต:** 6630054621

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 496 | 594 | 93.5% | 100% |
| t0-small | 7 | 20 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

>Large model ให้ accuracy สูงขึ้นจาก 72.5% เป็น 93.5% (+21 percentage points)แต่แลกกับ latency ที่สูงขึ้น โดย p50 เพิ่มจาก 7 เป็น 496 ms และ p95 จาก 20 เป็น 594 ms อีกทั้งเรียกใช้ large model 100% จึงมีต้นทุนการประมวลผลสูงกว่า small model

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 28.5 | 227 | 348 | 20 | 0 | 92.8% |
| t1-after | 9.3 | 399 | 1011 | 0 | 27 | 93.1% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

>ค่า p50 ที่ต่ำลงไม่ได้แปลว่าระบบทำงานดีขึ้น เพราะ request ที่ล้มเหลวรอจนครบ 10 วินาทีและไม่ถูกรวมในการคำนวณ latency อีกทั้งยังยึด client concurrency ไว้ ทำให้ request ที่เหลือแย่ง large model น้อยลง จึงเห็น p50 ต่ำลงทั้งที่ระบบให้ผลลัพธ์ไม่สำเร็จ คอลัมน์ที่บอกความจริงคือ errors = 20 ซึ่งแสดงว่าระบบยังมี request ที่ล้มเหลว

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

>เพราะ request ที่ค้างยังยึดทรัพยากรใน review service และ gateway เช่น worker, connection และ slot สำหรับรับ request เมื่อ request ค้างสะสม ทรัพยากรจะเต็ม ทำให้ request อื่นต้องต่อคิวหรือ timeout ไปด้วย ผลกระทบจึงลามไปทั้งระบบ ไม่ได้จำกัดแค่ 10% ของรีวิวที่เจอการค้าง

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

>ควรบันทึกรีวิวที่ pending ไว้ แล้วให้ background worker หรือ batch job นำไปประมวลผลซ้ำเพื่อติด label เมื่อสำเร็จจึงเปลี่ยนสถานะเป็น scored โดยใช้ topology T4: Asynchronous / batch

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

>หากตั้ง timeout เป็น 0.1 วินาที ซึ่งสั้นกว่าเวลาตอบปกติของ large model จะทำให้ request ที่ทำงานปกติถูกตัดจบและกลายเป็น pending จำนวนมาก แต่หากตั้งเป็น 10 วินาที เมื่อ model ค้าง request จะยึด worker และ connection ไว้นาน ทำให้คิวสะสมและกระทบ request อื่น รวมถึงทำให้ผู้ใช้ต้องรอนานกว่าจะได้รับผลลัพธ์ fallback

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 6 | 553 | 91.5% | 38% |
| t2-0.7 | 0.7 | 370 | 593 | 93.0% | 53% |
| t2-0.8 | 0.8 | 499 | 711 | 93.5% | 64% |
| t2-0.9 | 0.9 | 477 | 623 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

>เลือก threshold 0.6 เพราะได้ accuracy 91.5% และเรียก large model เพียง 38% ช่วยลดต้นทุนและทำให้รีวิวส่วนใหญ่ได้ผลลัพธ์เร็ว โดยมี p50 เพียง 6 ms สิ่งที่ยอมเสียคือ accuracy ซึ่งลดลง 2 percentage points จาก baseline large model ที่ 93.5%

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

>ที่ threshold 0.6 รีวิวส่วนใหญ่จบได้ด้วย small model จึงทำให้ p50 ต่ำเพียง 6 ms แต่รีวิวที่ small model ไม่มั่นใจยังต้องเรียก large model ต่อ ทำให้กลุ่ม request ที่ช้าที่สุดยังใช้เวลานาน ค่า p95 จึงอยู่ที่ 553 ms และลดลงจาก baseline เพียงเล็กน้อย

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

>เช่น รีวิวประชดว่า “Amazing service my food arrived cold and an hour late.” small model อาจเห็นคำว่า “Amazing” แล้วทำนายว่าเป็นรีวิวเชิงบวกด้วย confidence สูง จึงไม่ส่งต่อให้ large model ตรวจสอบ ทำให้ติด label ผิด ผู้ใช้อาจเข้าใจคุณภาพร้านคลาดเคลื่อน และร้านอาจพลาดข้อร้องเรียนที่ควรแก้ไข

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 4 | 543 | 91.5% | 31% | 36% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

>รีวิวที่มีข้อความซ้ำกับที่เคยประมวลผลและยังมีผลอยู่ใน cache จะได้ประโยชน์ เพราะคืนผลได้ทันทีโดยไม่ต้องเรียกโมเดล ส่วนรีวิวที่มีข้อความไม่ซ้ำกับที่เคยเก็บไว้เลยจะไม่ได้ประโยชน์ เพราะต้องประมวลผลใหม่เสมอ

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

>ต้องเลิกใช้ผลใน cache ที่มาจากโมเดลเก่า เพื่อให้รีวิวได้รับผลจากโมเดลใหม่ โดยใส่ model version เป็นส่วนหนึ่งของ cache key เช่น model_version + review_text เมื่อเปลี่ยนโมเดล ระบบจะเก็บผลชุดใหม่แยกจากเดิม และลบข้อมูลเก่าหรือปล่อยให้หมดอายุด้วย TTL

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

>ได้เรียนรู้จากการทดลองว่า การเพิ่ม threshold ไม่ได้ทำให้ accuracy สูงขึ้นเสมอไป เมื่อเพิ่ม threshold จาก 0.8 เป็น 0.9 สัดส่วนการเรียก large model เพิ่มจาก 64% เป็น 77% แต่ accuracy ยังคงอยู่ที่ 93.5% จึงเห็นว่าในชุดข้อมูลนี้ การเรียกโมเดลใหญ่เพิ่มขึ้นอาจเพิ่มต้นทุนโดยไม่ได้เพิ่มความแม่นยำ
