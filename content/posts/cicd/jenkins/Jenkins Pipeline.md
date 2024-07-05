---
title: "Jenkins Pipeline"
subtitle: ""
date: 2023-04-16T12:06:37+08:00
lastmod: 2024-06-26T12:06:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["Jenkins"]
tags: ["Jenkins"]
---
要实现在 Jenkins 中的构建工作，可以有多种方式，我们这里采用比较常用的 Pipeline 这种方式。
Pipeline，简单来说，就是一套运行在 Jenkins 上的工作流框架，将原来独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排和可视化的工作。

## Pipeline是什么？
- Pipeline是Jenkins的核心功能，提供一组可扩展的工具。
- 通过Pipeline 的DSL语法可以完成从简单到复杂的交付流水线实现。
- jenkins的Pipeline是通过Jenkinsfile（文本文件）来实现的。
- 这个文件可以定义Jenkins的执行步骤，例如检出代码。

## Jenkins file
- Jenkinsfile使用两种语法进行编写，分别是声明式和脚本式。
- 声明式和脚本式的流水线从根本上是不同的。
- 声明式是jenkins流水线更友好的特性。
- 脚本式的流水线语法，提供更丰富的语法特性。
- 声明式流水线使编写和读取流水线代码更容易设计。

## 为什么要使用Pipeline?
本质上，jenkins是一个自动化引擎，它支持许多自动模式。流水线向Jenkins添加了一组强大的工具，支持用例、简单的持续集成到全面的持续交付流水线。 
通过对一系列的发布任务建立标准的模板，用户可以利用更多流水线的特性，比如： 
- 代码化: 流水线是在代码中实现的，通常会存放到源代码控制，使团队具有编辑、审查和更新他们项目的交付流水线的能力。 
- 耐用性：流水线可以从Jenkins的master节点重启后继续运行。 
- 可暂停的：流水线可以由人功输入或批准继续执行流水线。 
- 解决复杂发布： 支持复杂的交付流程。例如循环、并行执行。 
- 可扩展性： 支持扩展DSL和其他插件集成。

## Jenkins Pipeline 几个核心概念
- Node：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行环境
- Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如：Build、Test、Deploy，Stage 是一个逻辑分组的概念，可以跨多个 Node
- Step：步骤，Step 是最基本的操作单元，可以是打印一句话，也可以是构建一个 Docker 镜像，由各类 Jenkins 插件提供，比如命令：sh 'make'，就相当于我们平时 shell 终端中执行 make 命令一样。

## 创建Jenkins Pipeline
- Pipeline 脚本是由 Groovy 语言实现的，但是我们没必要单独去学习 Groovy，当然你会的话最好
- Pipeline 支持两种语法：Declarative(声明式)和 Scripted Pipeline(脚本式)语法
- Pipeline 也有两种创建方法：可以直接在 Jenkins 的 Web UI 界面中输入脚本；也可以通过创建一个 Jenkinsfile 脚本文件放入项目源码库中


![jenkins-pipeline-demo](/images/cicd/jenkins-pipeline-demo.png)

一般我们都推荐在 Jenkins 中直接从源代码控制(SCMD)中直接载入 Jenkinsfile Pipeline 这种方法，正常项目中会将 Jenkinsfile 流水线文件放到项目目录下，可以选择`Pipeline script from SCM`
![jenkins-pipeline-scm](/images/cicd/jenkins-pipeline-scm.png)

