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

## 分支拉取

* 拉取分支文件

  ```shell
  $ git pull
  ```

* 转换到自己分支，然后 merge，就能合并拉取的主分支的内容了

## 从零开始将本地代码关联到仓库中

### 首先初始化 git

```shell
$ git init
```

### 关联远程仓库

```shell
$ git remote add origin <远程仓库的地址>
```

```shell
$ git remote -v  # 查看远程仓库
```

### 本地项目无代码的情况

* 先使用 `pull` 或者 `fetch` 拉取远程仓库

* 二者的区别

  * `fetch`命令仅获取远程分支的最新提交，但不会将其自动合并到当前分支。它将更新远程分支的引用（比如`origin/main`），但不会修改你的本地分支。

    例如，使用`git fetch origin`命令可以将远程仓库的最新提交下载到本地，但你需要手动合并或处理这些提交。

  * `pull`命令执行了`fetch`操作之后，自动将远程分支的内容合并到当前分支。它等效于执行`git fetch`后紧接着执行`git merge`。

    例如，使用`git pull origin main`命令将从远程仓库获取最新的`main`分支提交，并将其自动合并到你的当前分支。

### 本地存在代码的情况

**本地存存在代码，想要将本地的代码推到空的仓库中**

* 首先创建一个将代码进行 `add` 和 `commit`
* 然后创建一个新的开发分支，将代码提交到开发分支中
* 最后转换分支，将 `master` 分支拉取，然后，将开发分支的内容合并到 `master` 分支，在 `push` 上去
