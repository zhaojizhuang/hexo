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
hexo s
hexo g

hexo d 
```
