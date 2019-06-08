# Level:Container
> Dockerfile
```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.7

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]

```
> requirements.txt
```text
Flask
Redis
```

>app.py
```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)

```
>Build the app
```s
$ ls
Dockerfile		app.py			requirements.txt
```


```s
docker build --tag=friendlyhello .
```

```s
$ docker image ls

REPOSITORY            TAG                 IMAGE ID
friendlyhello         latest              326387cea398
```
Note how the tag defaulted to latest. The full syntax for the tag option would be something like --tag=friendlyhello:v0.0.1

>Run the app
```s
docker run -p 4000:80 friendlyhello
docker run --rm -d -p 80:80/tcp friendlyehello:latest
```
http://localhost:4000

>Stop the app
```s
$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED
1fa4ab2cf395        friendlyhello       "python app.py"     28 seconds ago

docker container stop 1fa4ab2cf395
```

>Recap and cheat sheet (optional)
```s
docker build -t friendlyhello .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyhello  # Run "friendlyhello" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyhello         # Same thing, but in detached mode
docker container ls                                # List all running containers
docker container ls -a             # List all containers, even those not running
docker container stop <hash>           # Gracefully stop the specified container
docker container kill <hash>         # Force shutdown of the specified container
docker container rm <hash>        # Remove specified container from this machine
docker container rm $(docker container ls -a -q)         # Remove all containers
docker image ls -a                             # List all images on this machine
docker image rm <image id>            # Remove specified image from this machine
docker image rm $(docker image ls -a -q)   # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag                   # Run image from a registry
```

# Level:Services
>在分布式应用程序中，应用程序的不同部分称为**服务**。

例如，如果您设想一个视频共享站点，它可能包括:
1. 一个用于在**数据库-DB**中存储应用程序数据的服务、
2. 一个用于用户上传内容后在**后端-Django**进行视频代码转换的服务、
3. 一个用于**前端-Vue**的服务，等等。

>服务实际上只是生产中的容器。

*一个服务只运行一个映像*，但是它规范了映像运行的方式:
1. 应该使用*什么端口*，
2. 应该运行*多少个容器副本*，以便服务具有所需的容量，等等。

总结：扩展服务可以改变运行该软件的容器*实例的数量*，从而为流程中的服务*分配更多的计算资源*。

幸运的是，使用Docker平台很容易定义、运行和扩展服务——只需编写一个*docker-compose.yml*文件

>docker-compose.yml
```dockerfile
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
networks:
  webnet:
```
这个docker-compose.yml文件告诉docker执行以下操作:

- 从注册表中取出我们在步骤2中上传的图像。

- 将该映像作为一种称为网络的服务运行5个实例，限制每个实例最多使用10%的单个内核的CPU时间(也可以是“1.5”表示每个内核有1.5个内核)，以及50MB的内存。

- 如果出现故障，立即重启容器。

- 将主机上的端口4000映射到网络的端口80。

- 通过一个叫做**webnet的负载平衡网络**，指示web容器共享端口80。(在内部，容器本身在一个短暂的端口发布到网络的端口80。)

- 用默认设置定义网络网络(这是一个负载平衡的覆盖网络)。

> 运行新的负载平衡应用程序

在进行服务部署前

Before we can use the docker stack deploy command we first run:

    docker swarm init
如果不运行docker swarm init，则会得到一个错误，即该节点不是群集管理器。

现在让我们运行它。你需要给你的应用命名。这里，它被设置为getstartedlab:

    docker stack deploy -c docker-compose.yml getstartedlab

我们的单一服务堆栈在一台主机上运行5个已部署映像的容器实例。让我们调查一下。

>获取我们应用程序中某项服务的服务标识

您可以运行docker堆栈服务，然后是堆栈的名称。下面的示例命令允许您查看与getstartedlab堆栈关联的所有服务
```
docker stack services getstartedlab
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
bqpve1djnk0x        getstartedlab_web   replicated          5/5                 username/repo:tag   *:4000->80/tcp
```
> 服务中运行的单个容器称为**任务**

任务被赋予惟一的id，该id以数字递增，直到您在docker- composition .yml中定义的副本数量为止。

列出服务的任务:`docker service ps getstartedlab_web`

