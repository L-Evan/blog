---

title: "Hugo教程" # 标题
date: 2021-10-03T11:56:50+08:00    # 创建时间
lastmod: 2021-10-03T11:56:50+08:00 # 最后修改时间
tags: ['博客']
categories: ['博客']  # 分类

---

## HuGo博客搭建

## 预备知识

本博客采用Typora写文章，图片由PicGo传腾讯云Cos，再整合到Hugo，推送到GitHub由Action推送到云服务器直接nginx代理

- 评论模块采用utteranc
- 主题采用Jane

## 前期准备

HuGo：https://github.com/gohugoio/hugo   记得配置环境变量，下载分为**扩展版和非扩展版**

Active：GitHub Action 不多说明，自行学习  下面有案例

PicGo：对Typora的配置，需要配置Cos的图床，也自行学习

utteranc in gihub安装：https://segmentfault.com/a/1190000039217582

Jane + hugo + git : https://www.kancloud.cn/yunduanio/gohugo_learning/1439166

nginx:    `yum install nginx`

服务器：自己买（推荐腾讯云的学生机淘宝有卖）

## 流程

1. 本博客的流程化采用Typora对文章编写MarkDown，将文章图片存储相对路径

![image-20211002231652860](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002231652860.png)

2. 需要上传文章的时候，将文章的图片推送到腾讯云Cos

   此处建议建立版本库，本人习惯上传前做一份版本控制，**记录图片的本地位置这样本地不上网也可以看**

![image-20211002231926277](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002231926277.png)

![image-20211002232003270](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002232003270.png)

3. 上传所有文章到Cos

   ![image-20211003115412197](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003115412197.png)



## 配置

## Action自部署

1. 创建自己的Actions

![image-20211003112650763](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003112650763.png)

![image-20211003112702722](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003112702722.png)

2. 将action写入

~~~yml
## github Action文件
name: deploy
# 触发时间
on:
  # push事件
  push:
    # 忽略某些文件和目录，自行定义
    paths-ignore:
      - '.forestry/**'
      - 'archetypes/**'
      - '.gitignore'
      - '.gitmodules'
      - 'README.md'
    branches: [ main ]

  # pull_request事件
  pull_request:
    # 忽略某些文件和目录，自行定义
    paths-ignore:
      - '.forestry/**'
      - 'archetypes/**'
      - '.gitignore'
      - '.gitmodules'
      - 'README.md'
    branches: [ main ]
  
  # 支持手动运行
  workflow_dispatch:
# 触发后的工作流
jobs:
  # job名称为deploy
  deploy:
    # 使用GitHub提供的runner，用github的服务器
    runs-on: ubuntu-20.04
    steps:
      # 检出代码，包括submodules，保证主题文件正常
      - name: Checkout source
        uses: actions/checkout@v2
        with:
          ref: main
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod
      
      # 准备Hugo环境
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      # Hugo构建静态站点，默认输出到public目录下    
      - name: Build
        run: hugo --gc --config=./config.toml --verbose --minify

      # 将public目录下的所有内容同步到远程服务器的nginx站点路径，注意path参数的写法，'public'和'public/'是不同的
      - name: Deploy
        uses: burnett01/rsync-deployments@5.1
        with:
          switches: -avzr --delete
          path: ./public/
          # 下面5个属性都是需要配置的环境变量  在github 
          remote_host: ${{ secrets.REMOTE_HOST }} # 自己服务器IP
          remote_port: ${{ secrets.REMOTE_PORT }} # 端口
          remote_path: ${{ secrets.REMOTE_PATH }} # 推送路径 /home/xxx/web（记得权限得和私钥的所有者同步）
          remote_user: ${{ secrets.REMOTE_USER }} # 登录的用户名
          remote_key: ${{ secrets.REMOTE_KEY }}  # 私钥

~~~

3. 配置变量

![image-20211003112951586](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003112951586.png)

