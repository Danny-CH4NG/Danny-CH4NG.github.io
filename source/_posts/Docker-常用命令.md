---
title: Docker-常用命令
date: 2021-08-22 15:33:44
tags:
  - Docker
categories:
  - Docker
---


https://docs.docker.com/reference/
### 幫助命令
版本訊息
```
docker version
```
系統訊息，包含鏡像與容器數量
```
docker info
```
幫助命令
```
docker <> --help
```

### 鏡像命令

#### docker images
```
docker images

REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
鏡像倉庫源      標籤         id           創建時間        大小

可選項

-a, -all 所有鏡像
-q, --quiet  只顯示id
```

#### docker search
```
docker search <名稱>

--filter=STARS=3000 搜索不小於3000
```

#### docker pull
```
docker pull <名稱>:<版本 >
```

#### docker rmi
```
docker rmi -f <名稱或id>

全刪
docker rmi -f $(docker images -aq)

```


### 容器命令
先有鏡像，才能創造容器

#### docker run 
```
docker run <參數> <image>

--name="名字" 容器名字
-d 後臺運行
-it 交互運行，可進入容器查看
-p 指定port 主機:容器

docker run -it centos /bin/bash 啟動centos並進入
exit 停止退出
Ctrl + p + q 不停止退出
```

#### docker ps
```
docker ps 正在執行的
-a  全部運行的
-n=1 最近1個
-q 只顯示編號
```

#### docker rm
```
docker rm <id>

全刪
docker rm -f $(docker ps -aq)
docker ps -a -q | xargs docker rm
```

#### docker start restart stop kill
```
docker start <id>
docker restart <id>
docker stop <id>
docker kill <id>  強制停止
```

#### 後臺啟動
```
docker run -d <名字> 
docker ps  容器停止了
常見錯誤： docker容器後台運行，需要前台進程，如果沒有就會自動停止
```

#### docker logs
```
docker logs -tf  <id>  全部
docker logs -tf --tail <數量> <id>  最近
```
插播：如何在docker centos寫腳本
```
docker run -d centos /bin/sh -c "while true;do echo SoGod; sleep 1;done"
```

#### docker top
```
docker top <id>  查看進程訊息
```

#### docker inspect
```
docker inspect <id>  查看容器詳細訊息
```

#### docker exec
```
docker exec -it <id> /bin/bash  進入正在運行的容器並開始新的終端
docker attach <id> 進入正在運行的容器，使用正在執行的終端
```

#### docker cp
```
docker cp <id>:<容器路徑> <主機路徑>  查看容器詳細訊息

此處為手動。一般使用自動同步
```

#### docker commit
```
docker commit <id> <目標鏡像名>:<tag> 將容器提交為新的鏡像
-m="訊息"
-a="作者"
```
