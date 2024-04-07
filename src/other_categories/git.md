# [git](git-scm.com)

## 强制 Git 在任何地方都使用固定的行结束符

以 LF 为例(默认情况下 Unix like 都是 LF，Windows 为 CRLF)：

```bash
git config core.eol lf
git config core.autocrlf input

git config --global core.eol lf
git config --global core.autocrlf input
```

如果需要的话，使用正确的行结束符重建当前仓库所有的文件：
```bash
git checkout-index --force --all
```

上面的操作仍然会导致一些文件的行结束符没按预期工作，请从本地副本中删除所有内容并更新它们。

```bash
git rm --cached -r .
git reset --hard
```



## Git 常见用法

下面介绍常见的 git 使用姿势：

**1.初始化本地仓库**

```bash
git init <directory>
```

`<directory>` 是可选的，如果不指定，将使用当前目录。

**2.克隆一个远程仓库**

```bash
git clone <url>
```

**3.添加文件到暂存区**

```bash
git add <file>
```

要添加当前目录中的所有文件，请使用 . 代替 `<file>`,代码如下：

```bash
git add .
```

**4. 提交更改**

```bash
git commit -m "<message>"
```

如果要添加对跟踪文件所做的所有更改并提交。

```bash
git commit -a -m "<message>"
```

**5.从暂存区删除一个文件**

```bash
git reset <file>
```

**6.移动或重命名文件**

```bash
git mv <current path> <new path>
```

**7. 从存储库中删除文件**

```bash
git rm <file>
```

您也可以仅使用 `--cached` 标志将其从暂存区中删除

```bash
git rm --cached <file>
```

**基本 Git 概念**

**8.默认分支名称：main**

**9.默认远程名称：origin**

**10.当前分支参考：HEAD**

**11. HEAD 的父级：HEAD^ 或 HEAD~1**

**12. HEAD 的祖父母：HEAD^^ 或 HEAD~2**

**13. 显示分支**

```bash
git branch
```

**有用的标志：**

`-a`：显示所有分支（本地和远程）

`-r`：显示远程分支

`-v`：显示最后一次提交的分支

**14.创建一个分支**

```bash
git branch <branch>
```

你可以创建一个分支并使用 `checkout` 命令切换到它。

```bash
git checkout -b <branch>
```

**15.切换到一个分支**

```bash
git checkout <branch>
```

**16.删除一个分支**

```bash
git branch -d <branch>
```

您还可以使用 `-D` 标志强制删除分支。

```bash
git branch -D <branch>
```

**17.合并分支**

```bash
git merge <branch to merge into HEAD>
```

**有用的标志：**

`--no-ff`：即使合并解析为快进，也创建合并提交

`--squash`：将指定分支中的所有提交压缩为单个提交

建议不要使用 `--squash` 标志，因为它会将所有提交压缩为单个提交，从而导致提交历史混乱。

**18. 变基分支**

变基是将一系列提交移动或组合到新的基本提交的过程。

```bash
git rebase <branch to rebase from>
```

**19. 查看之前的提交**

```bash
git checkout <commit id>
```

**20. 恢复提交**

```bash
git revert <commit id>
```

**21. 重置提交**

```bash
git reset <commit id>
```

您还可以添加 `--hard` 标志来删除所有更改，但请谨慎使用。

```bash
git reset --hard <commit id>
```

**22.查看存储库的状态**

```bash
git status
```

**23.显示提交历史**

```bash
git log
```

**24.显示对未暂存文件的更改**

```bash
git diff
```

您还可以使用 `--staged` 标志来显示对暂存文件的更改。

```bash
git diff --staged
```

**25.显示两次提交之间的变化**

```bash
git diff <commit id 01> <commit id 02>
```

**26. 存储更改**

`stash` 允许您在不提交更改的情况下临时存储更改。

```bash
git stash
```

您还可以将消息添加到存储中。

```bash
git stash save "<message>"
```

**27. 列出存储**

```bash
git stash list
```

**28.申请一个藏匿处**

应用存储不会将其从存储列表中删除。

```bash
git stash apply <stash id>
```

如果不指定 `<stash id>`，将应用最新的 `stash`（适用于所有类似的 `stash` 命令）

您还可以使用格式 `stash@{<index>}` 应用存储（适用于所有类似的存储命令）

```bash
git stash apply stash@{0}
```

**29.删除一个藏匿处**

```bash
git stash drop <stash id>
```

**30.删除所有藏匿处**

```bash
git stash clear
```

**31. 应用和删除存储**

```bash
git stash pop <stash id>
```

**32.显示存储中的更改**

```bash
git stash show <stash id>
```

**33.添加远程仓库**

```bash
git remote add <remote name> <url>
```

**34. 显示远程仓库**

```bash
git remote
```

添加 `-v` 标志以显示远程存储库的 URL。

```bash
git remote -v
```

**35.删除远程仓库**

```bash
git remote remove <remote name>
```

**36.重命名远程存储库**

```bash
git remote rename <old name> <new name>
```

**37. 从远程存储库中获取更改**

```bash
git fetch <remote name>
```

**38. 从特定分支获取更改**

```bash
git fetch <remote name> <branch>
```

**39. 从远程存储库中拉取更改**

```bash
git pull <remote name> <branch>
```

**40.将更改推送到远程存储库**

```bash
git push <remote name>
```

**41.将更改推送到特定分支**

```bash
git push <remote name> <branch>
```