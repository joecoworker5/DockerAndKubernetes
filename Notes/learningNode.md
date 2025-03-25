# Docker section2 Docker Images & containers: the core building block
## 多使用 --help 來使用 docker
## Write dockerfile and build images
- dockerfile 參考 [link](../section2/nodejs-app-finished/Dockerfile)
```
docker build -t name:tagname .(dockerfile所在地)
```

## run container with some custom options
``` 
docker run -p 3000:80(3000 代表本機port: 80 container對外的port) -d(detach) --rm(當container stop後自動刪除) --name containerName imageId
```
![image](./1.png)

## UnderStanding image layers (video.27)
dockerfile 每個 instruction 都是一層 layer
## Understand attach and detached Options(video.29)
attachMode 代表我們會監聽 container 的 output(error/ console.log etc) 顯示在 terminal 中, detach 讓container在背景執行
- docker run 預設為 attach, 所以如果 run image 的話, terminal 會自動監聽 container, 加上 -d 讓 container 在背景執行
- docker start 預設為 detach
```
docker run -p 8000:80 -d image
```

## Entering Interactive mode(v.31)
docker images 並非都是 server, 有些只是一段程式跑完就結束了, 當我們想要透過 terminal input 內容和 container 互動的話可以使用 -it 選項
```
docker run -it pythontest:1.0.0
docker start -a -i containerId(start 預設為 detach, 當需要attach 和互動則需要 -a -i options)
```

## Inspect image configuration
```
docker image inspect imageId
```
## delete image/container
```
docker rm containerId1 containerId2
docker rmi imageId1 imageId2
docker image prune -a(remove all images)
```

## Copy file into/from container(see docker cp --help)
```
docker cp ./Notes/. containerId:/test
docker cp containerId:/test  .
```

## practice
[link](../section2/assignment-problem/instructions.txt)

## Pushing/pull images to private dockerhub(v.38)
在 dockerhub 建立 private repo https://hub.docker.com/repository/docker/joesu0610/images_family/general
```
docker tag nodeserver:1.0.0 joesu0610/images_family:1.0.0 (rename tag, 原本的 image 一樣會保留)
docker login (因為是 private repo)
docker push joesu0610/images_family:1.0.0
docker pull joesu0610/images_family:1.0.0(如果未指定1.0.0, 會拉最新的)
```


# Docker section 3: Managing Data & working with volumes
container stop/start container 內檔案仍然會留著, 但被 rm 時在建新的 container時, 其內的 file 會消失
所以我們該如何解決 container rm 後檔案會消失的問題呢? 使用**docker volumes**

## Docker Volumes
Volumes are folders on your host machine hard drive which are mounted (“made available”, mapped) into containers, 所以當 container 消失時, 檔案仍然會存在 local machine 當中
![image](./2.png)
![image](./3.png)
主要分下列三種(volume: local path managed by docker automatic, mount: local path 由自己處理)
1. ananymous volume, 可以在 Dockerfile 中定義也可以在 run images 指定 **-v /app**
    如果 container run 時有加 -rm 參數, 當 container stop 時, ananymous 會消失, 但如果沒有加 -rm 參數, ananymous volume 其實不會在 container rm 時消失, 但當又 run 新的 container 時又會產生新的 ananymous volume, 無法指向原本的舊的 ananymous volume, 因此也無法獲得過去的資料, 可以透過下面指令清除 volume
    - 使用情境為當你需要確保 container 某些 file 不會被其他外部 external path 修改時
    ```
        docker volume ls (列出有哪些 volume)
        docker volume rm VOL_NAME
        docker volume prune
    ```
    ```
    ...
    EXPOSE 80
    # 此為 ananomous volume /app/feedback 代表 container 內要保存的資料
    VOLUME ["/app/feedback"] 
    CMD ["node", "server.js"]
    ```
2. Name volume 不是定義在 Dockerfile 中, 而是在 run docker images 時指定
    -v feedback:/app/feedback (feedback 為 volume name, /app/feedback 為 container 想保存的資料)
    ```
    docker run -p 3000:80 -d --name feed-app --rm -v feedback:/app/feedback feedback-node:volumes
    docker volume inspect 可以看到 feedback name volume
    docker stop 
    docker volume inspect 仍然可以看到 feedback name volume
    docker run -p 3000:80 -d --name feed-app --rm -v feedback:/app/feedback feedback-node:volumes 建立新的 container 一樣指向到 feedback volume 就可以得到之前儲存的 local data
    ``` 

