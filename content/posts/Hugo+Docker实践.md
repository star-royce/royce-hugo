---
title: "Hugo+Docker实践"
date: 2020-06-02
slug: "hugo docker operate"
draft: false
tags:
- Hugo
- Docker
categories:
- Hugo
---

# 快捷方式

1. 访问 https://github.com/star-royce/royce-hugo-docker.git
2. 将项目clone到本地，根据README.md指引操作即可。

# Docker

### 知识文档
> https://yeasy.gitbook.io/docker_practice/introduction

### 多阶段构建
> http://dockone.io/article/8179

### DockerFile 最佳实践
> https://segmentfault.com/a/1190000018108361

# 实践问题点记录

### 什么情况缓存会生效？
>Docker构建默认会开启缓存，只要一个构建指令满足以下三个条件，这一层镜像构建就不会再执行，它会直接利用之前构建的结果。
<br/>某一层的镜像缓存失效之后，它之后的镜像层缓存都会失效。
<br/>所以我们应该把变化最少的部分放在Dockerfile的前面，这样可以充分利用镜像缓存。
<br/>dockerfile中有可能导致缓存失效的命令WORKDIR、CMD、ENV、ADD等，像这些命令最好放到dockerfile底部，以便在构建镜像过程中最大限度使用缓存。
<br/><br/>缓存生效有三个关键点
> 1. 镜像父层没有发生变化
> 2. 构建指令不变
> 3. 添加文件校验和一致。

### 全局配置的国内源不起效

1. 国内源配置教程
> https://www.jianshu.com/p/34d3b4568059

2. 问题描述: 在参照1的教程配置国内源后，实际build docker时，依旧没有从那个国内源拉取

3. 解决方案: 在单个DockerFile中增加步骤，直接指定使用阿里的国内源
   -  `RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories`

### COPY与ADD指令

1. **ADD介绍**
   - ADD指令允许您使用URL作为参数,将从指定的URL下载并添加到容器的文件系统中
      - 指定下载文件的名称 `ADD http://foo.com/bar.go /tmp/main.go`
      - 使用默认文件名称 `ADD http://foo.com/bar.go /tmp/`
      - 下载后的文件权限自动设置为 600，如果这并不是想要的权限，需要增加额外的一层 RUN 进行权限调整
      - 下载的是个压缩包，需要解压缩，还需要额外的一层 RUN 指令。因此更建议直接使用 RUN 指令，然后使用 wget 或者 curl 工具下载，处理权限、解压缩、然后清理无用文件。
   - ADD的另一个功能是能够自动解压缩压缩文件。但前提是ADD的是来自上下文路径的资源，且压缩格式为 gzip, bzip2 以及 xz。
   - ADD有可能会令镜像构建缓存失效。特别是下载远程资源的情况，一定会下载一遍远程资源，用来跟缓存文件校验，从而判定要不要使用缓存。

2. **COPY介绍**
   - 与ADD不同的是，COPY直接将文件和文件夹从构建上下文复制到容器中。COPY不支持URL作为参数，因此它不能用于从远程位置下载文件。任何想要复制到容器中的东西都必须存在于本地构建上下文中。
   - COPY对压缩文件没有特别的处理。如果您复制归档文件，它将完全按照出现在构建上下文中的方式落入容器中，而不会尝试解压缩它。
   - 相比ADD，Docker团队的建议是在几乎所有情况下都使用COPY。

3. **为什么不建议使用ADD**
   - ADD试图做得太多，但是并没有省事太多，反而容易让人感到困惑，而COPY指令会有更明确的意图，只做拷贝
   - 只有当你有一个压缩文件，并且想自动解压到镜像中，ADD能很好的支持
   - 从远程URL获取软件包有更优解。使用curl命令下载压缩包，然后通过管道传递给tar命令解压。这样我们就不会在我们需要清理的文件系统上留下压缩文件

   ```
   RUN curl http://foo.com/package.tar.bz2 \
      | tar -xjC /tmpackage \
      && make -C /tmp/package
   ```

### ARG与ENV

1. 两者起作用的时机不同
   - arg 是在 build 的时候存在的, 可以在 Dockerfile 中当做变量来使用
   - env 是容器构建好之后的环境变量, 不能在 Dockerfile 中当参数使用
2. ENV 只需要在DockerFile中编写，而ARG的值需要在build指令中指明
3. 临时使用一下的变量没必要存环境变量的值就很适合使用 ARG
4. ARG可以用来实现指定某个Run指令之后缓存不生效的效果。
   - **场景需求**: 当你一个docer镜像下载安装了各种环境后，需要执行git clone获取最新代码，如果不做任何处理，build一次后，后续会一直使用缓存。则无法更新代码
   - **ARG的重要性**: 网上普遍提供的方案是在build指令上直接加上`no-cache`，但是这样会导致整个缓存失效，即需要重装各种环境。
   - **ARG实现方式**:
      - 在 Run git clone 这一步之前，增加命令 `ARG CACHEBUST=1`
      - 在 Build指令中增加参数 `--build-arg CACHEBUST=$(date +%s)`
   - **ARG失效缓存的原理**: 新增命令哦，因为 `CACHEBUST` 值为当前时间，所以每次运行都会不一样，而值一发生变化，后续缓存也就随之失效了

> 参考文档: https://blog.justwe.site/post/docker-arg-env/

### 博客网站样式异常

1. 如果服务启动成功后，前端展示样式异常，首先需要排查的是源项目的 `config.toml` 文件配置是否有误。`baseURL` 需要对应修改为自己服务的根路径。
