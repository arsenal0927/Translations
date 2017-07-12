
# Draft: 使Kubernetes容器开发变得便捷

*今天的文章是来自微软Azure和Kubernetes联合创始人之一的工程总监，Brendan Burns.*

大约一个月前，微软宣布收购Deis来扩大我们在容器和Kubernetes方面的专业知识。今天，我很高兴地宣布，从这个新扩展的Azure团队中衍生出一个新的开源项目：Draft。

现今Kubernetes在部署和管理大规模应用程序方面有非常好的优势。但是在开发一个新的Kubernetes应用过程仍是非常困难。如果你是容器方面、Kubernetes或者开发应用上的新手，则更加困难。

Draft填补了这个角色。正如它名字所示，它是一个工具，~~可以帮助你开始在Kubernetes运行为容器应用准备的第一个Draft。~~当你第一次运行draft工具时，它会自动发现你正在处理的代码，并构建脚手架以支持对应用程序的容器化。~~使用启发式和各种预定义的项目模板Draft将创建一个初始的dockerfile来容纳你的应用，和Helm图表，~~以使你的应用能够在Kubernetes群集中进行部署和维护。团队甚至可以自己编写项目模板来定制由该工具构建的支架。

~~但是Draft的价值远远超出了一些文件中的脚手架，以帮助你创建应用程序。~~Draft还将服务器部署到你现有的Kubernetes群集中，并且自动与你的笔记本电脑保持同步。只要你修改了应用程序，笔记本电脑上的Draft守护进程程序会将该代码与Kubernetes中的Draft服务器进行同步，并自动构建和部署新的容器，而不需要用户进行任何操作。

~~Draft也为云的“内部循环”提供开发经验。当然，与今天所有基础设施软件的期望一样，~~Draft是可以作为一个开源项目的，~~并且他本身就是“草案”形式的我们热切地邀请社区中的成员前来，我们认为它相当棒，即使这还是早期的形式。~~同时我们非常高兴能看到如何围绕Draft发展一个社区，让它对所有在Kubernetes上的开发容器化应用程序的所有开发人员更有利。

