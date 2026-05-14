# تقرير فحص قسم الحسابات والمالية (Accounting Module Audit Report)

بناءً على طلبك، قمت بإجراء فحص عميق (Deep Static Analysis) لجميع ملفات قسم الحسابات والمالية `src/app/api/accounting` ومطابقتها مع بنية قاعدة البيانات الحقيقية (`prisma/schema.prisma`).

النتيجة: **يوجد خلل هيكلي صامت (Silent Schema Mismatches)** منتشر في العديد من الواجهات البرمجية (APIs)، مما سيؤدي إلى ظهور خطأ الخادم `500 Internal Server Error` بمجرد محاولة أي عميل استخدام هذه الميزات. هذا الخلل يتجاوز فحص TypeScript لأن بعض الاستعلامات مكتوبة بشكل ديناميكي (Dynamic Selects).

## ⚠️ المشكلة الجذرية (Root Cause)
هناك العديد من ملفات الـ API تقوم بطلب الحقل `nameAr` من جداول قاعدة البيانات عند جلب البيانات.
ولكن بالعودة إلى تصميم قاعدة البيانات، فإن الحقل `nameAr` غير موجود إطلاقاً في جداول:
- الحسابات (`Account`)
- المنتجات (`Product`)
- العملاء (`Customer`)
- الموردين (`Supplier`)
- مراكز التكلفة (`CostCenter`)

(الاسم العربي في هذه الجداول محفوظ في حقل `name`، بينما الاسم الإنجليزي في `nameEn`).

## 🚨 تفصيل الواجهات المتضررة (Broken Endpoints)
بناءً على الفحص، الملفات والواجهات التالية ستتعطل تماماً إذا تم استدعاؤها ولن تعمل للعملاء الجدد أو الحاليين:

### 1. تقارير الإقفال السنوي (Year-End Close)
- **الملف:** `src/app/api/accounting/year-end/[runId]/reports/route.ts`
- **الخطأ:** محاولة جلب `nameAr` من جدول الحسابات `Account`.

### 2. كشوفات الحساب (Customer / Supplier Statements)
- **الملف:** `src/app/api/accounting/statement/route.ts`
- **الخطأ:** محاولة جلب `nameAr` من جدول العملاء `Customer` والموردين `Supplier`.

### 3. الأرصدة الافتتاحية (Opening Balances)
- **الملف:** `src/app/api/accounting/opening-balances/route.ts`
- **الخطأ:** محاولة قراءة `nameAr` من جدول الحسابات `Account`.

### 4. التقييم المخزني (Inventory Valuation Snapshot)
- **الملف:** `src/app/api/accounting/inventory-valuation-snapshot/route.ts`
- **الخطأ:** محاولة جلب `nameAr` من جدول المنتجات `Product`.

### 5. تسويات فواتير الموردين (GR/IR Clearing)
- **الملف:** `src/app/api/accounting/gr-ir-clearing/route.ts`
- **الخطأ:** قراءة `nameAr` من جدول الموردين `Supplier`.

### 6. تقارير مراكز التكلفة (Cost Center Reports)
- **الملف:** `src/app/api/accounting/cost-center-report/route.ts`
- **الخطأ:** قراءة `nameAr` من جدول مراكز التكلفة `CostCenter`.

### 7. مسار التحصيل (Collection Workflow)
- **الملف:** `src/app/api/accounting/collection-workflow/route.ts`
- **الخطأ:** قراءة `nameAr` من جدول العملاء `Customer`.

### 8. استيراد شجرة الحسابات (Chart of Accounts Import)
- **الملف:** `src/app/api/accounting/chart-of-accounts-import/route.ts`
- **الخطأ:** يعتمد بالكامل على استيراد وتحديث الحقل `nameAr` بدلاً من `name` للحسابات.

### 9. تقادم الديون (Aging Reports)
- **الملف:** `src/app/api/accounting/aging/route.ts`
- **الخطأ:** قراءة `nameAr` للموردين والعملاء.

### 10. المستحقات (Accruals)
- **الملف:** `src/app/api/accounting/accruals/route.ts`
- **الخطأ:** قراءة `nameAr` لحساب المصروفات والمستحقات.

---

## ✅ الواجهات السليمة (Healthy Endpoints)
بقية الواجهات مثل (المراكز الربحية، القيود اليومية، إعدادات الحسابات البنكية، إلخ) تعتبر سليمة من هذه المشكلة الجوهرية لأنها إما تستخدم `name` بشكل صحيح أو تستخدم جداول تحتوي فعلياً على `nameAr` مثل جدول الدفاتر المحاسبية (`AccountingBook`).

## 🛠️ التوصية
كافة الملفات المذكورة أعلاه تحتاج إلى عملية استبدال (Find and Replace) بسيطة:
- تغيير أي `nameAr` إلى `name` عند التعامل مع الجداول المذكورة.
- تغيير أي استدعاء يعتمد على `name` ليصبح `nameEn` إذا كان القصد هو الاسم الإنجليزي.

*تم حفظ هذا التقرير ولن يتم إجراء أي تعديل على الملفات التزاماً بتعليماتك.*
