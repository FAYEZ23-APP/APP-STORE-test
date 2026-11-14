# 🚂 النشر على Railway (Deploying to Railway)

الريبو ده فيه الباك إند لوحده (.NET 10)، والـ `Dockerfile` موجود في الجذر، فـ Railway هيكتشفه ويبني بيه تلقائياً.

> [!IMPORTANT]
> **مفيش أي أسرار حقيقية جوه الريبو.** كل القيم في `appsettings.json` قيم تطوير أو `CHANGE_ME`.
> لازم تحط القيم الحقيقية كـ **Variables** في Railway. في ASP.NET Core الـ `:` بيتكتب `__` (شرطتين تحت).

---

## 1. إعداد قاعدة البيانات (PostgreSQL)

1. في مشروع Railway: **New → Database → PostgreSQL**.
2. ⚠️ **التطبيق بيرفض يشتغل في Production بمستخدم `postgres`** (حماية DB-003). لازم تعمل مستخدم مخصص بصلاحيات محدودة.
   افتح **Data → Query** في خدمة Postgres ونفّذ (غيّر كلمة السر):
   ```sql
   CREATE ROLE marketplace_app LOGIN PASSWORD 'ضع_كلمة_سر_قوية_هنا';
   GRANT CONNECT ON DATABASE railway TO marketplace_app;
   GRANT USAGE, CREATE ON SCHEMA public TO marketplace_app;
   ```
3. ⚠️ **لازم `SSL Mode=Require`** في سلسلة الاتصال (حماية DB-004).

---

## 2. المتغيرات المطلوبة (Required Variables)

في خدمة الباك إند → **Variables**:

| المتغير | القيمة | ملاحظة |
| :--- | :--- | :--- |
| `ConnectionStrings__DefaultConnection` | `Host=${{Postgres.PGHOST}};Port=${{Postgres.PGPORT}};Database=${{Postgres.PGDATABASE}};Username=marketplace_app;Password=...;SSL Mode=Require;Trust Server Certificate=true` | **إجباري**. ممنوع `postgres` |
| `Jwt__Key` | نص عشوائي ≥ 32 حرف | **إجباري**. القيمة الافتراضية مرفوضة في Production |
| `Auth__CodeSecret` | نص عشوائي ≥ 32 حرف (مختلف عن Jwt) | **إجباري** |
| `Security__TwoFactorEncryptionKey` | مفتاح عشوائي (Base64 32 بايت) | **إجباري** في Production |
| `ASPNETCORE_ENVIRONMENT` | `Production` | موجود افتراضياً في الـ Dockerfile |

لتوليد قيم عشوائية قوية (PowerShell):
```powershell
[Convert]::ToBase64String((1..48 | % { Get-Random -Max 256 }) -as [byte[]])
```

### اختياري
| المتغير | الغرض |
| :--- | :--- |
| `Admin__Email` / `Admin__Password` | إنشاء حساب الأدمن أول مرة (مفيش أدمن افتراضي) |
| `Admin__WebEnabled=true` | تفعيل لوحة الأدمن. بيتطلب `Admin__RequireHttpsBehindProxy=true` |

> [!CAUTION]
> **ممنوع** تحط `TEST_DB` في Railway، التطبيق هيرفض يشتغل.
> الميزات المالية (`Features__EnableWallet` و`Features__EnableVendorPayouts` و`Features__EnableCoupons`) خليها **مقفولة** لحد الموافقة القانونية والمحاسبية.

---

## 3. سلوك التشغيل (Startup behavior)

- البورت: Railway بيبعت `PORT` تلقائياً، والتطبيق بيسمع على `0.0.0.0:$PORT`.
- ⚠️ **التطبيق بيعمل `MigrateAsync()` تلقائياً عند التشغيل**، يعني هيطبّق كل الـ migrations على قاعدة Railway في أول تشغيل، وهيضيف التصنيفات الأساسية.
- فحص الجاهزية: `GET /api/health`. حطه في **Settings → Healthcheck Path**.
- Swagger مقفول في Production.

---

## 4. ربط الموبايل بالسيرفر

بعد ما Railway يديك رابط `https://xxxx.up.railway.app`:
```env
EXPO_PUBLIC_API_URL=https://xxxx.up.railway.app/api
```
الرابط لازم يكون `https`، وده بيخلّي ممكن تعمل APK إنتاجي (preview/production).

---

## 5. ملاحظة عن الملفات المرفوعة (Uploads)

الصور بتتحفظ في `/app/wwwroot/uploads` جوه الكونتينر، وبتتمسح مع كل Deploy.
عشان تفضل محفوظة، ضيف **Volume** في Railway على المسار `/app/wwwroot/uploads`.
