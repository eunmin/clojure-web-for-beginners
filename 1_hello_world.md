# Hello World 웹 어플리케이션

웹 개발을 위한 클로저 라이브러리들은 많이 나와 있고 이것들을 잘 이용해 많은 코드 작성 없이 웹 어플리케이션을 만들 수 있다. 처음에는 이런 라이브러리들을 이해하기 위해 클로저 라이브러리가 아닌 자바 라이브러리를 가지고 웹 어플리케이션을 만들어 보자. 클로저는 자바 라이브러리를 사용할 수 있기 때문에 Jetty 웹 라이브러리를 이용해서 Hello World 문자열을 표시하는 웹 어플리케이션을 만들어보자.

Jetty 라이브러리는 [홈페이지](http://www.eclipse.org/jetty/)에 문서와 최신 버전에 대한 정보들이 잘 정리되어 있다. 이 문서를 작성하는 시점에 최신 버전인 `9.3.9.v20160517`를 사용하자.

먼저 새로운 `leiningen` 프로젝트를 만들자.

```bash
lein new webapp
```

새 프로젝트에서 `project.clj` 파일을 열어 Jetty 라이브러리 디팬던시를 추가하자.
```clojure
(defproject webapp "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.8.0"]
                 [org.eclipse.jetty/jetty-server "9.3.9.v20160517"]]
  :main webapp.core)
```

`lein run` 커맨드로 편리하게 실행하기 위해 `:main webapp.core`를 추가했다. 이제 `leiningen` 템플릿이 생성해준 `core.clj`에 `-main`에 Jetty 핸들러와 서버를 실행하는 코드를 작성하자. 예제는 Jetty 홈페이지에 있는[ Server 예제](http://www.eclipse.org/jetty/documentation/9.3.9.v20160517/quick-start-configure.html#_jetty_ioc_xml_format)와 [Handler 설명](http://www.eclipse.org/jetty/documentation/9.3.9.v20160517/jetty-handlers.html#handler-api)을 참고해서 아래와 같이 작성했다.

```clojure
(ns webapp.core
  (:import [org.eclipse.jetty.server Server ServerConnector]
           org.eclipse.jetty.server.handler.AbstractHandler))

(def handler
  (proxy [AbstractHandler] []
    (handle [target base-request request response]
      (with-open [writer (.getWriter response)]
        (.print writer "Hello World")))))

(defn -main [& args]
  (let [^Server server (Server.)
        ^ServerConnector connector (doto (ServerConnector. server)
                                     (.setPort 8080))]
    (doto server
      (.addConnector connector)
      (.setHandler handler)
      (.start)
      (.join))))
```

파일을 저장하고 쉘에서 `lein run`을 해보면 `8080` 포트로 서버가 실행되었다는 로그가 다음과 같이 나온다.

```bash
2016-06-22 18:42:52.821:INFO::main: Logging initialized @759ms
2016-06-22 18:42:52.857:INFO:oejs.Server:main: jetty-9.3.9.v20160517
2016-06-22 18:42:52.903:INFO:oejs.AbstractConnector:main: Started ServerConnector@f68f0dc{HTTP/1.1,[http/1.1]}{0.0.0.0:8080}
2016-06-22 18:42:52.903:INFO:oejs.Server:main: Started @842ms
```

브라우저에서 주소 창에 [localhost:8080](http://localhost:8080)를 넣어보면 `Hello World` 텍스트가 출력되는 것을 볼 수 있다.

위 예제를 다시 활용 할 수 있도록 공통으로 사용할 수 있는 부분을 다른 네임스페이스로 분리해보자.

`webapp.server` 네임스페이스에 공통으로 사용할 수 있는 코드를 옮기자.

```clojure
(ns webapp.server
  (:import [org.eclipse.jetty.server Server ServerConnector]
           org.eclipse.jetty.server.handler.AbstractHandler))

(defn- new-handler [handler]
  (proxy [AbstractHandler] []
    (handle [target base-request request response]
      (with-open [writer (.getWriter response)]
        (.print writer (handler request))))))

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

그리고 `webap.core`는 아래와 같이 바꾼다.

```clojure
(ns webapp.core
  (:require [webapp.server :refer [run-jetty]])
  (:import javax.servlet.http.HttpServletRequest))

(defn handler [^HttpServletRequest request]
  (str "Hello World "))

(defn -main [& args]
  (run-jetty handler))
```

이제 웹 요청을 인자로 받아 문자열을 리턴하는 `handler` 함수를 `run-jetty` 함수에 넘겨주면 주어진 핸들러 함수로 서버가 실행된다. 

지금 만든 `handler` 함수는 항상 `200 OK` 응답이 내려간다. 만약 다른 HTTP 상태 코드를 내려주도록 만들어보자. `handler` 함수가 지금은 문자열로 리턴하게 되어 있지만 `:body`와 `:status`라는 키로 본문과 HTTP 상태 코드를 리턴하게 만들고 `run-jetty`에서 `:status` 값을 사용하게 하자.

```clojure
(ns webapp.server
  (:import [org.eclipse.jetty.server Server ServerConnector]
           org.eclipse.jetty.server.handler.AbstractHandler))

(defn- new-handler [handler]
  (proxy [AbstractHandler] []
    (handle [target base-request request response]
      (let [{:keys [status body]} (handler request)]
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

그리고 `webapp.core`에 `handler` 함수는 아래와 같이 리턴하도록 고친다.

```clojure
(ns webapp.core
  (:require [webapp.server :refer [run-jetty]])
  (:import javax.servlet.http.HttpServletRequest))

(defn handler [^HttpServletRequest request]
  {:status 201 :body "Hello World"})

(defn -main [& args]
  (run-jetty handler))
```

이제 `handler` 함수에서 상태 코드를 내려 줄 수 있게 되었다.

`handler` 함수에 `HttpServletRequest` 의존성은 거슬리는 부분이다. `HttpServletRequest`를 인자로 받는 대신 `HttpServletRequest` 내용을 가진 맵을 받으면 `HttpServletRequest`의 의존도가 없어져 사용하기 더 편리할 것 같다. 예제에서는 `HttpServletRequest` 메서드 중 요청 매서드와 경로만 매핑해서 맵으로 받아보자.

```clojure
(defn- build-request-map [^HttpServletRequest request]
  {:method (keyword (lower-case (.getMethod request)))
   :path (str (.getPathInfo request) "?"
           (.getQueryString request))})

(defn- new-handler [handler]
  (proxy [AbstractHandler] []
    (handle [target base-request request response]
      (let [request-map (build-request-map request)
            {:keys [status body]} (handler request-map)]
        (.setStatus response status)
        (with-open [writer (.getWriter response)]
          (.print writer body))))))
```

메서드와 경로는 각각 `:method`와 `:path` 키에 넣었다. `:method`는 활용하기 좋게 키워드 형식으로 바꿨고 `:path`에는 쿼리스트링까지 추가했다.

이제 `Hello World` 대신 요청에 있는 메서드와 경로를 출력해보자.

```clojure
(defn handler [request]
  {:status 201 :body (str "method: " (:method request)
                       " path: " (:path request))})
```


