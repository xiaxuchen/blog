# 使用指南

## 持续部署

每次提交，.github/hugo.yml 将会调用 github actions 进行编译并上传到 github pages 进行同步，因此您可能需要申请一下 github pages,那么你的博客就能通过 github 的域名公网访问了。

## 前置工作

- 安装 hugo
  
  [官网](https://www.gohugo.org/)
- 同步拉取主题

    ```
    git submodule update --init --recursive
    ```

## 预览

```
hugo server -D -d=./public
```

`-D`表示展示草稿,访问 http://localhost:1313 即可

## 常用命令

- 创建博客

```
hugo new markdown文件.md
```
