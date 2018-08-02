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