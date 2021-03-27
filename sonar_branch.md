# SonarQube 多分支功能使用（社区版）

---

title: SonarQube 多分支功能使用（社区版）
date: 2021-03-27 23:24:00
author: mamian521#gmail.com

---



本文以 SonarQube 7.9 版本的为例，介绍社区版本的多分支功能。

### 准备

- 下载 [地址]([]()) ，并安装到 SonarQube 安装目录下的 `lib/commons` 目录 和 `extensions/plugins` 目录下。
- 可以直接使用我打包好的 Docker 镜像，参考

### 使用

使用多分支之前需要先扫描主分支，多分支有一些概念我简单介绍下。

- 主分支，首要的分支，一般为 master
- 长期分支，长期维护的分支，如 release 和 develop 分支，问题数据单独存储
- 短期分支，短期分支的问题和长期分支相比是增量的数据
- PR，PullRequest 扫描

### 主分支

主分支是 main branch，主分支是可修改的，默认情况下，系统设定为 master 。如果不设置 [sonar.branch.name](http://sonar.branch.name) 参数，那么系统默认此次任务的分支是 master 。

关于分支提交任务的问题，这里我整理了几种情况。

- 如果主分支不是 master ，建议在 SonarQube 创建project之后，修改主分支名称。
- sonar-scanner 提交扫描任务时，指定分支名为 主分支的名称 （例如 master），如果第一次指定的分支名不是主分支，也不是master，那么会报错。请按照上一条步骤解决。
- sonar-scanner 如果不指定分支名，默认将是 master 分支。
- sonar-scanner 扫描过主分支后，接下来可以扫描其他分支
- 可以在每个仓库的设置页面上配置长期分支的正则表达式，用来区分哪些分支是长期分支

### 设置扫描分支

在 sonar-scanner 命令中增加 `-Dsonar.branch.name={branch.name}` 参数，例如 `-Dsonar.branch.name=feature-foo` 然后就可以设置扫描分支

### API

获取 Issue 接口的 api ，例如 `/api/issues/search` 接口获取指定分支数据库，可以添加 branch 参数即可



![image-20210327232622670](https://tva1.sinaimg.cn/large/008eGmZEly1goyvp0bkmlj321a0jsn15.jpg)

![image-20210327232641591](https://tva1.sinaimg.cn/large/008eGmZEly1goyvpash83j30yw0im409.jpg)

![image-20210327232655053](https://tva1.sinaimg.cn/large/008eGmZEly1goyvpj2aobj30t80fgwf2.jpg)

![image-20210327232659091](/Users/mark/Library/Application Support/typora-user-images/image-20210327232659091.png)

![image-20210327232712345](https://tva1.sinaimg.cn/large/008eGmZEly1goyvptydelj30r40aqmyi.jpg)