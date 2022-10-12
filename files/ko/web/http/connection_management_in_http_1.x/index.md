---
title: HTTP/1.x의 커넥션 관리
slug: Web/HTTP/Connection_management_in_HTTP_1.x
translation_of: Web/HTTP/Connection_management_in_HTTP_1.x
---
{{HTTPSidebar}}

커넥션 관리는 HTTP의 주요 주제입니다: 대규모로 커넥션을 열고 유지하는 것은 웹 사이트 혹은 웹 애플리케이션의 성능에 많은 영향을 줍니다. HTTP/1.x에는 몇 가지 모델이 존재합니다: 단기 커넥션, 영속적인 커넥션, 그리고 _HTTP 파이프라이닝._

HTTP는 클라이언트와 서버 사이의 커넥션을 제공하는 TCP를 전송프로토콜로 주로 이용합니다. 초기에는, HTTP는 이런 커넥션들을 다루기 위해 단일 모델을 제공했습니다. 요청이 보내져야 할 때마다 커넥션들은 매번 새롭게 생성되었고 응답이 도착한 이후에 연결을 닫는 형태로 단기로만 유지되었습니다.

각각의 TCP 연결을 여는 것은 자원을 소비하기 때문에 이러한 단순한 모델은 선천적으로 성능상의 제약을 발생시킵니다. 몇몇 메시지들은 클라이언트와 서버 사이에서 교환되어야만 하며, 네트워크의 지연과 대역폭은 요청이 전송되어야 할 때마다 성능에 영향을 줍니다. 현대의 웹 페이지들은 필요로 하는 정보를 제공하기 위해 많은 요청(12개 혹은 그 이상)을 필요로 하므로, 이런 초창기 모델이 비효율적인 것은 자명합니다.

HTTP/1.1에서 두 가지 모델이 추가되었습니다. 영속적인 커넥션 모델은 연속적인 요청 사이에 커넥션을 유지하여 새 커넥션을 여는데 필요한 시간을 줄입니다. HTTP 파이프라이닝은 한 단계 더 나아가, 응답조차도 기다리지 않고 연속적인 요청을 보내서 네트워크 지연을 더욱 줄입니다.

