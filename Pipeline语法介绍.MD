## 一. 声明式流水线(Declarative Pipeline)

所有有效的声明式流水线必须包含在一个pipeline块中：
```groovy
pipeline {
    /* insert Declarative Pipeline here */
}
```
在声明式流水线中有效的基本语句和表达式遵循与 Groovy 语法相同的规则，但有以下例外：

- Pipeline 的顶层必须是一个块，具体来说是：***pipeline { }***。

- 没有分号作为语句分隔符。 每个语句都必须在它自己的行上。

- 块只能由章节(Sections)、指令(Directives)、步骤(Steps)或赋值语(assignment statements)句组成。

- 属性引用语句被视为无参数方法调用。 例如，***input***被视为***input()***。

Jenkins还提供了声明式指令***片段生成器***来帮助生成声明式流水线指令。

Jenkinsfile一般结构如下：
```groovy
pipeline {
  agent {
    //define agent
  }
  stages {
    stage('name') {
      steps {
        //some scripts
      }
    }
    ...
  }
}
```
### 1. agent （require）

agent声明整个pipeline脚本或某个stage脚本在jenkins中的运行环境，写在pipeline最外层则是声明整个pipeline的运行环境，写在stage里则是stage的运行环境。pipeline的最外层必须声明agent， 而在stage里则是可选项。

***agent的类型参数：***

#### 1.1 any

  在任意可用代理上运行pipeline。

示例：

  ```groovy
  pipeline {
  	agent any
  	stages {
  		stage {
  			//optional
  			agent any
  			steps {
  				//do something
  			}
  		}
  	}
  }
  ```


#### 1.2 none

  当在全局声明agent为none时，那么每一个stage都需声明自己的agent。

#### 1.3 label

  按在jenkins环境中设定的label标签名声明agent，如label所代表的节点。

  ```groovy
  agent {
  	label 'my-node1'
  }
  ```

  除此之外label还可以与逻辑符号一同使用：

  ```groovy
  agent {
  	label 'my-node1' && 'my-node2'
  }
  ```

  ```groovy
  agent {
  	label 'my-node1' || 'my-node2'
  }
  ```

#### 1.4 node

  node声明运行节点，与label功能类似，但是node可以附加选项参数， 如customWorkspace：

  ```groovy
  agent {
  	node {
  		label 'my-node1'
  	}
  }
  ```

  等同于

  ```groovy
  agent {
  	label 'my-node1'
  }
  ```

  node附加选项参数：

  ```groovy
  agent {
      node {
          label 'my-defined-label'
          customWorkspace '/some/other/path'
      }
  }
  ```

#### 1.5 docker

  用于基于docker的pipeline流水线，在预配置的节点上或指定label的节点上创建docker容器。除此之外还可以接收docker run的参数，以及docker registry等参数。

示例：

  ```groovy
  agent {
  	docker 'maven:3.8.1-adoptopenjdk-11'
  }
  ```

  ```groovy
  agent {
      docker {
          image 'maven:3.8.1-adoptopenjdk-11'
          label 'my-defined-label'
          args  '-v /tmp:/tmp'
          registryUrl 'https://myregistry.com/'
          registryCredentialsId 'myPredefinedCredentialsInJenkins'
      }
  }
  ```

#### 1.6 dockerfile

从一个Dockerfile创建docker容器来运行pipeline流水线。默认会从构建的根目录寻找Dockerfile：

```groovy
agent { 
  dockerfile true 
}
```

如果项目的Dockerfile存放在子目录下需通过***dir***参数声明，如果Dockerfile名字是其他自定义名则需通过***filename***参数声明，除此之外还可以通过***additionalBuildArgs***来设置***docker build***参数，使用***registryUrl***来设置docker仓库，使用***registryCredentialsId***从jenkins获取docker仓库的账号密码。

示例：

```groovy
agent {
    // Equivalent to "docker build -f Dockerfile.build --build-arg version=1.0.2 ./build/
    dockerfile {
        filename 'Dockerfile.build'
        dir 'build'
        label 'my-defined-label'
        additionalBuildArgs  '--build-arg version=1.0.2'
        args '-v /tmp:/tmp'
      	registryUrl 'https://myregistry.com/'
        registryCredentialsId 'myPredefinedCredentialsInJenkins'
    }
}
```

