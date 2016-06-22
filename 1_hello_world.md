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

그리고 `webap.core`는 `Hello World` 문자열을 리턴하는 함수를 만들어 `run-jetty`에 파라미터로 넘기게 수정하자.

```clojure
(ns webapp.core
  (:require [webapp.server :refer [run-jetty]])
  (:import javax.servlet.http.HttpServletRequest))

(defn handler [^HttpServletRequest request]
  (str "Hello World "))

(defn -main [& args]
  (run-jetty handler))
```

`Ctrl+C`로 실행한 서버를 중지하고 다시 `lein run`으로 실행해서 잘 동작하는지 확인해보자.

## 상태 코드

위에서 만든 `handler` 함수는 문자열을 리턴하기 때문에 항상 `200` 상태 코드로 응답이 내려간다. 웹 어플리케이션은 다양한 상태 코드를 내려줄 수 있어야 하기 때문에 `handler` 함수가 상태 코드와 문자열을 함께 내려주도록 하자. 리턴 값을 맵으로 하고 상태 코드는 `:status` 키에 넣고 본문은 `:body` 키에 넣어 리턴하도록 하자.

```clojure
(ns webapp.core
  (:require [webapp.server :refer [run-jetty]])
  (:import javax.servlet.http.HttpServletRequest))

(defn handler [^HttpServletRequest request]
  {:status 201 :body "Hello World"})

(defn -main [& args]
  (run-jetty handler))
```

이제 `webapp.server/new-handler` 함수에서 `handler` 함수의 리턴 값 중 `:status` 키 값을 가져와 `HttpServletResponse`의 `setStatus` 메서드로 설정해 주자.

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

이제 `handler` 함수에서 상태 코드를 내려 줄 수 있게 되었다.

## 요청

위에서 작성한 `handler` 함수의 인자는 `HttpServletRequest` 타입이다. 따라서 이 함수를 사용하려면 `HttpServletRequest` 타입 힌트를 주는 것이 좋다. 그래서 `HttpServletRequest` 의존성을 가지게 된다. 이 의존성을 해결하려면 함수 파라미터를 `HttpServletRequest` 타입이 아니고 일반 맵 타입으로 받으면 된다. 그리고 그 맵에는 요청에 대한 정보가 키로 들어 있으면 사용하기 편리할 것이다. 예제에서 요청에 모든 값을 담는 것은 의미가 없기 때문에 요청 메서드와 경로를 각각 `:method`와 `:path` 키에 담에 `handler` 함수에 전달하도록 해보자. 

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

`:method` 값은 활용하기 좋게 키워드 형식으로 바꿨고 `:path` 값은 경로에 쿼리스트링을 추가했다.

요청 맵 정보를 확인하기 위해 `Hello World` 대신 요청에 있는 메서드와 경로를 출력해보자.

```clojure
(defn handler [request]
  {:status 201 :body (str "method: " (:method request)
                       " path: " (:path request))})
```

## 응답 헤더

마지막으로 응답을 HTML 형식으로 내려줘보자. 응답을 HTML으로 내려주려면 리턴 값을 일반 문자열 대신 HTML 태그가 포함된 문자열로 내려줘야한다. 그리고 응답 해더에 Content-type을 `text/html`로 설정해야 한다. 지금은 응답 헤더를 내려줄 방법이 없지만 상태코드를 `:status` 키에 내려준 것과 같이 `:headers` 키를 만들어 다음과 같이 응답 헤더를 내려주면 좋을 것 같다.

```clojure
(defn handler [request]
  {:status 201
   :headers {"Content-Type" "text/html;charset=utf-8"}
   :body (str
           "<p>method: " (:method request) "</p>"
           "<p>path: " (:path request) "</p>")})
```

그리고 `webapp.server/new-handler` 함수에서 `handler`의 리턴 값(= 응답 맵)에 있는 `:headers` 키 값 중 `Content-Type`을 가져와 `HttpServletResponse`에 `setContentType`에 설정해 준다.

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

어플리케이션을 종료하고 `lein run`으로 재시작하고 브라우저에서 확인해보면 `html`형식으로 표시되는 것을 볼 수 있다. 

## 정리

정리를 해보면 Jetty 라이브러리로 핸들러를 만들어 `Hello World`를 출력해 줬다. 그리고 재사용을 위해 Jetty에 의존적인 부분들을 `webapp.server/run-jetty` 함수로 분리했다. 분리하면서 `webapp.core`는 요청 맵을 인자로 받고 응답 맵을 리턴 값으로 주는 `handler`라는 함수를 만들었다. 이 함수는 다른 라이브러리에 의존적이지 않은 함수다. 예제에서는 요청 맵에 `:method`와 `:path`만 추가해줬고 응답 헤더는 `Content-Type`만 처리할 수 있지만 더 작업을 하면 다른 정보들도 다룰 수 있다. 다음 시간에는 우리가 위해서 한 작업을 미리 해둔 Ring 라이브러리를 사용해 웹 어플리케이션을 만들어 보자.

