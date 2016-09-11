# Ring - 정리중

Ring은 클로저 웹 개발을 위한 Spec과 라이브러리다. Ring이 라이브러리를 제공하긴 하지만 중요한 것은 Spec이다. 지난 시간에 `Hello World` 예제를 만들 때 요청과 응답을 클로저의 맵으로 만들었다. 요청 맵에 `:method`와 `:path` 같은 키가 있었고 응답 맵은 `:status`, `:headers`, `:body`와 같은 키를 가지고 있었다. 이런 요청과 응답을 맵 형식으로 정의 한 것이 Ring Spec이다. 많은 클로저 웹 관련 라이브러리가 Ring Spec을 따르고 있기 때문에 기본 적인 Ring Spec을 알아야 웹 라이브러리를 쉽게 다룰 수 있다.

## 요청과 응답

Ring은 요청과 응답 맵에 대한 키를 정의 하고 있다. 예를 들면 요청 맵에는 `:request-method`에 키워드 형식으로 요청 메서드가 들어있고 응답 맵의 상태 코드는 `:status` 키에 값으로 표현된다라는 식이다. Spec은 [여기](https://github.com/ring-clojure/ring/blob/master/SPEC)에 정의되어 있고 요청 맵과 응답 맵의 예는 다음과 같다.

### 요청 맵의 예
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

요청 맵은 요청에 대한 다양한 값들이 있다. 대부분 서블릿 Request에 있는 내용을 거의 그대로 가지고 있다. 웹 서버 어플리케이션은 보통 요청에 따라 여러가지 다양한 동작을 하도록 만들기 때문에 요청 맵에서 `:uri`, `:query-string`, `:body`, `:request-method` 같은 키를 자주 사용한다.

### 응답 맵의 예
```clojure
{:status 200
 :headers {"Content-Type" "text/html"}
 :body "Hello World"}
```

응답 맵은 요청 맵 보다 단순하다. `:status`, `:headers`, `:body` 키를 정의 하고 있다. `:body`의 값은 문자열, 시퀀스, 파일, InputStream이 될 수 있다. `:status`와 `:headers`는 응답에 필수로 포함해야한다.

## 핸들러 함수

### 동기 핸들러 함수

Ring은 요청을 처리해 응답을 주는 함수에 대한 형식도 정의하고 있다. 이 함수를 Ring은 `handler`라고 부른다. 핸들러는 단순하게 요청 맵을 인자로 받고 응답 맵을 리턴하는 함수다. 아래는 접속 아이피 주소를 `text/html` 형식으로 출력하는 핸들러 함수다.

```clojure
(defn sample-handler [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body (:remote-addr request)})
```

### 비동기 핸들러 함수

Ring은 비동기 핸들러 함수에 대한 형식도 정의하고 있다. 비동기 핸들러 함수는 아래와 같이 생겼다.

```clojure
(defn sample-async-handler [request respond raise]
  (respond {:status 200
            :headers {"Content-Type" "text/html"}
            :body (:remote-addr request)}))
```

비동기 핸들러 함수는 `request` 외에 `respond`와 `raise` 함수를 받는다. `respond`함수는 응답을 보내는 함수이고 `raise` 함수는 예외를 보내는 함수다. 비동기 핸들러는 핸들러 함수가 종료되어도 응답이 전달되지 않고 `respond`나 `raise`함수를 불러야 응답이 전달된다. (글을 작성하는 시점에 비동기 핸들러 함수는 ring에서 만든 jetty 어댑터외에 아직 지원하는 어댑터가 없다.)

## 미들웨어

Ring은 핸들러 함수 이전과 이후에 공통 로직을 분리해서 재활용 할 수 있도록 미들웨어라는 Spec을 정의 했다. Ring 미들웨어는 로깅, 응답 변환, 요청 변환, 인증, 예외 처리, 라우팅등 여러가지 공통 로직들을 만드는데 사용할 수 있다. 그리고 이미 만들어 놓은 많은 미들웨어가 있기 때문에 가져다 사용할 수 있다.

미들웨어는 핸들러 함수를 인자로 받아서 다시 핸들러 함수를 리턴하는 함수다.

```clojure
(defn wrap-bypass-middleware [handler]
  (fn [request]
    (handler request)))
```

위에 예제는 핸들러 함수를 받아서 핸들러 함수를 실행하는 핸들러 함수를 리턴하는 미들웨어다. 결국 아무것도 하지 않는 미들웨어다.
사용은 아래와 같이 하면 된다. 보통 미들웨어의 이름은 `warp-`으로 시작한다.

```clojure
(defn sample-handler [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body (:remote-addr request)})
   
(def app (wrap-bypass-middleware sample-handler))
```

아래는 수행시간을 포함한 요청 맵을 로그에 출력하는 뭔가 일을 하는 미들웨어 예제다.

```clojure
(defn wrap-processing-time [handler]
  (fn [request]
    (let [start-time (System/currentTimeMillis)
          response (handler request)]
      (log/info (assoc request :processing-time (- (System/currentTimeMillis) start-time)))
      response)))
      
(def app (wrap-processing-time sample-handler))
```

만약 미들웨어를 여러개 적용하고 싶다면 연속적으로 미들웨어를 적용하면 된다.

```clojure
(def app 
  (-> sample-handler
      wrap-processing-time
      warp-access-log
      wrap-not-found))
```

미들웨어는 중첩된 함수이기 때문에 미들웨어 적용 순서가 다음과 같이 풀어보면 실행순서를 어렵지 않게 이해할 수 있다.

```clojure
(defn hello-world-handler [request]
  (log/info "hello world handler")
  {:status 200 :headers {} :body "Hello world"})

(defn wrap-middleware1 [handler]
  (fn [request]
    (log/info "pre handler - middleware1")
    (let [response (handler request)]
      (log/info "post handler - middleware1")
      response)))

(defn wrap-middleware2 [handler]
  (fn [request]
    (log/info "pre handler - middleware2")
    (let [response (handler request)]
      (log/info "post handler - middleware2")
      response)))

(def app (-> hello-world-handler
           wrap-middleware1
           wrap-middleware2))
```

위와 같이 `hello-world-handler` 핸들러에 `wrap-middleware1`과 `wrap-middleware2`를 순서대로 적용한 핸들러를 실행하면 아래와 같은 순서로 로그가 출력 된다.

```
pre handler - middleware2
pre handler - middleware1
hello world handler
post handler - middleware1
post handler - middleware2
```

핸들러를 처리하기 전에 실행되는 로직은 가장 마지막에 적용한 미들웨어 부터 적용되고 핸들러를 처리한 후에 실행되는 로직은 가장 먼저 적용한 미들웨어 부터 실행되는 것을 볼 수 있다.

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











