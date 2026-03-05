<div align="center">
  <a href="./README.md">English</a> |
  <b>العربية</b>
</div>

---

<div dir="rtl" align="right">

# تكامل المختبر — نظام البهمني باللغة العربية

برنامج وسيط لربط أجهزة تحليل المختبر بنظام Bahmni/OpenELIS. تنتقل النتائج تلقائياً من جهاز التحليل إلى سجل المريض.

```
Lab Analyzer  ──HL7/TCP──▶  Middleware  ──SQL──▶  OpenELIS  ──▶  OpenMRS
```

## المحتويات

| الملف | الوصف |
|------|-------------|
| `captureHl7Messages_final.rar` | الشيفرة المصدرية للبرنامج الوسيط HL7 (Java) |
| `middleware-docs.md` | التوثيق التقني + دليل الإعداد |

---

## نظرة عامة

**captureHl7Messages** هو برنامج وسيط مكتوب بلغة Java يربط أجهزة تحليل المختبر بنظام المعلومات الصحية Bahmni/OpenELIS. يستمع البرنامج لرسائل HL7 (Health Level 7) المرسلة عبر مقابس TCP من أجهزة التحليل، ويحلل النتائج، ويكتبها مباشرة في قاعدة بيانات PostgreSQL الخاصة بـ OpenELIS.

### آلية العمل

```
┌──────────────┐     HL7/TCP      ┌─────────────────────┐     SQL/JDBC     ┌──────────────┐
│ Lab Analyzer  │ ──────────────▶ │  captureHl7Messages  │ ──────────────▶ │  OpenELIS DB  │
│ (e.g. XP-300) │   (socket)      │     (middleware)      │  (PostgreSQL)   │   (Bahmni)    │
└──────────────┘                  └─────────────────────┘                  └──────────────┘
```

**مسار العمل:**
1. يُكمل جهاز التحليل الفحص ويُرسل النتائج كرسالة HL7 عبر TCP
2. يستقبل البرنامج الوسيط الرسالة، ويستخرج رقم العينة ونتائج الفحوصات باستخدام أنماط regex قابلة للتعديل
3. يتحقق البرنامج من قاعدة بيانات OpenELIS للبحث عن عينة معلقة مطابقة
4. في حال العثور عليها، يُدرج النتائج، ويُنشئ توقيعات النتائج، ويُحدّث حالة التحليل، ويُعلّم العينة كمعالجة
5. تُحفظ رسائل HL7 الخام على القرص كملفات احتياطية

---

## البنية التقنية

### هيكل الأصناف (Classes)

| الصنف | الدور |
|-------|------|
| **Main** | نقطة الدخول. يقرأ إعدادات أجهزة التحليل من ملف JSON، ويُهيئ مجمع اتصالات PostgreSQL، ويُنشئ خيطاً واحداً `WorkingThread` لكل جهاز تحليل. |
| **WorkingThread** | حلقة المعالجة الأساسية. يفتح `ServerSocket` على المنفذ المُعد للجهاز، ويقبل الاتصالات، ويحلل رسائل HL7 باستخدام regex، ويُنفذ عمليات قاعدة البيانات. |
| **Analyzer** | كائن إعدادات POJO لجهاز تحليل واحد. يحتوي على المنفذ، عنوان IP، أنماط regex، بادئة رقم العينة، مسار الإخراج، إعدادات ACK، وطرق كشف نهاية العينة. |
| **DataBaseUtility** | مدير مجمع اتصالات PostgreSQL (Singleton) باستخدام `PGPoolingDataSource`. يقرأ إعدادات قاعدة البيانات من `openelisDB.json`. |
| **PostgresDBConnectionInfo** | كائن POJO لمعاملات اتصال قاعدة البيانات (عنوان IP للخادم، المنفذ، اسم قاعدة البيانات، بيانات الاعتماد، الحد الأقصى للاتصالات). |
| **Result** | صنف بيانات بسيط يحمل نتيجة فحص محللة: اسم الفحص، القيمة، وعلامة الشذوذ. |

### المكتبات المطلوبة

| المكتبة | الإصدار | الغرض |
|---------|---------|---------|
| Jackson Databind | 2.16.1 | تحليل JSON لملفات الإعدادات |
| Jackson Annotations | 2.16.1 | تعليقات JSON التوضيحية |
| Jackson Core | 2.16.1 | معالجة JSON الأساسية |
| PostgreSQL JDBC | — | الاتصال بقاعدة بيانات OpenELIS |
| Log4j API | 2.23.0 | إطار عمل التسجيل |
| Log4j Core | 2.23.0 | تطبيق التسجيل |

