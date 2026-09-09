---
title: "A Brief Note on Migrating My Blog"
publishDate: 2026-09-09
description: "A note on the many problems I ran into while migrating my blog, and the beautiful mental state I reached after debugging for an entire afternoon"
tags: [Blog, Debugging, Rant]

---

Yesterday, for some reason, I suddenly decided to migrate my blog from a private server on Rainyun to GitHub.  
And once I had decided to do it, I figured, ~~since I'm migrating the site anyway, changing the whole look is perfectly reasonable, right~~

So I chose Astro Kami as the foundation of my new blog (and I'd also like to thank the author for making such a damn nice template).

*However, Astro Kami handles multilingual content with English as the default language.*

```text
src/post/.md          → English
src/post/zh/.md       → Chinese
```

So I changed some of the code and reorganized the file structure, then adjusted the routes accordingly.

```TypeScript
export const defaultLocale: Locale = "zh";
...

export function normalizeLocale(value?: string): Locale {
    const normalized = value?.toLowerCase();

    if (normalized === "en") return "en";

    return defaultLocale;
}
```

Then came the migration of my two blog posts. I translated them into English, added them under `/en/`, pushed everything to the repository, and called it a day!!!

### Was I really done?

My website is:

`https://omori-gohan.github.io/Gohan-s-blog/`

But my static assets were being served from:

`https://omori-gohan.github.io/_astro`

I checked the code and found this:

```YAML
BASE_PATH: ${{ steps.pages.outputs.base_path }}
```

So the correct approach should be to let GitHub Pages provide `base_path` and pass it directly to Astro.

Thus:

```YAML
pnpm build --site "${{ steps.pages.outputs.origin }}" --base "${{ steps.pages.outputs.base_path }}"
```

*Some genius accidentally typed an extra `--` at first, which caused `--site` and `--base` not to be passed to Astro as intended. I won't say who...*

### You thought that was the end? Nope. The real suffering had only just begun.

After visiting the deployed site, I found another problem. The article content had been updated, but for some reason, the default language was still English. Switching the language with Lang would turn it into Chinese.

So I went back to my local project and checked:

`<HomePage locale="zh" />`

Yep, correct.

Then:

`<html lang="zh-CN">`

Yep, correct.

Now that's weird.

The local build was correct, so let's check what was happening on the server.

I added a little `CI debug`:

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

> And let me complain about this stupidly strict indentation for a moment.  
> My CI broke at this point because I got the indentation wrong the first time.

Then I checked the logs and found:

`<HomePage locale="en" />`

English.

That's not right.

The local build was fine, but after pushing to the remote it somehow became `en`.

And `git status` had literally told me everything was clean.

### Why did this happen?

Remember what my local Astro file showed?

`<AboutPage locale="zh" />`

However:

```PowerShell
git show :src/pages/about.astro
```

showed:

`<AboutPage locale="en" />`

Bro, what the hell????????

Then:

```PowerShell
git hash-object src/pages/about.astro
git rev-parse HEAD:src/pages/about.astro
```

showed that the `Hash` of the file in my Working Tree was different from the `Hash` in `HEAD`.

So I forcibly made Git rebuild the index from the files on disk:

```PowerShell
git update-index --force-remove ...
git add ...
```

Then I used `git diff --cached` to check the changes:

```diff
-<AboutPage locale="en" />
+<AboutPage locale="zh" />

-<ShowcasePage locale="en" />
+<ShowcasePage locale="zh" />

-getPostsForLocale("en")
+getPostsForLocale("zh")
```

**Finally. FINALLY.**

We figured it out: Git's Index was not reflecting the changes that had already happened in the Working Tree.

And everything finally made sense:

```text
This time, I actually ran into two completely independent problems:

① Git / locale problem
────────────────────────────────────────

Local Working Tree
about.astro        → zh
showcase.astro     → zh
posts/[...page]    → zh
        │
        │ But the Git Index did not correctly reflect these changes
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
        └──────────→ Default pages were built in English
                         │
                         ▼
                    GitHub Pages
                         │
                         └──→ Home / Posts / Showcase / About
                              were partially displayed in English


② GitHub Pages / base path problem
────────────────────────────────────────

GitHub Pages
Actual URL:
https://omori-gohan.github.io/Gohan-s-blog/
        │
        │
        ▼
Astro initially generated assets as:
/_astro/...
/rss.xml
/posts/...
        │
        │ They should actually live under /Gohan-s-blog/
        ▼
CSS / JS request paths were wrong
        │
        ▼
Assets failed to load
        │
        ▼
The page was left as bare HTML / plain text
```

> Looking through the logs, I also noticed that the files had differences related to BOM / encoding. Although I can't say for certain that this was the only cause of the Index problem, I decided to add a `.gitattributes` file anyway, just to standardize how text files are handled and hopefully avoid more Windows line-ending nonsense in the future.

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

**Finally, finally, finally. A bilingual Astro static blog had made it through a painful CI/CD debugging session and was officially complete!!!**