为了给你展示Draft能够做什么，这里有一个从项目[GItHub仓库](https://github.com/Azure/draft)的[入门](https://github.com/Azure/draft/blob/master/docs/getting-started.md)页面展示的例子。

~~示例目录中包含了多个[示例应用程序](https://github.com/Azure/draft/tree/master/examples)。~~在这个演练中，我们将使用[Python示例应用](https://github.com/Azure/draft/tree/master/examples/python)，[Flask](http://flask.pocoo.org/)框架提供了一个非常简单的Hello World Web服务器。

	$ cd examples/python

## 创建一个Draft

我们需要一些“脚手架”部署我们应用程序到[Kubernetes](https://kubernetes.io/)集群中。Draft能够通过命令*draft create*创建一个*[Helm](https://github.com/kubernetes/helm)图表*，一个*Dockerfile*文件和一个*draft.toml*文件:

	$ draft create
	--> Python app detected
	--> Ready to sail
	$ ls
	Dockerfile  app.py  chart/  draft.toml  requirements.txt

被Draft创建的 *chart/*和*Dockerfiles*资源默认为基础的Python配置。这个*Dockerfile*文件利用了[python:onbuild image](https://hub.docker.com/_/python/)镜像，它将使用*requirements.txt*文件安装依赖，把当前目录复制到*/usr/src/app*目录中。并且要与*chart/values.yaml*文件中的值对其，这个*Dockerflie*从容器暴露了80端口。

*draft.toml* 文件包含与应用相关的基本配置，它的命名空间将会按此应用类似的名字部署。以及是否在本地文件更改时是自动部署应用程序。

	$ cat draft.toml
	[environments]
	  [environments.development]
	    name = "tufted-lamb"
	    namespace = "default"
	    watch = true
	    watch_delay = 2

## Draft Up

现在我们将准备好的 `app.py` 部署到Kubernetes集群。
Draft通过一个 `draft up` 命令来处理这些任务：
- 从 *draft.toml* 读取配置
- 将 *chart/* 目录和应用程序目录压缩成两个单独的压缩包
- 将压缩包上传到 *draftd* ，即服务器端组件
- 然后 *draftd* 将构建docker镜像并将这个镜像推送到注册表
- *draftd* 引用刚刚构建的Docker注册表镜像指导helm安装Helm图表

将 *watch* 选项设置为 *true*，我们可以让这个应用在后台运行，以便我们在后面做出改变...

	$ draft up
	--> Building Dockerfile
	Step 1 : FROM python:onbuild
	onbuild: Pulling from library/python
	...
	Successfully built 38f35b50162c
	--> Pushing docker.io/microsoft/tufted-lamb:5a3c633ae76c9bdb81b55f5d4a783398bf00658e
	The push refers to a repository [docker.io/microsoft/tufted-lamb]
	...
	5a3c633ae76c9bdb81b55f5d4a783398bf00658e: digest: sha256:9d9e9fdb8ee3139dd77a110fa2d2b87573c3ff5ec9c045db6009009d1c9ebf5b size: 16384
	--> Deploying to Kubernetes
	    Release "tufted-lamb" does not exist. Installing it now.
	--> Status: DEPLOYED
	--> Notes:
	     1. Get the application URL by running these commands:
	     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
	           You can watch the status of by running 'kubectl get svc -w tufted-lamb-tufted-lamb'
	  export SERVICE_IP=$(kubectl get svc --namespace default tufted-lamb-tufted-lamb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
	  echo http://$SERVICE_IP:80
	
	Watching local files for changes...

## 与已部属的应用程序进行交互

成功部署后我们就可以连接上我们的应用，进行非常便捷的输出了。请注意，由Kubernetes提供的负载平衡器可能需要几分钟时间。请耐心等待！

	$ export SERVICE_IP=$(kubectl get svc --namespace default tufted-lamb-tufted-lamb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
	$ curl http://$SERVICE_IP

当我们 `curl `应用程序时，就可以看到应用程序运行了。一个完美的“Hello World！”出现在面前。

## 升级应用

现在，把 `app.py` 文件输出的“Hello World！”修改成“Hello，Draft！”：

	$ cat <<EOF > app.py
	from flask import Flask
	
	app = Flask(__name__)
	
	@app.route("/")
	def hello():
	    return "Hello, Draft!\n"
	
	if __name__ == "__main__":
	    app.run(host='0.0.0.0', port=8080)
	EOF

## Draft更新

现在，如果我们看到最初的Draft终端，Draft将注意到在本地发生的变化，并且再次使用 `draft up`。然后Draft会确定Helm资源已经存在，并执行 `helm upgrade`，而不是尝试另外一个 `helm install`：

	--> Building Dockerfile
	Step 1 : FROM python:onbuild
	...
	Successfully built 9c90b0445146
	--> Pushing docker.io/microsoft/tufted-lamb:f031eb675112e2c942369a10815850a0b8bf190e
	The push refers to a repository [docker.io/microsoft/tufted-lamb]
	...
	--> Deploying to Kubernetes
	--> Status: DEPLOYED
	--> Notes:
	     1. Get the application URL by running these commands:
	     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
	           You can watch the status of by running 'kubectl get svc -w tufted-lamb-tufted-lamb'
	  export SERVICE_IP=$(kubectl get svc --namespace default tufted-lamb-tufted-lamb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
	  echo http://$SERVICE_IP:80

现在运行 `curl http://$SERVICE_IP `,我们的第一个应用程序就会通过Draft部署和更新到我们的Kubernetes集群！

我们希望这能使您了解Draft为简化Kubernetes开发所做的一切。

*--Brendan Burns，微软Azure创始人。*


