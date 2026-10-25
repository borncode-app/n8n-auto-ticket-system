# 🎫 Auto Ticket System — n8n IT Helpdesk Automation

ระบบเปิด/มอบหมาย/ปิด Ticket แจ้งปัญหา IT อัตโนมัติจากแชท LINE กลุ่ม โดยใช้ **n8n** + **AI (LLM ผ่าน OpenRouter)** จัดหมวดหมู่ปัญหา และเก็บข้อมูลลง **Microsoft SQL Server**

> Repo นี้ใช้นำเสนอ workflow เท่านั้น — ข้อมูลบริษัท/บุคคล ถูกแทนด้วย placeholder ทั้งหมด (เช่น `YourDatabase`, `YOUR_LINE_IT_GROUP_ID_1`, `example.com`)

## ✨ ความสามารถหลัก

- **Auto Ticket** — รับ event จาก LINE webhook แล้วแยกประเภทข้อความ (ข้อความ / รูป / สมาชิกเข้า-ออก / unsend)
- **AI Classification** — LLM วิเคราะห์ข้อความแจ้งปัญหา ระบุหมวดหมู่ หมวดย่อย และสาขา ด้วย structured output
- **Auto Assign** — เมื่อทีม IT reply ข้อความ ระบบจะมอบหมาย ticket ให้คนนั้นอัตโนมัติ
- **Auto Close** — ตรวจจับข้อความแจ้งว่าแก้ไขเสร็จ แล้วปิด ticket พร้อมส่งอีเมลสรุป
- **Schedule Summary** — สรุป ticket ค้างทุกเช้าทางอีเมล
- **Manual Ticket** — เปิด ticket ผ่านฟอร์ม n8n
- **Audit Log + Error Alert** — log ทุก action ลง SQL และแจ้ง error ไป Telegram

## 🧰 Stack

n8n · LINE Messaging API · OpenRouter (LLM) · Microsoft SQL Server · SMTP / Microsoft Graph · Telegram

## 🖼️ Screenshots

| Main workflow (LINE webhook) | AI classification |
|---|---|
| ![Main workflow](screenshots/main-workflow.png) | ![AI classification](screenshots/ai-classification.png) |
| **Auto Assign** | **Auto Close Ticket** |
| ![Auto Assign](screenshots/auto-assign.png) | ![Auto Close Ticket](screenshots/auto-close-ticket.png) |
| **Schedule Ticket Unclose** | **LogSQLServer** |
| ![Schedule Ticket Unclose](screenshots/schedule-ticket-unclose.png) | ![LogSQLServer](screenshots/log-sql-server.png) |

Architecture: [`screenshots/architecture.png`](screenshots/architecture.png) · Mermaid source: [`screenshots/architecture.mermaid`](screenshots/architecture.mermaid)

## 🧩 Workflows

ไฟล์ export ของ n8n อยู่ใน [`workflows/`](workflows/) — import ได้ผ่าน n8n (**Workflows → Import from File**) แล้วตั้งค่า credentials ของตัวเอง

### ภาพรวมการเชื่อมต่อระหว่าง Workflow

