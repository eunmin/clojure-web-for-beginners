# Hello World 웹 어플리케이션

클로저는 자바 라이브러리를 사용할 수 있기 때문에 Jetty 라이브러리를 이용해서 Hello World 문자열을 표시하는 웹 어플리케이션을 만들어보자.

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