>转到浏览器中的[URL](http://localhost:4000)，点击刷新几次。

无论哪种方式，容器ID都会发生变化，以显示负载平衡;对于每个请求，以循环方式选择5个任务中的一个来响应。容器id匹配上一个命令的输出
```
docker stack ps getstartedlab
ID                  NAME                  IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
uwiaw67sc0eh        getstartedlab_web.1   username/repo:tag   docker-desktop      Running             Running 9 minutes ago                       
sk50xbhmcae7        getstartedlab_web.2   username/repo:tag   docker-desktop      Running             Running 9 minutes ago                       
c4uuw5i6h02j        getstartedlab_web.3   username/repo:tag   docker-desktop      Running             Running 9 minutes ago                       
0dyb70ixu25s        getstartedlab_web.4   username/repo:tag   docker-desktop      Running             Running 9 minutes ago                       
aocrb88ap8b0        getstartedlab_web.5   username/repo:tag   docker-desktop      Running             Running 9 minutes ago
```

>拓展服务

您可以通过更改`docker-compose.yml`中的`replicas`值来扩展应用程序。保存更改并重新运行docker堆栈部署命令`docker stack deploy`
```
docker stack deploy -c docker-compose.yml getstartedlab
```
Docker执行就地更新，不需要先拆下堆栈或杀死任何容器。现在，重新运行`docker container ls -q`，查看重新配置的已部署实例。如果您放大副本，就会启动更多的任务，从而启动更多的容器。

>取下应用程序和群体


Take the app down with docker stack rm:

    docker stack rm getstartedlab

Take down the swarm.

    docker swarm leave --force

> 一些新命令
```s
docker stack ls                                            # List stacks or apps
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker service ls                 # List running services associated with an app
docker service ps <service>                  # List tasks associated with an app
docker inspect <task or container>                   # Inspect task or container
docker container ls -q                                      # List container IDs
docker stack rm <appname>                             # Tear down an application
docker swarm leave --force      # Take down a single node swarm from the manager
```

# Lecel:Swarm
1.准备
需要虚拟交换机`Primary virtual switch`  
myswitch
准备2个`virtual host`

In Git Bash(administrator)
```s
$ echo $PWD
/l/var/Hyper-V/my-docker-machine

docker-machine -s $PWD \
create -d hyperv \
--engine-registry-mirror https://td9f8xn7.mirror.aliyuncs.com \
--hyperv-virtual-switch "myswitch" \
--hyperv-memory "1024" \
manager

$ docker-machine -s $PWD ls
NAME      ACTIVE   DRIVER   STATE     URL                        SWARM   DOCKER     ERRORS
manager   -        hyperv   Running   tcp://192.168.0.102:2376           v18.09.6

docker-machine -s $PWD \
create -d hyperv \
--engine-registry-mirror https://td9f8xn7.mirror.aliyuncs.com \
--hyperv-virtual-switch "myswitch" \
--hyperv-memory "1024" \
worker

$ docker-machine -s $PWD ls
NAME      ACTIVE   DRIVER   STATE     URL                        SWARM   DOCKER     ERRORS
manager   -        hyperv   Running   tcp://192.168.0.102:2376           v18.09.6
worker    -        hyperv   Running   tcp://192.168.0.103:2376           v18.09.6

```
踩坑记录：  
1. 控制虚拟机内存大小，HOST内存不够也会失败,至少要1024
2. -s $PWD 制定在当前路径下的machine

```
$ docker-machine ls
NAME       ACTIVE   DRIVER   STATE     URL                        SWARM   DOCKER     ERRORS
manager1   -        hyperv   Running   tcp://192.168.0.104:2376           v18.09.6
worker1    -        hyperv   Running   tcp://192.168.0.105:2376           v18.09.6
worker2    -        hyperv   Stopped                                      Unknown
```
2.Swarm manager
```s
$ docker-machine ssh manager1 "docker swarm init --advertise-addr 192.168.0.104"
Swarm initialized: current node (ydfpt03uvyhapeh8xr41fooff) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2m1uojx83emz3jjl8670sm3ghfjlevj5908hedvc94rgrip0sl-d3upr9dw5dmik0pempd7gwkfi 192.168.0.104:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
>NOTE: Ports 2377 and 2376
>
>Always run docker swarm init and docker swarm join with port 2377 (the swarm management port), or no port at all and let it take the default.
>
>The machine IP addresses returned by docker-machine ls include port 2376, which is the Docker daemon port. Do not use this port or you may experience errors.

3.Swarm worker
```s
$ docker-machine ssh worker1 "docker swarm join --token SWMTKN-1-2m1uojx83emz3jjl8670sm3ghfjlevj5908hedvc94rgrip0sl-d3upr9dw5dmik0pempd7gwkfi 192.168.0.104:2377"

This node joined a swarm as a worker.

```

4.确认Swarm是否成功  
进入manager  
In Git Bash(管理员)
```s
$ docker-machine env manager1
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.0.104:2376"
export DOCKER_CERT_PATH="C:\Users\Administrator\.docker\machine\machines\manager1"
export DOCKER_MACHINE_NAME="manager1"
export COMPOSE_CONVERT_WINDOWS_PATHS="true"
# Run this command to configure your shell:
# eval $("C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env manager1)

$ eval $("C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env manager1)

$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
ydfpt03uvyhapeh8xr41fooff *   manager1            Ready               Active              Leader              18.09.6
lrp0fkxjbosjyndrder0yn6ae     worker1             Ready               Active                                  18.09.6

```
如果想要重头再来一次操作,you can run `docker swarm leave` for *each node*.

5.部署app到Swarm集群  
注意，只能通过`manager`来执行`Docker Command`.  
有两种方法通过`manager`执行命令
  - you’ve been wrapping Docker commands in `docker-machine ssh` to talk to the VMs

  - Another option is to run `docker-machine env <machine>` to get and run a command that configures your current shell to talk to the Docker daemon on the VM.  
  推荐使用这种方法  
  This method works better for the next step because **it allows you to use your local docker-compose.yml file to deploy the app “remotely” without having to copy it anywhere.**


In Git Bash(管理员)
```s
$ docker-machine ls
NAME       ACTIVE   DRIVER   STATE     URL                        SWARM   DOCKER     ERRORS
manager1   *        hyperv   Running   tcp://192.168.0.104:2376           v18.09.6
worker1    -        hyperv   Running   tcp://192.168.0.105:2376           v18.09.6
worker2    -        hyperv   Stopped                                      Unknown

# 可以看到当前SHELL已经进入manager VM，manager has been actived
$ docker stack deploy -c docker-compose.yml getstartedlab
Creating network getstartedlab_webnet
Creating service getstartedlab_web
Creating service getstartedlab_visualizer

```

```s
docker-machine create --driver virtualbox myvm1 # Create a VM (Mac, Win7, Linux)
docker-machine create -d hyperv --hyperv-virtual-switch "myswitch" myvm1 # Win10
docker-machine env myvm1                # View basic information about your node
docker-machine ssh myvm1 "docker node ls"         # List the nodes in your swarm
docker-machine ssh myvm1 "docker node inspect <node ID>"        # Inspect a node
docker-machine ssh myvm1 "docker swarm join-token -q worker"   # View join token
docker-machine ssh myvm1   # Open an SSH session with the VM; type "exit" to end
docker node ls                # View nodes in swarm (while logged on to manager)
docker-machine ssh myvm2 "docker swarm leave"  # Make the worker leave the swarm
docker-machine ssh myvm1 "docker swarm leave -f" # Make master leave, kill swarm
docker-machine ls # list VMs, asterisk shows which VM this shell is talking to
docker-machine start myvm1            # Start a VM that is currently not running
docker-machine env myvm1      # show environment variables and command for myvm1
eval $(docker-machine env myvm1)         # Mac command to connect shell to myvm1
& "C:\Program Files\Docker\Docker\Resources\bin\docker-machine.exe" env myvm1 | Invoke-Expression   # Windows command to connect shell to myvm1
docker stack deploy -c <file> <app>  # Deploy an app; command shell must be set to talk to manager (myvm1), uses local Compose file
docker-machine scp docker-compose.yml myvm1:~ # Copy file to node's home dir (only required if you use ssh to connect to manager and deploy the app)
docker-machine ssh myvm1 "docker stack deploy -c <file> <app>"   # Deploy an app using ssh (you must have first copied the Compose file to myvm1)
eval $(docker-machine env -u)     # Disconnect shell from VMs, use native docker
docker-machine stop $(docker-machine ls -q)               # Stop all running VMs
docker-machine rm $(docker-machine ls -q) # Delete all VMs and their disk images
```

# Level:Stack
