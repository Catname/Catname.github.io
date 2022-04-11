---
category: note-2020
published: true
layout: post
title: learnGitBranching 学习记录
description: 及时记录，定期回顾
---

# [Learn Git Branching](https://learngitbranching.js.org/?demo=&locale=zh_CN) 学习总结  

---
## 主要
### 基础篇
1. Git Commit  

```git
git commit
git commit
```

2. Git Branch  

```git
git checkout -b bugFix
```

3. Git Merge

```git
git checkout -b bugFix
git commit
git checkout main
git commit
git merge bugFix
```

4. Git Rebase

```git
git checkout -b bugFix
git commit
git checkout main
git commit
git checkout bugFix
git rebase main
```

### 高级篇

1. 分离 HEAD

```git
git checkout c4
```

2. 相对引用（^）

```git
git checkout bugFix^
```

3. 相对引用（~）

```git
git branch -f main c6
git checkout HEAD~1
git branch -f bugFix HEAD~1
```

4. 撤销变更  
**reset :（本地分支）**  
   修改本地分支，通过把分支记录回退来实现撤销改动。这种“改写历史”的方法对于多人使用的远程分支无效。（注意：在 reset 后，所做的变更还在，但是处于未加入暂存区状态）  
**revert : （远程分支）**  
   将撤销改动作为新的提交，这样就能推送到远程仓库与人共享。

```git
git reset HEAD~1
git checkout pushed
git revert HEAD
```

## 远程
### Push & Pull -- Git 远程仓库

1. Git Clone

```git
git clone
```

2. 远程分支

```git
git commit
git checkout o/main
git commit
```

3. Git Fetch

```git
git fetch
```

4. Git Pull

```git
git pull
```

5. 模拟团队合作

```git
git clone
git fakeTeamwork 2
git commit
git pull
```

6. Git Push

```git
git commit
git commit
git push
```

7. 偏离的提交历史  
   假设你周一克隆了一个仓库，然后开始研发某个新功能。到周五时，你新功能开发测试完毕，可以发布了。但是 —— 天啊！你的同事这周写了一堆代码，还改了许多你的功能中使用的 API，这些变动会导致你新开发的功能变得不可用。但是他们已经将那些提交推送到远程仓库了，因此你的工作就变成了基于项目旧版的代码，与远程仓库最新的代码不匹配了。  
   

```git
git clone
git fakeTeamwork
git commit
git pull --rebase
git push
```

8. 锁定的 Master (Locked Master)

```git
git reset --hard o/main
git checkout -b feature C2
git push origin feature
```
