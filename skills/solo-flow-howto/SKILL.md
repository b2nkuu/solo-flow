---
name: solo-flow:how-to
description: >
  Use when the user asks how to use solo-flow, how to set it up, what its
  workflow looks like, what commands it has, or how to get started — or asks
  the same in Thai ("ใช้ solo-flow ยังไง", "solo-flow ทำงานยังไง",
  "วิธีใช้ solo-flow", "solo-flow มี command อะไรบ้าง", "เริ่มใช้ solo-flow",
  "setup solo-flow"). Surfaces the install path, daily workflow, and command
  cheat-sheet.
---

# Solo-Flow How-To

คู่มือใช้งาน plugin `solo-flow` แบบสั้น สแกนได้เร็ว. ถ้า user แค่จะ *ทำงาน* (capture ไอเดีย, start งาน, ship) ให้ใช้ skill `solo-flow-assistant` แทน — skill นี้คือคู่มือ ไม่ใช่ coach.

## คืออะไร

solo-flow เปลี่ยน Claude Code เป็น task manager สำหรับ solopreneur โดยใช้ GitHub Issues เป็น backend. Capture ไอเดีย → ทำงาน → ship → review รายสัปดาห์ — ทำหมดจาก terminal ไม่ต้องเปิด browser. State อยู่ใน Issues + labels, auth ใช้ `gh`.

## Prerequisites

- [`gh` CLI](https://cli.github.com) ติดตั้งและ login (`gh auth status` ต้องผ่าน)
- GitHub repository สำหรับ project
- โหลด `solo-flow` plugin ใน Claude Code แล้ว — ดู `README.md` ของ plugin สำหรับวิธี install

## Workflow รายวัน

```
        ┌─────────────────────────────────┐
        │                                 │
        ▼                                 │
 /solo:capture ──▶ /solo:plan ──▶ /solo:start ──▶ /solo:note ──▶ /solo:done
   (Inbox)         (Planned)      (In Progress)   (จด note)      (Closed + PR)
        ▲                              │
        └────────── /solo:today ───────┘
```

`/solo:today` คือจุดเริ่มของทุก session — แสดงงานที่กำลังทำ, งานถัดไป, และงานที่ blocked.

## ครั้งแรก setup

```
cd path/to/your-repo
/solo:init                              # สร้าง labels + .solo/config.yml
/solo:capture "first thing I want to do"
```

`/solo:init` idempotent — รันซ้ำได้ปลอดภัย. มัน detect trunk branch (`main`/`master`) และเขียนลง config ให้.

## Scenarios ที่เจอบ่อย

**มีไอเดียระหว่างทำงาน** — อย่าหยุด flow แค่ capture:
```
/solo:capture "add rate limiting to the API"
```

**เริ่มวัน** — ดู list, เลือกงาน, เริ่มเลย:
```
/solo:today
/solo:start 42
```
`/solo:start` อัพ `trunk`, branch จากตรงนั้น, แล้ว flip status เป็น in-progress.

**ทำงานอยู่** — จด note ระหว่างทาง โดยเฉพาะ decision:
```
/solo:note 42 "tried bcrypt, settled on argon2"
/solo:note 42 "[decision] use argon2id with 64MB memory cost"
```
Note ธรรมดาไป comment thread. Note ที่ขึ้น `[decision]` mirror ไปที่ body ด้วย เพื่อ parse ได้ตอนหลัง.

**ติด** — จดเหตุผล แล้วไปทำอย่างอื่น:
```
/solo:block 42 "waiting on design spec from client"
```
พอ unblock:
```
/solo:unblock 42 "spec arrived, going with option B"
```

**เสร็จงาน** — จด outcome, close, ship:
```
/solo:done 42
```
ถ้า local branch ตรงกับที่บันทึกไว้ จะถามว่าเปิด PR ไป trunk เลยมั้ย.

**Inbox เต็ม** — triage priority + size รวดเดียว:
```
/solo:plan
```

**Review รายสัปดาห์** — ship อะไรบ้าง, ค้างอะไร, decision อะไร:
```
/solo:week
```

**Overview project** — counts, stale in-progress warnings:
```
/solo:status
```

## Command cheat-sheet

| Command | ทำอะไร | Args |
|---|---|---|
| `/solo:capture` | Capture เข้า Inbox | `"text"` |
| `/solo:today` | Focus วันนี้ | — |
| `/solo:start` | Flip in-progress + branch | `<issue#>` |
| `/solo:done` | Outcome + close + PR (optional) | `<issue#>` |
| `/solo:note` | Append note | `<issue#> "text"` |
| `/solo:block` | Mark blocked | `<issue#> "reason"` |
| `/solo:unblock` | Resume | `<issue#> ["resolution"]` |
| `/solo:plan` | Triage Inbox | — |
| `/solo:week` | 7 วันที่ผ่านมา | — |
| `/solo:status` | Snapshot | — |
| `/solo:init` | Setup (idempotent) | — |

## Trunk-based principles (ฝังไว้ใน plugin)

solo-flow ใช้ [trunk-based development](https://trunkbaseddevelopment.com/):

- **Branch จาก trunk เสมอ** — `/solo:start` pull trunk ล่าสุดก่อน branch
- **Branch อายุสั้น (≤ 2 วัน)** — `/solo:today` และ `/solo:status` เตือนเมื่องาน in-progress ค้างนาน
- **One issue → one branch → one PR** — ห้ามรวม
- **Feature flags ดีกว่า branch ยาว** — งานหลายสัปดาห์ใช้ flag gate แล้ว merge เข้า trunk ต่อเนื่อง

ปรับใน `.solo/config.yml` ที่ `trunk.max_branch_age_days` (default `2`; `0` = ปิด).

## ใช้คู่กับ spirit-mindset

ถ้าติดตั้ง [spirit-mindset](https://github.com/b2nkuu/spirit-mindset) ด้วย จะได้ workflow เต็มขึ้น — solo-flow จัดการ task, spirit-mindset เพิ่มชั้น philosophy ในแต่ละ step:

| สถานการณ์ | solo-flow | spirit-mindset |
|---|---|---|
| ไอเดียใหม่ scope ใหญ่ | `/solo:capture` | `/design` (Ikigai — เริ่มจาก purpose) |
| Triage Inbox | `/solo:plan` | — |
| Start งาน `type:bug` | `/solo:start <n>` | `/debug` (Gaman — root cause) |
| Block เพราะ bug | `/solo:block <n>` | `/debug` |
| ระหว่างทำ code เริ่มเลอะ | `/solo:note <n>` | `/refactor` (Kaizen) |
| Done + เปิด PR | `/solo:done <n>` | `/inspect` (Shokunin — review งานฝีมือ) |
| Weekly review | `/solo:week` | reflect ด้วย Kaizen mindset |

ทั้งคู่ใช้คนละ namespace ไม่ชนกัน. ใช้แค่ solo-flow อย่างเดียวก็ทำงานได้เต็ม.

## ไปต่อ

- Reference เต็มและ config: `README.md` ของ plugin
- Natural-language mode: skill `solo-flow-assistant` activate อัตโนมัติเวลา user คุยเรื่อง task (เช่น "อยากเพิ่ม dark mode" → จะเสนอ capture)
- spirit-mindset: https://github.com/b2nkuu/spirit-mindset
