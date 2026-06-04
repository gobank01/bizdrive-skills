# BIZDRIVE Skills

Claude Code skills ของ BizDrive (พี่แบงค์ปรัชญา) — user-level skills ที่ใช้ได้ทุกโปรเจค

## Skills

| Skill | ทำอะไร |
|---|---|
| [`save`](save/SKILL.md) | บันทึกจุดที่ทำงานค้างไว้ ให้ session หน้าทำต่อได้ทันที (และโหลดกลับมาอ่านตอนเริ่มงานใหม่). เก็บไฟล์ถาวร 1 ไฟล์ต่อ 1 โปรเจคใต้ `~/.claude/handoffs/`. พิมพ์ `/save` เพื่อเซฟ, `/save ต่อ` เพื่อโหลด |

## ติดตั้ง

ก๊อปโฟลเดอร์สกิลที่ต้องการไปไว้ใน `~/.claude/skills/` :

```bash
git clone https://github.com/gobank01/bizdrive-skills.git
cp -r bizdrive-skills/save ~/.claude/skills/
```

จากนั้นเรียกใช้ใน Claude Code ด้วย `/save` ได้ทุกโปรเจค

## License

MIT