4. 私钥的获取和配置

~~~bash
cd  /home/xxxx(用户目录)/.ssh  # 进入用户目录，不推荐用/root 权限过高
ssh-keygen -t ed25519 -C  "1129190684@qq.com"   # 生成私钥和公钥  ed25519 加密算法自行选择
cat  rsync.pub >> authorized_keys   # rsync.pub 是生成的公钥，自己选的名字,追加到末尾（authorized_keys是认证列表）
cat rsync   # 私钥需要赋值到 github secrets 的remote_key  
~~~

![image-20211003113541857](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003113541857.png)

## Nginx配置

nginx需要配置(注意nginx权限问题，博主采用的是将nginx用户加入到私钥所属用户的用户组的方式)

![image-20211003114140959](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003114140959.png)

记得设置用户组的读权限（默认是700）

![image-20211003114207130](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003114207130.png)

> 另外，不好的做法：直接设置启动用户为root或者对应私钥用户

![image-20211003114334810](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003114334810.png)



## nginx的配置文件

在此路径下（注意吧/etc/nginx/nginx.conf 的默认端口改为其他）

![image-20211003114449648](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003114449648.png)

~~~nginx
# 重定向到443
server {
    listen 80;
    server_name levani.cn;
    rewrite ^(.*)$  https://$host$1 permanent;
}

server {

#SSL 访问端口号为 443
    listen 443 ssl;
 #填写绑定证书的域名
    server_name levani.cn;
 #证书文件名称  https
    ssl_certificate /etc/nginx/ssl/1_levani.cn_bundle.crt;
 #私钥文件名称
    ssl_certificate_key /etc/nginx/ssl/2_levani.cn.key;
    ssl_session_timeout 5m;
 #请按照以下协议配置
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
 #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    location / {
        #root /docker/laravel7/www/blog/public;
        #try_files $uri $uri/ /index.php?$query_string;
        # proxy_pass http://172.18.0.21:80/;
        root /home/lighthouse/web;
        #include proxy.conf;
     }

}

~~~



## 代码样式

Jane的代码高亮样式采用的是HuGo自带的Chroma，需要Hugo(0.68+ 记得)才支持，不过最新好像有高亮影响（issu282）

因为拓展版会去编译原生sass而非扩展版可以直接用缓存

- 注意点：hugo需要使用扩展版本否则不能转sass: https://github.com/gohugoio/hugo/issues/7182
- github Action 是否开启 extended 扩展版
- 报错信息`Check your Hugo installation; you need the extended version to build SCSS/SASS.`

![image-20211003013819903](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003013819903.png)

![image-20211003013903842](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003013903842.png)

### 流程

1. 直接上配置文件

~~~toml
# 语法高亮 https://gohugo.io/content-management/syntax-highlighting/#configure-syntax-highlighter
[markup]
  [markup.highlight]
    codeFences = true
    # 没表明语音自动检测
    guessSyntax = true
    # 高亮行数组 1
    hl_Lines = ''
    anchorLineNos = false
    lineAnchors = ''
    lineNoStart = 1
    # 开启行号
    lineNos = true
    ## true 的话行号不影响代码复制
    lineNumbersInTable = true
    # 是否开启style 也就是下面的代码样式
    noClasses = true 
    tabWidth = 4
    # 代码样式
    style = 'solarized-light'  # 多种选择https://help.farbox.com/pygments.html 		              https://xyproto.github.io/splash/docs/
~~~

2. 修改默认的高亮代码部分（作者认为是XY问题推荐 导出css noClasses = false 的方法）

![image-20211003111548503](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003111548503.png)

~~~css
// normal <code> style
code {
  padding: 3px 7px;
  background: $code-background;
  border-radius: 4px;
  color: $code-color;
}
// 代码块都none 侵入式的 不推荐 多添加这个css
td code {
  background: none;
}
~~~







### 老版本支持

