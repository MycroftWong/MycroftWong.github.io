---
title: gitlab配置SMTP总结
date: 2021-09-17 14:28:33
categories: CI/CD
tags:
- Docker
- gitlab
- CI/CD
---

# gitlab配置SMTP总结

自建`gitlab`有两种配置邮箱管理的方式：

1. `MTA(Mail Transport Agent)`，即搭建邮箱服务器，如`postfix`、`sendmail`
2. 使用`SMTP`服务器，即使用其他平台的注册邮箱。

`gitlab`的`Docker`镜像并不包含`MTA`，如果需要，建议在单独的容器中运行`MTA`，这样也易于升级。相比于搭建邮箱服务器，更推荐使用`SMTP`服务器，目前也有很多平台推出企业邮箱，所以直接使用企业邮箱即可。

由于我目前没有企业邮箱，下面我以`QQ`邮箱为例，进行配置。相比于传统的账号+密码的方式使用邮箱，现在很多平台退出了授权码，更加安全。所以这里使用`QQ`邮箱授权码，也更加符合真实场景。

## docker-compose配置QQ邮箱

直接看配置

```yml
web:
  image: 'gitlab/gitlab-ce:latest'
  container_name: 'gitlab'
  restart: always
  hostname: 'gitlab.mycroft.com'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'http://gitlab.mycroft.com:9080'   # 指定http host:port
      gitlab_rails['gitlab_shell_ssh_port'] = 9022    # 指定ssh port
      gitlab_rails['time_zone'] = 'Asia/Shanghai'     # 指定时区

      # SMTP设置
      gitlab_rails['smtp_enable'] = true                        # 开启SMTP
      gitlab_rails['smtp_address'] = "smtp.qq.com"              # SMTP服务器地址
      gitlab_rails['smtp_port'] = 465                           # SMTP服务器端口
      gitlab_rails['smtp_user_name'] = "mycroftwong@qq.com"     # 邮箱
      gitlab_rails['smtp_password'] = "xxxxxxxxxxxxxxxx"        # 授权码
      gitlab_rails['smtp_domain'] = "smtp.qq.com"               # 服务器域名
      gitlab_rails['smtp_authentication'] = "login"             # 认证方式
      gitlab_rails['smtp_enable_starttls_auto'] = true          # 自动启用starttls
      gitlab_rails['smtp_tls'] = true                           # 使用TLS
      gitlab_rails['smtp_pool'] = false                         # SMTP连接池

      # 邮箱设置
      gitlab_rails['gitlab_email_from'] = 'mycroftwong@qq.com'      # 发件人
      gitlab_rails['gitlab_email_display_name'] = 'gitlab admin'    # 显示的发件人

  ports:
    - '9080:9080'
    - '9443:443'
    - '9022:22'
  volumes:
    - '$GITLAB_HOME/config:/etc/gitlab'
    - '$GITLAB_HOME/logs:/var/log/gitlab'
    - '$GITLAB_HOME/data:/var/opt/gitlab'
```

## 邮箱配置

使用邮箱有三个目的：

- 登录
- 提交
- 接收事件触发邮件

### 登录

`gitlab`可配置是否允许用户注册，可在管理员设置中启用/关闭：

![注册限制](admin_enable_register.png)

如果不允许注册，那么将由管理员创建账号，创建账号时填写邮箱，自动认为验证成功：

![新建账号](admin_new_user.png)

但一般内部使用，不会启用用户注册。

管理员创建账号后为其设置密码，就可以使用邮箱登录了，`gitlab`会提示新用户修改密码。

### 提交

用户通过`git`配置`user.name`和`user.email`就可以配置提交的邮箱。

### 接收事件触发邮件

我们一般会希望团队合作时发送必要的邮件，如合并请求、`CI/CD`等。

## 管理员邮箱

目前没有找到在初始化安装时指定`root`用户的邮箱（默认为`admin@example.com`），如果需要修改`root`用户的邮箱，可以登录`root`账户，添加邮箱，并将新的邮箱设置为主邮箱。

## 参考

[SMTP settings](https://docs.gitlab.com/omnibus/settings/smtp.html)

[Does ActionMailer's "enable_starttls_auto" setting protect my email credentials when communicating with Gmail?](https://stackoverflow.com/questions/39320664/does-actionmailers-enable-starttls-auto-setting-protect-my-email-credentials)

[GitLab 配置邮箱](https://www.cnblogs.com/kika/p/10851615.html)

[How to setup a GitLab server using Docker](https://docs.bytemark.co.uk/article/how-to-setup-a-gitlab-server-using-docker/)