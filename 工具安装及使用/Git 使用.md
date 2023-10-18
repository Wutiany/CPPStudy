# Git 使用

## 配置 Git 全局信息

* 配置邮箱和名字

  ```shell
  $ git config --global user.email "email"
  $ git config --global user.name "name"
  ```

## 分支操作

**多个不同的分支，独立更新自己的内容，不会影响其他内容**

* 克隆已经存在的库

* 在库目录下进入 `terminal`

* 通过 git 命令创建新分支 

  ```shell
  $ git branch new_branch
  ```

* 转换分支

  ```shell
  $ git checkout new_branch
  ```

* 将当前分支内容进行提交

  * 先查看当前分支（* 表示当前 git 所在的分支）

    ```shell
    $ git branch
    ```

  * 添加要提交的文件或更改到暂存区

    * 根据文件名添加

      ```shell
      $ git add <file1> <file2> ...
      ```

    * 添加全部文件

      ```shell
      $ git add .
      ```

    * 查看暂存区内容

      ```shell
      $ git status
      ```

  * 提交暂存内容到本地仓库

    * 提交内容

      ```shell
      $ git commit -m "提交消息"
      ```

      查看文件在暂存区的更改

      ```shell
      $ git diff --staged <file>
      ```

    * 查看 commit 内容

      ```shell
      $ git log
      ```

      查看最近几个提交

      ```shell
      $ git log -n <number>
      ```

      查看文件在已提交内容的更改

      ```shell
      $ git show <commit_hash>:<file>
      ```

  * 将当前分支的内容提交到远程仓库

    ``` shell
    $ git push origin <branch_name>
    ```

  * 将其他分支的更改合并到当前分支（使用其他人的分支的模块）

    ```shell
    $ git merge <other_branch>
    ```

* 内容总结：

  分支相当于树杈，克隆下来主分支之后，通过创建新的分支，来添加内容或者修改，不影响主分支的内容，与主分支完全隔离，但是可以通过 merge 合并内容，将当前分支的内容合并到主分支，主分支就能看见所有内容。

## 操作文件

* 更改文件名

  * 更改

    ```shell
    $ git mv <旧文件名> <新文件名>
    ```

  * 提交更改

    ```shell
    $ git commit -m "重命名文件夹"
    ```

* 如果更改文件夹名字导致 merge 的时候发生冲突，需要进行冲突解决