3. Mount(local path 由自己指定, 可以修改裡面的內容同時反映在container中 e.g source code(熱部署的感覺)), 一樣是在 run dokcer image 時指定（video.53)
   - local path 必需要是絕對路徑或可使用 $(pwd) 簡化
   - 使用 mountm 當修改 local path 檔案內容時, container 也會同時受到影響(熱部署)
   - 必須要給予 container 權限來寫入 local path 檔案, 須在 docker->setting -> resource/fileSharing 下加入需要 mount 的 folder 或 parent folder
   - 當存在多個 volume/mount 時, container Internal path 越長, 優先序越高
   ```
    docker run -p 3000:80 -d --name feed-app -v feedback:/app/feedback  -v "/Users/joe.su/IdeaProjects/docker and kubernetes resource/section3/data-volumes-01-starting-setup:/app" feedback-node:volumes
    或
    docker run -p 3000:80 -d --name feed-app -v feedback:/app/feedback -v $(pwd):/app feedback-node:volumes

    
    但卻發現 container 起不起來, 因為找不到 node module, 原因是因為 local data-volumes-01-starting-setup mount /app, 將原本 Dockerfile 一開始 install 的 node_module override, 但本地資料夾沒有 node_module 導致 container 沒有 node_module 而啟動失敗, 解決方法為不要 override node_modules, 此時可以使用 -v /app/node_modules(ananymous container), 當同時存在多個 volume/mount, docker 會根據 container 的 Internal path 長短決定優先序, path越長則會override其他, 因為 /app/node_modules > /app 所以不會受到 mount override 的影響

    docker run -p 3000:80 -d --name feed-app -v feedback:/app/feedback  -v $(pwd):/app 
    -v /app/node_modules feedback-node:volumes

    此時 container 就可以正常起動, 且改動本地檔案時, container 內容也會即時刷新

**下面為這三種方式的 summary 和 比較**
![image](./4.png)
![image](./5.png)

**Manange docker volumes(v.57)**
```
docker volume --help
docker volume ls
docker volume inspect volumeId （MountPoint 為 docker 自動產生的 local path(基本上找不到)
docker volume rm volumeId (注意 remove volume 後, container 中和 volume 連結的檔案會消失!!)
```

    

## NodeJS-specific adjustment: Using Nodemon in a Container(參考 data-volumes-04-added-nodemon)
雖然 container 已經透過 mount 到 local file, 理論上當本地 js file 改變時, container 的 js file 也會同步改變(console.log('TEST!!!')), 但 log container 卻沒看到變化, 原因是 node server 造成, 必須要 restart node server in container 才會生效, 為了解決此問題可以透過安裝 nodemon 來監控 file 的變化並即時 restart node server
下面為所需設定
**package.json**
```
  "scripts": {
    "start": "nodemon server.js"
  },
  ...
  "devDependencies": {
    "nodemon": "2.0.4"
  }
```
**Dockerfile**
```
CMD [ "npm", "start" ]
```

## Read-Only Volumes
**-v $(pwd):/app:ro** 中的 ro(read-only), 代表 app folder 在 container 當中為唯獨無法寫入, 只能透過 local file system 來寫入, 確保不會因為 container 檔案寫入而影響 local 的檔案, 但 container 當中又需要寫入 /app/feedback 和 /app/temp 資料夾, 所以加上 **-v feedback:/app/feedback**, **-v /app/temp** 來 override 先前 /app:ro 的設定, 確保這兩個資料夾可以被寫入不受 mount 設定影響
```
   docker run -p 3000:80 -d --name feed-app -v feedback:/app/feedback  -v $(pwd):/app:ro 
    -v /app/node_modules -v /app/temp feedback-node:volumes
