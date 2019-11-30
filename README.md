# Working with Jekyll with my personal blog

This project is the source code for my personal blog generation. So, everything written here is aimed to help myself to keep using Jekyll even after long perioeds :)

## Develop mode

To work at development mode just export the following env variable followed by some jekyll commands to rebuild local version of the blog:

```
export JEKYLL_ENV=development
jekyll clean
jekyll build
```

## Publishing to production

Before publishing to github pages, make sure to build the blog properly with the following commands:

```
export JEKYLL_ENV=production
jekyll clean
jekyll build
```

## Serving drafts locally

```
bundle exec jekyll serve --watch --drafts
```

## Start the server locally

To star the server just run

```
jekyll serve
```

and go to [http://localhost:4000/](http://localhost:4000/)
