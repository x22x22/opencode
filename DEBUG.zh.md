# opencode --help 无输出问题分析

> 本文分析 `opencode --help` 执行后没有任何输出的可能原因，并介绍如何查看日志及让日志输出到控制台。

---

## 一、`opencode --help` 无输出的原因分析

### 1.1 启动器脚本的工作机制

执行 `opencode` 命令时，实际上首先运行的是一个 Node.js 包装脚本（`packages/opencode/bin/opencode`）。该脚本的职责是**定位并启动平台原生二进制文件**，按以下顺序查找：

1. 若设置了环境变量 `OPENCODE_BIN_PATH`，直接使用该路径指向的二进制；
2. 在脚本所在目录检查是否存在 `.opencode` 缓存二进制；
3. 向上遍历 `node_modules` 目录，寻找对应平台的预编译二进制包（如 `opencode-linux-x64`、`opencode-darwin-arm64` 等）。

若找不到任何可用的二进制，脚本会打印错误并退出：

```
It seems that your package manager failed to install the right version of the opencode CLI for your platform.
```

若上述查找过程中发生其他异常（例如文件系统错误），则可能**静默失败、无任何输出**。

### 1.2 异步中间件可能阻塞帮助输出

主入口文件（`packages/opencode/src/index.ts`）使用 yargs 18 构建 CLI。在 yargs 中，`.middleware()` 注册的回调**会在命令处理前执行，包括 `--help` 调用时**。项目中的中间件是一个异步函数，主要做两件事：

1. **初始化日志系统**（`Log.init()`）；
2. **首次运行数据库迁移**：若检测到 `opencode.db` 不存在，则运行 `JsonMigration.run()`（可能耗时数分钟）。

若数据库迁移过程中出现异常或卡死，`--help` 的输出将被阻塞或永远不会显示。

### 1.3 帮助内容输出到 stdout，而 UI 输出到 stderr

项目中所有 `UI.*` 方法均调用 `process.stderr.write()`，日志默认写入**日志文件**（不是控制台）。

yargs 本身默认将帮助文本输出到 **stdout**。因此，如果终端环境中 stdout 被重定向（例如 `opencode --help | less` 或 `opencode --help > /dev/null`），帮助内容将不会显示在终端上。

### 1.4 ANSI 转义码导致 logo 不可见

帮助的使用说明（`.usage()`）包含了 ASCII art logo（`UI.logo()`），其中大量使用 ANSI 转义码来实现彩色渲染：

```
\x1b[48;5;235m  \x1b[0m  \x1b[38;5;235m▀ ...
```

在不支持 ANSI 的终端（如某些 CI 环境、Windows CMD 不开启 VT 模式、或终端模拟器配置错误）中，这些转义码虽然不会导致"无输出"，但会让视觉上显得凌乱或"空白"。

### 1.5 小结：最常见的原因

| 原因 | 说明 |
|------|------|
| 原生二进制未找到 | 包管理器未安装平台对应的二进制包 |
| 首次数据库迁移卡死 | 中间件挂起导致 `--help` 输出被阻塞 |
| stdout 被重定向 | 帮助文本被管道或重定向丢弃 |
| ANSI 不支持 | Logo 渲染乱码，看起来像无输出 |

---

## 二、日志系统介绍

### 2.1 日志的默认行为

日志系统定义在 `packages/opencode/src/util/log.ts`。默认行为：

- **未调用 `Log.init()`（极少发生）**：日志写入 `process.stderr`；
- **调用 `Log.init()` 且未传入 `print: true`**：日志写入**文件**，不输出到终端；
- **调用 `Log.init({ print: true })`**（即使用 `--print-logs` 参数）：日志输出到 `process.stderr`。

默认日志级别：
- 正式发布版本：`INFO`
- 本地开发模式（`CHANNEL === "local"`）：`DEBUG`

### 2.2 日志文件位置

项目遵循 [XDG 基础目录规范](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)，日志存储在数据目录下：

```
$XDG_DATA_HOME/opencode/log/
```

各平台默认路径如下：

