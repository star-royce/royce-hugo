---
title: "Hugo+Github最佳实践"
date: 2020-06-02
slug: "hugo github combine"
draft: false
tags:
- Hugo
- Github
categories:
- Hugo
---

# 快捷方式

1. 访问 https://github.com/star-royce/hugo-leaveIt-project
2. 将项目clone到本地，根据README.md指引操作即可。

# 白手起家

## 环境准备

#### 1. Hugo下载安装

  - 安装包方式: https://github.com/gohugoio/hugo/releases
  - HomeBrew(**仅Mac**): `brew install hugo`
  - Windows10安装, 参考文档: https://www.gohugo.org/doc/tutorials/installing-on-windows/
    - 下载得到zip文件
    - 将zip文件放置到预期存放hugo的问题, 如"D:\Hugo\Sites"
    - 解压至当前文件夹，确定exe文件的名称，确保为hugo.exe，如果不是，请重命名
    - 将"D:\Hugo\Sites"配置到环境变量中
  - Git Clone方式，参考文档 https://s0gohugo0io.icopy.site/getting-started/installing/
  ```shell
  mkdir $HOME/src
  cd $HOME/src
  git clone https://github.com/gohugoio/hugo.git
  cd hugo
  go install --tags extended # 如果不想/不需要Sass / SCSS支持，请删除--tags extended
  ```


#### 2. Git下载安装

- 参考官方文档: https://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%AE%89%E8%A3%85-Git
- HomeBrew安装(仅Mac): `brew install git`

#### 3. 本地Git配置

- 直接联网搜索相关文档即可.

## GitHub Page 准备

#### 1. github 账号注册登录
#### 2. 网站仓库初始化

