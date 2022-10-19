---
title: "Mysql Findinset Multiplevalues"
date: 2022-10-19T10:54:13+08:00
tags: ['mysql']
---

```sql
/*
Something likes find_in_set('a,b,c', 'a,b,c,d')
We can use OR:
find_in_set('a', 'a,b,c,d') OR find_in_set('b', 'a,b,c,d') OR find_in_set('b', 'a,b,c,d')
*/
WHERE CONCAT(",", `setcolumn`, ",") REGEXP ",(val1|val2|val3),"
```

