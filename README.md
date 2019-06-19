# DevOps
docker快速搭配一套ci/cd的环境

![img](https://github.com/cyneck/DevOps/blob/master/clipboard.png)

**安装gitlab-ce代码仓库**

```bash
docker run -d --name gitlab-ce-zh \
    -p 10080:80 \
    -p 10443:443 \
    -p 10022:22 \
    --restart unless-stopped \
    -v gitlab-config:/etc/gitlab \
    -v gitlab-logs:/var/log/gitlab \
    -v gitlab-data:/var/opt/gitlab \
    twang2218/gitlab-ce-zh
```

**安装CI/CD工具**

**Jenkins** 

（使用配置，请自行查找学习）

```bash
docker run -d --name jenkins \
    -p 18080:8080 \
    jenkins/jenkins:lts
```

**（或者gitlab-runner）**

gitlab-ce最新版本的已经默认包含。

**（或者teamcity）**

启动过程中，初始化会用到postgres数据库，先创建数据库

```bash
docker pull sonarqube
docker run --name postgres \
     -e POSTGRES_USER=sonar \
     -e POSTGRES_PASSWORD=sonar \
     -p 5432:5432 \
     -d postgres
```

再创建teamcity-server

```bash
docker run -it -d --name teamcity-server \
    -v /teamcity/datadir:/data/teamcity_server/datadir \
    -v /teamcity/logs:/opt/teamcity/logs  \
    -p 18111:8111 \
    jetbrains/teamcity-servers
```

Server创建好了，我们还需要创建TeamCity Build Agent来为我们构建代码。

```bash
docker run -it \
    -e SERVER_URL="http://192.168.88.128:18111" \
    -v /teamcity/conf:/data/teamcity_agent/conf \
    -v docker_volumes:/var/lib/docker \
    --privileged -e DOCKER_IN_DOCKER=start \
    jetbrains/teamcity-agent
```

**安装sonar（代码静态检查工具）**

快速体验： 账号：admin 密码：admin

```bash
docker pull postgres
docker pull sonarqube
docker run --name postgres \
     -e POSTGRES_USER=sonar \
     -e POSTGRES_PASSWORD=sonar \
     -p 5432:5432 \
     -d postgres
docker run --name sonarqube \
     --link postgres \
    -e sonar.jdbc.username=sonar \
    -e sonar.jdbc.password=sonar \
    -e sonar.jdbc.url=jdbc:postgresql://postgres:5432/sonar \
    -p 19000:9000 \
    -d sonarqube
```

或者通过配置sonar.properties来设置

```bash
docker run -d --name sonarqube \
    -p 19000:9000 \
    -v /path/to/conf:/opt/sonarqube/conf \
    -v /path/to/data:/opt/sonarqube/data \
    -v /path/to/logs:/opt/sonarqube/logs \
    -v /path/to/extensions:/opt/sonarqube/extensions \
    sonarqube
```

或者docker-compose.yml文件安装，通过执行

```bash
docker-compose up -d
```

启动服务

docker-compose.yml文件内容

```bash
version: "2"

services:
  sonarqube:
    image: sonarqube
    ports:
      - "19000:9000"
    networks:
      - sonarnet
    environment:
      - SONARQUBE_JDBC_URL=jdbc:postgresql://db:5432/sonar
    volumes:
      - sonarqube_conf:/opt/sonarqube/conf
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins

  db:
    image: postgres
    networks:
      - sonarnet
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
    volumes:
      - postgresql:/var/lib/postgresql
      - postgresql_data:/var/lib/postgresql/data

networks:
  sonarnet:
    driver: bridge

volumes:
  sonarqube_conf:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_bundled-plugins:
  postgresql:
  postgresql_data:
```

