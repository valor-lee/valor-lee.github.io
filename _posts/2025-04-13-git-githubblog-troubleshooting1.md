---
title: '[github blog]GitHub Actions에서 setup-ruby 캐시 에러 해결'
date: 2025-04-12 00:32:00 +09:00
categories: [git, githubblog]
tags:
  [
    git,
    github blog,
    jekyll,
    setup-ruby
  ]
---

# 개요

최근 GitHub Actions를 이용해 내 블로그를 빌드하던 중, 갑작스럽게 setup-ruby 단계에서 캐시 관련 에러가 발생했습니다. 

```
The current runner (ubuntu-24.04-x64) was detected as self-hosted because the platform does not match a GitHub-hosted runner image (or that image is deprecated and no longer supported).

In such a case, you should install Ruby in the $RUNNER_TOOL_CACHE yourself, for example using https://github.com/rbenv/ruby-build

You can take inspiration from this workflow for more details: https://github.com/ruby/ruby-builder/blob/master/.github/workflows/build.yml

$ ruby-build 3.1.4 /opt/hostedtoolcache/Ruby/3.1.4/x64

Once that completes successfully, mark it as complete with:

$ touch /opt/hostedtoolcache/Ruby/3.1.4/x64.complete

It is your responsibility to ensure installing Ruby like that is not done in parallel.
```

환경 변화에 따른 이슈였고, 아래와 같은 간단한 수정으로 문제를 해결할 수 있었다.ㄴ

# 문제 원인

최근 GitHub Actions의 runner 환경이 ubuntu-24.04로 업데이트되면서, 해당 단계에서 다음과 같은 오류가 발생한 것으로 추정된다.

```
The current runner (ubuntu-24.04-x64) was detected as self-hosted because the platform does not match a GitHub-hosted runner image (or that image is deprecated and no longer supported).
In such a case, you should install Ruby in the $RUNNER_TOOL_CACHE yourself...
```

오류 메시지를 요약하면:

- ubuntu-24.04 환경에서는 기존에 사용하던 GitHub-hosted runner 이미지가 지원되지 않거나 변경되었고,
- 해당 상황에서는 직접 Ruby를 설치해야 한다는 것.

```
You can take inspiration from this workflow for more details: https://github.com/ruby/ruby-builder/blob/master/.github/workflows/build.yml
```

제 github blog는 .github/workflow의 jekyll.yml에 blog 테마나 필요한 툴 정보가 정리되어있습니다.

지금 문제가 되고 있는 ruby 버전도 기재되어 있습니다.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Ruby
        uses: ruby/setup-ruby@8575951200e472d5f2d95c625da0c7bec8217c42 # v1.161.0
```

문제는 ruby/setup-ruby 액션의 버전 고정으로 인해 최신 환경 대응이 되지 않았다는 점입니다.


# 해결 방법

특정 커밋 해시 대신 `@v1` 태그를 사용하면, 액션이 자동으로 최신 릴리스를 따라가게 되어 새로운 GitHub runner 환경에도 대응할 수 있다고 합니다.

```md
## Versioning

It is highly recommended to use `ruby/setup-ruby@v1` for the version of this action.
This will provide the best experience by automatically getting bug fixes, new Ruby versions and new features.

If you instead choose a specific version (v1.2.3) or a commit sha, there will be no automatic bug fixes and
it will be your responsibility to update every time the action no longer works.
Make sure to always use the latest release before reporting an issue on GitHub.

This action follows semantic versioning with a moving `v1` branch.
This follows the [recommendations](https://github.com/actions/toolkit/blob/master/docs/action-versioning.md) of GitHub Actions.
```
- [참고한 README](https://github.com/ruby/setup-ruby/blob/f6e05710eced3c9c28e489afdc6fd8a3bc685325/README.md?plain=1#L238-L248)


```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
```

# 끝으로

뭐든 version을 고정적으로 관리되지 않도록 주의하자 !