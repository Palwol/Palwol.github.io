---
title: Github Actions에서 action의 버전을 commit SHA로 명시해야 하는 이유
date: 2024-04-21 +0900
categories: [github]
tags: [github-actions, dependabot]
img_path: /assets/img/posts/2024-04-19
image: GitHub-Actions.png
render_with_liquid: false
---

> 이 포스트는 Github Actions에 대한 설명은 포함하지 않습니다. Github Actions에 대해 자세히 알고 싶으시다면 [Github Actions 설명서](https://docs.github.com/ko/actions)를 참고해 주세요.

Github Actions는 Github에서 제공하는 CI/CD 도구로, workflow 자동화를 지원합니다. CI뿐만 아니라 테스트 자동화와 같이 여러 방면으로 활용할 수 있어 많은 Github 사용자들이 애용하는 도구입니다.

저 또한 CI와 테스트에 Github Actions를 사용하고 있는데요. 이번에 AWS ECS에 CI/CD를 구축하기 위해 [Deploy to Amazon ECS](https://docs.github.com/en/actions/deployment/deploying-to-your-cloud-provider/deploying-to-amazon-elastic-container-service#creating-the-workflow) 문서를 참고하던 도중 workflow 예시 주석에서 다음과 같은 내용을 마주했습니다.

```yaml
# GitHub recommends pinning actions to a commit SHA.
# To get a newer version, you will need to update the SHA.
# You can also reference a tag or branch, but the action may change without warning.
```
해석하면 다음과 같습니다.
- Github는 action(의 버전)들을 commit SHA로 고정하기를 추천합니다.
- 새로운 버전을 사용하려면 SHA를 업데이트 해야 합니다.
- SHA 대신 tag나 branch를 사용할 수도 있지만, action이 경고 없이 변경될 수 있습니다.

action을 commit SHA로 고정하라니 이게 무슨 소리인가 싶어서 찾아보니 생각보다 중요한 보안 이슈가 관련된 내용이었습니다. 이번 포스팅에서는 Node.js 개발자인 RAFAEL GONZAGA의 [Why you should pin your GitHub Actions by commit-hash](https://blog.rafaelgss.dev/why-you-should-pin-actions-by-commit-hash) 포스팅 내용을 중심으로, 관련 내용을 조사하면서 알게 된 내용을 공유해보려 합니다.

## action 버전을 고정하지 않을 경우
### 메이저 버전 사용 시
Github Actions를 사용할 때에는 [Marketplace](https://github.com/marketplace?type=actions)에 등록되어 있는 다양한 action들을 주로 사용하게 됩니다. 일반적으로 workflow에서 action은 다음과 같이 사용합니다.

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

action을 사용할 때에는 위와 같이 `{action 이름}@${action 버전}` 형식으로 어떤 action과 버전을 사용할 지를 명시합니다. 이 때 action의 버전은 `@3`과 같이 **메이저 릴리즈 버전**을 명시하는 것이 일반적입니다. 

이렇게 메이저 버전을 명시하면 workflow가 실행될 때마다  `@3` 버전 중 가장 최신 릴리즈 버전을 사용하게 됩니다. 이렇게 되면 workflow 실행마다 매번 다른 내용의 action이 실행될 수 있습니다. 이를테면 이전 배포에서는 action의 `@3.1.1` 버전이 실행되었는데, 다음 배포에서는 동일한 action의 `@3.1.2` 버전이 실행될 수 있는 것입니다.

action의 내용이 달라지는 것은 의도치 않은 동작을 일으킬 수 있습니다. 물론 마이너 버전 업데이트는 기존 동작의 호환성을 해치지 않는 업데이트일 가능성이 크지만, 그럼에도 코드 변경은 항상 예상하지 못한 부작용을 일으킬 수 있기 때문에 의도하지 않은 변경은 가급적 막는 것이 좋습니다.

### 마이너 버전 사용 시
그렇다면 action에 **마이너 릴리즈 버전**을 명시하면 어떨까요? 처음부터 `@3.1.1`와 같이 마이너 버전을 명시해 둔다면 버전 업데이트와 상관 없이 항상 `@3.1.1` 버전을 사용할 것이기 때문에 의도치 않은 변경을 막을 수 있을 듯 합니다.

그러나 마이너 버전을 명시하더라도 action의 내용은 변경될 수 있습니다. 버전은 Github에서의 [릴리즈 태그](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)와 동일한데, Github에서는 이 릴리즈를 삭제할 수도 있고 기존 릴리즈 태그에 다른 내용을 덮어 씌우는 것도 가능하기 때문입니다.

결국 action 제작자가 의도적으로 동일한 릴리즈 태그에 다른 내용을 덮어씌우거나 해당 릴리즈 태그를 삭제하고 다른 내용을 동일한 릴리즈 태그를 생성한다면 action 사용자는 내용이 변경된 action을 알아채지 못하고 사용하게 됩니다.

## action 버전 고정하기
### 해결책: commit SHA 사용하기
commit SHA란?
현재 전체 commit SHA를 명시하는 것이 action 내용의 불변성을 보장하는 유일한 방법.
commit SHA 사용 시 장점

### action 업데이트 하기(dependabot)
그럼 평생 고정된 버전의 action을 사용해야 하나? 아니면 일일이 수동으로 업데이트 해줘야 하나?

dependabot을 사용하면 됨.
- dependency 버전을 최신으로 업데이트해주고 관리해줌.
- dependency의 취약점을 확인하고 알려줌.
- github actions뿐만 아니라 npm, yarn 등 여러 패키지 관리자에서도 사용 가능.
- 항상 최신 버전으로 유지하고 싶다면 dependabot이 최신 버전 업데이트 시 업데이트 내용과 커밋을 알려주기 때문에 확인하고 업데이트할 수 있음.
- 취약점만 알려주도록 설정할 수도 있음!
- (블로그 dependabot 추가 예시 삽입)

## 마치며: XZ-utils 백도어 사건과 공급망 공격
사용하는 action의 불변성을 보장하는 것은 코드의 의도치 않은 동작을 막는 데에도 도움이 되지만, **공급망 공격(Supply Chain Attack)**을 막을 수 있다는 데에 큰 의의가 있습니다.

공급망 공격이란 ~~~

- 최근에 발생한 XZ Utils 백도어 사건과 같이 오픈소스에 크게 의존하고 있는 IT 업계는 공급망 공격같은 이슈에 취약할 수밖에 없음. XZ utils 사건은 Outsider님의 [XZ Utils 백도어 사건으로 돌아보는 오픈소스 생태계](https://blog.outsider.ne.kr/1714) 포스팅에 자세히 설명되어 있습니다.

- 오픈소스 라이브러리에 취약점 발생, 지원 중단, 호환 불가 등의 이슈가 발생한다면 메인테이너에게 모든 책임을 떠넘기고 해결해 주기만을 기다릴 수는 없음. 사용자도 어느 정도 책임감을 가지고 라이브러리를 사용해야 하고, 오픈소스인 이상 라이브러리에 언제든 문제가 생길 수 있다는 사실을 인지하고 대책을 가지고 있어야 할 듯.