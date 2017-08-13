# CockroachDB 文档中文翻译

### 文件说明

- v1.0 目录里的文件和 design.md 是原文。


  - v1.0 目录的内容来自https://github.com/cockroachdb/docs/tree/master/v1.0/ 用于生成官网的[docs](https://www.cockroachlabs.com/docs/stable/)，是用户手册。格式较为复杂，嵌有 HTML 和 JavaScript。

  - design.md 是设计文档，是简单的 Markdown 格式。
  
  **注意**：

    - 不要修改原文，相应的中文译文放在 v1.0\_cn 目录（文件名不变）和 design\_cn.md 中。
  
    - 译文中的链接如果是指向本目录内的文件，将后缀由 html 改为 md，并且不需要加服务器域名和路径。

- 其他引用的文件
  
  - images 目录里是用户手册引用的图片文件。

  - media 和 RFCS 目录里是设计文档引用的图片文件。

- 中文译文以 gitbook 格式组织。

  在当前目录下运行命令：

```sh
   $ gitbook serve
```
	   然后在 http://localhost:4000 可以看到生成的 gitbook。
	
- [glossary.md](glossary.md) 为词汇表。

### 分工及进度

见[任务表](tasks.md)。
