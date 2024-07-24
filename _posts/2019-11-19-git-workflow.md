---
layout: post
title: "Git 工作流程"
subtitle: "Git Workflow"
author: "DamonFish"
header-style: text
tags:
  - Git
---

Git flow/ workflow；  
Git 工作流程是团队在使用 Git 进行版本控制时商定的一个统一工作流程。  
这个工作流程可以自定义，也可以使用已经定义好的流程，最重要的是适合你的项目。

这个流程主要是针对分支管理策略。

常见的 Git 工作流程有三种：

- Git flow
- GitHub flow
- GitLab flow

## Git flow

最传统的 Git 工作流程。  
Git flow 通常包含五种类型的分支：

- **Master 分支**：主干分支，也是正式发布版本的分支，包含可以部署到生产环境中的代码。通常情况下，只允许其他分支将代码合入，不允许直接向 Master 分支提交代码（对应生产环境）。
- **Develop 分支**：开发分支，用来集成测试最新合入的开发成果，包含要发布到下一个 Release 的代码（对应开发环境）。
- **Feature 分支**：特性分支，通常从 Develop 分支拉出，每个新特性的开发对应一个特性分支，用于开发人员提交代码并进行自测。自测完成后，会将 Feature 分支的代码合并至 Develop 分支，进入下一个 Release。
- **Release 分支**：发布分支，发布新版本时基于 Develop 分支创建，发布完成后合并到 Master 和 Develop 分支（对应集成测试环境）。
- **Hotfix 分支**：热修复分支，生产环境发现新 Bug 时创建的临时分支，问题验证通过后，合并到 Master 和 Develop 分支。

![Git Flow](https://bbs-img.huaweicloud.com/blogs/img/image1(11).png)

## GitHub flow

代码托管到 GitHub 时默认使用的工作流程，比较简洁。只有 Master 分支固定。  
开发功能时，从 Master 分支拉出 Feature 分支，开发完成后提交 PR，合入 Master 分支。  
发布时，比如发布 1.0.0 版本，直接从 Master 分支创建 v1.0.0 版本并打标签。

这个合入最好是快进合并（fast-forward），即合并本身并不产生一个 commit，因为一个 Feature 分支一般只做一件事，通常只有一个 commit。  
关于这个，Linus Torvalds 特地吐槽过，简单讲他认为：GitHub 这种合并如果要生成一个 commit 的话，是多此一举。

## GitLab flow

GitLab 推荐使用的工作流。

相比于 GitHub flow，GitLab flow 增加了对预生产环境和生产环境的管理；规定 Master 分支是 Pre-Production 分支的上游，Pre-Production 是 Production 分支的上游；GitLab flow 规定代码必须从上游向下游发展；即新功能或修复 Bug 时，特性分支的代码测试后必须先合入 Master 分支，然后才能由 Master 分支向 Pre-Production 环境合入，最后由 Pre-Production 合入到 Production。

![GitLab Flow](https://bbs-img.huaweicloud.com/blogs/img/image7(7).png)

## 工作流优缺点分析

个人认为，对工作流的要求有以下几点：

1. 简单
2. 提交记录清晰，最好保持一条直线
3. 能满足基本需求：Feature 分支通过 MR/PR 合入，Release 发布，Hotfix 发布

Git flow 最大的问题是分支较复杂，而且分支的历史不是一条直线，合并频繁。

GitHub flow 和 GitLab flow 的差距不大，都比 Git flow 简单。  
建议从 GitHub flow 或 GitLab flow 中选一种，一般只需要简单调整即可满足团队需求。

## 推荐工作流

目前使用的工作流类似 GitHub flow，讲解一下细节：

### Feature 开发

主开发分支为 Master。

开发任何功能或修改，开 Feature 分支，开发完成后提交 PR。  
PR 的合并一定是快进合并（fast forward）的，不允许合并本身产生 commit 节点。  
即要求如果 Feature 分支和 Master 有冲突，rebase Master 分支。

一般一个 Feature 分支只有一次 commit，多余的 commit 可以 squash。

有些平台支持 commit squash 和自动 rebase，以及合并必须是快进合并，注意配置。

### Release 发布

比如 1.0 版本，1.0 版本的 Feature 全部合入后，开新的 release 1.0 分支。

如果 release 1.0 分支存在后，Master 上有了新的两次提交 A 和 B，B 需要在 1.0 发布。  
那么 cherry-pick B 到 release 1.0 分支。release 1.0 发布后必须打标签，然后删除 release 1.0 分支。

### Hotfix 发布

如果上面的 1.0 发布后，过了一段时间需要发修复版本，从 release 1.0 的标签开修复版本发布分支：release 1.1 分支。  
可以从 release 1.1 上开分支 hotfixFeature，开发完成后合入 release 1.1 分支，并 cherry-pick 到 Master 分支。

### 总结

简单的分支策略就这些，其中需要注意测试。  
测试以 Master 分支为主测试分支，然后是准备发布的 release 分支。  
另外可以使用 git-flow 工具做一些配置，这里不再赘述。

这个适合生产环境比较单一的场景。  
优点是：

1. 简单的分支关系：Master 主分支、功能分支、临时的 release 版本分支。
2. 提交记录是一条直线，方便阅读和追溯。
3. 一切皆有记录，每个版本都有标签，每个版本发布了什么、何时发布，简单明了。

不过还是那句话，工作流程是根据团队和场景决定的，没有最好的，只有最适合的。
