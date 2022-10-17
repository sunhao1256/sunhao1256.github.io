---
title: "Essential Linux Command"
date: 2022-10-17T15:32:10+08:00
tags: ['Linux']
---

# awk

```shell
docker images|grep gateway| awk '$2 ~ /^v/ && substr($2,2)>535  {print($3)}' |sort -r
```

- started with 'v'
- version number greater than 535
- print imageId
