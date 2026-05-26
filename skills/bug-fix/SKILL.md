---
name: bug-fix
description: 故障修复流程。用于修复用户报告的故障（bug）：(1) 编写测试用例复现故障，(2) 分析原因并修复，(3) 运行全量测试验证。覆盖后端 Rust 与前端 React/TypeScript 双栈。
---

# Bug Fix 故障修复流程

本 skill 定义了系统化的故障修复流程，确保每次修复都可测试、可验证、不引入回归。

项目包含两个技术栈：
- **后端**：Rust（`crates/sprite/`）
- **前端**：React + TypeScript + Vite（`web-client/`）

## 工作流程

```
┌─────────────────┐
│  1. 复现故障    │ ← 编写测试用例，确保故障现象可重现
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  2. 分析并修复  │ ← 定位根因，编写修复代码
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  3. 验证修复    │ ← 运行全量测试，确保无回归
└─────────────────┘
```

## 步骤 1: 复现故障

**目标**：编写一个测试用例，精确复现用户报告的故障现象。

### 1.1 理解故障现象

- 仔细阅读用户提供的 bug 报告或错误描述
- 确认故障涉及的模块、输入条件和预期/实际输出
- **判断技术栈**：是后端 Rust 代码、前端 React/TypeScript 代码，还是两者交互问题
- **判断测试范围**：单元测试 / 集成测试 / 组件测试 / Hooks 测试

### 1.2 编写复现测试

> **详细测试规范请参考对应技能**：
> - 后端 Rust 测试 → `rust-tests-guidelines`
> - 前端 web-client 测试 → `web-client-tests-guidelines`

#### 后端（Rust）

**选择测试位置**：
- **单元测试**：模块内部逻辑错误，在源文件的 `#[cfg(test)]` 模块中添加
- **集成测试**：跨模块协作或公开 API 问题，在 `tests/` 目录添加新测试

**核心原则（摘自 `rust-tests-guidelines`）**：
- 所有文件操作必须使用 `testdir!()` 宏，**禁止**使用真实文件系统
- 异步测试使用 `#[tokio::test]`，禁止在同步测试中 `block_on`
- 集成测试仅 mock 外部 HTTP 服务（`wiremock`）；单元测试通过注入 `MockLlmProvider` mock LLM 层
- 断言优先使用具体断言（`assert_eq!`、`assert!(..., "msg")`），避免模糊的 `assert!(result.is_ok())`

