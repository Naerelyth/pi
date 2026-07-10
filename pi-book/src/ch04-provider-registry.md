# 第 4 章：Provider 不是 Adapter

> **定位**：本章解剖 pi-ai 的核心抽象 — 用一个 98 行的注册表统一 20+ 家 LLM 厂商。
> 前置依赖：第 2 章（分层架构）。
> 适用场景：当你想理解如何设计一个多 provider LLM 抽象层，或者想为 pi 添加新的 LLM 供应商。

## 20+ 家厂商，如何用一个接口统一？

这是本章的核心设计问题。

打开 `packages/ai/src/types.ts`，前 43 行定义了 pi 对 LLM 世界的全部认知：

```typescript
// packages/ai/src/types.ts:5-17

type KnownApi =
  | "openai-completions"
  | "mistral-conversations"
  | "openai-responses"
  | "azure-openai-responses"
  | "openai-codex-responses"
  | "anthropic-messages"
  | "bedrock-converse-stream"
  | "google-generative-ai"
  | "google-gemini-cli"
  | "google-vertex";

type Api = KnownApi | (string & {});
```

注意 `Api` 类型的设计：它是 `KnownApi`（已知的 10 种 API 协议）加上 `(string & {})`（任意字符串）的联合。这个看似奇怪的 `(string & {})` 是 TypeScript 的一个技巧 — 它让类型系统对已知值提供自动补全，同时允许任意新值。内建的 10 种协议有 IDE 提示，自定义协议可以用任意字符串注册。

`Provider` 类型列出了 23 个已知供应商，也允许扩展 — 但用的是普通的 `string` 联合而非 `(string & {})` 技巧（IDE 补全效果略弱）：

```typescript
type Provider = KnownProvider | string;
```

这里隐含了 pi 最重要的设计决策之一：**Provider 和 Api 是两个独立的维度**。

一个 provider（比如 `"google"`）可能暴露多种 api（`"google-generative-ai"` 和 `"google-vertex"`）。一种 api 协议（比如 `"openai-responses"`）可能被多个 provider 使用（`"openai"`、`"azure-openai-responses"`、`"github-copilot"`）。

如果把 provider 和 api 绑死，每增加一个 Azure OpenAI 部署就要写一个新 provider。分离之后，Azure OpenAI 只需注册一个使用 `"azure-openai-responses"` api 的 provider。

## `Model<TApi>` — 携带一切上下文的值对象

在理解注册表之前，先看它操作的核心数据：`Model`。

```typescript
// packages/ai/src/types.ts:316-338

export interface Model<TApi extends Api> {
  id: string;
  name: string;
  api: TApi;
  provider: Provider;
  baseUrl: string;
  reasoning: boolean;
  input: ("text" | "image")[];
  cost: {
    input: number;   // $/million tokens
    output: number;  // $/million tokens
    cacheRead: number;
    cacheWrite: number;
  };
  contextWindow: number;
  maxTokens: number;
  headers?: Record<string, string>;
  compat?: /* conditional type based on TApi */ ;
}
```

`Model` 不只是一个"模型名称"，它是一个**自描述的值对象**，携带了调用一个 LLM 所需的全部元信息：

- **身份**：`id`（如 `"claude-sonnet-4-20250514"`）、`name`（人类可读名称）、`provider`（`"anthropic"`）
- **协议**：`api` 字段决定了用哪种 API 协议与这个模型通信
- **能力**：`reasoning`（是否支持 extended thinking）、`input`（支持哪些输入模态）
- **约束**：`contextWindow`（上下文窗口大小）、`maxTokens`（单次最大输出）
- **经济**：`cost` 对象精确到四种计价维度 — 输入、输出、缓存读、缓存写

### 为什么 Model 是泛型的？

`Model<TApi>` 的泛型参数 `TApi extends Api` 是整个类型系统的支点。它的作用不在于 Model 本身的字段差异（大部分字段对所有 api 都一样），而在于**向下传播协议信息**。

看 `compat` 字段的条件类型：

```typescript
compat?: TApi extends "openai-completions"
  ? OpenAICompletionsCompat
  : TApi extends "openai-responses"
    ? OpenAIResponsesCompat
    : never;
```

