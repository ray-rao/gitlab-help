## GitLab 自动化部署环境搭建

##### GitLab安装

```
1. docker search gitlab
2. docker pull gitlab/gitlab-ce
3. touch startup.sh
#!/bin/bash
docker run -d  -p 8443:443 -p 8080:80 -p 2222:22 --name gitlab \
--restart always \
-v /home/gitlab/config:/etc/gitlab \
-v /home/gitlab/logs:/var/log/gitlab 
-v /home/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce \
# -d：后台运行
# -p：将容器内部端口向外映射
# --name：命名容器名称
# -v：将容器内数据文件夹或者日志、配置等文件夹挂载到宿主机指定目录


# 用./startup.sh启动时候提示权限不够
chmod u+x *.sh
```

##### 安装 gItlab-runner

```
1. docker search gitlab-runner
2. docker pull gitlab/gitlab-runner
3. touch startup.sh
#!/bin/bash
sudo docker run -d \
  --name gitlab-runner --restart always \
  --user root\
  --add-host gitlab.jskj.com:192.168.1.117 \
  -v /home/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /bin/docker:/bin/docker \
  -v /usr/local/bin/docker-compose:/usr/local/bin/docker-compose \
  -v /home/jskj/deploy:/home/deploy \
  --privileged=true \
  gitlab/gitlab-runner:latest
```



##### 注册 gitlab-runner tags 要与.gitlab-ci.yml 对上

```
1. docker exec -it gitlab-runner gitlab-runner register
- /xx/gitlab-runner/config/config.toml 

-- 参考内容
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "java"
  url = "http://gitlab.jskj.com/"
  token = "HFFC7b3hmroGeB1C3rHo" // gitlab 设置runner下面的token
  executor = "docker"
  clone_url ="http://192.168.1.117:8090" // 部署host解析不了 增加ip访问
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "docker:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache","/var/run/docker.sock:/var/run/docker.sock","/data/.m2/:/.m2/","/etc/hosts:/etc/hosts"]
    pull_policy = "if-not-present"
    shm_size = 0
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```

#####  .gitlab-ci.yml 

```
stages:
- build
- deploy

variables:
  JIB_OPTS: "-Djib.baseImageCache=/root/.jib/base -Djib.applicationCache=/root/.jib/app -DsendCredentialsOverHttp=true"

build-docker:
  stage: build
  image: registry.jskj.com/mydocker/jskj-maven
  tags:
  - java
  script:
  - "mvn clean $JIB_OPTS -B -X compile jib:build"

build-deploy:
  stage: deploy
  tags:
  - java-shell
  before_script:
  - "echo $CI_REGISTRY_USER"
  - "echo $CI_REGISTRY_PASSWORD"
  - "echo $CI_REGISTRY"
  - "echo $UID $USER"
  - "docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY"
  script:
 # - "cd /home/deploy"
 # - "stat .git/FETCH_HEAD"
  - "cd /home/deploy/ci/"
  - "git pull"
  - "docker-compose pull $CI_PROJECT_NAME"
  - "docker-compose stop $CI_PROJECT_NAME"
  - "docker-compose rm $CI_PROJECT_NAME"
  - "docker-compose up -d $CI_PROJECT_NAME"
```

##### docker-compose.yml