---

## الإعدادات

### إعدادات جهاز التحليل (JSON)

يقرأ البرنامج الوسيط تعريفات أجهزة التحليل من مصفوفة JSON. كل عنصر يحدد:

```json
[
  {
    "name": "SYSMIX",
    "port": 9100,
    "ip": "192.168.1.50",
    "prefix": "300",
    "outputPath": "/var/log/hl7/SYSMIX/",
    "sampleIDLineRegex": "O\\|\\d+\\|\\|\\^\\^\\s+(\\d+)",
    "valueLineRegex": "R\\|\\d+\\|\\^\\^\\^\\^(\\w+)\\^\\d+\\|\\s*([\\d.]+)",
    "testNameGroupNum": 1,
    "testValueGroupNum": 2,
    "flagGroupNum": -1,
    "expectedNumOfLines": 0,
    "lastLineRegex": "L\\|1\\|N",
    "highExp": "H",
    "lowExp": "L",
    "nonPrintableStop": false,
    "sendACK": false,
    "controlIDRegex": "",
    "ackMsgTemplate": ""
  }
]
```

| الحقل | النوع | الوصف |
|-------|------|-------------|
| `name` | String | معرّف فريد لجهاز التحليل. يجب أن يطابق `machineid` في جدول `machine_mapping`. |
| `port` | int | منفذ TCP للاستماع لرسائل HL7 من هذا الجهاز. |
| `ip` | String | عنوان IP لجهاز التحليل (للمعلومات فقط). |
| `prefix` | String | بادئة رقم القبول تُضاف لأرقام العينات عند الاستعلام من قاعدة البيانات. |
| `outputPath` | String | مسار المجلد لحفظ ملفات رسائل HL7 الخام كنسخ احتياطية. |
| `sampleIDLineRegex` | String | نمط regex لاستخراج رقم العينة/القبول من رسائل HL7. المجموعة 1 تلتقط الرقم. |
| `valueLineRegex` | String | نمط regex لاستخراج أسماء الفحوصات والقيم من سطور النتائج. |
| `testNameGroupNum` | int | رقم مجموعة regex لاسم الفحص في `valueLineRegex`. |
| `testValueGroupNum` | int | رقم مجموعة regex لقيمة الفحص في `valueLineRegex`. |
| `flagGroupNum` | int | رقم مجموعة regex لعلامة الشذوذ. تُعيَّن إلى `-1` إذا كانت غير قابلة للتطبيق. |
| `expectedNumOfLines` | int | إذا كانت > 0، تُعتبر العينة مكتملة بعد هذا العدد من الأسطر. |
| `lastLineRegex` | String | نمط regex يطابق السطر الأخير من كتلة رسالة HL7 للعينة. |
| `highExp` / `lowExp` | String | قيم العلامات التي تشير إلى نتائج مرتفعة/منخفضة غير طبيعية (مثل "H"، "L"). |
| `nonPrintableStop` | boolean | إذا كانت true، فإن حرفاً غير قابل للطباعة يُشير إلى نهاية العينة. |
| `sendACK` | boolean | إذا كانت true، يُرسل رسالة تأكيد HL7 ACK إلى الجهاز بعد المعالجة. |
| `controlIDRegex` | String | نمط regex لاستخراج معرّف التحكم بالرسالة (لرسائل ACK). |
| `ackMsgTemplate` | String | قالب رسالة HL7 ACK. يُستبدل `#@msgControlID#@` بمعرّف التحكم الفعلي. |

### إعدادات قاعدة البيانات (`openelisDB.json`)

```json
[
  {
    "serverIP": "localhost",
    "port": 5432,
    "dbName": "clinlims",
    "userName": "clinlims",
    "password": "clinlims",
    "maxDBConnections": 10
  }
]
```

### إعدادات التسجيل (`log4j2.xml`)

- إخراج وحدة التحكم + ملحق ملفات متدحرجة
- تدور ملفات السجل عند 10 ميغابايت، مع الاحتفاظ بـ 5 أرشيفات
- تُحذف الملفات الأقدم من 30 يوماً تلقائياً
- **ملاحظة:** حدّث خاصية `LOG_DIR` لتطابق مسار النشر الخاص بك

---

## تفاعلات قاعدة البيانات

يتفاعل البرنامج الوسيط مع جداول OpenELIS التالية:

