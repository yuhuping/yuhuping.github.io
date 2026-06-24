---
title: GitHub Pages 默认 Jekyll 误抢构建,Hexo 部署报 butterfly theme not found
date: 2026-06-24 17:01:52
tags:
  - Hexo
  - GitHub Pages
  - GitHub Actions
  - Jekyll
  - 部署
categories:
  - 问题排查
---

> **运行环境**
> - 系统:macOS 26.5.1 (arm64)
> - 本地:Node v26.3.0 / npm 11.17.0 / hexo-cli 4.3.2 / hexo 8.1.2 / hexo-theme-butterfly 5.5.5 / gh 2.95.0
> - CI:GitHub Actions(ubuntu-latest)+ 自建 deploy.yml(Node 22)
> - 场景:用 Hexo + Butterfly 搭个人博客,推送到 `用户名.github.io` 仓库,经 GitHub Actions 部署到 GitHub Pages

## 问题现象

博客首次部署后,本地一切正常,线上也能打开(HTTP 200)。但紧接着收到一封 GitHub 的构建失败邮件,日志关键部分如下:

```text
build  Build with Jekyll
Logging at level: debug
Configuration file: /github/workspace/./_config.yml
Theme: butterfly
.../jekyll-3.10.0/lib/jekyll/theme.rb:84:in `rescue in gemspec':
  The butterfly theme could not be found. (Jekyll::Errors::MissingDependencyException)
...
Could not find 'butterfly' (>= 0) among 179 total gem(s) (Gem::MissingSpecError)
```

诡异的地方:本地 `hexo generate` 能正常找到 Butterfly 主题并出图,自己写的 GitHub Actions 工作流也跑成功了(57s),站点确实上线了——可偏偏又来一封"构建失败"。

## 排查过程

第一反应是"主题没装好",于是逐一排除:

1. **确认主题在依赖里**。`package.json` 有 `hexo-theme-butterfly: ^5.5.5`,`package-lock.json` 也包含它,且都已提交。
2. **复现 CI 环境**。本地删掉 `node_modules` 后 `npm ci && hexo generate`,结果 **22 个文件正常生成,主题找得到**。说明本地侧、依赖侧都没问题。
3. **直接看失败 run 的真实日志**(不靠猜):

   ```bash
   gh run list --limit 6
   gh run view <failed_run_id> --log-failed
   ```

   这一步定位到关键:失败的 job 名字叫 **`pages build and deployment` → `Build with Jekyll`**,堆栈全是 `jekyll-3.10.0` / `github-pages-232` 这些 **Ruby gem**。

线索瞬间清晰:报错根本不是来自我的 Hexo 工作流,而是另一个 **Jekyll** 构建器。

## 原因

GitHub Pages 有一个**内置的默认 Jekyll 构建器**(workflow 名为 `pages-build-deployment`)。当仓库 Pages 的发布源是 "Deploy from a branch / 默认" 时,每次 push 它都会自动跑,并尝试用 Jekyll 解析根目录的 `_config.yml`。

而 Hexo 的 `_config.yml` 里有一行:

```yaml
theme: butterfly
```

Jekyll 把它当成 **Jekyll 的 gem 主题**去加载,在 179 个 gem 里找不到名为 `butterfly` 的 gem,于是抛出 `The butterfly theme could not be found`。

换句话说:**Butterfly 是 Hexo 主题,却被毫不相干的 Jekyll 拿去构建了。** 我自建的 Actions 工作流其实一直是成功的,这封失败邮件来自"另一条流水线"。

再核对时间线也对得上:那次失败发生在我把 Pages 发布源切换为 "GitHub Actions" **之前**,是切换前残留的最后一次 Jekyll 构建。

## 解决方案

核心就一件事:**把 Pages 的发布源(build type)改成 GitHub Actions**,让默认 Jekyll 构建器不再触发。

用 API 一行搞定(也可在 仓库 Settings → Pages → Build and deployment → Source 选 "GitHub Actions"):

```bash
gh api -X PUT repos/<owner>/<repo>/pages -f build_type=workflow

# 确认
gh api repos/<owner>/<repo>/pages | grep build_type
# => "build_type":"workflow"
```

验证修复:再 push 一次,观察本次只触发自己的 Hexo 工作流,不再有 `pages-build-deployment`:

```bash
gh run list --limit 3 --json name,event,conclusion,status
# 只剩 "Deploy Hexo to GitHub Pages" 在跑,无 Jekyll
```

最终 deploy 成功(38s),线上 `HTTP 200`,告警邮件不再出现。

## 小结

- "构建失败"邮件不一定来自你写的工作流——**先 `gh run view --log-failed` 看清是哪条流水线在报错**,堆栈里的 `jekyll` / `gem` 字样就是默认 Jekyll 构建器的指纹。
- 用非 Jekyll 的静态站点生成器(Hexo / Hugo / VitePress 等)部署 GitHub Pages 时,**第一步就把 Pages 发布源设为 "GitHub Actions"**,避免默认 Jekyll 来抢构建。
- 本地能跑 ≠ 线上同一套逻辑;但本地复现(`npm ci`)能快速排除依赖问题,把范围缩小到 CI 环境本身。