```
version: "3"
services:
  fe-client:
    image: registry.jskj.com/frontend/fe-client:latest
    container_name: fe-client
    restart: always
    environment:
      TZ: Asia/Shanghai
    ports:
      - 8100:8100
    network_mode: host
  qdjy-svc-eureka:
    image: registry.jskj.com/mydocker/qdjy-svc-eureka:latest
    container_name: qdjy-svc-eureka
    hostname: eureka-server
    ports:
      - 8761:8761
    network_mode: host
    restart: always
  qdjy-svc-gateway:
    image: registry.jskj.com/mydocker/qdjy-svc-gateway:latest
    container_name: qdjy-svc-gateway
    ports:
      - 5020:5020
    depends_on:
      - qdjy-svc-eureka
    network_mode: host
    restart: always
  qdjy-svc-config:
    image: registry.jskj.com/mydocker/qdjy-svc-config:latest
    container_name: qdjy-svc-config
    ports:
      - 5010:5010
    network_mode: host
    depends_on:
      - qdjy-svc-eureka
      - qdjy-svc-gateway
    restart: always
  qdjy-svc-auth:
    image: registry.jskj.com/mydocker/qdjy-svc-auth:latest
    container_name: qdjy-svc-auth
    ports:
      - 6001:6001
    network_mode: host
    depends_on:
      - qdjy-svc-eureka
      - qdjy-svc-gateway
    restart: always
  qdjy-svc-user:
    image: registry.jskj.com/mydocker/qdjy-svc-user:latest
    container_name: qdjy-svc-user
    ports:
      - 6002:6002
    network_mode: host
    depends_on:
      - qdjy-svc-eureka
      - qdjy-svc-gateway
    restart: always
  qdjy-svc-category:
    image: registry.jskj.com/mydocker/qdjy-svc-category:latest
    container_name: qdjy-svc-category
    ports:
      - 6009:6009
    network_mode: host
    restart: always
  qdjy-svc-repository:
    image: registry.jskj.com/mydocker/qdjy-svc-repository:latest
    container_name: qdjy-svc-repository
    ports:
      - 6010:6010
    network_mode: host
    restart: always
  qdjy-svc-lesson:
    image: registry.jskj.com/mydocker/qdjy-svc-lesson:latest
    container_name: qdjy-svc-lesson
    ports:
      - 6011:6011
    network_mode: host
    restart: always
  qdjy-svc-file:
    image: registry.jskj.com/mydocker/qdjy-svc-file:latest
    container_name: qdjy-svc-file
    ports:
      - 6012:6012
    network_mode: host
    restart: always
  qdjy-svc-socket:
    image: registry.jskj.com/mydocker/qdjy-svc-socket:latest
    container_name: qdjy-svc-socket
    ports:
      - 6015:6015
    network_mode: host
    restart: always
```

#####  nexus 增加docker私人仓库和构建maven仓库

```
1. 去nexus添加docker类型的仓库
2. 上传settings.xml到服务器
3. touch Dockerfile
4. touch build.sh
```

###### settings.xml

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">

    <localRepository>/usr/share/maven/ref/repository</localRepository>

    <pluginGroups>

    </pluginGroups>

    <proxies>
    </proxies>

    <servers>
        <server>
            <id>jskj-releases</id>
            <username>admin</username>
            <password>jskj@123</password>
        </server>
        <server>
            <id>jskj-snapshots</id>
            <username>admin</username>
            <password>jskj@123</password>
        </server>
        <server>
            <id>jskj-public</id>
            <username>admin</username>
            <password>jskj@123</password>
        </server>
        <server>
            <id>registry.jskj.com</id>
            <username>admin</username>
            <password>jskj@123</password>
        </server>

    </servers>

    <mirrors>

    </mirrors>

    <profiles>
        <profile>
            <id>test-profile</id>
            <repositories>
                <repository>
                    <id>jskj-releases</id>
                    <url>http://192.168.1.136:8081/repository/maven-public/</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                </repository>
                <repository>
                    <id>jskj-snapshots</id>
                    <url>http://192.168.1.136:8081/repository/maven-public/</url>
                    <releases>
                        <enabled>false</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <id>jskj-public</id>
                    <url>http://192.168.1.136:8081/repository/maven-public/</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>

    <activeProfiles>
        <activeProfile>test-profile</activeProfile>
    </activeProfiles>
</settings>

```

###### Dockerfile

```
FROM maven:3.6.1-jdk-8

COPY settings.xml /usr/share/maven/ref/
CMD ["mvn"]
```



###### build.sh
```
#!/bin/bash

docker build --tag jskj/jskj-maven:3.6.1-jdk-8 .
docker tag jskj/jskj-maven:3.6.1-jdk-8 192.168.1.136:8082/mydocker/jskj-maven:latest
docker push 192.168.1.136:8082/mydocker/jskj-maven
```





##### 补充

```
gitlab 变量在项目组下的 CI/CD 项配置
git 文件权限：chown -R polkitd:ssh_keys xxx
```

