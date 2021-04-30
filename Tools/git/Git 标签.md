列出标签

git tag

查找指定的标签

`git tag -l/--list "v1.8*"`：可以使用通配符



创建标签

轻量标签

只是某个特定提交的引用

git tag <tag_name>



附注标签

> 附注标签是存储在 Git 数据库中的一个完整对象， 它们是可以被校验的，其中包含打标签者的名字、电子邮件地址、日期时间， 此外还有一个标签信息，并且可以使用 GNU Privacy Guard （GPG）签名并验证。 通常会建议创建附注标签，这样你可以拥有以上所有信息。但是如果你只是想用一个临时的标签， 或者因为某些原因不想要保存这些信息，那么也可以用轻量标签

`git tag -a <tag_name> -m "tag_msg"`

补标签

对过去的提交补标签

`git tag [-a] <tag_name> <commit_id>`



共享标签

git push 默认不会将标签信息也传到服务器上，需要手动显示进行推送。

`git push origin <tagname>`：推送某个指定标签

`git push origin --tags`：推送所有标签



删除标签

删除本地标签

`git tag -d <tag_name>`

删除远程标签

`git push origin --delete <tag_name>`

`git push origin :refs/tags/<tagname>`



检出标签

`git checkout [-b <branch_name>] <tag_name>`：默认会进入游离态



参考资料

[Git-打标签](https://git-scm.com/book/zh/v2/Git-%E5%9F%BA%E7%A1%80-%E6%89%93%E6%A0%87%E7%AD%BE)