当 `TApi` 是 `"openai-completions"` 时，`compat` 的类型是 `OpenAICompletionsCompat`（包含兼容性开关如 `supportsStreaming`）。当 `TApi` 是 `"anthropic-messages"` 时，`compat` 是 `never` — 根本不存在这个字段。

更重要的是，`Model<TApi>` 的泛型参数会传递给 `StreamFunction`：

```typescript
// packages/ai/src/types.ts:125-129

export type StreamFunction<
  TApi extends Api = Api,
  TOptions extends StreamOptions = StreamOptions
> = (
  model: Model<TApi>,
  context: Context,
  options?: TOptions,
) => AssistantMessageEventStream;
```

当一个 provider 声明自己的 stream 函数为 `StreamFunction<"anthropic-messages", AnthropicOptions>` 时，TypeScript 保证：

1. 传入的 `model` 一定是 `Model<"anthropic-messages">`，即 `model.api` 一定是 `"anthropic-messages"`
2. `options` 一定是 `AnthropicOptions`（包含 Anthropic 特有的 cache control 等选项）

这种设计让每个 provider 的实现在**类型层面就知道自己服务的是哪种协议**，不需要运行时判断。

## 98 行的注册表

`api-registry.ts` 是整个 pi-ai 层的枢纽。它只有 98 行，是极简设计的范例。完整代码如下：

```typescript
// packages/ai/src/api-registry.ts:1-38（类型定义和状态）

export type ApiStreamFunction = (
  model: Model<Api>,
  context: Context,
  options?: StreamOptions,
) => AssistantMessageEventStream;

export type ApiStreamSimpleFunction = (
  model: Model<Api>,
  context: Context,
  options?: SimpleStreamOptions,
) => AssistantMessageEventStream;

export interface ApiProvider<
  TApi extends Api = Api,
  TOptions extends StreamOptions = StreamOptions
> {
  api: TApi;
  stream: StreamFunction<TApi, TOptions>;
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>;
}

interface ApiProviderInternal {
  api: Api;
  stream: ApiStreamFunction;
  streamSimple: ApiStreamSimpleFunction;
}

type RegisteredApiProvider = {
  provider: ApiProviderInternal;
  sourceId?: string;
};

const apiProviderRegistry = new Map<string, RegisteredApiProvider>();
```

```typescript
// packages/ai/src/api-registry.ts:66-98（公共 API）

export function registerApiProvider<TApi extends Api,
  TOptions extends StreamOptions>(
  provider: ApiProvider<TApi, TOptions>,
  sourceId?: string,
): void {
  apiProviderRegistry.set(provider.api, {
    provider: {
      api: provider.api,
      stream: wrapStream(provider.api, provider.stream),
      streamSimple: wrapStreamSimple(provider.api,
        provider.streamSimple),
    },
    sourceId,
  });
}

export function getApiProvider(api: Api) {
  return apiProviderRegistry.get(api)?.provider;
}

export function getApiProviders(): ApiProviderInternal[] {
  return Array.from(apiProviderRegistry.values(),
    (entry) => entry.provider);
}

export function unregisterApiProviders(sourceId: string): void {
  for (const [api, entry] of apiProviderRegistry.entries()) {
    if (entry.sourceId === sourceId) {
      apiProviderRegistry.delete(api);
    }
  }
}

export function clearApiProviders(): void {
  apiProviderRegistry.clear();
}
```

整个注册表的 API 面只有五个函数：`registerApiProvider`（注册）、`getApiProvider`（查找单个）、`getApiProviders`（列出全部）、`unregisterApiProviders`（按 sourceId 批量注销）、`clearApiProviders`（清空，用于测试）。其中前四个是常用的。

### 为什么 `ApiProvider` 只有两个方法？

`ApiProvider` 接口只要求实现者提供 `stream` 和 `streamSimple` 两个方法。没有 `complete`、没有 `embed`、没有 `tokenCount`：

- **`stream`**：接收完整的 `StreamOptions`（包含 provider 特定选项），返回事件流
- **`streamSimple`**：接收 `SimpleStreamOptions`（统一选项 + reasoning level），返回事件流

为什么不直接用一个 `stream` 方法？因为两者的职责不同：

