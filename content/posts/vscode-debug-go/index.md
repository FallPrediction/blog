+++
title = '用VSCode Debug GO程式吧'
date = 2024-03-25T21:33:22+08:00
draft = false
description = '用VSCode Debug GO程式吧'
toc = true
tags = ['GO']
categories = ['GO']
+++

這篇分為兩個部分，需要的人可以跳著看
1. 本地開發
2. Docker開發

## Debug GO
[官方文件](https://go.dev/doc/gdb)推薦使用Delve，而不是GDB
雖然Delve可以用command來debug，但我已經很習慣用VSCode邊寫邊debug了，所以接下來來設定環境吧~

## 本地開發
側邊欄選擇Run and Debug，點擊create a launch.json file，然後選擇Go: Launch Package，VSCode會自動建立一個.vscode/launch.json檔，長得像這樣：
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch Package",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${fileDirname}"
        }
    ]
}
```
然後就可以下斷點，F5開始debug，很簡單吧！

## 但我用的是Docker
現在很多人用Docker作為開發環境，尤其是Docker Compose能一次開很多服務真的很方便

這種情況可以用Remote Debug

一樣在側邊欄選擇Run and Debug，點擊create a launch.json file，然後選擇Connect to a server

VSCode也會建立一個.vscode/launch.json檔，但還沒完，需要修改兩個參數：
- port：這個隨便取，不要跟其他服務撞到就好
- substitutePath：怎麼把code volume 到conatiner裡面，這裡就怎麼設
最後我的launch.json長這樣：
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Connect to server",
            "type": "go",
            "request": "attach",
            "mode": "remote",
            "substitutePath": [
                {
                    "from": "${workspaceFolder}",
                    "to": "/app"
                },
            ],
            "port": 4040,
            "host": "127.0.0.1"
        }
    ]
}
```

現在來給net/http的[範例code](https://pkg.go.dev/net/http#ListenAndServe)寫一個Dockerfile
```
FROM golang:1.21.4-alpine

RUN mkdir /app
COPY .. /app
WORKDIR /app

RUN go install github.com/go-delve/delve/cmd/dlv@master
ENTRYPOINT dlv debug --headless --continue --listen :4040 --accept-multiclient .
```
`--continue`可以在delve 伺服器一啟動後立刻開始debug，而用了`--continue`以後就要一並使用`--accept-multiclient`

注意compose.yaml也要設定port mapping和volume！
```yaml
version: "3"
services:
  app:
    build:
      dockerfile: Dockerfile
      context: .
    volumes:
      - ./:/app
    ports:
      - 8080:8080
      - 4040:4040
    networks:
      - go-docker-net

networks:
  go-docker-net:
```

## 跟CompileDaemon一起
開發的時候，每次變更都要手動build非常不方便，所以我都會用[CompileDaemon](https://github.com/githubnemo/CompileDaemon)，它可以偵測*.go檔案變更，自動build專案

每次編譯之後，可以用CompileDaemon的**command option**，寫一個script 發送SIGINT終止Delve的headless server

kill_and_debug.sh
```sh
#! /bin/bash
kill -SIGINT $(ps aux | grep '[d]lv exec' | awk '{print $1}')
dlv exec --headless --continue --listen :4040 --accept-multiclient ./app
```
修改一下Dockerfile
```
FROM golang:1.21.4-alpine

# Environment variables which CompileDaemon requires to run
ENV PROJECT_DIR=/app \
    GO111MODULE=on \
    CGO_ENABLED=0

RUN mkdir /app
COPY .. /app
WORKDIR /app

RUN go get github.com/githubnemo/CompileDaemon
RUN go install github.com/githubnemo/CompileDaemon
RUN go install github.com/go-delve/delve/cmd/dlv@master

ENTRYPOINT CompileDaemon -build="go build -o app" -command="sh kill_and_debug.sh"
```

每次在儲存新修改之前，先斷開remote debug的連接，儲存後等待build，之後就可以跟平常一樣debug了~

## 參考
- [vscode-go debugging](https://github.com/golang/vscode-go/wiki/debugging)
- [delve F&Q](https://github.com/go-delve/delve/blob/master/Documentation/faq.md)
- [Live Reloading Your Golang Apps Made Easy – Here’s How](https://thegodev.com/live-reloading/)