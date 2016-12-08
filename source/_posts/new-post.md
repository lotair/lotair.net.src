---
title: New Post
date: 2016-12-08 03:51:08
author: sel-fish
tags:
  - new post
categories:
  - Tutorial
---

## Init

```
su lotair
cd ~/lotair.net

hexo new new-post
INFO  Created: ~/lotair.net/source/_posts/new-post.md
```

<!-- more -->

## Edit body

```
vim ~/lotair.net/source/_posts/new-post.md

...
author: sel-fish
tags:
  - new post
categories:
  - Tutorial
...
```

## Preview

```
hexo s

INFO  Start processing
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```

visit http://lotair.net:4000 to preview

## Deploy

```
./hexo_git_deploy.sh
```

