---
title: Spring bug解决方案
date: 2019-08-10 17:14:35
categories:
- spring boot
    - bug
tags:
- spring boot
- bug
- mail
---

# 这是在实际过程中遇到的bug及解决方案

### 1. QQ邮件发送错误

在本地测试发送邮件没有问题，打包发送到服务器之后，无法发送邮件

问题：端口修改成465，出现错误`spring boot Could not connect to SMTP host: smtp.xxx.com, port: 465, response: -1`

参考[spring boot Could not connect to SMTP host: smtp.xxx.com, port: 465, response: -1](https://www.codeleading.com/article/54261413643/)

解决：在`properties`配置中加上
```properties
spring.mail.properties.mail.smtp.ssl.enable=true

# 完整如下
spring.mail.host=smtp.qq.com
spring.mail.username=XXXX@qq.com
spring.mail.password=密码
spring.mail.protocol=smtp
spring.mail.properties.mail.smtp.port=465
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.ssl.enable=true
spring.mail.default-encoding=utf-8
```

### 2. 访问`http://localhost:8080/druid`需要用户名和密码

问题：使用`druid`监控，访问监控网页时，需要输入用户名和密码

解决：在`application.properties`中配置账号密码

```properties
spring.datasource.druid.stat-view-servlet.login-username=用户名
spring.datasource.druid.stat-view-servlet.login-password=密码
```