### الجداول المعنية

| الجدول | العملية | الغرض |
|-------|-----------|---------|
| `sample` | SELECT, UPDATE | البحث عن العينات المعلقة برقم القبول؛ تعليمها كمعالجة (status_id = 2) |
| `sample_item` | SELECT (عبر استعلام فرعي) | ربط العينات بعناصرها |
| `machine_mapping` | SELECT | ربط رموز فحوصات الجهاز بمعرّفات فحوصات OpenELIS |
| `result` | INSERT | تخزين قيم نتائج الفحوصات |
| `result_signature` | INSERT | التوقيع التلقائي على النتائج باسم "Open ELIS" |
| `analysis` | UPDATE | تعليم التحليل كمكتمل (status_id = 16) |
| `result_seq` | NEXTVAL | توليد معرّفات النتائج |
| `result_signature_seq` | NEXTVAL | توليد معرّفات توقيعات النتائج |

### مسار المعالجة (لكل عينة)

```
1. Receive HL7 message → Extract sample ID
2. SELECT from sample WHERE accession_number = prefix + sampleID
   └─ If no pending sample found → skip, log warning
3. For each test result in the message:
   a. Check if test exists in machine_mapping for this analyzer
   b. Check if the test slot is empty (no existing result)
   c. NEXTVAL('result_seq') → get new result ID
   d. INSERT INTO result (value, abnormal flag, etc.)
   e. NEXTVAL('result_signature_seq') → get signature ID
   f. INSERT INTO result_signature (auto-signed by "Open ELIS")
   g. UPDATE analysis SET status_id = 16 (completed)
4. UPDATE sample SET status_id = 2 (processed)
5. Save raw HL7 to file for audit trail
```

---

## تنسيق رسائل HL7

يحلل البرنامج الوسيط رسائل بأسلوب HL7 (متغير بروتوكول ASTM/LIS2-A2). مثال من جهاز تحليل الدم Sysmex XP-300:

```
H|\^&|||XP-300^00-14^^^^B6508^AP807129||||||||E1394-97    ← Header
P|1                                                        ← Patient
O|1||^^           1984^M|^^^^WBC\^^^^RBC\...|||||||N|...   ← Order (sample ID: 1984)
R|1|^^^^WBC^1| 10.2|10*3/uL||N||||||20240221164355        ← Result (WBC = 10.2)
R|2|^^^^RBC^1| 4.73|10*6/uL||N||||||20240221164355        ← Result (RBC = 4.73)
R|3|^^^^HGB^1| 13.2|g/dL||N||||||20240221164355           ← Result (HGB = 13.2)
...
R|20|^^^^PCT^1| 0.25|%||W||||||20240221164355             ← Result (PCT = 0.25)
L|1|N                                                      ← Terminator
```

**الأجزاء الرئيسية:**
- **H** — الترويسة: تحدد طراز الجهاز والرقم التسلسلي
- **P** — المريض: تسلسل المريض
- **O** — الطلب: يحتوي على رقم القبول/العينة والفحوصات المطلوبة
- **R** — النتيجة: سطر واحد لكل فحص مع الاسم، القيمة، الوحدات، وعلامة الشذوذ (N=طبيعي، H=مرتفع، L=منخفض، W=تحذير)
- **L** — المنهي: يُشير إلى نهاية الرسالة

---

## كشف نهاية العينة

يستخدم البرنامج الوسيط ثلاث استراتيجيات لكشف اكتمال استقبال رسالة العينة (تُقيَّم بالترتيب):

1. **عدد الأسطر** — إذا كانت `expectedNumOfLines > 0`، تُعتبر الرسالة مكتملة عند استقبال هذا العدد من الأسطر.
2. **نمط regex للسطر الأخير** — إذا طابق السطر الحالي `lastLineRegex` (مثل `L|1|N`)، تُعتبر الرسالة مكتملة.
3. **حرف غير قابل للطباعة** — إذا كانت `nonPrintableStop` مفعّلة (true)، فإن حرفاً غير قابل للطباعة في بداية السطر يُشير إلى الاكتمال.

---

## دليل النشر

### المتطلبات الأساسية

```bash
# Java 8+ runtime
sudo apt install openjdk-11-jre-headless

# Verify
java -version
```

### 1. إنشاء هيكل المجلدات

```bash
mkdir -p /opt/hl7-middleware
mkdir -p /var/log/hl7/{SYSMIX,logs}   # one subdir per analyzer + logs dir
```