```

## .dockerignore 讓你可以指定哪些 file/folder 不要被 COPY 進 image, 類似 gitIgnore

## Working with Environment variables & .env files(v.61 data-volumes-08-args-and-env)
  ![image](./6.png)
  - 不要在 Dockerfile 裡面寫機密的的內容, 否則透過 **docker history imageId** 都可以看到, 盡量用
    Evironment variable, 且不要把 .env 等機密文件commit到 .git
  - ARG 只用在 build 
  - ENV 則用在 runtime
  **Dockerfile**
  ```
  ARG DEFAULT_PORT=80 // build image 時可以指定 --build-arg DEFAULT_PORT=90 複寫

  ENV PORT $DEFAULT_PORT // run container 時可以透過 --env PORT=300 複寫, 或讀 .env檔案

  EXPOSE $PORT

  CMD [ "npm", "start" ]
  ```
  **server.js**
  ```
    app.listen(process.env.PORT); // 吃 PORT 環境參數, 各種語言都有開放吃環境參數功能
  ```
  **.env**
  ```
  PORT=1234
  ```
  ```
    docker build -t feedback-node:env --build-arg DEFAULT_PORT=9000 . (複寫 DEFAULT_PORT=9000)
    docker run -p 3000:80 -d --name feedback-app --rm f6837b4354f0 (指向不到 container 因為這時候 PORT 預設為 9000)
    docker run -p 3000:9000 -d --name feedback-app --rm f6837b4354f0 (可指向 PORT 9000 container)

    docker run -p 3000:1234 -d --env(-e等效) PORT=1234 --name feedback-app --rm f6837b4354f0 (env指定 PORT為 1234, 同時也改成 3000:1234, 可正確指向)
    docker run -p 3000:1234 -d -e key=value -e key=value  --name feedback-app --rm f6837b4354f0 (多個 env)

    docker run -p 3000:1234 -d --env-file ./.env --name feedback-app --rm f6837b4354f0(直接讀設定檔)
```
## Module Summary
  ![image](./7.png)
  ![image](./8.png)


# Docker section4: Networking cross-container communication
  ![image](./9.png)
  ![image](./10.png)
  ref: https://azole.medium.com/docker-container-%E5%9F%BA%E7%A4%8E%E5%85%A5%E9%96%80%E7%AF%87-2-c14d8f852ae4
## Case1: Container <-> Outside Network
不需要什麼設定 container 就可以和外面的 Network 聯繫

## Case2: Container <-> Your HostMachine
localhost:1234 -> **host.docker.internal**:1234 (docker會自動幫你轉成本機的IP)

## Case3: Container <-> Container
Node Container and Mongo DB Container for example
```
docker container inspect containerId 找到 container 的 IP address ex: 172.17.0.3, 預設 network 是 docker0(bridge)
在 bridge 下,只能透過 ip 連上 mongoDB container, 所以 url 為 mongodb://172.17.0.3:27017, 但這樣每次建立新的 mongo DB container 時 ip 位址就有可能改變, 導致 node server 無法正確指向 DB

所以我們可以透過建立新的 docker network, 並將 container 指定在這個 network 之下, 這樣 container 之間就可以直間用 containerName 來互相連線 i.e mongodb://mongodb:27017

