---
title: "git 提交信息的七条建议"
date: 2021-09-12T10:45:50+08:00
toc: true
---

觉得[这篇](https://chris.beams.io/posts/git-commit/)文章写的不错，翻译过来作为参考。

## 好的提交信息的重要性

良好的提交信息能够帮助开发者了解代码更改的上下文背景，并且能够帮助其他开发者更好的进行项目维护。

## 写出一个好的提交信息的七条建议

### 1. 使用空行分开标题和内容

git commit 的[文档页面](https://www.kernel.org/pub/software/scm/git/docs/git-commit.html#_discussion)给出了标题的建议

> Though not required, it’s a good idea to begin the commit message with a single short (less than 50 character) line summarizing the change, followed by a blank line and then a more thorough description. The text up to the first blank line in a commit message is treated as the commit title, and that title is used throughout Git. For example, Git-format-patch(1) turns a commit into email, and it uses the title on the Subject line and the rest of the commit in the body.

并不是每条提交都需要标题和内容，有时候一行内容就足够了，尤其是当代码更改比较简单，不需要提供更改背景。例如

```
Fix typo in introduction to user guide
```

如果一个提交值得一些解释和背景知识，就需要在 commit 里写上一些内容

```
Derezz the master control program

MCP turned out to be evil and had become intent on world domination.
This commit throws Tron's disc into MCP (causing its deresolution)
and turns it back into a chess game.
```

### 2. 标题限制在50字以内

50字以内并不是强制规定，而是一个推荐标准。限制在50字以内能够使得标题更可读，不会被网页界面截断，并且迫使作者思考如何能够精准概括所提交的修改。

### 3. 标题首单词首字母要大写

### 4. 不要在标题后面加句号

### 5. 使用命令式语气撰写标题

例如

- Clean your room
- Close the door
- Task out the trash

使用命令式语气的一个原因是 git 默认的提交信息也是命令式语气，例如`Merge branch XXX`，`Revert XXX`

一个撰写命令式语气标题的方法是，你的标题总能够补全下面一句话

- If applied, this commit will *your subject line here*

### 6. 消息的内容每行不超过72个字符

消息内容如果超过72个字符，就要换到下一行，因此 Git 能够有足够的空间去排列文本，方便阅读

### 7. 使用消息内容解释 what, why

在提交信息中要写出代码做了什么改动（更改前是什么样子的，发生了什么错误，更改后是如何正常工作的），为什么要这样修改。

说不定后面感谢你这样写的人就是你自己！