```mermaid
flowchart LR
  w0["Auto Assign 1.2.1<br/><i>Called by workflow</i>"]
  class w0 k_sub
  w1["Auto Close Ticket 1.3<br/><i>Called by workflow</i>"]
  class w1 k_sub
  w2["Auto Ticket 1.7.2<br/><i>Webhook (POST)</i>"]
  class w2 k_entry
  w3["Auto Ticket CoreAI 1.4<br/><i>Called by workflow</i>"]
  class w3 k_sub
  w4["Error Notify Telegram<br/><i>On error</i>"]
  class w4 k_onerror
  w6["LogSQLServer 1.0.1<br/><i>Called by workflow</i>"]
  class w6 k_sub
  w7["Manual Ticket 1.1.0<br/><i>Form</i>"]
  class w7 k_entry
  w8["Schedule Auto Open Ticket Unclose 1.1<br/><i>Schedule cron 0 8,18 * * *</i>"]
  class w8 k_entry
  w9["Schedule Ticket Unclose 1.4.1<br/><i>Schedule ทุกวัน 08:00</i>"]
  class w9 k_entry
  w10["Send Email (Graph)<br/><i>Called by workflow</i>"]
  class w10 k_sub
  w0 -->|"calls"| w6
  w0 -.->|"on error"| w4
  w1 -->|"calls"| w10
  w1 -->|"calls"| w3
  w1 -->|"calls"| w6
  w1 -.->|"on error"| w4
  w10 -.->|"on error"| w4
  w2 -->|"calls"| w0
  w2 -->|"calls"| w1
  w2 -->|"calls"| w3
  w2 -->|"calls"| w6
  w3 -->|"calls"| w6
  w6 -.->|"on error"| w4
  w7 -->|"calls"| w10
  w7 -->|"calls"| w3
  w7 -->|"calls"| w6
  w8 -->|"calls"| w10
  w8 -->|"calls"| w6
  w8 -.->|"on error"| w4
  w9 -->|"calls"| w10
  w9 -.->|"on error"| w4
  classDef k_entry fill:#d1fae5,stroke:#059669,color:#064e3b
  classDef k_sub fill:#ccfbf1,stroke:#0d9488,color:#134e4a
  classDef k_onerror fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
```

🟩 จุดเริ่ม (Webhook / Form / Schedule) · 🟦 Sub-workflow ที่ถูกเรียกใช้ · 🟥 ตัวจัดการ Error → Telegram