### 2. وضع الملفات

```
/opt/hl7-middleware/
├── captureHl7Messages.jar    # the fat JAR (built from IntelliJ artifact)
├── openelisDB.json           # DB connection config
└── analyzers.json            # analyzer definitions
```

ملف JAR يُبنى كعنصر IntelliJ IDEA مع جميع المكتبات المطلوبة مُضمّنة:
```bash
# Output: out/artifacts/captureHl7Messages_jar/captureHl7Messages.jar
```

### 3. إعداد اتصال قاعدة البيانات (`openelisDB.json`)

وجّه هذا الملف إلى نسخة PostgreSQL الخاصة بـ OpenELIS. إذا كان Bahmni يعمل في حاويات Docker، استخدم عنوان IP الحاوية أو اسم شبكة Docker:

```json
[
  {
    "serverIP": "172.18.0.2",
    "port": 5432,
    "dbName": "clinlims",
    "userName": "clinlims",
    "password": "<actual_password>",
    "maxDBConnections": 10
  }
]
```

> **ملاحظة Docker:** يحتاج البرنامج الوسيط إما للعمل على شبكة المضيف أو داخل شبكة Docker للوصول إلى حاوية PostgreSQL. استخدم `docker inspect <container>` للعثور على عنوان IP الحاوية، أو أضف البرنامج الوسيط إلى نفس شبكة Docker.

### 4. إعداد جهاز/أجهزة التحليل (`analyzers.json`)

كل جهاز تحليل في مصفوفة JSON يحصل على خيط استماع خاص به. مثال لجهاز Sysmex XP-300:

```json
[
  {
    "name": "XP-300",
    "port": 9100,
    "ip": "0.0.0.0",
    "prefix": "300",
    "outputPath": "/var/log/hl7/SYSMIX/",
    "sampleIDLineRegex": "O\\|\\d+\\|\\|\\^\\^\\s+(\\d+)",
    "valueLineRegex": "R\\|\\d+\\|\\^\\^\\^\\^(\\w[\\w-]*)\\^\\d+\\|\\s*([\\d.]+)\\|[^|]*\\|\\|([NHLW])",
    "testNameGroupNum": 1,
    "testValueGroupNum": 2,
    "flagGroupNum": 3,
    "expectedNumOfLines": 0,
    "lastLineRegex": "L\\|1\\|N",
    "highExp": "H",
    "lowExp": "L",
    "nonPrintableStop": false,
    "sendACK": false,
    "controlIDRegex": "",
    "ackMsgTemplate": ""
  }
]
```

> **⚠️ ملاحظة:** قد تكون آلية تحميل مسار ملف إعدادات الجهاز في `Main.java` مشفرة صلبةً (hardcoded) أو تُمرر كمعامل سطر أوامر. تحقق من ملف JAR أو الشيفرة المصدرية قبل النشر.

### 5. ملء جدول `machine_mapping`

جدول `machine_mapping` هو جدول مخصص في قاعدة بيانات OpenELIS يربط رموز فحوصات الجهاز بمعرّفات فحوصات OpenELIS. صف واحد لكل فحص لكل جهاز:

```sql
-- Connect to the clinlims database
INSERT INTO machine_mapping (machineid, machinetestcode, listestcode)
VALUES 
  ('XP-300', 'WBC', '<openelis_test_id>'),
  ('XP-300', 'RBC', '<openelis_test_id>'),
  ('XP-300', 'HGB', '<openelis_test_id>'),
  ('XP-300', 'HCT', '<openelis_test_id>'),
  ('XP-300', 'MCV', '<openelis_test_id>'),
  ('XP-300', 'MCH', '<openelis_test_id>'),
  ('XP-300', 'MCHC', '<openelis_test_id>'),
  ('XP-300', 'PLT', '<openelis_test_id>'),
  ('XP-300', 'LYM%', '<openelis_test_id>'),
  ('XP-300', 'MXD%', '<openelis_test_id>'),
  ('XP-300', 'NEUT%', '<openelis_test_id>'),
  ('XP-300', 'LYM#', '<openelis_test_id>'),
  ('XP-300', 'MXD#', '<openelis_test_id>'),
  ('XP-300', 'NEUT#', '<openelis_test_id>'),
  ('XP-300', 'RDW-SD', '<openelis_test_id>'),
  ('XP-300', 'RDW-CV', '<openelis_test_id>'),
  ('XP-300', 'PDW', '<openelis_test_id>'),
  ('XP-300', 'MPV', '<openelis_test_id>'),
  ('XP-300', 'P-LCR', '<openelis_test_id>'),
  ('XP-300', 'PCT', '<openelis_test_id>');
```