docker network create favorites-net
docker network ls (可以看到新建立的 favorites-net)
docker run -d --name mongodb --network favorites-net --rm mongo ( --network network名稱 指定 container network)
ocker run -d --name favorite --rm --network favorites-net favorites-node
docker network inspect favorites-net(可以看到 mongo, node server 兩 container)
此時兩個 container 就可以透過 container name 來互相溝通
```

## Docker Network Driver(v.76)
```
docker network create  --driver bridge(default)/host network-name
```


# Docker section5: Building multi-Container Application with Docker 
  ![image](./11.png)
1. mongodb
  ```
  docker network create network-interface
  docker network ls
  (不一定需要 -p 27017:27017, 因為 backend 和 mongodb 在相同 network 下可以直接用 containerName 互相連線)
  docker run -d --network network-interface --name mongodb -v dbvolume:/data/db \
        -e MONGO_INITDB_ROOT_USERNAME=joe \
        -e MONGO_INITDB_ROOT_PASSWORD=password \
        mongo
  ```
  - mongodb authentication error: https://www.udemy.com/course/docker-kubernetes-the-practical-guide/learn/lecture/22626647/#questions/13014650
  - mongodb container 資料都存在 /data/db 中, 所以透過 name volume, -v dbvolume:/data/db 我們可以保存 db 資料及使 container 被 rm


2. backend
   **app.js**
   ```
   mongodb://${process.env.MONGODB_USERNAME}:${process.env.MONGODB_PASSWORD}@mongodb:27017/course-goals?authSource=admin
   ```
   **Dockerfile**
   ```
   ENV MONGODB_USERNAME=root
   ENV MONGODB_PASSWORD=secret
   ```
    ```
    docker build -t frontend .
    docker run -p -d 80:80 --name backend --rm -v "$(pwd):/app" -v /app/node_modules -v backendVolume:/app/logs --network network-interface  -e MONGODB_USERNAME=joe -e MONGODB_PASSWORD=password backend
    ```
  - 透過 ENV, 我們可以透過 -e MONGODB_USERNAME=joe -e MONGODB_PASSWORD=password 來彈性指定要登入資料庫的帳號
  - React code, bundle into index.js, 是透過 browser 發 API 給後端, 因此後端的 container 需要 -p 80:80 讓外部可以連到 container
  -   因為需要和 mongodb 連線所以需要和 mongdb container註冊在相同 network-interface 下, 可以用 **mongodb://mongodb:27017** containerName 和 mongodb連線
  -  **-v "$(pwd):/app"** 確保 localfile 更新會即時更新到 container中(熱部署, 搭配 nodemon 確保每次 js file有變動, 就 restart server) **-v /app/node_modules -v backendVolume:/app/logs** 確保 node_modules 和 app/logs 不會被外部 localfile 複寫, 可以分別儲存在 anonymous/name volume 當中
  - 撰寫 .dockerignore 指定node_modules(local的可能是舊的或有版本不相容問題), .git Dockerfile 不要COPY 到 image 中

3. frontend(React)
  ```
  docker build -t frontend .
  docker run -p 3000:3000 -d --name frontend -v "$(pwd):/app" -v:/app/node_modules --rm frontend

  ```

## Room for Improvement
  ![image](./12.png)
  - 每次 build run container 都要下很長的指令, 下 section 會優化此問題
  - 本 module 主要是用在 dev 開發, 實際上 prod 不會希望有 localfile 影響 container 的事情發生(每個 container 應該是 isolated 不受外部影響, 才不會動了某個file container就壞掉)

# Docker section6: Docker Compose: elegant multi-container Orchestration
- docker compose 取代 multi-container 繁雜的 build run(如 docker-commands.txt), 統一定義在一個檔案中, 看起來更簡潔
- docker compose 不會取代 Dockerfile 或是 image / container 只是比較方便
- https://docs.docker.com/compose/compose-file/
![image](./13.png)
![image](./14.png)
![image](./15.png)
- 撰寫 docker-compose.yml(參考 [link](../section6/compose-02-finished/docker-compose.yml)
- docker compose 相關指令
```
docker-compose up -d(detach mode)
docker-compose down(會刪除   container network, 不會移除 volume)
docker-compose down -v(會移除 volume, 不過通常不會用到, 因為希望資料可以被保存下來)
docker-compose up --help
docker-compose up --build
docker-compose build (build image)
```

# Docker section7: Working with utility container & Executing Commands in containers
本機我們需要安裝 node 環境才可以執行相關 npm 指令, utility container 提供 node 環境, 讓我們可以直接進到 container 中執行指令
![image](./16.png)
```
docker run -it -d node
docker exec -it 54275387fc77
進到 container 下 npm init 等 node 相關指令
```

有時候不想安裝相關環境但又想執行某些指令該怎麼辦? 用 utility container, 然後透過 mount 將本機的 file mount 到 container 就可以在 container執行指令
**Dockerfile**
```
FROM node:14-alpine
WORKDIR /app
```
```
docker build -t node-util
docker run -it -v "$(pwd):/app" node-util npm init (在 image 後面的 npm init 代表 container 初始指令, 相當於 Dockerfile 中的 CMD ["npm", "init"], 又因為 mount 的關係, 所以本機也會有 package.json)
```
## utilizing ENRTYPOINT(v.106)
run image 後的指令會拼在 ENTRYPOINT 之後
```
FROM node:14-alpine
WORKDIR /app
ENTRYPOINT [ "npm" ]
```
```
docker build -t node-util
docker run -it -v "$(pwd):/app" node-util init (在 image 後面的 ENTRYPOINT的 npm + init 變成 npm init)

docker build -t node-util
docker run -it -v "$(pwd):/app" node-util install express --save(npm install express module)
```

## 轉換成 docker-compose.yml 簡化 build run
```
version: "3.8"
services:
  npmcontainer:
    build: ./ # Dockerfile 路徑
    stdin_open: true,
    tty: true
    volumes:
      - ./:/app
```
```
  docker-compose run --rm npmContainer(containerName) init
  (docker-compose run 讓我們可以 run 單一 utility container, 但不會自動 rm container 當 container stop, 所以加上 --rm, npm(entrypoint) + init)