`stream` 是给**知道自己在做什么**的调用者用的 — 它传递 provider 特定的选项（比如 Anthropic 的 cache control、Google 的 safety settings）。类型是 `StreamFunction<TApi, TOptions>`，其中 `TOptions` 是泛型的。

`streamSimple` 是给**不关心 provider 差异**的调用者用的 — 它只传递统一选项（temperature、maxTokens、reasoning level）。循环引擎（第 8 章）用的就是 `streamSimple`。

`complete` 和 `completeSimple` 不在 provider 接口中，因为它们只是 `stream` + `await result()` 的语法糖 — 我们在下一节详述。

### `sourceId`：为动态注销准备

`registerApiProvider` 接受一个可选的 `sourceId`。这个 ID 的用途是批量注销：

```typescript
export function unregisterApiProviders(sourceId: string): void {
  for (const [api, entry] of apiProviderRegistry.entries()) {
    if (entry.sourceId === sourceId) {
      apiProviderRegistry.delete(api);
    }
  }
}
```

当一个 extension 注册了多个自定义 provider，卸载时只需 `unregisterApiProviders(extensionId)`，不需要记住注册了哪些。这是"注册 / 注销对称性"的设计模式。

### 类型擦除桥接

`registerApiProvider` 内部做了一件微妙的事：把泛型的 `StreamFunction<TApi, TOptions>` 包装成非泛型的 `ApiStreamFunction`。这是整个注册表最精巧的部分，完整代码只有两个函数：

```typescript
// packages/ai/src/api-registry.ts:42-64

function wrapStream<TApi extends Api,
  TOptions extends StreamOptions>(
  api: TApi,
  stream: StreamFunction<TApi, TOptions>,
): ApiStreamFunction {
  return (model, context, options) => {
    if (model.api !== api) {
      throw new Error(
        `Mismatched api: ${model.api} expected ${api}`);
    }
    return stream(
      model as Model<TApi>, context, options as TOptions);
  };
}

function wrapStreamSimple<TApi extends Api>(
  api: TApi,
  streamSimple: StreamFunction<TApi, SimpleStreamOptions>,
): ApiStreamSimpleFunction {
  return (model, context, options) => {
    if (model.api !== api) {
      throw new Error(
        `Mismatched api: ${model.api} expected ${api}`);
    }
    return streamSimple(model as Model<TApi>, context, options);
  };
}
```

这两个函数做了同样的事：**捕获** `api` 值，在运行时检查 `model.api !== api`，然后用 `as` 把类型"恢复"回去。

为什么需要类型擦除？因为 `Map<string, RegisteredApiProvider>` 只能存一种类型。如果 Map 的 value 类型带泛型参数（比如 `ApiProvider<TApi, TOptions>`），每个 entry 的泛型参数不同，TypeScript 无法表达"一个 Map，每个 value 的泛型参数各不相同"这种存在类型（existential type）。

解决方案是经典的"入口检查 + 内部擦除"模式：

1. **注册时**：泛型约束保证 provider 的 `stream` 函数类型与 `api` 一致
2. **存储时**：`wrapStream` 把泛型函数包装为非泛型的 `ApiStreamFunction`
3. **取出时**：`getApiProvider` 返回 `ApiProviderInternal`（非泛型），调用者拿到的函数签名丢失了 `TOptions` 信息
4. **运行时**：`model.api !== api` 检查保证不会把 Anthropic 的 model 传给 OpenAI 的 stream 函数

类型安全的边界从编译时移到了运行时，但只在**一个点**（`wrapStream`）发生。这个权衡是值得的 — 它让注册表的使用者和实现者都保持简单，只在桥接层承担一次类型转换的代价。

## `stream.ts` — 薄到透明的公共 API 层

注册表本身不暴露给最终用户。用户看到的是 `stream.ts` 导出的四个函数。整个文件只有 59 行：

```typescript
// packages/ai/src/stream.ts:1-14

import "./providers/register-builtins.js";

import { getApiProvider } from "./api-registry.js";
import type {
  Api, AssistantMessage, AssistantMessageEventStream,
  Context, Model, ProviderStreamOptions,
  SimpleStreamOptions, StreamOptions,
} from "./types.js";

function resolveApiProvider(api: Api) {
  const provider = getApiProvider(api);
  if (!provider) {
    throw new Error(
      `No API provider registered for api: ${api}`);
  }
  return provider;
}
```

