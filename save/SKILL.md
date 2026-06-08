---
name: save
description: "จุดเซฟแบบเกม (save point) ของโปรเจค — พิมพ์ save ตอนจบงานเพื่อเซฟสถานะ, พิมพ์ save ตอนเริ่ม session ใหม่เพื่อโหลดกลับมาทำต่อทันที. เก็บ 1 ไฟล์ต่อ 1 โปรเจคที่ ~/.handoffs/ (ไฟล์กลาง ใช้ร่วมได้ทุก AI — Claude/Codex/อื่นๆ ชี้มาที่เดียวกัน). เขียนทับทุกครั้ง = ไฟล์ล่าสุดคือแหล่งความจริง. Call at END of session to save, or START of a new one to resume. แทน /handoff เดิม ใช้ไฟล์เดียวกัน."
argument-hint: "[เซฟ|โหลด] หรือบอกว่ารอบหน้าจะโฟกัสอะไร"
---

# 🎮 Save — จุดเซฟโปรเจค (ถาวร, ข้าม session, ข้าม AI)

เก็บ "สถานะงาน" ไว้ **1 ไฟล์ต่อ 1 โปรเจค** เพื่อกลับมาทำต่อได้ทันทีโดยไม่ต้องศึกษา
โปรเจคใหม่ — เหมือนจุดเซฟในเกม. คำสั่งเดียวทำได้ทั้ง **เซฟ** (จบงาน) และ **โหลด/ต่อ**
(เริ่มงานใหม่) — สกิลตัดสินเองจาก argument + ว่ามีงานเกิดใน conversation นี้แล้วหรือยัง.

> **ข้าม AI ได้:** ไฟล์เซฟเป็น Markdown ธรรมดาที่ตำแหน่งกลาง `~/.handoffs/` — Claude,
> Codex, หรือ agent ตัวไหนก็อ่าน/เขียนไฟล์เดียวกัน. เซฟด้วย Claude แล้วเปิด Codex พิมพ์
> `save` ก็โหลดสถานะเดิมต่อได้. (เข้ากันได้กับ `/handoff` เดิม — ไฟล์ชุดเดียวกัน.)

## Step 1 — หาไฟล์เซฟของโปรเจคนี้

ทำก่อนเสมอ. ไฟล์ผูกกับ **root ของโปรเจค** (git top-level ถ้ามี ไม่งั้น cwd) ด้วย slug
แบบ deterministic → ไม่ว่าอยู่ subfolder ไหน หรือใช้ AI ตัวไหน ก็ได้ไฟล์เดียวกัน.

```bash
STORE="$HOME/.handoffs"                 # ตำแหน่งกลาง ใช้ร่วมทุก AI
[ -e "$STORE" ] || STORE="$HOME/.claude/handoffs"   # fallback ถ้า alias หาย
mkdir -p "$STORE"
ROOT="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
SLUG="$(printf '%s' "$ROOT" | sed 's#^/##; s#[/ ]#-#g')"
SAVE_FILE="$STORE/$SLUG.md"
if [ -f "$SAVE_FILE" ]; then
  echo "STATUS=EXISTS"
  echo "MTIME=$(date -r "$SAVE_FILE" '+%Y-%m-%d %H:%M')"
else
  echo "STATUS=MISSING"
fi
echo "FILE=$SAVE_FILE"; echo "ROOT=$ROOT"
echo "BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo -)"
```

## Step 2 — เลือกโหมด

- argument = `save`,`เซฟ`,`บันทึก`,`end`,`stop`,`write`,`จบ`,`พัก` → **SAVE**.
- argument = `load`,`โหลด`,`ต่อ`,`resume`,`start`,`read`,`open`,`อ่าน` → **LOAD**.
- ไม่มี keyword ชัด → ตัดสินอัตโนมัติ:
  - ไฟล์ **มีอยู่แล้ว** + conversation นี้ **ยังแทบไม่มีงานจริง** (เพิ่งเริ่ม session) → **LOAD**.
  - กรณีอื่น (มีงานเกิดจริงใน session นี้ / ยังไม่มีไฟล์) → **SAVE**.
