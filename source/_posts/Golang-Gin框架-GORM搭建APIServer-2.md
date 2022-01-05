---
title: Golang-Gin框架+GORM搭建APIServer(2)
date: 2021-10-21 12:27:30
tags:
  - Gin
  - GORM
  - PostgreSQL
categories:
  - Golang
---

## 前言
功能需求如下：
1. 取得所有的已被排除在外的設備與基本訊息
2. 根據device_id變換設備是否排除
3. 取得所有的「異常但未被排除」設備

根據Restful的設計風格，我們應有以下三支API：
1. GET("/api/v1/actives/:active") 獲取已排除、未排除or全部的設備狀態
2. PUT("/api/v1/actives/:device_id") 將設備狀態變換
3. GET("/api/v1/states/:state") 獲取「異常但未被排除」設備狀態

整體流程：
1. 由配合device_actives表的API與device_states表的API開始
2. router.go 將API Group，並將gin.Engine的建立移到此處
3. 改寫server.go

## 本節目標
* 建立API架構與路由註冊

### 基本目錄結構
```
server/
├── config // 配置讀取yaml設定
│   └── config.go // 讀取yml設定
├── middleware // 中間件
├── models // 放置gorm的數據庫模型
├── routers // 路由邏輯
│   ├── api
│   │   └── v1
│   │       ├── actives.go // device_actives表的api
│   │       └── states.go // device_states表的api
│   └── router.go // 路由邏輯
├── config.yml // 設定檔
└── server.go // 入口
```

## 編寫API

### actives.go
```
package v1

import (
	"github.com/gin-gonic/gin"
	"github.com/rs/zerolog/log"
)

// 獲取已排除、未排除or全部的設備狀態
func GetActives(c *gin.Context) {
	log.Log().Msg("GetActives")
}

// 修改設備活躍狀態
func EditActive(c *gin.Context) {
	log.Log().Msg("EditActive")
}
```
建立兩支針對device_actives表進行服務的API，我們下一節再進入GORM的編寫，因此先以log方式檢驗呼叫API是否符合預期

### states.go
```
package v1

import (
	"github.com/gin-gonic/gin"
	"github.com/rs/zerolog/log"
)

// 獲取「異常但未被排除」設備狀態
func GetStates(c *gin.Context) {
	log.Log().Msg("GetStates")
}
```

## 路由註冊
### router.go
```
package routers

import (
	v1 "server/routers/v1"

	"github.com/gin-gonic/gin"
)

func InitRouter() *gin.Engine {
	// 生成gin
	r := gin.Default()

	api := r.Group("/api/v1")
	{
		// 獲取已排除、未排除or全部的設備狀態
		api.GET("/actives/:active", v1.GetActives)
		// 將設備狀態變換
		api.PUT("/actives/:device_id", v1.EditActive)
		// 獲取「異常但未被排除」設備狀態
		api.GET("/states/:state", v1.GetStates)
	}

	return r
}
```

## 改寫啟動文件
### server.go
```
package main

import (
	"github.com/rs/zerolog/log"

	"server/config"
	"server/routers"
)

func init() {
	// 調用zerolog結構化log輸出
	log.Logger = log.With().Caller().Logger()
}

func main() {
	router := routers.InitRouter()
	router.Run(":" + config.Config.Server.Port)
}
```

## 測試
使用PostMan或其他API測試工具
GET訪問：
http://localhost:3601/api/v1/actives/1
http://localhost:3601/api/v1/states/1
PUT訪問：
http://localhost:3601/api/v1/actives/1
查看是否反饋正確的log
