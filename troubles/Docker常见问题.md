# Docker常见问题

1. 权限问题：

   这个问题在CentOS下面出现的比较频繁，很多时候会给你一个Permission Denied之类的提示，即使你在root权限之下。最好的解决办法是将CentOS下的SELinux禁用，一般都不会有很大问题了。具体操作：终端进入/etc/selinux，修改当前目录的config文件，修改为SELINUX=disabled，重启服务器即可。

2. 是直接用 `yum` / `apt-get` 安装 Docker 吗？ 

    `docker`, `docker.io`, `docker-engine` 甚至 `lxc-docker` 都有什么区别？ 其中，RHEL/CentOS 软件源中的 Docker 包名为 `docker`；Ubuntu 软件源中的 Docker 包名为 `docker.io`；而很古老的 Docker 源中 Docker 也曾叫做 `lxc-docker`。这些都是非常老旧的 Docker 版本，并且基本不会更新到最新的版本，而对于使用 Docker 而言，使用最新版本非常重要。

    所以，不要使用操作系统软件源中的 Docker。

    使用阿里云或者DaoCloud的安装脚本

    阿里云：

    ```Shell
    curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
    ```

    DaoCloud:

    ```shell
    curl -sSL https://get.daocloud.io/docker | sh
    ```

3. 关于底层的操作系统选用

    选用ubuntu会好很多，docker更加偏向于ubuntu，具体参照这个[文章](http://dockone.io/article/972)

