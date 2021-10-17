---
title: Golang-Gin框架+GORM搭建APIServer(1)
date: 2021-10-17 15:01:36
tags:
  - Gin
  - GORM
  - PostgreSQL
categories:
  - Golang
---

## 前言
* 後端：Golang Gin框架
* 資料庫：PostgreSQL

這裡我們來做一個簡單的設備監控平台，初期階段有以下功能：
* 儲存設備的基本資料(id, 類型, 地址)
* 從資料庫中取得狀態異常的設備(id, 異常狀態代碼)
* 讓使用者可以排除特定設備，因此需要一個設備活躍狀態的資料表(id, 活躍狀態代碼)

以上需求構成了我們的三張資料表：
1. device_info(device_id, type, address)<靜態>
2. device_state(id, device_id, state, update_time)<動態/未來會有其他功能進行寫入>
3. device_active(device_id, active)<靜態>

## 本節目標
* 建立基本目錄結構與資料庫
* 讀取yml設定檔進行配置

## 項目初始化
### 基本目錄結構
```
server/
├── config // 配置讀取yaml設定
├── middleware // 中間件
├── models // 放置gorm的數據庫模型
├── routers // 路由邏輯
├── config.yml // 設定檔
└── server.go // 入口
```
### 建立一個基本資料庫(不會用docker可跳過)
建立data資料庫，建議使用docker-compose快速建立
#### docker-compose.yaml
```
version: "3"
services:
  pgsql:
    image: postgres
    container_name: pgsql
    volumes:
      - ./Postgresql:/data/postgres
      # - ./sqls:/docker-entrypoint-initdb.d
    environment:
      POSTGRES_ROOT_PASSWORD: password
      POSTGRES_DB: data
      POSTGRES_USER: ${POSTGRES_USER:-user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
    ports:
      - "5432:5432"
    restart: always
```

#### 1. 建立設備基本資料靜態表
```
CREATE TABLE IF NOT EXISTS public.device_info
(
    device_id character varying(20) COLLATE pg_catalog."default" NOT NULL,
    type character varying(10) COLLATE pg_catalog."default" NOT NULL,
    address character varying(100) COLLATE pg_catalog."default",
    CONSTRAINT device_info_pkey PRIMARY KEY (device_id)
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.device_info
    OWNER to user;

COMMENT ON COLUMN public.device_info.device_id
    IS '設備id';

COMMENT ON COLUMN public.device_info.type
    IS '設備類型(''CMS'', ''TC'', ''CCTV'', ''eTag'')';

COMMENT ON COLUMN public.device_info.address
    IS '設備中文地址';
```
#### 2. 建立設備活躍狀態靜態表
```
-- Table: public.device_active

-- DROP TABLE IF EXISTS public.device_active;

CREATE TABLE IF NOT EXISTS public.device_active
(
    device_id character varying(20) COLLATE pg_catalog."default" NOT NULL,
    active integer DEFAULT 0,
    CONSTRAINT device_active_pkey PRIMARY KEY (device_id)
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.device_active
    OWNER to user;

COMMENT ON COLUMN public.device_active.device_id
    IS '設備id';

COMMENT ON COLUMN public.device_active.active
    IS '設備是否被排除(0: 否, 1:是)';
```
#### 3. 建立設備狀態表
```
CREATE TABLE IF NOT EXISTS public.device_state
(
    id integer NOT NULL DEFAULT nextval('device_state_id_seq'::regclass),
    device_id character varying(20) COLLATE pg_catalog."default" NOT NULL,
    state integer DEFAULT 0,
    update_time time with time zone,
    CONSTRAINT device_state_pkey PRIMARY KEY (id)
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.device_state
    OWNER to user;

COMMENT ON COLUMN public.device_state.id
    IS '流水號';

COMMENT ON COLUMN public.device_state.device_id
    IS '設備id';

COMMENT ON COLUMN public.device_state.state
    IS '設備狀態(0:正常, 1:自動通報異常, 2:人工通報異常)';

COMMENT ON COLUMN public.device_state.update_time
    IS '通報時間';
```
### 編寫簡易的設定檔
config.yml
```
mode: "debug"

server:
  port: "10080"

db:
  type: "postgresql"
  ip: "localhost"
  port: "5432"
  userID: "thi"
  password: "thi"
  database: "device"
```
注意，yml的編寫縮排有強制規定

## 編寫啟動入口文件
### server.go
```
package main

import (
	"github.com/rs/zerolog/log"
	"github.com/gin-gonic/gin"
)


func main() {
	log.Logger = log.With().Caller().Logger()
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
        log.Info().Msg("/ping success")
	})
	r.Run(":10080")
}

```
導入gin package，建立一個最基本的server
```
go mod init server
go mod tidy
go run server.go
```
訪問 localhost:10080/ping，測試是否返回"pong"

## 編寫讀取設定檔的方式
```
server/
├── config // 配置讀取yaml設定
│   └── config.go // 讀取yml設定
├── middleware // 中間件
├── models // 放置gorm的數據庫模型
├── routers // 路由邏輯
├── config.yml // 設定檔
└── server.go // 入口
```
### config.go
```
package config

import (
	"io/ioutil"

	"github.com/rs/zerolog/log"
	"gopkg.in/yaml.v2"
)

var Config Configer // 設定參數

// Config
type Configer struct {
	Mode string `yaml:"mode"`
	Api  struct {
		Url       string `yaml:"url"`
		Frequency int    `yaml:"frequency"`
		Refresh   int    `yaml:"refresh"`
	} `yaml:"api"`
	Server struct {
		Port string `yaml:"port"`
	} `yaml:"server"`
	Pgsql struct {
		IP       string `yaml:"ip"`
		UserID   string `yaml:"userID"`
		Password string `yaml:"password"`
		Database string `yaml:"database"`
	} `yaml:"pgsql"`
}

func init() {
	// 讀取yml文件
	yamlFile, err := ioutil.ReadFile("config.yml")
	if err != nil {
		log.Error().Err(err).Msg("無法讀取yaml file")
		return
	}
	// 將yml檔解析成struct
	err = yaml.Unmarshal(yamlFile, &Config)
	if err != nil {
		log.Error().Err(err).Msg("yaml無法解析")
		return
	}

	if Config.Mode == "debug" {
		log.Info().Msgf("yaml解析結果: %v", Config)
	}
}
```

將server.go 的port改成吃設定檔而非寫死，未來移植時不用從code一個一個找出來變更
### server.go
```
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/rs/zerolog/log"

	"server/config"
)

func main() {
	// 調用zerolog結構化log輸出
	log.Logger = log.With().Caller().Logger()
    
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
        log.Info().Msg("/ping success")
	})
	r.Run(":" + config.Config.Server.Port)
}
```

---
###### tags: `GO`, `gin`
---