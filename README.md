# CockroachDB 文档中文翻译

### 文件说明

- `v1.0/` 和 `releases/` 目录里的文件和 `design.md` 是原文。

  * `v1.0/` 和 `releases/` 目录的内容来自[Github repo](https://github.com/cockroachdb/docs.git)，用于生成[官网的 docs](https://www.cockroachlabs.com/docs/stable/)，是用户手册。格式较为复杂，嵌有 HTML 和 JavaScript。

  * `design.md` 是设计文档，是简单的 Markdown 格式。
  
  * `images/` 目录里是用户手册引用的图片文件，也用于译文。

  * `media` 和 `RFCS` 目录里是设计文档引用的图片文件。

- 中文译文

  * 相应放在 `v1.0\_cn/` 和 `releases/` 目录下（文件名不变）和 `design\_cn.md` 中。
  
  * `v1.0\_cn/images/` 目录存在上层的 `images` 目录中没有的图片文件，可以从[官网的 docs](https://www.cockroachlabs.com/docs/stable/) 页面截屏获得。
   
- **注意**：

  * 推荐使用可同时看 Markdown 格式渲染效果的编辑器。

  * 不要修改原文。

  * 注意对照[官网的 docs](https://www.cockroachlabs.com/docs/stable/) 页面检查译文 Markdown 文件的渲染效果。

  * 译文中的链接如果是指向本目录内的文件，将后缀由 html 改为 md，并且不需要加服务器域名和路径。
    
  * 原文中以形式`images/admin_ui_sql_queries.png`引用图形文件，在译文里可以改为`../images/admin_ui_sql_queries.png`。

- [`glossary.md`](glossary.md) 为词汇表。

- 中文译文以 gitbook 格式组织。

  在当前目录下运行命令：

```sh
   $ gitbook serve
```
	   然后在 http://localhost:4000 可以看到生成的 gitbook。
	
### 分工及进度

见[任务表](tasks.md)。
