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

### 修改上次以提交的commit

```bash
## commit id 是上次提交的 commit id
$ git rebase -i ceccc
Stopped at 823affa...  gotty 101
## 按照下面提示操作即可
You can amend the commit now, with

  git commit --amend

Once you are satisfied with your changes, run

  git rebase --continue

## 输入新的commit message
$ git commit --amend --message="2022-10-18: gotty 101"
[detached HEAD cb4df21] 2022-10-18: gotty 101
 Date: Tue Oct 18 23:32:47 2022 +0800
 6 files changed, 415 insertions(+), 2 deletions(-)
 create mode 100644 Develop/[LGTM] Golang gotty--web console.md
 create mode 100644 "Develop/golang/Golang Temporal \345\210\206\345\270\203\345\274\217\344\273\273\345\212\241\350\260\203\345\272\246\346\241\206\346\236\266.md"
# 
(main|REBASE 1/1)$  git rebase --continue
Successfully rebased and updated refs/heads/main.

w25506@w25506 MINGW64 /d/data_files/bgyq/bgyq (main)
# 需要强制push
$ git push -f


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

