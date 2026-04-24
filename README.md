# Code Reviewer - 使用指南

一个 AI 驱动的代码审查 Skill，支持 Android / iOS / 通用项目，自动检测平台，生成带严重度分级的审查报告。

## 快速开始

### 最简单的用法

```
帮我 review 一下
```

就这一句话。Skill 会自动：
1. 检测当前目录是 git 仓库
2. 识别平台（Android / iOS / 通用）
3. 对比未提交的变更（staged + unstaged）
4. 输出 P0 / P1 / P2 分级的审查结果

### 指定 commit

```
review abc1234
```

```
review 最近3次提交
```

```
review feature-branch
```

## 审查范围

| 你说的话 | 审查范围 |
|---------|---------|
| `review` / `帮我看看` | 未提交的所有变更 |
| `review staged` / `review 暂存的` | 仅暂存区 |
| `review abc1234` | 单个 commit |
| `review abc1234..def5678` | commit 范围 |
| `review branch feature-x` | 分支 vs main 的差异 |
| `review last 5 commits` / `review 最近5次提交` | 最近 N 次提交 |

## 三种模式

### 标准模式（默认）

```
review
```

完整审查，报告所有 P0 / P1 / P2 问题。

### 快速模式

```
quick review
快速看看
```

只报 P0（必须修复的问题），快速回答"能不能合并"。

### 安全模式

```
security review
安全审查
```

专注安全漏洞：注入、权限绕过、密钥泄露、不安全的反序列化等。忽略代码风格。

## 严重度说明

| 级别 | 含义 | 举例 |
|------|------|------|
| **P0** | 必须修复 | Crash、数据丢失、安全漏洞、死锁、无限循环 |
| **P1** | 建议修复 | 竞态条件、边界情况资源泄漏、未处理的错误路径 |
| **P2** | 锦上添花 | 命名、代码结构、轻微冗余 |

## HTML 报告（可选）

默认输出在终端/聊天中。如果需要 HTML 报告：

```
review，生成 HTML 报告
review abc1234，generate report
```

报告保存在项目的 `.code-reviews/` 目录下，自动在浏览器中打开。

建议把 `.code-reviews/` 加入 `.gitignore`。

## 支持的平台

| 平台 | 自动检测标记 | 专项审查维度 |
|------|-------------|-------------|
| **Android** | `build.gradle*`, `gradlew`, `AndroidManifest.xml` | 生命周期安全、协程泄漏、Room事务、ProGuard 兼容等 9 个维度 |
| **iOS** | `*.xcodeproj`, `Podfile`, `Package.swift` | 多线程安全、Block循环引用、dispatch死锁等 9 个维度 |
| **通用** | 其他项目 | 正确性、线程安全、内存、性能、API设计、错误处理、安全 7 个维度 |

## 智能行为

- **重复问题合并**：同一问题出现 3+ 次，合并为一条 + "Found in N files"
- **API 变更追溯**：函数签名改了自动检查调用方是否更新
- **测试覆盖提示**：改了代码没改测试，温柔提醒（不算问题）
- **大 diff 保护**：超 5000 行变更会提前警告

## 文件过滤

以下文件自动跳过，不浪费审查时间：

- 二进制文件、图片、字体
- 生成文件：`R.java`, `BuildConfig.java`, `*.pb.go`, `*.generated.*`
- 依赖锁：`package-lock.json`, `Podfile.lock`, `yarn.lock`
- 第三方代码：`vendor/`, `node_modules/`, `Pods/`
- IDE 配置：`.idea/`, `.vscode/`, `*.xcuserdata`

## 目录结构

```
code-reviewer/
├── SKILL.md                          # Skill 定义和工作流
├── references/
│   ├── review-general.md             # 通用审查规则（7 维度）
│   ├── review-android.md             # Android 审查规则（9 维度）
│   └── review-ios.md                 # iOS 审查规则（9 维度）
└── scripts/
    └── render_report.py              # HTML 报告渲染器
```

## 注意事项

- 只读操作，不会修改你的代码
- 不会执行 `git reset`, `git clean`, `git push --force` 等破坏性命令
- 严重度判断保守原则：不确定时选低级别，宁可漏报不误报
