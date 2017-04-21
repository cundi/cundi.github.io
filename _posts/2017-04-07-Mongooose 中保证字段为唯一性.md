---
layout: post
title:  "Mongooose 中保证字段为唯一性"
categories: Mongoose
tags: Docker Dockerfile 代理 Git
---

* content
{:toc}




Mongooose 中保证字段为唯一性，

Use dropDups to ensure dropping duplicate records in your schemas like;

var SimSchema = new Schema({
    msisdn     : { type : String , unique : true, required : true, dropDups: true },
    imsi       : { type : String , unique : true, required : true, dropDups: true },
    status     : { type : Boolean, default: true},
    signal     : { type : Number },
    probe_name : { type:  String , required : true }
});
And before running your tests, restart mongodb
