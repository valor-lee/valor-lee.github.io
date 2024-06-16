---
title: '[Web] Web Programming의 진화'
date: 2024-06-02 15:43:00 +09:00
categories: [Language, Java]
tags:
  [
    Web,
    CGI,
    servlet,
    
  ]
---

# 개요

최근에 많은 프레임워크와 자동화된 툴들이 상용화된 시점에서 개발일을 시작하면서,

기술의 근간에 대한 이해와 왜 사용하는지, 왜 필요한지, 등 프로그래밍의 역사에 대해서 너무 무지하다는 생각이 들었습니다.

프로그래밍의 히스토리를 모르는 상태에서 여러 기술들을 사용하다보니, 왜 필요한지에 대한 답을 모르고,
문제가 발생했을 때 대처도 어렵다는 스스로에 대한 부족함을 느끼는 요즘입니다..

그래서 당분간 <span style='background-color:grey'>백엔드 관점에서 프로그래밍의 역사나 기술에 대한 근간을 이해하는데 학습의 중점</span>을 두고자 합니다.

# Web

web이란, 인터넷 서비스 중 하나로, 인터넷을 통해 연결된 컴퓨터를 통해 사람들이 정보를 공유할 수 있는 네트워크 또는, 정보 공간을 의미합니다.

World Wide Web이라고 불리는 것이 Web으로 볼 수 있습니다.

- 인터넷 서비스로, 전자우편(SMTP), 파일전송(FTP), Telnet(원격접속) 등이 있습니다.

# Java Web Programmig history

먼저, 자바의 web 기술은

<span style='background-color: grey'>CGI -> servlet -> JSP -> (servlet + JSP) MVC 패턴 -> 스트럿츠, 웹워크 -> 스프링 MVC</span> 

순으로 발전해왔습니다.

> CGI는 Java에 국한된 기술은 아닙니다.

이에 따라 Web server의 구성은(이렇게 구분하는 것이 옳지 않을 수 있지만,) 

<span style='background-color: grey'>web server -> WAS -> Web server + WAS</span>

순으로 발전했습니다.

- server에 관련한 자세한 내용은 추후 포스팅하고자 합니다.

## static web page


그러면 web page는 web이라는 공간에 정보 공유라는 목적을 위한 수단으로 볼 수 있겠네요.

HTML(HyperText Markup Language)를 통해 구현된 Web Page는 초기에 정적인 정보를 담고 있습니다.

이를 **static web page**라고 부릅니다.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>정적 웹 페이지 예시</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f0f0f0;
        }
        header {
            background-color: #4CAF50;
            color: white;
            padding: 1em;
            text-align: center;
        }
        main {
            padding: 1em;
        }
        footer {
            background-color: #4CAF50;
            color: white;
            text-align: center;
            padding: 1em;
            position: fixed;
            width: 100%;
            bottom: 0;
        }
    </style>
</head>
<body>
    <header>
        <h1>정적 웹 페이지 예시</h1>
    </header>
    <main>
        <h2>소개</h2>
        <p>이 페이지는 HTML로 작성된 간단한 정적 웹 페이지 예시입니다.</p>
        
        <h2>링크</h2>
        <p>이를 통해 <a href="https://www.example.com">Example</a> 사이트로 이동할 수 있습니다.</p>
        
        <h2>이미지</h2>
        <p>아래는 샘플 이미지입니다:</p>
        <img src="https://via.placeholder.com/150" alt="샘플 이미지">
    </main>
    <footer>
        <p>&copy; 2024 정적 웹 페이지 예시</p>
    </footer>
</body>
</html>
```

- 위는 static web page의 예시.


점점 web이 대중화되면서 web이 보여주는 정보를 동적으로 변경되길 바라고, 데이터를 입출력, 갱신 기능에 대한 need가 발생하면서, <span style='background-color:grey;font-weight:bold'>dynamic web programming에 대한 필요성</span>이 대두되었습니다.


## CGI(Common Gateway Interface)

![image](https://github.com/valor-lee/valor-lee.github.io/assets/109330610/566f2d04-bbf7-41ce-90da-605aa3cef625)


웹 서버와 외부 프로그램간의 인터페이스 표준으로, 동적 html 파일을 생성하기 위해 도입된 기술입니다.


### 동작 방식

1. 클라이언트가 특정 url로 web server에 http 요청을 보냄
2. web server는 url에 따라 개발된 CGI program을 표준 입력과 함께 실행
3. CGI 프로세스는 표준 입력과 환경변수에 따라 동적으로 HTML 코드를 표준 출력으로 응답
4. web server는 CGI 프로세스의 표준 출력을 클라이언트에게 전달


### CGI 예시

CGI 프로그램은 다양한 언어로 구현될 수 있습니다.

```python
#!/usr/bin/env python3
import os
import cgi

# HTTP 헤더 출력
print("Content-Type: text/html")
print()

# 환경 변수 출력
print("<html><body>")
print("<h1>CGI Environment Variables</h1>")
print("<ul>")
for key, value in os.environ.items():
    print(f"<li>{key}: {value}</li>")
print("</ul>")
print("</body></html>")
```
- 이해를 돕기 위한 간단한 CGI 프로그램을 예시.

이러한 형태는 아래와 같은 **단점**이 있습니다.

1. 요청마다 새로운 CGI 프로세스가 생성되는 **프로세스 기반의 요청 처리**로 자원 소모가 많았습니다.
2. 사용자 복잡한 애플리케이션 개발에 부적절했습니다.

## Java Sevlets

CGI의 단점인, 해결하기 위해 나타난, Java기반의 동적 웹 컨텐츠를 생성하고 클라이언트 요청을 처리하는 프로그램인 Java EE의 기술입니다.

### 주요 특징은,

- 클라이언트 요청에 따라 동적으로 동작하여 응답(HTML, xml, json 등)을 반환하는 Java component
- Java Thread 기반 동작
- MVC 패턴에서 컨트롤러로 이용됨
- 보안 기능 적용에 용이함






출처

https://live-everyday.tistory.com/197
https://velog.io/@jihoson94/Servlet-Container-%EC%A0%95%EB%A6%AC
https://velog.io/@falling_star3/Tomcat-%EC%84%9C%EB%B8%94%EB%A6%BFServlet%EC%9D%B4%EB%9E%80
https://velog.io/@kdhyo/Apache-Tomcat-%EB%91%98%EC%9D%B4-%EB%AC%B4%EC%8A%A8-%EC%B0%A8%EC%9D%B4%EC%A7%80
https://velog.io/@bky373/Web-%EC%9B%B9-%EC%84%9C%EB%B2%84%EC%99%80-WAS