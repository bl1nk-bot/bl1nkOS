---
name: add-tool
description:
---

# 🛠️ เพิ่มเครื่องมือ AI (Add AI Tool) - BL1NK Edition

คู่มือสำหรับการเพิ่ม AI SDK tool ใหม่ลงในแพ็กเกจ `@bl1nk/agent`

---

### 1. สร้างเครื่องมือ (Create the Tool)

สร้างไฟล์ที่ `packages/agent/src/tools/my-tool.ts`

เครื่องมือต้องใช้ `async function*` (generator) และทำการ `yield` สถานะการอัปเดต เพื่อให้ฝั่ง Frontend สามารถแสดงสถานะการโหลดและผลลัพธ์ได้โดยอัตโนมัติ

```typescript
import { tool } from 'ai'
import { z } from 'zod'

export const myTool = tool({
  description: 'คำอธิบายที่ชัดเจนว่าเครื่องมือนี้ทำอะไร — โมเดล AI จะอ่านส่วนนี้เพื่อตัดสินใจว่าจะใช้งานเมื่อใด',
  inputSchema: z.object({
    query: z.string().describe('พารามิเตอร์นี้มีไว้เพื่ออะไร'),
  }),
  execute: async function* ({ query }) {
    // 1. Yield สถานะ loading — ฝั่ง frontend จะแสดงตัวหมุนโหลด (spinner)
    yield { status: 'loading' as const }
    const start = Date.now()

    try {
      // 2. ใส่ตรรกะ (Logic) ของเครื่องมือที่นี่ (เช่น ต่อ API ภายนอก หรือคำนวณ)
      const result = await doSomething(query)

      // 3. Yield สถานะ done พร้อมผลลัพธ์ — ฝั่ง frontend จะแสดงค่า output ออกมา
      yield {
        status: 'done' as const,
        durationMs: Date.now() - start,
        text: result,
        commands: [
          {
            title: `เครื่องมือของฉัน: "${query}"`,
            command: '',
            stdout: result,
            stderr: '',
            exitCode: 0,
            success: true,
          },
        ],
      }
    } catch (error) {
      // 4. Yield สถานะกรณีเกิดข้อผิดพลาด (error)
      yield {
        status: 'done' as const,
        durationMs: Date.now() - start,
        text: '',
        commands: [
          {
            title: `เครื่องมือของฉัน: "${query}"`,
            command: '',
            stdout: '',
            stderr: error instanceof Error ? error.message : 'ล้มเหลว',
            exitCode: 1,
            success: false,
          },
        ],
      }
    }
  },
})
```

💡 ประเด็นสำคัญเกี่ยวกับ Yield
 * yield { status: 'loading' } — ต้องส่งค่านี้ออกไปเป็นอันดับแรก เพื่อให้ Frontend แสดงสถานะกำลังโหลด
 * yield { status: 'done', ... } — ต้องส่งค่านี้ออกไปเป็นอันดับสุดท้าย โดยจะบรรจุผลลัพธ์ที่จะแสดงในหน้า Chat UI
 * รูปแบบของอาร์เรย์ commands จะเหมือนกับ output ของเครื่องมือ sandbox bash เพื่อให้ Frontend แสดงผลในรูปแบบเดียวกัน
 * สามารถดูตัวอย่างการใช้งานจริงได้ที่ packages/agent/src/tools/web-search.ts

### 2. ส่งออกเครื่องมือ (Export the Tool)

เพิ่มการ export ในไฟล์ packages/agent/src/index.ts:

```typescript
export { myTool } from './tools/my-tool'
```

### 3. ลงทะเบียนใน Agent (Register in the Agent)

ในส่วนการสร้าง agent (สำหรับ Next.js คือ apps/app/src/lib/ai/) ให้เพิ่มเครื่องมือลงใน object tools:

```typescript
import { myTool } from '@bl1nk/agent'

const tools = {
  ...bl1nk.tools, // เรียกใช้เครื่องมือมาตรฐานของ BL1NK
  my_tool: myTool,
}
```

### 4. อัปเดต Prompts (ทางเลือก)

หากเครื่องมือต้องการคำแนะนำเฉพาะเจาะจง ให้ทำการอัปเดต prompt ที่เกี่ยวข้องใน packages/agent/src/prompts/:

```
// ในไฟล์ chat.ts หรือ bot.ts ให้เพิ่มข้อมูลในส่วน Available Tools:
// - **my_tool**: คำอธิบายว่าควรใช้เครื่องมือนี้เมื่อใดและอย่างไร
```

### 📦 เครื่องมือที่มีอยู่แล้ว (Existing Tools)

| เครื่องมือ | ไฟล์ | คำอธิบาย |
|---|---|---|
| webSearchTool | tools/web-search.ts | ค้นหาข้อมูลจากเว็บสำหรับข้อมูลที่ไม่มีใน sandbox |
| bash | @bl1nk/sdk | รันคำสั่ง bash ใน sandbox |
| bash_batch | @bl1nk/sdk | รันคำสั่ง bash หลายคำสั่งพร้อมกัน |

---