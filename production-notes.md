# ملاحظات الإنتاج والخطوات التالية

هذا الملف مرجع لأي جلسة تطوير لاحقة (خصوصًا عبر Claude Code) — يلخّص كل قرار معماري
اتُّخذ خلال التصميم، وما تم تنفيذه فعليًا في هذا الـ Scaffold، وما تبقى.

---

## 1) القرارات المعمارية الثابتة (لا تُغيَّر بدون سبب قوي)

- **Tech Stack**: Next.js 14 (Frontend) + NestJS (Backend) + PostgreSQL + Redis + BullMQ
- **قاعدة البيانات**: نمط Base Table + Translation Table لكل محتوى متعدد اللغات (courses, chapters...)
- **المصادقة مزدوجة تمامًا**: `/auth/staff/*` (دعوة فقط، لا تسجيل ذاتي أبدًا) مقابل `/auth/public/*`
  (تسجيل ذاتي + OTP إلزامي، طلاب ومدربون فقط)
- **المدرب**: بعد OTP يبقى `instructor_profiles.account_status = 'active_pending_approval'`
  حتى موافقة إدارية صريحة — لا ينشر أي دورة قبلها
- **RBAC ديناميكي بالكامل من DB** — لا صلاحيات مُثبَّتة بالكود، كل شيء عبر
  `roles` / `permissions` / `role_permissions` + `PermissionsGuard`
- **العملة**: USD كمصدر حقيقة وحيد، تحويل تلقائي للعرض، **تجميد سعر الصرف** عند إنشاء أي `payment_plan`
- **التعثر عن السداد**: مهلة سماح 7 أيام → حجب جزئي (المحتوى المُكمَل يبقى متاحًا، الجديد يُحجب)
- **الموافقة المزدوجة على مستحقات المدربين**: إلزامية لأي `payout` ≥ 500$ (شخصان مختلفان إلزاميًا)
- **عمولة بوابة الدفع تُخصم أولًا** من المبلغ الإجمالي، ثم يُقسَّم الصافي بين المدرب والمنصة
- **حماية الفيديو**: نهج متدرج — Signed URLs + Overlay Watermark فقط في MVP،
  DRM الكامل (Widevine/FairPlay) لاحقًا للمحتوى عالي القيمة فقط
- **الأمان السلوكي (تسريب فيديو/تواصل)**: تنبيه بشري فقط عبر `security_review_cases`،
  **بدون أي إجراء تلقائي** على حساب المستخدم
- **الذكاء الاصطناعي**: توجيه متعدد النماذج حسب الحساسية (Haiku للمهام البسيطة/الحجم الكبير،
  Sonnet للمتوسطة، Opus للحرجة كبنك الأسئلة والتسعير) + حدود استخدام مرنة بالكامل من لوحة الإدارة
  (لا أرقام ثابتة بالكود) + RAG محصور بمحتوى الدورة فقط للطالب

---

## 2) المُنفَّذ فعليًا في هذا الـ Scaffold

- Prisma Schema: `User`, `AuthProvider`, `UserDevice`, `OtpVerification`, `Role`,
  `Permission`, `RolePermission`, `UserRole`, `StaffInvitation`, `ActivityLog`,
  `InstructorProfile`, `Course` + `CourseTranslation`, `Chapter`, `Lesson`, `Enrollment`,
  `CustomRequest` + ملفاتها وسجل حالتها، `RequestStatusTransition`
- Auth: تسجيل ذاتي + OTP (منطق كامل، الإرسال الفعلي Stub)، دعوة موظفين + قبول + دخول منفصل،
  JWT (Access + Refresh)، `PermissionsGuard` + `@RequirePermissions`
- Seed: كل الأدوار السبعة + الصلاحيات الأساسية + أول `super_admin`
- Frontend: توجيه `next-intl` لـ 4 لغات، RTL تلقائي للعربية، middleware، صفحة رئيسية تجريبية

---

## 3) غير مُنفَّذ بعد — بالترتيب المقترح للتنفيذ

### أ) استكمال المصادقة
- [ ] تفعيل الإرسال الفعلي لـ OTP عبر **Twilio Verify** (SMS/WhatsApp حسب دولة المستخدم)
      و**SendGrid** للبريد — راجع `otp.service.ts` (موسوم `TODO(production)`)
