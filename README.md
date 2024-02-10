# 使用指南
## 持续部署
每次提交，.github/hugo.yml将会调用github actions进行编译并上传到github pages进行同步，因此您可能需要申请一下github pages,那么你的博客就能通过github的域名公网访问了。
## 预览

```
hugo server -D -d=./public
```

`-D`表示展示草稿,访问http://localhost:1313即可

## 常用命令
- 创建博客
```
hugo new markdown文件.md
```
