---
title: 쿠버네티스의 매니페스트 관리
weight: 31
description: BFF에서의 쿠버네티느 매니페스트 관리와 관련한 문제 정리 및 대응 방법을 간단하 소개한다.
---

# 쿠버네티스의 매니페스트 관리

이번 장에서는 매니페스트 관리를 어떻게 하고 있는지 소개한다.
결론부터 말하면 우리는 쿠버네티스에서 이용할 매니페스트를 생성해 주는 제너레이터를 타입스크립트로 구축했다.

어떻게 구축하여 운용하고 있는지 설명한다.

## 이전 후 각 컴포넌트의 파일 수로 살펴보는 규모

[도입](../../../#이전-후-쿠버네티스의-대략적인-규모) 부분에서 다뤘던 자료를 다시금 언급. **프런트엔드와 관련한 마이크로서비스**의 매니페스트는 다음과 같은 규모로 존재한다.
이는 간단히 관리할 수 없는 컴포넌트 수이며 앞으로도 늘어날 전망이다.

| 컴포넌트                                       |  파일 수 |
|:-------------------------------------------|------:|
| v1/Deployment                              |    20 |
| v1/Service                                 |    60 |
| v1/Config Map                              |    15 |
| batch/v1/Job                               |    15 |
| argoproj.io/v1alpha1/Rollout               |    20 |
| networking.istio.io/v1beta1/VirtualService |    20 |
| networking.istio.io/v1alpha3/EnvoyFilter   |    20 |

## 문제 의식

이전 전 단계에서 이미 파일 수는 YAML로 관리하기에는 곤란한 양이었으며 도구를 활용한 보완이나 테스트 없이는 반드시 큰 문제를 야기할 수 있다고 쉽게 예상할 수 있다. 이러한 상황은 이전 프로젝트 처음에 정했던 두 가지의 목표에 어긋난다.

* 디플로이를 민첩하고 간단하며 안전하게 실시 할 수 있다.
* 웹 프런트엔드 개발자가 갱신에 필요한 설정을 최소한으로 간단히 변경할 수 있다.

### 타입스크립트로 매니페스트의 관리 문제를 해결

이 문제를 전반적으로 해결하는 한 가지 방법은 타입스크립트를 통해 쿠버네티스 YAML을 생성하는 것이다.
타입스크립트 자체의 장점은 이미 많은 글이 공개돼 있으므로 생략한다. 

쿠버네티스를 운용하는 팀은 타입스크립트를 일상적으로 이용하고 있는 웹 프런트엔드 엔지니어들로 이뤄져 있다는 점만 참고한다. 그래서 타입스크립트로 매니페스트를 작성하는 것 자체에 대한 장벽은 매우 낮은 상태다.

또, 매니페스트 테스트도 타입스크립트로 YAML을 생성하는 단계에서 `Exception`을 발생시켜 버리면 되기 때문에 테스트 전략도 매우 단순하게 가져갈 수 있다.

만일 타입스크립트로 작성하는 것을 그만두고 싶은 경우에는 생성된 YAML로 대체하면 되기 때문에 타입스크립트 자체를 걷어내는 것도 간단하게 가능하다.

위와 같이 타입스크립트로 작성하지 않을 이유가 특별히 이전 설계 단계에서 드러나지 않았기 때문에 매니페스트를 YAML로 작성하는 것을 버리고 타입스크립트로 작성하기로 결정했다.

## 타입스크립트로 쿠버네티스를 작성하기 위한 지원 라이브러리와 스키마

쿠버네티스는 `CustomResourceDefinition`을 정의할 때 OpenAPI Schema V3로 기술한다. 이에 의해 실제 스키마가 적용 될 때에 유효성 검사가 이뤄진다.

다르게 말하면 OpenAPI의 스키마를 타입스크립트의 타입 정의에 활용할 수 있다면 유효성 검사를 타입스크립트의 정적 타입 분석으로 변환할 수 있다는 뜻이다.

다행히 필자는 OpanAPI 스키마와 타입스크립트에 대해 조금 자세히 알고있어 자화자찬 이지만 [@himenon/openapi-typescript-code-generator](https://github.com/Himenon/openapi-typescript-code-generator)를 기반으로 쿠버네티스의 타입을 정의했다.

* [@himenon/kubernetes-typescript-openapi](https://github.com/Himenon/kubernetes-typescript-openapi)
* [@himenon/argocd-typescript-openapi](https://github.com/Himenon/argocd-typescript-openapi)
* [@himenon/argo-rollouts-typescript-openapi](https://github.com/Himenon/argo-rollouts-typescript-openapi)

물론 이 밖에 비슷한 접근법으로 타입 정의를 제공하고 있는 라이브러리도 있지만 다음과 같은 이유로 사용을 보류했다.

- 직접 타입스크립트의 객체로 하여금 간단하게 타입을 정의하고 싶다.
    - 이는 라이브러리 쪽 지원이 중단돼도 스스로 수정할 수 있기 때문이다.
- ArgoCD나 ArgoRollouts, 이스티오 등 또 다른 사용자 정의 리소스를 사용할 때에도 동일 라이브러리를 편히 사용하고 싶다.
- 최신 버전 뿐 아니라 임의의 오래된 버전도 지원한다.

위와 같은 내용을 미루어 볼 때 되도록 라이브러리는 얇게 구현돼 있는 것이 바람직하며, 타입 정의 라이브러리를 포크한 경우에도 간단하게 유지보수가 가능한 구현체가 필요했다. 이러한 조건을 충족하는 설계 컨셉은 [@himenon/openapi-typescript-code-generator](https://github.com/Himenon/openapi-typescript-code-generator) 였다.

이와 관련한 자세한 내용은 다음 장에 이어서 설명하겠다.

- [타입스크립트로 쿠버네티스의 매니페스트를 작성한다](../kubernetes-manifest-written-by-typescript/)
- [타입스크립트로 매니페스트를 생성하는 제너레이터 아키텍처](../kubernetes-manifest-generator-architecture/)

## 다른 라이브러리

쿠버네티스 용 타입스크립트 라이브러리는 다음과 같다.

* [cdk8s](https://github.com/cdk8s-team/cdk8s)
* [kosko](https://github.com/tommy351/kosko)

쿠버네티스의 정의(definition)는 [CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)를 참고한다.