- [ ] `passport-google-oauth20` و `passport-microsoft` Strategies + منطق **Account Linking**
      الآمن الموصوف (ربط تلقائي فقط لبريد `email_verified_at` موجود مسبقًا)
- [ ] Refresh Token rotation + endpoint `/auth/refresh`
- [ ] `user_devices` enforcement الفعلي (منع تسجيل دخول من جهاز ثانٍ متزامن)

### ب) الوحدة المالية (الأضخم)
جداول يجب إضافتها لـ `schema.prisma`: `payments`, `payment_plans`, `installments`,
`saved_payment_methods`, `payment_gateways`, `exchange_rates`, `refunds`,
`instructor_earnings`, `instructor_payouts`, `instructor_deductions`,
`instructor_commission_agreements`, `commission_policies`, `dual_approval_thresholds`

- [ ] Payment Gateway Adapter Pattern (`StripeAdapter`, `PayPalAdapter`, `TabbyAdapter`, `TamaraAdapter`)
      خلف `PaymentGatewayAdapter` interface موحّد
- [ ] Cron (BullMQ): تحصيل أقساط داخلية + Dunning (محاولات إعادة + مهلة 7 أيام + حجب جزئي)
- [ ] Cron: تحديث `exchange_rates` من Open Exchange Rates كل 6-24 ساعة
- [ ] منطق `instructor_earnings` بالترتيب الصحيح: gross → خصم عمولة البوابة → تقسيم النسبة
- [ ] Workflow الموافقة المزدوجة على `instructor_payouts` (حد 500$، منع نفس الشخص للموافقتين)

### ج) الطلبات الخاصة (State Machine)
- [ ] تفعيل `request_status_transitions` فعليًا (seed القواعد + خدمة `transitionRequestStatus`
      المركزية الموصوفة في التصميم)
- [ ] رفع الملفات عبر Presigned URLs (S3/R2)
- [ ] ربط AI Gateway لصياغة الـ Brief عند الإنشاء واقتراح التسعير للمراجع

### د) حماية الفيديو
- [ ] تكامل Mux أو Cloudflare Stream (HLS + Signed Playback IDs)
- [ ] `video_playback_sessions` + Heartbeat endpoint
- [ ] Overlay Watermark ديناميكي على مستوى الـ Player (Frontend)

### هـ) الذكاء الاصطناعي
جداول: `ai_interactions`, `ai_usage_limits`, `ai_usage_tracking`, `ai_model_routing`,
`ai_moderation_flags`, `course_content_embeddings` (يتطلب `pgvector` extension على PostgreSQL)

- [ ] AI Gateway Service مركزي (NestJS Module) يستدعي Anthropic API
- [ ] RAG pipeline: تفريغ نصي (Transcript) → تقطيع → embeddings → بحث تشابه محصور بالدورة

### و) الرقابة على التواصل
- [ ] جدول `messages` موحّد + Layer 1 (Regex) + Layer 2 (AI تصنيف عبر Haiku)
- [ ] `security_review_cases` + لوحة مراجعة موحّدة (فيديو + تواصل معًا)

### ز) التقارير
- [ ] Read Replica منفصلة (أو تأجيلها والاكتفاء بـ Materialized Views في البداية)
- [ ] Reporting Service + `report_definitions` (استعلامات Whitelisted فقط، ممنوع SQL حر)
- [ ] توليد PDF عبر Puppeteer + قوالب Handlebars بحسب locale (RTL/LTR ديناميكي)

---

## 4) تنبيه أمني دائم

أي جدول/Endpoint جديد يُضاف مستقبلًا يجب أن يمر عبر نفس الأنماط المُتَّبعة هنا:
- لا صلاحيات مُثبَّتة — تُضاف عبر `permissions` + `role_permissions` في الـ seed
- أي نطاق بيانات شخصي (مالي/أكاديمي) يُحقن شرطه (`WHERE user_id = current_user.id`)
  من الخادم عبر الـ JWT، وليس من مدخلات الطلب أبدًا
- كل عملية حساسة (مالية، صلاحيات، حذف) تُسجَّل في `activity_logs`
