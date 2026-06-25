# TechStore — נקודת פתיחה לסדנה

אפליקציית **Full-Stack** לדוגמה (חנות טכנולוגיה) המשמשת כחומר גלם לסדנה.
זו נקודת ההתחלה: כל קוד האפליקציה, הטסטים וה-CI כבר כאן — מה שתבני בעצמך בסדנה
(קובצי context, Hooks ו-Agents) **עדיין לא קיים**, וזו בדיוק המטרה.

## הסטאק
- **Frontend:** React + Vite + TypeScript, Tailwind CSS
- **Backend:** Node.js + Express + TypeScript
- **Database:** MongoDB (Mongoose)
- **Validation:** Zod · **API Docs:** Swagger/OpenAPI
- **Tests:** Vitest · **CI:** GitHub Actions ([.github/workflows/ci.yml](.github/workflows/ci.yml))

## מבנה התיקיות
```
client/    אפליקציית React (Vite) — רכיבים, עמודים, services
server/    שרת Express — controllers, routes, models, middleware, schemas
shared/    טייפים משותפים בין client ל-server
```

## הרצה מקומית

**דרישות מוקדמות:** Node.js 18+ ו-MongoDB מקומי (או URI ל-Atlas).

```bash
# 1. התקנת תלויות (root + client + server)
npm run install:all

# 2. הגדרת משתני סביבה לשרת
cp server/.env.example server/.env      # ערכי את הערכים לפי הצורך

# 3. (אופציונלי) זריעת נתוני דמו ל-DB
npm --prefix server run seed

# 4. הרצת client + server יחד
npm run dev
```
- Client: http://localhost:5173
- Server: http://localhost:3000 (Swagger בדרך כלל תחת `/api-docs`)

## טסטים
```bash
npm --prefix client test      # טסטים של ה-client (Vitest)
npm --prefix server test      # טסטים של ה-server (Vitest)
```

## קבצים שימושיים
- [server/.env.example](server/.env.example) — תבנית משתני הסביבה.
- [design-system.md](design-system.md) — ייחוס מערכת העיצוב (צבעים, טיפוגרפיה,
  וקוד TSX של כל הרכיבים: Button, Input, Card, Table, Modal ועוד). נוח להעביר
  כ-reference כשבונים רכיב חדש.