初始化一个仓库，用于承载未来的博客网站Html文件(**必须公开，必须初始化README.md**)
![repostory_prepare](https://i.loli.net/2020/07/28/NUV36xjQ54duF7a.jpg)

#### 3. 使用 github page 功能
- 进入setting页
  ![into_setting](https://i.loli.net/2020/07/28/htKMmdB8IbG4fer.png)
- 找到GitHub Pages，选择Choose a theme(随便选，只要是拿到托管的网址)
  ![github_page_config](https://i.loli.net/2020/07/28/QK53FBl8yGcN2ed.png)
- Commit README.md
    - 选择一个theme后,会自动生成README.md内容，并跳转到提交页
    - 这也是为什么一开始初始化项目的时候必须初始化一个README.md
    - **这里的commit不能忽略**，后续网站是否能正常的应用css/js，跟这个有一定关系

- 确定网址可用
  ![url_confirm](https://i.loli.net/2020/07/28/1GvYyldBPbgrjF4.png)

## 源文件项目准备

用于生成的github page的项目，只能存放最终打包好的html文件，即md原始文件，hugo项目配置等，都只能存放在本地。为了解决这一问题，我们需要一个额外的远程仓库来存放这类文件。

> 参考文档: https://jeshs.github.io/2019/01/hugo%E9%83%A8%E7%BD%B2%E5%88%B0github/

## 源文件项目设置子模块项目

1. 进入准备存放项目的根目录
3. 执行git clone指令，将源文件项目拉取到本地。
4. 在源文件项目根目录下，创建子模块，用于单独管理打包后的html文件，将其推送到GitHub Page所在的远程仓库。

    - `git submodule add -b master <xxx.io项目clone url> public`
    - **<font color=red>执行报错</font>**, "'public' already exists in the index", 可能是因为子模块有缓存，先删除缓存即可。使用git指令 `git rm --cached public`
    - **<font color=red>执行报错</font>**，"A git directory for 'public' is found locally with remote(s):", 是因为根目录项目已经关联过一次，需要删干净。需要在项目根目录下执行指令
       - `rm -rf .gitmodules`, 删除.gitmodules文件
       - `vim .git/config`, 删除 subModule配置
       ![config_submodule](https://i.loli.net/2020/07/28/CJ42oahsc79SbPf.png)
       - `rm -rf .git/modules/path_to_submodule`, 删除本地相关文件
       - 再次执行git submodule指令即可

## Hugo 项目配置

#### 1. 进入下载下来的源文件项目根路径

#### 2. 执行hugo执行，生成项目所需文件，项目名可自定义，如blog

  `hugo new site <自定义项目名，如blog>`

#### 3. 创建完成后，会把blog作为根目录文件夹，然后在该文件夹下会生成以下文件结构。
```shell
├── archetypes # 存放生成博客的模版
├── assets # 存放被 Hugo Pipes 处理的文件
├── config # 存放 hugo 配置文件 支持 JSON YAML TOML 三种格式配置文件
├── content # 存放 markdown 文件
├── data # 存放 Hugo 处理的数据
├── layouts # 存放布局文件
├── static # 存放静态文件 图片 CSS JS文件
└── themes # 存放主题
```

#### 4. 目录结构调整
因为我们计划是用一个源文件项目来存放这些文件，所以多了一层目录结构可能不太友好。可以考虑在源文件项目根路径下执行以下命令。(非必须操作，但后续步骤是假设已经做了第5步操作的前提下。如果没做，涉及目录问题，需要多进入一层目录。)
```
cp blog/ ./
rm -rf blog
```
#### 5. 主题添加

  - **以下操作默认起点是在源文件项目跟路径**
  - 因为hugo默认是没有主题，所以必须先下载主题
  - 当前选用LeaveIt主题，可以到 https://themes.gohugo.io/ 挑选自己满意的主题，但是后续提及的主题配置，可能就不适用。
  - 将主题下载到themes文件夹中
    `git clone https://github.com/liuzc/LeaveIt.git theme/`
  - 删除.git文件，避免git冲突
    `rm -rf theme/LeaveIt/.git`
  - 主题应用
  ```shell
  # 基础配置文件，指定所用主题，以及主题一些关键信息
  cp themes/LeaveIt/exampleSite/config.toml ./
  # 用于存放md文件
  cp -rf themes/LeaveIt/exampleSite/content/ ./content/
  # 静态资源文件，主要是使用image
  cp -rf themes/LeaveIt/exampleSite/static ./
  ```

## 主题配置

#### 1. config.toml 文件配置

  ```toml
  baseURL = "实际Github Page生成的网站地址"
  languageCode = "zh-cn" # 指定为中文
  defaultContentLanguage = "zh-cn" #指定为中文
  hasCJKLanguage = true # 开启可以让「字数统计」统计汉字
  title = "TODO" # 网站标题
  theme = "LeaveIt" # 所用主题
  [Permalinks]
   posts = "/:year/:month/:slug/" # 分享链接，相比默认值，增加了:slug，主要是避免链接带了中文，但需要md文件上的Front Matter配置
  [params]
    "这下面的的配置影响界面展示，可以改了结合本地网站查看效果"
  [params.social]
    # 这里影响的是首页的关联地址
      GitHub = "star-royce"
      Email   = "private.royce@gmail.com"
      Wechat = ""  # Wechat QRcode image
    # 新增配置，标注作者
  [author]
      name = "royce.li"
      [params.publisher]
        name = "royce.li"
  ```

#### 2. HTML文件自定义修改

1. 首页头像修改
    - 在 "static/images/me"路径下，存放想要用的头像照片
    - 修改 "config.toml"中的`avatar = "/images/me/main.jpeg"`, 替换成想要的头像照片路径

2. 首页页脚修改
    - 网站开始运行时间及作者，需要修改 "config.toml"中的配置
    ```
    since = 2020
    author = "Royce.li"    
    ```
    - 首页关联账号配置，见"config.toml"中的`params.social`
    - 爱心移除以及后续listen移除(可选)
        - 修改文件内容, `themes/LeaveIt/layouts/partials/footer.html`
3. 文章字数及阅读时长统计
   - 找到文件`themes/LeaveIt/layouts/_default/single.html`
   - 新增内容
   ```html
   # 方式1，只显示字数
   <!-- <span class="post-word-count">, {{ .WordCount }} words</span> -->
   # 方式2，显示字数及预估阅读时长
   <div class="post-meta">
       <span id="busuanzi_container_page_pv">|<span id="busuanzi_value_page_pv"></span>
       <span class="post-date">共{{ .WordCount }}字</span>，阅读约
       <span class="more-meta"> {{ .ReadingTime }} 分钟</span>
   </div>
   ```
4. 文章页脚license修改
   
    - 修改 "config.toml"中的 `license= 'xxxx'`
5. md内html语法支持
    - 新版本的 Hugo （从 version 0.6）使用的 Markdown 渲染器从 blackfriday 改成了 goldmark，默认禁止在 Markdown 中使用 raw html 代码。需要的话得自己开启
    - 在"config.toml"中新增如下配置

          [markup]
          defaultMarkdownHandler = "goldmark"
          [markup.goldmark]
            [markup.goldmark.renderer]
              unsafe = true

6. 字体选中样式
   - 原效果字体选中为淡蓝色，不太显眼，参考文档后进行修改，改为深黑色效果.具体操作如下
   - 打开 `/themes/LeaveIt/assets/css/_common/_core/base.scss`, 搜索 `::selection`, 替换为

        ```css
        ::selection {
          background: #080808;
          color: #FFFFFF;
            }
            ::-moz-selection {
              background: #080808;
              color: #FFFFFF;
            }
            ::-webkit-selection {
              background: #080808;
              color: #FFFFFF;
         }
        ```
7. 代码高亮样式修改
    - 原样式的代码高亮看着不舒服，可以进行以下修改
    - 打开 `themes/LeaveIt/assets/css/_common/_page/post.scss`, 搜索 `code:not([class])`, 替换为

      ```css
      code:not([class]) {
        padding: 2px 4px;
        color: #ef3982;
        border-radius: 2px;
        margin-left: 3px;
      margin-right: 3px;
      
        .dark-theme &:not([class]) {
            background: #2d2d2d;
            color: #e06c75;
        }
      }
      ```



  > 参考文档: https://huaien.co/technology/leaveit-theme-modified-log-5-selection/

## MarkDown 文件编辑
  - 新建一个md文档 `hugo new posts/my-first-blog.md`, 配置Front Matter填充一定内容，用于页面分类展示

      ---
        title: "Hugo+Github最佳实践"
        date: 2020-06-02
        slug: "hugo github combine"
        draft: false
        tags:
        - Environment
        - Hugo
          categories:
        - Environment    
      ---
  - 可以配置默认模版，每次用执行新建md文档时会自己生成，再根据需求稍作修改即可
      - 编辑文件, `/archetypes/default.md`
      - 具体内容参考下文

            ---
            title: "{{ replace .Name "-" " " | title }}"
            date: {{ dateFormat "2006-01-02" .Date }}
            slug: ""
            draft: true
            tags:
            - Tech
            - SelfLearning
            categories:
            -
            ---

> Front Matter参考文档： https://gohugo.io/content-management/front-matter/>

## Hugo 项目部署

#### 1. 本地网站检查
- 回到项目根路径，执行 `hugo server -w`, 启动成功后，访问相应地址即可
  ![hugo_run](https://i.loli.net/2020/07/28/71sAu8FvnYfD3jx.png)

#### 2. GitHub Pages 网站部署
- 确定本地网站正常后，执行 `hugo` 打包静态html，此时会把内容输入到 "public"文件夹中
- 进入public文件夹，将静态文件推送到Github Page所在项目

    ```shell
    cd public
    git status
    git add .
    git commit -m "first commit static html"
    git push
    ```
- 访问Github Page生成的网站地址，确定网站是否正常
>1. 接口404，尝试增加/index后缀
>2. css,js文件不起效，检查config.toml的baseURL配置是否跟Github Page生成的网站地址一致
>3. css,js文件依旧不起效，config.toml中尝试新增配置 `canonifyurls = true`


## Hugo 源项目备份到远程仓库
返回根目录，保存源文件到远程仓库
  ```shell
  cd ..
  git status
  git add .
  git commit -m "first commit all"
  git push
  ```

## 一键Shell部署到github

可以在源项目根路径下新建一个shell脚本，内容如下，后续编写博客内容后，直接运行此脚本即可一键部署。

```shell
#!/bin/bash
# 部署到 github pages 脚本
# 错误时终止脚本
set -e

# 打包。even 是主题
hugo

# 进入打包文件夹
cd public

# Add changes to git.

git add .

# Commit changes.
msg="博客内容更新 `date`"
if [ $# -eq 1 ]
  then msg="$1"
fi
git commit -m "$msg"

# 推送到github
# nusr.github.io 只能使用 master分支
git push

# 回到原文件夹
cd ..
```
