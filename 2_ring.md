# Ring 라이브러리 - 정리중

지난 시간에 자바 Jetty 라이브러리로 `Hell World` 웹 어플리케이션을 만들어 봤다. 지난 시간에 만들었던 `run-jetty` 함수와 `handler` 함수의 요청 맵과 응답 맵 형식은 이미 Ring 라이브러리에 구현되어 있는 기능이다. Ring 라이브러리는 요청을 처리하는 핸들러의 요청 맵과 응답 맵의 형식(예를 들면 클라이언트 IP 주소는 `:remote-addr` 키에 들어있다)을 정의 한 Spec과 이 맵을 처리하는 라이브러리들로 구성되어 있다. 그리고 편의를 위해 핸들러 함수를 Jetty로 실행 할 수 있는 어댑터도 제공한다.

## Ring
- SPEC
  - 요청/응답 맵에 어떤 키를 사용하는지 설명되어 있다. 그리고 핸들러, 미들웨어, 어댑터에 역할에 대해 설명하고 있다.
- ring-core
  - 요청 맵과 응답 맵을 다루는 함수들과 일반적으로 많이 사용하는 미들웨어를 제공한다.
- ring-devel
  - 개발 환경에서 사용할 수 있는 Reload 미들웨어나 Stackstrace 미들웨어 같은 것을 제공한다.
- ring-jetty-adapter
  - Jetty로 핸들러를 실행해 볼 수 있는 `run-jetty` 함수를 제공한다.
- ring-servlet
  - 자바 요청/응답 서블릿 객체를 클로저 맵으로 바꿔주는 함수를 제공한다.

Ring은 SPEC 문서와 4개 라이브러리가 있다. 지난 시간에 만든 `Hello World`를 Ring 라이브러리로 만들어 보자. 우리는 `run-jetty`만 필요하기 때문에 `ring-jetty-adapter` 라이브러리만 사용할 것이다.

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

지난번 작성한 코드랑 거의 비슷하다. 다른 점은 `handler` 함수의 파라미터인 `request` 맵에 들어있는 키 이름과 경로 형식이다. 특히 경로는 `:uri`와 `:query-string`이 분리되어 들어온다. 또 다른 점은 `run-jetty` 함수에 두번째 인자로 옵션을 받아 포트 번호 같은 것을 넘겨줬다.