| Workflow | Trigger | Nodes | ระบบที่เชื่อมต่อ |
|---|---|---:|---|
| [Auto Assign 1.2.1](#auto-assign-121) | Called by workflow | 10 | MS SQL |
| [Auto Close Ticket 1.3](#auto-close-ticket-13) | Called by workflow | 11 | MS SQL |
| [Auto Ticket 1.7.2](#auto-ticket-172) | Webhook (POST) | 25 | HTTP Request, MS SQL |
| [Auto Ticket CoreAI 1.4](#auto-ticket-coreai-14) | Called by workflow | 25 | Output Parser, MS SQL, LLM Chain, OpenRouter |
| [Error Notify Telegram](#error-notify-telegram) | On error | 4 | Telegram |
| [LogSQLServer 1.0.1](#logsqlserver-101) | Called by workflow | 4 | MS SQL |
| [Manual Ticket 1.1.0](#manual-ticket-110) | Form | 19 | Form, MS SQL |
| [Schedule Auto Open Ticket Unclose 1.1](#schedule-auto-open-ticket-unclose-11) | Schedule cron `0 8,18 * * *` | 13 | MS SQL |
| [Schedule Ticket Unclose 1.4.1](#schedule-ticket-unclose-141) | Schedule ทุกวัน 08:00 | 16 | MS SQL |
| [Send Email (Graph)](#send-email-graph) | Called by workflow | 4 | HTTP Request |

### รายละเอียดแต่ละ Workflow

#### Auto Assign 1.2.1

**Trigger:** Called by workflow · **ไฟล์:** [`Auto Assign 1.2.1.json`](workflows/Auto%20Assign%201.2.1.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart LR
  n0{"If Reply"}
  class n0 k_branch
  n1{"If notEmpty"}
  class n1 k_branch
  n2["Send email To DavMail"]
  class n2 k_disabled
  n3(["Start<br/><i>Called by workflow</i>"])
  class n3 k_trigger
  n4["Wait"]
  class n4 k_flow
  n5[("Lookup Ticket<br/><i>MS SQL</i>")]
  class n5 k_data
  n6[["Call LogSQLServer v1.0.0<br/><i>→ LogSQLServer 1.0.1</i>"]]
  class n6 k_call
  n7[("Update Ticket Status<br/><i>MS SQL</i>")]
  class n7 k_data
  n8[("Match IT Team (OLD)<br/><i>Data Table</i>")]
  class n8 k_disabled
  n9[("Match IT Team<br/><i>MS SQL</i>")]
  class n9 k_data
  n0 -->|"true"| n4
  n1 -->|"true"| n9
  n3 --> n0
  n4 --> n5
  n5 --> n1
  n7 --> n6
  n9 --> n7
  classDef k_branch fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef k_call fill:#ccfbf1,stroke:#0d9488,color:#134e4a
  classDef k_data fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef k_disabled fill:#f9fafb,stroke:#9ca3af,color:#9ca3af,stroke-dasharray:4 3
  classDef k_flow fill:#f3f4f6,stroke:#6b7280,color:#111827
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### Auto Close Ticket 1.3

**Trigger:** Called by workflow · **ไฟล์:** [`Auto Close Ticket 1.3.json`](workflows/Auto%20Close%20Ticket%201.3.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart LR
  n0{"If"}
  class n0 k_branch
  n1[("Microsoft SQL<br/><i>MS SQL</i>")]
  class n1 k_data
  n2{"If notEmpty"}
  class n2 k_branch
  n3["Send email To DavMail"]
  class n3 k_disabled
  n4[("Update Ticket Status<br/><i>MS SQL</i>")]
  class n4 k_data
  n5[["Call LogSQLServer<br/><i>→ LogSQLServer 1.0.1</i>"]]
  class n5 k_call
  n6(["Start<br/><i>Called by workflow</i>"])
  class n6 k_trigger
  n7[("Match IT Team<br/><i>MS SQL</i>")]
  class n7 k_data
  n8[["Call Auto Ticket CoreAI<br/><i>→ Auto Ticket CoreAI 1.4</i>"]]
  class n8 k_call
  n9[("Find Ticket<br/><i>MS SQL</i>")]
  class n9 k_data
  n10[["Send email via Graph<br/><i>→ Send Email (Graph)</i>"]]
  class n10 k_call
  n0 -->|"true"| n1
  n1 --> n2
  n2 -->|"true"| n7
  n4 --> n5
  n6 --> n0
  n7 --> n8
  n7 --> n9
  n8 --> n9
  n9 --> n10
  n10 --> n4
  classDef k_branch fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef k_call fill:#ccfbf1,stroke:#0d9488,color:#134e4a
  classDef k_data fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef k_disabled fill:#f9fafb,stroke:#9ca3af,color:#9ca3af,stroke-dasharray:4 3
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### Auto Ticket 1.7.2

**Trigger:** Webhook (POST) · **ไฟล์:** [`Auto Ticket 1.7.2.json`](workflows/Auto%20Ticket%201.7.2.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart TD
  n0["SET - Normalize Clean Text"]
  class n0 k_flow
  n1["SET - Extract Sender IDs"]
  class n1 k_flow
  n2["HTTP - Get Sender Profile<br/><i>HTTP Request</i>"]
  class n2 k_http
  n3["Get Group/Room Summary<br/><i>HTTP Request</i>"]
  class n3 k_http
  n4{"If image"}
  class n4 k_branch
  n5[("FTP")]
  class n5 k_disabled
  n6["Get Image<br/><i>HTTP Request</i>"]
  class n6 k_http
  n7["filename<br/><i>Set</i>"]
  class n7 k_flow
  n8["Upload Image<br/><i>Set</i>"]
  class n8 k_flow
  n9{"Switch"}
  class n9 k_branch
  n10["Set (sticker)"]
  class n10 k_flow
  n11["Set (memberJoined)"]
  class n11 k_flow
  n12["Set (memberLeft)"]
  class n12 k_flow
  n13["Limit"]
  class n13 k_flow
  n14{"If Ticket"}
  class n14 k_branch
  n15(["Webhook Line"])
  class n15 k_trigger
  n16[["Call Auto Ticket CoreAI<br/><i>→ Auto Ticket CoreAI 1.4</i>"]]
  class n16 k_call
  n17[("Get IT Team<br/><i>MS SQL</i>")]
  class n17 k_data
  n18{"If not IT Team"}
  class n18 k_branch
  n19[("Update Unsend Status (SQL)<br/><i>MS SQL</i>")]
  class n19 k_data
  n20[["Call LogSQLServer v1.0.0<br/><i>→ LogSQLServer 1.0.1</i>"]]
  class n20 k_call
  n21["Wait"]
  class n21 k_flow
  n22[["Call Auto Close Ticket<br/><i>→ Auto Close Ticket 1.3</i>"]]
  class n22 k_call
  n23[["Call Auto Assign<br/><i>→ Auto Assign 1.2.1</i>"]]
  class n23 k_call
  n24{"If IT Group"}
  class n24 k_branch
  n0 --> n24
  n1 --> n2
  n2 --> n3
  n3 --> n9
  n4 -->|"true"| n6
  n4 -->|"false"| n0
  n5 --> n8
  n6 --> n7
  n7 --> n5
  n9 -->|"1"| n11
  n9 -->|"2"| n10
  n9 -->|"3"| n21
  n9 -->|"4"| n12
  n9 -->|"fallback"| n4
  n13 --> n1
  n14 -->|"true"| n16
  n15 --> n13
  n17 --> n18
  n18 -->|"true"| n23
  n18 -->|"false"| n14
  n19 --> n20
  n21 --> n19
  n23 --> n22
  n24 -->|"true"| n23
  n24 -->|"true"| n14
  n24 -->|"false"| n17
  classDef k_branch fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef k_call fill:#ccfbf1,stroke:#0d9488,color:#134e4a
  classDef k_data fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef k_disabled fill:#f9fafb,stroke:#9ca3af,color:#9ca3af,stroke-dasharray:4 3
  classDef k_flow fill:#f3f4f6,stroke:#6b7280,color:#111827
  classDef k_http fill:#ffedd5,stroke:#ea580c,color:#7c2d12
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### Auto Ticket CoreAI 1.4

**Trigger:** Called by workflow · **ไฟล์:** [`Auto Ticket CoreAI 1.4.json`](workflows/Auto%20Ticket%20CoreAI%201.4.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart TD
  n0{{"Structured Output Sub Category<br/><i>Output Parser</i>"}}
  class n0 k_ai
  n1[("Get Sub Category<br/><i>MS SQL</i>")]
  class n1 k_data
  n2{{"Find Sub Category<br/><i>LLM Chain</i>"}}
  class n2 k_ai
  n3[("Get Branch Franchise<br/><i>MS SQL</i>")]
  class n3 k_data
  n4["Set Branch Franchise"]
  class n4 k_flow
  n5{{"Structured Output Core Agent<br/><i>Output Parser</i>"}}
  class n5 k_ai
  n6["Set Sub/Body"]
  class n6 k_flow
  n7["Merge"]
  class n7 k_flow
  n8["Set Command Ticket"]
  class n8 k_flow
  n9["Aggregate"]
  class n9 k_flow
  n10{{"FInd Branch<br/><i>LLM Chain</i>"}}
  class n10 k_ai
  n11["Merge Company"]
  class n11 k_flow
  n12[("Get Category<br/><i>MS SQL</i>")]
  class n12 k_data
  n13["Set Branch Main"]
  class n13 k_flow
  n14[("Get Branch Main<br/><i>MS SQL</i>")]
  class n14 k_data
  n15{{"Core Agent<br/><i>LLM Chain</i>"}}
  class n15 k_ai
  n16{{"Structured Output Branch<br/><i>Output Parser</i>"}}
  class n16 k_ai
  n17{{"OpenRouter Chat Model"}}
  class n17 k_ai
  n18(["Start<br/><i>Called by workflow</i>"])
  class n18 k_trigger
  n19[["Call LogSQLServer v1.0.0<br/><i>→ LogSQLServer 1.0.1</i>"]]
  class n19 k_call
  n20["Set (ticket)"]
  class n20 k_flow
  n21[("Microsoft SQL (Insert)<br/><i>MS SQL</i>")]
  class n21 k_data
  n22{"Check Action<br/><i>IF</i>"}
  class n22 k_branch
  n23[("Microsoft SQL (Update)<br/><i>MS SQL</i>")]
  class n23 k_data
  n24[("Microsoft SQL(OLD)<br/><i>MS SQL</i>")]
  class n24 k_disabled
  n0 -.->|"parser"| n2
  n1 --> n2
  n2 --> n7
  n3 --> n4
  n4 --> n11
  n5 -.->|"parser"| n15
  n6 --> n20
  n7 --> n8
  n8 --> n6
  n9 --> n10
  n10 --> n7
  n10 --> n1
  n11 --> n9
  n12 --> n15
  n13 --> n11
  n14 --> n13
  n15 --> n14
  n15 --> n3
  n16 -.->|"parser"| n10
  n17 -.->|"model"| n15
  n17 -.->|"model"| n10
  n17 -.->|"model"| n2
  n18 --> n12
  n20 --> n22
  n22 -->|"true"| n23
  n22 -->|"false"| n21
  n23 --> n19
  n21 --> n19
  classDef k_ai fill:#ede9fe,stroke:#7c3aed,color:#4c1d95
  classDef k_branch fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef k_call fill:#ccfbf1,stroke:#0d9488,color:#134e4a
  classDef k_data fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef k_disabled fill:#f9fafb,stroke:#9ca3af,color:#9ca3af,stroke-dasharray:4 3
  classDef k_flow fill:#f3f4f6,stroke:#6b7280,color:#111827
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### Error Notify Telegram

**Trigger:** On error · **ไฟล์:** [`Error Notify Telegram.json`](workflows/Error%20Notify%20Telegram.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart LR
  n0(["Error Trigger<br/><i>On error</i>"])
  class n0 k_trigger
  n1[["Format Error Message<br/><i>Code</i>"]]
  class n1 k_code
  n2["Telegram"]
  class n2 k_notify
  n3[["Log Notification<br/><i>Code</i>"]]
  class n3 k_code
  n0 --> n1
  n1 --> n2
  n2 --> n3
  classDef k_code fill:#e5e7eb,stroke:#374151,color:#111827
  classDef k_notify fill:#fce7f3,stroke:#db2777,color:#831843
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### LogSQLServer 1.0.1

**Trigger:** Called by workflow · **ไฟล์:** [`LogSQLServer 1.0.1.json`](workflows/LogSQLServer%201.0.1.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart LR
  n0[("Insert Log<br/><i>MS SQL</i>")]
  class n0 k_data
  n1["Set Insert Log"]
  class n1 k_flow
  n2(["Start<br/><i>Called by workflow</i>"])
  class n2 k_trigger
  n3[("SELECT TOP (1)<br/><i>MS SQL</i>")]
  class n3 k_data
  n1 --> n0
  n2 --> n3
  n3 --> n1
  classDef k_data fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef k_flow fill:#f3f4f6,stroke:#6b7280,color:#111827
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### Manual Ticket 1.1.0

**Trigger:** Form · **ไฟล์:** [`Manual Ticket 1.1.0.json`](workflows/Manual%20Ticket%201.1.0.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart TD
  n0(["On form submission"])
  class n0 k_trigger
  n1{"Check Action<br/><i>IF</i>"}
  class n1 k_branch
  n2["Prepare Close Form<br/><i>Set</i>"]
  class n2 k_flow
  n3["Close Ticket Form"]
  class n3 k_notify
  n4[("Find Ticket<br/><i>MS SQL</i>")]
  class n4 k_data
  n5{"If notEmpty"}
  class n5 k_branch
  n6[("Match IT Team<br/><i>MS SQL</i>")]
  class n6 k_data
  n7[["Call Auto Ticket CoreAI<br/><i>→ Auto Ticket CoreAI 1.4</i>"]]
  class n7 k_call
  n8[("Find Ticket After CoreAI<br/><i>MS SQL</i>")]
  class n8 k_data
  n9["Send email To DavMail"]
  class n9 k_disabled
  n10[("Update Ticket Status<br/><i>MS SQL</i>")]
  class n10 k_data
  n11[["Call LogSQLServer<br/><i>→ LogSQLServer 1.0.1</i>"]]
  class n11 k_call
  n12["Close Complete<br/><i>Form</i>"]
  class n12 k_notify
  n13["Prepare Delete Form<br/><i>Set</i>"]
  class n13 k_flow
  n14["Delete Ticket Form"]
  class n14 k_notify
  n15[("Delete Ticket<br/><i>MS SQL</i>")]
  class n15 k_data
  n16[["Call LogSQLServer Delete<br/><i>→ LogSQLServer 1.0.1</i>"]]
  class n16 k_call
  n17["Delete Complete<br/><i>Form</i>"]
  class n17 k_notify
  n18[["Send email via Graph<br/><i>→ Send Email (Graph)</i>"]]
  class n18 k_call
  n0 --> n1
  n1 -->|"true"| n2
  n1 -->|"false"| n13
  n2 --> n3
  n3 --> n4
  n4 --> n5
  n5 -->|"true"| n6
  n6 --> n7
  n7 --> n8
  n8 --> n18
  n10 --> n11
  n11 --> n12
  n13 --> n14
  n14 --> n15
  n15 --> n16
  n16 --> n17
  n18 --> n10
  classDef k_branch fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef k_call fill:#ccfbf1,stroke:#0d9488,color:#134e4a
  classDef k_data fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef k_disabled fill:#f9fafb,stroke:#9ca3af,color:#9ca3af,stroke-dasharray:4 3
  classDef k_flow fill:#f3f4f6,stroke:#6b7280,color:#111827
  classDef k_notify fill:#fce7f3,stroke:#db2777,color:#831843
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### Schedule Auto Open Ticket Unclose 1.1

**Trigger:** Schedule cron `0 8,18 * * *` · **ไฟล์:** [`Schedule Auto Open Ticket Unclose 1.1.json`](workflows/Schedule%20Auto%20Open%20Ticket%20Unclose%201.1.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart LR
  n0["Send email To DavMail"]
  class n0 k_disabled
  n1["Loop Over Items"]
  class n1 k_flow
  n2[("Update Ticket Status<br/><i>MS SQL</i>")]
  class n2 k_data
  n3[["Call LogSQLServer v1.0.0<br/><i>→ LogSQLServer 1.0.1</i>"]]
  class n3 k_call
  n4{"If notEmpty"}
  class n4 k_branch
  n5{"If2"}
  class n5 k_branch
  n6[("Match IT Team<br/><i>MS SQL</i>")]
  class n6 k_data
  n7["No Operation, do nothing<br/><i>No-op</i>"]
  class n7 k_flow
  n8[("Get Pending Tickets<br/><i>MS SQL</i>")]
  class n8 k_data
  n9(["Schedule Trigger"])
  class n9 k_trigger
  n10[["Check Last Day of Month<br/><i>Code</i>"]]
  class n10 k_code
  n11{"If Last Day"}
  class n11 k_branch
  n12[["Send email via Graph<br/><i>→ Send Email (Graph)</i>"]]
  class n12 k_call
  n1 -->|"done"| n6
  n1 -->|"loop"| n1
  n2 --> n3
  n4 -->|"true"| n1
  n5 -->|"true"| n7
  n5 -->|"false"| n12
  n6 --> n5
  n8 --> n4
  n9 --> n10
  n10 --> n11
  n11 -->|"true"| n8
  n12 --> n2
  classDef k_branch fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef k_call fill:#ccfbf1,stroke:#0d9488,color:#134e4a
  classDef k_code fill:#e5e7eb,stroke:#374151,color:#111827
  classDef k_data fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef k_disabled fill:#f9fafb,stroke:#9ca3af,color:#9ca3af,stroke-dasharray:4 3
  classDef k_flow fill:#f3f4f6,stroke:#6b7280,color:#111827
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### Schedule Ticket Unclose 1.4.1

**Trigger:** Schedule ทุกวัน 08:00 · **ไฟล์:** [`Schedule Ticket Unclose 1.4.1.json`](workflows/Schedule%20Ticket%20Unclose%201.4.1.json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart TD
  n0["Send email To DavMail"]
  class n0 k_disabled
  n1["Loop Over Items"]
  class n1 k_disabled
  n2[("Update Ticket Status<br/><i>MS SQL</i>")]
  class n2 k_disabled
  n3[["Call LogSQLServer v1.0.0<br/><i>→ LogSQLServer v1.0.1</i>"]]
  class n3 k_disabled
  n4(["Schedule Trigger"])
  class n4 k_trigger
  n5{"If notEmpty"}
  class n5 k_branch
  n6["Aggregate"]
  class n6 k_flow
  n7[("Get Pending Tickets 2<br/><i>MS SQL</i>")]
  class n7 k_data
  n8[("Get Pending Tickets 1<br/><i>MS SQL</i>")]
  class n8 k_data
  n9{"If"}
  class n9 k_branch
  n10["Aggregate1"]
  class n10 k_flow
  n11["Send email To IT Support Group"]
  class n11 k_disabled
  n12["Send email To Senior Group"]
  class n12 k_disabled
  n13[("Match IT Team Old<br/><i>Data Table</i>")]
  class n13 k_disabled
  n14[["Send email To Senior Group (Graph)<br/><i>→ Send Email (Graph)</i>"]]
  class n14 k_call
  n15[["Send email To IT Support Group (Graph)<br/><i>→ Send Email (Graph)</i>"]]
  class n15 k_call
  n0 --> n2
  n1 -->|"done"| n13
  n1 -->|"loop"| n1
  n2 --> n3
  n4 --> n8
  n5 -->|"true"| n7
  n6 --> n14
  n7 --> n9
  n8 --> n5
  n9 -->|"true"| n6
  n9 -->|"false"| n10
  n10 --> n15
  n13 --> n0
  classDef k_branch fill:#fef3c7,stroke:#d97706,color:#78350f
  classDef k_call fill:#ccfbf1,stroke:#0d9488,color:#134e4a
  classDef k_data fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
  classDef k_disabled fill:#f9fafb,stroke:#9ca3af,color:#9ca3af,stroke-dasharray:4 3
  classDef k_flow fill:#f3f4f6,stroke:#6b7280,color:#111827
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>

#### Send Email (Graph)

**Trigger:** Called by workflow · **ไฟล์:** [`Send Email (Graph).json`](workflows/Send%20Email%20(Graph).json)

<details open>
<summary>ดู Flow</summary>

```mermaid
flowchart LR
  n0(["Start<br/><i>Called by workflow</i>"])
  class n0 k_trigger
  n1[["Build Graph payload<br/><i>Code</i>"]]
  class n1 k_code
  n2["Graph sendMail<br/><i>HTTP Request</i>"]
  class n2 k_http
  n3[["Result<br/><i>Code</i>"]]
  class n3 k_code
  n0 --> n1
  n1 --> n2
  n2 --> n3
  classDef k_code fill:#e5e7eb,stroke:#374151,color:#111827
  classDef k_http fill:#ffedd5,stroke:#ea580c,color:#7c2d12
  classDef k_trigger fill:#d1fae5,stroke:#059669,color:#064e3b
```

</details>
