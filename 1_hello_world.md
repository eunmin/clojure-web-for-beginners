# Hello World 웹 어플리케이션

클로저도 다른 언어들과 마찬가지로 웹 개발을 도와주는 많은 라이브러리가 있다. 이런 라이브러리를 이용하면 많은 코드를 작성하지 않아도 쉽게 웹 어플리케이션을 만들 수 있다. 이런 라이브러리들은 차차 소개하기로 하고 이번 장에서는 자바 라이브러리를 사용해 `Hello World`를 표시해주는 웹 어플리케이션을 만들어볼 것이다. Tomcat, Jetty, Undertow, Grizzly 등 다양한 웹 어플리케이션 서버 라이브러리가 있지만 여기서는 Jetty 라이브러리를 사용할 것이다. 하지만 기본적인 방식은 다른 라이브러리도 마찬가지다.

Jetty 라이브러리는 [홈페이지](http://www.eclipse.org/jetty/)에 문서와 다운로드에 대한 정보가 잘 정리되어 있다. 이 문서를 작성하는 때 최신 버전인 `9.3.9.v20160517`를 사용하겠다. 

먼저 새로운 `leiningen` 프로젝트를 만들자.

```bash
lein new webapp
```

새 프로젝트에서 `project.clj` 파일을 열어 Jetty 라이브러리 메이븐 의존성을 추가하자.

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

그리고 `lein run` 커맨드로 어플리케이션을 편리하게 실행하기 위해 `:main webapp.core`를 추가했다.
이제 `leiningen` 템플릿이 만들어준 `core.clj` 파일에 `-main` 함수를 만들자.
여기에 Jetty 홈페이지에 있는[ Server 예제](http://www.eclipse.org/jetty/documentation/9.3.9.v20160517/quick-start-configure.html#_jetty_ioc_xml_format)와 [Handler 설명](http://www.eclipse.org/jetty/documentation/9.3.9.v20160517/jetty-handlers.html#handler-api)을 참고해서 간단한 Handler와 서버를 실행하는 코드를 아래와 같이 작성하자.

```clojure
(ns webapp.core
  (:import [org.eclipse.jetty.server Server ServerConnector]
           org.eclipse.jetty.server.handler.AbstractHandler))

(def handler
  (proxy [AbstractHandler] []
    (handle [target base-request request ^HttpServletResponse response]
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

파일을 저장하고 쉘에서 `lein run`을 입력하면 `8080` 포트로 서버가 실행되었다는 로그가 나온다.

```bash
2016-06-22 18:42:52.821:INFO::main: Logging initialized @759ms
2016-06-22 18:42:52.857:INFO:oejs.Server:main: jetty-9.3.9.v20160517
2016-06-22 18:42:52.903:INFO:oejs.AbstractConnector:main: Started ServerConnector@f68f0dc{HTTP/1.1,[http/1.1]}{0.0.0.0:8080}
2016-06-22 18:42:52.903:INFO:oejs.Server:main: Started @842ms
```

브라우저에서 주소 창에 [localhost:8080](http://localhost:8080)를 입력하면 `Hello World` 텍스트가 출력되는 것을 볼 수 있다.
위에서 만든 예제를 재사용 할 수 있도록 `Hello World` 부분을 빼고 다른 네임스페이스에 함수로 옮겨보자. 
네임스페이스와 함수 이름은 `webapp.server/run-jetty`가 좋을 것 같다. 옮긴 모습은 아래와 같다.

```clojure
(ns webapp.server
  (:import [org.eclipse.jetty.server Server ServerConnector]
           org.eclipse.jetty.server.handler.AbstractHandler))

(defn- new-handler [handler]
  (proxy [AbstractHandler] []
    (handle [target base-request request ^HttpServletResponse response]
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
    (handle [target base-request request ^HttpServletResponse response]
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
    (handle [target base-request request ^HttpServletResponse response]
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

마지막으로 응답을 HTML 형식으로 내려줘보자. 응답을 HTML으로 내려주려면 응답 해더에 Content-type을 `text/html`로 설정해야 한다. 그래서 응답 맵을 아래와 같이 내려 줄 수 있으면 좋을 것 같다.


```clojure
(defn handler [request]
  {:status 201
   :headers {"Content-Type" "text/html;charset=utf-8"}
   :body (str
           "<p>method: " (:method request) "</p>"
           "<p>path: " (:path request) "</p>")})
```

그리고 Jetty 핸들러를 만드는 부분에서 응답에 `:headers`에 `Content-Type`을 가져와 `HttpServletResponse`에 `setContentType`을 불러준다.

```clojure
(defn- new-handler [handler]
  (proxy [AbstractHandler] []
    (handle [target base-request request ^HttpServletResponse response]
      (let [request-map (build-request-map request)
            {:keys [status headers body]} (handler request-map)
            content-type (get headers "Content-Type")]
        (.setStatus response status)
        (.setContentType response content-type)
        (with-open [writer (.getWriter response)]
          (.print writer body))))))
```

이제 첫번째 웹 어플리케이션 예제는 끝났다. 정리를 해보면 Jetty 라이브러리로 핸들러를 만들어 서버를 시작했다. 그리고 Jetty에 의존적인 부분들을 `webapp.server/run-jetty` 함수로 분리했다. 분리하면서 `webapp.core`에는 `handler`라는 함수를 만들었는데 이 함수는 웹 요청을 맵 형식의 파라미터로 받고 맵 형식의 응답을 주면 되는 함수로 다른 라이브러리에 의존적이지 않는 함수로 작성했다. 물론 예제에서는 모든 요청의 값을 담지 못하고 응답에서도 역시 모든 응답 헤더를 설정해주지 않았지만 추가적으로 작업하면 되는 일이다. 실제로 위에서 작성한 `run-jetty`는 Ring 이라는 라이브러리가 제공해주는 함수이고 우리는 직접 작업하지 않고 다음 시간에 Ring 라이브러리를 가지고 작업을 계속 할 것이다.

