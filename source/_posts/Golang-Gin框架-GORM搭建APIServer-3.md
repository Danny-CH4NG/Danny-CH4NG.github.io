---
title: Golang-Gin框架-GORM搭建APIServer-3
date: 2022-01-05 22:26:50
tags:
  - Gin
  - GORM
  - PostgreSQL
categories:
  - Golang
---

# [Golang] Gin框架+GORM 搭建API Server(3)

## 前言
上一節完成了基本的路由註冊功能，本節將完成對資料庫的連線部分。
根據第一節，建立了
* device_infos
* device_states
* device_actives

而三支API分別為
* /api/v1/actives/:active：
    * SELECT device_actives WHERE active=? JOIN device_infos 
    * SELECT actives WHERE active=?
* /api/v1/actives/:device_id：
    * UPDATE device_actives SET active=1 WHERE device_id=?
* /api/v1/states/:state：
    * SELECT device_states WHERE state=? JOIN device_infos

將完成資料庫select、update、join等操作

## 本節目標
* 建立actives的view結合device_infos與device_actives，避免每次查詢重覆join
* 建立GORM Model架構
* 完成API與GORM對接

### 基本目錄結構
```
server/
├── config // 配置讀取yaml設定
│   └── config.go // 讀取yml設定
├── middleware // 中間件
├── models // 放置gorm的數據庫模型
│   ├── actives.go // 對應actives檢視表
│   ├── device_actives.go // 對應device_actives資料表
│   ├── device_infos.go // 對應device_infos資料表
│   ├── device_states.go // 對應device_states資料表
│   └── models.go // model init
├── routers // 路由邏輯
│   ├── api
│   │   └── v1
│   │       ├── actives.go // actives相關api
│   │       └── states.go // states相關api
│   └── router.go // 路由邏輯
├── config.yml // 設定檔
└── server.go // 入口
```

## PostgreSQL建立檢視表
避免查詢多次JOIN浪費效能，並可以讓整體架構更清晰
### actives
```
CREATE OR REPLACE VIEW public.actives
 AS
 SELECT device_actives.device_id,
    device_infos.type,
    device_infos.address,
    device_actives.active
   FROM device_infos,
    device_actives
  WHERE device_infos.device_id::text = device_actives.device_id::text;
```

### 建立資料
在device_actives表與devices_infos表建立對應資料，查看actives表是否如預期顯示
#### device_actives
```
INSERT INTO public.device_actives(
	device_id, active)
	VALUES ('test001', 0);
```
#### devices_infos
```
INSERT INTO public.device_infos(
	device_id, type, address)
	VALUES ('test001', 'test', '測試地址');
```
#### devices_states
```
INSERT INTO public.device_states(
	device_id, state, update_time)
	VALUES ('test001', 1, '2021-10-19 05:49:25');
```
## 建立GORM連線
### models.go
建立db的連線，由config獲取連線資訊
```
package models

import (
	"fmt"
	"server/config"

	"github.com/rs/zerolog/log"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
)

// db實例
var db *gorm.DB

func init() {
	dsn := fmt.Sprintf(
		"host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
		config.Config.Pgsql.IP,
		config.Config.Pgsql.Port,
		config.Config.Pgsql.User,
		config.Config.Pgsql.Password,
		config.Config.Pgsql.Database,
	)

	var err error

	// 嘗試連線PostgreSQL
	// 注意：這裡不要用:=，會改變db記憶體位置，db成為nil且不會報錯直到調用
	db, err = gorm.Open(postgres.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Error().Err(err).Msg("無法建立PostgreSQL連線")
	} else {
		log.Info().Msg("成功建立PostgreSQL連線")
	}

	sqlDB, err := db.DB()
	sqlDB.SetMaxIdleConns(10)  // 設定最高空閒連接數
	sqlDB.SetMaxOpenConns(100) // 設定最高併發數
}

// 關閉DB(不可寫在init，init階段結束會直接觸發)
func CloseDB() {
	sqlDB, _ := db.DB()
	defer sqlDB.Close()
}
```

## 設備狀態請求與獲取資料
### models/actives.go
將表的欄位以struct進行對應，並填寫輸出成json的樣式
```
package models

// actives表的模型
type Actives struct {
	DeviceId string `json:"deviceId"`
	Type     string `json:"type"`
	Address  string `json:"address"`
	Active   int    `json:"active"`
}

// 取得符合Actives條件的row
func GetActives(maps interface{}) (res []Actives) {
	a, ok := maps.(map[string]interface{}) // 型別斷言為map
	if ok && a["active"] == 99 {           // 如果請求值為99，則為全選
		db.Find(&res)
	} else {
		db.Where(maps).Find(&res)
	}
	return
}
```
### routes/v1/actives.go
```
package v1

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/rs/zerolog/log"

	"server/models"
)

// 獲取已排除、未排除or全部的設備狀態
func GetActives(c *gin.Context) {
	active := c.Param("active") // 取得active值

	req := make(map[string]interface{}) // 傳遞給GORM的參數
	res := make(map[string]interface{}) // 準備回傳給前端的映射

	if active != "" {
		i, err := strconv.Atoi(active) // 字串轉數字
		if err != nil {
			log.Error().Err(err).Msg("字串轉數字錯誤")
		}
		req["active"] = i
	}

	res["data"] = models.GetActives(req)
	res["code"] = "success"
	c.JSON(http.StatusOK, res)
}

// 修改設備活躍狀態
func EditActive(c *gin.Context) {
	log.Log().Msg("EditActive")
}
```

