修改多个提交信息，可以使用 git rebase -i 来交互式地进行变基。

例如想要修改最近的 2 次提交信息，有两种方式

```bash
git rebase -i <commit_id> # commit_id 为想要修改的最远一次提交的父提交的 commit_id
git rebase -i HEAD~2
```

重排序提交

进入交互式页面后，上下移动每行的 commit id 可以对 commit 进行重排序。



压缩提交



拆分提交

先使用 `git rebase -i` 在需要拆分的提交上标注 edit ，使得 `rebase` 过程在该提交停下

使用 `git reset HEAD^` 撤销该次提交并取消暂存

使用 git add 和 `git commit` 重新构建提交信息

使用 `git rebase --continue` 继续流程



参考资料

1. [Git 重写历史](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E9%87%8D%E5%86%99%E5%8E%86%E5%8F%B2)

    