# CedarlandBlog
This is the source for https://cedarland.blog

## How do I add or edit content?
1. Install `mdbook` if you haven't already
 * Instructions at [https://rust-lang.github.io/mdBook/guide/installation.html](https://rust-lang.github.io/mdBook/guide/installation.html)
2. Add the new blog content under `CedarlandBlog/blog/src`
 * Each blog post has its own folder, containing the blog content (named `content.md`), plus any image files.
 * Tool Tip: From the directory `CedarlandBlog/blog`, you can run `mdbook serve -o` to preview your content as you write it.
3. If adding a new blog post...
 * Update the table of contents at `CedarlandBlog/blog/src/SUMMARY.md`
 * Update the "Recent Posts" list at the bottom of `CedarlandBlog/blog/src/about.md`
 * See existing examples inside those files. Should be self-evident.
4. When ready to publish, generate HTML from the source files by following these steps
 * `cd` into the directory for `CedarlandBlog/blog`
 *  Run `mdbook build -d ../docs`
5. Push the new content into GitHub with the normal commands
 * `git add -A`
 * `git commit -m "Blog update"`
 * `git push`
6. Wait a minute or so...
 * The blog is hosted from GitHub pages, so it should automatically update after pushing new content to GitHub. Due to caching, it can take a minute or so.