# Claude Code Source Snapshot

> ملاحظة مهمة  
> هذا المجلد يبدو كأنه **لقطة كود (source snapshot)** كبيرة من مشروع قريب جدًا من **Claude Code CLI** أو مبني عليه، وليس سورس نموذج Claude نفسه.  
> النموذج/الأوزان غير موجودة هنا، كما أن هذه النسخة تبدو **غير مكتملة كريبو تشغيل رسمي** لأن ملفات المشروع المعتادة مثل `package.json` و`README` الأصلي وملفات البناء غير ظاهرة في هذا الـ snapshot.

## ما الذي يحتويه هذا المشروع؟

هذا المشروع يحتوي على أغلب الطبقات التي تتوقعها في وكيل برمجي من نوع CLI:

- نقطة دخول للـ CLI وتهيئة الجلسة.
- واجهة طرفية تفاعلية مبنية على React/Ink.
- محرك محادثة agentic loop يرسل الرسائل إلى Anthropic ويستقبل tool calls.
- نظام prompts متعدد الطبقات: system prompt، prompts الأدوات، الذاكرة، الضغط التلقائي للسياق.
- أدوات محلية مثل القراءة، التعديل، البحث، تنفيذ الأوامر، وإدارة المهام.
- تكامل مع MCP كعميل، وكخادم MCP أيضًا.
- دعم للـ skills، والـ commands، والـ subagents، والـ remote sessions.

باختصار: هذا ليس "سورس Claude" كنموذج ذكاء اصطناعي، بل هو أقرب إلى **سورس منظومة التشغيل حول Claude**: الواجهة، orchestration، الأدوات، السياق، والـ prompts.

## التقنيات الظاهرة في الكود

من الملفات الموجودة يمكن استنتاج أن المشروع يعتمد على:

- `TypeScript`
- `Bun` عبر `bun:bundle`
- `React` و`Ink` لواجهة الطرفية
- `@anthropic-ai/sdk`
- `@modelcontextprotocol/sdk`
- بنية كبيرة قائمة على `feature flags`

## المعمارية على مستوى عالٍ

يمكن تلخيص المعمارية بالشكل التالي:

```text
CLI / Entrypoints
    |
    v
Bootstrap + Init + Settings + Session State
    |
    v
Commands / REPL / Main UI
    |
    v
QueryEngine
    |
    +--> Prompt Assembly
    |      - system prompt
    |      - user/project context
    |      - CLAUDE.md
    |      - output styles
    |      - MCP instructions
    |      - memory / compact prompts
    |
    +--> Query Loop
    |      - send to Anthropic API
    |      - receive text / thinking / tool_use
    |      - run tools
    |      - feed tool results back
    |
    +--> Side Systems
           - permissions
           - session memory
           - auto compact
           - MCP
           - remote sessions
           - skills / agents / tasks
```

## دورة العمل الفعلية داخل المشروع

### 1. الإقلاع والدخول

- `entrypoints/cli.tsx` هو الـ bootstrap السريع للـ CLI.
- `main.tsx` يحمل معظم طبقات التطبيق: الإعدادات، الـ commands، الأدوات، الـ MCP، الجلسات، الواجهة، والـ telemetry.
- توجد entrypoints أخرى مثل:
  - `entrypoints/mcp.ts` لتشغيل المشروع كخادم MCP.
  - `entrypoints/agentSdkTypes.ts` لواجهة SDK/الأنواع العامة.

### 2. تحميل الإعدادات والسياق

قبل أول استدعاء للنموذج، يتم تجهيز السياق من عدة مصادر:

- إعدادات الجلسة والـ flags.
- الملفات التعليمية/القاعدية مثل `CLAUDE.md`.
- معلومات البيئة الحالية: المسار، النظام، الـ shell، التاريخ.
- حالة git الأولية.
- الـ output style المختار.
- تعليمات MCP والخدمات المتصلة.

هذه الطبقة تظهر بشكل واضح في:

- `context.ts`
- `utils/claudemd.ts`
- `constants/prompts.ts`
- `constants/outputStyles.ts`

### 3. بناء الـ system prompt

الـ system prompt هنا ليس نصًا واحدًا ثابتًا، بل يتم تركيبه من طبقات:

