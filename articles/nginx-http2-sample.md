---
title: "NginxのリバースプロキシでHTTP/2で通信する"
emoji: "🐶"
type: "tech"
topics: ["nginx", "http2"]
published: false
---

# はじめに
Nginxをproxyとして使う場合にHTTPではなくHTTP/2で通信したい場合がある。
どのようにするかはNginxのconfigに `http2` を追加してあげるだけですが、実際に試して動作確認できるように簡単なサンプルとその際の設定を書いていきます。

動作確認はxxxxxで行っています。

※M1 Macで確認しているので arm64のイメージになっています


# サンプルアプリケーションを動作させる
## Hello goアプリケーション
動作確認用の簡単なGoアプリケーションを用意します。
リクエストを受け取って `Hello go!!` と返すだけのアプリケーションです。

```go:main.go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", handler)

	log.Println("server started")
	server := &http.Server{Addr: ":9000"}
	if err := server.ListenAndServe(); err != nil {
		log.Println(err)
	}
}

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello go!!")
}
```

## docker composeで起動
NginxとGoアプリケーションを動かすためのDockerfileを用意してdocker composeでまとめて起動できるようにします。


```docker:Dockerfile
FROM arm64v8/golang:1.17

WORKDIR /go/src/app
COPY ./main.go ./main.go

CMD ["go", "run", "main.go"]
```

```docker:DockerfileNginx
FROM nginx

# デフォルトのnginx imageの設定を上書きする
COPY etc/go-app.conf /etc/nginx/conf.d/default.conf
```

```text:etc/go-app.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        proxy_pass   http://app:9000/;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

```yml:docker-compose.yml
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - 9000:9000
  nginx:
    build:
      context: .
      dockerfile: DockerfileNginx
    ports:
      - 80:80
```

動作確認

```bash
% docker compose up -d
% docker compose ps
NAME                COMMAND                  SERVICE             STATUS              PORTS
nginx-go_app_1      "go run main.go"         app                 running             0.0.0.0:9000->9000/tcp, :::9000->9000/tcp
nginx-go_nginx_1    "/docker-entrypoint.…"   nginx               running             80/tcp
% curl localhost:80
Hello go!!
```

curlを`-i`オプションで実行するとHTTP/1.1で通信していることがわかる

```bash
% curl -i localhost:80
HTTP/1.1 200 OK
```

## 自己証明書を使ってhttpsで動かす
HTTP/2自体は仕様上はHTTPSに限らずHTTPの通信にも対応していますが、主要ブラウザーがTLS拡張のみを対応するなど実際にWebサービスで使う場合はHTTPS前提になる場合がほとんどだと思います。
そのため、自己証明書を使ってHTTPSで動くようにしていきます。

証明書作成

```bash
% mdkir ./ssl
% openssl req -new -x509 -sha256 -newkey rsa:2048 -days 365 -nodes -out ./ssl/ssl.pem -keyout ./ssl/ssl.key
```

作成した証明書をnginxで使用する

```yml:docker-compose.yml
---
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - 9000:9000
  nginx:
    build:
      context: .
      dockerfile: DockerfileNginx
    ports:
+      - 443:443
+    volumes:
+      - ./ssl:/ssl
```

```text:etc/go-app.conf
server {
+    listen       443 ssl;
+    listen  [::]:443 ssl;
    server_name  localhost;

+    ssl_certificate /ssl/ssl.pem;
+    ssl_certificate_key /ssl/ssl.key;

    location / {
        proxy_pass   http://app:9000/;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

動作確認

```
% curl -k https://localhost
Hello go!!
```

# nginxをHTTP2に対応させる

ようやく本題にたどり着きましたが、やることは以下のようにnginxのconifgファイルにhttp2を追加するだけです。

```text:etc/go-app.conf
server {
+    listen       443 ssl http2;
+    listen  [::]:443 ssl http2;
```


```
# 設定反映
% dc up --build -d

# 動作確認
% curl -k -I https://localhost
HTTP/2 200
```

HTTP/2で通信していることが確認できました🎉

今回curlで確認しましたが、curlのバージョンが古いなどでhttp2で通信してくれない場合もあるので注意です。
HTTP/2で通信しようとしているかは `-v` オプションをつけて実行することで確認できます。

以下のように `ALPN, offering h2` が表示されればHTTP/2で通信しようとしてくれています。
自分の環境だと最初はh2が出ていなかったためcurlを入れ直することで、h2が表示されレスポンスもHTTP/2と表示されるようになりました。

```
% curl -k -v https://localhost
*   Trying 127.0.0.1:443...
* Connected to localhost (127.0.0.1) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
```

# まとめ
nginxをproxyサーバーとしてHTTP/2で通信する方法を簡単なアプリケーションで試してみました。
設定自体はすごく簡単なものですが、試す中でHTTP/2やそのプロトコルネゴシエーションの仕様や、周辺ツールの対応状況なども知ることができてよかったです。


# 余談：SSL/TLSなしでHTTP2を動かすことができるか
試せてはないですが [SSL/TLSなしのHTTPでもHTTP/2に対応してくれるWebサーバーのリバースプロキシ](https://scrapbox.io/nwtgck/SSL%2FTLS%E3%81%AA%E3%81%97%E3%81%AEHTTP%E3%81%A7%E3%82%82HTTP%2F2%E3%81%AB%E5%AF%BE%E5%BF%9C%E3%81%97%E3%81%A6%E3%81%8F%E3%82%8C%E3%82%8BWeb%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E3%81%AE%E3%83%AA%E3%83%90%E3%83%BC%E3%82%B9%E3%83%97%E3%83%AD%E3%82%AD%E3%82%B7) に書かれているようにNginxでも `direct` ネゴシエーションを使えば可能だそうです。

また[http2-spec](https://github.com/httpwg/http2-spec/wiki/Implementations)を見れば各ツールがどんなプロトコルに対応した実装されているか確認できるのでHTTP/2を使う時に参考になりそうです。