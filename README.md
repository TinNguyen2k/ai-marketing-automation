# AI Marketing Automation

AI-powered workflow for automating media plan generation using n8n and Claude AI.

## Overview

Hệ thống tự động tạo media plan từ Jira ticket — từ việc chọn channel/objective, tính toán phân bổ ngân sách, đến ghi kết quả vào Google Sheets và notify qua Google Chat.

## Tech Stack

- **n8n** — workflow automation
- **Claude AI** (Haiku + Sonnet) — AI agents
- **Google BigQuery** — data source (benchmark, ticket)
- **Google Sheets / Drive** — output template
- **Google Chat** — notification
- **PostgreSQL** — chat memory cho AI agents

## Flow Overview

```mermaid
flowchart TD
    WH[Webhook] --> BQ1[BigQuery: MSP ticket]
    WH --> BQ2[BigQuery: Benchmark]
    WH --> BQ3[BigQuery: Fallback benchmark]

    BQ1 --> UC[usecase_metric]
    BQ2 --> UC
    BQ1 --> BU[bu_metric]
    BQ2 --> BU
    BQ3 --> MG[Merge 3 nguồn]
    UC --> MG
    BU --> MG

    MG --> FB[final_benchmark]
    FB --> LM[Limit + SKILL_1]

    LM --> AG1["AI Agent1 — Haiku\nChọn channel / objective\nOutput: CPM, CPC, WHT"]
    AG1 --> CJ1[Code JS — clean JSON]

    CJ1 --> AG2["AI Agent — Sonnet\nParse segment, budget\nSKILL_2 + File1"]
    AG2 --> CJ2[Code JS — clean JSON]

    CJ2 --> JS8["Budget algorithm\n4-pass Awareness\n+ Traffic / Engagement"]
    JS8 --> JS13["AVG metric adjust\nCap theo benchmark thực tế"]
    JS13 --> IF{remaining > 0?}

    IF -->|No| MG4[Merge kết quả]
    IF -->|Yes| AG7["AI Agent7 — Haiku\nSecondary channel"]
    AG7 --> JS19[Budget algo channel 2]
    JS19 --> JS21[Merge 2 channels]
    JS21 --> MG4

    MG4 --> SK["AI Agent3 — Haiku\nFill plan rows vào sheet"]
    SK --> CJ5[Rebalance pct_budget]
    CJ5 --> MSP1[BigQuery: lấy issue_key]
    MSP1 --> CP[Google Drive: copy template]

    CP --> UB[Update budget]
    CP --> UIK[Update issue_key]
    CP --> AG4["AI Agent4\nFill Target Audience"]
    CP --> MP[Parse plan rows]

    MP --> CJ6[Map AJ index + metrics]
    CJ6 --> URS2[Update Detail Plan sheet]
    CJ6 --> DEL1[Delete unused rows]
    CJ6 --> DEL2[Delete unused columns]

    CP --> NOTIFY[Python: build URL]
    NOTIFY --> HR1[Lấy messages Google Chat]
    HR1 --> CJ15[Tìm thread theo ticket ID]
    CJ15 --> HR2[Lấy members]
    HR2 --> CJ16[Map assignee → userId]
    CJ16 --> HR3[POST message vào thread]
```

## Các phase chính

| Phase | Mô tả |
|---|---|
| ① Data fetch | Kéo ticket data + benchmark từ BigQuery, merge theo priority: usecase > BU > fallback |
| ② AI Agent 1 | Chọn channel/objective mix, output CPM/CPC/WHT/channels_active |
| ③ AI Agent 2 | Parse segment sizes, match rate, budget, additional phases |
| ④ Budget algorithm | 4-pass awareness + phân bổ các objective còn lại, adjust theo AVG metric |
| ⑤ Secondary channel | Nếu còn remaining budget → chạy thêm channel 2, merge kết quả |
| ⑥ Output | Copy template Drive → điền Sheets → notify Google Chat với mention assignee |

## Docs

- [Budget Algorithm](docs/algorithm.md) — giải thích logic tính toán phân bổ ngân sách