#### 1.7 kubernetes

  在kubernetes集群中的一个pod内执行pipeline流水线，pod模版将定义在***kubernetes{}***模块中。如果需要在pod中安装kaniko则可在***yaml***参数中声明。

示例：

```groovy
agent {
    kubernetes {
        label podlabel
        yaml """
kind: Pod
metadata:
  name: jenkins-agent
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
      - name: aws-secret
        mountPath: /root/.aws/
      - name: docker-registry-config
        mountPath: /kaniko/.docker
  restartPolicy: Never
  volumes:
    - name: aws-secret
      secret:
        secretName: aws-secret
    - name: docker-registry-config
      configMap:
        name: docker-registry-config
"""
   }
```

你需要创建一个aws-secret密钥提供给kaniko来进行ECR身份验证，此密钥需包含***~/.aws/credentials***的内容。另外一个卷是个***configMap***，它包含ECR注册的***endpoint***。

```groovy
{
      "credHelpers": {
        "<your-aws-account-id>.dkr.ecr.eu-central-1.amazonaws.com": "ecr-login"
      }
}
```



***agent的选项参数：***

- #### label

  字符串类型。此参数适用于***node***，***docker***，***dockerfile***，而且在ndoe中是必输参数。

- #### customWorkspace

  字符串类型。在自定义工作空间目录中运行此pipeline流水线而不是使用默认的。可以是相对路径(相对于运行节点工作空间的根目录)也可以是绝对路径。此参数适用于***node***，***docker***，***dockerfile***。

  示例：

  ```groovy
  agent {
      node {
          label 'my-defined-label'
          customWorkspace '/some/other/path'
      }
  }
  ```

- #### reuseNode

  布尔值类型，默认false。此参数适用于***docker***和***dockerfile***， 且只在stage代理中有效。当设置为true时，则在pipeline的全局agent节点中运行容器，并且是在同一个工作空间中运行而不是在一个全新的节点中运行。

- #### args

  字符串类型。适用于***docker***和***dockerfile***，用于设置***docker run***参数。
  
  示例：
  
  ```groovy
  pipeline {
      agent { docker 'maven:3.8.1-adoptopenjdk-11' } 
      stages {
          stage('Example Build') {
              steps {
                  sh 'mvn -B clean verify'
              }
          }
      }
  }
  ```

### 2. post

定义附加***steps***在整个pipeline流水线或某个stage执行完之后运行。***post***可以定义在***pipeline***的最外层或某个***stage***里， 如果定义在最外层则表示这个pipeline执行完之后运行，如果在stage里则表示该stage执行完后运行。

***post的执行条件场景包括：***

- #### always

  不管pipeline或stage的执行结果状态，总会执行的***steps***。

- #### changed

  只有在pipeline或stage的执行结果状态与前一次执行相比发生改变时执行。

- #### fixed

  当前pipeline或stage执行成功且它的前一次执行结果是***failure***或***unstable***时执行。

- #### regression

  当前pipeline或stage执行结果是***failure***, ***unstable***或***aborted***且它的前一次执行成功时执行。

- #### aborted

  当前pipeline或stage执行结果是***aborted***（人工停止pipeline）时执行。

- #### failure

  当前pipeline或stage执行结果是失败时执行。

- #### success

  当前pipeline或stage执行结果是成功时执行。

- #### unstable

  当前pipeline或stage执行结果是***unstable***时执行。

- #### unsuccessful

  当前pipeline或stage执行结果不是成功时执行。

- #### cleanup

  在其他所有的***post***场景脚本都处理完之后执行，不管当前pipeline或stage执行结果是什么。

示例：

```groovy
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
    post { 
        always { 
            echo 'I will always say Hello again!'
        }
    }
}
```



### 3. stages（require）

pipeline阶段集声明部分，至少包含一个或多个stage。

### 4. stage（atlest one）

构建阶段模块。定义构建阶段要做的事情，每个pipeline流水线至少包含一个stage。一个stage里有且只有一个steps`, `stages`, `parallel, 或matrix。当一个stage内嵌在parallel或matrix中的时候，那么该stage就不能再包含parallel或matrix。

