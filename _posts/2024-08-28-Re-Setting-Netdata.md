---
title: "Re-Setting Netdata in Home Server"
writer: Hyogeun Park
date: 2024-08-29 19:42:23 +09:00
categories: [Home-server, Netdata]
tags: [netdata]
description: 홈서버에 netdata 재설치
---

기존에 Netdata를 사용하고 있었지만 뭔가 꼬여서 그냥 재설치를 해줬다.

## 기존 netdata 삭제
새로 설치를 하기 위해서 먼저 기존의 netdata를 깔끔하게 삭제하는 과정이 필요했다  
```zsh
curl https://get.netdata.cloud/kickstart.sh > /tmp/netdata-kickstart.sh && sh /tmp/netdata-kickstart.sh --uninstall
```

위 명령어를 통해 기존 netdata를 삭제하려고 하였지만 uninstall을 실행한 후에도 기존 파일들이 남아있다는 경고문이 떠서  
그냥 netdata 관련 파일들을 모조리 찾아 지워버리기로 했다  
```zsh
sudo find / -name "*netdata*" -exec rm -rf {} \;
```
~~???: 진정 새로운 무언가를 창조하려 한다면 처음부터 다시 시작해야 합니다~~

## Netdata 재설치
설치는 자동화가 되어있기 때문에 딱히 건드릴 부분은 없었다
```zsh
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh
```

## Nginx 세팅
`localhost:19999`에서 잘 돌아가는 것을 확인했으면 외부에서 간단하게 접속할 수 있도록 nginx에서 프록시 세팅을 해주자
```
upstream netdata-backend {
    server 127.0.0.1:19999;
    keepalive 64;
}

server {
    listen 80;
    server_name SERVER_NAME;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name SERVER_NAME;
    ssl_certificate CERT_PATH;
    ssl_certificate_key CERT_KEY_PATH;

    location / {
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://netdata-backend;
        proxy_http_version 1.1;
        proxy_pass_request_headers on;
        proxy_set_header Connection "keep-alive";
        proxy_store off;
    }
}
```

## 암호를 통해 netdata 접근 제한
Netdata의 문제는 로그인 기능을 지원하지 않아 누구나 내 서버의 상태를 모니터링 할 수 있다는 것이다  
따라서 암호를 걸어 보호해야하는데 간단하게 Linux Apache Password를 이용했다.
```zsh
sudo htpasswd -c USERFILE_PATH USERNAME
```
암호를 만들어 설정하고 nginx에 아래의 두 줄을 추가해준다
```
    auth_basic "Authentication Required";
    auth_basic_user_file USERFILE_NAME;
```

