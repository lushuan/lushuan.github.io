---
title: "使用Blue Ocean创建一个简单的流水线"
subtitle: ""
date: 2023-04-14T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Jenkins"]
tags: ["Jenkins"]
---
## 什么是 Blue Ocean?

Blue Ocean 是Jenkins的开源子项目，在保证原有强大的功能不变的基础下，对持续交付(CD)Pipeline过程的可视化方面相较于Jenkins 之前的经典界面有了很大的提升。

Blue Ocean 重新思考Jenkins的用户体验，从头开始设计Jenkins Pipeline, 但仍然与自由式作业兼容，Blue Ocean减少了混乱而且进一步明确了团队中每个成员 Blue Ocean 的主要特性包括：

- 持续交付(CD)Pipeline的 复杂可视化 ，可以让您快速直观地理解管道状态。

- Pipeline 编辑器 - 引导用户通过直观的、可视化的过程来创建Pipeline，从而使Pipeline的创建变得平易近人。

- 个性化 以适应团队中每个成员不同角色的需求。

- 在需要干预和/或出现问题时 精确定位 。 Blue Ocean 展示 Pipeline中需要关注的地方， 简化异常处理，提高生产力

- 本地集成分支和合并请求, 在与GitHub 和 Bitbucket中的其他人协作编码时实现最大程度的开发人员生产力。

## 访问Blue Ocean
在系统管理>插件管理>可选插件中搜索 Blue Ocean >安装,安装成功之后，就可以在页面上看到Blue Ocean的图标,打开后注册git 仓库地址
认证后就可以创建流水线了。

![jenkins-blue-ocean-4](/images/cicd/jenkins-blue-ocean-4.png "Blue Ocean")

## 创建流水线
创建流水线前需要先安装对应的插件
- Blue Ocean  


这边使用的git 仓库是gitee，使用gitlab 也是一样的方式，有两种认证的方式
1. 通过账号口令
2. 通过生成 ssh 公钥在gitlab 或者gitee 仓库上进行注册
![jenkins-blue-ocean](/images/cicd/jenkins-blue-ocean.png "Blue Ocean")

首先先进行证书认证，然后再创建流水线，如果项目中没有 Jenkinsfile 文件，会自动生成一个文件并默认推到 master 分支。

### 在Blue Ocean 查看任务进度视图
点击对应的工作节点，可以查询任务运行过程中的日志详情
![jenkins-blue-ocean-3](/images/cicd/jenkins-blue-ocean-3.png "Blue Ocean")

blue ocean 反向生成的pipeline 代码
```
pipeline {
  agent any
  stages {
    stage('gitlab pull code') {
      parallel {
        stage('gitlab pull code') {
          steps {
            sh 'echo \'gitlab pull code\''
          }
        }

        stage('code test') {
          steps {
            sh 'echo \'code test\''
          }
        }

      }
    }

    stage('docker build') {
      parallel {
        stage('docker build') {
          steps {
            sh 'echo \'docker build\''
          }
        }

        stage('docker test') {
          steps {
            sh 'echo \'docker test\''
          }
        }

      }
    }

    stage('deployment') {
      steps {
        sh 'echo \'deployment\''
      }
    }

    stage('sanity test') {
      steps {
        sh 'echo \'sanity test\''
      }
    }

  }
}
```




## 参考
- https://www.jenkins.io/zh/doc/book/blueocean/