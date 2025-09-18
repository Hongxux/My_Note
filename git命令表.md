![[Screenshot_20250903_103205_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]

![[Screenshot_20250903_103433_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]



# git本地仓库

## git配置

## 新建仓库（版本库）Repo

### 方法一 本地init一个仓库
![[Pasted image 20250903105155.png]]



### 方法二 远程clone一个仓库


## 查看配置 git config --global list
![[Pasted image 20250903104101.png]]

## 设置用户名和邮箱
![[Pasted image 20250903103825.png]]


## git工作区域和文件状态
![[Screenshot_20250903_105543_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]
## 添加和提交文件
查看仓库的状态：git status
文件的状态：
在工作区新建了一文件test1.txt
此时文件处于工作区，未被跟踪的状态
![[Pasted image 20250903110339.png]]![[Pasted image 20250903110500.png]]
### git add
使用git add命令让文件被跟踪，进入暂存区（如果创建一个空文件，git默认是不会把他加入到暂存区的）


> [!NOTE] add的一次性添加多个文件
>
> 方法一：使用正则表达 git add *.txt添加结尾为txt的所有文件
> 方法二：添加一整个文件夹 git add .
> 末尾加入个“.”，表示当前目录









也可以使用 git rm --cached 文件名的方式让文件离开暂存区
![[Pasted image 20250903110604.png]]
![[Pasted image 20250903110625.png]]
### git commit
使用git commit命令让暂存区的文件提交到本地仓库中，而不会提交工作区的文件
git commit -m “提交信息”

![[Pasted image 20250903111158.png]]

### git log
用git log 查看提交的信息
![[Pasted image 20250903111807.png]]
commit后面的参数是回退版本的时候用于指明需要回退到哪一个版本

## git reset回退版本的三种模式

![[Screenshot_20250903_112420_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]

## git diff 查看差异

### 比较两个区域的差异
> [!NOTE] git diff 使用
> git diff 默认是比较工作区和暂存区的差异
> git diff HEAD 是比较工作区和版本库的差异
> git diff --cached 是比较暂存区和版本库的差异
> 

> [!NOTE] git diff 颜色的含义
>红色表示被删除，绿色表示添加的
>![[Pasted image 20250903113803.png]]


### 比较两个特定版本的差异

后一个版本相对于前一个版本改变了什么（顺序影响结果）
![[Pasted image 20250903114127.png]]
![[Pasted image 20250903114310.png]]

> [!NOTE] tips
> HEAD表示当前版本 HEAD~表示上一个版本
> HEAD~2 表示上两个版本
> 若在 版本后加上 特定文件名，则只会查看特定文件的两个版本差异

### 比较两个分支的差异
直接加入分支名即可

## git rm删除文件

git rm 能同时删除工作区和暂存区文件

![[Screenshot_20250903_191545_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]


## .gitignore 配置文件，忽略文件
![[Screenshot_20250903_191837_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]
将想要被忽略的文件加入到==.gitignore==文件夹里面，则不会被加入暂存区和版本库

若是想要被忽略的文件已经加入到版本库，则要先用
git rm --cached 文件名删除暂存区的文件，然后再commit提交
这样既能保存文件，又能使得文件不被追踪


> [!NOTE] .gitignore的文件匹配规则
>文件匹配规则是一行表示一个忽略模式
>忽略模式是正则的![[Screenshot_20250903_195752_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]
>![[Screenshot_20250903_200101_tv_danmaku_bilibilihd_HDUnitedBizDetailsActivity.jpg]]

# git远程仓库
## SSH 配置


![[Pasted image 20250903203325.png]]

>

## 关联本地仓库和远程仓库