![Compares the performance of the three HTTP/1.x connection models: short-lived connections, persistent connections, and HTTP pipelining.](https://mdn.mozillademos.org/files/13727/HTTP1_x_Connections.png)

> **참고:** HTTP/2는 커넥션 관리의 몇가지 모델을 더 추가합니다.

명심해야 할 중요한 점은 HTTP 내 커넥션 관리가 [end-to-end](/ko/docs/Web/HTTP/Headers#e2e)가 아닌 [hop-by-hop](/ko/docs/Web/HTTP/Headers#hbh)인 두 개의 연속된 노드 사이의 커넥션에 적용된다는 것입니다. 클라이언트와 첫 번째 프록시 사이의 커넥션 모델은 프록시와 최종 목적 서버(혹은 중간 프록시들) 간의 것과는 다를 수도 있습니다. {{HTTPHeader("Connection")}}와 {{HTTPHeader("Keep-Alive")}}와 같이 커넥션 모델을 정의하는 데 관여하는 HTTP 헤더들은 [hop-by-hop](/ko/docs/Web/HTTP/Headers#hbh) 헤더이며, 중간 노드에 의해 그 값들이 변경될 수 있습니다.

HTTP/1.1 커넥션이 TLS/1.0이나 WebSocket, 혹은 평문 HTTP/2와 같은 다른 프로토콜로 업그레이드 되었다는 점에서 관련된 주제는 HTTP 커넥션 업그레이드의 개념입니다. 이 프로토콜 업그레이드 메커니즘은 다른 곳에서 더 자세히 설명되어 있습니다.

## 단기 커넥션

HTTP 본래의 모델이자 HTTP/1.0의 기본 커넥션은 *단기 커넥션*입니다. 각각의 HTTP 요청은 각각의 커넥션 상에서 실행됩니다. 이는 TCP 핸드 셰이크는 각 HTTP 요청 전에 발생하고, 이들이 직렬화됨을 의미합니다.

TCP 핸드셰이크는 그 자체로 시간을 소모하기는 하지만 TCP 커넥션은 지속적으로 연결되었을 때 부하에 맞춰 더욱 예열되어 더욱 효율적으로 작동합니다. 단기 커넥션들은 TCP의 이러한 효율적인 특성을 사용하지 않게 하며 예열되지 않은 새로운 연결을 통해 지속적으로 전송함으로써 성능이 최적 상태보다 저하됩니다.

이 모델은 HTTP/1.0에서 사용된 기본 모델입니다({{HTTPHeader("Connection")}} 헤더가 존재하지 않거나, 그것의 값이 `close`로 설정된 경우). HTTP/1.1에서는, 이 모델은 {{HTTPHeader("Connection")}} 헤더가 `close` 값으로 설정되어 전송된 경우에만 사용됩니다.

> **참고:** 영속적인 커넥션을 지원하지 않는 매우 낡은 시스템을 다루는 것이 아니라면, 이 모델을 사용하려고 애쓸 필요가 없습니다.

## 영속적인 커넥션

단기 커넥션은 두 가지 결점을 지니고 있습니다: 새로운 연결을 맺는데 드는 시간이 상당하다는 것과, TCP기반 커넥션의 성능은 오직 커넥션이 예열된 상태일 때만 나아진다는 것입니다. 이런 문제를 완화시키기 위해, HTTP/1.1보다도 앞서 영*속적인 커넥션*의 컨셉이 만들어졌습니다. 이는 *keep-alive 커넥션*이라고 불리기도 합니다.

영속적인 커넥션은 얼마간 연결을 열어놓고 여러 요청에 재사용함으로써, 새로운 TCP 핸드셰이크를 하는 비용을 아끼고, TCP의 성능 향상 기능을 활용할 수 있습니다. 커넥션은 영원히 열려있는지 않으며, 유휴 커넥션들은 얼마 후에 닫힙니다(서버는 {{HTTPHeader("Keep-Alive")}} 헤더를 사용해서 연결이 최소한 얼마나 열려있어야 할지를 설정할 수 있습니다).

물론 영속적인 커넥션도 단점을 가지고 있습니다. 유휴 상태일때에도 서버 리소스를 소비하며, 과부하 상태에서는 {{glossary("DoS attack", "DoS attacks")}}을 당할 수 있습니다. 이런 경우에는 커넥션이 유휴 상태가 되자마자 닫히는 비영속적 커넥션(non-persistent connections)을 사용하는 것이 더 나은 성능을 보일 수 있습니다.

HTTP/1.0 커넥션은 기본적으로 영속적이지 않습니다. {{HTTPHeader("Connection")}}를 `close`가 아닌 다른 것으로, 일반적으로 `retry-after`로 설정하면 영속적으로 동작하게 될 겁니다.

반면, HTTP/1.1에서는 기본적으로 영속적이며 헤더도 필요하지 않습니다(그러나 HTTP/1.0으로 동작하는 경우(fallback)에 대비해서 종종 추가하기도 합니다.).

## HTTP 파이프라이닝

> **참고:** HTTP 파이프라이닝은 모던 브라우저에서 기본적으로 활성화되어있지 않습니다:
>
> - 버그가 있는 [프록시](https://en.wikipedia.org/wiki/Proxy_server)들이 여전히 많은데, 이들은 웹 개발자들이 쉽게 예상하거나 분석하기 힘든 이상하고 오류가 있는 동작을 야기합니다.
> - 파이프라이닝은 정확히 구현해내기 복잡합니다: 전송 중인 리소스의 크기, 사용될 효과적인 [RTT](https://en.wikipedia.org/wiki/Round-trip_delay_time), 그리고 효과적인 대역폭은 파이프라인이 제공하는 성능 향상에 직접적으로 영향을 미칩니다. 이런 내용을 모른다면, 중요한 메시지가 덜 중요한 메시지에 밀려 지연될 수 있습니다. 중요성에 대한 생각은 페이지 레이아웃 중에도 진전됩니다. 그러므로 파이프라이닝은 대부분의 경우 미미한 수준의 향상만을 가져다 줍니다.
> - 파이프라이닝은 [HOL](https://en.wikipedia.org/wiki/Head-of-line_blocking) 문제에 영향을 받습니다.
>
> 이런 이유들로, 파이프라이닝은 더 나은 알고리즘인 멀티플렉싱으로 대체되었는데, 이는 HTTP/2에서 사용됩니다.

기본적으로, [HTTP](/en/HTTP "en/HTTP") 요청은 순차적입니다. 현재의 요청에 대한 응답을 받고 나서야 다음 요청을 실시합니다. 네트워크 지연과 대역폭 제한에 걸려 다음 요청을 보내는 데까지 상당한 딜레이가 발생할 수 있습니다.

파이프라이닝이란 같은 영속적인 커넥션을 통해서, 응답을 기다리지 않고 요청을 연속적으로 보내는 기능입니다. 이것은 커넥션의 지연를 회피하고자 하는 방법입니다. 이론적으로는, 두 개의 HTTP 요청을 하나의 TCP 메시지 안에 채워서(be packed) 성능을 더 향상시킬 수 있습니다. HTTP 요청의 사이즈는 지속적으로 커져왔지만, 일반적인 [MSS](https://en.wikipedia.org/wiki/Maximum_segment_size)(최대 세그먼트 크기)는 몇 개의 간단한 요청을 포함하기에는 충분히 여유있습니다.

모든 종류의 HTTP 요청이 파이프라인으로 처리될 수 있는 것은 아닙니다: {{HTTPMethod("GET")}}, {{HTTPMethod("HEAD")}}, {{HTTPMethod("PUT")}} 그리고 {{HTTPMethod("DELETE")}} 메서드같은 {{glossary("idempotent")}} 메서드만 가능합니다. 실패가 발생한 경우에는 단순히 파이프라인 컨텐츠를 다시 반복하면 됩니다.

오늘날, 모든 HTTP/1.1 호환 프록시와 서버들은 파이프라이닝을 지원해야 하지만, 실제로는 많은 프록시와 서버들은 제한을 가지고 있습니다. 모던 브라우저가 이 기능을 기본적으로 활성화하지 않는 이유입니다.

## 도메인 샤딩

> **참고:** 매우 명확하고 당면해있는 요구사항이 아니라면, 이 제외된(deprecated) 기술을 사용하지 마십시오. 대신 HTTP/2로 전환하시기 바랍니다. HTTP/2에서 도메인 샤딩은 더 이상 유용하지 않습니다. HTTP/2 커넥션은 우선 순위가 없는 병렬 요청들을 매우 잘 다룹니다. 도메인 샤딩은 성능 측면에서도 좋지 못합니다. 대부분의 HTTP/2 구현체는 우발적으로 일어나는 도메인 샤딩을 되돌리기 위해 [커넥션 합치기(connection coalescing)](https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/)라고 불리는 기술을 사용합니다.

요청 사이에 실제 정렬이 없음에도 HTTP/1.x 커넥션이 요청을 직렬화함으로써, 대역폭이 충분히 큰 경우가 아니고는 효율적이지 못합니다. 이런 단점을 피하기 위해, 브라우저들은 각 도메인에 대한 몇 개의 커넥션을 맺고 병렬로 요청을 보냅니다. 기본값은 한때 2개 혹은 3개였지만, 지금은 이것이 증가하여 일반적으로 병력 커넥션은 6개입니다. 만약 이것보다 많이 시도한다면 서버 측의 [DoS](/ko/docs/Glossary/DOS_attack) 보호 동작을 야기할 위험이 있습니다.

서버가 더 빠른 웹 사이트나 애플리케이션 반응을 원한다면, 서버가 더 많은 커넥션을 열도록 강제할 수 있습니다. 예를 들어, `www.example.com` 라는 하나의 도메인에서 모든 리소스를 가져오는 대신, `www1.example.com`, `www2.example.com`, `www3.example.com`와 같이 몇 개의 도메인으로 분할할 수 있습니다. 이런 각각의 도메인들은 _동일한_ 서버로 연결되고, 브라우저는 그런 각각의 도메인마다 6개의 커넥션을 맺을 것입니다(예제에서는 총 18개로 늘어납니다). 이러한 기법을 *도메인 샤딩*이라고 부릅니다.

![](https://mdn.mozillademos.org/files/13783/HTTPSharding.png)

## 결론

개선된 커넥션 관리는 HTTP 성능을 상당한 수준만큼 향상시킬 수 있습니다. 영속적인 커넥션을 사용하는 HTTP/1.1 혹은 HTTP/1.0는 - 적어도 그 커넥션이 유휴 상태가 될 때까지 - 최상의 성능을 이끌어냅니다. 그러나, 파이프라이닝의 실패로 더 나은 커넥션 관리 모델이 고안되었고, 이는 HTTP/2에 포함되었습니다.