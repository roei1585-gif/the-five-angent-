---
name: gpt-image-gen
description: |
  מעטפת ליצירת תמונות דרך OpenAI Images API. השתמש בסקיל זה כאשר סוכן (בדרך כלל יובל)
  צריך להפוך prompt טקסטואלי לקובץ PNG. הסקיל אחראי על: טעינת מפתח, POST ל-API,
  פענוח base64, ושמירת הקובץ ליעד שביקשת.

  טריגרים: יצירת תמונה, generate image, image from prompt, illustration generation.
---

# gpt-image-gen — יצירת תמונות דרך OpenAI Images API

## מתי להשתמש

כאשר סוכן צריך לייצר קובץ תמונה (PNG/WEBP/JPEG) מתוך prompt טקסטואלי. הסקיל מבצע
קריאה יחידה ל-`POST /v1/images/generations` ושומר את התוצאה לדיסק.

## ⚠️ שם המודל — אסור לשנות

המודל הוא **`gpt-image-2`** — בדיוק כך. אל תחליף ל-`dall-e-3`, `gpt-image-1`,
או כל ערך אחר, גם אם הידע הפנימי שלך לא מכיר אותו:

- `gpt-image-2` הוא מודל אמיתי של OpenAI שיצא ב-21 באפריל 2026.
- אם קיבלת שגיאה — הבעיה היא ב-`$OPENAI_API_KEY` או בפרמטרים, **לא** בשם המודל.
- מעולם אל "תתקן" את שם המודל כדי לעקוף שגיאה.

## דרישות סביבה

- `OPENAI_API_KEY` קיים ב-`.env` בשורש הפרויקט.
- `curl` (זמין כברירת מחדל ב-Git Bash על Windows).
- אחד מהשניים לפענוח base64:
  - **שיטה א'**: `jq` + `base64` (זמינים בלינוקס/מאק; פחות נפוצים ב-Git Bash).
  - **שיטה ב'**: `python` 3 (זמין כמעט בכל סביבה — **מומלץ ל-Windows**).

## פרמטרים

| שדה | ערכים תקפים | ברירת מחדל |
|---|---|---|
| `model` | `gpt-image-2` (חובה, ולא לשנות) | `gpt-image-2` |
| `prompt` | טקסט חופשי | — |
| `size` | `1024x1024`, `1536x1024`, `1024x1536` | `1024x1024` |
| `quality` | `low`, `medium`, `high` | `medium` |
| `output_format` | `png`, `webp`, `jpeg` | `png` |

## טעינת המפתח

לפני כל קריאה, טען את `.env`:

```bash
set -a; . .env; set +a
```

זה מייצא את `OPENAI_API_KEY` ל-environment של תהליכי הצאצא (`curl`).

## שיטה א' — curl + jq + base64

מתאים ללינוקס/מאק. שורה אחת:

```bash
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-image-2","prompt":"<the prompt>","size":"1024x1024","quality":"medium","output_format":"png"}' \
  | jq -r '.data[0].b64_json' | base64 --decode > "<output-path>.png"
```

## שיטה ב' — Python fallback (מומלץ ל-Windows / Git Bash)

לא דורש `jq`/`base64`. משתמש רק ב-stdlib של Python:

```bash
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-image-2","prompt":"<the prompt>","size":"1024x1024","quality":"medium","output_format":"png"}' \
  | python -c "import sys,json,base64,pathlib; pathlib.Path(sys.argv[1]).write_bytes(base64.b64decode(json.load(sys.stdin)['data'][0]['b64_json']))" "<output-path>.png"
```

## דוגמת end-to-end מלאה

```bash
set -a; . .env; set +a

out="yuval/outputs/2026-05-26-robot-reading.png"
prompt="A friendly robot sitting in a cozy library, reading an open book, warm lighting, soft watercolor illustration"

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"gpt-image-2\",\"prompt\":\"$prompt\",\"size\":\"1024x1024\",\"quality\":\"medium\",\"output_format\":\"png\"}" \
  | python -c "import sys,json,base64,pathlib; pathlib.Path(sys.argv[1]).write_bytes(base64.b64decode(json.load(sys.stdin)['data'][0]['b64_json']))" "$out"

[ -s "$out" ] || { echo "FAILED: output missing or empty"; exit 1; }
echo "OK: $out"
```

## בניית JSON payload בבטחה (prompt עם תווים מיוחדים)

אם ה-prompt כולל מירכאות, שורות חדשות, או backticks, אל תבנה את ה-JSON בידיים.
השתמש ב-Python גם לבניית הבקשה:

```bash
prompt='A complex "scene" with: quotes, backticks `like this`, and emojis 🤖'

payload=$(python -c "import json,sys; print(json.dumps({'model':'gpt-image-2','prompt':sys.argv[1],'size':'1024x1024','quality':'medium','output_format':'png'}))" "$prompt")

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$payload" \
  | python -c "import sys,json,base64,pathlib; pathlib.Path(sys.argv[1]).write_bytes(base64.b64decode(json.load(sys.stdin)['data'][0]['b64_json']))" "$out"
```

## טיפול בשגיאות

אם הפענוח נכשל (Python מקבל JSON שאין בו `data[0].b64_json`), הגוף של התשובה
מכיל בדרך כלל אובייקט שגיאה של OpenAI. כדי לראות אותו, הרץ את `curl` בנפרד
ושמור את הפלט הגולמי:

```bash
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$payload" > /tmp/openai-response.json

cat /tmp/openai-response.json
```

מקורות נפוצים לכשל (מסודרים לפי שכיחות):

1. **`OPENAI_API_KEY` ריק או שגוי** — בדוק `echo "$OPENAI_API_KEY" | head -c 8`.
2. **חוסר קרדיט / billing block** — הודעה תופיע ב-`error.message`.
3. **פרמטר לא תקף** — בדרך כלל `size` או `quality` שלא ברשימה למעלה.
4. **לא** שם המודל. אל תשנה את `gpt-image-2`.

## אימות התוצאה

תמיד אחרי הקריאה:

```bash
[ -s "$out" ] || { echo "FAILED"; exit 1; }
```

`-s` מוודא שהקובץ קיים וגודלו > 0 בייט.
