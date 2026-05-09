## 准备工作

1. 安装 **Node.js**（必须，用来启动本地预览）
    官网下载无脑下一步安装即可。
2. 注册好 **Gitee 码云** 账号（[gitee.com](https://gitee.com)）

---

## 第一步：安装 Docsify 工具

运行

```bash
# i:install
# -g:全局安装，安装后，`docsify` 命令在系统的任何目录下都可以直接使用，而不是只在当前项目的 `node_modules` 里。
npm i docsify-cli -g
```

运行

```bash
docsify -v
```

---

## 第二步：创建你的文档目录

1. 电脑新建一个文件夹，比如叫 `my-notes`
2. 把你所有 **Markdown 文件（.md）** 全部丢进去

---

## 第三步：创建 2 个必须配置文件

进入 `my-notes` 文件夹，新建两个文件：

### 1. 新建 `index.html`

复制下面全部粘贴进去（不用改任何东西）：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>我的知识库</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/docsify/lib/themes/vue.css">
</head>
<body>
  <div id="app"></div>
  <script>
    window.$docsify = {
      name: '我的文档',
      search: { placeholder: '搜索文档' },
      loadSidebar: true,
      subMaxLevel: 2
    }
  </script>
  <!-- 核心依赖 -->
  <script src="https://cdn.jsdelivr.net/npm/docsify/lib/docsify.min.js"></script>
  <!-- 搜索插件 -->
  <script src="https://cdn.jsdelivr.net/npm/docsify/lib/plugins/search.min.js"></script>
  <!-- 侧边栏 -->
  <script src="https://cdn.jsdelivr.net/npm/docsify/lib/plugins/sidebar.min.js"></script>
</body>
</html>
```

### 2. 新建 `README.md`

这个是**网站首页**，随便写点内容，比如：

```markdown
# 欢迎来到我的文档库
这里存放所有我的笔记、技术文档
```

### 3.（可选）新建 `_sidebar.md` 侧边栏菜单

用来自定义左侧目录，示例：

```markdown
- 首页
  - [介绍](README.md)
- 技术笔记
  - [Jenkins学习](jenkins.md)
  - [CICD概念](cicd.md)
```

> 你有多少个 md 文件，就照着格式加一行就行。

---

## 第四步：本地先预览（确认效果）

进入 `my-notes` 文件夹，打开终端执行：

```bash
docsify serve .
```

浏览器打开：`http://localhost:3000`

就能看到你的所有 markdown 已经变成**带侧边栏、搜索、美化的网站**了。

---

## 第五步：上传到 Gitee

1. 登录 Gitee → 右上角 **新建仓库**
    - 仓库名：随便填，比如 `my-docs`
    - 公开仓库
    - 不要初始化 README
2. 回到本地 `my-notes` 文件夹，右键 **Git Bash** 执行：
	
	```bash
	git init
	git add .
	git commit -m "初始化文档"
	git remote add origin https://gitee.com/你的用户名/my-docs.git
	git push -u origin master
	```

## 第六步：开启 Gitee Pages 生成在线网址

1. Gitee 进入你的仓库 → 右上角 **服务** → **Gitee Pages**
2. 设置：
    
    - 部署分支：`master`
    - 部署目录：`/` 根目录
    
3. 点击 **启动**
4. 等待 1 分钟，会生成一个网址：

```plaintext
https://你的用户名.gitee.io/my-docs/
```

**手机、电脑随时打开这个网址就能看所有文档**

---

## 后续使用

以后你新增 / 修改 `.md` 文件，只需要：


```bash
git add .
git commit -m "更新文档"
git push
```

然后去 Gitee Pages 点一下**更新**，网站就同步更新了。

---

我可以给你**打包好现成模板**，你直接解压把自己的 md 丢进去，直接上传就能用，要不要我发给你？