# Jekyll and GitHub Pages

Setting up a new site -- working 26th June 2017.

1. Create repository in github with readme.  Name/description don't matter.
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
8. Edit `_config.yml` to set `title:` and `description:`
9. `bundle exec jekyll serve` to check all is well. 
1. Commit everything, push.