第一行 `import "./providers/register-builtins.js"` 是一个**副作用导入** — 它不导入任何值，只确保 `register-builtins.ts` 被执行（我们在下一节详述）。

然后是四个公共函数，每个都是"解析 provider + 委托"的一行逻辑：

```typescript
// packages/ai/src/stream.ts:25-59

export function stream<TApi extends Api>(
  model: Model<TApi>, context: Context,
  options?: ProviderStreamOptions,
): AssistantMessageEventStream {
  const provider = resolveApiProvider(model.api);
  return provider.stream(model, context, options);
}

export async function complete<TApi extends Api>(
  model: Model<TApi>, context: Context,
  options?: ProviderStreamOptions,
): Promise<AssistantMessage> {
  const s = stream(model, context, options);
  return s.result();
}

export function streamSimple<TApi extends Api>(
  model: Model<TApi>, context: Context,
  options?: SimpleStreamOptions,
): AssistantMessageEventStream {
  const provider = resolveApiProvider(model.api);
  return provider.streamSimple(model, context, options);
}

export async function completeSimple<TApi extends Api>(
  model: Model<TApi>, context: Context,
  options?: SimpleStreamOptions,
): Promise<AssistantMessage> {
  const s = streamSimple(model, context, options);
  return s.result();
}
```

四个函数的逻辑完全对称：

| 函数 | 选项类型 | 返回值 | 实质 |
|------|---------|--------|------|
| `stream` | `ProviderStreamOptions`（provider 特定） | 事件流 | 委托 `provider.stream` |
| `streamSimple` | `SimpleStreamOptions`（统一） | 事件流 | 委托 `provider.streamSimple` |
| `complete` | `ProviderStreamOptions` | `Promise<AssistantMessage>` | `stream` + `result()` |
| `completeSimple` | `SimpleStreamOptions` | `Promise<AssistantMessage>` | `streamSimple` + `result()` |

`complete` 和 `completeSimple` 根本不是独立的实现 — 它们只是对 stream 版本调用 `.result()` 的语法糖。这就是为什么 `ApiProvider` 接口只需要两个方法而不是四个：**stream 是原语，complete 是派生**。

这个文件的存在证明了注册表设计的成功：98 行的 `api-registry.ts` 承担了全部复杂性，公共 API 层薄到几乎可以内联。对调用者来说，`stream(model, context, options)` 看起来就像在直接调用 provider，注册表完全隐形。

## 延迟加载 — `register-builtins.ts` 的启动策略

`stream.ts` 的第一行 `import "./providers/register-builtins.js"` 触发了一个精心设计的启动流程。`register-builtins.ts` 是一个 433 行的文件，但它的核心逻辑只有一个函数和一行调用：

```typescript
// packages/ai/src/providers/register-builtins.ts:366-433

export function registerBuiltInApiProviders(): void {
  registerApiProvider({
    api: "anthropic-messages",
    stream: streamAnthropic,
    streamSimple: streamSimpleAnthropic,
  });
  registerApiProvider({
    api: "openai-responses",
    stream: streamOpenAIResponses,
    streamSimple: streamSimpleOpenAIResponses,
  });
  // ... 8 more providers ...
}

registerBuiltInApiProviders(); // 模块加载时立即执行
```

但这里的 `streamAnthropic` 并不是真正的 Anthropic stream 函数 — 它是一个**延迟加载的包装器**。看它的创建方式：

```typescript
// packages/ai/src/providers/register-builtins.ts:345-346

export const streamAnthropic =
  createLazyStream(loadAnthropicProviderModule);
export const streamSimpleAnthropic =
  createLazySimpleStream(loadAnthropicProviderModule);
```

`createLazyStream` 返回一个 `StreamFunction`，但它**不会在注册时加载 provider 模块**。真正的 `import("./anthropic.js")` 只在第一次调用 `streamAnthropic()` 时发生：

