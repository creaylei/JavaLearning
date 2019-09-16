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

![image-20190916171727637](/Users/zhangleishuidihuzhu.com/Pictures/wiznote/image-20190916171727637.png)

## 高级篇

### 分离HEAD

> 此处的C1相当于提交记录的哈希值，真实的是`d98d61ce9d6c3e639b1fb2cd3bfc1f35622ed815` 一长串，手动check的时候可以输入前几个字母就可以了  d98d61

![UTOOLS1568625999086.png](https://i.loli.net/2019/09/16/vMkP7CG2HWpSzFD.png)

## 相对引用

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