```

# Docker section8: A laravel & PHP Dockerized project
![image](./17.png)
- 本章節重點就是本機不用安裝各種環境, 透過 Docker 和撰寫 dockerfile, docker-compose.yml 搭配 localfile source code 檔案映射到 container 就可以起 container(v.121重點)
- [link](../section8/laravel-04-fixed/docker-compose.yaml)
- 
```
docker-compose run --rm composer create-project --prefer-dist laravel/laravel .
docker-compose up container1 container2 container3 (指定 build run 哪些 container)
docker-compose up -d --build server (因為 depends_on 會啟動php, mysql, --build force rebuild image)
```

## 什麼時候用 COPY 什麼時候用 mount
for dev 可以用 mount, 方便開發, 當要部署到 prod 的時候, 我們需要將 localfile copy 到 image 當中且希望 container isolated 這時候用 COPY



# Docker section 9: Deploying Docker containers
Development to Production
![image](./18.png)
1. A Basic First Example: Standalone NodeJS App
![image](./19.png)
![image](./20.png)
- install Docker https://docs.docker.com/engine/install/ (under "Server").(v.131)
- 本機 push image 到 Docker hub, 連到 remote server pull and run image
![image](./21.png)
![image](./22.png)
- disadvantage: 上述做法需要建立 EC2 Instance 還需要安裝 Docker 自己 build run image
- alternative: using ECS

2. Deploy with AWS ECS: A Managed Docker Container Service
**deployment-03-final-single-container-example,  ECS cost money 所以看影片就好 v.140, 141, 142**
   - 搞懂 ECS 的 container, task, service, cluster
   - basic AWS tutorial: https://academind.com/tutorials/aws-the-basics
   - ECS 幫我們 manage image, container, EC2, netwerk...設定好 container, task, service, cluster(network) 後就可以跑了.
   -  update service -> rerun task -> pull update image in container 
**Preparing a multi-container App(v.143 deployment-05-ecs-two-containers-node-mongo, v.144, 145**
   - Deploy mutiple container App to AWS ECS, docker-compose.yml 通常都是 deploy multiple container in **single machine**, 但在 AWS 上, 這些 container 可能佈在不同的 machine, 所以 docker-compose 不能直接拿來用但可以當作參考
   - 在 AWS ECS 如果 container 都在相同 task 下, 他們有相同的 host(machine), 可以用 localhost 互相連接
   - 如果不在相同 network 下就無法用 containerName/ localhost 相連
   - EFS 保存 volume 資料
   - ALB 取得 static DNS
   - ![image](./23.png)
1. Database & containers: An Important Consideration(v.149 deployment-06-switched-to-mongodb-atlas)
上面的架構 database container 是用我們自己 database 的 container 的但有以下這些缺點: 可以透過使用 RDS  AWS service, MongoDB cloud Atlas...等雲端服務來幫我們管理 DB
![image](./24.png)
- move to MongoDB atlas, 所以我們也不需要 EFS 和 mongoDB container on AWS
- ![image](./25.png)

1. Our Updated & Target Architecture(加上 frontEnd v.154 deployment-09-finished)
![image](./26.png)
![image](./27.png)
- 前端的 project 很多在 deployment 到 prod 之前都需要經過 build step, 和平常在 dev 開發時的指令不同
- dev: 我們透過 npm start:  node server 來跑 react code, 但到了 prod: 我們則是透過 node 來將前端框架的程式碼 (ex: jsx) build compile 成 browser 能理解的 js,  node 後面就不需要, 可用 nginx server 替代
- 為了達成上面的目的, 我們可以使用 Multi-stage build in dockerfile, 參考 [link](../section9/deployment-09-finished/frontend/Dockerfile.prod)
```
  docker build -t tagName -f frontend/Dockerfile.prod ./frontend (-f 指定 dockerfile build)
  docker build --target build ./frontend(--target build 指定只跑 build 這個 stage)
