# AI 生成全栈项目 + 实时预览：技术方案思路

## 1. 场景与目标

**场景**：不懂编码的人员通过自然语言向 AI 描述产品需求，系统自动生成前后端分离的项目，并在浏览器中**实时渲染显示**。

**目标**：
- 需求 → 可运行的前端页面 + 服务端接口
- 前后端分离架构
- 浏览器端实时预览（所见即所得）
- 非技术人员可理解、可迭代（通过继续对话修改）

---

## 2. 整体流程（端到端）

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  用户自然语言    │ ──► │  AI 需求理解与   │ ──► │  代码生成引擎       │
│  描述产品需求    │     │  拆解（规划）    │     │  前端 + 后端        │
└─────────────────┘     └──────────────────┘     └──────────┬──────────┘
                                                             │
                                                             ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  浏览器实时      │ ◄── │  预览/运行环境    │ ◄── │  项目文件/API 服务  │
│  渲染与交互      │     │  (沙箱 + 热更新)  │     │  持久化与启动       │
└─────────────────┘     └──────────────────┘     └─────────────────────┘
```

1. **需求输入**：用户用自然语言描述（如「做一个待办列表，可以增删改查，有登录」）。
2. **AI 规划与拆解**：将需求拆成「前端页面结构、接口契约、数据模型」等可执行规格。
3. **代码生成**：按规格生成前端代码（React/Vue 等）和后端接口（Node/FastAPI 等）。
4. **运行与预览**：在隔离环境中启动前后端，通过热更新/代理在浏览器中实时展示。

---

## 3. 技术架构分层

### 3.1 需求理解与规划层（Agent + 规划）

| 模块 | 职责 | 技术思路 |
|------|------|----------|
| 需求解析 | 从自然语言抽取出「功能点、实体、页面、接口」 | LLM + 结构化输出（JSON Schema），如「页面列表、接口列表、数据模型」 |
| 规划/拆解 | 把需求变成可执行的「任务序列」 | ReAct / 思维链；输出：先建数据模型 → 再建 API → 再建前端页面 |
| 规格输出 | 供代码生成使用的统一规格 | 固定 Schema：`{ pages, apis, dataModels, routes }` 等 |

**要点**：  
- 用 **Few-shot + 严格 Schema** 减少幻觉，保证下游代码生成拿到的是一致、可解析的规格。  
- 可引入「多轮澄清」：当需求模糊时，Agent 生成 1～2 个选择题让用户点选，再继续生成。

### 3.2 代码生成层（前端 + 后端）

| 维度 | 建议 | 说明 |
|------|------|------|
| 前端技术栈 | React + Tailwind + 组件库（如 Shadcn）或 Vue 3 + 同类型方案 | 组件化明确、Prompt 易写、社区示例多；便于「按页面/组件」增量生成 |
| 后端技术栈 | Node.js (Express/Fastify) 或 Python (FastAPI) | 与前端同语言可减少上下文切换；FastAPI 自带 OpenAPI，便于前端对接 |
| 生成粒度 | 按「页面/接口」为单位生成，再组装 | 避免单次生成长度过大；便于后续「只改某一页/某一接口」的迭代 |
| 接口契约 | 先定 OpenAPI / 简单 JSON 描述，再生成后端实现和前端请求代码 | 保证前后端约定一致，减少联调失败 |

**生成策略**：  
- **模板 + 填空**：常用 CRUD、列表页、表单页用模板，LLM 只填「实体名、字段、路由」等。  
- **纯生成**：针对复杂交互或自定义页面，用 LLM 直接生成组件/接口代码。  
- **混合**：先由规划层输出规格，再对每个「页面/接口」调用代码生成（可同模型，也可分工：一模型负责规划，一模型负责代码）。

### 3.3 运行与预览层（实时在浏览器里看到）

| 能力 | 实现思路 |
|------|----------|
| 前端实时预览 | 生成的前端代码落在虚拟文件系统或真实目录，通过 **Dev Server（Vite/Webpack）** 跑起来，浏览器 iframe 或新 Tab 打开该 Dev Server 的 URL；代码变更时热更新（HMR） |
| 后端接口可用 | 在沙箱/容器或本机子进程中启动后端服务（如 `node server.js` / `uvicorn app:app`），并分配端口；前端通过「预览环境」的代理访问该端口，避免跨域 |
| 沙箱与安全 | 后端/构建进程放在 **Docker 容器** 或 **云函数** 中执行，限制网络、文件系统、内存；前端若需执行用户代码（不推荐），可用 **iframe + postMessage** 或 **Web Worker** 隔离 |
| 持久化 | 生成的项目文件写入「项目存储」（对象存储或宿主机目录），并记录「当前预览」对应的 commit/版本，便于回滚或对比 |

**「实时」的两种含义**：  
1. **流式生成 + 分段预览**：每生成完一个页面或一个接口就触发一次预览更新（例如先展示静态页，再挂上接口）。  
2. **全量生成后再预览**：全部生成完毕再启动前后端、刷新预览。  
前者体验更好，实现难度更高（需要增量构建、部分热更新）。

---

## 4. 关键技术点

### 4.1 需求 → 结构化规格（Structured Output）

- 用 LLM 输出 **JSON**，并约定 Schema，例如：

```json
{
  "pages": [
    { "name": "TodoList", "path": "/", "type": "list", "entity": "Todo", "actions": ["create", "edit", "delete"] }
  ],
  "apis": [
    { "method": "GET", "path": "/api/todos", "description": "获取列表" },
    { "method": "POST", "path": "/api/todos", "body": "Todo" }
  ],
  "dataModels": [
    { "name": "Todo", "fields": [{ "name": "title", "type": "string" }, { "name": "done", "type": "boolean" }] }
  ]
}
```

- 实现方式：**OpenAI / Claude 的 structured output**，或 **Pydantic + 解析重试**，保证规格可被下游稳定消费。

### 4.2 前端实时预览的几种实现

| 方案 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| A. 真实 Dev Server | 在服务器/容器里起 Vite（或 CRA），把生成代码写到目录，浏览器访问该 Server 的 URL | 最接近真实项目，HMR 自然支持 | 需要固定/动态端口、资源占用、多用户要隔离 |
| B. 在线 IDE 型 | 类似 CodeSandbox/StackBlitz：把项目放在虚拟 FS，用 WebContainers 等在浏览器内跑 Node，再起 Dev Server | 用户端无需自建后端，体验统一 | 依赖 WebContainers 等，复杂度高 |
| C. 运行时渲染组件 | 仅生成「组件树 + Props」的 JSON，前端用预置的组件库按 JSON 渲染（不跑完整 SPA） | 实现简单、安全、无需起 Node | 灵活性低，只适合标准化页面 |

**推荐**：  
- 若资源可控、希望「真实项目级」预览：采用 **A**，并为每个会话/项目分配独立目录与端口。  
- 若希望零后端、纯浏览器：采用 **B** 或 **C**；**C** 更适合「表单/列表/仪表盘」类标准化产品。

### 4.3 后端接口的生成与调用

- **接口契约**：由规划层输出的 `apis` + `dataModels` 生成 **OpenAPI 描述** 或简单路由表。  
- **后端实现**：根据契约生成路由与 Handler（如 Express 的 `app.get/post` 或 FastAPI 的 `@router`）；可先用 **内存存储**（数组/Map）实现 CRUD，满足「可运行、可预览」即可。  
- **前端调用**：生成 `axios/fetch` 或基于 OpenAPI 的 client，请求发往「预览环境代理」转发的后端端口。  
- **跨域**：预览页与后端不同源时，通过 **预览平台的反向代理** 把 `/api/*` 转到后端服务，避免 CORS。

### 4.4 安全与隔离

- **执行后端/构建的代码**：必须在 **Docker 或沙箱进程** 中跑，限制 CPU/内存/网络。  
- **用户上传或生成的文件**：不直接在本机敏感路径执行，使用独立工作目录。  
- **前端若渲染「用户自定义组件」**：优先用 **服务端构建好的静态资源** 在 iframe 中展示，避免 `eval` 或 `new Function` 跑用户代码；若必须跑用户代码，再用 iframe + CSP 或 Web Worker 做隔离。

---

## 5. 技术选型小结

| 层级 | 可选方案 | 备注 |
|------|----------|------|
| 需求解析与规划 | Claude / GPT-4 + Structured Output | 规划能力要强，输出要稳定 |
| 前端生成 | React + Vite + Tailwind + Shadcn UI | 便于按页面/组件生成与 HMR |
| 后端生成 | Node (Fastify) 或 Python (FastAPI) | 与前端协作简单，FastAPI 自带 OpenAPI |
| 预览运行时 | 方案 A：服务器上 Vite + 反向代理；或 方案 B/C | 见 4.2 |
| 代码执行/构建 | Docker 容器 或 云函数 | 隔离与资源限制 |
| 会话与项目存储 | Redis + 对象存储 / 文件系统 | 会话状态 + 生成的项目文件 |

---

## 6. 多 Agent 闭环方案

下面把「规范 Agent → 动态 Prompt 组装 → 前端/后端 Agent 生成 → 联调 Agent → 测试 Agent」整理成闭环方案，并给出核心伪代码。

### 6.1 角色分工与输入输出

| Agent | 输入 | 输出 | 说明 |
|------|------|------|------|
| 规范 Agent | 用户需求 | JSON Schema | 需求结构化，成为唯一可信规格 |
| Prompt 组装器 | JSON Schema | 前端/后端 Prompt | 将 Schema 转成高质量生成指令 |
| 前端页面 Agent | 前端 Prompt | 前端代码 | 生成页面、组件、路由、请求封装 |
| 服务端接口 Agent | 后端 Prompt | 后端代码 | 生成 API、数据模型、路由、校验 |
| 联调 Agent | 前后端代码 | 修复 patch | 对齐接口路径、字段、类型、错误处理 |
| 测试 Agent | 代码 + 运行结果 | 测试报告 + 修复建议 | 检测类型、lint、接口可用性 |

### 6.2 多 Agent 协调与编排机制

**编排层职责**：  
- **路由与调度**：决定哪个 Agent 何时执行、是否并行。  
- **状态机**：维护任务阶段与通过/失败门禁。  
- **共享内存**：统一存放 Schema、代码产物、日志与补丁。  
- **失败回流**：根据错误类型决定回到哪个环节。  

**协调方式**：  
- **工作流编排器（Orchestrator）**：中心化调度，适合确定性流程。  
- **事件总线（Event Bus）**：Agent 输出事件，编排器消费事件驱动下一步。  
- **黑板系统（Blackboard）**：所有 Agent 在同一空间读写产物，避免上下文丢失。  

**并行与串行规则**：  
- 前端 Agent 与后端 Agent **可并行**生成。  
- 联调 Agent 与测试 Agent **必须串行**执行。  
- 失败回流时，优先 **局部修复**，避免全量重生成。

**编排伪代码（DAG + 事件驱动）**：

```typescript
const workflow = {
  schema: ["build_prompts"],
  build_prompts: ["gen_frontend", "gen_backend"],
  gen_frontend: ["integrate"],
  gen_backend: ["integrate"],
  integrate: ["test"],
  test: ["publish"]
};

async function orchestrate(input) {
  emit("schema:start");
  const schema = await schemaAgent.generateSchema(input);
  save("schema", schema);
  emit("schema:done");

  emit("build_prompts:start");
  const prompts = buildPromptsFromSchema(schema);
  save("prompts", prompts);
  emit("build_prompts:done");

  await Promise.all([
    frontendAgent.generateCode(prompts.fePrompt).then(code => save("fe", code)),
    backendAgent.generateCode(prompts.bePrompt).then(code => save("be", code))
  ]);

  const patched = await integrationAgent.align(load("fe"), load("be"), schema);
  save("patched", patched);

  const report = await testAgent.run(patched);
  if (!report.passed) return retryLoop(report, schema, patched);

  return publishPreview(patched);
}
```

### 6.3 闭环流程图（文本版）

```
用户需求
   │
   ▼
规范 Agent ──> JSON Schema
   │
   ▼
Prompt 组装器 ──> 前端 Prompt / 后端 Prompt
   │                     │
   ▼                     ▼
前端页面 Agent        服务端接口 Agent
   │                     │
   └─────────┬──────────┘
             ▼
         联调 Agent
             ▼
         测试 Agent
             ▼
       通过 / 失败
           ├─ 通过：发布预览链接
           └─ 失败：回到联调/生成环节
```

### 6.4 核心伪代码（闭环编排）

```typescript
async function runFullStackGeneration(userRequirement: string) {
  // 1) 规范 Agent：把需求变成 JSON Schema
  const schema = await schemaAgent.generateSchema(userRequirement);
  validateSchema(schema);

  // 2) Prompt 组装：根据 Schema 生成前后端 Prompt
  const { fePrompt, bePrompt } = buildPromptsFromSchema(schema);

  // 3) 代码生成：前后端分别生成
  const feCode = await frontendAgent.generateCode(fePrompt);
  const beCode = await backendAgent.generateCode(bePrompt);

  // 4) 联调：读取前后端代码，对齐接口契约
  const patched = await integrationAgent.align(feCode, beCode, schema);

  // 5) 测试：类型检查、lint、基础接口连通性
  const testReport = await testAgent.run(patched);

  // 6) 闭环控制
  if (testReport.passed) {
    return publishPreview(patched);
  }

  // 失败回流：把错误回馈给联调或生成环节
  return retryLoop(testReport, schema, patched);
}
```

### 6.5 Schema 驱动 Prompt 组装伪代码

```typescript
function buildPromptsFromSchema(schema) {
  const fePrompt = `
你是前端页面 Agent。
根据以下 JSON Schema 生成 React + Vite 页面与组件：
${JSON.stringify(schema.pages)}
要求：
1) 使用 fetch/axios 调用 /api 接口
2) 与接口字段严格一致
3) 输出可运行代码
`;

  const bePrompt = `
你是服务端接口 Agent。
根据以下 JSON Schema 生成 FastAPI/Express 接口：
${JSON.stringify(schema.apis)}
数据模型：
${JSON.stringify(schema.dataModels)}
要求：
1) 路由与字段与前端一致
2) 提供最小可运行 CRUD
`;

  return { fePrompt, bePrompt };
}
```

### 6.6 联调 Agent 伪代码（对齐契约）

```typescript
async function align(feCode, beCode, schema) {
  const apiDiff = diffApiContract(feCode, beCode, schema);
  if (!apiDiff.hasIssue) return { feCode, beCode };

  const patch = await integrationAgent.generatePatch({
    feCode,
    beCode,
    issues: apiDiff.issues
  });

  return applyPatch({ feCode, beCode }, patch);
}
```

### 6.7 测试 Agent 伪代码（闭环终止条件）

```typescript
async function run(patched) {
  const typecheck = await runTypeCheck(patched);
  const lint = await runLint(patched);
  const apiSmoke = await runApiSmokeTest(patched);

  return {
    passed: typecheck.ok && lint.ok && apiSmoke.ok,
    issues: [...typecheck.issues, ...lint.issues, ...apiSmoke.issues]
  };
}
```

### 6.8 质量门禁与失败回流策略（闭环稳态）

**闭环关键点**：  
- **Schema 是唯一真相**：任何 Agent 修改都需回写 Schema 或产出对 Schema 的修正建议。  
- **失败回流**：测试失败 → 联调 Agent 优先修补 → 仍失败才回到前后端 Agent 重生成。  
- **版本化**：每一轮生成都记录为一个版本，便于对比与回滚。  
- **质量门禁**：未通过门禁的版本不发布预览链接。

**失败回流伪代码**：

```typescript
async function retryLoop(testReport, schema, patched) {
  if (testReport.onlyApiMismatch) {
    return integrationAgent.align(patched.feCode, patched.beCode, schema);
  }
  if (testReport.onlyFrontendIssue) {
    const fePrompt = buildFeFixPrompt(schema, testReport);
    return frontendAgent.generateCode(fePrompt);
  }
  if (testReport.onlyBackendIssue) {
    const bePrompt = buildBeFixPrompt(schema, testReport);
    return backendAgent.generateCode(bePrompt);
  }
  return schemaAgent.reviseSchema(schema, testReport);
}
```

### 6.9 Schema 校验与版本记录（可追溯）

```typescript
function validateSchema(schema) {
  const ok = jsonSchemaValidator.validate(SCHEMA_V1, schema);
  if (!ok) throw new Error("Schema validation failed");
  return true;
}

function saveVersion(stage, artifacts) {
  versionStore.save({
    stage,
    timestamp: Date.now(),
    artifacts
  });
}
```

### 6.10 联调对齐的核心算法（接口契约一致性）

```typescript
function diffApiContract(feCode, beCode, schema) {
  const feApis = parseFrontendApiCalls(feCode);
  const beApis = parseBackendRoutes(beCode);
  const spec = normalizeSchemaApis(schema.apis);
  const issues = compareApis(spec, feApis, beApis);
  return { hasIssue: issues.length > 0, issues };
}
```

### 6.11 编排状态机（可观测 + 可控）

```typescript
const states = [
  "schema_ready",
  "prompt_ready",
  "code_generated",
  "integration_done",
  "test_passed",
  "preview_published"
];

async function orchestrate() {
  emitState("schema_ready");
  emitState("prompt_ready");
  emitState("code_generated");
  emitState("integration_done");
  emitState("test_passed");
  emitState("preview_published");
}
```

### 6.12 状态存储与锁策略（避免并发冲突）

**目标**：多个 Agent 并行时，避免相互覆盖产物或重复生成。  
- **项目级锁**：每个项目/会话只有一个「写锁」，写入产物时必须持有。  
- **阶段级锁**：同一阶段只允许一个 Agent 执行（如联调、测试）。  
- **幂等写入**：同一任务重复执行不会造成脏数据。

```typescript
async function withProjectLock(projectId, fn) {
  const lockKey = `lock:${projectId}`;
  const acquired = await redis.set(lockKey, "1", "NX", "EX", 60);
  if (!acquired) throw new Error("Project is locked");
  try {
    return await fn();
  } finally {
    await redis.del(lockKey);
  }
}
```

### 6.13 事件模型与幂等处理（Event Bus）

**事件格式建议**：  
`{ eventId, projectId, stage, agent, status, payload, timestamp }`

```typescript
function emitEvent(event) {
  if (eventStore.exists(event.eventId)) return;
  eventStore.save(event);
  eventBus.publish(event);
}

eventBus.on("stage:done", (event) => {
  if (event.stage === "gen_frontend" && isReady("gen_backend")) {
    schedule("integrate");
  }
});
```

### 6.14 重试与降级策略（可恢复性）

- **重试次数**：每阶段 2~3 次，超过后降级或人工介入。  
- **退避策略**：指数退避 + 抖动，避免系统雪崩。  
- **降级路径**：复杂页面生成失败 → 退化为模板页；复杂接口失败 → 退化为只读接口。

```typescript
async function retryWithBackoff(task, maxRetry = 3) {
  for (let i = 0; i < maxRetry; i++) {
    try { return await task(); }
    catch (e) {
      await sleep(Math.pow(2, i) * 200 + random(0, 200));
    }
  }
  return fallbackTask();
}
```

### 6.15 跨 Agent 上下文协议（统一字段与格式）

**统一上下文字段**：  
`{ schema, prompts, frontendCode, backendCode, patches, testReport, artifacts }`

```typescript
const ContextSchema = {
  schema: "json",
  prompts: { fe: "string", be: "string" },
  frontendCode: "files",
  backendCode: "files",
  patches: "diff",
  testReport: "json",
  artifacts: "list"
};

function writeContext(ctxKey, data) {
  return contextStore.save(ctxKey, data);
}
```

---

## 7. 与现有 Agent 设计的衔接

- **规划与工具**：可把「生成前端」「生成后端」「启动预览」做成 Agent 的 **Tool**，由主 Agent 按规划依次调用（参考你现有的 `how_to_design_ai_agent.md`）。  
- **流式与可观测**：生成过程用 **SSE/WebSocket** 推送到前端，结合你 `frontend_ai_agent_design.md` 中的「思维折叠、进度槽、Artifacts 预览区」，把「当前在生成哪一页、哪一接口」展示出来。  
- **Generative UI**：若采用「组件树 JSON + 预置组件」的预览方式，与文档中的 **Generative UI** 思路一致：Agent 输出结构化数据，前端映射到 React 组件。

---

## 8. 实现路线图（建议）

1. **MVP**  
   - 固定模板：只支持「单页 CRUD」（如待办列表）。  
   - 需求 → 固定 Schema（手写解析或简单 LLM）→ 填模板生成前端 + 后端 → 本地起一个 Dev Server + 一个 API 服务 → 浏览器打开预览。

2. **增强**  
   - 引入完整「需求 → 规划」的 LLM 层，输出统一规格。  
   - 支持多页、多实体、简单路由。  
   - 预览改为「流式」：每生成一页就刷新预览或插入 iframe。

3. **生产化**  
   - 多用户/多会话隔离（目录、端口、容器）。  
   - 持久化项目、版本、回滚。  
   - 可导出源码或部署到托管环境（如 Vercel + Serverless API）。

---

## 9. 参考产品与开源

- **Bolt.new**：浏览器内全栈生成 + 预览，对接 Supabase。  
- **v0.dev (Vercel)**：自然语言生成 UI，偏前端组件。  
- **AI Website Builder (GitHub)**：FastAPI + React，从描述生成多页站 + 预览。  
- **CodeSandbox / StackBlitz**：浏览器内 Node + Dev Server，可作为「预览运行时」的参考实现。

---

## 10. 总结

- **需求侧**：用自然语言描述产品 → 通过 **Agent + 规划** 得到「页面、接口、数据模型」等结构化规格。  
- **生成侧**：按规格用 **模板 + LLM** 生成前端与后端代码，并保证 **接口契约一致**。  
- **预览侧**：在 **隔离环境** 中启动前端 Dev Server 与后端 API，通过 **反向代理 + 热更新** 在浏览器中实时看到效果。  

这样，不懂编码的人员只需描述需求并可在浏览器中实时看到生成的前后端分离项目，并可持续通过对话迭代（例如「加一个筛选」「把登录改成微信登录」），由 Agent 再次规划与生成增量代码并刷新预览。
