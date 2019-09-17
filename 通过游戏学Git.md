# 通过游戏学Git

> Git小游戏网站  **https://learngitbranching.js.org**
>
> 2019-09-16 creaylei

## 基础篇

### **git commit**

> C2的提交基于C1之上

![UTOOLS1568624081760.png](https://i.loli.net/2019/09/16/XQ5M8Doi2P3NYmp.png)

---

### git branch

> git分支是轻量的，只是简单的指向某个提交记录
>
> 早建分支，多用分支

`git branch bugFix`      新增了指针到C1

![UTOOLS1568624310428.png](https://i.loli.net/2019/09/16/qG5uSFlKejJYVHi.png)

`git checkout bugFix`   切换到分支

上述两条命令可以合并为:   `git checkout -b bugFix`  新建并切换到新分支

---

### git merge

> 现有两条不同的分支，都有各自的提交，需要合并

将bugFix分支合并到master

![UTOOLS1568624652002.png](https://i.loli.net/2019/09/16/weqPR3KVxliGC2X.png)

此时C4继承了C2和C3，即C4包含了所有的修改

如果再将master合并到bugFix，则会出现: **bugFix指向C4**，简单明了

![UTOOLS1568624917084.png](https://i.loli.net/2019/09/16/ygLTpro57xubzCR.png)

---

### git rebase: 另一种合并方式

先是这样的一个分支情况

![UTOOLS1568625309365.png](https://i.loli.net/2019/09/16/LElrsJzdIKPiyM2.png)

`git rebase mater`

![UTOOLS1568625370817.png](https://i.loli.net/2019/09/16/n7rTIta1NVmdEs4.png)

切换到master上，并执行`git rebase bugFix`

![UTOOLS1568634968033.png](https://i.loli.net/2019/09/16/94d6laILYnv8EGJ.png)

## 高级篇

### 分离HEAD

> 此处的C1相当于提交记录的哈希值，真实的是`d98d61ce9d6c3e639b1fb2cd3bfc1f35622ed815` 一长串，手动check的时候可以输入前几个字母就可以了  d98d61

![UTOOLS1568625999086.png](https://i.loli.net/2019/09/16/vMkP7CG2HWpSzFD.png)

### 相对引用

> master^^   

`git checkout master^` 移动到第一个父节点， 也可以 `git checkout head^`

![UTOOLS1568626268603.png](https://i.loli.net/2019/09/16/WgdBfZmH5h1Ve9x.png)

使用

![UTOOLS1568626824842.png](https://i.loli.net/2019/09/16/LIjF854n26CHNlc.png)

`git branch -f master HEAD~3`  强制将分支指向某个节点

---

### 撤销提交

`git reset` 和 `git revert`

> 远程使用revert  本地reset

![UTOOLS1568627459568.png](https://i.loli.net/2019/09/16/roTALk2EZtBYG5b.png)

`git reset`

git reset： 通过把分支记录回退几个提交记录来实现撤销改动。你可以将这想象成“改写历史”。`git reset` 向上移动分支，原来指向的提交记录就跟从来没有提交过一样。 详情如下

![UTOOLS1568627606728.png](https://i.loli.net/2019/09/16/qvPUjeSiIN4ohHW.png)

`git revert` : 可以理解为，恢复之前的修改后提交

![UTOOLS1568628254945.png](https://i.loli.net/2019/09/16/aFNI9EeK4w8uYAL.png)

此时 C2‘ = C1

## 移动提交记录

### git cherry-pick

> 自由修改提交树   `git cherry-pick`

`git cherry-pick <提交号>`

如果你想将一些提交复制到当前所在的位置（`HEAD`）下面的话， Cherry-pick 是最直接的方式了。我个人非常喜欢 `cherry-pick`，因为它特别简单。

![UTOOLS1568688842466.png](https://i.loli.net/2019/09/17/9xusNaRbrdHw45o.png)

在master分支上，使用`git cherry-pick C2 C4` 就把side分支上的C2和C4分支复制过来了，非常优秀

---

### 交互式 rebase

`git rebase -i HEAD~3`   -i 代表 interact 交互，这样会弹出一个交互界面

![UTOOLS1568689034311.png](https://img02.sogoucdn.com/app/a/100520146/d8e00f1dec60c7c8ccac4ea1dc98993c)

在目标分支输入 `git rebase -i HEAD~3`

![UTOOLS1568689676638.png](https://img04.sogoucdn.com/app/a/100520146/4e965a3502dae4f6cc408c1df11649a1)

可以删除提交，拖动修改顺序

![UTOOLS1568689765890.png](https://img02.sogoucdn.com/app/a/100520146/d0244384dddbd8c3e58ef8b4dc3627a6)

这里主要记住git cherry-pick， 工作中常用

## 杂项

> 技术、技巧和小贴士集合

### 场景一

> 我正在解决某个特别棘手的 Bug，为了便于调试而在代码中添加了一些调试命令并向控制台打印了一些信息。
>
> 这些调试和打印语句都在它们各自的提交记录里。最后我终于找到了造成这个 Bug 的根本原因，解决掉以后觉得沾沾自喜！
>
> 最后就差把 `bugFix` 分支里的工作合并回 `master` 分支了。你可以选择通过 fast-forward 快速合并到 `master` 分支上，但这样的话 `master` 分支就会包含我这些调试语句了。你肯定不想这样，应该还有更好的方式……

**实际上我们只需要让Git复制解决问题的那个提交就可以了。 这里我们可以使用cherry-pick  或者  rebase -i** 

`git cherry-pick <hash值>`

### 场景二

>  你之前在 `newImage` 分支上进行了一次提交，然后又基于它创建了 `caption` 分支，然后又提交了一次。
>
> 此时你想对的某个以前的提交记录进行一些小小的调整。比如设计师想修改一下 `newImage` 中图片的分辨率，尽管那个提交记录并不是最新的了。

用git rebase 可以先将  提交2 和提交3进行更换，然后更改提交2（amend，附加提交），再进行交换提交2和提交3

![UTOOLS1568690632413.png](https://img03.sogoucdn.com/app/a/100520146/692452ca447a83a320bbd5c4e1c67f81)

### 场景三

> 分支很容易被人为移动，并且当有新的提交时，它也会移动。分支很容易被改变，大部分分支还只是临时的，并且还一直在变。
>
> 有没有什么永远指向某个提交记录的标识呢？ git tag

![UTOOLS1568691221101.png](https://img04.sogoucdn.com/app/a/100520146/ae6bcd07ac3416c4fd204d9f99e77f5f)

除非删除了某个提交，否则这个tag永远就是标识这次提交

### 场景四

> git describe ：找出和你距离最近的tag

![UTOOLS1568691532735.png](https://img01.sogoucdn.com/app/a/100520146/0b52526a9cabec8cc528873f8ba6a667)

例子哈

![UTOOLS1568691569895.png](https://img01.sogoucdn.com/app/a/100520146/5beefe80f36af6d9422b0116040e3059)

