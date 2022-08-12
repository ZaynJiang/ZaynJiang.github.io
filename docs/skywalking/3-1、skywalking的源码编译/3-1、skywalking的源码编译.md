## 同步代码

如何通过本地命令行同步更新？

1. 验证远程分支可以 fetch 或 push

```text
git remote -v
```

2. 指明我们需要同步的仓库

```text
git remote add upstream https://github.com/OriginalRepo/OriginalProject.git
```

3. 验证

```text
git remote -v
```

4. 拉取更新的 branches 和 commits

```text
git fetch upstream
```

5. Checkout 本地分支

```text
git checkout master
```

6. 合并

```text
git merge upstream/master
```

7. 提交

```text
git push origin master
```

## 编译步骤

* fork源代码

* git clone源代码

* git submodule init

* git submodule update

  git submodule update报错，或是没有任何反应都是不行的。很多时候因为网络原因，update的文件不全，我们就需要重新执行update命令，执行前，需要删除上面.gitmodules对应的path目录，重新执行命令让它重新下载。比如编译到apm-network这一步报错，往往是因为apm-protocol/apm-network/src/main/proto下的文件缺失，所以重新执行命令下载

* mvnw clean package -DskipTests

  

