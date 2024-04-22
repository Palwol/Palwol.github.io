---
title: Github Actions에서 action 버전을 고정해야 하는 이유
date: 2024-04-22 +0900
categories: [github]
tags: [github-actions, dependabot, security]
img_path: /assets/img/posts/2024-04-19
image: GitHub-Actions.png
render_with_liquid: false
---

> 이 포스트는 Github Actions에 대한 설명은 포함하지 않습니다. Github Actions에 대해 자세히 알고 싶으시다면 [Github Actions 설명서](https://docs.github.com/ko/actions)를 참고해 주세요.

팀에서 Github Actions를 이용해 AWS ECS CI/CD를 구축하기 위해 [Deploy to Amazon ECS](https://docs.github.com/en/actions/deployment/deploying-to-your-cloud-provider/deploying-to-amazon-elastic-container-service#creating-the-workflow) 문서를 참고하던 도중, Github Actions workflow 예시 주석에서 다음과 같은 내용을 마주했습니다.

```yaml
# GitHub recommends pinning actions to a commit SHA.
# To get a newer version, you will need to update the SHA.
# You can also reference a tag or branch, but the action may change without warning.
```
해석하면 다음과 같습니다.
- Github는 action(의 버전)들을 commit SHA로 고정하기를 추천합니다.
- action의 새로운 버전을 사용하려면 SHA를 업데이트 해야 합니다.
- SHA 대신 tag나 branch를 사용할 수도 있지만, 그럴 경우 action이 경고 없이 변경될 수 있습니다.

action 버전을 commit SHA로 고정하라니 이게 무슨 소리인가 싶어서 찾아보니 중요한 보안 이슈인 **공급망 공격(Supply Chain Attack)**과 관련된 내용이었습니다.

이번 포스팅에서는 Node.js 개발자인 RAFAEL GONZAGA의 [Why you should pin your GitHub Actions by commit-hash](https://blog.rafaelgss.dev/why-you-should-pin-actions-by-commit-hash) 포스팅 내용을 중심으로, Github Actions에서 action 버전을 관리하는 방식이 공급망 공격과 어떻게 관련이 있는지에 대해 공유 드리려고 합니다.

## 공급망 공격(Supply Chain Attack)
본론에 앞서, 공급망 공격이 무엇일까요? 공급망 공격(Supply Chain Attack)이란 사용자들이 이미 신뢰하며 사용 중인 완성된 소프트웨어나 라이브러리에 악성 코드를 심거나 공급망을 다른 곳으로 속여서 사용자에게 악영향을 끼치는 해킹 공격입니다. 말 그대로 소프트웨어의 공급망을 공격해 해당 소프트웨어를 사용하는 제 3자(소비자)가 피해를 보게 되는 수법입니다.

특정 조직에서 관리하고 있는 소프트웨어는 일반적으로 내부 코드를 공개하지 않고, 수정하려면 관리자 권한이 필요하기 때문에 악성 코드를 심는 것이 쉽지 않습니다. 그러나 오픈소스는 모든 코드가 공개되어 있고 누구나 기여할 수 있기 때문에 상대적으로 공급망 공격에 취약해집니다.

### 예시) XZ Utils 백도어 사건
최근에 발생한 XZ Utils 백도어 사건 역시 공급망 공격으로 발생한 사건이었습니다. 사건을 간략하게 설명하자면, XZ Utils라는 오픈소스 리눅스 압축 라이브러리에 Jia Tan이라는 닉네임을 가진 유저가 꾸준히 PR을 올리는 등 신뢰를 얻어 XZ Utils의 메인테이너가 된 후 백도어가 심어진 버전을 릴리즈한 사건입니다.

다행히 마이크로소프트 개발자인 Andres Freund에 의해 우연히 초기에 백도어가 발견되어 큰 피해 없이 사건이 마무리 되었지만, Jia Tan이 관리자 권한을 가진 메인테이너가 되기 위해 신뢰를 얻는 과정이 무려 3년이나 걸렸기 때문에 아무도 그를 의심하지 않았습니다.

> XZ Utils 백도어 사건은 Outsider님의 [XZ Utils 백도어 사건으로 돌아보는 오픈소스 생태계](https://blog.outsider.ne.kr/1714) 포스팅에 자세히 설명되어 있으니 참고해 주세요!

XZ Utils 사건에서 알 수 있다시피, 오픈소스는 누구나 기여할 수 있다는 자유로움이 장점이자 단점으로 작용합니다. 결국 오픈소스에 크게 의존하고 있는 IT 생태계는 공급망 공격에서 자유로울 수 없고, 대비책을 마련해야 합니다.

## Github Actions에서 action 버전을 명시하는 방법
그렇다면 Github Actions에서 action의 버전을 명시하는 방법이 어떻게 공급망 공격과 관련이 있을까요? 먼저 [Github Marketplace](https://github.com/marketplace?type=actions)에 등록되어 있는 action을 사용할 때 버전을 명시하는 방법에는 다음 3가지 방법이 있습니다.
1. 릴리즈 태그 사용 (`@v3`, `@v3.1.2`)
2. 브랜치 이름 사용 (`@main`)
3. commit SHA 사용 (`@a824008085750b8e136effc585c3cd6082bd575f`)

위 3가지 방법 중 현재 공급망 공격을 완전히 방지할 수 있는 방법은 3번째 방법인 commit SHA를 사용하는 방법 뿐입니다. 그 이유를 각 방법이 어떻게 동작하는지 살펴보면서 알아보겠습니다.

### 릴리즈 태그 사용
릴리즈 태그 방식은 action의 버전을 명시하는 가장 대표적인 방법입니다. 대부분의 Github Actions workflow 예시에는 action의 버전이 릴리즈 태그로 명시되어 있었습니다.

```yaml
# ...

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4 # 👈 actions 사용

# ...
```

Github Actions에서 action을 사용할 때에는 위와 같이 `{action 이름}@${action 버전}` 형식으로 어떤 action과 버전을 사용할 지를 명시합니다. 이 때 action의 버전에 `@3`과 같이 **메이저 릴리즈 버전**을 명시하면 workflow가 실행될 때마다  `@3` 버전 중 가장 최신 릴리즈 버전을 사용하게 됩니다.

이렇게 되면 workflow 실행마다 매번 다른 내용의 action이 실행될 수 있습니다. 이를테면 이전 배포에서는 action의 `@3.1.1` 버전이 실행되었는데, 다음 배포에서는 동일한 action의 `@3.1.2` 버전이 실행될 수 있는 것입니다.

action의 내용이 달라지면 의도치 않은 동작이 발생할 수 있고, 공급망 공격에도 취약해집니다. 마이너 버전 업데이트가 있을 때 사용자는 어떤 알림도 없이 업데이트 된 action을 사용하게 되기 때문이죠.

그렇다면 action에 **마이너 릴리즈 버전**을 명시하면 어떨까요? 처음부터 `@3.1.1`와 같이 마이너 버전을 명시해 둔다면 버전 업데이트와 상관 없이 항상 `@3.1.1` 버전을 사용할 것이기 때문에 공급망 공격을 막을 수 있을 것처럼 보입니다.

그러나 마이너 버전을 명시하더라도 action의 내용은 변경될 수 있습니다. Github에서는 릴리즈를 삭제할 수도 있고, 기존 릴리즈 태그에 다른 내용을 덮어 씌우는 것도 가능하기 때문입니다. 결국 의도적으로 동일한 릴리즈 태그에 다른 내용을 덮어씌우거나 해당 릴리즈 태그를 삭제하고 다른 내용으로 동일한 릴리즈 태그를 생성한다면 action 사용자는 내용이 변경되었음을 알아채지 못하고 사용하게 됩니다.

### 브랜치 이름 사용
브랜치 이름을 사용하는 방식 역시 앞선 릴리즈 태그 사용과 마찬가지로 브랜치의 내용이 변경되는 것을 알 수 없기 때문에 공급망 공격에 취약합니다. action을 사용할 때 브랜치 이름에 해당하는 브랜치의 가장 최신 버전을 사용하기 때문에 action의 내용이 알림 없이 변경되기 가장 쉬운 방식입니다.

`main`과 같은 대표 브랜치의 경우는 메이저 버전 업데이트조차 알림 없이 발생할 수 있기 때문에 action의 기존 내용과의 호환성이 깨질 수 있습니다. 따라서 브랜치 이름으로 action의 버전을 명시하는 방식은 공급망 공격 때문이 아니더라도 가급적 사용하지 않는 것이 좋습니다.

### commit SHA 사용
commit SHA란?
현재 전체 commit SHA를 명시하는 것이 action 내용의 불변성을 보장하는 유일한 방법.
commit SHA 사용 시 장점 ~~~

## action 업데이트 하기(dependabot)
그럼 평생 고정된 버전의 action을 사용해야 하나? 아니면 일일이 수동으로 업데이트 해줘야 하나?

dependabot을 사용하면 됨.
- dependency 버전을 최신으로 업데이트해주고 관리해줌.
- dependency의 취약점을 확인하고 알려줌.
- github actions뿐만 아니라 npm, yarn 등 여러 패키지 관리자에서도 사용 가능.
- 항상 최신 버전으로 유지하고 싶다면 dependabot이 최신 버전 업데이트 시 업데이트 내용과 커밋을 알려주기 때문에 확인하고 업데이트할 수 있음.
- 취약점만 알려주도록 설정할 수도 있음!
- (블로그 dependabot 추가 예시 삽입)

## 마치며

- 오픈소스 라이브러리에 취약점 발생, 지원 중단, 호환 불가 등의 이슈가 발생한다면 메인테이너에게 모든 책임을 떠넘기고 해결해 주기만을 기다릴 수는 없음. 사용자도 어느 정도 책임감을 가지고 라이브러리를 사용해야 하고, 오픈소스인 이상 라이브러리에 언제든 문제가 생길 수 있다는 사실을 인지하고 대책을 가지고 있어야 할 듯.