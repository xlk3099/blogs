---
title: "Mysql Useful Tricks"
date: 2018-04-25T14:02:00+08:00
draft: true
author: "xlk3099"
categories: ["golang"]
tags: ["tricks"]
---

> mysql 得到各databases的大小

    SELECT table_schema "DB Name",
        ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) "DB Size in MB" 
    FROM information_schema.tables 
    GROUP BY table_schema; 