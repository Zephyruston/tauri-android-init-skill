# tauri-android-init Skill

tauri-android-init Skill 是一个 Claude Code 的 Skill，帮助用户将 Tauri v2 项目配置为 Android 开发环境，覆盖从初始化到签名的完整流程。在 Claude Code 中，该 Skill 会在用户提及 Tauri Android、`tauri android init`、Gradle/NDK 错误、APK 签名等关键词时自动触发，也可以手动通过 `/tauri-android-init` 使用。

> 适用于 `npm create tauri-app@latest` 创建的 Tauri v2 项目。

## 项目结构

```
tauri-android-init-skill/
├── skills/
│   └── tauri-android-init/
│       ├── SKILL.md                  # Skill 定义文件
│       └── references/
│           └── tauri-android-workflow.md  # 完整配置清单、代码片段、诊断流程
├── skills.sh.json                    # skills.sh 展示页配置
├── LICENSE                           # MIT 许可证
└── README.md                         # 本文件
```

## 安装

### gh CLI

```bash
gh skill install Zephyruston/tauri-android-init-skill tauri-android-init
```

### npx

```bash
npx skills add Zephyruston/tauri-android-init-skill
```

### 验证安装

安装完成后，在 Claude Code 中提及 Tauri Android 相关内容（如 "tauri android init"、"Gradle 链接错误"、"APK 签名配置"），Skill 应自动触发加载。

## 使用方法

该 Skill 通过关键词自动触发，也可以手动引用。典型使用场景：

```
我的 tauri android init 一直报 NDK linker 错误
Tauri Android 构建成功了但手机白屏，怎么调试
怎么配置 Vite HMR 让真机可以热更新
APK release 签名怎么配置
Gradle 下载依赖一直超时，怎么设代理
```

## 支持的功能

| 分类               | 说明                                 | 示例                                     |
| ------------------ | ------------------------------------ | ---------------------------------------- |
| 环境检查           | 确认 SDK / NDK / JDK / Rust targets   | 检查 `ANDROID_HOME`、`ANDROID_NDK_HOME` 等 |
| Android 初始化     | `tauri android init` 初始化 Android 项目 | 处理 `npm install` → `init` 流程         |
| Gradle / NDK 排错  | Gradle 依赖解析、NDK 链接器配置       | `gradle.properties` 代理、`config.toml` 链接器 |
| 真机调试           | WebView 白屏排查、chrome://inspect    | adb 连接、Chrome DevTools 远程调试        |
| Vite HMR 配置      | 前端热更新 + 真机联调                 | `vite --host`、`TAURI_DEV_HOST` 配置      |
| APK 发布签名       | keystore 生成、Gradle signingConfigs   | `npm run tauri android build` 构建       |

> 更多细节参见 [`references/tauri-android-workflow.md`](./skills/tauri-android-init/references/tauri-android-workflow.md)。

## 贡献

欢迎提交 Issue 和 Pull Request 来改进这个项目。

贡献前请阅读 [SKILL.md](./skills/tauri-android-init/SKILL.md) 了解该 Skill 的工作方式和约定。

## 许可证

- [MIT License](./LICENSE)