示例：

```groovy
// Declarative //
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```



### 5. steps（require）

***steps***包含一个或多个可执行指令。

### 6. environment

environment命令用于声明一系列的键值对作为整个pipeline或某个stage指令执行的环境变量，取决于它定义在pipeline的最外层或stage里。该指令还支持通过***credentials()***获取预设的账号密码。

其中支持的credential类型包括：

- **Secret Text**

  该环境变量的值将会被设置为Secret Text的内容。

- **Secret File**

  该环境变量的值将会被设置为临时创建的文件路径。

- **Username and password**

  该环境变量的值将会被设置为***username:password***, 并且还会自动创建两个环境变量：***MYVARNAME_USR***和***MYVARNAME_PSW***。

- **SSH with Private Key**

  该环境变量的值将会被设置为临时创建的ssh key文件路径，并且还会自动创建两个环境变量：***MYVARNAME_USR***和***MYVARNAME_PSW***。

示例：

```groovy
pipeline {
    agent any
    environment { 
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment { 
                AN_ACCESS_KEY = credentials('my-predefined-secret-text') 
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

```groovy
pipeline {
    agent any
    stages {
        stage('Example Username/Password') {
            environment {
                SERVICE_CREDS = credentials('my-predefined-username-password')
            }
            steps {
                sh 'echo "Service user is $SERVICE_CREDS_USR"'
                sh 'echo "Service password is $SERVICE_CREDS_PSW"'
                sh 'curl -u $SERVICE_CREDS https://myservice.example.com'
            }
        }
        stage('Example SSH Username with private key') {
            environment {
                SSH_CREDS = credentials('my-predefined-ssh-creds')
            }
            steps {
                sh 'echo "SSH private key is located at $SSH_CREDS"'
                sh 'echo "SSH user is $SSH_CREDS_USR"'
                sh 'echo "SSH passphrase is $SSH_CREDS_PSW"'
            }
        }
    }
}
```

### 7. options

pipeline流水线选项参数，pipeline流水线提供了一系列的选项参数。其中也有些由插件提供， 例如***timestamps***。全局options声明在最外层， 而写在stage里的则只在该stage有效且stage options只支持：**skipDefaultCheckout**，**timeout**，**retry**和**timestamps**。

#### 7.1 buildDiscarder

保存最近构建构建制品和控制台日志输出数量。

示例：

```groovy
//只保存最近一次构建日志
options { 
  buildDiscarder(logRotator(numToKeepStr: '1')) 
}
```



#### 7.2 checkoutToSubdirectory

将源码拉取到工作空间指定的子目录下。

示例：

```groovy
options { 
  checkoutToSubdirectory('foo') 
}
```



#### 7.3 disableConcurrentBuilds

不允许同时构建。

示例：

```groovy
options { 
  disableConcurrentBuilds() 
}
```



#### 7.4 disableResume

不允许控制器重新开始。

示例：

```groovy
options { 
  disableResume() 
}
```



#### 7.5 newContainerPerStage

与docker或dockerfile代理一起使用，表示每个stage都将在同一个节点里启动新容器运行而不是在同一个容器中运行。

示例：

```groovy
options {
  newContainerPerStage()
}
```



#### 7.6 overrideIndexTriggers

重写分支索引触发器的处理方式。如果设置为***true***， 当分支索引触发器在多分支或机构标签中禁止时，他们将为当前构建开启。如果设置为***false***， 当分支索引触发器在多分支或机构标签中开启时，他们将为当前构建禁止。

示例：

```groovy
options { 
  overrideIndexTriggers(true) 
} 
```



#### 7.7 preserveStashes

当stage重新开始时，保存最近构建。

示例：

```groovy
//保存最近一次完成的构建
options { 
  preserveStashes() 
}
```



```groovy
//保存最近5次完成的构建
options { 
  preserveStashes(buildCount: 5) 
}
```



#### 7.8 quietPeriod

延迟启动时间，单位：秒。此设置将重写默认值。

示例：

```groovy
options { 
  quietPeriod(30) 
}
```



#### 7.9 retry（stageable）

失败重试次数。

示例：

```groovy
options { 
  retry(3) 
}
```



#### 7.10 skipDefaultCheckout （stageable）

跳过默认代码拉取。

示例：

```groovy
options { 
  skipDefaultCheckout() 
}
```



#### 7.11 skipStagesAfterUnstable

当构建状态出现***unstable***时跳过其他stages。

示例：

```groovy
options { 
  skipStagesAfterUnstable() 
}
```



#### 7.12 timeout（stageable）

设置构建超时时间，超时后自动终止执行。

示例：

```groovy
pipeline {
    agent any
    options {
        timeout(time: 1, unit: 'HOURS') 
    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```



#### 7.13 timestamps（stageable）

设置日志输出时间戳。

示例：

```groovy
options { 
  timestamps() 
}
```



#### 7.14 parallelsAlwaysFailFast

将所有并行stage的failfast设置为true。

示例：

```groovy
options { 
  parallelsAlwaysFailFast() 
}
```



### 8. parameters

***parameters***声明当构建触发时用户要输入的所有参数，steps指令将通过***params***对象获取这些参数。

参数类型包括：

- ***string***
- ***text***
- ***booleanParam***
- ***choice***
- ***password***

示例：

```groovy
pipeline {
    agent any
    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')

        text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')

        booleanParam(name: 'TOGGLE', defaultValue: true, description: 'Toggle this value')

        choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')

        password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    }
    stages {
        stage('Example') {
            steps {
                echo "Hello ${params.PERSON}"

                echo "Biography: ${params.BIOGRAPHY}"

                echo "Toggle: ${params.TOGGLE}"

                echo "Choice: ${params.CHOICE}"

                echo "Password: ${params.PASSWORD}"
            }
        }
    }
}
```



### 9. triggers 

构建触发器。不常用，一般使用webhooks居多。

- ***cron***

  定时器触发。

  示例：

  ```groovy
  //每隔4小时构建一次
  triggers { 
    cron('H */4 * * 1-5') 
  }
  ```

  

- ***pollSCM***

  定时检查源码变更触发。jenkins定时检查源码是否变更，如果发生变更则触发构建。

  示例：

  ```groovy
  triggers { 
    pollSCM('H */4 * * 1-5') 
  }
  ```

  

- ***upstream***

  上游构建触发。当上游构建完成状态匹配时触发构建。

  示例：

  ```groovy
  triggers { 
    upstream(upstreamProjects: 'job1,job2', threshold: hudson.model.Result.SUCCESS) 
  }
  ```

  

### 10. tools

声明需要自动安装构建所需的工具，并设置到PATH环境变量里。当agent none时，tools声明将会被忽略。

支持自动安装的工具有：

- ***jdk***
- ***maven***
- ***gradle***

示例：

```groovy
pipeline {
    agent any
    tools {
        maven 'apache-maven-3.0.1' 
    }
    stages {
        stage('Example') {
            steps {
                sh 'mvn --version'
            }
        }
    }
}
```



### 11. input (stage only)

stage参数输入指令。在进入agent模块和when条件判断之前，stage将会被暂停并弹出输入对话框，直到用户输入可用参数后stage才会继续执行。所有输入的参数在余下的stage指令当中都可以通过环境变量获取。

input配置选项：

- ***messgae*** （必输）

  提示信息。

- ***id***

  唯一标识，默认使用该stage名称。

- ***ok***

  “OK”按钮名称。

- ***submitter***

  以","分割的用户或外部组的列表，定义谁可以提交这个输入项，默认所有人。

- ***submitterParameter***

  设置submitter的环境变量名称。

- ***parameters***

  配置需要输入的参数。

示例：

```groovy
pipeline {
    agent any
    stages {
        stage('Example') {
            input {
                message "Should we continue?"
                ok "Yes, we should."
                submitter "alice,bob"
                parameters {
                    string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
                }
            }
            steps {
                echo "Hello, ${PERSON}, nice to meet you."
            }
        }
    }
}
```



### 12. when (stage only)

when命令通过设定的条件判断决定pipeline流水线的当前stage是否执行， when命令至少包含一个条件判断。如果when包含多个条件判断，那么当且仅当所有条件判断都为true时，当前stage才会被执行。正常情况下when判断时在***agent***，***input***，***options***命令之后才执行的，但是可以通过***beforeAgent***，***beforeInput***，***beforeOptions***参数来设置提前执行。

示例：

```groovy
pipeline {
    agent none
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            agent {
                label "some-label"
            }
            when {
                beforeAgent true
              	beforeInput true
              	beforeOptions true
                branch 'production'
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```



#### 12.1 branch

  只使用于多分支结构流水线。当构建分支匹配给定的分支或正则表达式时为true。

  匹配规则：

  - EQUALS 

    字符串比较。

  - GLOB

    ANT通配符。

  - REGEXP 

    正则表达式。

  示例：

  ```groovy
  when { 
    branch 'master' 
  }
  ```

  ```groovy
  when { 
    branch pattern: "release-\\d+", comparator: "REGEXP"
  }
  ```

  

#### 12.2 buildingTag

  当构建一个版本标签时为true。

  ```groovy
  when { 
    buildingTag() 
  }
  ```

  

#### 12.3 changelog

  当SCM的变更日志匹配给定的正则表达式时为true。

  ```groovy
  when { 
    changelog '.*^\\[DEPENDENCY\\] .+$' 
  }
  ```

  

#### 12.4 changeset

当SCM变更集包含一个或以上文件匹配给定的文件路径表达式时为true。

匹配规则：

  - EQUALS 

    字符串比较。

  - GLOB

    ANT通配符。

  - REGEXP 

    正则表达式。

通过caseSensitive参数设置是否区分大小写。

示例：

```groovy
when { 
  changeset "**/*.js" 
}
```

```groovy
when { 
  changeset pattern: ".TEST\\.java", comparator: "REGEXP" 
} 
or when { 
  changeset pattern: "*/*TEST.java", caseSensitive: true 
}
```



#### 12.5 changeRequest

当因SCM发生变更请求（aka，GitHub and Bitbucket：Pull Request，GitLab：Merge Request， Gerrit： Change Request）而构建时为true。可添加过滤属性，来筛选符合条件的变更请求：

- ***id***
- ***target***
- ***branch***
- ***fork***
- ***url***
- ***title***
- ***author***
- ***authorDisplayName***
- ***authorEmail***

过滤属性匹配规则：

  - EQUALS 

    字符串比较。

  - GLOB

    ANT通配符。

  - REGEXP 

    正则表达式。


示例：

```groovy
 when { 
   changeRequest() 
 }
```

```groovy
when { 
  changeRequest target: 'master' 
}
```

```groovy
when { 
  changeRequest authorEmail: "[\\w_-.]+@example.com", comparator: 'REGEXP' 
}
```



#### 12.6 environment

当某个环境变量值等于给定值时为true。

示例：

```groovy
hen { 
  environment name: 'DEPLOY_TO', value: 'production' 
}
```



#### 12.7 equals

当实际值等于期望值时为true。

示例：

```groovy
when { 
  equals expected: 2, actual: currentBuild.number 
}
```



#### 12.8 expression

当给定的groovy表达式返回值为true。如果返回值时字符串那么当返回为null时为false，其他字符串都表示为true。

示例：

```groovy
when { 
  expression { 
    return params.DEBUG_BUILD 
  } 
}
```



#### 12.9 tag

当TAG_NAME环境变量匹配给定正则表达式时为true。如果表达式为空则只要TAG_NAME环境变量存在则为true。

匹配规则：

  - EQUALS 

    字符串比较。

  - GLOB

    ANT通配符。

  - REGEXP 

    正则表达式。

示例：

```groovy
when { 
  tag "release-*" 
}
```

```groovy
when { 
  tag pattern: "release-\\d+", comparator: "REGEXP"
}
```



#### 12.10 not

当内嵌条件为true时。

示例：

```groovy
when { 
  not { 
    branch 'master' 
  } 
}
```



#### 12.11 allOf

当所有内嵌条件都为true时。

示例：

```groovy
when { 
  allOf { 
    branch 'master'; 
    environment name: 'DEPLOY_TO', value: 'production' 
  } 
}
```



#### 12.12 anyOf

当任意内嵌条件为true时。

示例：

```groovy
when { 
  anyOf { 
    branch 'master'; branch 'staging' 
  } 
}
```



#### 12.13 triggeredBy

当构建由给定触发时为true。

示例：

```groovy
when { triggeredBy 'SCMTrigger' }
```

```groovy
when { triggeredBy 'TimerTrigger' }
```

```groovy
when { triggeredBy 'UpstreamCause' }
```

```groovy
when { triggeredBy cause: "UserIdCause", detail: "vlinde" }
```



### 13. parallel

声明并行执行的stage。通过在包含parallel的stage里设置failFast true，当其中一个并行stage执行失败时强制终止其他并行stage。也可以在options里做全局设置parallelsAlwaysFailFast()。

示例：

```groovy
pipeline {
    agent any
    options {
        parallelsAlwaysFailFast()
    }
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
          	//failFast true
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
                stage('Branch C') {
                    agent {
                        label "for-branch-c"
                    }
                    stages {
                        stage('Nested 1') {
                            steps {
                                echo "In stage Nested 1 within Branch C"
                            }
                        }
                        stage('Nested 2') {
                            steps {
                                echo "In stage Nested 2 within Branch C"
                            }
                        }
                    }
                }
            }
        }
    }
}
```



### 14. matrix

按name-value定义的命名空间矩阵，可在stage里使用定义并行任务。matrix必需包含一个axes和一个stages。failFast和parallelsAlwaysFailFast()也使用于matrix。

- axes

  每一个axes必需包含一个或以上axis

- axis

  每个axis包含一个name和一个values列表，所有axies中的values列表彼此组合产生命名空间cell。stages将按循序在各个cell中被执行。每个命名空间的name-value键值对将自动创建对应的环境变量。

- excludes

  按命name-value名空间移除不需要执行的cell。当移除的命名空间cell列表很长时可以使用***notValues***替代***values***。

每个cell中仍然可以使用以下这些命令, 在matrix最外层声明：

- agent
- environment
- input
- options
- post
- tools
- when

示例：

```groovy
pipeline {
    parameters {
        choice(name: 'PLATFORM_FILTER', choices: ['all', 'linux', 'windows', 'mac'], description: 'Run on specific platform')
    }
    agent none
    stages {
        stage('BuildAndTest') {
            matrix {
                agent {
                    label "${PLATFORM}-agent"
                }
                when { anyOf {
                    expression { params.PLATFORM_FILTER == 'all' }
                    expression { params.PLATFORM_FILTER == env.PLATFORM }
                } }
                axes {
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'windows', 'mac'
                    }
                    axis {
                        name 'BROWSER'
                        values 'firefox', 'chrome', 'safari', 'edge'
                    }
                }
                excludes {
                    exclude {
                        axis {
                            name 'PLATFORM'
                            values 'linux'
                        }
                        axis {
                            name 'BROWSER'
                            values 'safari'
                        }
                    }
                    exclude {
                        axis {
                            name 'PLATFORM'
                            notValues 'windows'
                        }
                        axis {
                            name 'BROWSER'
                            values 'edge'
                        }
                    }
                }
                stages {
                    stage('Build') {
                        steps {
                            echo "Do Build for ${PLATFORM} - ${BROWSER}"
                        }
                    }
                    stage('Test') {
                        steps {
                            echo "Do Test for ${PLATFORM} - ${BROWSER}"
                        }
                    }
                }
            }
        }
    }
}

```



### 15. script

在steps里使用脚本pipeline代码段。

示例：

```groovy
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'

                script {
                    def browsers = ['chrome', 'firefox']
                    for (int i = 0; i < browsers.size(); ++i) {
                        echo "Testing the ${browsers[i]} browser"
                    }
                }
            }
        }
    }
}
```



## 二. 脚本式流水线(Scripted Pipeline)

基于groovy的脚本，相对于声明式pipeline，脚本式pipeline表达能力更强更有弹性。

示例：

```groovy
node {
    stage('Example') {
        try {
            sh 'exit 1'
        }
        catch (exc) {
            echo 'Something failed, I should sound the klaxons!'
            throw
        }
    }
}

```





***参考：https://www.jenkins.io/doc/book/pipeline/syntax/#tools***