- تعليمات عامة للسلوك.
- تعليمات تنفيذ المهام البرمجية.
- تعليمات السلامة والصلاحيات.
- معلومات البيئة.
- تعليمات اللغة.
- style prompt.
- تعليمات skills وMCP.
- تعليمات الذاكرة والضغط التلقائي.

الملف الأهم هنا هو:

- `constants/prompts.ts`

وهو القلب الحقيقي لفهم "كيف يفكر الوكيل" قبل أي tool call.

### 4. تجهيز الـ Query Engine

بعد تجهيز الـ prompt والسياق، يقوم `QueryEngine.ts` بـ:

- جلب أجزاء الـ prompt والسياق.
- دمج `defaultSystemPrompt` أو `customSystemPrompt`.
- إضافة `appendSystemPrompt` إن وجد.
- إنشاء `ToolUseContext` الذي يحمل الأدوات، الرسائل، الـ state، والصلاحيات.

هذا الملف هو نقطة الربط بين:

- prompts
- tools
- messages
- permissions
- API layer

### 5. إرسال الطلب للنموذج

بعد ذلك يتم:

- تحويل الـ system prompt إلى blocks.
- تجهيز الرسائل.
- اختيار الموديل وخصائصه.
- استدعاء `anthropic.beta.messages.create(...)`

هذه المرحلة موجودة بشكل أساسي في:

- `services/api/client.ts`
- `services/api/claude.ts`

ومن المهم ملاحظة أن المشروع لا يبدو محصورًا في Anthropic direct API فقط، بل يحتوي أيضًا على مسارات لـ:

- Bedrock
- Foundry
- Vertex

وهذا يعطيه طبقة مزودات (providers) أكثر مرونة من مجرد عميل API بسيط.

### 6. الحلقة الوكيلة Agentic Loop

بعد استجابة النموذج:

- إذا خرج نص عادي، يتم عرضه.
- إذا خرج `tool_use`، يتم تشغيل الأداة المناسبة.
- نتيجة الأداة تعاد إلى الحلقة كرسالة جديدة.
- يتكرر ذلك حتى تتوقف الدورة أو ينتهي الدور.

هذا السلوك موزع بين:

- `query.ts`
- `services/tools/toolExecution.ts`
- `services/tools/toolOrchestration.ts`
- `services/tools/StreamingToolExecutor.ts`

ميزة مهمة هنا أن النظام يفرّق بين:

- أدوات يمكن تشغيلها بالتوازي.
- أدوات يجب تشغيلها بالتسلسل.

أي أن المشروع ليس مجرد "call tool ثم انتظر"، بل عنده orchestration حقيقي للأدوات.

## التركيب العام للمجلدات

فيما يلي خريطة عملية لأهم المجلدات والملفات:

| المسار | الوظيفة |
| --- | --- |
| `entrypoints/` | نقاط دخول CLI/MCP/SDK |
| `main.tsx` | الإقلاع الرئيسي للتطبيق التفاعلي |
| `QueryEngine.ts` | تنسيق الطلبات وبناء الجلسة وربط prompts مع tools |
| `query.ts` | الحلقة الرئيسية للمحادثة وتنفيذ الأدوار |
| `Tool.ts` | العقدة المركزية لأنواع الأدوات و`ToolUseContext` |
| `tools.ts` | السجل المركزي للأدوات المتاحة |
| `tools/` | تنفيذ الأدوات نفسها وprompts الخاصة بها |
| `commands/` | أوامر الـ slash/CLI |
| `constants/` | prompts، ثوابت النماذج، الإعدادات الثابتة |
| `services/` | API، MCP، compact، memory، analytics، tool services |
| `utils/` | وظائف مساعدة كثيرة: auth، config، permissions، files، git، context |
| `skills/` | نظام الـ skills والمهارات القابلة للتحميل/الاكتشاف |
| `state/` | إدارة حالة التطبيق والجلسة |
| `screens/`, `components/`, `ink/` | الواجهة التفاعلية في الطرفية |
| `remote/`, `bridge/`, `server/` | جلسات remote والتحكم والربط |
| `memdir/` | إدارة الذاكرة وملفات التعليمات |
| `tasks/` | إدارة العمل الخلفي والمهام |

## أهم الملفات التي تبدأ منها القراءة

إذا أردت فهم المشروع بسرعة، فهذا هو أفضل ترتيب:

1. `entrypoints/cli.tsx`  
   لتفهم كيف يبدأ التطبيق.

