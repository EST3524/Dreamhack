# **Tools**
- 이 파일에는 nc, SSH, Docker, 정규 표현식에 대해서 간단하게 정리한다.

## **I. nc**
- nc(netcat)은 클라이언트가 서버에서 특정 포트를 통해 서비스하고 있는 프로그램과
  통신하기 위해서 사용한다.

### **nc 사용법**
- nc의 가장 간단한 사용 방식은 **nc hostname(ip) port** 이다.

''' bash
$ nc google.com 80
GET / HTTP/1.1

HTTP/1.1 200 OK
Date: Sat, 23 Nov 2024 01:12:03 GMT
Expires: -1
Cache-Control: private, max-age=0
Content-Type: text/html; charset=ISO-8859-1
...
'''