| 平台 | 路径 |
|------|------|
| **Linux** | `~/.local/share/opencode/log/` |
| **macOS** | `~/Library/Application Support/opencode/log/` |
| **Windows** | `%LOCALAPPDATA%\opencode\log\` |

### 2.3 日志文件命名规则

- **正式运行**：每次启动生成一个以时间戳命名的日志文件，格式为 `YYYY-MM-DDTHHmmss.log`，例如：
  ```
  2024-03-20T142530.log
  ```
- **本地开发模式**：固定文件名 `dev.log`（每次启动会截断后重新写入）。
- **自动清理**：日志目录超过 10 个文件时，自动删除最旧的文件，**最多保留最新的 10 个**。

### 2.4 快速查看日志路径

使用内置的调试命令查看所有全局路径：

```bash
opencode debug paths
```

输出示例（Linux）：

```
home       /home/yourname
data       /home/yourname/.local/share/opencode
bin        /home/yourname/.cache/opencode/bin
log        /home/yourname/.local/share/opencode/log
cache      /home/yourname/.cache/opencode
config     /home/yourname/.config/opencode
state      /home/yourname/.local/state/opencode
```

---

## 三、如何让日志输出到控制台

### 3.1 使用 `--print-logs` 参数

最直接的方式：将日志输出到 `stderr`（而非写入文件）。

```bash
opencode --print-logs
opencode --print-logs --help
opencode --print-logs run "请帮我重构这个函数"
```

### 3.2 使用 `--log-level` 控制详细程度

```bash
# 只显示 DEBUG 级别及以上的日志（最详细）
opencode --print-logs --log-level DEBUG

# 只显示 ERROR 级别（最简洁）
opencode --print-logs --log-level ERROR
```

可用级别（从低到高）：`DEBUG` → `INFO` → `WARN` → `ERROR`

### 3.3 直接读取日志文件

先查看日志目录路径（以 Linux 为例）：

```bash
ls ~/.local/share/opencode/log/
```

查看最新日志：

```bash
# 查看最新日志文件（Linux/macOS）
tail -f $(ls -t ~/.local/share/opencode/log/*.log | head -1)

# 或实时跟踪
ls -lt ~/.local/share/opencode/log/
```

### 3.4 使用环境变量指定二进制路径

若怀疑是二进制找不到导致静默失败，可手动指定：

```bash
OPENCODE_BIN_PATH=/path/to/opencode-binary opencode --help
```

---

## 四、其他调试方法

### 4.1 `opencode debug` 子命令

```bash
# 查看全局路径
opencode debug paths

# 查看当前解析后的配置
opencode debug config

# 查看 LSP 状态
opencode debug lsp
```

### 4.2 相关环境变量

| 环境变量 | 作用 |
|----------|------|
| `OPENCODE_BIN_PATH` | 手动指定原生二进制文件路径 |
| `OPENCODE_SKIP_MIGRATIONS` | 跳过数据库迁移（设为 `1`），避免首次运行卡死 |
| `OPENCODE_CONFIG` | 指定配置文件路径 |
| `OPENCODE_CONFIG_DIR` | 指定配置目录路径 |

### 4.3 检查进程输出流

若仍无法看到 `--help` 输出，检查 stdout 是否被重定向：

```bash
# 确保不重定向，直接在终端执行
opencode --help 2>&1
```

`2>&1` 将 stderr 合并到 stdout，可以同时看到所有输出（包括日志和错误信息）。

---

## 五、问题诊断流程图

```
opencode --help 无输出
        │
        ├─► 是否有任何错误信息？
        │       │
        │       ├─ 有 "failed to install the right version" →
        │       │   重新安装对应平台包管理器的 opencode 包
        │       │
        │       └─ 无任何输出 →
        │           尝试 `opencode --print-logs --help`
        │
        ├─► `--print-logs` 有输出？
        │       │
        │       ├─ 有 → 日志/错误信息中查找原因
        │       │
        │       └─ 无 → 检查 stdout 是否被重定向：
        │               `opencode --help 2>&1`
        │
        └─► 查看日志文件
                `ls ~/.local/share/opencode/log/`
                `tail -f <最新日志文件>`
```
