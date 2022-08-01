## git--draft

### config user and email

gayhub 推送代码前一定要设置`user`and`email`，否则会默认使用域账号或主机名信息，这样会导致信息泄漏！！！

```bash
$ git config --global --list
credential.http://10.156.23.50.provider=generic
user.name=chandler

$ git config --global user.email "chandler@bgyq.com"

$ git config --global --list
credential.http://10.156.23.50.provider=generic
user.name=chandler
user.email=chandler@bgyq.com
```



### 远程仓库重命名

```bash
#The default branch has been renamed!
#dev is now named main

#If you have a local clone, you can update it by running the following commands.

git branch -m dev main
git fetch origin
git branch -u origin/main main
git remote set-head origin -a
```

### rebase不连续commit

这样做，有大量的conflict！！！

1. 使用git rebase -i <需要合并的commit中最早的那个commit的前一个commit-ID>，启动rebase操作。

2. 进入到编辑页面，手动调整提交的顺序，将需要合并的提交放到一起，需要保留的commit前面使用pick，需要合并的commit前面使用s或squash，紧跟在两个要合并的commit中需要保留的那个commit后面。

   即先`yy`指令复制`pick commit_ID`到指定位置，然后把`pick`换成`s`或`f`，实现合并

3. 弹出commit信息页面，需要手动修改合并后的commit信息。

### 修改未push commit message

```bash
--amend               amend previous commit
$ git commit --amend
[main b71d7f6] 2022-07-31: git config user and email
 Date: Sun Jul 31 19:52:35 2022 +0800
 1 file changed, 20 insertions(+), 1 deletion(-)

```

