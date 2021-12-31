# README

This repo is used to build a personal blog using [hugo](https://gohugo.io/getting-started/installing/).

## Prepare Environments

On macOS just a single line to install hugo

```bash
brew install hugo
```

Notice that this repo contains submodules, so remember to do the following things when cloning the repo

```bash
git clone git@github.com:lsongseven/blog.git
cd blog
git submodule init && git submodule update
```


## How to Post Blogs

1. place your markdown file in `content/posts/` folder and also put the static files in `static/` folder

2. use `hugo server` to debug at local environment

3. if everything is ok, type `hugo` in root path of repo, update the submodule

4. push updates of this repo and submodule to remote

## Reference

- <https://segmentfault.com/a/1190000040475072>
  