- [参考项目](https://gitee.com/lu_shuan/spring-mytest)

## Pipeline 模板
可以根据以下模板进行动态的填充和修改
```
// 所有的脚步命令都放在pipeline 中
pipeline{
	// 在任一节点都可以使用
	agent any
// 	parameters {
// 		string{name:'DEPLOY_ENV',defaultValue:'staging',description:''},
// 		booleanParam{name:'DEBUG_BUID',defaultValue:true,description:''}
// 	}

	// 定义全局变量
	environment{
	  key = 'value'
	}

	options {
        //timestamps()  // 日志会有时间，依赖Timestamper 插件
		// skipDefaultCheckout() // 删除隐式checkout scm 语句
		disableConcurrentBuilds()  // 禁止并行
		timeout(time: 1,unit:'HOURS') // 流水线超时设置1h
	}

	stages{

        // Example
        stage('Input 交换的方式'){

            steps {
                timeout(time:5,unit:"MINUTES"){
                    script{
                        echo "nice to meet you"
                        //input id: 'Id', message: 'message', ok: '是否继续', parameters: [choice(choices: ['a', 'b', 'c'], name: 'abc')], submitter: 'lushuan,admin'
                        println("hello")
                    }

                  }
             }
        }

		// 下载代码
		stage('拉取git 仓库代码'){
			steps {
				timeout(time:5,unit:"MINUTES"){
					echo '拉取代码成功-SUCCESS'
				}
			}
		}
		stage('通过maven构建项目'){
			steps {
				timeout(time:20,unit:"MINUTES"){
					sh "/var/jenkins_home/maven/bin/mvn clean package -DskipTests"
				}
			}
		}
		stage('通过SnoarQube做代码质量检测'){
			steps {
				timeout(time:30,unit:"MINUTES"){
					echo '通过SnoarQube做代码质量检测-SUCCESSS'
				}
			}
		}
		stage('通过Docker制作自定义镜像'){
			steps {
				timeout(time:10,unit:"MINUTES"){
                    //sh "docker build -t my-spring-boot-app:latest ."
                    sshPublisher(publishers: [sshPublisherDesc(configName: 'mytest', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '''cd /opt/jenkins/jenkins/workspace/mytest
                    docker build -t my-spring-boot-app:latest .
                    docker stop mytest
                    docker rm mytest
                    docker run -it -d --name mytest -p 8080:8080 my-spring-boot-app:latest''', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
					echo '通过Docker制作自定义镜像-SUCCESSS'
				}
			}
		}
		stage('将自定义镜像推送至Harbor'){
			steps {
				timeout(time:10,unit:"MINUTES"){
					echo '将自定义镜像推送至Harbor-SUCCESSS'
				}
			}
		}
		stage('通过Publish Over SSH 通知目标服务器'){
			steps {
				timeout(time:10,unit:"MINUTES"){
					echo '通过Publish Over SSH 通知目标服务器-SUCCESSS'
				}
			}
		}
	}

	// 构建后操作
	post {
		// 无论成功失败总是执行
		always{
			script{
				println("always")
			}
		}
		success {
			script{
				currentBuild.description += "\n 构建成功!"
			}
			// 需要安装 dingtalk 插件
// 			dingtalk(
// 				robot: "钉钉自定义名称",
// 				type: 'MARKDOWN',
// 				title: "success: ${JOB_NAME}"
// 				text: ["- 成功构建:${JOB_NAME}! \n-版本:${tag} \n-持续时间:${currentBuild.durationString}"]
// 			)
	   	}
		failure {
			script{
				currentBuild.description += "\n 构建失败!"
			}
// 			dingtalk(
// 				robot: "钉钉自定义名称",
// 				type: 'MARKDOWN',
// 				title: "failure: ${JOB_NAME}"
// 				text: ["- 失败构建:${JOB_NAME}! \n-版本:${tag} \n-持续时间:${currentBuild.durationString}"]
// 			)
	   	}
	   	aborted {
			script{
				currentBuild.description += "\n 构建取消!"
			}
	   	}
	}
}

```

## 流水线基础语法
以下是简要的对流水线语法进行粗浅的人生，更详细的请登录[官网流水线语法参考](https://www.jenkins.io/zh/doc/book/pipeline/syntax/)进行查看

- 简介pipeline 基础语法，声明式脚步式，节点步骤

1. 安装声明式插件 `Pipeline: Declarative`
2. Jenkinsfile 组成
	1. 指定node 节点/workspace
	2. 指定运行选项
	3. 指定stages 阶段
	4. 指定构建后操作
3. Pipeline 的语法分为两种，一种是声明式的，另外一种是脚步式的，声明式编写更加友好，声明式内可以嵌入脚本式

### Pipeline定义- post
- 指定构建后操作
- 介绍
	- always{}: 总是执行脚本片段
	- success{}: 成功后执行
	- failure{}: 失败后执行
	- aborted{}: 取消后执行
	- changed 只有当流水线或者阶段完成状态与之前不同时
	- unstable 只有当流水线或者阶段状态为"unstable运行"，例如测试失败
### Pipeline语法- agent
agent 指定了流水线的执行节点
参数：
	- any 在任何可用的节点上执行pipeling
	- none 没有指定agent 的时候默认
	- lable 在指定的标签上的节点上运行Pipeline
	- node 允许额外的选项
	```
	两种是一样的
	agent { node {label 'labelname'}}
	agent { label 'labelname'}
	```
### Pipeline语法- option
- buildDiscarder:为最近的流水线运行的特定数量保存组件和控制台输出。
- disableConcurrentBuilds:不允许同时执行流水线。可被用来防止同时访问共享资源等
- overridelndexTriggers: 允许覆盖分支索引触发器的默认处理。
- skipDefaultCheckout: 在agent 指令中,跳过从源代码控制中检出代码的默认情况。
- skipStagesAfterUnstable:一旦构建状态变得UNSTABLE,跳过该阶段。
- checkoutToSubdirectory: 在工作空间的子目录中自动地执行源代码控制检出。
- timeout: 设置流水线运行的超时时间,在此之后,Jenkins将中止流水线。
- retry:在失败时,重新尝试整个流水线的指定次数。
- timestamps 预测所有由流水线生成的控制台输出,与该流水线发出的时间一致。

### Pipeline语法- parameters
添加参数时，可以在jenkins 脚本中获取到
```
parameters { 
		string{name:'DEPLOY_ENV',defaultValue:'staging',description:''},
		booleanParam{name:'DEBUG_BUID',defaultValue:true,description:''}
	}
```
### Pipeline语法- trigger
触发器
```
triggers {
    cron('H */4 * * 1-5')
}
```
### Pipeline语法- tool
主要是用来指定全局工具
```
tools {
	maven 'apache-maven-3.0.1'
}
```
如在pipeline 中集成maven时使用
```
stage("build"){
	mvHome="/usr/local/apache-maven-3.5.0/bin"
	sh "${mvHome}/mvn clean package" 
}
// 或者通过 tool 引用
stage("build"){
	mvHome = tool 'M3' // M3: /usr/local/apache-maven-3.5.0/bin
	sh "${mvHome}/mvn clean package" 
}
```

### Pipeline语法- input
input 用户在执行各个阶段的时候，由人工确认是否继续执行
- message 呈现给用户的提示信息。
- id 可选,默认为stage名称。
- ok 默认表单上的ok文本。
- submitter 可选的,以逗号分隔的用户列表或允许提交的外部组名。默认允许任何用户。
- submitterParameter环境变量的可选名称。如果存在,用submitter 名称设置。
- parameters 提示提交者提供的一个可选的参数列表。

### Pipeline语法- when
when 指令允许流水线根据给定的条件决定是否应该执行阶段。when 指令必须包含至少一个条件。如果when 指令包含多个条件，所有的条件必须返回True，阶段才能执行。

语法和在stage 平级
内置条件
- branch: 当正在构建的分支与模式给定的分支匹配是，执行这个阶段，这只适用于多分支流水线例如：
```
when { branch 'master'}
```
environment:当指定的环境变量是给定的值时，执行这个步骤，例如：
```
when { environment name:'DEPLOY_TO',value: 'production'}
```
not 当嵌套的条件错误时，执行这个阶段，必须包含一个条件，例如
```
when { not {branch 'master'}}
```
anyof
```
when { anyof {branch 'master';branch 'staging'}}
```
### Pipeline语法- parallel 并行
声明式流水线的阶段可以在他们内部声明多隔嵌套阶段, 它们将并行执行。

另外, 通过添加 failFast true 到包含parallel的 stage中， 当其中一个进程失败时，可以强制所有的 parallel 阶段都被终止。

```
pipeline {
    agent any
    stages {
        stage('Non-Parallel Stage') {
            steps {
                echo 'This stage will be executed first.'
            }
        }
        stage('Parallel Stage') {
            when {
                branch 'master'
            }
            failFast true
            parallel {
                stage('Branch A') {
                    agent {
                        label "for-branch-a"
                    }
                    steps {
                        echo "On Branch A"
                    }
                }
                stage('Branch B') {
                    agent {
                        label "for-branch-b"
                    }
                    steps {
                        echo "On Branch B"
                    }
                }
            }
        }
    }
}
```

## 参考
- [Jenkins 流水线](https://www.jenkins.io/zh/doc/book/pipeline/)



