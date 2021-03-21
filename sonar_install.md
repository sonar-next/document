介绍

本文将介绍 SonarQube 的安装和运行，以下主要使用，SonarQube架构上主要分为客户端和服务端。服务端为 SonarQube ，客户端为 SonarScanner 。其中 SonarScanner 可以嵌入到某些构建工具中，比如 mvn 和 gradle。

## 安装 SonarQube

Docker 方式来进行安装

## Docker

建议直接使用 docker 方式进行安装，SonarQube支持外部数据库的设置，最新版本已经不支持 MySQL 数据库，所以我们可以使用内置的H2数据库或者是 Postgresql、SQL Server、Oracle，在这里我推荐演示 Demo 时直接使用 内置数据库，线上运行时使用 Postgresql 数据库，线上运行时可使用 `docker-compose` ，我已经整理了相关配置信息，放置在 https://github.com/sonar-next/docker ，欢迎浏览。

### docker 方式 + 内置数据库

该方式仅供Demo演示使用

```jsx
docker pull sonarqube:lts
docker run --name sonarqube -p 9000:9000 -d
# 然后访问 localhost:9000
```

### docker-compose

使用 docker-compose 一键部署 SonarQube 和 Postgresql 数据库

该 Dockerfile 内置了主流的语言插件，例如官方社区版本不支持 C++/Swift/Objc/Flutter等语言，该Dockerfile均已内置

```jsx
git clone https://github.com/sonar-next/docker.git
cd docker
# 修改 POSTGRES_PASSWORD 和 SONARQUBE_JDBC_PASSWORD 为你自己的密码。
# Postgresql 数据库的数据默认会挂载在执行目录下
# 启动
docker-compose up -d
```

## SonarScanner

如果是Java语言，优先选择 mvn 和 gradle ，因为会分析字节码，其他语言请选择 sonar-scanner

### sonar-scanner

sonar提供了2种安装方式，提供平台特供版本或者基于jvm的版本，通常我们的电脑上装有Java语言，所以我们可以直接使用基于jvm的版本，下载地址：

https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/

选择 Any (Requires a pre-installed JVM) 下载即可。

### mvn

mvn 版本无需额外下载，执行 mvn sonar:sonar 即可运行，运行前请提前编译项目。

### gradle

gradle 也无需提前下载 scanner，需要设置插件，详细配置见后续文章。