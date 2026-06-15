# 可复用模式物料(Patterns)

skill 多处要求"第一次写某类东西时先建立可复用的模式",这份文件就是那套模式的**现货**,免得每次临场现编、产出漂移。

怎么用这份文件:

- **契约**是栈无关的底线,任何项目都成立,**不要破坏**。
- **示例**用 Next.js(App Router)和 Vue 落地,只为校准粒度。**用别的栈就照契约把示例改写一下直接套用**,别照抄,也别因为"我这栈不一样"就丢掉契约。
- **MVP 导向(砍范围不砍质量)**:这几样模式本身就是"地基",**必须按生产级做扎实**,不能用"反正是 MVP"省掉。
  "不镀金"只针对**范围**——不提前抽象、不加这版用不上的配置项,而不是降低这些模式的质量。判据见 SKILL.md"MVP 的标准"。
  模式是为了"加一个同类东西变成填空题",不是炫技。
- 第一次按这里确立,**记进约定文档**;第 N 次严格复用,不发明第二套。

---

## 1. 统一 API 错误响应格式

**契约**:

- 所有接口的成功与失败都返回**同构 JSON**,调用方解析方式统一。
- 失败统一形如 `{ error: { code, message } }`,并配**正确的 HTTP 状态码**(400/401/403/404/409/500…)。
- `code` 机器可读(枚举,前端据此分支);`message` 对用户友好。
- 服务端记录完整堆栈/内部细节;**对外绝不泄露**栈、SQL、内部路径(见 security-checklist 第 11 条)。
- 用**一个统一出口**(error handler / wrapper)产出错误响应,不在每个接口里手写 try/catch 拼 JSON。

**Next.js(App Router Route Handler)示例**:

```ts
// lib/api/errors.ts
export class AppError extends Error {
  constructor(public code: string, message: string, public status = 400) {
    super(message);
  }
}

// lib/api/handler.ts —— 统一包裹,所有 route 用它
import { NextResponse } from "next/server";
export function route(fn: (req: Request) => Promise<Response>) {
  return async (req: Request) => {
    try {
      return await fn(req);
    } catch (e) {
      if (e instanceof AppError) {
        return NextResponse.json({ error: { code: e.code, message: e.message } }, { status: e.status });
      }
      console.error(e); // 完整细节只进服务端日志
      return NextResponse.json(
        { error: { code: "INTERNAL", message: "服务器开小差了,请稍后再试" } },
        { status: 500 },
      );
    }
  };
}

// app/api/posts/route.ts
export const POST = route(async (req) => {
  // ...业务;要报错就 throw new AppError("FORBIDDEN", "无权操作", 403)
  return NextResponse.json({ data: created });
});
```

**别的栈怎么改**:Express → 写一个 error-handling 中间件(`app.use((err,req,res,next)=>…)`)产出同样的 JSON;
Vue 项目的后端(Nest/Fastify/Hono…)同理,核心是"一个统一出口 + 同构结构",示例的形状照搬即可。

---

## 2. 输入校验

**契约**:

- **服务端校验才是边界**,客户端校验只是体验(见 security-checklist 第 1 条)。
- 用 **schema 库**(zod / valibot / yup)把规则定义一次,前后端尽量**共用同一份 schema**,不手写 if 判断散落各处。
- 校验失败 → 走上面的错误格式,`code: "VALIDATION"`,HTTP 400,最好带字段级错误。

**示例(zod,前后端共用)**:

```ts
// shared/schemas/post.ts
import { z } from "zod";
export const createPostSchema = z.object({
  title: z.string().min(1, "标题不能为空").max(120),
  body: z.string().min(1),
});
export type CreatePostInput = z.infer<typeof createPostSchema>;

// 服务端(接上面的 route)
const parsed = createPostSchema.safeParse(await req.json());
if (!parsed.success) {
  throw new AppError("VALIDATION", parsed.error.issues[0].message, 400);
}
// parsed.data 已是干净且类型安全的数据
```

前端表单用**同一个** `createPostSchema` 做即时校验(见第 4 节),规则只有一处真相。

---

## 3. 前端统一请求层

**契约**:

- **禁止散落的裸 `fetch`/`axios` 调用**。所有请求走一个 `api` 模块。
- 这一层统一负责:base URL、认证头、超时、**把后端 `{ error }` 解包成可抛出的错误**(让调用方拿到干净的 `message`)、统一 loading 约定。
- 调用方只写业务,不重复处理这些横切关注点。

