--- 

title: "{{ replace .Name "-" " " | title }}" # 标题
date: {{ .Date }}    # 创建时间
lastmod: {{ .Date }} # 最后修改时间
tags: ['{{index (split .Dir "\\") 1}}']
categories: ['{{index (split .Dir "\\") 1}}']  # 分类

---

## 预备知识


