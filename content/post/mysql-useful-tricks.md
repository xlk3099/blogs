---
title: "Mysql Useful Tricks" 
date: 2018-07-26T14:02:00+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["tricks"]
---

> mysql 得到各databases的大小

    SELECT table_schema "DB Name",
        ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) "DB Size in MB" 
    FROM information_schema.tables 
    GROUP BY table_schema; 

> mysql 得到各个table的大小
    SELECT 
    table_name AS "Table",  
    round(((data_length + index_length) / 1024 / 1024), 2) as size   
    FROM information_schema.TABLES  
    WHERE table_schema = "YOUR_DATABASE_NAME"  
    ORDER BY size DESC; 

mysql 5.7 在macos 改密码须知：
>  mysql> UPDATE user SET authentication_string=PASSWORD("root") WHERE User='root';

mysql 8.0 改密码：
> set PASSWORD ='your_password'  // 给当前用户改密码
或者
> SET PASSWORD FOR 'USER'@'HOST' ='your_password';

Clean mysql completely

```
sudo rm /usr/local/mysql
sudo rm -rf /usr/local/var/mysql
sudo rm -rf /usr/local/mysql*
sudo rm ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist
sudo rm -rf /Library/StartupItems/MySQLCOM
sudo rm -rf /Library/PreferencePanes/My*
```


> MySQL 提升写入速度 几个tips
AWS RDS MySQL 服务器很多参数已经是动态优化了，因此多余的参数设置也不多提，但是还是有些参数需要优化

1. 批量插入，插入时注意尽量避开键值冲突
2. 多线程插入，MySQL 5.7 32个链接同时插入速度是峰值（读的话是64）， 8.0以后性能有提升，64比较快
3. `max_allowed_packet` size, 默认5M左右， 设成1G 比较好，便于批量插入
4. 将`innodb_flush_log_at_trx_commit` 设置为0
5. 默认`innodb_buffer_pool_size` 设为可用内存的 70%~80%， 也就是说 16G的内存可以设为 **12G**
6. RDS 将transaction isolation 修改成 read commited 有助于减少gaps lock
7. RDS 增加innodb_sort_buffer_size有助于提升大表索引创建速度
8. 索引最后添加
