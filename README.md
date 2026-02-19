# Lab Integrations — Bahmni RTL

Middleware for connecting laboratory analyzer machines to Bahmni/OpenELIS. Results flow automatically from the analyzer into the patient record.

```
Lab Analyzer  ──HL7/TCP──▶  Middleware  ──SQL──▶  OpenELIS  ──▶  OpenMRS
```

## Contents

| File | Description |
|------|-------------|
| `captureHl7Messages_final.rar` | HL7 middleware source code (Java) |
| `MIDDLEWARE.md` | Technical documentation + setup guide |

## Quick Summary

- Java TCP server — one listening thread per analyzer
- Parses HL7/ASTM messages using configurable regex patterns
- Writes results directly into the OpenELIS PostgreSQL database
- Auto-signs results and updates sample status
- Saves raw HL7 messages to disk as audit trail
- Tested with Sysmex XP-300 hematology analyzer

See [MIDDLEWARE.md](MIDDLEWARE.md) for full details and setup instructions.

## License

AGPL-3.0

---

## العربية

# تكامل المختبر — باهمني

برنامج وسيط لربط أجهزة التحليل المخبرية بنظام باهمني/OpenELIS. تنتقل النتائج تلقائياً من جهاز التحليل إلى سجل المريض.

```
جهاز التحليل  ──HL7/TCP──▶  البرنامج الوسيط  ──SQL──▶  OpenELIS  ──▶  OpenMRS
```

## المحتويات

| الملف | الوصف |
|------|-------------|
| `captureHl7Messages_final.rar` | الكود المصدري للبرنامج الوسيط (Java) |
| `MIDDLEWARE.md` | التوثيق التقني ودليل الإعداد |

## ملخص

- خادم TCP بلغة Java — خيط استماع واحد لكل جهاز تحليل
- يحلل رسائل HL7/ASTM باستخدام أنماط regex قابلة للتهيئة
- يكتب النتائج مباشرة في قاعدة بيانات OpenELIS (PostgreSQL)
- يوقّع النتائج تلقائياً ويحدّث حالة العينة
- يحفظ رسائل HL7 الخام على القرص كسجل مرجعي
- تم اختباره مع جهاز Sysmex XP-300 لتحليل الدم

راجع [MIDDLEWARE.md](MIDDLEWARE.md) للتفاصيل الكاملة وتعليمات الإعداد.

## الترخيص

AGPL-3.0