2. `main.tsx`  
   لتفهم كيف يتم تجميع المكونات الكبيرة معًا.

3. `constants/prompts.ts`  
   لتفهم سلوك الوكيل وتعليماته الأساسية.

4. `context.ts` و`utils/claudemd.ts`  
   لتفهم كيف تدخل تعليمات المشروع/المستخدم والسياق إلى النموذج.

5. `QueryEngine.ts`  
   لتفهم كيف يتم تركيب كل ذلك في طلب فعلي.

6. `query.ts`  
   لتفهم حلقة العمل كاملة.

7. `Tool.ts` ثم `tools.ts`  
   لتفهم عقدة الأدوات وسجلها المركزي.

8. `services/tools/toolExecution.ts` و`services/tools/toolOrchestration.ts`  
   لتفهم تشغيل الأدوات فعلًا.

9. `services/api/claude.ts` و`services/api/client.ts`  
   لتفهم طبقة الإرسال إلى API.

10. `services/mcp/`  
   لتفهم التكامل مع MCP.

## نظام الـ prompts في هذا المشروع

واحدة من أهم نقاط القوة هنا أن الـ prompts موزعة منطقيًا، وليست محشورة في ملف وحيد.

### 1. الـ system prompt الرئيسي

المصدر الأهم:

- `constants/prompts.ts`

وهو يحتوي على:

- مقدمة سلوك الوكيل
- تعليمات تنفيذ المهام
- تعليمات tone/style
- تعليمات safety
- تعليمات الأدوات
- طبقات ديناميكية للسياق
- تقسيم بين جزء static وجزء dynamic لتحسين prompt caching

### 2. أنماط الإخراج

المصدر:

- `constants/outputStyles.ts`

وهنا توجد أوضاع مثل:

- `Explanatory`
- `Learning`

ما يعني أن سلوك الرد ليس ثابتًا فقط، بل قابل للتبديل بأسلوب موجه للتعلم أو الشرح.

### 3. prompts الأدوات

كل أداة تقريبًا لها prompt مستقل داخل:

- `tools/*/prompt.ts`

مثل:

- `tools/BashTool/prompt.ts`
- `tools/FileReadTool/prompt.ts`
- `tools/AgentTool/prompt.ts`
- `tools/AskUserQuestionTool/prompt.ts`

وهذا مهم جدًا إذا كنت تريد:

- تعديل سلوك أداة معينة
- فهم كيف تُوصف الأداة للنموذج
- تقليل أو توسيع الحرية الممنوحة للأداة

### 4. prompts الذاكرة والضغط التلقائي

المصادر:

- `services/SessionMemory/prompts.ts`
- `services/compact/prompt.ts`
- `services/MagicDocs/prompts.ts`

وهذه مسؤولة عن:

- تلخيص الجلسة
- تحديث ملاحظات الذاكرة
- تحديث ملفات الوثائق
- ضغط السياق عند كبر المحادثة

### 5. prompts المشروع/المستخدم

هذا المشروع يعتمد بشدة على:

- `CLAUDE.md`
- `.claude/CLAUDE.md`
- القواعد داخل `.claude/rules/`

تحميل هذه الملفات موضح في:

- `utils/claudemd.ts`
- `context.ts`

وهنا تكمن إحدى أقوى ميزاته: **تخصيص السلوك لكل مشروع دون تعديل الكود نفسه**.

## أهم الميزات وكيف نستفيد منها

### 1. المعمارية القائمة على الأدوات

**ما هي؟**

النموذج لا يعمل كنص فقط، بل كوكيل يستخدم أدوات محلية وخارجية: قراءة ملفات، تحرير، تنفيذ أوامر، بحث، MCP، مهارات، مهام، وغير ذلك.

**كيف تستفيد؟**

- إذا كنت تريد بناء وكيل برمجي مشابه، فهذا المشروع يعطيك مثالًا ممتازًا على separation بين:
  - model loop
  - tool registry
  - execution
  - permissions
- إذا أردت إضافة أداة جديدة، ادرس:
  - `Tool.ts`
  - `tools.ts`
  - مجلد `tools/`

### 2. prompt layering بدل prompt واحد

**ما هي؟**

السلوك النهائي ناتج عن تركيب عدة طبقات من التعليمات:

- system prompt
- output style
- project memory
- user memory
- tool prompts
- MCP instructions
- session memory

**كيف تستفيد؟**

