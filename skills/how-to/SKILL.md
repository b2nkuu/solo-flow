---
name: how-to
description: >
  Use when the user asks how to use solo, how to set it up, what its
  workflow looks like, what commands it has, or how to get started — or asks
  the same in Thai ("ใช้ solo ยังไง", "solo ทำงานยังไง",
  "วิธีใช้ solo", "solo มี command อะไรบ้าง", "เริ่มใช้ solo",
  "setup solo"). Surfaces the install path, daily workflow, and command
  cheat-sheet.
---

# Solo How-To

คู่มือใช้งาน plugin `solo` แบบสั้น สแกนได้เร็ว. ถ้า user แค่จะ *ทำงาน* (capture ไอเดีย, start งาน, ship) ให้ใช้ skill `solo-assistant` แทน — skill นี้คือคู่มือ ไม่ใช่ coach.

## คืออะไร

solo เปลี่ยน Claude Code เป็น task manager สำหรับ solopreneur โดยใช้ GitHub Issues เป็น backend. Capture ไอเดีย → ทำงาน → ship → review รายสัปดาห์ — ทำหมดจาก terminal ไม่ต้องเปิด browser. State อยู่ใน Issues + labels, auth ใช้ `gh`.

## Prerequisites

- [`gh` CLI](https://cli.github.com) ติดตั้งและ login (`gh auth status` ต้องผ่าน)
- GitHub repository สำหรับ project
- โหลด `solo` plugin ใน Claude Code แล้ว — ดู `README.md` ของ plugin สำหรับวิธี install

## Workflow รายวัน

```
        ┌─────────────────────────────────────────────┐
        │                                             │
        ▼                                             │
 /solo:capture ──▶ /solo:plan ──▶ /solo:start ──▶ /solo:note ──▶ /solo:test ──▶ /solo:done
   (Inbox)         (Planned)      (In Progress)   (จด note)      (Verify)       (Closed + PR)
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

**Batch ลุยทุก planned ผ่าน Workflow** — ใช้เมื่อ `/solo:plan` วาง backlog เสร็จและพร้อม implement:
```
/solo:start workflow
```
หยิบทุก issue `status:planned` ที่ตรง `milestone.current` (หรือทุกตัวถ้าไม่ set), spawn 1 pipeline ต่อ issue parallel (cap `workflow.max_parallel` default 4). แต่ละ pipeline: claim (flip status + branch + worktree) → plan → implement (ใน worktree แยก, ไม่ชนกัน) → verify Test Plan + loop. Refuse ทันทีถ้า batch ว่าง / มี `size:xl` / มีตัว AC หรือ Test Plan ว่าง. ดู progress ที่ `/workflows`.

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

**ตรวจตาม test plan** — รัน/verify ทีละ item ก่อน close:
```
/solo:test 42
```
เดินผ่าน `## Test Plan` ทีละข้อ — เสนอคำสั่งให้รัน หรือ verify เอง — แล้วติ๊กข้อที่ผ่านกลับเข้า body. ไม่เปลี่ยน status.

**เสร็จงาน** — จด outcome, close, ship:
```
/solo:done 42
```
ถ้า local branch ตรงกับที่บันทึกไว้ จะถามว่าเปิด PR ไป trunk เลยมั้ย. ตอน close ก็เสนอ `Tick all? [Y/edit/n]` ให้กับ AC + test plan ที่เหลือด้วย.

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

**Cut a release** — tag จาก trunk, generate notes, close milestone:
```
/solo:release --dry-run     # preview ก่อน
/solo:release               # tag + push + GitHub Release + open next milestone
```

**จัดการ milestone**:
```
/solo:plan milestone create v0.4
/solo:plan milestone list
/solo:plan milestone current v0.4
/solo:plan milestone close v0.3
```

## Command cheat-sheet

| Command | ทำอะไร | Args |
|---|---|---|
| `/solo:capture` | Capture เข้า Inbox (auto-attach `milestone.current`) | `"text"` |
| `/solo:today` | Focus วันนี้ + milestone progress | — |
| `/solo:start` | Flip in-progress + branch (single), หรือ `workflow` (batch ทุก planned) | `<issue#> \| workflow` |
| `/solo:test` | Walk test plan, run or verify, tick passed items | `<issue#>` |
| `/solo:done` | Outcome + close + PR (optional) | `<issue#>` |
| `/solo:note` | Append note | `<issue#> "text"` |
| `/solo:block` | Mark blocked | `<issue#> "reason"` |
| `/solo:unblock` | Resume | `<issue#> ["resolution"]` |
| `/solo:plan` | Triage Inbox | — |
| `/solo:plan milestone` | จัดการ milestone | `<action> [name]` |
| `/solo:release` | Tag from trunk + notes + close milestone | `[--dry-run]` |
| `/solo:week` | 7 วันที่ผ่านมา | — |
| `/solo:status` | Snapshot | — |
| `/solo:init` | Setup (idempotent) | — |

## Trunk-based principles (ฝังไว้ใน plugin)

solo ใช้ [trunk-based development](https://trunkbaseddevelopment.com/):

- **Branch จาก trunk เสมอ** — `/solo:start` pull trunk ล่าสุดก่อน branch
- **Branch อายุสั้น (≤ 2 วัน)** — `/solo:today` และ `/solo:status` เตือนเมื่องาน in-progress ค้างนาน
- **One issue → one branch → one PR** — ห้ามรวม
- **Feature flags ดีกว่า branch ยาว** — งานหลายสัปดาห์ใช้ flag gate แล้ว merge เข้า trunk ต่อเนื่อง
- **Release = tag จาก trunk** — `/solo:release` ตัด tag ตรงจาก trunk. ไม่มี release branch. Hotfix = commit ใหม่บน trunk + patch tag
- **Milestone group ให้ release** — ทุก issue ผูก `milestone.current` (เปิด strict ผ่าน `milestone.required: true`)

ปรับใน `.solo/config.yml` ที่ `trunk.max_branch_age_days` (default `2`; `0` = ปิด), `release.initial_version`, `milestone.required`.

## ใช้คู่กับ spirit-mindset

ถ้าติดตั้ง [spirit-mindset](https://github.com/b2nkuu/spirit-mindset) ด้วย จะได้ workflow เต็มขึ้น — solo จัดการ task, spirit-mindset เพิ่มชั้น philosophy ในแต่ละ step:

| สถานการณ์ | solo | spirit-mindset |
|---|---|---|
| ไอเดียใหม่ scope ใหญ่ | `/solo:capture` | `/design` (Ikigai — เริ่มจาก purpose) |
| Triage Inbox | `/solo:plan` | — |
| Start งาน `type:bug` | `/solo:start <n>` | `/debug` (Gaman — root cause) |
| Block เพราะ bug | `/solo:block <n>` | `/debug` |
| ระหว่างทำ code เริ่มเลอะ | `/solo:note <n>` | `/refactor` (Kaizen) |
| Done + เปิด PR | `/solo:done <n>` | `/inspect` (Shokunin — review งานฝีมือ) |
| ตัด release | `/solo:release` | `/inspect` (review trunk ก่อน tag) |
| Weekly review | `/solo:week` | reflect ด้วย Kaizen mindset |

ทั้งคู่ใช้คนละ namespace ไม่ชนกัน. ใช้แค่ solo อย่างเดียวก็ทำงานได้เต็ม.

## ไปต่อ

- Reference เต็มและ config: `README.md` ของ plugin
- Natural-language mode: skill `solo-assistant` activate อัตโนมัติเวลา user คุยเรื่อง task (เช่น "อยากเพิ่ม dark mode" → จะเสนอ capture)
- spirit-mindset: https://github.com/b2nkuu/spirit-mindset
