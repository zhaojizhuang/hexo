# hexo
## 1. 发布电子书

```shell 
cd public/ebook-note
gitbook build
rm -rf ../gbook/*
mv _book/* ../gbook
cd -
```

## 2. 调试电子书

```shell 
cd public/ebook-note
gitbook serve
```

## 2. 发布hexo

```shell
hexo s  #本地调试
hexo g  #生成静态文件
hexo d  #部署hexo blog到github
```