- إذا كنت تريد جعل الوكيل أكثر انضباطًا، عدّل الطبقات العامة في `constants/prompts.ts`.
- إذا أردت تخصيص السلوك لمشروع محدد، استخدم `CLAUDE.md` بدل تعديل الكود.
- إذا أردت تعديل وصف أداة بعينها، عدّل `tools/.../prompt.ts`.

### 3. نظام الصلاحيات والحوكمة

**ما هي؟**

هناك طبقة Permissions واضحة تحكم:

- متى يحق للأداة أن تعمل
- متى يجب سؤال المستخدم
- متى يمنع التنفيذ
- كيف تُسجل القرارات

**كيف تستفيد؟**

- هذا مفيد جدًا إذا كنت تبني agent يعمل على ملفات حقيقية أو shell.
- يمكنك دراسة كيف يتم الحد من المخاطر قبل تقليد سلوك "وكيل يحرر المشروع" في منتجك.

### 4. الذاكرة المستمرة Session Memory

**ما هي؟**

المشروع يملك نظامًا يحدث ملف ملاحظات للجلسة تلقائيًا في الخلفية، لتقليل ضياع السياق مع الزمن.

**كيف تستفيد؟**

- إذا كنت تبني وكيلًا طويل الجلسات، هذا القسم من أكثر الأجزاء قيمة للدراسة.
- ابدأ من:
  - `services/SessionMemory/sessionMemory.ts`
  - `services/SessionMemory/prompts.ts`

### 5. الضغط التلقائي للسياق Auto Compact

**ما هي؟**

عندما يكبر السياق، لا يعتمد المشروع فقط على truncation، بل يضغط المحادثة إلى summary قابل للمتابعة.

**كيف تستفيد؟**

- هذا جزء عملي جدًا إذا كان مشروعك يتعامل مع جلسات طويلة.
- ابدأ من:
  - `services/compact/compact.ts`
  - `services/compact/prompt.ts`

### 6. دعم MCP كعميل وكخادم

**ما هي؟**

المشروع لا يستهلك MCP فقط، بل يمكنه أيضًا عرض أدواته كخادم MCP.

**كيف تستفيد؟**

- إذا كنت تريد توصيل الوكيل بأدوات خارجية، ادرس:
  - `services/mcp/client.ts`
  - `services/mcp/`
- إذا أردت أن تعرض أدوات هذا المشروع لعملاء آخرين، ادرس:
  - `entrypoints/mcp.ts`

### 7. دعم الـ skills والـ commands

**ما هي؟**

هناك طبقة skills فوق الأدوات، تسمح بتركيب workflows أعلى مستوى من مجرد tool calls منفصلة.

**كيف تستفيد؟**

- إذا كنت تريد بناء "أوامر ذكية" reusable فوق الوكيل، فهذا القسم عملي جدًا.
- ابدأ من:
  - `skills/`
  - `skills/loadSkillsDir.ts`
  - `commands.ts`

### 8. subagents / teammates / coordinator

**ما هي؟**

المشروع يحتوي على نمط agents متعدّد، مع إمكانية تفويض المهام أو fork أو teammate/coordinator flows.

**كيف تستفيد؟**

- إذا كان هدفك بناء multi-agent system، فهذا القسم مهم جدًا.
- ابدأ من:
  - `tools/AgentTool/`
  - `coordinator/`
  - `tasks/`

### 9. remote sessions والربط عن بعد

**ما هي؟**

يوجد دعم لجلسات بعيدة، WebSocket، permission requests عن بعد، وربط بين عميل محلي وجلسة remote.

**كيف تستفيد؟**

- إذا كنت تريد فصل الواجهة عن بيئة التنفيذ، فهذا المشروع يعطيك مسارًا عمليًا.
- راجع:
  - `remote/RemoteSessionManager.ts`
  - `remote/`
  - `bridge/`
  - `server/`

### 10. دعم عدة مزودات للنموذج

**ما هي؟**

الكود يشير إلى دعم أكثر من backend، وليس فقط Anthropic المباشر.

**كيف تستفيد؟**

- إذا كنت تريد abstraction بين المزودات، راجع:
  - `services/api/client.ts`
  - `utils/model/`

## كيف نستفيد من هذا السورس عمليًا؟

### إذا كان هدفك فهم Claude Code

افعل الآتي:

