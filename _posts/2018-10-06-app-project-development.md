---
layout: post
title: "App项目开发最佳实践"
subtitle: 'App Project Development Best Practices'
author: "DamonFish"
header-style: text
tags:
  - Project
---
今天主要讲下面几点：

1. 规范化（Format，Lint，Sonarcube）
2. 自动化（CI和CD）
3. 组件化（内部库）
4. 版本控制（Git Flow）
5. 测试（单元测试，UI测试）

## 规范化（Format，Lint，SonarQube）

简单讲就是代码的规范化；
代码格式化（Format）和代码规范检查（Lint），一般你不是用Xcode就不用特别在意，因为其他平台的开发工具都自带Format和Lint，或者提供插件。
可是Xcode，连format都不给！ 

你需要安装[SwiftFormat](https://github.com/nicklockwood/SwiftFormat)，[SwiftLint](https://github.com/realm/SwiftLint)，然后在工程里做配置脚本。

个人的SwiftFormat 配置
```
if [ "${CONFIGURATION}" = "Debug" ]; then

"${PODS_ROOT}/SwiftFormat/CommandLineTool/swiftformat" --disable unusedArguments,initCoderUnavailable,andOperator,trailingClosures,redundantReturn --nospaceoperators "...,..<" --swiftversion "5.0" "${SRCROOT}/Project/"

echo "format debug build swift code"

else

echo "Not Debug Env, skip format"

fi
```

个人的[SwiftLint](https://gist.github.com/DamonFish/df081b716551cc2c2377ea3a186d1ecd)配置

部分代码
```
disabled_rules:
  - unused_closure_parameter
  - trailing_comma 
  - force_cast
  - opening_brace
  - large_tuple
  - inclusive_language
#  - trailing_whitespace
#  - todo
```

也可以借助SonarQube这一类的第三方静态分析工具，critical的issue务必解决。

## 自动化（CI和CD）

个人常用的自动化平台是CircleCI和Jenkins。
Jenkins是免费的，免费工具的最大优点通常就是免费。 很多插件已经比较古老了。 比如自搭server的Gitlab， MRrequest检测插件很久都没人更新，最好还是按照热心网友在github提的一个pr，然后自己按照pr内容从新编译了一个新的插件。
如果有更好的选择，不推荐使用。

CircleCI也很老牌了， 和Bitbucket全家桶在国外算是标配了。
功能强大，文档全。

个人用的config配置，仅作参考
[Config](https://gist.github.com/DamonFish/5fada002fa6dd045822ccb69e496f044)

当然，在App端，自动化平台都依赖Fastlane，没有ta， 寸步难行啊。
其实App端的自动化就是：
Fastlane + CI/CD工具。

当然大厂可能会用自己的平台和工具。

## 组件化（内部库）

就是模块化后，将基础模块封装成库，提供给多个工程或者多个团队使用。

iOS常用的方式就是利用cocoapod做私用库，需要私用库spec仓库。 或者指定tag。
总之，利用cocoapod来加载这些库。

可以组件化的库一般分两种：

1. 基础设施库， 例如： 网络功能，日志，路由， debug模块，数据库模块， 文件操作等
2. 基础功能库， 例如： 网络请求模块（带基础业务功能）， 登陆模块， 账号管理模块， 会员内购模块， 分享功能等

这里需要指出的是，写代码给自己用和给一群人用，完全是两个概念。
给一群人用其实是写SDK，SDK用户的需求多变，场景复杂， 版本兼容。 稳定性，兼容性， 扩展性，都是另一个级别。 
到时候，你能更深刻的了解KISS原则，代码设计六大原则。
这里不展开讨论了。

## 版本控制

关于git工作流会在另一篇文章中单独讲。

这里说几个注意点：

1. 团队的git水平应该达到工作流程需要的水平之上；
讲个极端例子，如果团队都是用SVN的，那就谨慎考虑git，先培训分享吧。
2. 权限控制，
仍然是极端例子： 我第一次丝滑的删除了Bitbucket上的项目时， 意识到了这一点； 相比之下Github好很多。
权限必须要控制，实习生真删库跑路了，可怪不了他们。
3. 流程开始不要追求完美， 流程的制定是结合实际情况的，该调整得调整

## 测试（单元测试，UI测试，集成测试）

先抛却技术层面，应该严肃对待测试的重要性。这个也说给我自己听。
尊重QA人员， 应该互相尊重，都是打工人。 开发和测试，应该是相辅相成的关系。

技术层面上讲， TDD的思想能反过来让开发从另一个角度考虑代码结构和质量。 
曾经在客户端，特别是iOS， 单元测试是很鸡肋的东西，官方的支持很少。
但是这个时代已经彻底过去了， 应该跟上时代了。 官方的测试框架已经比较好用了；

目前我对测试的看法是， 
工具类， 数据相关， 单元测试是很必要的。
UI测试是实用性不佳，另外国内对Accessibility的支持比较少；所以目前不是很必要。

另外经验上讲：
每次发布版本，要考虑几个步骤必须测试

1. 新账号的注册和使用，被坑过几次了
2. 大规模的修改，要通知测试配合，最好一蹴而就，不要牵连多个版本。

## 个人经验，其他

1. 工程目录和模版，层次不要太深
建议工程目录按照功能模块分， 功能模块文件夹下放mvc/mvvm文件夹， 目录不要太深。 公用的放外层公用文件夹。 其他没有强制要求。
2. 第三方库的维护
版本使用稳定版本；
第三方库发布最新版本后，至少经过两周后再升级； 这个让我们避免很多次被坑，第三方库的稳定性无法保证的，经过一段时间验证后再使用。
定期更新，避免差距在大版本号之上，版本差距最好不要超过两个小版本号；
注意查看第三方库的release notes和issues列表；
3. 添加或者修改底层重要的框架或库，需要通知其他开发人员，最好大家一起评估必要性和影响。 如果修改和替换需要花较长时间，认真评估和排期，尽快的全部修改掉， 避免新旧实现共存太久。
4. 提PR前，可以先跟自己讲一个遍， 我称之为费曼提PR法。
5. 前期避免过早不必要的优化。  
6. 提倡使用新技术， 新技术的采用者请务必调研清楚， 说明必要性； 决定采用后，请让整个团队在新技术的能力在某一个水平之上，最好有最佳实践手册。
