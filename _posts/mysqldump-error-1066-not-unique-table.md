---
title: mysqldump_error_1066_not_unique_table
date: 2017-02-09 15:01:54
tags:
- MySQL
---


```
mysqldump: Got error: 1066: Not unique table/alias
```

myql 导出时提示如下：

```
[root@localhost MySQL]# mysqldump  -uroot  -p 123456  test >test_bak
mysqldump: Got error: 1066: Not unique table/alias: 'robots_excludeurl' when using LOCK TABLES
```

修改/etc/my.cnf，将下面这行用#注释掉即可：
`lower_case_table_names=1`（等于1表示不区分表名大小写）
注释掉后，重启mysql：

```
service  mysql  restart
```

再导出，好了。