jane博客初期是用的highlight.js，[`e683155`](https://github.com/xianmin/hugo-theme-jane/commit/e683155138ca3144c5a84426df323b15c2a1a708)后进行适配，如果是老版本需要设置

~~~toml
#突出显示选项。参见 https://github.com/xianmin/hugo-theme-jane/issues/46
pygmentsCodeFences = true  #使用 GitHub 风格的代码栅栏启用语法高亮
pygmentsUseClasses = true  #使用 CSS 类来格式化高亮代码
pygmentsCodefencesGuessSyntax = true 
pygmentsOptions = " linenos=table "
~~~





## 博客功能添加

### 评论

评论采用的是utteranc在仓库，根据文章建立issues的方式

1. 首先要在GitHub装对应的GitHubApp: https://github.com/apps/utterances

   只需要添加到自己的博客仓库就可以了（当然自己有多个需要也可以）

![image-20211002233042389](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002233042389.png)

2. 添加标签支持（可选）

   jane有对utteranc进行适配，不过没有添加标签的支持，可以自己添加  `label="{{ .Site.Params.utteranc.label}}"`

![image-20211002233529003](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002233529003.png)

3. 配置文件添加评论的开启

   将配置添加到config.toml

~~~toml
[params]
	comment =  true  # 开启评论功能
## 配置 utteranc评论, 教程参考 https://utteranc.es/
[params.utteranc]
  enable = true # 开启utteranc评论功能
  repo = "L-Evan/blog" ##user/仓库name
  issueTerm = "title" # pathname issue匹配模式，详细看utteranc介绍
  label = "comment"  # issue标签（可选）
  theme = "github-light" # 评论区的样式，详细看utteranc介绍
~~~

## HuGo博客搭建

## 预备知识

本博客采用Typora写文章，图片由PicGo传腾讯云Cos，再整合到Hugo，推送到GitHub由Action推送到云服务器直接nginx代理

- 评论模块采用utteranc
- 主题采用Jane

## 前期准备

HuGo：https://github.com/gohugoio/hugo   记得配置环境变量，下载分为**扩展版和非扩展版**

Active：GitHub Action 不多说明，自行学习  下面有案例

PicGo：对Typora的配置，需要配置Cos的图床，也自行学习

utteranc in gihub安装：https://segmentfault.com/a/1190000039217582

Jane + hugo + git : https://www.kancloud.cn/yunduanio/gohugo_learning/1439166

nginx:    `yum install nginx`

服务器：自己买（推荐腾讯云的学生机淘宝有卖）

## 流程

1. 本博客的流程化采用Typora对文章编写MarkDown，将文章图片存储相对路径

![image-20211002231652860](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002231652860.png)

2. 需要上传文章的时候，将文章的图片推送到腾讯云Cos

   此处建议建立版本库，本人习惯上传前做一份版本控制，**记录图片的本地位置这样本地不上网也可以看**

![image-20211002231926277](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002231926277.png)

![image-20211002232003270](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002232003270.png)

3. 上传所有文章到Cos

   ![image-20211003115412197](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003115412197.png)



## 配置

## Action自部署

1. 创建自己的Actions

![image-20211003112650763](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003112650763.png)

![image-20211003112702722](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003112702722.png)

2. 将action写入

~~~yml
## github Action文件
name: deploy
# 触发时间
on:
  # push事件
  push:
    # 忽略某些文件和目录，自行定义
    paths-ignore:
      - '.forestry/**'
      - 'archetypes/**'
      - '.gitignore'
      - '.gitmodules'
      - 'README.md'
    branches: [ main ]

  # pull_request事件
  pull_request:
    # 忽略某些文件和目录，自行定义
    paths-ignore:
      - '.forestry/**'
      - 'archetypes/**'
      - '.gitignore'
      - '.gitmodules'
      - 'README.md'
    branches: [ main ]
  
  # 支持手动运行
  workflow_dispatch:
# 触发后的工作流
jobs:
  # job名称为deploy
  deploy:
    # 使用GitHub提供的runner，用github的服务器
    runs-on: ubuntu-20.04
    steps:
      # 检出代码，包括submodules，保证主题文件正常
      - name: Checkout source
        uses: actions/checkout@v2
        with:
          ref: main
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod
      
      # 准备Hugo环境
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      # Hugo构建静态站点，默认输出到public目录下    
      - name: Build
        run: hugo --gc --config=./config.toml --verbose --minify

      # 将public目录下的所有内容同步到远程服务器的nginx站点路径，注意path参数的写法，'public'和'public/'是不同的
      - name: Deploy
        uses: burnett01/rsync-deployments@5.1
        with:
          switches: -avzr --delete
          path: ./public/
          # 下面5个属性都是需要配置的环境变量  在github 
          remote_host: ${{ secrets.REMOTE_HOST }} # 自己服务器IP
          remote_port: ${{ secrets.REMOTE_PORT }} # 端口
          remote_path: ${{ secrets.REMOTE_PATH }} # 推送路径 /home/xxx/web（记得权限得和私钥的所有者同步）
          remote_user: ${{ secrets.REMOTE_USER }} # 登录的用户名
          remote_key: ${{ secrets.REMOTE_KEY }}  # 私钥

~~~

3. 配置变量

![image-20211003112951586](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003112951586.png)

4. 私钥的获取和配置

~~~bash
cd  /home/xxxx(用户目录)/.ssh  # 进入用户目录，不推荐用/root 权限过高
ssh-keygen -t ed25519 -C  "1129190684@qq.com"   # 生成私钥和公钥  ed25519 加密算法自行选择
cat  rsync.pub >> authorized_keys   # rsync.pub 是生成的公钥，自己选的名字,追加到末尾（authorized_keys是认证列表）
cat rsync   # 私钥需要赋值到 github secrets 的remote_key  
~~~

![image-20211003113541857](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003113541857.png)

## Nginx配置

nginx需要配置(注意nginx权限问题，博主采用的是将nginx用户加入到私钥所属用户的用户组的方式)

![image-20211003114140959](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003114140959.png)

记得设置用户组的读权限（默认是700）

![image-20211003114207130](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003114207130.png)

> 另外，不好的做法：直接设置启动用户为root或者对应私钥用户

![image-20211003114334810](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003114334810.png)



## nginx的配置文件

在此路径下（注意吧/etc/nginx/nginx.conf 的默认端口改为其他）

![image-20211003114449648](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003114449648.png)

~~~nginx
# 重定向到443
server {
    listen 80;
    server_name levani.cn;
    rewrite ^(.*)$  https://$host$1 permanent;
}

server {

#SSL 访问端口号为 443
    listen 443 ssl;
 #填写绑定证书的域名
    server_name levani.cn;
 #证书文件名称  https
    ssl_certificate /etc/nginx/ssl/1_levani.cn_bundle.crt;
 #私钥文件名称
    ssl_certificate_key /etc/nginx/ssl/2_levani.cn.key;
    ssl_session_timeout 5m;
 #请按照以下协议配置
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
 #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    location / {
        #root /docker/laravel7/www/blog/public;
        #try_files $uri $uri/ /index.php?$query_string;
        # proxy_pass http://172.18.0.21:80/;
        root /home/lighthouse/web;
        #include proxy.conf;
     }

}

