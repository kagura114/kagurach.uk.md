# 首先
请阅读[singbox wiki](https://sing-box.sagernet.org/)\
下面例子都是ss，并且默认以ss-2022作为加密方式（安全性更好）
## 最简单的基本配置（作为底）
```json
{
    "log": {
        "disabled": false,
        "level": "info",
        "output": "/root/box.log",
        "timestamp": true
    },
    "dns": {},
    "ntp": {},
    "inbounds": [],
    "outbounds": [
        {
            "type": "direct",
            "tag": "direct"
        }
    ],
    "route": {
        "rules":[],
        "final": "direct"
    },
    "experimental": {}
} 
```

## SS
```json
{
    "type": "shadowsocks",
    "listen": "::",
    "listen_port": 11451,
    "network": "tcp",
    "method": "2022-blake3-aes-128-gcm",
    "password": "8JCsPssfgS8tiRwiMlhARg==",
    "multiplex": {
        "enabled": true
        }
},

```
|Method|Key Length|
|--|--|
|2022-blake3-aes-128-gcm|16|
|2022-blake3-aes-256-gcm|32|
|2022-blake3-chacha20-poly1305|32|
|none|/|
|aes-128-gcm|/|
|aes-192-gcm|/|
|aes-256-gcm|/|
|chacha20-ietf-poly1305|/|
|xchacha20-ietf-poly1305|/|

对于要求了Key Length的Method(2022系列)，使用`sing-box generate rand --base64 <Key Length>`生成密码\
**2024年了该用2022Method了**（虽然Stash等不支持）
## SS 中转写法

```
               -----------------     -----------------
     psk1:psk2 |   server1     |     |    server2    |
user---------->| Relay struct  |---> | Normal config |
               | configuation  |     |   with psk2   |
               -----------------     -----------------

```
所以对server1配置如下
```json
{
  "type": "shadowsocks",
  "method": "2022-blake3-aes-128-gcm",
  "password": "8JCsPssfgS8tiRwiMlhARg==",
  "destinations": [
    {
      "name": "test",
      "server": "example.com",
      "server_port": 8080,
      "password": "PCD2Z4o12bKUoFa3cC97Hw=="
    }
  ],
  "multiplex": {}
}
```
对于2022系列Method,用户端的密码应为`{source}:{dest}`这样的写法，再这里就是`8JCsPssfgS8tiRwiMlhARg==:PCD2Z4o12bKUoFa3cC97Hw==`

## 从某入口到特定出口
比如你的服务商提供了一些内网的proxy可以解锁等(其实也可以当中转用，但是效率可能不是特别好，但是可以是任意协议的)\
这样可以很方便的实现不同入口（比如端口）对应不同的出口
```
      ---------------------------
user->|special-in--->special-out|->...
      ------------^--------------
      inbound match tag:special-in -> Rule  
```
```json
"inbounds": [    
    {
        "type": "shadowsocks",
        "tag": "special-in",
        "listen": "::",
        "listen_port": 11451,
        "network": "tcp",
        "method": "2022-blake3-aes-128-gcm",
        "password": "8JCsPssfgS8tiRwiMlhARg==",
        "multiplex": {
            "enabled": true
            }
    }
],
"outbounds": [
    {
        "type": "direct",
        "tag": "direct"
    },
    {
        "type": "socks",
        "tag": "special-out",
        "server": "114.5.1.4",
        "server_port": 19198,
        "version": "5"
    }
],
"route": {
    "rules":[
        {
            "inbound": [
                "special-in"
            ],
            "outbound": "special-out"
        }
    ],
    "final": "direct"
}
```