### 測試
Postman GET
http://localhost:10080/api/v1/actives/0
檢查是否回傳預想中的值，完成第一支API連線

## 修改設備活躍狀態
### models/device_actives.go
```
package models

type DeviceActives struct {
	DeviceId string `json:"deviceId" gorm:"column:device_id"`
	Active   int    `json:"active"`
}

// 依據device_id toogle active欄位
func UpdateDeviceActives(maps interface{}) (deviceActives DeviceActives) {
	db.Where(maps).First(&deviceActives)
	if deviceActives.Active == 0 {
		db.Model(&DeviceActives{}).Where(maps).Update("active", 1)
	} else if deviceActives.Active == 1 {
		db.Model(&DeviceActives{}).Where(maps).Update("active", 0)
	}
	return
}
```
### routes/v1/actives.go
```
package v1

import (
	"net/http"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/rs/zerolog/log"

	"server/models"
)

// 獲取已排除、未排除or全部的設備狀態
func GetActives(c *gin.Context) {
	active := c.Param("active") // 取得active值

	req := make(map[string]interface{}) // 傳遞給GORM的參數
	res := make(map[string]interface{}) // 準備回傳給前端的映射

	if active != "" {
		i, err := strconv.Atoi(active) // 字串轉數字
		if err != nil {
			log.Error().Err(err).Msg("字串轉數字錯誤")
		}
		req["active"] = i
	}

	res["data"] = models.GetActives(req)
	res["code"] = "success"
	c.JSON(http.StatusOK, res)
}

// 修改設備活躍狀態
func EditActive(c *gin.Context) {
	deviceId := c.Param("device_id") // 取得device_id值

	req := make(map[string]interface{}) // 傳遞給GORM的參數
	res := make(map[string]interface{}) // 準備回傳給前端的映射

	if deviceId != "" {
		req["device_id"] = deviceId
	}

	models.UpdateDeviceActives(req)                                      // toogle所選擇設備
	res["data"] = models.GetActives(map[string]interface{}{"active": 1}) // 取得新的已排除設備列表
	res["code"] = "success"
	c.JSON(http.StatusOK, res)
}
```

### 測試
Postman PUT
http://localhost:10080/api/v1/actives/test001
檢查是否回傳預想中的值

## 獲取「異常但未被排除」設備狀態
### models/device_states.go
```
package models

import "time"

type DeviceStates struct {
	ID         int       `json:"id"`
	DeviceId   string    `json:"device_id"`
	State      int       `json:"state"`
	UpdateTime time.Time `json:"update_time"`
}

type GetDeviceStatesResult struct {
	Actives
	State      int       `json:"state"`
	UpdateTime time.Time `json:"update_time"`
}

// 取得符合State條件的row，並JOIN device_info
func GetDeviceStates(maps interface{}) (res []GetDeviceStatesResult) {
	a, ok := maps.(map[string]interface{}) // 型別斷言為map
	if ok && a["state"] == 99 {            // 如果請求值為99，則為全選
		db.
			Model(&DeviceStates{}).
			Select("*").
			Joins("left join actives on actives.device_id = device_states.device_id").
			Scan(&res)
	} else {
		db.
			Model(&DeviceStates{}).
			Select("*").
			Joins("left join actives on actives.device_id = device_states.device_id").
			Where(maps).
			Scan(&res)
	}
	return
}
```
### routes/v1/states.go
```
package v1

import (
	"net/http"
	"server/models"
	"strconv"

	"github.com/gin-gonic/gin"
	"github.com/rs/zerolog/log"
)

// 獲取「異常但未被排除」設備狀態
func GetStates(c *gin.Context) {
	state := c.Param("state") // 取得active值

	req := make(map[string]interface{}) // 傳遞給GORM的參數
	res := make(map[string]interface{}) // 準備回傳給前端的映射

	if state != "" {
		i, err := strconv.Atoi(state) // 字串轉數字
		if err != nil {
			log.Error().Err(err).Msg("字串轉數字錯誤")
		}
		req["state"] = i
	}
	req["active"] = 0

	res["data"] = models.GetDeviceStates(req)
	res["code"] = "success"
	c.JSON(http.StatusOK, res)
}
```

### 測試
Postman GET
http://localhost:10080/api/v1/states/1
檢查是否回傳預想中的值

