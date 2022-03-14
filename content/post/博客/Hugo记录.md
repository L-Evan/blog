--- 

title: "Hugo记录" # 标题
date: 2022-03-14T15:54:10+08:00    # 创建时间
lastmod: 2022-03-14T15:54:10+08:00 # 最后修改时间
tags: ['Blog']
categories: ['Blog']  # 分类

---

## 概况

经常忘记一些配置和指令所以开个帖子负责回忆的，每次增加

## 内容

### 指令

~~~java
// 注意需要进入hugo目录进行，也就是hugo/base
hugo new post/Blog/Hugo记录
~~~

### 推送流程

1. git add . （进行文章添加缓存区）
2. cz （进行github Conventional Commit规范提交）
3. git pull (防止冲突)
4. git push (推送到github除法github action)

### hugo目录结构

此处以我自己本人的目录为准

~~~java
- bash // hugo目录
  - .github
    - workflows
      - deploy.yml // github action文件
  - content // 文章目录
    - post //核心文章目录
    - about.md // 关于文章
  - archetypes // 模板目录 用来生成自动化（简单脚本）
  - themes // 主题目录  
    - hyde  //hyde是现在的主题(主题被我手动修复过代码高亮问题，所以暂时不更新)
- bin // hugo程序目录，使用hugo指令需要注意下配置环境变量
