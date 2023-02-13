# Git基本命令

| 命令                        | 作用                                 |
| --------------------------- | ------------------------------------ |
| git status                  | 查看仓库状态                         |
| git init                    | 初始化仓库                           |
| git add                     | 将文件加入缓冲区（此时未提交到仓库） |
| git commit                  | 将文件提交到仓库                     |
| git branch                  | 查看分支                             |
| git branch *branch* name    | 创建分支                             |
| git branch -d *branch name* | 删除分支                             |
| git merge                   | 分支合并                             |
| git log                     | 查看日志                             |
| git tag                     | 为当前分支添加标签                   |
| git checkout *branch name*  | 切换分支                             |
| git push                    | 将本地仓库push到远程仓库             |
| git pull                    | 将远程仓库pull到本地仓库             |

git commit --amend  -m "content"  ->上一次修改有错误或忘记提交部分文件时 使用--amend 覆盖上一次提交记录，避免产生过多无用记录。


# Git学习资源

- [git闯关网站](https://learngitbranching.js.org/?locale=zh_CN&NODEMO=)：使用命令模拟结合动画的方式学习 git，使用后发现很适合对 git 的理念了解但不熟悉命令细节的用户。如果不知道什么是版本控制直接使用该网站可能一头雾水。
- [手撕Git，告别盲目记忆](https://zhuanlan.zhihu.com/p/98679880)

# Git实用工具

- TortoiseGit：Git 的图形化工具，在文件管理器打开仓库所在的文件夹，直接用右键的方式进行 Git 操作。
- GitHub Desktop：GitHub 的官方客户端，可以对仓库进行管理以及实现各种 Git 操作。

