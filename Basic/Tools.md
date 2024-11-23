# **Tools**
- 이 파일에는 nc, SSH, Docker, 정규 표현식에 대해서 간단하게 정리한다.

## **I. nc**
- nc(netcat)은 클라이언트가 서버에서 특정 포트를 통해 서비스하고 있는 프로그램과
통신하기 위해서 사용한다.
- 현대의 거의 모든 네트워크 통신은 네트워크 소켓을 통해 이루어지는데, 소켓은
통신을 위한 가장 작은 단위의 프로토콜이다.

### **nc 사용법**
- nc의 가장 간단한 사용 방식은 **nc hostname(ip) port** 이다.

```bash
$ nc google.com 80
GET / HTTP/1.1

HTTP/1.1 200 OK
Date: Sat, 23 Nov 2024 01:12:03 GMT
Expires: -1
Cache-Control: private, max-age=0
Content-Type: text/html; charset=ISO-8859-1
...
```

- 앞의 예시는 google.com에 80번 포트로 연결을 요청한 것이다.
80번 포트는 HTTP 통신에 사용되는 기본 포트이다.
80번 포트로 네트워크 연결을 시도한 후에, **GET / HTTP/1.1** 을 입력한
후 입력이 끝났다는 뜻으로 엔터키를 한번 더 입력해야 한다. 이 때,  
표준 규약에 따라서 HTTP와 1.1(버전) 사이에 **HTTP / 1.1**와 같이 공백이 있다면
잘못된 요청으로 간주한다.  

## **II. SSH**
- SSH(Secure Shell, Secure Socket Shell)는 원격 서버(컴퓨터)에 연결할 수 있도록 해 주는
암호화된 네트워크 프로토콜이다.