```

# Docker section 10: docker 大回顧
 [link](../section10/slides-mid-summary.pdf)


# kubernetes section 11: Getting start with kubernetes 
kubernetes 幫我們處理 deployment, autoscaling, load balance 等在不同的機器上, 有點像 docker-compose 但卻部屬在不同機器上, 而這些機器我們必須要事先準備好並安裝 kubernetes 相關的套件, 才可以 work, 類似我們需要先安裝 docker 才能跑 Docker compose...
![image](./29.png)
![image](./30.png)
![image](./31.png)
![image](./32.png)
![image](./34.png)
- 介紹 kubernetes Cluster, Worker Node, Pod, Master Node 等術語
![image](./33.png)
![image](./35.png)
![image](./36.png)
![image](./37.png)

# kubernetes section 12: Kubernetes in action: dive into the core concept
![image](./34.png)
- kubernetes 只在乎如何 deploy application, scaling, LB 等等, 但所需的機器和套件都必須我們自己準備如上圖的右側
- 如果不想自己處理機器等安裝, 可以使用 kubermatic, AWS EKS(elastic Kubernetes Service) 等服務, 我們只要給他們 kubernetes configuration, 他們就會幫我們建好一切

## Kubernetes: Required SetUp & Installation Steps
![image](./38.png)
  - 安裝 kubectl and minikube(local clustor) 在本機:
  - kubectl 負責下指令(kubernetes Object)給 clustor 告訴 clustor 要幹嘛
  - 先在本機測試 kubernetes configuration 是否 work, 在 deploy 到 remote machine
  - https://minikube.sigs.k8s.io/docs/start/
  - https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/
```
brew install kubectl
brew install minikube
minikube start (--driver=docker)(lauch local single Node clustor)
docker ps(可以看到 minikube 確實有起起來)
minikube status (check the status of cluster)
minikube dashboard(web dashboard)

minikube stop (stop clustor)
```

## Understanding kubernetes Objects (Resources, v.186)
kubernetes Objects 如同指令, 餵給 kubernetes clustor 他就知道要做什麼
![image](./39.png)
1. Pod Object & Deployment Object
   - 通常我們不會自己建立 Pod Object 因為這是 kubernetes 要幫我們用的, 我們會建立 Deployment Object 來指示 kubernetes 要怎麼處理 Pods
    ![image](./40.png)
    ![image](./41.png)
    ```
    docker build -t kub-first-app . (建立讓 cluster 可以 run container 的 image)
    kubectl --help
    kubectl create deployment first-app --image=kub-first-app (建立 deployment Obj to clustor, 並使用剛剛build的 kub-first-app image)
    kubectl get deployments (取得有哪些 deployment)
    kubectl get pods (看 pods 狀況)
    但我們會發現 pod Status 為 ErrImagePull, 原因時因為 clustor 是在另一個 VM 所以沒有 kub-first-app 這個 local image, 所以我們需要把他 push 到 docker hub 上面, 這樣 clustor 才找得到
    
    kubectl delete deployment first-app
    docker tag kub-first-app joesu0610/kubernetes
    docker push joesu0610/kubernetes ( push 到自己的 docker hub)
    kubectl create deployment first-app --image=joesu0610/kubernetes
    kubectl get deployments
    kubectl get pods (看到 Status running 代表 pod 有成功起起來)
    這時候看 minikube dashboard 就可以看到 1 pod 和相關資訊
    ```
  - What "kubectl create deployment first-app --image=joesu0610/kubernetes" Do?
   ![image](./42.png)

2. Service Object (v.190)
  ![image](./43.png)
  - pod 雖然有 InternalIp 但我們無法透過 InternalIp 直接和他們聯繫
  - 透過 Service Obj 我們可以從 clustor 外和 pod 連線
  ```
  kubectl expose deployment first-app --type=LoadBalancer(還有別種type可選擇, 查看 document) --port=8080(expose first-app deployment on 8080 port, 我們可以直接透過 localhost:8080 和 pod 連線)
  kubectl get services (如果我們在新的機器上就應該會有 external_ip, 但因為 minikube 是 local VM, 所以才會在 pending 狀態)
  -----------------------------------------------------------------------------
  NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
  first-app    LoadBalancer   10.99.146.56   <pending>     8080:30347/TCP   5s
  -----------------------------------------------------------------------------

  minikube service first-app(讓我們可以和 VM 中的 service 連線, 一般來說不需要這一行)

  如果我們打 /error 來讓 container crash 
  kubectl get pods(我們會發現過一陣子 pod 裡的 container 又被 restart 起來 restart +1)
  
  kubectl scale deployment/first-app --replicas=3 (replicas 代表我們要幾個 pod)
  觀看 minikube dashboard 發現 pod 變成 3 個, 且這三個 pod 都在同一個 service(LB 底下)
  kubectl scale deployment/first-app --replicas=1
  ```
3. Updating Deployment(v.194)
  ```
  更改 source code 後
  docker build -t joesu0610/kubernetes:1.0.0 .
  docker push joesu0610/kubernetes:1.0.0
  kubectl set image deployment/first-app kubernetes=joesu0610/kubernetes:1.0.0(old image=new image, 跑完這一行我們就完成 update deployment 動作了)
  ```
4. Deployment RollBack & history(v.195)
  ```
  kubectl set image deployment/first-app kubernetes=joesu0610/kubernetes:3.0.0(故意用一個不存在的 image, 導致 update fail)
  kubectl rollout status deployment/first-app(觀察 pod rolling out 的狀況, 由於新的 pod 找不到 image一直起不起來, 所以舊的不會被關掉)
  kubectl get pods(舊的存在，但新的起不起來)
  kubectl rollout undo deployment/first-app(rollback rollout to previous deployment)
  kubectl rollout history deployment/first-app (查看 deployment history)
  kubectl rollout history deployment/first-app --revision=1 (查看 revision 1 deployment history)
  kubectl rollout undo deployment/first-app --to-revision=1 (rollback to revision=1 deployment, 會發現 app 回到最一開始的版本)

  kubectl delete service first-app (清除 service first-app)
  kubectl delete deployment first-app (清除 deployment first-app)
  ```

5. The imperative vs **declarative approach**(參考kub-action-02-declarative-approach-basics, v.196~)
![image](./44.png)
- 前面的範例都是 imperative 下指令的方法, 但每次都要手動下指令很麻煩, 所以透過 declarative approach 來下指令 trigger Configuration file 更方便 (類似 docker-compose.yml file 的功用)
- 透過 Selector configuration 讓我們可以將不同的 kubernetes obj 互相關聯 

**Creating a Deployment Obj**
- https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.29/#deployment-v1-apps
- 參考 deployment.yaml(名稱自己取) [link](../section12/kub-action-02-declarative-approach-basics/deployment.yaml)
```
定義 deployment.yaml
kubectl apply -f=deployment.yaml (執行 deployment.yaml file)
kubectl get deployments
kubectl get pods(1 個 pod)
```

**Creating a Service Obj(v.200)**
- https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.29/#service-v1-core
- 參考 service.yaml(名稱自己取) [link](../section12/kub-action-02-declarative-approach-basics/service.yaml)
```
kubectl apply -f=service.yaml
kubectl get services
minikube service backend(service name)
```

**Updating and Deleting Resources(v.201)**
```
更改 yaml file
kubectl apply -f=deployment.yaml （ then update works!!)