1. اقرأ `constants/prompts.ts`
2. اقرأ `context.ts` و`utils/claudemd.ts`
3. اقرأ `QueryEngine.ts`
4. اقرأ `query.ts`
5. اقرأ `Tool.ts` و`tools.ts`

بهذا التسلسل ستفهم 80% من السلوك العام سريعًا.

### إذا كان هدفك تعديل الـ prompts

ابدأ من:

- `constants/prompts.ts`
- `constants/outputStyles.ts`
- `tools/*/prompt.ts`

ولا تبدأ من `query.ts` إلا إذا كنت تريد تغيير منطق الحلقة نفسها.

### إذا كان هدفك إضافة أداة جديدة

المسار العملي:

1. أنشئ الأداة داخل `tools/<NewTool>/`
2. عرّف prompt/input/output/schema
3. أضفها إلى `tools.ts`
4. راقب أثرها على الصلاحيات والـ prompt

### إذا كان هدفك تخصيص السلوك لكل مشروع

أفضل مكان ليس الكود، بل:

- `CLAUDE.md`
- `.claude/CLAUDE.md`
- `.claude/rules/*.md`

وهذا مناسب جدًا إن كنت تريد:

- قواعد تحرير خاصة بالمشروع
- تعليمات testing
- أسلوب commit/PR
- قيود أمان

### إذا كان هدفك بناء Agent مشابه من الصفر

أهم الأفكار التي تستحق إعادة الاستخدام:

- فصل prompt assembly عن query execution
- وجود `ToolUseContext` موحد
- registry مركزي للأدوات
- طبقة permissions مستقلة
- memory + compaction
- دعم project-level instructions عبر ملفات خارجية

## لماذا هذا المشروع مثير للاهتمام؟

لأنه يوضح أن قوة هذا النوع من المنتجات لا تأتي من "prompt واحد خارق"، بل من تركيب عدة طبقات تعمل معًا:

- prompts
- state
- permissions
- tools
- memory
- compression
- integrations
- remote execution

وهذا بالضبط ما يحتاجه أي وكيل برمجي جاد.

## ملاحظات وحدود هذه النسخة

قبل أن تتعامل مع هذا المجلد كمنتج جاهز، انتبه للنقاط التالية:

- لا يظهر `package.json` أو ملفات البناء المعتادة، لذلك قد لا يكون المشروع قابلًا للتشغيل مباشرة.
- يوجد الكثير من المسارات المبنية على `feature flags`، وبعضها قد يكون داخليًا أو خاصًا ببناءات معينة.
- بعض النصوص تقول "official CLI for Claude"، لكن هذا الـ snapshot نفسه موجود داخل مستودع شخصي محلي، لذلك يجب التمييز بين **محتوى الكود** وبين **توثيق المصدر الأصلي**.
- هذا ليس سورس الموديل نفسه، بل سورس المنظومة المحيطة بالموديل.

## أفضل خطة لاستكشاف المشروع

إذا أردت دراسة هذا المشروع بترتيب عملي، استخدم هذا المسار:

1. **الفهم العام**
   - `README.md`
   - `entrypoints/cli.tsx`
   - `main.tsx`

2. **فهم السلوك**
   - `constants/prompts.ts`
   - `constants/outputStyles.ts`
   - `context.ts`
   - `utils/claudemd.ts`

3. **فهم الحلقة الوكيلة**
   - `QueryEngine.ts`
   - `query.ts`

4. **فهم الأدوات**
   - `Tool.ts`
   - `tools.ts`
   - `services/tools/`
   - `tools/`

5. **فهم الذاكرة والاستمرارية**
   - `services/SessionMemory/`
   - `services/compact/`

6. **فهم التكاملات**
   - `services/mcp/`
   - `remote/`
   - `bridge/`

## خلاصة

إذا كنت تبحث عن:

- فهم كيف يعمل وكيل برمجي متقدم داخل الطرفية
- دراسة prompts واقعية وليست أمثلة تعليمية مبسطة
- بناء نظام tools + permissions + memory + MCP
- تخصيص سلوك الوكيل على مستوى المشروع

فهذا المجلد مفيد جدًا كمادة دراسة معمارية.

أما إذا كنت تبحث عن:

- أوزان نموذج Claude
- سورس نموذج inference نفسه
- ريبو رسمي كامل جاهز للبناء والتشغيل

فهذا **ليس ما يظهر في هذه النسخة**.
