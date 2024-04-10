---
title: 캐시 때문에 CORS 오류가 발생할 수 있다고?
date: 2024-03-31 +0900
categories: [HTTP]
tags: [HTTP, Cache, CORS]
img_path: /assets/img/posts/2024-03-31
image: cors-thumbnail.png
render_with_liquid: false
---

CORS 오류는 잊을만 하면 나타나서 개발자들을 괴롭힙니다. 그런데 캐시가 CORS 오류를 일으킬 수 있다는 사실을 알고 계셨나요? 이번에는 브라우저 캐시가 어떻게 CORS 오류와 관련이 있고, 어떻게 해결할 수 있는지 예시와 함께 공유 드리려고 합니다.

---

## 웹에서의 캐시
웹 서비스는 여러 종류의 리소스(HTML, JS, 이미지 등)를 서버로부터 받아와서 사용자(클라이언트)에게 보여줍니다. 그런데 같은 리소스를 자주 요청하는 경우, 매번 서버에서 새로 리소스를 가져온다면 서버와의 긴 통신 시간으로 인해 로딩 시간이 길어지고 잦은 요청으로 인해 서버에 부담이 갈 수 있습니다. 이 때 가까운 곳에 리소스를 저장해두고 사용하면 훨씬 빠르게 리소스를 가져올 수 있기 때문에 로딩 시간이 짧아지고 서버 부담도 완화할 수 있습니다. 이러한 방식을 **캐싱**이라고 합니다.

웹에서는 주로 [HTTP 캐시](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)를 이용해 **서버의 응답**을 캐싱합니다. 리소스만 캐싱하는 것이 아니라 서버의 응답 자체를 캐싱한다는 것이 중요한데요. 서버 응답에는 상태 코드, 헤더, 본문이 포함됩니다. 이 때 헤더에 존재하는 `Access-Control-Allow-Origin` 항목까지 캐싱되기 때문에 CORS가 발생할 수 있습니다.

![서버 응답](http_response.png)
_출처: https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages#http_responses_

## 캐시와 CORS
`Access-Control-Allow-Origin` 항목이 캐싱되는 것이 어떻게 CORS를 발생시킬 수 있을까요? 이해를 돕기 위해 CORS 오류가 발생하는 예시 상황을 만들어 보겠습니다. 다음과 같은 API 서버가 있다고 가정해 봅시다.

```javascript
const express = require('express');
const path = require('path');
const PORT = 4000;

const app = express();

app.get('/', (req, res) => {
  res.send('hello world');
});

app.get('/image', (req, res) => {
  res
    .setHeader('Cache-Control', 'public, max-age=3600')
    .setHeader('Access-Control-Allow-Origin', req.get('origin'))
    .sendFile(path.join(__dirname, './img/palwol.jpeg'));
});

app.listen(PORT, () => {
  console.log(`Server is listening on port ${PORT}...`);
});
```

이 서버에 `GET /image` 경로로 이미지를 요청하면 3600초간 이미지가 캐싱되도록 설정되어 있습니다. 또한, `Access-Control-Allow-Origin` 값으로 요청의 origin 값을 사용합니다. 즉 `http://localhost:3000`에서 요청을 보내면 `Access-Control-Allow-Origin`는 `http://localhost:3000`이 되고, `http://localhost:3001`에서 요청을 보내면 `Access-Control-Allow-Origin`는 `http://localhost:3001`이 됩니다.

> 서버에서 `Access-Control-Allow-Origin` 값으로 와일드카드(*) 대신 요청의 origin을 사용하는 방식은 주로 인증 정보를 포함한 요청을 제한 없이 받아야 하는 경우에 사용합니다. [인증 정보를 포함한 요청에는 `Access-Control-Allow-Origin` 값으로 와일드카드를 사용할 수 없기 때문](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/Errors/CORSNotSupportingCredentials)입니다.
{: .prompt-info}

그럼 이 서버에 서로 다른 도메인을 가진 클라이언트가 각각 `GET /image` 요청을 하면 어떻게 될까요? `http://localhost:3000`과 `http://localhost:3001`이라는 두 클라이언트에서 서버에 `GET /image` 요청을 보내 보았습니다.  
그 결과, `http://localhost:3000`에서 이미지를 요청했을 때는 정상적인 응답이 왔지만 그 다음 `http://localhost:3001`에서 이미지를 요청했을 때는 다음과 같은 CORS 오류가 발생했습니다.

![CORS 에러](cors-error-prod.png)

오류 메세지 내용을 읽어보면 다음과 같습니다.

> `Access-Control-Allow-Origin` 값이 `http://localhost:3000`이야. 너는 origin이 `http://localhost:3001`이라서 CORS 정책에 의해 `http://localhost:4000/image(이미지 서버)`로의 접근이 막혔어.

마치 서버에서 `Access-Control-Allow-Origin` 값을 `http://localhost:3000`으로 고정해 둔 듯한 오류가 발생했습니다. 이는 `http://localhost:3000`에서 이미지를 요청했을 때의 응답이 캐싱되었기 때문입니다.

`http://localhost:3000`에서 이미지를 요청했을 때의 응답 헤더를 살펴보면 다음과 같습니다.

