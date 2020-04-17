---
layout: post
title: "「devops」argo workflow笔记(四)"
subtitle: 'argo python adls支持库分析'
author: "odin"
header-style: text
tags:
  - DevOps
---
![]({{site.baseurl}}/img/in-post/post-devops/argo-log.jpg)

### 写在前面
如果您不熟悉Argo，建议您以纯YAML格式查看示例. 该语言是描述性的，Argo 示例提供了详尽的解释.
对于经验丰富的受众，此DSL授予您使用Python编程定义Argo工作流的能力，然后将其转换为Argo YAML规范.
DSL使用在Argo Python客户端存储库中定义的Argo模型. 结合这两种方法，我们得到了对Argo Workflows的整个底层控制.
```python
# 首先安装python-argo客户端
pip install argo-workflows
# 再安装argo-workflow-dsl
pip install argo-workflow-dsl
```
### argo-client-python
与Kubernetes client很类似，这里不做过多分析，参考github即可。[argo-client-python](https://github.com/CermakM/argo-client-python)

### argo-python-dsl
[argo-python-dsl](https://github.com/CermakM/argo-python-dsl) 将argo主要的几种template模式实现在argo.workflows.dsl下的template.py中，通过引用template，添加相应的模板装饰器即可，函数对象式实现argo adsl。
下面是几种argo-python-dsl与argo adsl的对比
#### 容器模板
```python
from argo.workflows.dsl import Workflow
from argo.workflows.dsl import template
 
from argo.workflows.dsl.templates import V1Container
 
 
class HelloWorld(Workflow):
 
    entrypoint = "whalesay"
 
    @template
    def whalesay(self) -> V1Container:
        container = V1Container(
            image="docker/whalesay:latest",
            name="whalesay",
            command=["cowsay"],
            args=["hello world"]
        )
 
        return container
```
转换为adsl如下
```yaml
# @file: hello-world.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: hello-world
  generateName: hello-world-
spec:
  entrypoint: whalesay
  templates:
  - name: whalesay
    container:
      name: whalesay
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["hello world"]
```
#### DAG模板
```python
from argo.workflows.dsl import Workflow
 
from argo.workflows.dsl.tasks import *
from argo.workflows.dsl.templates import *
 
 
class DagDiamond(Workflow):
 
    @task
    @parameter(name="message", value="A")
    def A(self, message: V1alpha1Parameter) -> V1alpha1Template:
        return self.echo(message=message)
 
    @task
    @parameter(name="message", value="B")
    @dependencies(["A"])
    def B(self, message: V1alpha1Parameter) -> V1alpha1Template:
        return self.echo(message=message)
 
    @task
    @parameter(name="message", value="C")
    @dependencies(["A"])
    def C(self, message: V1alpha1Parameter) -> V1alpha1Template:
        return self.echo(message=message)
 
    @task
    @parameter(name="message", value="D")
    @dependencies(["B", "C"])
    def D(self, message: V1alpha1Parameter) -> V1alpha1Template:
        return self.echo(message=message)
 
    @template
    @inputs.parameter(name="message")
    def echo(self, message: V1alpha1Parameter) -> V1Container:
        container = V1Container(
            image="alpine:3.7",
            name="echo",
            command=["echo", "{{inputs.parameters.message}}"],
        )
 
        return container
```
转换为adsl如下
```yaml
# @file: dag-diamond.yaml
# The following workflow executes a diamond workflow
#
#   A
#  / \
# B   C
#  \ /
#   D
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: dag-diamond
  generateName: dag-diamond-
spec:
  entrypoint: main
  templates:
  - name: main
    dag:
      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        dependencies: [B, C]
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
 
  # @task: [A, B, C, D]
  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      name: echo
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]
```
#### 传递参数模板
```python
from argo.workflows.dsl import Workflow
 
 
from argo.workflows.dsl.tasks import *
from argo.workflows.dsl.templates import *
 
class ArtifactPassing(Workflow):
 
    @task
    def generate_artifact(self) -> V1alpha1Template:
        return self.whalesay()
 
    @task
    @artifact(
        name="message",
        _from="{{tasks.generate-artifact.outputs.artifacts.hello-art}}"
    )
    def consume_artifact(self, message: V1alpha1Artifact) -> V1alpha1Template:
        return self.print_message(message=message)
 
    @template
    @outputs.artifact(name="hello-art", path="/tmp/hello_world.txt")
    def whalesay(self) -> V1Container:
        container = V1Container(
            name="whalesay",
            image="docker/whalesay:latest",
            command=["sh", "-c"],
            args=["cowsay hello world | tee /tmp/hello_world.txt"]
        )
 
        return container
 
    @template
    @inputs.artifact(name="message", path="/tmp/message")
    def print_message(self, message: V1alpha1Artifact) -> V1Container:
        container = V1Container(
            name="print-message",
            image="alpine:latest",
            command=["sh", "-c"],
            args=["cat", "/tmp/message"],
        )
 
        return container
```
转换为adsl如下
```yaml
# @file: artifacts.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: artifact-passing
  generateName: artifact-passing-
spec:
  entrypoint: main
  templates:
  - name: main
    dag:
      tasks:
      - name: generate-artifact
        template: whalesay
      - name: consume-artifact
        template: print-message
        arguments:
          artifacts:
          # bind message to the hello-art artifact
          # generated by the generate-artifact step
          - name: message
            from: "{{tasks.generate-artifact.outputs.artifacts.hello-art}}"
 
  - name: whalesay
    container:
      name: "whalesay"
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["cowsay hello world | tee /tmp/hello_world.txt"]
    outputs:
      artifacts:
      # generate hello-art artifact from /tmp/hello_world.txt
      # artifacts can be directories as well as files
      - name: hello-art
        path: /tmp/hello_world.txt
 
  - name: print-message
    inputs:
      artifacts:
      # unpack the message input artifact
      # and put it at /tmp/message
      - name: message
        path: /tmp/message
    container:
      name: "print-message"
      image: alpine:latest
      command: [sh, -c]
      args: ["cat", "/tmp/message"]
```
#### 脚本模板
最有趣的部分，在argo adsl中支持脚本的形式是在yaml文件中source关键字处以文本形式编写代码，argo-python-dsl提供@closure和@scope两种装饰器，提供两种不同的对script风格的支持，后者更加符合pythonic
```python
# @template形式，以字符串形式赋值给source
import textwrap
 
class ScriptsPython(Workflow):
 
    ...
 
    @template
    def gen_random_int(self) -> V1alpha1ScriptTemplate:
        source = textwrap.dedent("""\
          import random
          i = random.randint(1, 100)
          print(i)
        """)
 
        template = V1alpha1ScriptTemplate(
            image="python:alpine3.6",
            name="gen-random-int",
            command=["python"],
            source=source
        )
 
        return template
```     
```python
# @closure形式，符合我们的风格
class ScriptsPython(Workflow):
 
    ...
 
    @closure(
      image="python:alpine3.6"
    )
    def gen_random_int() -> V1alpha1ScriptTemplate:
          import random
 
          i = random.randint(1, 100)
          print(i)
```  
```python
# 使用@scope可更进一步包装，支持避免单个函数过于复杂，可以将部分逻辑拆分至@scope中
  ...
 
 @closure(
   scope="main",
   image="python:alpine3.6"
 )
 def gen_random_int(main) -> V1alpha1ScriptTemplate:
       import random
 
       main.init_logging()
 
       i = random.randint(1, 100)
       print(i)
 
 @scope(name="main")
 def init_logging(level="DEBUG"):
     import logging
 
     logging_level = getattr(logging, level, "INFO")
     logging.getLogger("__main__").setLevel(logging_level)
```