- argument ที่เป็นข้อความอิสระ = คำใบ้ว่ารอบหน้าจะโฟกัสอะไร — SAVE: ปรับเนื้อหาให้สอดคล้อง;
  LOAD: ใช้ bias ว่าจะหยิบส่วนไหนมาทำต่อ.

บอกโหมดที่เลือกสั้นๆ 1 บรรทัดก่อนลงมือ.

> ถ้าเพิ่งเซฟไปและ **ยังไม่มีงานใหม่** ตั้งแต่นั้น — ไม่ต้องเขียนทับซ้ำ บอกผู้ใช้ว่าไฟล์ปัจจุบันอยู่แล้ว.

## Step 3a — LOAD (โหลด / ต่อ)

1. **อ่านไฟล์** `$SAVE_FILE` (ใช้เครื่องมืออ่านไฟล์ของ agent ที่กำลังใช้อยู่ — Read tool /
   `cat` / อะไรก็ได้).
2. บรีฟผู้ใช้กระชับ: ทำอะไรไปแล้ว, ค้างตรงไหน, **step ถัดไปทันที** (จากหัวข้อ "▶️ Resume here"),
   blocker/decision ที่ค้าง. อย่าเทเนื้อหาทั้งไฟล์.
3. **ตรวจก่อนเชื่อ:** เอกสารสะท้อน ณ ตอนเขียน. ถ้าอ้างไฟล์/branch/flag/คำสั่ง ให้เช็คว่ายังมี
   จริงก่อนแนะนำ.
4. มีหัวข้อ "Suggested skills" → เรียกถ้าเข้ากับงานที่จะทำต่อ.
5. **ห้ามเขียนทับไฟล์ในโหมด LOAD** แล้วทำงานต่อ.

## Step 3b — SAVE (เซฟ / เขียนทับ)

**เขียนไฟล์** ไปที่ `$SAVE_FILE` **ทับของเดิม** (ไม่เก็บ history — ไฟล์ล่าสุด = แหล่งความจริง).
เขียนให้ agent ที่ไม่มี context มาก่อนเลย (Claude หรือ Codex) หยิบทำต่อได้ทันที. ใช้โครงนี้:

```markdown
# 🎮 SAVE — <ชื่อโปรเจค>
_Checkpoint: <YYYY-MM-DD HH:MM> · Branch: <branch> · Focus: <arg หรือที่อนุมานได้>_

## ▶️ Resume here
1-2 บรรทัด: เปิด session มาให้ทำ/พิมพ์อะไรต่อทันที (action ที่เป็นรูปธรรมที่สุด).

## State of play
ย่อหน้าเดียว: ตอนนี้งานอยู่ตรงไหน.

## Done this session
- สิ่งที่ทำเสร็จ (อ้าง commit/PR/path — อย่าวาง diff).

## In progress / next step
- งานถัดไปสำคัญสุด 1 อย่าง แล้วตามด้วยที่เหลือเรียงตามความสำคัญ.

## Open decisions & blockers
- อะไรที่ยังค้าง ต้องตัดสินใจหรือปลดล็อกก่อน.

## Key context not in the repo
- gotcha, ความแปลกของ env, ข้อเท็จจริงที่ไม่ชัดในโค้ด. อ้าง PRD/plan/ADR/issue ด้วย path/URL.

## Suggested skills
- สกิลที่รอบหน้าควรเรียก และเพราะอะไร.
```

กฎตอน SAVE:
- **อย่าซ้ำ** เนื้อหาที่มีใน PRD/plan/ADR/issue/commit/diff อยู่แล้ว — อ้าง path/URL.
- **ปกปิดความลับ** — API key, password, token, ข้อมูลส่วนตัว ห้ามเขียนลงไฟล์เด็ดขาด.
- เขียน "▶️ Resume here" ให้เป็น action รูปธรรม (เป็นหัวใจของจุดเซฟ — โหลดมาแล้วเดินต่อได้เลย).
- เขียนเสร็จ ยืนยัน absolute path กลับให้ผู้ใช้ 1 บรรทัด.
