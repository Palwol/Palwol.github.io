---
title: Github Actions에서 action 버전을 고정해야 하는 이유
date: 2024-04-25 +0900
categories: [github]
tags: [github-actions, dependabot, security]
img_path: /assets/img/posts/2024-04-19
image: github-actions.png
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
[commit SHA](https://docs.github.com/en/pull-requests/committing-changes-to-your-project/creating-and-editing-commits/about-commits#about-commits)는 SHA-1이라는 해시 알고리즘을 이용해 각 commit에 부여된 고유한 id입니다. 이 commit SHA는 변경된 내용, 변경된 시간, 변경한 사람에 따라서 달라지기 때문에 commit SHA가 다르면 변경이 발생한 다른 commit이라는 것을 보장할 수 있습니다.

```yaml
# ...

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1

# ...
```

action 버전에 commit SHA만을 입력하면 현재 사용하고 있는 action이 어떤 버전인지 알기 어렵기 때문에 action 버전 뒤에 주석으로 릴리즈 태그를 달아주면 편하게 버전 정보까지 확인할 수 있습니다.

현재 저장소의 commit 이력과 commit SHA는 `git log` 명령어로 확인할 수 있습니다. `git log`를 입력하면 다음과 같이 commit 이력의 commit SHA, 커밋 작성자, 날짜, 커밋 메세지를 보여줍니다.

![커밋 SHA](commit-sha.png)
_commit 옆의 알 수 없는 문자 배열이 commit SHA입니다._

따라서 action의 버전을 전체 commit SHA로 고정하는 것은 action의 불변성을 보장해주며, 공급망 공격을 막을 수 있는 매우 중요한 보안 수단입니다. 또한 action이 각 실행에서 완벽하게 같은 동작을 하기 때문에 배포 위험을 최소화할 수 있고 디버깅이 쉬워집니다.

## action 업데이트 하기(Dependabot)
그렇다면 보안을 위해 영원히 고정된 버전의 action을 사용해야 할까요? action 제작자가 성능이 개선되거나 기능이 추가된, 또는 보안 이슈가 해결된(!) 새로운 버전을 릴리즈 한다면 버전 업데이트가 필요할 것입니다. 이런 경우에는 일일이 수동으로 버전 업데이트를 해줘야 할까요?

이럴 때 사용하면 좋은 도구가 바로 [Dependabot](https://docs.github.com/en/code-security/getting-started/dependabot-quickstart-guide)입니다. Dependabot은 Github에서 제공하는 dependency 업데이트 자동화 도구로, 다음과 같은 기능을 가지고 있습니다.
- 의존성 보안 취약점 발생 시 알림을 보냅니다.
- 의존성에 보안 업데이트, 버전 업데이트가 발생하면 자동으로 업데이트 PR을 생성합니다.

Dependabot 관리하는 의존성 시스템은 Github Actions뿐만 아니라 npm, yarn과 같은 패키지 매니저도 포함합니다. [package-ecosystem](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#package-ecosystem)에서 Dependabot이 지원하는 패키지 시스템 목록을 확인할 수 있습니다.

### Dependabot 사용하기
Dependabot을 활성화시키는 방법은 매우 간단합니다. Dependabot으로 의존성을 관리하고 싶은 레포지토리에서 **Settings > (왼쪽 메뉴의) Security > Code security and analysis** 메뉴에 들어갑니다.

![Dependabot 메뉴](dependabot.png)
_Dependabot 설정 항목들_

그럼 위와 같은 항목들이 나타나는데, 여기서 원하는 항목의 `Enable` 버튼을 눌러 활성화 시키면 끝입니다. 너무 간단하죠? 따로 설정 파일을 만들어 줄 필요도 없습니다.

github는 이미 레포지토리에서 사용하고 있는 각종 의존성의 정보를 가지고 있습니다. 현재 레포지토리서 사용 중인 의존성 정보는 레포지토리의 **Insights > (왼쪽 메뉴의) Dependency graph > Dependencies** 탭에서 확인할 수 있습니다.

Dependabot을 활성화하면 이 정보들을 이용해서 어떤 의존성 모듈이 최신 업데이트 되었는지, 보안 이슈가 발생했는지를 확인하고 해결하는 PR을 자동으로 생성합니다. 어떤 변경이 발생했는지 전체 release 로그도 PR 내용에 같이 남겨주므로 release 내용에 문제가 없는지 확인하고 PR을 반영할 수 있습니다. 여러모로 효자 로봇(?)인 듯 합니다.

![Dependabot이 올린 PR](dependabot-pr.png)
_Dependabot이 이쁘게 PR을 올려줍니다._

참고로 action 버전으로 commit SHA를 사용할 때 가독성을 위해 주석으로 릴리즈 태그를 같이 작성해주는데, dependabot은 의존성 업데이트 시 이 주석까지 같이 수정해줍니다.

![주석까지 수정해주는 Dependabot](dependabot-tag.png)
_v4.1.1에서 v4.1.3으로 주석까지 업데이트 해주는 멋쟁이 Dependabot 🎉_

또한 앞서 설정 파일이 필요하지 않다고 했는데, `dependabot.yml` 파일을 작성하면 버전 확인을 원하지 않는 모듈을 제거하거나 버전 확인 주기를 변경하는 등의 상세한 설정도 가능합니다. 자세한 내용은 [Configuring Dependabot version updates](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates) 문서를 참고해 주세요.

## 마치며
이번 일을 겪으면서 그 동안 수많은 오픈소스를 사용하면서도 보안에 너무 무심했던 것 같다는 생각이 들었습니다. 사용 중인 오픈소스에 취약점 발생, 지원 중단, 호환 불가 등의 이슈가 발생했을 때 메인테이너에게 모든 책임을 떠넘기고 해결해 주기만을 기다릴 수는 없을 것입니다. 사용자도 오픈소스에 언제든 문제가 생길 수 있다는 사실을 인지하고 대비해야 하지 않을까 합니다.

읽어주셔서 감사합니다!