kubectl delete -f=deployment.yaml,service.yaml (delete deployment.yaml 和 service.yaml 產生的 resource)
當然也可以用一般的 imperative del way
kubectl delete service backend
kubectl delete deployment second-app-deployment
```

**Mutiple vs Single Config Files(參考kub-action-03-merging-config-files, v.202)**
- 參考 master-deployment.yaml(名稱自己取) [link](../section12/kub-action-03-declarative-approach-basics/master-deployment.yaml)
- 透過 --- 將不同 yaml 檔案合併
- 這些 kubernetes obj 並非在 create 當下作用一次，而是會持續 running 狀態, 當 pod 發生改變時, 根據 selector 決定是否加入 service 等等
- Service 通常會放在 deployment 前面, 這樣執行到 deployment 時 service 就可以把 deployment 產生得 pod 加進去，但誰先誰後其實都可以 work
```
kubectl apply -f=master-deployment.yaml 
kubectl get services
minikube service backend(service name)
```

**More on Labels and Selectors(參考 kub-action-04-more-on-labels-and-selectors v.203)**
- 透過 Selector configuration 讓我們可以將不同的 kubernetes obj 互相關聯
- - 有 matchLabels, matchExpressions
- 參考 deployment.yaml(名稱自己取) [link](../section12/kub-action-04-more-on-labels-and-selectors/deployment.yaml)
```
kubectl apply -f=deployment.yaml,service.yaml
kubectl delete deployments,services -l group=example (delete by label)
kubectl delete deployment second-app-deployment(del by name)
```

**Liveness probs(Health Check 參考kub-action-05-finished v.204)**
- 參考 deployment.yaml(名稱自己取) [link](../section12/kub-action-05-finished/deployment.yaml)


# kubernetes section 13: Managing Data & Volume with kubernetes
1. kub-data-01-starting-setup(v.209)
```
docker-comopse up -d --build(確保每次都 build image)
docker-compose down
docker-comopse up -d --build (發現即使 container rm 後 recreate 仍然保有原本資料, 因為有 volume)
我們也希望在 kubernetes 達到相同得目的
```
- kubernetes 同樣透過 volume 來保存 data
- container restart/recreate 時, volume 不會消失因此資料可以獲得保存
- 和 docker 不一樣的地方在於, 
  - kubernetes volume 是在 pod 當中, 所以當 pod 消失時, volume 也會跟這消失
  - kubernetes volume 有很多種種類，可以存在 local machine, 或雲端服務當中 https://kubernetes.io/docs/concepts/storage/volumes/
- 透過 configuration, 可以告訴 kubernetes 該如何處理 volume
![image](./45.png)
![image](./46.png)
![image](./47.png)

2. Creating New Deployment and Service(kub-data-02-deployment-and-service, v.212)
```
docker build -t joesu0610/kubernetes:2.0.0 .
docker push joesu0610/kubernetes:2.0.0
kubectl apply -f=deployment.yaml,service.yaml
kubectl get services
minikube service story-service
```
現在我們還沒有加入 volume, 所以當 container recreate 後資料就會不見

3. A First Volume: The "emptyDir" Type(參考 kub-data-03-first-volume v.214)
- 參考 deployment.yaml(名稱自己取) [link](../section13/kub-data-03-first-volume/deployment.yaml)
- emptyDir: 在 pod 裡面的資料夾, mount 到 container, 因為是在 pod 裡面, 所以要定義在 pod object 中
- 所以當 container restart, 資料仍可以保留
- 但當 pod 被消滅, volume 也會跟著消失

4. A Second Volume: The "hostPath" Type(參考kub-data-04-hostpath, v.215)
- 參考 deployment.yaml(名稱自己取) [link](../section13/kub-data-04-hostpath/deployment.yaml)  
- 假設今天打到別的 pod, 或 pod 被消滅, 那就打不到原本的 emptyDir volume, 這時候可以使用 hostPath
- hostPath volume 位於 Node(machine 當中), Node 內的 pod 共享, 所以即使 pod 消失也不會不見
- hostPath 如同 Mount, 當我們在 Node folder 新增檔案時，也會同時映射到 container 當中
- 但如果當 Node 被消滅, 或有多個 Node, hostPath 則不管用, 在 prod 一般都是 multi-Node

5. Understanding the "CSI" volume type(參考, v.216)
- CSI: Container Storage Interface
- 實在太多 volume 供 kubernetes 去對接, kubernetes team 直接設計 CSI interface 讓各種 volume type 去實作 ex: AWS EFS CSI

6. From volumes to persistent volume(參考kub-data-05-pv-and-pvc, v.217)
![image](./48.png)
![image](./49.png)
- https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- 前面提到的 volume 定義在 Node or pod, 我們可以定義獨立於 node / pod 的 volume, 讓各種 node / pod 都可以指向他
- persistent volume independent from pod, Node, 獨立於 pod or Node, 只需建立 pv 一次, pod/node 可透過 pv claim 指向 pv 來使用 pv(我們也可以使用現有的雲端儲存服務 ex AWS EBS etc...)
- 因為 minikube 只是 single Node, 所以範例的 persistent volume type 為 hostPath, 當然也可以使用其他的 pv type
- 參考 deployment.yaml[link](../section13/kub-data-05-pv-and-pvc/deployment.yaml)  
- 參考 host-pv.yaml[link](../section13/kub-data-05-pv-and-pvc/host-pv.yaml)  
- 參考 host-pvc.yaml[link](../section13/kub-data-05-pv-and-pvc/host-pvc.yaml)  

```
也可以把上面三個檔案用 --- 串再一起
kubectl apply -f=host-pv.yaml (建立 persistent volume(pv))
kubectl apply -f=host-pvc.yaml (建立 pv claim)
kubectl apply -f=deployment.yaml （ pod 指定 pv claim -> pv)
kubectl get pv (查看 persistent volume)
kubectl get pvc (查看 persistent volume claim)
```

7. Using Environment variable(參考kub-data-06-env, v.222)
  
**app.js**
```
const filePath = path.join(__dirname, process.env.STORY_FOLDER, 'text.txt');
```
**deployment.yaml**
```
   spec:
      containers:
        - name: story
          image: joesu0610/kubernetes:3.0.0
          env:
            - name: STORY_FOLDER
              # value: 'story' # 直接寫 value
              valueFrom: # 從 configMap obj 中讀取
                configMapKeyRef:
                  name: data-store-env
                  key: folder
```
**environment.yaml**
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: data-store-env
data:
  folder: 'story'
```







  

  






