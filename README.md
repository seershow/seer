# Seer


## 1. 概述

一个应用程序编排系统，降低非容器化应用程序部署、扩缩容与管理的复杂性



## 2. 问题

目前仍存在不少业务场景不适合使用容器化方案，主要有两方面：

1. 比如数据库、大数据和对性能要求极高的系统等
2. 中小型团队，业务量不太大，人工运维又出现了瓶颈，k8s学习使用成本高，不适合使用

业界无统一的编排方案，导致各个团队重复造轮子



## 3. 解决思路

参考k8s Pod与Container的隔离设计思路，将每个应用程序及其依赖打包成一个package，调度时解压放到一个目录里，这样能整体干净地创建和移除。



## 4. 设计方案

总体设计参考k8s，Pod和Container的使用保持不变，唯一区别是没有了namespace的隔离与control-group的限制，程序自身的依赖通过打包成一个package解决，解压到一个目录就可以直接运行。

### 4.1. 架构

```
                      Master
     -----------------------------------------
    |                                         |
    |   ------                                |                                Node
    |  | Etcd |                               |                      ------------------------
    |   ------  *                             |                     |  [Pod1] [Pod2] [Pod3]  |
    |              * ------------             |                     |                        |
    |               |            |            |                     |       ---------        |
    |               | API-Server |*   *   *   |*   *   *   *   *   *|  * * | Seerlet |       |
    |               |            |            |                     |       ---------        |
    |                ------------             |                     |                        |
    |             *            *              |                     |                        |
    |          *                  *           |                       -----------------------                       
    |  --------------------     -----------   |              
    | | Controller-Manager |   | Scheduler |  |                 
    |  --------------------     -----------   |
    |                                         |
     -----------------------------------------
    

```



### 4.2. 声明式配置

配置跟k8s基本保持一致，区别是命名改为小写，多个单词用下滑线分开；只支持基本的能力，按需支持。

**单个Pod配置**

```yaml
# passport.yaml
# Pod 定义跟k8s基本保持一致，区别在k8s Pod中的command为单个命令，此处为多个命令commands
# 
api_version: v1
kind: pod
metadata:
  name: passport
spec:
  containers:
  - name: nginx
    package: hub.x.com/nginx:v1.9
    commands:                          
      start: sbin/nginx                 # 不以根“/"开头的路径，都是相对容器的根目录
      stop: sbin/nginx -s stop
      reload: sbin/nginx -s reload
      restart: sbin/nginx -s reopen
    ports:
    - name: http
    	container_port: 80
    - name: https
      container_port: 443
  - name: fpm
    package: hub.x.com/fpm:v7.0
    commands:
      start: sbin/fpm
    ports:
    - name: fpm-port
      container_port: 9000
  - name: passport
    package: hub.x.com/passport:v1.0
```



**Deployment配置**

```yaml
#
# passport-deployment.yaml
# 
api_version: v1
kind: deployment
metadata:
  name: passport-deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: passport
          package: hub.x.com/passport:v1.0
```



**Component配置**

为了解决多个服务共用同一个组件的需求，如：PHP Web服务，请求 --> 【Nginx】--> 【FPM】 --> 【PHP】，多个PHP Web服务会共用同样的Nginx和FPM进程。

我们引入Component的概念，Component不能单独部署运行，如果有其它Pod依赖它，就会自动部署。

```yaml
api_version: v1
kind: component
metadata:
  name: nginx-component
spec:
  template:
  	spec:
      containers:
        - name: nginx
          package: hub.x.com/nginx:v1.9
          sharing_dir: conf.d                   # 供依赖方写入配置文件
          commands:
            start: sbin/nginx
            stop: sbin/nginx -s stop
            reload: sbin/nginx -s reload
            restart: sbin/nginx -s reopen
          ports:
            - name: http
              container_port: 80
            - name: https
              container_port: 443

---

api_version: v1
kind: component
metadata:
  name: fpm-component
spec:
  template:
  	spec:
      containers:
        - name: fpm
          package: hub.x.com/fpm:v7.0
          commands:
            start: sbin/fpm
          ports:
            - name: fpm-port
              container_port: 9000

---

api_version: v1
kind: deployment
metadata:
  name: passport-deployment
spec:
  replicas: 3
  template:
    spec:
      required_components:
        - name: nginx-component
          config:
            passport.x.com.conf: | # 该配置将会写入组件【nginx-component】的sharing_dir
              server {
                  listen 80;
                  server_name passport.x.com
                  root /opt/web/passport/public;
                  index index.php index.html;

                  access_log /opt/log/nginx/access-passport.log main;
                  error_log /opt/log/nginx/error-passport.log;

                  fastcgi_intercept_errors on;

                  location / {
                      try_files $uri $uri/ /index.php?$args;
                  }

                  location ~ ^/index.php$ {
                      fastcgi_pass 127.0.0.1:9000;
                      fastcgi_index index.php;
                      include fastcgi.conf;
                  }
              }
        - name: fpm-component
      containers:
        - name: passport
          package: hub.x.com/passport:v1.0
          deployment_dir: /seer/web/passport   # 指定部署目录
```



### 4.3. 目录约定

采用了目录来隔离Pod和Container，那必须约定目录规范。

#### 4.3.1. 平台级目录

如下目录为seerlet在部署程序时必须使用的目录规范

**根目录**

```
/seer  # 默认值，可配置（集群级别统一）
```

**Pod根目录**

```
${ROOT_DIR}/pod
```

**单个Pod目录**

```
${ROOT_DIR}/pod/{pod-name}
```

**单个容器目录**

```
${ROOT_DIR}/pod/{pod-name}/container/{container-name}
```

#### 4.3.2. 程序使用的目录

为了方便日志收集，建议约定程序使用的目录，以下为建议值（不强求）

**日志目录**

```
/opt/var/log   # 单个容器子目录划分提前约定，平台不要求
```

**其它写根目录**

```
/opt/var/run   # 单个容器子目录划分提前约定，平台不要求
```

**临时目录**

```
/opt/var/tmp   
```