**最小示例**：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_bug_<short_description>() {
        let input = /* 复现条件 */;
        let result = /* 调用函数 */;
        assert!(/* 期望的正确行为 */);
    }
}
```

```rust
#[tokio::test]
async fn bug_<component>_<scenario>() {
    let temp_dir = testdir!();
    // ... 构造复现场景
    assert!(/* 期望的正确行为 */);
}
```

#### 前端（web-client）

**选择测试位置**：
- **纯函数 / API 层**：`src/<模块>/__tests__/*.test.ts`
- **Hooks 测试**：`src/hooks/__tests__/*.test.tsx`
- **组件测试**：`src/components/__tests__/*.test.tsx`

**核心原则（摘自 `web-client-tests-guidelines`）**：
- 纯函数测试不 Mock，直接调用；一个 `describe` 对应一个被测函数
- Hooks 测试使用 `renderHook(() => useHook())` + `act()`；每个 `beforeEach` 执行 `vi.resetAllMocks()`
- 组件测试使用 `render()` + `screen` 查询 + `@testing-library/user-event` 交互
- API 层 Mock `global.fetch`（`vi.stubGlobal`），WebSocket Mock `global.WebSocket`
- `vi.mock()` 创建的模块级 mock 是单例，注意 `mockResolvedValueOnce` 队列残留问题

**最小示例**：

```ts
// 纯函数单元测试
import { describe, it, expect } from 'vitest'
import { streamingEventsToMessages } from '../streaming'

describe('streamingEventsToMessages()', () => {
  it('空事件数组应返回空数组', () => {
    expect(streamingEventsToMessages([])).toEqual([])
  })
})
```

```ts
// Hook 测试
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { renderHook, act } from '@testing-library/react'
import { useChat } from '../useChat'

vi.mock('../../api/client')

describe('useChat', () => {
  beforeEach(() => {
    vi.resetAllMocks()
  })

  it('bug: agentName 变化时应重置 currentSession', async () => {
    const { result, rerender } = renderHook(
      ({ name }) => useChat(name),
      { initialProps: { name: 'agent1' } }
    )
    await act(async () => {
      await result.current.selectSession({ id: 's1', created_at: '', message_count: 0 })
    })
    expect(result.current.currentSession).not.toBeNull()

    rerender({ name: 'agent2' })
    expect(result.current.currentSession).toBeNull()
  })
})
```

```tsx
// 组件测试
import { describe, it, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { MessageInput } from '../MessageInput'

describe('MessageInput', () => {
  it('bug: 回车应触发发送', async () => {
    const onSend = vi.fn()
    render(<MessageInput onSend={onSend} disabled={false} />)
    const textarea = screen.getByPlaceholderText('输入消息...')
    await userEvent.type(textarea, 'hi')
    await userEvent.keyboard('{Enter}')
    expect(onSend).toHaveBeenCalledWith('hi')
  })
})
```

### 1.3 运行测试确认故障

**后端**：

```bash
# 运行新编写的测试，确认故障可复现
cargo test <test_name> -- --nocapture

# 或运行整个测试模块
cargo test <module_name>
```

**前端**：

```bash
cd web-client

# 运行单个测试文件
npx vitest run src/hooks/__tests__/useChat.test.tsx

# 或运行匹配名称的测试
npx vitest run -t "bug:"
```

**预期结果**：测试应该**失败**，失败信息与用户报告一致。

**检查点**：
- [ ] 测试能够稳定复现故障
- [ ] 测试失败信息清晰描述问题
- [ ] 测试覆盖了用户报告的场景

## 步骤 2: 分析并修复

**目标**：定位根因，编写最小化修复代码，使步骤 1 的测试通过。

### 2.1 分析根因

- 阅读测试失败输出和堆栈跟踪
- **后端**：追踪相关代码执行路径，使用 `dbg!()` 或 `println!` 临时定位问题
- **前端**：使用 `console.log` 或浏览器 DevTools 定位问题；对于 Hooks 可用 `@testing-library/react` 的 `renderHook` + `act` 调试状态流转
- 识别是逻辑错误、边界条件未处理、还是资源管理问题

### 2.2 编写修复代码

**原则**：
- **最小修改**：仅修改必要的代码，避免大面积重构
- **保持语义**：修复不应改变 API 的语义或破坏现有契约
- **添加注释**：如 bug 复杂，在修复处添加注释说明原因

```rust
// 后端修复示例
fn process_data(input: &str) -> Result<String> {
    // 修复：处理空字符串情况，避免 panic
    if input.is_empty() {
        return Err(anyhow!("input cannot be empty"));
    }
    Ok(input.to_uppercase())
}
```

```ts
// 前端修复示例
useEffect(() => {
  // 修复：agentName 变化时重置状态和连接，防止用旧 session 连接新 agent 导致 404
  setCurrentSession(null);
  setMessages([]);
  setStreamingEvents([]);
  setError(null);
  if (streamClientRef.current) {
    streamClientRef.current.disconnect();
    streamClientRef.current = null;
  }
}, [agentName]);
```

### 2.3 验证修复

**后端**：

```bash
# 运行步骤1编写的测试，确认现在通过
cargo test <test_name_from_step_1>
```

**前端**：

```bash
cd web-client
npx vitest run <test_file_path>
```

**预期结果**：测试应该**通过**。

**检查点**：
- [ ] 步骤 1 的测试现在通过
- [ ] 修复代码简洁、无多余更改
- [ ] 如需要，添加了边界条件测试

## 步骤 3: 验证全量测试

**目标**：运行全部测试，确保修复未引入回归。

### 3.1 运行全量测试

**后端**：

```bash
# 运行所有测试（单元测试 + 集成测试）
cargo test

# 如果测试较多，可串行运行以避免并发干扰
cargo test -- --test-threads=1

# 查看详细输出（调试用）
cargo test -- --nocapture
```

**前端**：

```bash
cd web-client

# 运行所有前端测试
npm run test:run

# 运行 TypeScript 类型检查（构建前的关键检查）
npx tsc -b
```

### 3.2 处理失败的测试

如果有测试失败：
1. **分析失败原因**：是修复引入的 bug，还是测试本身有问题？
2. **如是修复引入**：重新评估修复方案，可能需要不同的修复策略
3. **如是测试问题**：更新测试以匹配正确行为

### 3.3 最终确认

**后端**：

```bash
# 确认所有测试通过
cargo test

# 检查代码质量
cargo clippy

# 检查代码格式
cargo fmt -- --check
```

**前端**：

```bash
cd web-client

# 确认所有测试通过
npm run test:run

# 确认 TypeScript 类型检查通过且生产构建成功
npm run build
```

**检查点**：
- [ ] 所有测试通过（后端 + 前端）
- [ ] 无 clippy 警告（或仅有既有的警告）
- [ ] 前端 `npm run build` 构建成功
- [ ] 代码格式正确（`cargo fmt`、Prettier 等）

## 完整示例

### 场景 A：后端 — 用户报告 `parse_config("")` 导致 panic

#### 步骤 1: 复现

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]  // 当前期望 panic，修复后移除
    fn test_bug_parse_config_empty_string_panics() {
        parse_config("");
    }
}
```

运行测试确认故障：
```bash
cargo test test_bug_parse_config_empty_string_panics
```

#### 步骤 2: 修复

```rust
pub fn parse_config(input: &str) -> Result<AppConfig> {
    if input.is_empty() {
        return Err(anyhow!("config input cannot be empty"));
    }
    toml::from_str(input).map_err(Into::into)
}
```

更新测试（移除 `should_panic`）：
```rust
#[test]
fn test_parse_config_empty_string_returns_error() {
    let result = parse_config("");
    assert!(result.is_err());
    assert!(result.unwrap_err().to_string().contains("empty"));
}
```

#### 步骤 3: 验证

```bash
cargo test
cargo clippy
cargo fmt -- --check
```

### 场景 B：前端 — 切换 Agent 时 WebSocket 连接报错

#### 步骤 1: 复现

```tsx
it('bug: agentName 变化时应重置 currentSession 并断开旧连接', async () => {
  vi.mocked(api.getSessionMessages).mockResolvedValue([])
  const { result, rerender } = renderHook(
    ({ name }) => useChat(name),
    { initialProps: { name: 'agent1' } }
  )

  await act(async () => {
    await result.current.selectSession(mockSession)
  })
  expect(result.current.currentSession).toEqual(mockSession)

  rerender({ name: 'agent2' })

  expect(result.current.currentSession).toBeNull()
  expect(result.current.messages).toEqual([])
  expect(mockStreamClient.disconnect).toHaveBeenCalled()
})
```

运行测试确认故障：
```bash
cd web-client
npx vitest run src/hooks/__tests__/useChat.test.tsx
```

#### 步骤 2: 修复

```ts
useEffect(() => {
  // 修复：agentName 变化时重置状态和连接，防止用旧 session 连接新 agent 导致 404
  setCurrentSession(null);
  setMessages([]);
  setStreamingEvents([]);
  setError(null);
  if (streamClientRef.current) {
    streamClientRef.current.disconnect();
    streamClientRef.current = null;
  }
}, [agentName]);
```

#### 步骤 3: 验证

```bash
cd web-client
npm run test:run
npm run build
```

## 测试命名约定

### 后端（Rust）

详细规范见 `rust-tests-guidelines`。

```rust
// 单元测试
test_bug_<function>_<short_description>

// 集成测试
bug_<component>_<scenario>_<expected>
```

示例：
- `test_bug_parse_config_empty_string_panics`
- `bug_server_websocket_empty_message_crashes`
- `bug_agent_tool_call_timeout_returns_error`

### 前端（TypeScript / Vitest）

详细规范见 `web-client-tests-guidelines`。

```ts
// 纯函数 / 单元测试
it('bug: <function> <short_description>')

// Hook / 组件测试
it('bug: <scenario> <expected>')
```

示例：
- `it('bug: streamingEventsToMessages empty tool_calls should not crash')`
- `it('bug: useChat agentName change should reset currentSession')`
- `it('bug: MessageInput enter should trigger send when not disabled')`

## 提交信息格式

修复完成后，commit message 应包含：

```
fix(<module>): <brief description>

- Describe the bug and its impact
- Explain the fix approach
- Add test case to prevent regression

Fixes: #<issue_number>  (if applicable)
```

## 常见陷阱

### ❌ 不要跳过步骤 1

```
// ❌ 错误：直接修复，没有先写测试
// 用户报告 bug -> 直接修改代码 -> 推送
// 问题：无法确认修复是否有效，可能引入回归

// ✅ 正确：先写失败的测试
// 用户报告 bug -> 写测试复现 -> 确认测试失败 -> 修复 -> 测试通过
```

### ❌ 不要过度修复

```rust
// ❌ 错误：修复 bug 时重构了大量无关代码
fn fixed_function() {
    // 重写了整个函数...
    // 改变了 API...
    // 添加了新功能...
}

// ✅ 正确：最小化修复
fn fixed_function() {
    // 仅添加必要的检查
    if problematic_condition {
        handle_edge_case();
    }
    // 保持原有逻辑不变
}
```

### ❌ 不要遗漏前端构建检查

```
// ❌ 错误：只运行了 vitest，没运行 npm run build
// vitest 不检查 TypeScript 类型，tsc -b 才会
// 可能导致合并后 npm run build 失败

// ✅ 正确：修复后同时验证测试和构建
npm run test:run
npm run build
```

### ❌ 不要违反测试规范

> 后端 Rust 测试的详细规范见 `rust-tests-guidelines`：
> - 不要使用 `tempfile`，必须使用 `testdir!()`
> - 不要 mock 内部模块，仅 mock 外部服务
> - 不要在同步测试中 `block_on`

> 前端 web-client 测试的详细规范见 `web-client-tests-guidelines`：
> - 注意 `mockResolvedValueOnce` 队列残留，每个 `beforeEach` 执行 `vi.resetAllMocks()`
> - `vi.useFakeTimers()` 是全局的，注意并发测试影响
> - 当代码执行 `new WebSocket(url)` 时，mock 需使用普通函数表达式 `vi.fn(function () { ... })`

## 检查清单

完成修复前确认：

- [ ] 步骤 1：编写了复现测试
- [ ] 步骤 1：测试在修复前失败
- [ ] 步骤 2：修复使复现测试通过
- [ ] 步骤 2：修复代码最小化、无多余更改
- [ ] 步骤 3：后端全量测试通过（`cargo test`），无回归
- [ ] 步骤 3：前端全量测试通过（`npm run test:run`），无回归
- [ ] 步骤 3：前端 TypeScript 类型检查通过（`npm run build` 或 `npx tsc -b`）
- [ ] 后端代码通过 `cargo clippy` 检查
- [ ] 后端代码通过 `cargo fmt` 格式化
- [ ] Commit message 清晰描述修复内容
