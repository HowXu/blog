---
title: 浅谈git的commit规范
date: 2024-11-27 16:38
tags: github
---

# 前言

参考[git commit 规范指南](https://segmentfault.com/a/1190000009048911), [Git Commit 之道:规范化 Commit Message 写作指南](https://hezephyr.github.io/posts/01.git-commit-%E4%B9%8B%E9%81%93%E8%A7%84%E8%8C%83%E5%8C%96-commit-message-%E5%86%99%E4%BD%9C%E6%8C%87%E5%8D%97/)

最近做了一点项目,commit多了就发现自己这个commit信息非常复杂而且没有章法,大多是些update:xxx,upload:xxx,可读性基本是没有了.

所以这里来写一下更正之后的commit规范,采集自上述两篇文章,用作备忘录.

# 正文

首先Commit message都包括三个部分:header,body,footer.

```
<type>(<scope>): <subject>

<body>

<footer>
```
## type

type 用于说明 git commit 的类别, 使用下面几个标识。

feat:新功能(Feature)

“feat" 用于表示引入新功能或特性的变动。这种变动通常是在代码库中新增的功能, 而不仅仅是修复错误或进行代码重构。

fix/to:修复 bug。

*fix 关键字用于那些直接解决问题的提交。当创建一个包含必要更改的提交, 并且这些更改能够直接修复已识别的 bug 时, 应使用 fix。这表明提交的代码引入了解决方案, 并且问题已被立即解决。*

*to 关键字则用于那些部分处理问题的提交。在一些复杂的修复过程中, 可能需要多个步骤或多次提交来完全解决问题。在这种情况下, 初始和中间的提交应使用 to 标记, 表示它们为最终解决方案做出了贡献, 但并未完全解决问题。最终解决问题的提交应使用 fix 标记, 以表明问题已被彻底修复。*

**feat和fix应该出现在Changelog里面**

docs:文档(Documentation)

*“docs” 表示对文档的变动, 这包括对代码库中的注释、README 文件或其他文档的修改。这个前缀的提交通常用于更新文档以反映代码的变更, 或者提供更好的代码理解和使用说明。*

style: 格式(Format)

*“style” 用于表示对代码格式的变动, 这些变动不影响代码的运行。通常包括空格、缩进、换行等风格调整。*

refactor:重构(不是新增功能, 也不是修改bug的代码变动)

*“refactor” 表示对代码的重构, 即修改代码的结构和实现方式, 但不影响其外部行为。重构的目的是改进代码的可读性、可维护性和性能, 而不是引入新功能或修复错误。*

perf: 优化相关, 比如提升性能、体验

*“perf” 表示与性能优化相关的变动。这可能包括对算法、数据结构或代码实现的修改, 以提高代码的执行效率和用户体验。*

test:增加测试

*“test” 表示增加测试, 包括单元测试、集成测试或其他类型的测试。*

chore:构建过程或辅助工具的变动(杂物活)

*“chore” 表示对构建过程或辅助工具的变动。这可能包括更新构建脚本、配置文件或其他与构建和工具相关的内容。*

revert:回滚到上一个版本

*“revert” 用于回滚到以前的版本, 撤销之前的提交。*

merge:代码合并

*“merge” 表示进行代码合并, 通常是在分支开发完成后将代码合并回主线。*

sync:同步主线或分支的 Bug

*“sync” 表示同步主线或分支的 Bug, 通常用于解决因为合并而引入的问题。*
</br>

然后,为你的commit message添加一个Scope(也可以不添加),标注它的影响范围:

```
feat(User): //表示为用户模块添加功能
```

最后,你可以在你的commit message里写详细描述了,一般动词开头:

```
feat(User): 增加用户登录功能
```

## body

body是对这次改动的详细描述,最好解释一些重要的内容,改动原因什么的,反正看着来就行

## footer

footer 部分只用于两种情况。

不兼容变动

*如果当前代码与上一个版本不兼容, 则Footer部分以BREAKING CHANGE开头, 后面是对变动的描述、以及变动理由和迁移方法。*

关闭 Issue

*如果当前commit针对某个 issue, 那么可以在 Footer 部分关闭这个 issue*

```
Closes #122,#152
```

## 其它情况

### Revert

如果当前 commit 用于撤销以前的 commit, 则必须以revert:开头, 后面跟着被撤销 Commit 的 Header。

```
revert: feat(pencil): add 'graphiteWidth' option

This reverts commit 667ecc1654a317a13331b17617d973392f415f02.
```
Body部分的格式是固定的, 必须写成This reverts commit [hash], 其中的hash是被撤销 commit 的 SHA 标识符。