**示例(TypeScript,框架无关的 fetch 封装)**:

```ts
// lib/api/client.ts
export class ApiError extends Error {
  constructor(public code: string, message: string, public status: number) {
    super(message);
  }
}

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`/api${path}`, {
    ...init,
    headers: { "Content-Type": "application/json", ...(init?.headers ?? {}) },
    // 认证头在此统一注入(从 store/cookie 读 token)
  });
  const json = await res.json().catch(() => ({}));
  if (!res.ok) {
    const err = json.error ?? { code: "UNKNOWN", message: "请求失败" };
    throw new ApiError(err.code, err.message, res.status);
  }
  return json.data as T;
}
```

- **React/Next**:配合 TanStack Query 或自写 hook 调 `api()`,自动管理 loading/error。
- **Vue**:在 composable(如 `usePost()`)里调 `api()`,用 `ref` 管理 `data/loading/error`,或配 VueQuery。

---

## 4. 表单模式

**契约**:任何表单都具备——

1. **客户端即时校验**(复用第 2 节的 schema,提升体验);
2. **服务端权威校验**(边界);
3. **提交中禁用按钮 + loading**,防重复提交;
4. **成功/失败明确反馈**,失败时**字段级错误**展示。

**Next.js / React 示例(精简)**:

```tsx
const [errors, setErrors] = useState<Record<string, string>>({});
const [submitting, setSubmitting] = useState(false);

async function onSubmit(values: CreatePostInput) {
  const parsed = createPostSchema.safeParse(values);
  if (!parsed.success) { setErrors(fieldErrors(parsed.error)); return; }
  setSubmitting(true);
  try {
    await api("/posts", { method: "POST", body: JSON.stringify(values) });
    toast.success("已发布");
  } catch (e) {
    if (e instanceof ApiError) toast.error(e.message); // 后端权威错误
  } finally {
    setSubmitting(false);
  }
}
// <button disabled={submitting}>{submitting ? "提交中…" : "发布"}</button>
```

**Vue 示例(精简)**:

```vue
<script setup lang="ts">
const errors = ref<Record<string, string>>({});
const submitting = ref(false);
async function onSubmit() {
  const parsed = createPostSchema.safeParse(form);
  if (!parsed.success) { errors.value = fieldErrors(parsed.error); return; }
  submitting.value = true;
  try { await api("/posts", { method: "POST", body: JSON.stringify(form) }); }
  catch (e) { if (e instanceof ApiError) toast.error(e.message); }
  finally { submitting.value = false; }
}
</script>
<!-- <button :disabled="submitting">{{ submitting ? "提交中…" : "发布" }}</button> -->
```

---

## 5. 页面三态

**契约**:任何要取数据的页面/区块都自带 **loading 态、空态、错误态**,并做响应式。
把它抽成**一个可复用的封装**(组件或 hook),不要每个页面手写三套 if。

**React 示例(可复用包装组件)**:

```tsx
function AsyncBoundary<T>({ query, children, empty }: {
  query: { data?: T; isLoading: boolean; error?: unknown };
  children: (data: T) => React.ReactNode;
  empty?: React.ReactNode;
}) {
  if (query.isLoading) return <Spinner />;
  if (query.error) return <ErrorState message="加载失败,点击重试" />;
  if (isEmpty(query.data)) return <>{empty ?? <EmptyState text="这里还什么都没有" />}</>;
  return <>{children(query.data as T)}</>;
}
```

**Vue 示例(同一套约定,用 slot)**:

```vue
<!-- <AsyncBoundary :loading="loading" :error="error" :empty="!items.length"> -->
<!--   <template #loading><Spinner/></template> -->
<!--   <template #error><ErrorState/></template> -->
<!--   <template #empty><EmptyState text="这里还什么都没有"/></template> -->
<!--   <PostList :items="items"/> -->
<!-- </AsyncBoundary> -->
```

空态文案要具体("还没有评论""搜索无结果"),不要只甩一句"无数据";错误态要能重试。

---

## 落地纪律

- **第一次**写某类东西时按上面的契约建立模式,把选定的实现(错误格式、schema 库、请求层位置、三态组件)**记进约定文档**。
- **第 N 次**严格复用,不发明第二套错误格式、不写裸 fetch、不另起一套三态。
- **契约是底线,示例是校准**:换栈就改写示例,但别破坏契约。发现自己写的和已有模式不一致时,停下来对齐。
