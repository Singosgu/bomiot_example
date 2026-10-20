# bomiot deploy

为项目生成 GitHub Actions 工作流配置，配合 GitHub 实现 Windows / macOS / Linux 三平台的自动化构建打包。

```bash
bomiot deploy
```

## 概述

`bomiot deploy` 在当前项目中生成（或更新）`.github/workflows/greaterwms.yaml`。项目推送到 GitHub 后，该工作流自动在三个平台的 CI 环境中完成构建打包，产出可直接交付的绿色版产物，实现"一次配置，三平台自动出包"。

命令必须在**项目根目录**（`launcher.py` 所在目录）下执行。

## 使用前提

1. 已通过 `bomiot init` 初始化项目，`launcher.py` 中定义了 `app_name` 与 `version`
2. 项目计划托管到 GitHub（工作流只在 GitHub Actions 中运行）
3. **项目根目录必须存在 `requirements.txt`**——GitHub Actions 构建时通过 `pip install -r requirements.txt` 安装依赖，缺失该文件会导致云端构建直接失败。本地开发环境中执行 `pip freeze > requirements.txt` 生成

## 执行流程

```text
bomiot deploy
  │
  ├─ 1. 读取 launcher.py，获取 app_name 与 version
  ├─ 2. 为本项目生成一对授权密钥（COMMUNITY_KEY / SPONSOR_KEY）
  ├─ 3. 创建 .github/workflows/ 目录（已存在则复用）
  ├─ 4. 写入 greaterwms.yaml，并将项目信息注入工作流环境变量
  └─ 5. 完成，推送仓库后自动触发三平台构建
```

## 注入的环境变量

工作流顶部的 `env` 区块包含 4 个项目级变量，构建全程使用：

| 变量 | 来源 | 作用 |
|---|---|---|
| `APP_NAME` | launcher.py 的 `app_name` | 产物命名 |
| `BASE_VERSION` | launcher.py 的 `version` | 版本号 |
| `COMMUNITY_KEY` | 命令运行时生成 | 项目授权标识（社区） |
| `SPONSOR_KEY` | 命令运行时生成 | 项目授权标识（赞助） |

> 重复执行 `bomiot deploy` 会就地更新这 4 个变量的值（例如改版本号后重新执行），工作流中的其他自定义内容不受影响。
>
> 密钥对每次执行都会重新生成；两个密钥共同构成项目的唯一标识，即构建产物中 `build.json` 的来源。

## 与跨平台构建打包的配合

`bomiot deploy` 本身不编译任何代码，它是**云端三平台构建的开关**：

| 环节 | 命令 | 说明 |
|---|---|---|
| 本地构建 | `bomiot build` | 只构建当前机器所在的平台，适合本地验证 |
| 云端构建 | `bomiot deploy` + git push | 生成工作流后推送，GitHub Actions 三平台并行构建 |

完整协作链路：

1. **配置**：`bomiot deploy` 把应用名称、版本号、项目密钥写入工作流
2. **触发**：代码 push 到 `main` 分支，GitHub Actions 自动启动
3. **并行构建**：工作流在 Windows、macOS、Linux 三个 runner 上并行执行，各自安装依赖（读取 `requirements.txt`）并完成打包，产出携带项目标识（build.json）的平台产物
4. **交付**：三个平台的产物以 Artifact 形式挂在构建记录下，下载解压即可运行，无需 Python 环境
5. **更新**：构建产物可与 `bomiot publish` 配合发布，客户端通过增量更新获取新版本

各平台的详细构建流程与产物说明，见**构建**标签下的「构建分发」文档。

## 注意事项

- `bomiot deploy` 只生成/更新工作流文件，不会上传代码；需要手动 `git push` 才会触发构建
- 工作流文件名固定为 `.github/workflows/greaterwms.yaml`，请勿重命名，否则 GitHub 无法识别
- 项目密钥写入工作流环境变量，请评估仓库可见性（Public 仓库中的 env 对所有人可见）
- 修改 `launcher.py` 的版本号后，建议重新执行 `bomiot deploy` 同步工作流中的 `BASE_VERSION`
