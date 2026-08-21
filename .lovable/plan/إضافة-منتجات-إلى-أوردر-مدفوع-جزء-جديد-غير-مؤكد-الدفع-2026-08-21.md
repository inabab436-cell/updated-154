# إضافة منتجات إلى أوردر مدفوع = جزء جديد غير مؤكد الدفع

## المشكلة في الوضع الحالي

عند إضافة منتج/كمية إلى أوردر `payment_status = 'confirmed'`:

- `update_order_with_stock` تخصم مخزون الزيادة فورًا (لأن الأوردر ليس pending).
- `priceOrderItems` تعيد تسعير السلة كاملة، فتُعاد كتابة `subtotal/discount/total` فوق القيم المؤكدة، وقد يُعاد تطبيق أو إلغاء خصم قديم.
- لا يوجد أي تمييز بين الكمية المدفوعة والكمية المضافة لاحقًا.

## كيف نمثّل الحالة في الـschema

الـschema الحالي لا يستطيع التمييز بين جزء مدفوع وجزء غير مدفوع داخل نفس الصف، و`payment_status` حقل واحد للأوردر كله — لذلك لن نغيّره للطلب كله.

**أقل تعديل ضروري** (نفس صف الأوردر، بدون جدول جديد وبدون أوردر ثانٍ):

```sql
alter table public.orders
  add column if not exists pending_items      jsonb   not null default '[]'::jsonb,
  add column if not exists pending_subtotal   numeric not null default 0,
  add column if not exists pending_discount   numeric not null default 0,
  add column if not exists pending_total      numeric not null default 0,
  add column if not exists pending_since      timestamptz;
```

- `items` + `subtotal_price/discount_amount/total_price` = **الجزء المؤكَّد فقط** (يبقى مجمَّدًا كما تم تأكيده).
- `pending_items` + مبالغه = **الإضافة الجديدة فقط** بانتظار تأكيد الدفع.
- `payment_status` للأوردر يظل `confirmed`؛ وجود `pending_items` غير فارغة هو علامة «توجد إضافة غير مدفوعة». لا نلمس الجزء القديم إطلاقًا.

## تغييرات قاعدة البيانات (ملف هجرة واحد جديد)

`db/2026-09-05_pending_order_additions.sql`:

1. الأعمدة أعلاه.
2. `add_pending_order_items(p_order_number, p_conversation_id, p_merchant_id, p_addition_items, p_subtotal, p_discount, p_total, p_notes)`
   - تُستدعى فقط عندما يكون الأوردر مؤكَّد الدفع.
   - تتحقق من توفر مخزون الإضافة (قفل الصفوف) **بدون أي خصم**.
   - تدمج الإضافة داخل `pending_items` (نفس منطق دمج السطور) وتزيد المبالغ المعلّقة، وتضبط `pending_since`.
   - لا تلمس `items`، ولا المبالغ المؤكَّدة، ولا `payment_status`، ولا `stock_deducted`.
3. توسيع `confirm_order_payment` (نفس اسم الدالة ونفس نقطة الاستدعاء):
   - المسار القديم (`payment_status = 'pending'`) يبقى **حرفيًا كما هو**.
   - مسار جديد: لو `payment_status = 'confirmed'` و`pending_items` غير فارغة → تحقّق من المخزون ثم اخصم **الإضافة فقط**، ثم ادمج `pending_items` داخل `items`، وأضف المبالغ المعلّقة إلى `subtotal_price/discount_amount/total_price`، وفرّغ الحقول المعلّقة، ثم استدعِ `record_order_offer_redemptions` (idempotent، `on conflict do nothing` يمنع التكرار).
   - لو مؤكَّد ولا توجد إضافة معلّقة → نفس السلوك الحالي (`already_confirmed`).
4. `update_order_with_stock` تبقى **بدون تغيير** لمسار الأوردر غير المدفوع.

## تغييرات الكود

- `src/routes/api/chat-ai.ts` (مسار `create_order` عند وجود أوردر قائم):
  - لو الأوردر القائم `pending` → السلوك الحالي بالكامل (`update_order_with_stock`).
  - لو الأوردر القائم مؤكَّد الدفع → تسعير **سطور الإضافة (الدلتا) فقط** عبر `priceOrderItems`، ثم استدعاء `add_pending_order_items`، وإرجاع رد للوكيل يوضّح أن الإضافة مسجَّلة وبانتظار تأكيد الدفع مع قيمتها.
  - حساب الدلتا الحالي (`canonicalizeOrderItems` → `subtractAlreadyDeducted`) يُستخدم كما هو، مع احتساب `pending_items` أيضًا ضمن ما هو مسجَّل حتى لا تتكرر الإضافة.
- `src/lib/order-pricing.server.ts`: بدون تغيير في المنطق؛ يُستدعى على سطور الإضافة فقط (العرض/الخصم يُطبَّق على الجديد فقط).
- `src/lib/order-item-merge.ts`: بدون تغيير (يُعاد استخدامه للدمج داخل الدالة والواجهة).
- ملف جديد `src/lib/order-pending-additions.ts`: دوال خالصة لقراءة الإضافة المعلّقة (هل توجد؟ قيمتها، سطورها) لتستخدمها الواجهة والوكيل، مع اختبارات.
- الواجهة (`src/routes/orders.tsx`، `src/routes/conversation.$id.tsx`، `src/routes/awaiting-payment.tsx`): إظهار الإضافة المعلّقة كسطر منفصل «إضافة بانتظار تأكيد الدفع» مع قيمتها وزر «تأكيد الدفع» الذي يستدعي نفس `confirm_order_payment` الحالي، وإدراج الأوردرات ذات الإضافة المعلّقة ضمن ما يُعرض للتأكيد.
- `src/lib/order-payment.server.ts`: يشمل في التأكيد الجماعي الأوردرات المؤكَّدة التي لديها `pending_items` (نفس RPC، لا منطق جديد).
- `src/lib/order-status-gate.ts`: التنفيذ/الشحن ممنوع طالما هناك إضافة معلّقة غير مدفوعة.

## عقل الوكيل

قاعدة صريحة جديدة في `src/lib/agent-prompt.ts` تحت «ADDING TO A REGISTERED ORDER»:

- الإضافة على أوردر سبق تأكيد دفعه تُعامَل كإضافة **غير مؤكدة الدفع** حتى يؤكدها التاجر؛ لا تُعتبر مدفوعة تلقائيًا.
- تُسعَّر الإضافة على الجديد فقط، والجزء القديم يبقى بسعره وخصمه المسجَّلين.
- المخزون لا يُخصم للإضافة إلا بعد تأكيد دفعها.
- يجب أن يذكر الوكيل للعميل قيمة الإضافة المطلوبة وطريقة دفعها، وأن يميّز بين ما سبق دفعه وما ينتظر الدفع.
- كتلة سياق الأوردر المُرسلة للوكيل تُظهر «مدفوع» مقابل «إضافة بانتظار الدفع».

## اختبارات

- الإضافة على أوردر مدفوع لا تغيّر `items` ولا مبالغ الجزء القديم ولا تخصم مخزونًا.
- التأكيد يخصم الزيادة فقط ثم يدمجها ويجمع المبالغ.
- تأكيد مكرر لا يخصم مرتين ولا يسجّل العرض مرتين.
- الخصم يُحسب على سطور الإضافة فقط.
- مسار الأوردر غير المدفوع (`update_order_with_stock` + الدلتا) يبقى كما هو.
