# 更新日志 (Changelog)

本项目所有重要变更记录于此文件。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/),版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

---

## [0.1.16] - 2026-07-02

### 修复 (Fixed)

修复 `jlc-bridge` 扩展中 3 个导致原理图读取功能实际不可用的 bug。upstream 代码由 AI(opus4.6)生成,含若干幻觉方法调用,经端到端实测定位并修复。

#### 1. 硬编码路径在真实机器上不存在

```typescript
// 修复前：路径写死成任何真实用户机器都不存在的目录
const BRIDGE_DIR = 'C:\\Users\\0\\.openclaw\\workspace\\jlc-bridge';
```

文件轮询兜底通路写入一个不存在的目录,Windows 需用 `mklink /J` 建 junction 绕过,Linux/macOS 完全无法使用。

```typescript
// 修复后：优先读环境变量,否则回退到当前用户主目录,跨平台可写
function resolveBridgeDir(): string {
  // JLC_BRIDGE_DIR > OPENCLAW_HOME > ~/.jlc-bridge
}
const BRIDGE_DIR = resolveBridgeDir();
```

#### 2. 原理图元件库 UUID 调用了不存在的幻觉方法

```typescript
// 修复前：这 4 个 getState_* 方法在 @jlceda/pro-api-types 0.1.175 中均不存在
libraryUuid: r?.getState_LibraryUuid?.() || r?.getState_ComponentLibraryUuid?.() || '',
uuid:        r?.getState_Uuid?.() || r?.getState_ComponentUuid?.() || '',
```

opus4.6 编造的方法名导致原理图元件的 `libraryUuid` / `uuid` 字段永远为空。

```typescript
// 修复后：调用真实存在的 getState_Component(),它返回 { libraryUuid, uuid, name }
const comp = safeGet(r, 'getState_Component');  // { libraryUuid, uuid, name }
```

#### 3. `.map()` 无错误保护导致 `sch_get_state` 卡死不返回

原代码用 `.map()` 批量取值,任一 getter 抛异常就会让整个 Promise reject。bridge 收到 `get_schematic_state` 命令后**永远不返回结果**(表现为 MCP 调用 30 秒超时,relay 日志只见 `mcp -> bridge` 不见 `bridge -> mcp`)。

```typescript
// 修复后：for 循环逐元件处理 + safeGet() 包装,单字段失败只跳过不影响整体
const safeGet = (obj, fn) => {
  try { return typeof obj?.[fn] === 'function' ? obj[fn]() : undefined; }
  catch { return undefined; }
};
for (const r of rows) {
  try { components.push({ ... }); }   // 单个元件失败只 skip
  catch { /* skip one bad component */ }
}
```

### 实测验证 (Verified)

环境:Windows 11 + Node v22.18.0 + 嘉立创 EDA 专业版 + 89 元件原理图。

| 测试项 | 结果 |
|--------|------|
| `sch_get_state` 端到端 | ✅ 30 秒内正常返回(修复前卡死超时) |
| 元件 `designator` / `name` | ✅ 完整非空(D1/C1/C7/C10 等 51 个真实元件) |
| 元件 `libraryUuid` / `uuid` | ✅ 完整非空(修复前全空) |
| 元件坐标 / 旋转 / 库符号名 | ✅ 完整返回 |
| 导线 (wires) | ✅ 150 根 |
| 电源符号 (GND/VCC) | ✅ 38 个,无位号属正常 |

返回元件示例:
```
D1   name=power   符号=LED0    (460, 810)
C1   name=10uF    符号=CAP     (260, 795)
C7   name=1uF     符号=CAP_1   (565, 315)
```

### 变更文件

- `jlc-bridge/src/index.ts` — 3 处修复(+76 / -25 行)
- `jlc-bridge/extension.json` — 版本号 0.1.13 → 0.1.16

---

## [0.1.13] - 2026-03

### 新增 (Added)

- 阻抗计算工具 `calc_impedance`(微带线/带状线/差分,支持反算线宽)
- 走线宽度计算工具 `calc_trace_width`(IPC-2221 标准)
- 轻量级 PCB Agent 工具 `pcb_agent`(基于 Anthropic Claude tool-use 循环,可自主编排多步操作)

### 文档 (Docs)

- 完善 README,补充架构图、工具清单、环境变量表、使用示例

---

## [0.1.12] - 2026-03

### 新增 (Added)

- `jlc-bridge` 嘉立创 EDA 扩展插件源码(运行在 EDA 内部,执行实际 PCB/原理图操作)
- 打包脚本,生成 `.eext` / `.lcex` 安装包

---

## [0.1.11] - 2026-02

### 变更 (Changed)

- 移除后端服务类工具,精简为 28 个纯 bridge MCP 工具
- 工具粒度重新划分,聚焦元件/走线/铺铜/DRC 等核心 PCB 操作

---

## [0.1.0] - 2026-02-25

### 新增 (Added)

- 初始版本发布
- 32 个 PCB 自动化工具(状态查询、元件操作、走线/过孔、铺铜/禁布区、丝印、高级约束、原理图)
- 基于 `@modelcontextprotocol/sdk` 的 MCP Server(stdio transport)
- WebSocket 客户端连接 gateway `/ws/bridge` 端点

---

## 版本号说明

- `jlc-bridge/extension.json` 的 `version` 字段对应 EDA 扩展版本
- 修改扩展代码后**必须递增版本号**,否则嘉立创 EDA 会使用缓存不重新加载
- 历史版本号 0.1.13 之前的变更信息有限,以 git 提交记录为准
