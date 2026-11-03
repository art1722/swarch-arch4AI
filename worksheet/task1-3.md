# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** Chawakorn Samosorn / ชวกรณ์ สโมสร

**Student ID / รหัสนิสิต:** 6631308221

This is **individual work**: each student completes and submits their own copy, in their own fork.

_งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง_

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls |
| -------- | ------ | ------ | -------- | ----------- |
| t0-large | 498    | 597    | 93.5%    | 100%        |
| t0-small | 28     | 61     | 72.5%    | 0%          |

**Q0.** What does the large model buy, and what does it cost?

_large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร_

> The large model increases accuracy from 72.5% to 93.5%, meeting the accuracy requirement. The tradeoffs are higher latency, p95 rises from 61 ms to 597 ms, and large-model calls increase from 0% to 100%, causing higher inference costs.

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run       | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
| --------- | ------------ | ------ | ------ | ------ | ------- | -------- |
| t1-before | 35.0         | 245    | 477    | 26     | 0       | 93.7%    |
| t1-after  | 9.5          | 371    | 1034   | 0      | 27      | 93.6%    |

**Q1.1** In `t1-before`, p50 can look _better_ than the baseline. Why is p50 misleading here? Which column tells the real story?

_ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง_

> p50 is misleading because it is calculated only from successful responses, excluding requests that failed or timed out. Although p50 dropped from 498 ms to 245 ms, the errors column shows that 26 of 200 requests failed, and total run time increased from 10.1 s to 35.0 s.

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

_การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway_

> Waiting requests retain connections, sockets, and memory in both the review service and gateway. As these requests accumulate, they consume shared resources and reduce the capacity available for other reviews, allowing an inference-service slowdown to affect the wider system.

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

_รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น_

> Pending reviews should be stored durably and processed later by a background worker, with retries when inference becomes available. This uses the asynchronous / batch topology (T4), allowing review submission to succeed without waiting for classification.

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

_เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที_

> A 0.1-second timeout is shorter than the large model’s normal processing time of about 0.2 seconds, so even healthy calls would frequently become pending. A 10-second timeout retains resources too long, violates the 1-second responsiveness requirement, and risks the client timing out before receiving the fallback response.

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run    | threshold | p50 ms | p95 ms | accuracy | large calls |
| ------ | --------- | ------ | ------ | -------- | ----------- |
| t2-0.6 | 0.6       | 63     | 513    | 91.5%    | 38%         |
| t2-0.7 | 0.7       | 339    | 560    | 93.0%    | 53%         |
| t2-0.8 | 0.8       | 422    | 609    | 93.5%    | 64%         |
| t2-0.9 | 0.9       | 457    | 611    | 93.5%    | 77%         |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

_จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป_

> I would deploy threshold 0.6 because it achieves 91.5% accuracy, exceeding the 90% requirement, while using the large model for only 38% of requests, the lowest among the tested thresholds. Its p95 is 513 ms, meeting the 1-second target. The quality sacrificed is accuracy: it is 2 percentage points lower than the large-only baseline.

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

_ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก_

> At threshold 0.6, most requests are answered by the fast small model, reducing p50 from 498 ms to 63 ms. However, the slowest requests still require the large model and may wait for its limited capacity, so p95 decreases much less, from 597 ms to 513 ms.

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

_cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้_

> The small model might confidently classify “The food was not good” as positive because it recognizes “good” but mishandles the negation. The cascade would accept this incorrect prediction without consulting the large model, potentially causing the restaurant owner to overlook an unhappy customer.

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls | cache hits |
| -------- | ------ | ------ | -------- | ----------- | ---------- |
| t3-cache | 20     | 491    | 91.5%    | 31%         | 36%        |

**Q3.1** Which reviews benefit from the cache, and which never will?

_รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์_

> Reviews with the same normalized text benefit when their prediction is still cached, even if capitalization or whitespace differs. Reviews whose normalized text never repeats cannot benefit; repeated reviews also miss if their entry has been evicted.

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

_สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร_

> Predictions from the old model must be invalidated so users do not keep receiving outdated results. I would include the model versions, strategy, and threshold in the cache key, so changing the prediction configuration causes fresh inference.

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

_ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture_

> I learned that lower reported latency does not necessarily mean a healthier system when failed requests are excluded from the calculation. In t1-before, p50 decreased from 498 ms to 245 ms, yet 26 of 200 requests failed and total run time increased from 10.1 s to 35.0 s. Therefore, latency must be interpreted alongside errors and total run time.
