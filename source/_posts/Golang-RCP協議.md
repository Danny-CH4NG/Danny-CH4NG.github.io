---
title: Golang-RCP協議
date: 2021-09-09 21:26:59
tags:
  - RPC
  - 網路架構
categories:
  - Golang
---
RPC: 遠程行程(進程，單一程式)調用協議，使用TCP，屬於應用層協議，與http協議同層
**==可理解為調用內部函數一樣，調用網路中其他程式的函數==**
通過RPC協議，傳遞：函數名、參數，從原處調用另一處函數，返回結果到原處
* 每個微服務彼此獨立
* 程式與程式之間可使用不同程式語言

### 前導知識 Go socket
#### server
```
net.Listen() -- listener
listener.Accept() -- conn
conn.read()
conn.write()
defer conn.close()/listener.Close()
```

#### client
```
net.Dial() -- conn
conn.Write()
conn.Read()
defer conn.Close()
```

## RPC使用步驟
#### server
1. 註冊RPC服務對象，給對象綁定方法(1.定義類 2.綁定類方法)
```
rpc.RegisterName(<服務名>, <回調對象>)
```
2. 創建監聽器
```
listener, err := net.Listen()
```
3. 建立連結
```
conn, err := listener.Accept()
```
4. 將連結綁定RPC服務
```
rpc.ServeConn(conn)
```

#### client
1. 用RPC連結server
```
conn, err := rpc.Dial()
```
2. 調用遠程函數
```
conn.Call(<服務名.方法名>, <傳入參數>, <傳出參數>)
```

## RPC相關函數
1. ==RegisterName()== 註冊服務
```
func (server *Server) RegisterName(name<服務名> string, rcvr<對應rpc對象> interface{}) error
```
rcvr必須滿足條件：
(1) 必須導出：public, 首字母大寫
(2) 方法必須有2個參數：public、內建類型
(3) 第2個參數為指針
(4) 方法只有一個error接口返回值
```
type Sample struct{
}

func (this *Sample) Test(name string, resp *string) error {
}

rpc.RegisterName("服務名", new(Test))
```

2. ==ServeConn()== 綁定rpc服務
```
func (server *Server) ServeConn(conn io.ReadWriteCloser)
```
conn: 建立連線的socket

3. ==Call()== 調用遠程函數
```
func (client *Client) Go(serviceMethod string, args interface{}, reply interface{}, done chan *Call) *Call
```
serviceMethod: 服務名.方法名
args: 傳入參數(方法需要的參數)
reply: 傳出參數(方法返回的結果)(建立變量，&變量)

## DEMO

### server端
```
package main

import (
	"fmt"
	"net"
	"net/rpc"
)

// 定義對象
type Sample struct {
}

// 建立方法
func (this *Sample) TestSample(name string, resp *string) error {
	*resp = name + "返回"
	return nil
}

func main() {
	// 1. 註冊rpc，綁定對象方法
	err := rpc.RegisterName("service", new(Sample))
	if err != nil {
		fmt.Printf("註冊rpc失敗，err = %v", err)
		return
	}

	// 2. 設置監聽
	listener, err := net.Listen("tcp", "127.0.0.1:8800")
	if err != nil {
		fmt.Printf("設置監聽失敗，err = %v", err)
		return
	}
	defer listener.Close()
	fmt.Println("Start Listen...")

	// 3. 建立連結
	conn, err := listener.Accept()
	if err != nil {
		fmt.Printf("建立連結失敗，err = %v", err)
		return
	}
	defer conn.Close()
	fmt.Println("Start Connect...")

	// 4. 綁定服務
	rpc.ServeConn(conn)
}
```

### client端
```
package main

import (
	"fmt"
	"net/rpc"
)

func main() {
	// 1. 用rpc連接server
	conn, err := rpc.Dial("tcp", "127.0.0.1:8800")
	if err != nil {
		fmt.Printf("rpc連接失敗，err = %v", err)
		return
	}
	defer conn.Close()
	// 2. 調用遠程函數
	var req string
	err = conn.Call("service.TestSample", "something", &req)
	if err != nil {
		fmt.Printf("調用遠程函數失敗，err = %v", err)
		return
	}

	fmt.Println(req)
}

```

## 序列化
因為rpc使用go特有序列化gob，網路通訊中其他語言將會產生亂碼
因此需使用通用序列化、反序列化方案：JSON、protobuf

有內建package =="net/rpc/jsonrpc"==

### server
```
jsonrpc.ServeConn(conn)
```

### client端
```
conn, err := jsonrpc.Dial("tcp", "127.0.0.1:8800")