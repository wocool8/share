# GIT操作指南
## 一 基础命令
---
### git init 
    把当前的目录变成git可以管理的仓库
### git add fileName                              
    把文件添加到暂存区中去   
### git add .
    把当前目录的所有没有版本控制的文件添加到暂存区     
### git rm --cached filename    
     
    删除已经暂存的文件 即被add的文件 
    
### git stash                                           
    
    Git还提供了一个stash功能，可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作

### git stash  list                                     
    
    查看stash    

### git stash  pop                                      
    
    恢复stash

### git stash apply stash@{0}                           
    
    先查看 然后恢复指定的stash(stash@{0}为标识)    
      
### git commit -m"描述内容"
    
    将暂存区里的改动提交到本地的版本库，每次使用git commit 命令我们都会在本地版本库生成一个40位的哈希值
    这个哈希值也叫commit-id
    
### git status                                  	    

     git status命令用于显示工作目录和暂存区的状态。使用此命令能看到那些修改被暂存到了, 哪些没有, 
     哪些文件没有被Git tracked到
     
### git log                                           
     
    显示从最近到最远的显示日志 例子
    
### git log --pretty=oneline  

    显示一行的日志信息
    
### git reflog                                         
 
    获取所有操作的版本号 
    
### cat filename                         
               
     查看文件内容          
    
### rm filename
                                        
    删除文件  
      
## 二 分支及合并命令
---

### git branch

    查看本地库的所有分支(当前分支上面会加上*)	
    
### git branch branchName

    基于当前分支代码新建branchName分支
    
### git branch -d (分支名)    
                          
    删除这个分支，例子：git branch -d test 删除test分支    

### git checkout branchName                           
    
    切换到branchName分支
    
### git checkout -b branchName                           
    
    新建并切换到branchName分支   
    
### git remote

    查看所有的关联远端仓库
    
### git remote -v                                       

    查看远程仓库的详细信息    

### git remote add origin (项目的地址)

    关联远程仓库  例子 git remote add http://source.jd.com/app/las-schedule.git  
    
### git clone (项目的地址) 
                                                    
    克隆远端仓库代码到本地 git clone http://source.jd.com/app/las-schedule.git   
    clone后会自动关联本地仓库和远端仓库  
    
### git remote rm (远程仓库名)                          
     删除某个远程仓库，例子：git remote rm origin 删除远程仓库origin                                           
                                                   
### git fetch 

    创建并更新本地的远程分支。即创建并更新origin/xxx 分支，拉取代码到origin/xxx分支上
    
### git merge (分支名)        
     合并这个分支到当前分支，例子：git merge test  合并test分支到当前的分支上    

### git pull <远程主机名> <远程分支名>:<本地分支名>
    git pull 相当于 git fetch 和 git merge 的合并语法
    例子   git pull origin master:master   将远程仓库origin的master分支取回并merge本地仓库的dev分支
                   
### git push origin branchName                     

    将本地branchName分支推送到某个远端仓库的branchName分支 


### git push origin (分支名)                          
     例子   git push origin master       把本地的master分支推送到远程的master分支上
	                                                
### git tag           
 
    查看标签信息
	                                                
### git tag <tagname>                                   
    默认标签是打在最新提交的commit上的
    例子：git tag version02 6224937    给commit id 为6224937打上version02标签
    例子：git tag version01            添加一个标签version01
    
### git show <tagname>                                  

    查看某个标签的详细信息，例子：git show version01

### git tag -d version01                                

    删除标签version01    

### git tag -a version03 -m "information" 6224937 
    
    给commit-id为6224937打上version03标签，打标签的信息为information

### git push origin --tags                            

    把本地的所有标签push到远程   
    
   
### git revert
 
    git revert是提交一个新的版本，将需要revert的版本的内容再反向修改回去，版本会递增，不影响之前提交的内容 
    git revert HEAD                  撤销前一次 commit
    git revert HEAD^               撤销前前一次 commit
    git revert commit commit-id 撤销指定的版本，撤销也会作为一次提交进行保存
    
-soft  

将HEAD引用指向给提交，索引和工作目录内容保持不变。只改变一个符号的引用状态使其指向一个新提交

-mixed 

将HEAD指向给提交，索引也跟着改变以符合提交的树形结构，但是工作目录保持不变，这个版本的命令将索引变成上次提交的全部变化时的状态，它会显示工作目录中还有什么修改

-hard 

这个命令会将HEAD指向给定提交，索引的内容也跟着改变以符合给定提交的树结构，此外工作目录也随之改变成给定提交表示的树的状态
    