```typescript
// packages/ai/src/providers/register-builtins.ts:168-187

function createLazyStream<TApi extends Api,
  TOptions extends StreamOptions,
  TSimpleOptions extends SimpleStreamOptions>(
  loadModule: () => Promise<
    LazyProviderModule<TApi, TOptions, TSimpleOptions>>,
): StreamFunction<TApi, TOptions> {
  return (model, context, options) => {
    const outer = new AssistantMessageEventStream();
    loadModule()
      .then((module) => {
        const inner = module.stream(model, context, options);
        forwardStream(outer, inner);
      })
      .catch((error) => {
        const message = createLazyLoadErrorMessage(model, error);
        outer.push({ type: "error", reason: "error", error: message });
        outer.end(message);
      });
    return outer;
  };
}
```

这个设计有三个关键特性：

**1. 同步返回，异步加载。** `createLazyStream` 返回的函数**立即**返回一个 `AssistantMessageEventStream`（`outer`）。模块加载和实际的 stream 调用在后台发生，事件通过 `forwardStream` 从内部流转发到外部流。调用者不需要知道 provider 是否已经加载。

**2. 一次加载，永久缓存。** 每个 provider 有一个 module-level 的 `Promise` 变量（如 `anthropicProviderModulePromise`）。`loadAnthropicProviderModule` 使用 `||=` 运算符确保 `import()` 只执行一次：

```typescript
// packages/ai/src/providers/register-builtins.ts:212-223

function loadAnthropicProviderModule() {
  anthropicProviderModulePromise ||=
    import("./anthropic.js").then((module) => {
      const provider = module as AnthropicProviderModule;
      return {
        stream: provider.streamAnthropic,
        streamSimple: provider.streamSimpleAnthropic,
      };
    });
  return anthropicProviderModulePromise;
}
```

第一次调用触发 `import()`，后续调用直接返回同一个 Promise。即使多个并发请求同时到达，它们也会共享同一个 Promise，不会重复加载。

**3. 加载失败不会崩溃。** 如果某个 provider 的模块加载失败（比如缺少 native 依赖），错误会被编码为一个带 `stopReason: "error"` 的 `AssistantMessage`，通过事件流返回给调用者。整个系统不会因为一个 provider 的加载失败而崩溃。

### 为什么不在启动时全部加载？

pi 支持 10 种内建 API 协议。如果在启动时 eager load 所有 provider 模块，会引入大量不需要的代码（每个 provider 模块都依赖对应厂商的 SDK）。一个只用 Anthropic 的用户不需要加载 Google Vertex 的 SDK；一个只用 OpenAI 的用户不需要加载 AWS Bedrock 的 SDK。

延迟加载的代价是第一次调用某个 provider 时有一个微小的延迟（模块加载的时间）。但这个代价只发生一次，而且在 agent 场景下，第一次 LLM 调用的网络延迟远大于模块加载延迟。

## 实战：添加一个新 provider 的完整步骤

为了让注册表的设计不停留在抽象层面，我们用一个具体例子走一遍完整流程。假设你要为 pi 添加一个新的 LLM 供应商 — 比如 DeepSeek，它使用 OpenAI 兼容的 API。

**第一步：确定 Api 协议。** DeepSeek 兼容 OpenAI 的 responses API，所以你不需要创建新的 api 类型。直接复用 `"openai-responses"`。这是 Provider 和 Api 分离带来的第一个好处 — 只要协议兼容，新 provider 就是零 api 代码。

**第二步：在 `KnownProvider` 中添加名称。** 在 `types.ts` 的 `KnownProvider` 联合中加入 `"deepseek"`。这不是必须的（`Provider = KnownProvider | string`，任意字符串都合法），但加入后 IDE 会提供自动补全。

**第三步：定义 Model 对象。** 在模型目录（model catalog）中添加 DeepSeek 的模型定义：

```typescript
const deepseekR1: Model<"openai-responses"> = {
  id: "deepseek-reasoner",
  name: "DeepSeek R1",
  api: "openai-responses",
  provider: "deepseek",
  baseUrl: "https://api.deepseek.com/v1",
  reasoning: true,
  input: ["text"],
  cost: { input: 0.55, output: 2.19,
    cacheRead: 0.14, cacheWrite: 0.55 },
  contextWindow: 64000,
  maxTokens: 8192,
};
```

注意 `api: "openai-responses"` — 这告诉注册表用 OpenAI responses 协议与 DeepSeek 通信。