يجب أن يطابق `machineid` حقل `name` في إعدادات JSON للجهاز. يجب أن تطابق قيم `listestcode` حقل `test_id` في جدول `test` الخاص بـ OpenELIS.

### 6. الشبكة وجدار الحماية

يجب أن يتمكن جهاز التحليل من الوصول إلى خادم البرنامج الوسيط على منفذ TCP المُعد:

```bash
# Open the port (if using ufw)
sudo ufw allow 9100/tcp

# On the analyzer hardware, configure the LIS host IP and port
# to point to the middleware server's IP address
```

### 7. إصلاح مسار السجل

يُشير ملف `log4j2.xml` المُضمّن إلى مسار تطوير macOS. تجاوزه عبر خاصية JVM:

```bash
java -DLOG_DIR=/var/log/hl7/logs -jar captureHl7Messages.jar
```

### 8. التشغيل كخدمة Systemd

```ini
# /etc/systemd/system/hl7-middleware.service
[Unit]
Description=HL7 Lab Middleware
After=network.target docker.service

[Service]
Type=simple
WorkingDirectory=/opt/hl7-middleware
ExecStart=/usr/bin/java -DLOG_DIR=/var/log/hl7/logs -jar captureHl7Messages.jar
Restart=always
RestartSec=10
User=bahmni

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable hl7-middleware
sudo systemctl start hl7-middleware

# Check status
sudo systemctl status hl7-middleware
journalctl -u hl7-middleware -f
```

### إيقاف التشغيل

يُسجل البرنامج الوسيط خطاف إيقاف JVM (shutdown hook) يُغلق مجمع اتصالات قاعدة البيانات بشكل نظيف. أمر `systemctl stop` يُفعّل إيقافاً سلساً.

---

## إضافة جهاز تحليل جديد

1. **التقاط رسالة HL7 نموذجية** من الجهاز باستخدام مستمع TCP:
   ```bash
   nc -l -p 9100 > sample_message.txt
   ```

2. **كتابة أنماط regex** لـ:
   - استخراج رقم العينة (`sampleIDLineRegex`)
   - استخراج نتائج الفحوصات (`valueLineRegex`)
   - كشف نهاية الرسالة (`lastLineRegex`)

3. **إنشاء إدخالات `machine_mapping`** في قاعدة بيانات OpenELIS:
   ```sql
   INSERT INTO machine_mapping (machineid, machinetestcode, listestcode)
   VALUES ('NEW_ANALYZER', 'WBC', '<openelis_test_id>');
   -- repeat for each test
   ```

4. **إضافة الجهاز** إلى مصفوفة إعدادات JSON بمنفذ فريد.

5. **إنشاء مجلد الإخراج** لملفات HL7 الاحتياطية.

6. **فتح منفذ جدار الحماية** للجهاز الجديد.

7. **إعادة تشغيل البرنامج الوسيط:**
   ```bash
   sudo systemctl restart hl7-middleware
   ```

---

## القيود المعروفة

- **خيط واحد لكل جهاز** — يحصل كل جهاز على خيط واحد. إذا أرسل الجهاز عينة جديدة قبل اكتمال معالجة السابقة، قد تتداخل الرسائل.
- **لا يوجد TLS** — اتصالات TCP غير مشفرة. مناسب فقط لشبكات المستشفيات المعزولة.
- **خطر حقن SQL** — تستخدم الاستعلامات `String.format()` بدلاً من الاستعلامات ذات المعاملات. أرقام القبول ورموز الفحوصات تأتي من عتاد الجهاز، لكن يجب تقوية هذا الجانب.
- **لا توجد آلية إعادة اتصال** — إذا فشل اتصال قاعدة البيانات أثناء المعالجة، قد تُكتب نتائج العينة الحالية جزئياً.
- **لا يوجد التفاف بالمعاملات** — إدراج النتائج وإنشاء التوقيعات وتحديث التحليل غير مغلفة في معاملة واحدة. قد يؤدي فشل في المنتصف إلى نتائج جزئية.
- **تحميل إعدادات الجهاز** — قد يكون مسار ملف JSON للجهاز مشفراً صلبةً في `Main.java`. تحقق قبل النشر.

---

## الرخصة

AGPL-3.0

</div>
