---
title: "Ngrok로 외부에서 로컬 웹서버에 접근"
datePublished: Thu Jan 27 2022 04:40:26 GMT+0000 (Coordinated Universal Time)
cuid: ckywhn16b0wjn2vs15qki7b7e
slug: ngrok
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1643257790467/53nS-k4t-.png
tags: programming

---


![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643257429533/bFKV8yTqz.png)

[ngrok](https://ngrok.com/)는 localhost로 동작하고 있는 웹서버를 외부에서 접근할 수 있도록 하는 서비스입니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643257790467/53nS-k4t-.png)

가격은 기본적으로 무료이며 고정 도메인 및 서브도메인을 사용하기 위해서는 유료 라이센스가 필요합니다. 가격이 저렴한 편이므로 원격 API로 로컬 호스트를 테스트하는 분께는 좋은 솔루션이 됩니다.

계정을 생성하면 메뉴에서 `Setup & Installation`을 통해 클라이언트를 다운로드 받고 ngrok 서비스를 이용할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643257891716/X1vB_Y6rM.png)

적절한 위치에 다운로드 한 파일을 내려받고,

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643257921578/Y5ds0MIrG.png)

계정 연결을 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643258020215/Hq2XGbC94.png)

이후 연결 할 http 포트를 다음처럼 지정해서 실행하면

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643258051119/PA3aCjhnJ.png)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643258355212/QLVV_nf7Q.png)

외부에서 `http://8a88-180-83-215-76.ngrok.io/` 형태의 주소로 로컬에서 실행되고 있는 웹서버에 접속이 가능하게 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1643258136496/cETnYsPo4.png)
