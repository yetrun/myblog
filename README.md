# 我的个人博客

> 使用 Hexo 搭建个人博客

## 准备

```bash
$ yarn global add hexo-cli
$ yarn
```

## 命令

新建文章（使用如下命令创建 sources/_posts/my-article-file.md）

```bash
$ hexo new my-article-file
```

生成

```bash
$ hexo clean # 若网页正常可忽略这条命令
$ hexo generate
```

生成后部署

```bash
$ hexo deploy
```

生成后启动服务预览

```bash
$ hexo server
```

等价命令

| 使用 hexo 调用  | 简写形式     | 使用 yarn 调用               |
| --------------- | ------------ | ---------------------------- |
| `hexo new`      | `hexo n`     | `node_modules/.bin/hexo new` |
| `hexo clean`    | `hexo clean` | `yarn clean`                 |
| `hexo generate` | `hexo g`     | `yarn build`                 |
| `hexo server`   | `hexo s`     | `yarn server`                |
| `hexo deploy`   | `hexo d`     | `yarn deploy`                |

## 配置域名为 blog.yet.run

只需要在域名解析中添加一条记录 CNAME 记录：

| 主机记录 | 记录类型 | 记录值           |
| -------- | -------- | ---------------- |
| blog     | CNAME    | yetrun.github.io |
