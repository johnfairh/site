---
title: Jekyll and GitHub Pages
---
# {{ page.title }}

## New site

Assume the repo name, forms path under <i>user</i>.github.io, is <i>SITE</i>.

Working 26th June 2017.

1. Create repository in github with readme.  Name is SITE for example purposes
   (but does not affect web content); description doesn't matter.
2. Repository settings -> pick theme.
3. Clone repo to mac.
4. Create `.gitignore`:

        .DS_Store
        _site/
        *.swp

5. Set ruby version to at least 2.2.
6. Create `Gemfile`:

        source 'https://rubygems.org'
        gem 'github-pages', group: :jekyll_plugins

7. `bundle install`
8. Edit `_config.yml` to set `title:` and `description:`.  Add `baseurl: SITE`.
9. `bundle exec jekyll serve` to check all is well. 
1. Commit everything, push.

## Everyday

* `bundle exec jekyll serve [--detach]` watches for changes to most files,
   specifically not the `_config.yml` though.
* Default URL is [`http://127.0.0.1:4000/SITE/`](http://127.0.0.1:4000/site/).

## Markup

* [Jekyll docs](http://jekyllrb.com/docs/frontmatter/).
* Use {% raw %} `{{ site.url }}{{ site.baseurl }}` {% endraw %} as root context in URLs.
