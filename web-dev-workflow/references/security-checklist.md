# 安全基线(Security Checklist)

写任何接口、表单、数据库查询、文件上传时,默认执行本清单。安全是第一次写代码时就落地的,不是事后修补的——
事后补的成本是当时做的十倍,而且总有漏网的。

**汇报纪律:默认沉默执行,不逐条汇报。** 只有某项因故未覆盖、或有意偏离清单时,才在完成汇报中说明原因。
用户不想读十条"我做了 XSS 转义",只想知道哪里"没"做。

## 清单

1. **所有输入做服务端校验**。客户端校验只是体验,服务端校验才是边界。类型、长度、范围、格式都要管。
2. **参数化查询 / ORM 防注入**。永远不拼接 SQL 字符串;用 ORM 时警惕 raw query 入口。
3. **XSS 转义**。输出用户内容时转义;富文本必须过白名单 sanitizer(如 DOMPurify),不要自己写正则。
   - *栈相关*:React/Vue/Svelte 默认转义,危险出口是 `dangerouslySetInnerHTML` / `v-html` / `{@html}`;
     模板引擎(EJS 的 `<%-`、Handlebars 的三花括号)和手拼 HTML 是重灾区。
4. **CSRF 防护**。
   - *栈相关*:Cookie/Session 认证 → 必须有 CSRF token 或 SameSite=Lax/Strict + 关键操作二次确认;
     纯 JWT 走 Authorization header(不放 cookie)→ 传统 CSRF 基本不适用,不要机械加 token,但要防 XSS 偷 token。
5. **每个接口都有认证 + 授权检查**。认证 = 你是谁;授权 = 你能动这条数据吗。
   重点防**水平越权**:`GET /api/orders/:id` 必须校验这条 order 属于当前用户,不能只查登录态。
   这是真实项目里最常见也最严重的漏洞。
6. **密码用 bcrypt/argon2 哈希**。永不明文、永不 md5/sha1;不要自己发明方案。
7. **密钥走环境变量**。API key、数据库密码、JWT secret 一律 `.env`(并进 `.gitignore`),提供 `.env.example`。
   代码和提交历史里出现真实密钥 = 事故。
8. **Rate limiting**。至少覆盖:登录/注册/密码重置(防爆破)、发送邮件/短信类接口(防滥用)、公开写接口。
9. **安全响应头**。`X-Content-Type-Options: nosniff`、`X-Frame-Options`/CSP、HSTS(上 HTTPS 后)。
   - *栈相关*:Express 用 helmet 一行解决;Next.js 在 `next.config` 配 headers。
10. **文件上传校验**:类型白名单(校验内容而不只是扩展名)、大小上限、存储文件名重新生成(防路径穿越)、
    上传目录不可执行。
11. **错误信息不泄露内部细节**。给用户的错误是友好文案;堆栈、SQL、内部路径只进服务端日志。
    统一错误处理中间件是落实这条的最佳位置。

## 按栈执行,不机械套用

清单是底线不是仪式。先判断项目实际栈,跳过不适用项(如纯静态站没有 5/6/8),补上栈特有的坑
(如 Next.js Server Actions 的输入同样要校验、Supabase 要开 RLS)。判断的依据写在心里,偏离写在汇报里。