![localhost:3000 응답 헤더](3000_response_header.png)

`Access-Control-Allow-Origin` 값이 `http://localhost:3000`으로 설정되어 있는 것을 확인할 수 있습니다. 앞서 말했다시피 HTTP 캐싱은 리소스뿐이 아니라 서버 응답 자체를 캐싱합니다. 즉 응답 헤더의 `Access-Control-Allow-Origin` 값도 같이 캐싱되게 됩니다.

이 상태에서 같은 서버에 같은 리소스를 요청하면 브라우저는 이미 캐싱되어 있는 응답을 사용하려고 합니다. 실제로 `http://localhost:3000`에서 동일한 이미지를 한 번 더 요청하면 이미 캐싱된 응답을 사용하는 것을 확인할 수 있습니다.

![캐싱된 이미지](image_cache.png)

따라서 `http://localhost:3000`에서 요청한 응답이 캐싱되어 있는 상태에서 `http://localhost:3001`에서 동일한 이미지를 요청하면 캐싱된 응답의 `Access-Control-Allow-Origin` 값이 요청 origin과 맞지 않아 CORS가 발생하게 되는 것입니다.

## 캐시로 인한 CORS 오류 방지하기
그럼 이렇게 캐시로 인해 CORS 오류가 발생하는 경우를 어떻게 방지할 수 있을까요? 여러 방법이 있을 수 있겠지만, 여기서는 서버에서의 해결 방법과 클라이언트에서의 해결방법을 각각 한 가지씩 소개하려고 합니다.

### `Vary` 헤더
먼저 서버에서의 해결법으로, [Vary 헤더](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary)를 사용하는 방법입니다. Vary 헤더는 HTTP 응답 헤더로, 어떤 요청 헤더를 기반으로 응답을 캐시할 것인지를 알려주는 헤더입니다.

예를 들어 요청의 `User-Agent` 헤더를 기반으로 응답을 캐싱하고 싶다면 응답 헤더에 `Vary: User-Agent`라고 명시해주면 됩니다. 그러면 `User-Agent`가 동일할 때에만, 즉 동일한 브라우저와 동일한 기기로 요청을 할 때에만 캐시된 응답을 사용하게 됩니다.

이를 위의 CORS 오류 발생 상황에 적용해보면, 응답의 `Vary` 헤더 값으로 origin을 사용하면 요청의 origin에 따라 캐시된 응답 사용 여부를 결정하기 때문에 CORS 오류를 방지할 수 있게 됩니다.

```javascript
/* ... */
app.get('/image', (req, res) => {
  res
    .setHeader('Cache-Control', 'public, max-age=3600')
    .setHeader('Access-Control-Allow-Origin', req.get('origin'))
    .setHeader('Vary', 'origin') // Vary 헤더 추가
    .sendFile(path.join(__dirname, './img/palwol.jpeg'));
});
/* ... */
```

### `Cache-Control: no-cache` 헤더
다음은 클라이언트에서의 해결 방법으로, 요청 헤더에 `Cache-Control: no-cache`를 추가하는 방법입니다. 사실 서버 응답 헤더에도 `Cache-Control: no-cache`를 사용할 수 있지만, 서버 응답을 수정할 수 없는 경우 클라이언트 요청 헤더만으로 CORS 문제를 해결할 수 있기 때문에 클라이언트 해결 방안으로 소개 드렸습니다.

참고로 `Cache-Control`의 값으로 `no-store`도 사용할 수 있는데요. `no-cache`와 `no-store`의 차이는 다음과 같습니다.
- `no-cache`: 응답이 캐시될 수 있지만, 응답을 재사용할 때 반드시 origin 서버에 재검증을 해야 합니다.
- `no-store`: 응답이 캐시되지 않습니다. 서버에서 응답이 캐시되도록 설정한 경우에도 캐시되지 않습니다.

즉, `no-cache`를 사용하면 응답을 캐싱하여 요청 부담을 줄일 수 있지만 캐싱된 응답을 매번 재검증하여 사용하기 때문에 위와 같은 CORS 오류 상황에서 새로운 `Access-Control-Allow-Origin` 값을 사용하여 CORS 오류를 방지할 수 있습니다. `no-store`를 사용해도 캐시가 되지 않기 때문에 CORS 오류는 방지할 수 있겠지만, 그렇게 되면 캐싱의 이점 역시 포기하게 되기 때문에 `no-cache`를 사용하는 것이 더 적절해 보입니다.

---

## 마치며
사실 위의 CORS 예시는 실제로 저희 팀에서 겪은 상황이었는데요. 캐시가 CORS 오류를 발생시킬 수도 있다는 사실을 모르는 상태에서 마주하니 굉장히 당황스러운 경험이었습니다.😅 그래도 이 경험으로 인해 캐시를 무분별하게 사용해선 안된다는 교훈을 얻게 되었습니다. 혹시라도 비슷한 문제로 고민하고 계신 분이 있다면 이 포스트가 도움이 되기를 바랍니다.

의견이나 오류 제보는 댓글 부탁드립니다!
감사합니다.