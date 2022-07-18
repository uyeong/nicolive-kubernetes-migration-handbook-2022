---
title: 타입스크립트로 쿠버네티스의 매니페스트를 작성한다
weight: 32
description: 쿠버네티스의 매니페스트를 타입스크립트로 작성하는 방법을 다룬다. OpenAPI 스키마로 부터 자동 생성된 타입 정의를 사용하며 타입스크립트와 npm 라이브러리의 에코시스템을 활용한다.
---

# 타입스크립트로 쿠버네티스의 매니페스트를 작성한다

이번 절에서는 기본적인 작성 방법을 소개한다.

## 기본적인 작성 방법

아래 코드는 타입스크립트로 작성한 스크립트(노드에서 동작하는)의 예다. 이를 `ts-node` 등으로 실행하면 `deployment.yml`가 출력되며 `kubectl apply -f deployment.yml`하여 쿠버네티스 상에 팟을 기동 시킬 수 있다.

```ts
import * as fs from "fs";
import * as yaml from "js-yaml";
import type { Schemas } from "@himenon/kubernetes-typescript-openapi/v1.22.3";

const podTemplateSpec: Schemas.io$k8s$api$core$v1$PodTemplateSpec = {
  metadata: {
    labels: {
      app: "nginx",
    },
  },
  spec: {
    containers: [
      {
        name: "nginx",
        image: "nginx:1.14.2",
        ports: [
          {
            containerPort: 80,
          },
        ],
      },
    ],
  },
};

const deployment: Schemas.io$k8s$api$apps$v1$Deployment = {
  apiVersion: "apps/v1",
  kind: "Deployment",
  metadata: {
    name: "nginx-deployment",
    labels: {
      app: "nginx",
    },
  },
  spec: {
    replicas: 3,
    selector: {
      matchLabels: {
        app: "nginx",
      },
    },
    template: podTemplateSpec,
  },
};

const text = yaml.dump(deployment, { noRefs: true, lineWidth: 144 });
fs.writeFileSync("deployment.yml", text, "utf-8");
```

## 타입스크립트로 작성할 때의 특징

타입스크립트로 작성할 때의 특정을 소개한다.

### 자유로운 YAML 작성법

우선 가장 느끼기 쉬운 특징으로는 YAML의 작성 방법에 껄끄러움이 없다는 것이다.
YAML은 단순한 출력 결과이며 출력하는 처리가 기법을 규격화하기 때문에 YAML의 작성 기법에 대한 리뷰가 필요 없다.

1. 들여쓰기는 공백인지 탭인지 여부
2. 들여쓰기는 공백 2칸인지 4칸인지 여부
3. 여러 행 주석은 `|`, `>` 중 어느 것으로 할 것인지 여부
4. 알파벳 순으로 정렬할 것인지 여부

등, 위와 같은 사항을 생각할 필요 없다.

### 주석을 활용한 효율적인 문서화

타입스크립트의 코드 주석을 그대로 이용할 수 있다.
따라서 변수명에 마우스 커서를 올렸을 때 주석이 함께 보이는 등의 에디터의 편의 기능을 지원 받을 수 있다.

또, 주석은 그대로 문서의 의미 갖게 되므로 매니페스트와 문서의 괴리를 예방해 혼란스러움을 막을 수 있다.

```ts
/** podTemplate 관련 주석 */
const podTemplateSpec: Schemas.io$k8s$api$core$v1$PodTemplateSpec = {};

const deployment: Schemas.io$k8s$api$apps$v1$Deployment = {
  apiVersion: "apps/v1",
  kind: "Deployment",
  metadata: {
    name: "nginx-deployment",
    labels: {
      /** 이 라벨을 붙인 이유 작성 */
      app: "nginx",
    },
  },
  spec: {
    /** 레플리카를 3으로 설정한 이유 */
    replicas: 3,
    /** 이 셀렉터를 붙인 이유 */
    selector: {
      matchLabels: {
        app: "nginx",
      },
    },
    template: podTemplateSpec,
  },
};
```

### 변수를 이용한 관계성 표현

