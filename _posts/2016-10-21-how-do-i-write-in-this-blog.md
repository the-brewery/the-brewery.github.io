---
layout:     post
title:      How do I write in this blog?
tags:       blog jekyll github github-pages ruby markdown
categories: howtos
author:     Olof Montin
---

This blog is a static generated website. It's simply a set of markdown files that automagicly turns into a blog when deployed.

So, how do I use it then?
-------------------------

### Run the generator locally

1. Install Ruby 2.x if you don't already have it: `ruby --version`
2. Install Bundler: `gem install bundler`
3. Make a clone of the website's repository: `git clone git@github.com:the-brewery/the-brewery.github.io.git`
4. Install dependencies: `bundle install --path vendor/bundle`
5. Run the website generator with a watch locally: `bundle exec jekyll serve`

### Write a post

1. Place a file in the `_posts` directory with name pattern: `YEAR-MONTH-DAY-title.md`
2. Write a [front matter](http://jekyllrb.com/docs/frontmatter/) in the file's header:

    ```yaml
    ---
    layout:     post
    title:      Your title
    tags:       some-tag another-tag
    categories: some-category
    author:     Your Name
    ---
    ```
3. Publish you stuff with git:

    ```bash
    git add _post/your-post.md
    git commit -m 'Post about something'
    git push
    ```

4. If unsure, [read the docs](http://jekyllrb.com/docs/posts/).
