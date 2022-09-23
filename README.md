# hexo-blog
hexo博客


## 安装hexo客户端
```
npm install -g hexo-cli
```
1. 创建一个用来放 hexo 的文件夹
```
hexo init
```

## 安装 hexo-server
```
npm install hexo-server --save
```

1. 启动 server
```
hexo s
```

## 安装 deploy 插件
```
npm install hexo-deployer-git --save
```

## 安装Hexo-Admin
1. cd hexo目录
```
npm install --save hexo-admin

```

2. 启动hexo

```
hexo s
```
3. 访问 http://localhost:4000/admin 访问管理页面

### 界面说明
Pages - 新加 page；
Posts - 新加或删除 post；双击一个 post，你可以编辑，预览，新增修改 tags、categories，选择发布或不发布；
Settings - 一些配置；
Deploy - 可以直接部署到 github。


### 打包
```
hexo g
```