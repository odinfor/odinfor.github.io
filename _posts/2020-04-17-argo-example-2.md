---
layout: post
title: "「devops」argo workflow笔记(三)"
subtitle: 'argo adls模板示例'
author: "odin"
header-style: text
tags:
  - DevOps
---
![]({{site.baseurl}}/img/in-post/post-devops/argo-log.jpg)

### argo template示例
#### 4.argo scripts adsl example
##### 示例1：scripts template
argo adsl支持template steps中执行脚本。
```yaml
# 脚本只输出执行结果
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: scripts-bash-
spec:
  entrypoint: bash-script-example
  templates:
  - name: bash-script-example
    steps:
    - - name: generate
        template: gen-random-int-bash
    - - name: print
        template: print-message
        arguments:
          parameters:
          - name: message
            value: "{{steps.generate.outputs.result}}"  # The result of the here-script
 
  - name: gen-random-int-bash
    script:
      image: debian:9.4
      command: [bash]
      source: |                                         # Contents of the here-script
        cat /dev/urandom | od -N2 -An -i | awk -v f=1 -v r=100 '{printf "%i\n", f + r * $1 / 65536}'
 
  - name: gen-random-int-python
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        i = random.randint(1, 100)
        print(i)
 
  - name: gen-random-int-javascript
    script:
      image: node:9.1-alpine
      command: [node]
      source: |
        var rand = Math.floor(Math.random() * 100);
        console.log(rand);
 
  - name: print-message
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo result was: {{inputs.parameters.message}}"]
```
使用script关键字源标记指定脚本主体。将创建一个包含脚本主体的临时文件，然后将临时文件的名称作为最后一个参数传递给command, command是一个执行脚本主体的解释器。需要指明script需要的解释器。
改脚本有输出值的脚本(输出只有结果值)执行完毕后会用一个result 的一个默认的结果字段收集该脚本的输出值，供template流程节点调用。

##### 示例2：output parameters scripts template
上面是直接返回结果的脚本，当然argo srcipts steps支持使用参数输出的的方式提供不同srcipts之间使用依赖参数，甚至的其他元件都可以调用，示例如下。
```yaml
# 带输出参数供后续工作流程调用的scripts
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: output-parameter-
spec:
  entrypoint: output-parameter
  templates:
  - name: output-parameter
    steps:
    - - name: generate-parameter
        template: whalesay
    - - name: consume-parameter
        template: print-message
        arguments:
          parameters:
          # Pass the hello-param output from the generate-parameter step as the message input to print-message
          - name: message
            value: "{{steps.generate-parameter.outputs.parameters.hello-param}}"
 
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["echo -n hello world > /tmp/hello_world.txt"]  # generate the content of hello_world.txt
    outputs:
      parameters:
      - name: hello-param       # name of output parameter
        valueFrom:
          path: /tmp/hello_world.txt    # set the value of hello-param to the contents of this hello-world.txt
 
  - name: print-message
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```
这个template中template模板whalesay使用outputs关键字定义了输出参数hello-param，在模板print-message中通过{{steps.generate-parameter.outputs.parameters.hello-param}}使用参数hello-param的赋值给message

#### 5.argo retry&limit adsl example
##### 示例1：错误重试
argo adsl提供retryStrategy 关键字来声明重试逻辑，可以给流程定义我们需要重试的机制。
```yaml
# This example demonstrates the use of retry back offs
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: retry-backoff-
spec:
  entrypoint: retry-backoff
  templates:
  - name: retry-backoff
    retryStrategy:
      limit: 10
      retryPolicy: "Always"
      backoff:
        duration: "1"      # Must be a string. Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"
        factor: 2
        maxDuration: "1m"  # Must be a string. Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"
    container:
      image: python:alpine3.6
      command: ["python", -c]
      # fail with a 66% probability
      args: ["import random; import sys; exit_code = random.choice([0, 1, 1]); sys.exit(exit_code)"]
```
重试机制中包含以下几部分：
* limit：最大重试限制次数
* retryPolicy：重试机制，分为OnFailure(失败重试，当工作流执行失败重试)、OnError(出错重试，当工作流执行出现异常触发重试，默认重试规则)和Always(无论是执行出错还是执行失败都触发重试)
* backoff：回退相关，详细内容暂时没有弄明白  
注：若配置一个空retryStrategy对象，即retryStrategy：{} 则该工作流所有steps都会一直重试直到整个工作流成功执行完成

##### 示例2：限制执行时间
argo adsl提供activeDeadlineSeconds关键字限制工作流执行时间  
```yaml
# 该工作流超时时间为10s
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: timeouts-
spec:
  entrypoint: sleep
  templates:
  - name: sleep
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo sleeping for 1m; sleep 60; echo done"]
    activeDeadlineSeconds: 10           # terminate container template after 10 seconds
```
除了执行时间限制以外，argo也支持其他限制条件，例如文件大小限制，参考：[Volumes限制](https://github.com/argoproj/argo/blob/master/examples/README.md)  Volumes部分

#### 6.argo exit handlers adsl example
argo提供退出句柄供我们使用，在工作流中我们应该积极使用退出句柄，退出句柄常用于：
* 对执行完成的workflow进行清理
* 可以将执行完成的workflow结果通过介质途径发送通知，如邮件推送
* 可以将执行完成的workflow结果通过webhook途径发送通知，如消息推送
* 触发重新执行或者调用执行其他的
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: exit-handlers-
spec:
  entrypoint: intentional-fail
  onExit: exit-handler                  # invoke exit-hander template at end of the workflow
  templates:
  # primary workflow template
  - name: intentional-fail
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo intentional failure; exit 1"]
 
  # Exit handler templates
  # After the completion of the entrypoint template, the status of the
  # workflow is made available in the global variable {{workflow.status}}.
  # {{workflow.status}} will be one of: Succeeded, Failed, Error
  - name: exit-handler
    steps:
    - - name: notify
        template: send-email
      - name: celebrate
        template: celebrate
        when: "{{workflow.status}} == Succeeded"
      - name: cry
        template: cry
        when: "{{workflow.status}} != Succeeded"
  - name: send-email
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo send e-mail: {{workflow.name}} {{workflow.status}}"]
  - name: celebrate
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo hooray!"]
  - name: cry
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo boohoo!"]
```