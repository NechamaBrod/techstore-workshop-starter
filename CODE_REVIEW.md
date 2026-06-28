# TechStore — תיאור הפרויקט וסקירת קוד

> מסמך זה מסכם את מבנה הפרויקט, אופן ההרצה, וסקירת קוד (code review).
> נכתב ב-2026-06-28.

## מה הפרויקט

**TechStore** — אפליקציית **full-stack** לדוגמה (חנות אלקטרוניקה) שמשמשת כ"נקודת פתיחה לסדנה". הקוד, הטסטים וה-CI כבר קיימים; מה שנבנה בסדנה (קובצי context, Hooks, Agents) עדיין לא.

### הסטאק
- **Client** — React + Vite + TypeScript + Tailwind. ראוטינג ב-[App.tsx](client/src/App.tsx), שכבת services מבוססת Axios ([apiClient.ts](client/src/services/apiClient.ts)).
- **Server** — Node + Express 5 + TypeScript + Mongoose (MongoDB). מבנה נקי: routes → middleware → controllers → services → models.
- **shared/** — חבילת טיפוסים משותפת (`@architect/shared`) בין client לשרת.
- ולידציה ב-**Zod**, תיעוד **Swagger** ב-`/api-docs`, טסטים ב-**Vitest**, CI ב-GitHub Actions.

### מה האפליקציה עושה
- **אימות + RBAC** — login/register/logout/me, JWT ב-cookie מסוג `httpOnly`, שלושה תפקידים: `admin` / `manager` / `user` ([Auth.ts](server/src/controllers/Auth.ts), [tokenService.ts](server/src/services/tokenService.ts)).
- **דשבורד ניהולי** (admin/manager) — סטטיסטיקות והזמנות, ודוח אנליטיקה מתקדם עם MongoDB Aggregation `$facet` (הכנסות לפי קטגוריה, top customers, מגמה שבועית, סגמנטציה) ב-[Dashboard.ts](server/src/controllers/Dashboard.ts).
- **חנות ציבורית + מוצרים** — צפייה ציבורית, יצירת מוצר ל-admin בלבד.
- **פיצ'רי AI** (admin בלבד) — יצירת תיאור מוצר וזיהוי מוצר מתמונה דרך OpenAI, עם cache, rate-limit, ו-guardrail נגד prompt-injection ([aiVisionService.ts](server/src/services/aiVisionService.ts)).
- **טופס משוב** ציבורי.

## איך מריצים

דרישות: Node 18+ ו-MongoDB (מקומי או Atlas URI).

```bash
npm run install:all                 # התקנת תלויות root + client + server
cp server/.env.example server/.env  # ערכי את הסודות (JWT_SECRET, MONGO_URI, OPENAI_API_KEY)
npm --prefix server run seed        # אופציונלי — זריעת 50 מוצרים, 22 משתמשים, ~45K₪ הזמנות
npm run dev                         # מריץ client + server יחד
```

- Client: http://localhost:5173 · Server: http://localhost:3000 · Swagger: `/api-docs`
- משתמשי דמו מה-seed: `admin@techstore.com` / `manager@techstore.com`, סיסמה `Admin1234`.
- טסטים: `npm --prefix server test` ו-`npm --prefix client test`.
- AI דורש `OPENAI_API_KEY`; בלעדיו אותם נתיבים מחזירים 503 בצורה מסודרת.

## Code Review

הבסיס **איכותי ובטוח** באופן יחסי: `helmet`, `bcrypt` cost 12, סיסמה `select:false`, הודעות אימות גנריות (לא חושף אילו אימיילים קיימים), `sameSite:lax` + `secure` בפרודקשן, rate-limit על auth ו-AI, ולידציית Zod בכל נתיב, error handler מרכזי עם קודי שגיאה אחידים, ו-guardrail נגד prompt-injection ב-vision. הממצאים למטה הם **שיפורים**, לא כשלים קריטיים.

### Medium

**1. פער הרשאות בדשבורד** — ב-[dashboardRoutes.ts](server/src/routes/dashboardRoutes.ts) הנתיבים `/todays-orders` ו-`/stats` מוגנים רק ב-`requireAuth`. כל משתמש מחובר עם role `user` (לקוח רשום רגיל) יכול לקרוא את **סך ההכנסות, מספר הלקוחות והמוצרים**. רק `/sales-analytics` מוגן ב-`requireRole("admin","manager")`. ה-UI מתייחס לכל הדשבורד כניהולי, אבל ה-API לא אוכף זאת.
→ להוסיף `router.use(requireRole("admin","manager"))` אחרי ה-`requireAuth`.

**2. נתיב feedback ציבורי — אחסון לא חסום + ללא rate-limit** — [Feedback.ts](server/src/controllers/Feedback.ts) דוחף ל-`feedbackStore`, מערך ברמת מודול שלא נקרא ולא נחסם, ו-[feedbackRoutes.ts](server/src/routes/feedbackRoutes.ts) ציבורי לגמרי **ללא rate-limiter**. גדילה בלתי-חסומה = דליפת זיכרון / וקטור DoS, והנתונים נמחקים בכל restart.
→ rate-limiter (כמו `authLimiter`) + persist ל-Mongo, או לפחות TODO מפורש שזה placeholder לסדנה.

### Low / הקשחה

**3. טיפול שגיאות לא עקבי** — [Dashboard.ts](server/src/controllers/Dashboard.ts) משתמש ב-`try/catch` ידני + `res.status(500)` במקום תבנית `asyncHandler` + `errorHandler` המרכזי שבכל שאר הקוד. עובד, אך עוקף את מבנה ה-`{error, code}` האחיד.
→ לעטוף ב-`asyncHandler`.

**4. JWT ללא pinning של אלגוריתם** — [tokenService.ts](server/src/services/tokenService.ts) קורא ל-`jwt.verify` ללא `{ algorithms: ["HS256"] }`. v9 בטוח כברירת מחדל, אבל pinning הוא הקשחה זולה.

**5. מירוץ ביצירת מוצר** — [Products.ts](server/src/controllers/Products.ts) מבצע `findOne(name)` ואז `create` (TOCTOU), ול-`Product.name` אין `unique:true` (בניגוד ל-`Customer.email` שיש לו unique + טיפול 11000). שתי בקשות במקביל יכולות ליצור כפילות.
→ אינדקס unique + טיפול ב-duplicate.

**6. `getAllProducts` ללא pagination** — מחזיר את כל הקולקציה. תקין ב-50 שורות, נקודה לעתיד.

### הערות מודל-נתונים (מחוץ לסקופ)

`Order.items` הוא מערך ObjectId **ללא כמות וללא snapshot מחיר** — לקוח לא יכול לקנות 2 יחידות מאותו פריט, והאנליטיקה סוכמת מחיר קטלוגי נוכחי (משתנה אם מחיר מוצר משתנה). ה-shuffle ב-seed (`sort(() => 0.5 - Math.random())`) מוטה — קוסמטי בלבד.

## עדיפות מומלצת לתיקון

1. ממצא **1** (פער הרשאות) ו-**2** (feedback) — ההשפעה הגבוהה ביותר.
2. אחריהם **3–5** כהקשחות.