### git reset -- hard HEAD^                              

    回退到上一个版本
    git reset –hard HEAD^ 那么如果要回退到上上个版本只需把HEAD^ 改成 HEAD^^ 以此类推。
    那如果要回退到前100个版本的话，使用上面的方法肯定不方便，
    我们可以使用下面的简便命令操作：git reset  –hard HEAD~100 即可
    
### git reset --hard 版本号                             
    head指针 回到这个版本号的版本    
    
### git rebase branchName
    
    当前分支rebase branchName分支
    
### git rebase -i commit-id 

    rebase到commit-id   
    
## 三 stash代码 
---
通常我们在本地环境开发会修改`jsf`别名及其他配置文件，此时如果我们想`checkout`到其他分支会出现如下问题

![can  not checkout](../picture/git/cannotCheckout.PNG)

通常不应该提交修改别名代码，如果想要切换分支就要把修改的代码`revert`，此时可以使用stash把代码暂存
然后checkout到别的分支出处理，当处理结束后使用`git stash pop` 把暂存的代码取回(如下图操作)

![stash and pop](../picture/git/stash.PNG)
## 四 解决merge冲突  
---
 
    <<<<<<< HEAD
        private Date time2;
    =======
        private Date time1;
    >>>>>>> feature1
指的是当前分支的`HEAD`与`feature1`分支合并发生冲突，可以直接编辑解决冲突，推荐使用idea的图形化页面解决
在`Local Changes`选中标红文件然后右键选择`git`，然后选择`resolve conflicts`再双击需要解决的文件如下图

![stash and pop](../picture/git/conflicts.PNG)

`但代码发生冲突一定要找到相关人一起解决，避免把别人代码合丢`


## 五 使用Reset回滚代码版本
---
revert也可以回滚代码（基于逆向提交的方式），但是会使得代码提交记录存在正向和逆向的版本记录，这段版本号是无用提交，所以采用reset（HEAD指向回滚的版本号）

![stash and pop](../picture/git/reset.PNG)

将代码回滚到使用use gradle这个版本

    PS C:\thread> git reset --hard b25cb13a04779554d665a00f482c3465f1eeff98
    HEAD is now at b25cb13 use gradle

此时本地代码和远端仓库代码已经不一致，需要将本地仓库代码覆盖远端仓库，使用相对安全的命令`git push -f`推到远端分支
## 六 使用Rebase变基提交
---
`rebase` 分为`git rebase branchName` 和 `git rebase -i commit-id` 两种操作，本次主要介绍`-i`的方式

![stash and pop](../picture/git/rebase.PNG)

如上图由于本地`commit`记录比较乱，所以想要对提交记录进行`rebase`操作首先执行 `git rebase -i d653b689bb73441ba461775c8b1b3f1404c37cae`
(版本号对应的是feature2 update这次提交)然后进入如下页面

![stash and pop](../picture/git/pick.PNG)

通常使用`squash`使用上次提交的版本号

![stash and pop](../picture/git/squash.PNG) 

修改成如上图`:wq`保存，然后会进入下图

![stash and pop](../picture/git/combination.PNG)

直接`:q`退出

![stash and pop](../picture/git/rebaseResult.PNG)

操作过程中可能出现冲突，解决冲突后使用`git rebase -- continue`继续`rebase`操作，也可以使用`git rebase --skip`退出，
`rebase操作一定要在feature分支上操作，不要在公共分支上进行rebase，否则有可能会操作不当搞乱代码库`

## 七 使用cherry-pick合并指定版本代码
---
当两个人使用共同分支进行开发时，有可能出现部分提交需要上线，部分提交不需要上线的情况，此时`reset`会导致丢失代码，
使用`cherry-pick`取回可以上线的提交版本，当前的提交记录如下图

![stash and pop](../picture/git/cherrypick.PNG)

想要在feature2分支上取回需求1的两次提交，进行如下操作

    PS C:\thread> git cherry-pick -i 5f99b83b1212d5da2b9d47f40445b44bec1f1c64
    [feature2 869bf8a] 需求1第一次提交
     Date: Mon Jan 14 23:54:26 2019 +0800
     1 file changed, 1 insertion(+), 1 deletion(-)
    PS C:\thread> git cherry-pick -i 2335895c25455df52f4b0030f9d290d0278915f4
    [feature2 34bc9a4] 需求1第二次提交
     Date: Mon Jan 14 23:56:05 2019 +0800
     1 file changed, 3 insertions(+), 3 deletions(-)

![stash and pop](../picture/git/cherrypickResult.PNG)