---
title: "O'ReillyでGo言語に入門してみた"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

# はじめに

こんにちは。
にしやまです。

私事ですが、この度転職が決まり、3月から新しい会社で働くこととなりました。
新しい会社ではGo言語を使っているそうで、買ってからしばらく温めていたO'Reillyの「Go言語によるWebアプリケーション開発」を利用してGo言語に入門することにしました。

やってみたところ、Go言語の基本的なところで躓くことが多かったので、第一章「WebSocketを使ったチャットアプリケーション」の内容をできるだけ細かく、初心者向けになるように解説していこうと思います。

解説に利用するコードは
https://github.com/nsym-m/go-programming-blueprints

# 動作環境

Windows 10 Pro

```bash
PS C:\Users\nsym\Projects\go-programming-blueprints> go version
go version go1.17.7 windows/amd64
```

# 環境構築

最新版のGo言語をインストールすることを想定しています。
https://go.dev/dl/
より
OS: Windows
Kind: Installer
Arch: 自分のPCと同じもの
をインストールし、実行します。
記載内容に問題がなければYesを何度か繰り返し、Go言語のインストールが完了します。
この時点でPATHも通っています。

# 解説

main.go
```go
package main

import (
	"flag"
	"html/template"
	"log"
	"net/http"
	"path/filepath"
	"sync"
)

type templateHandler struct {
	once     sync.Once
	filename string
	templ    *template.Template
}

func (t *templateHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	t.once.Do(func() {
		t.templ =
			template.Must(template.ParseFiles(filepath.Join("templates", t.filename)))
	})
	t.templ.Execute(w, r)
}

func main() {
	var addr = flag.String("addr", ":8080", "アプリケーションのアドレス")
	flag.Parse()
	r := newRoom()
	// r.tracer = trace.New(os.Stdout)
	http.Handle("/", &templateHandler{filename: "chat.html"})
	http.Handle("/room", r)

	// チャットルームの開始
	go r.run()

	// Webサーバーを開始
	log.Println("Webサーバーを起動します。ポート: ", *addr)
	if err := http.ListenAndServe(*addr, nil); err != nil {
		log.Fatal("ListenAndServe:", err)
	}
}
```

chat.html
```html
<html>
    <head>
        <title>チャットだよ</title>
        <style>
            input { display: block; }
            ul { list-style: none; }
        </style>
    </head>
    <body>
        <ul id="messages"></ul>
        WebSocketを使ったチャットアプリ
        <form id="chatbox">
            <textarea></textarea>
            <input type="submit" value="送信" />
        </form>
        <script src="//ajax.googleapis.com/ajax/libs/jquery/1.5.1/jquery.min.js"></script>
        <script>
            $(function(){
                let socket = null;
                let msgBox = $("#chatbox textarea");
                let messages = $("#messages");
                $("#chatbox").submit(function() {
                    if (!msgBox.val()) return false;
                    if (!socket) {
                        alert("error: WebSocket接続が行われていません。");
                        return false;
                    }
                    socket.send(msgBox.val());
                    msgBox.val("");
                    return false;
                })
                if (!window["WebSocket"]) {
                    alert("error: WebSocketに対応していないブラウザです。")
                } else {
                    socket = new WebSocket("ws://{{.Host}}/room");
                    socket.onclose = function() {
                        alert("接続が終了しました")
                    }
                    socket.onmessage = function(e) {
                        console.log(e.data);
                        messages.append($("<li>").text(e.data));
                    }
                }
            });
        </script>
    </body>
</html>
```


client.go
```go
package main

import (
	"github.com/gorilla/websocket"
)

// clientはチャットを行っている1人のユーザーを表す
type client struct {
	// socketはこのクライアントの為のWebSocket
	socket *websocket.Conn
	// sendはメッセージが送られるチャネル
	send chan []byte
	// roomはこのクライアントが参加しているチャットルーム
	room *room
}

func (c *client) read() {
	for {
		if _, msg, err := c.socket.ReadMessage(); err == nil {
			c.room.forward <- msg
		} else {
			break
		}
	}
	c.socket.Close()
}

func (c *client) write() {
	for msg := range c.send {
		if err := c.socket.WriteMessage(websocket.TextMessage, msg); err != nil {
			break
		}
	}
}

```

room.go
```go
package main

import (
	"log"
	"net/http"

	"go-programming-blueprints/trace"

	"github.com/gorilla/websocket"
)

type room struct {
	// forwardは他のクライアントに転送するためのメッセージを保持するチャネル
	forward chan []byte
	// joinはチャットルームに参加しようとしているクライアントの為のチャネル
	join chan *client
	// leaveはチャットルームから退出しようとしているクライアントの為のチャネル
	leave chan *client
	// clientsには在室しているすべてのクライアントが保持されます
	clients map[*client]bool
	// tracerはチャットルームで行われた操作のログを受け取る
	tracer trace.Tracer
}

func (r *room) run() {
	for {
		select {
		case client := <-r.join:
			// 参加
			r.clients[client] = true
			r.tracer.Trace("新しいクライアントが参加しました")
		case client := <-r.leave:
			// 退室
			delete(r.clients, client)
			close(client.send)
			r.tracer.Trace("クライアントが退室しました")
		case msg := <-r.forward:
			// 全てのクライアントにメッセージを転送
			r.tracer.Trace("メッセージを受信しました: ", string(msg))
			for client := range r.clients {
				select {
				case client.send <- msg:
					// メッセージ送信
					r.tracer.Trace(" -- クライアントに送信されました")
				default:
					// 送信失敗
					delete(r.clients, client)
					close(client.send)
					r.tracer.Trace(" -- 送信に失敗しました。クライアントをクリーンアップします")
				}
			}
		}
	}
}

const (
	socketBufferSize  = 1024
	messageBufferSize = 256
)

var upgrader = &websocket.Upgrader{
	ReadBufferSize:  socketBufferSize,
	WriteBufferSize: socketBufferSize,
}

// websocketでHttp Serverをラップする
func (r *room) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	socket, err := upgrader.Upgrade(w, req, nil)
	if err != nil {
		log.Fatal("ServeHTTP:", err)
		return
	}
	client := &client{
		socket: socket,
		send:   make(chan []byte, messageBufferSize),
		room:   r,
	}
	r.join <- client
	defer func() { r.leave <- client }()
	go client.write()
	client.read()
}

// すぐに利用できるチャットルームを生成して返す
func newRoom() *room {
	return &room{
		forward: make(chan []byte),
		join:    make(chan *client),
		leave:   make(chan *client),
		clients: make(map[*client]bool),
	}
}

```