~~~



## 代码样式

Jane的代码高亮样式采用的是HuGo自带的Chroma，需要Hugo(0.68+ 记得)才支持，不过最新好像有高亮影响（issu282）

因为拓展版会去编译原生sass而非扩展版可以直接用缓存

- 注意点：hugo需要使用扩展版本否则不能转sass: https://github.com/gohugoio/hugo/issues/7182
- github Action 是否开启 extended 扩展版
- 报错信息`Check your Hugo installation; you need the extended version to build SCSS/SASS.`

![image-20211003013819903](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003013819903.png)

![image-20211003013903842](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003013903842.png)

### 流程

1. 直接上配置文件

~~~toml
# 语法高亮 https://gohugo.io/content-management/syntax-highlighting/#configure-syntax-highlighter
[markup]
  [markup.highlight]
    codeFences = true
    # 没表明语音自动检测
    guessSyntax = true
    # 高亮行数组 1
    hl_Lines = ''
    anchorLineNos = false
    lineAnchors = ''
    lineNoStart = 1
    # 开启行号
    lineNos = true
    ## true 的话行号不影响代码复制
    lineNumbersInTable = true
    # 是否开启style 也就是下面的代码样式
    noClasses = true 
    tabWidth = 4
    # 代码样式
    style = 'solarized-light'  # 多种选择https://help.farbox.com/pygments.html 		              https://xyproto.github.io/splash/docs/
