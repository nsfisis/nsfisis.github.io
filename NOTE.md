# My note

## Commands

Generate the site.

```
$ hugo -d docs
```

Create a new post.

```
$ hugo new posts/$(date +'%Y-%m-%d')/[TITLE].md
```


Update `highlight.min.js`.

```
$ curl -sL https://raw.githubusercontent.com/highlightjs/cdn-release/main/build/highlight.min.js >| themes/mypaper/static/highlight.min.js
```
