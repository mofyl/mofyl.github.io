---
title: article
date: 2020-03-22 21:30:06
tags: 
	- git
	- git op
categories: test
---
- 提交代码：
    - git add .  // 意思是将所有的修改添加到本次提交中
    - git commit // 输入该命令后 会进入一个提交文件中，可以再次确认需要提交哪些修改，若要提交则将该文件的注释去掉。确认完了后，在第一行加上提交 log 然后保存退出 就好。
    - git push 将代码上传的当前分支
- 拉取代码：
    - git pull ： 后面不跟分支名 默认拉取当前分支
    - git pull orgin 分支名 ：拉取该分支的代码 到当前分支。
- git log：
    - git log --graph 图形化展示所有提交
- git checkout： 切换分支命令
    - git checkout 分支名： 切换到该分支
    - git checkout -b 分支名 ：创建并 切换到该分支
- git status : 查看当前git 状态
- git放弃本次修改   git checkout
    - 放弃某个文件 git checkout 文件路径+文件名
    - 放弃全部修 git checkout .
- 分支操作
    -  git branch -d branch_name:删除本地分支
    -  git push origin --delete branch_name :删除远端分支
    - git branch 查看分支分支
    - git checkout -b <name> 创建名为name的分支 并切换到该分支
    - git push --set-upstream origin <name> 推送当前分支并建立与远程上游的跟踪
    - 合并某分支到当前分支：git merge <name>
- 如果想放弃本地的文件修改，可以使用git reset --hard FETCH_HEAD，FETCH_HEAD表示上一次成功git pull之后形成的commit点。然后git pull
- git config --global core.editor nvim 修改默认编辑器
### git stash
- 该命令用于预保存文件　，
    - 应用场景：我们当正在处理一些事情，但是中间有新的需求过来，我们就可以先将本次的修改预保存，因为本次的修改可能不是最终版，因此使用commit是不合理的。所以我们就需要　git stash　命令了 。
    - 该命令会先将当前的修改存储起来(是所有的修改)，然后后面可以释放这些存储的文件
- git stash save "message": 储存命令，存储当前修改
- git stash list : 查看保存的内容，保存的修改批次 可能有多个
- git stash pop : 恢复最新的保存进度，并且将 恢复的工作从 stash  list中移除 按时间倒叙排列
- git stash drop  [stash_id] ： 删除某个stash 任务
- git stash clear #删除所有储存进度
- df1e875e052fc919c3f55bc1c47ca878318fe53c


