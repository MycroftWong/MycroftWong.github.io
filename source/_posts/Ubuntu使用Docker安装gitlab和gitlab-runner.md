---
title: Ubuntu使用Docker安装gitlab和gitlab-runner
date: 2021-09-15 20:24:37
tags:
- Ubuntu
- Docker
- gitlab
- gitlab-runner
- CI/CD
---

`Ubuntu 20.04`使用`Docker`安装`gitlab`和`gitlab runner`实现`CI/CD`

## 安装配置gitlab

### 1. Docker 拉取 gitlab 镜像

```bash
docker pull gitlab/gitlab-ce:latest
```

### 2. 配置挂载文件目录

在环境变量中配置`GITLAB_HOME`，指向挂载文件目录的根目录：

```bash
export GITLAB_HOME=/usr/local/src/docker/gitlab
```

### 3. 创建gitlab容器

使用命令：

```bash
sudo docker run --detach \
  --hostname gitlab.mycroft.com \
  --env GITLAB_OMNIBUS_CONFIG="external_url 'http://gitlab.mycroft.com:9080'; gitlab_rails['gitlab_shell_ssh_port'] = 9022; gitlab_rails['smtp_enable'] = false; gitlab_rails['time_zone'] = 'Asia/Shanghai';" \
  --publish 9443:443 --publish 9080:9080 --publish 9022:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab \
  --volume $GITLAB_HOME/logs:/var/log/gitlab \
  --volume $GITLAB_HOME/data:/var/opt/gitlab \
  gitlab/gitlab-ce:latest
```

使用`docker-compose.yml`：

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
      gitlab_rails['smtp_enable'] = false             # 指定不使用smtp
      gitlab_rails['time_zone'] = 'Asia/Shanghai'     # 指定时区
  ports:
    - '9080:9080'
    - '9443:443'
    - '9022:22'
  volumes:
    - '$GITLAB_HOME/config:/etc/gitlab'
    - '$GITLAB_HOME/logs:/var/log/gitlab'
    - '$GITLAB_HOME/data:/var/opt/gitlab'
```

随后执行命令：`docker-compose up -d` 启动安装。

其中需要注意的点：

- 启动前的环境变量配置`GITLAB_OMNIBUS_CONFIG`查看[gitlab.rb.template](https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/files/gitlab-config-template/gitlab.rb.template)，实际上就是处理配置文件`gitlab.rb`里面的变量
- 启动时的`GITLAB_OMNIBUS_CONFIG`变量内容，并不会写入`gitlab.rb`配置文件，如果是命令启动，建议在启动后，手动修改`gitlab.rb`并重启`gitlab`。如果是`docker-compose`启动，则不用修改。
- 由于主机`ssh`一般占用了`22`端口，所以`gitlab`的`ssh`映射主机其他端口。因为配置中`gitlab_rails['gitlab_shell_ssh_port'] = 9022`修改的是`gitlab`的网页显示端口，实际的`ssh`仍然是`22`
- 挂载的目录：配置文件目录、日志文件目录、应用数据文件目录。
- `hostname`用于访问，一般我们会将其添加到`hosts`文件中方便访问，同时需要修改`gitlab`的配置文件，已便于后续操作

创建需要大约3分钟，可以使用命令查看日志

```bash
docker logs -f gitlab
```

### 4. 修改配置

如果使用命令安装，则需要修改配置

修改配置文件：

```bash
# $GITLAB_HOME/config/gitlab.rb
external_url 'http://gitlab.mycroft.com:9080/'  # 修改暴露的http url，但是不能添加9080，不然无法运行
gitlab_rails['gitlab_shell_ssh_port'] = 9022    # 修改暴露的ssh端口
gitlab_rails['smtp_enable'] = false             # 关闭smtp
gitlab_rails['time_zone'] = 'Asia/Shanghai'     # 设置时区
```

让配置生效：

```bash
# 让配置生效
docker exec -it gitlab bash
gitlab-ctl reconfigure
gitlab-ctl restart
exit

# 重启gitlab，配置并没有生效
docker restart gitlab
```

## 安装配置gitlab-runner

### 1. 安装gitlab-runner

使用命令：

```bash
docker run --detach \
  --add-host=gitlab.mycroft.com:192.168.0.30 \
  --name gitlab-runner \
  --restart always \
  --volume $GITLAB_HOME/runner:/etc/gitlab-runner \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

使用`docker-compose`：

```yml
version: '3'
services:
  gitlab-runner:
    image: 'gitlab/gitlab-runner:latest'
    container_name: 'gitlab-runner'
    volumes:
      - $GITLAB_HOME/runner:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
    extra_hosts:
      - "gitlab.mycroft.com:192.168.0.30"
```

### 2. 注册

使用`gitlab-runner`命令可以进行注册，如下所示，其中`url`是`gitlab`服务器`url`，`registration-token`是注册`token`，可能是特定的项目，也可能是共享的项目，`description`是名字，可以用于删除，`tag-list`是注册的表切列表，用于匹配合适的`runner`。

```bash
docker exec -it gitlab-runner \
  gitlab-runner register \
  --non-interactive \
  --executor "docker" \
  --docker-image alpine:latest \
  --url "http://gitlab.mycroft.com:9080/" \
  --registration-token "yZAtiZf_H24DCsmZYjGy" \
  --description "first-runner" \
  --tag-list "docker,test" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected"
```

由于我们上面命令指定的`executor`是`docker`，镜像是`alpine:latest`，所以实际会通过镜像`alpine:latest`创建临时的容器执行项目中的脚本。

然而我们是通过`hosts`配置的域名-`ip`解析，临时容器没有配置`hosts`，所以在拉取项目时，会出现错误：`Could not resolve host 'gitlab.mycroft.com'`。

解决方法是在`gitlab-runner`配置文件`$GITLAB_HOME/runner/config.toml`中为`runner`添加`hosts`，如下：

```toml
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "first-runner"
  url = "http://gitlab.mycroft.com:9080/"
  token = "XWtxoA5SVNzBEhS_NsbJ"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0

    # 新添加的hosts
    extra_hosts = ["gitlab.mycroft.com:192.168.0.30"]
```

## 参考

[GitLab Docker images](https://docs.gitlab.com/ee/install/docker.html)

[Configuration options](https://docs.gitlab.com/omnibus/settings/configuration.html)

[gitlab.rb.template](https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/files/gitlab-config-template/gitlab.rb.template)

[Docker-compose部署gitlab中文版](https://www.cnblogs.com/linuxk/p/10100431.html)

['unable to access 'XXX': Could not resolve host' Gitlab CI/CD pipeline](https://stackoverflow.com/questions/64452521/unable-to-access-xxx-could-not-resolve-host-gitlab-ci-cd-pipeline)

[Gitlab runner docker Could not resolve host](https://stackoverflow.com/questions/50325932/gitlab-runner-docker-could-not-resolve-host)

[gitlab-runner Could not resolve host](https://xmanyou.com/gitlab-runner-could-not-resolve-host/)

[add hosts redirection in docker](https://stackoverflow.com/questions/34242634/add-hosts-redirection-in-docker/34709445#34709445)

[Local Gitlab cicd failed 'fatal: unable to access...Could not resolve host:...' with linux runner](https://stackoverflow.com/questions/62239995/local-gitlab-cicd-failed-fatal-unable-to-access-could-not-resolve-host)