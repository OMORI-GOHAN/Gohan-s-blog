---
title: "博客迁移小记"
publishDate: 2026-09-09
description: "迁移博客遇到的诸多问题，以及debug了一下午的美丽精神状态"
tags: [博客, debug, 吐槽]

---

昨日，本人突发奇想，决定将博客站点从雨云的私人服务器上迁移至GitHub托管  
说干就干，~~既然都迁站了，换个外观也是很合理的吧~~  

于是我选择了Astro Kami 作为博客底子（在此也感谢大佬制作如此帅气的模板）

*然而Astro Kami 的多语言处理是以英文为默认的*

```
src/post/.md          → English
src/post/zh/.md       → Chinese

```

于是我改了下部分代码逻辑和文件分布，接着让路由也同步修改

```TypeScript
export const defaultLocale: Locale = "zh";
...

export function normalizeLocale(value?: string): Locale {
    const normalized = value?.toLowerCase();

    if (normalized === "en") return "en";

    return defaultLocale;
}

```

接着是迁移两篇博客，翻译为英文后添加到/en/，然后push到仓库，收工！！！

###真收工了吗？
由于我的网址是`https://omori-gohan.github.io/Gohan-s-blog/`  
但是我的静态资源上传给了`https://omori-gohan.github.io/_astro`  

我查阅代码发现上传BSE_PATH的代码
```YAML
BASE_PATH: ${{ steps.pages.outputs.base_path }}
```

正确的方向应该是让 GitHub Pages 提供 `base_path`，然后直接提交给Astro
于是：
```YAML
pnpm build --site "${{ steps.pages.outputs.origin }}" --base "${{ steps.pages.outputs.base_path }}"
```
*有个春竹一开始多打了一个 --，导致 --site 和 --base 没有按预期传给 Astro，我不说是谁……*

###你以为这就结束了吗？不，更折磨的才刚刚开始 

我来到线上后发现一个问题，文章内容更新了，但是不知道为啥，默认语言仍然是英语，切换lang之后变为中文  

我先切回本地检查`<HomePage locale="zh" />` 嗯，是对的
再看`<html lang="zh-CN">` 嗯，是对的

这就奇怪了，本地构建是对的。那就检查线上看看

增加个`CI debug`

```YAML
- name: Debug i18n config
  run: |
    echo "=== index.astro ==="
    cat src/pages/index.astro
    echo "=== i18n config ==="
    sed -n '1,130p' src/i18n/config.ts
    echo "=== git commit ==="
    git rev-parse HEAD
```
>在此吐槽一句这个死板的缩进
>由于一开始缩进不对导致我这里CI都出了问题

结果查看日志发现：`<HomePage locale="en" />`,英文。这就不对了，本地构建没问题，push到线上就变成en了
明明使用`git status`之后显示了clean

###为什么会这样？

还记得本地的astro显示的什么吗：`<AboutPage locale="zh" />`
然而:
```PowerShell
git show :src/pages/about.astro
```
却显示`<AboutPage locale="en" />` 不是哥们？？？？

```PowerShell
git hash-object src/pages/about.astro
git rev-parse HEAD:src/pages/about.astro
```
可以发现，工作区`Hash`和HEAD的`Hash`是不一样的

强制让Git从磁盘重新建立index
```PowerShell
git update-index --force-remove ...
git add ...
```

接着使用`git diff --cached`查看

```diff
-<AboutPage locale="en" />
+<AboutPage locale="zh" />

-<ShowcasePage locale="en" />
+<ShowcasePage locale="zh" />

-getPostsForLocale("en")
+getPostsForLocale("zh")

```
**终于啊，终于啊**，查明了 Git Index 没有反映工作区里已经发生的修改
至此，一切明了了：

```
这次其实同时遇到了两个独立的问题：

① Git / locale 问题
────────────────────────────────────────

本地 Working Tree
about.astro        → zh
showcase.astro     → zh
posts/[...page]    → zh
        │
        │ 但 Git Index 没有正确反映这些修改
        ▼
Git Index
about.astro        → en
showcase.astro     → en
posts/[...page]    → en
        │
        │ git commit / git push
        ▼
GitHub main
about.astro        → en
showcase.astro     → en
posts/[...page]    → en
        │
        │ GitHub Actions checkout
        ▼
Astro Build
        │
        └──────────→ 默认页面被构建成 English
                         │
                         ▼
                    GitHub Pages
                         │
                         └──→ 首页 / 文章 / 项目 / 关于
                              部分页面都是 English


② GitHub Pages / base path 问题
────────────────────────────────────────

GitHub Pages
实际网址：
https://omori-gohan.github.io/Gohan-s-blog/
        │
        │
        ▼
Astro 最开始生成资源：
/_astro/...
/rss.xml
/posts/...
        │
        │ 实际应该位于 /Gohan-s-blog/ 下
        ▼
CSS / JS 请求路径错误
        │
        ▼
资源加载失败
        │
        ▼
线上页面只剩裸 HTML / 文字
```

>查看日志后，我注意到文件还存在 BOM / 编码方面的差异。虽然不能确定这就是 Index 异常的唯一原因，
>但为了避免以后继续踩 Windows 换行符和文本格式的坑，我顺手加了一个 .gitattributes，统一一下文本文件的处理规则。

```gitattributes
* text=auto
*.astro text eol=lf
*.ts text eol=lf
*.js text eol=lf
*.json text eol=lf
*.md text eol=lf
*.mdx text eol=lf
*.yml text eol=lf
*.yaml text eol=lf
```

**终于，一个双语 Astro 静态博客的完整工程化部署，在经历了一次艰难的 CI/CD 排错后宣布完工！！！**