~~~

2. 修改默认的高亮代码部分（作者认为是XY问题推荐 导出css noClasses = false 的方法）

![image-20211003111548503](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003111548503.png)

~~~css
// normal <code> style
code {
  padding: 3px 7px;
  background: $code-background;
  border-radius: 4px;
  color: $code-color;
}
// 代码块都none 侵入式的 不推荐 多添加这个css
td code {
  background: none;
}
~~~



### 老版本支持

jane博客初期是用的highlight.js，[`e683155`](https://github.com/xianmin/hugo-theme-jane/commit/e683155138ca3144c5a84426df323b15c2a1a708)后进行适配，如果是老版本需要设置

~~~toml
#突出显示选项。参见 https://github.com/xianmin/hugo-theme-jane/issues/46
pygmentsCodeFences = true  #使用 GitHub 风格的代码栅栏启用语法高亮
pygmentsUseClasses = true  #使用 CSS 类来格式化高亮代码
pygmentsCodefencesGuessSyntax = true 
pygmentsOptions = " linenos=table "
~~~





## 博客功能添加

### 评论

评论采用的是utteranc在仓库，根据文章建立issues的方式

1. 首先要在GitHub装对应的GitHubApp: https://github.com/apps/utterances

   只需要添加到自己的博客仓库就可以了（当然自己有多个需要也可以）

![image-20211002233042389](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002233042389.png)

2. 添加标签支持（可选）

   jane有对utteranc进行适配，不过没有添加标签的支持，可以自己添加  `label="{{ .Site.Params.utteranc.label}}"`

![image-20211002233529003](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002233529003.png)

3. 配置文件添加评论的开启

   将配置添加到config.toml

~~~toml
[params]
	comment =  true  # 开启评论功能
## 配置 utteranc评论, 教程参考 https://utteranc.es/
[params.utteranc]
  enable = true # 开启utteranc评论功能
  repo = "L-Evan/blog" ##user/仓库name
  issueTerm = "title" # pathname issue匹配模式，详细看utteranc介绍
  label = "comment"  # issue标签（可选）
  theme = "github-light" # 评论区的样式，详细看utteranc介绍
~~~

4. **最终效果**

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002234040062.png" alt="image-20211002234040062" style="zoom:50%;" /><img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211002234106883.png" alt="image-20211002234106883"  />



## 最终效果

![image-20211003120531616](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20211003120531616.png)


------------------



> 主题：https://www.zhihu.com/question/266175192
>
> 教程：https://www.kancloud.cn/yunduanio/gohugo_learning/1439166
>
> 文档：https://gohugo.io/getting-started/installing/
>
> jane:https://github.com/xianmin/hugo-theme-jane/blob/master/README-zh.md
>
> function : https://gohugo.io/functions/index-function/
>
> 变量：https://gohugo.io/variables/site/
>
> 详细介绍：https://kuang.netlify.app/blog/hugo.html
>
> 主题的一些定制：https://sr-c.github.io/2020/07/21/Hugo-custom/