쿠버네티스에서 기본적인 서비스와 디플로이먼트라는 세트를 떠올려보자. 서비스간 통신을 위해서는 서비스의 셀렉터를 팟의 라벨과 일치시켜야 한다. 이 셀렉터와 라벨 부분을 타입스크립트의 변수로 지정하면 확실하게 관계를 드러내어 서비스와 디플로이먼트 매니페스트를 생성할 수 있다.

이 외에도 [권장되는 라벨](https://kubernetes.io/ja/docs/concepts/overview/working-with-objects/common-labels/)인 `app.kubernetes.io/version` 등도 누락없이 적절하게 지정할 수 있다.

```ts
const Namespace = "mynamespace";

export const generateService = (applicationName: string, applicationVersion: string): Schemas.io$k8s$api$core$v1$Service => {
  return {
    apiVersion: "v1",
    kind: "Service",
    metadata: {
      name: applicationName,
      namespace: Namespace,
    },
    spec: {
      type: "ClusterIP",
      selector: {
        app: applicationName,
        "app.kubernetes.io/name": applicationName,
      },
      ports: [
        {
          name: `http-${applicationName}-svc`,
          port: 80,
          targetPort: 80,
        },
      ],
    },
  };
}


export const generateDeployment = (applicationName: string, applicationVersion: string): Schemas.io$k8s$api$apps$v1$Deployment => {
  return {
    apiVersion: "apps/v1",
    kind: "Deployment",
    metadata: {
      name: applicationName,
      namespace: Namespace,
      labels: {
        app: applicationName,
        "app.kubernetes.io/name": applicationName,
      },
      annotations: {},
    },
    spec: {
      selector: {
        matchLabels: {
          "app.kubernetes.io/name": applicationName,
        },
      },
      /** 생략 */
    },
  };
}

const applicationName = "my-nginx";
const applicationVersion = "1.14.2";

generateService(applicationName, applicationVersion);
generateDeployment(applicationName, applicationVersion);
```

### 템플릿의 표현력 증가

예를 들어 노드, 고, 스칼라 등 다양한 언어로 구현된 마이크로서비스를 위한 기본적인 디플로이먼트 템플릿 등을 준비할 수 있다.
예를 들어 `/a`와 `/b`의 엔드포인트가 같은 서버에서 제공되고 있지만, 수평 확장(scale out) 단위나 CPU / MEM 등 각각의 자원을 분리하여 관리하고자 매니페스트를 분할 하고 싶을 때에 큰 도움이 된다.

```ts
export const generateNodeJsDeployment = ():Schemas.io$k8s$api$apps$v1$Deployment => {};
export const generateRubyOnRailsDeployment = ():Schemas.io$k8s$api$apps$v1$Deployment => {};
export const generateScalaDeployment = ():Schemas.io$k8s$api$apps$v1$Deployment => {};
```

### 테스트 역할을 대신하는 제너레이터

매니페스트를 생성 시에 활용할 테스트 프레임워크는 불필요하며, 단순하게 `Exception`을 발생시키는 것 만으로도 충분한 테스트가 된다.
예를 들어 `metadata.name`에 지정할 수 있는 문자열 또는 문자수는 `Service`나 `Job` 등의 리소스 타입으로 정해져 있다（[참고](https://kubernetes.io/ja/docs/concepts/overview/working-with-objects/names/#dns-label-names)）。

큰 변경을 시도하고나서 `kubectl apply` 실행 후 문제를 발견 한 경우라면 원인을 찾는데 꽤 수고가 필요하다. 하지만 매니페스트를 생성하는 시점에 구체적인 에러 메시지를 출력하고 처리를 중단한다면 문제의 원인을 즉시 알 수 있다.
로컬 환경에서 생성해보지 않고 곧바로 PR 할 경우 CI 단계에서 생성을 시도하여 검증할 수 있다.

```ts
export const validateMetadataName = (text: string, throwError?: true): string => {
  if (throwError && text.length > 63) {
    throw new Error(`May not be deployed correctly because it exceeds 63 characters.\nValue: "${text}"`);
  }
  return text.slice(0, 63);
};

export const generateJob = (applicationName: string): Schemas.io$k8s$api$batch$v1$Job => {
  return {
    apiVersion: "batch/v1",
    kind: "Job",
    metadata: {
      name: validateMetadataName(applicationName, true),
    },
  };
};
```
