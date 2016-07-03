# Ring - 정리중

Ring은 클로저 웹 개발을 위한 Spec과 라이브러리다. Ring이 라이브러리를 제공하긴 하지만 중요한 것은 Spec이다. 지난 시간에 `Hello World` 예제를 만들 때 요청과 응답을 클로저의 맵으로 만들었다. 요청 맵에 `:method`와 `:path` 같은 키가 있었고 응답 맵은 `:status`, `:headers`, `:body`와 같은 키를 가지고 있었다. 먼저 Ring이 정의하고 있는 요청 맵과 응답 맵의 형식을 알아보자.

## 요청과 응답

Ring은 요청과 응답 맵에 대한 키를 정의 하고 있다. 예를 들면 요청 맵에는 `:request-method`에 키워드 형식으로 요청 메서드가 들어있고 응답 맵의 상태 코드는 `:status` 키에 값으로 표현된다라는 식이다. Spec은 [여기](https://github.com/ring-clojure/ring/blob/master/SPEC)에 정의되어 있고 요청 맵과 응답 맵의 예는 다음과 같다.

### 요청
```clojure
{:ssl-client-cert nil
 :protocol "HTTP/1.1"
 :remote-addr "0:0:0:0:0:0:0:1"
 :headers {"cookie" "JSESSIONID=QOm1Y9_QqqCFdMCkhkJHqzjK6jGYO99yEpG0kKFF; XSRF-TOKEN=Jd3xsCJYWB; ring-session=3feda23c-c493-4365-b7c6-821350460a34"
           "cache-control" "max-age=0"
           "accept" "text/htmlapplication/xhtml+xmlapplication/xml;q=0.9image/webp*/*;q=0.8"
           "upgrade-insecure-requests" "1"
           "connection" "keep-alive"
           "user-agent" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_5) AppleWebKit/537.36 (KHTML like Gecko) Chrome/51.0.2704.103 Safari/537.36"
           "host" "localhost:3000"
           "accept-encoding"
           "gzip deflate sdch"
           "accept-language" "ko-KRko;q=0.8en-US;q=0.6en;q=0.4it;q=0.2"}
 :server-port 3000
 :content-length nil
 :content-type nil
 :character-encoding nil
 :uri "/"
 :server-name "localhost"
 :query-string nil
 :body #object[org.eclipse.jetty.server.HttpInputOverHTTP 0x5247e9b6 "HttpInputOverHTTP@5247e9b6"]
 :scheme :http
 :request-method :get}
```

### 응답
```clojure
{:status 200
 :headers {"Content-Type" "text/html"}
 :body "Hello World"}
```

## 핸들러 함수

Ring은 요청을 처리해 응답을 주는 함수에 대한 형식도 정의하고 있다. 이 함수를 Ring은 `handler`라고 부른다. 핸들러는 단순하게 요청 맵을 인자로 받고 응답 맵을 리턴하는 함수다. 아래는 접속 아이피 주소를 `text/html` 형식으로 출력하는 핸들러 함수다.

```clojure
(defn sample-handler [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body (:remote-addr request)})
```

## 어댑터

Ring에서 HTTP 요청과 응답을 어떻게 맵으로 표현하고 또 HTTP 요청을 처리해서 응답을 생성하는 함수가 어떻게 생겼는지 알아봤다. Ring은 서버 라이브러리도 아니고 서버를 어떻게 구현해야하는지에 대한 Spec은 없다. 그래서 Ring을 사용하려면 Ring 핸들러 함수를 실행해줄 서버 구현체가 필요하다. 이 구현체를 Ring 어댑터라고 부른다. 다행이도 Ring은 표준 뿐만 아니라 라이브러리 를 제공하고 있는데 그 중에 Jetty 서버로 Ring 어댑터를 구현한 `ring-jetty-adapter`라는 라이브러리를 제공한다. 하지만 Netty, Undertow, Grizzly, Tomcat, Nginx등 다양한 Ring 어댑터가 있기 때문에 꼭 Ring에서 제공하는 `ring-jetty-adapter`를 사용하지 않아도 된다.

지난 시간에 만든 `Hello World`는 Ring과 비슷하게 요청 맵과 응답 맵을 정의하고 어댑터와 비슷한 `run-jetty` 함수를 만들었다. 하지만 몇가지 Ring Spec과 맞지 않는 부분이있다. 먼저 `build-request-map`은 요청 중에서 `:method`와 `:path`만 처리할 수 있다. 또 `run-jetty` 함수는 두번째 인자로 서버 옵션을 받을 수 있어야 Ring 어댑터 Spec에 맞다.

그럼 Ring Spec에 맞게 고쳐보자. 아래는 지난 시간에 만든 `webapp.server` 코드다.

```clojure
(ns webapp.server
  (:import [org.eclipse.jetty.server Server ServerConnector]
           org.eclipse.jetty.server.handler.AbstractHandler))

(defn- build-request-map [^HttpServletRequest request]
  {:method (keyword (lower-case (.getMethod request)))
   :path (str (.getPathInfo request) "?"
           (.getQueryString request))})

(defn- new-handler [handler]
  (proxy [AbstractHandler] []
    (handle [target base-request request ^HttpServletResponse response]
      (let [request-map (build-request-map request)
            {:keys [status body]} (handler request-map)]
        (.setStatus response status)
        (with-open [writer (.getWriter response)]
          (.print writer body))))))
          
(defn run-jetty [handler]
  (let [^Server server (Server.)
        ^ServerConnector connector (doto (ServerConnector. server)
                                     (.setPort 8080))]
    (doto server
      (.addConnector connector)
      (.setHandler (new-handler handler))
      (.start)
      (.join))))
```





지난 시간에 자바 Jetty 라이브러리로 `Hell World` 웹 어플리케이션을 만들어 봤다. 지난 시간에 만들었던 `run-jetty` 함수와 `handler` 함수의 요청 맵과 응답 맵 형식은 이미 Ring 라이브러리에 구현되어 있는 기능이다. Ring 라이브러리는 요청을 처리하는 핸들러의 요청 맵과 응답 맵의 형식(예를 들면 클라이언트 IP 주소는 `:remote-addr` 키에 들어있다)을 정의 한 Spec과 이 맵을 처리하는 라이브러리들로 구성되어 있다. 그리고 편의를 위해 핸들러 함수를 Jetty로 실행 할 수 있는 어댑터도 제공한다.

## Ring
- [SPEC](https://github.com/ring-clojure/ring/blob/master/SPEC)
  - 요청/응답 맵에 어떤 키를 사용하는지 설명되어 있다. 그리고 핸들러, 미들웨어, 어댑터에 역할에 대해 설명하고 있다.
- [ring-core](https://github.com/ring-clojure/ring/tree/master/ring-core)
  - 요청 맵과 응답 맵을 다루는 함수들과 일반적으로 많이 사용하는 미들웨어를 제공한다.
- [ring-devel](https://github.com/ring-clojure/ring/tree/master/ring-devel)
  - 개발 환경에서 사용할 수 있는 Reload 미들웨어나 Stackstrace 미들웨어, 더미 핸들러, 디버깅 미들웨어 같은 것을 제공한다.
- [ring-jetty-adapter](https://github.com/ring-clojure/ring/tree/master/ring-jetty-adapter)
  - Jetty로 핸들러를 실행해 볼 수 있는 `run-jetty` 함수를 제공한다.
- [ring-servlet](https://github.com/ring-clojure/ring/tree/master/ring-servlet)
  - 자바 요청/응답 서블릿 객체를 클로저 맵으로 바꿔주는 함수를 제공한다.

Ring은 SPEC 문서와 4개 라이브러리가 있다. 지난 시간에 만든 `Hello World`를 Ring 라이브러리로 만들어 보자. 우리는 `run-jetty`만 필요하기 때문에 `ring-jetty-adapter` 라이브러리만 사용할 것이다.

## ring-jetty-adapter

먼저 빈 프로젝트를 만들고 `project.clj`에 `ring-jetty-adapter` 라이브러리를 추가하자.

```bash
lein new webapp
```

```clojure
(defproject webapp "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.8.0"]
                 [ring/ring-jetty-adapter "1.5.0"]]
  :main webapp.core)
```

그리고 `webapp.core` 네임스페이스에 `-main` 함수를 만들고 지난 시간 예제에서 마지막에 했던 요청 메서드와 경로를 출력해주는 코드를 작성해보자.

```clojure
(ns webapp.core
  (:require [ring.adapter.jetty :refer [run-jetty]]))

(defn handler [request]
  {:status 201
   :headers {"Content-Type" "text/html;charset=utf-8"}
   :body (str
           "<p>method: " (:request-method request) "</p>"
           "<p>path: " (str (:uri request) "?" (:query-string request)) "</p>")})

(defn -main [& args]
  (run-jetty handler {:port 8080}))
```

지난번 작성한 코드랑 거의 비슷하다. 다른 점은 `handler` 함수의 파라미터인 `request` 맵에 들어있는 키 이름과 경로 형식이다. 특히 경로는 `:uri`와 `:query-string`이 분리되어 들어온다. 또 다른 점은 `run-jetty` 함수에 두번째 인자로 옵션을 받아 포트 번호 같은 것을 넘겨줬다. 그래도 지난번 직접 만든 `run-jetty`와 구조가 거의 비슷하기 때문에 `ring.adapter.jetty/run-jetty` 함수가 어떻게 구현되었을지 예상할 수 있다.

## ring-core

ring.core에 있는

ring.util.response에 redirect, not-found, response, status, header, content-type, set-cookie, , file-response, resource-response 설명하기

## ring-devel

## ring-servelt







