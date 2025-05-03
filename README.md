# Working with Jekyll with my personal blog

This project is the source code for my personal blog generation. So, everything written here is aimed to help myself to keep using Jekyll even after long perioeds :)

## Local setup

Jekyll [uses Ruby version 2.7.0 or higher](https://jekyllrb.com/docs/).
This repository uses `rbenv` in order to use the latest Ruby version.

## Develop mode

To work at development mode just export the following env variable followed by some jekyll commands to rebuild local version of the blog:

```
export JEKYLL_ENV=development
bundle exec jekyll clean
bundle exec jekyll build
```

## Publishing to production

Before publishing to github pages, make sure to build the blog properly with the following commands:

```
export JEKYLL_ENV=production
bundle exec jekyll clean
bundle exec jekyll build
```

## Serving drafts locally

```
bundle exec jekyll serve --watch --drafts
```

## Start the server locally

To star the server just run

```
bundle exec jekyll serve
```

and go to [http://localhost:4000/](http://localhost:4000/)

## Important links to consider when changing the defaults

- [Liquid](https://shopify.github.io/liquid/filters/split/), the templating language used by Jekyll;
- [Jekyll cheatsheet](https://devhints.io/jekyll)
- [Front Matter](https://jekyllrb.com/docs/front-matter/), content that when added as the header into posts, pages, etc makes the file to be a special file for Jekyll (Jekyll will process the front matter content).