**第四步：结束。** 没有第四步。

因为 `"openai-responses"` 这个 api 已经在 `register-builtins.ts` 中注册了对应的 `stream` 和 `streamSimple` 实现，任何声明 `api: "openai-responses"` 的 model 都可以直接使用。用户调用 `stream(deepseekR1, context)` 时，注册表按 `model.api` 查找到 OpenAI responses 的 provider，然后将 `model.baseUrl`（`https://api.deepseek.com/v1`）传递给它。

如果 DeepSeek 使用的是**完全不兼容**的私有协议，那么你需要：

1. 在 `KnownApi` 中添加 `"deepseek-native"`（或用任意字符串，不修改 KnownApi）
2. 创建 `providers/deepseek.ts`，实现 `stream` 和 `streamSimple` 两个函数
3. 在 `register-builtins.ts` 中添加延迟加载逻辑（约 20 行样板代码）
4. 调用 `registerApiProvider({ api: "deepseek-native", stream, streamSimple })`

即使是这种最复杂的情况，修改 `api-registry.ts` 的代码量也是 **零行**。注册表的代码从引入以来没有因为增加 provider 而改变过。这正是注册表模式的设计目标：**核心不变，边缘生长**。

## 取舍分析

### 得到了什么

**1. 无限扩展性**。任何人都可以在运行时注册新的 provider，不需要修改 pi-ai 的代码。Extension 可以在用户启动后动态加载 provider。

**2. Provider 和 Api 的解耦**。同一个 api 协议可以被多个 provider 复用。增加 Azure OpenAI 或 GitHub Copilot 不需要重写 OpenAI 的 api 实现。

**3. 极简的公共 API**。注册表只暴露 5 个函数。用户面对的 `stream.ts` 只有 4 个函数。新的 provider 开发者只需要实现 `stream` 和 `streamSimple` 两个方法。

**4. 启动零成本**。延迟加载确保了只有实际使用的 provider 模块才会被加载。10 个内建 provider 中，一次会话通常只加载 1-2 个。

**5. 复杂性集中**。整个 pi-ai 层的"设计复杂性"集中在 98 行的 `api-registry.ts` 中。`stream.ts` 是纯粹的委托；`register-builtins.ts` 是纯粹的样板。理解了注册表，就理解了一切。

### 放弃了什么

**1. 静态分析能力**。因为 provider 是运行时注册的，编译器不知道哪些 provider 可用。你不能写 `if (api === "openai-responses")` 然后让编译器保证这个 provider 一定存在。

**2. 类型擦除的代价**。存储时的类型擦除意味着 `getApiProvider` 返回的是非泛型的 `ApiProviderInternal`。调用者拿到的函数签名丢失了 `TOptions` 的具体类型信息。在 `stream.ts` 中，`options` 被类型为 `ProviderStreamOptions`（即 `StreamOptions & Record<string, unknown>`），这是一个"什么都可以传"的类型 — provider 特定的选项在类型层面不受检查。

**3. 运行时才知道 provider 是否可用**。如果用户配置了一个尚未注册的 provider，错误只会在第一次调用 `stream` 时发生（`resolveApiProvider` 抛出 `No API provider registered for api`），而不是在配置时。

**4. 延迟加载的第一次调用延迟**。虽然通常可以忽略，但在极端场景下（冷启动 + 第一次调用 + provider 模块很大），用户可能感受到一个短暂的停顿。

对于 pi 的定位 — 支持 20+ 家厂商且允许用户扩展 — 运行时注册是唯一可行的选择。静态注册意味着每增加一个 provider 就要改核心代码、重新编译、重新发布。注册表用 98 行代码买到了无限扩展性，这笔交易划算。

---

### 版本演化说明
> 本章核心分析基于 pi-mono v0.66.0。`api-registry.ts` 自引入以来结构保持稳定。
> `sourceId` 机制是后来为支持 extension 动态注册/注销而添加的。
> `KnownApi` 和 `KnownProvider` 的列表会随新 provider 的加入而增长，但注册表的设计不变。
> 延迟加载模式（`createLazyStream`）在早期版本中不存在，是随着 provider 数量增长到 10+ 后引入的性能优化。
