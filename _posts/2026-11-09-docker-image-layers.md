---
layout: post
title: "코드 한 줄을 바꿔도 Docker 이미지는 37MB를 새로 올린다"
summary: "traffic-stats-lab의 Spring Boot API를 Dockerfile 네 가지로 빌드해 이미지 크기를 재고, 코드 한 줄을 바꾼 뒤 로컬 레지스트리에 push할 때 실제로 올라가는 레이어 크기를 비교했다."
categories: [직접 해본 것]
topic: 인프라·배포
tags: [Docker, 레이어, 멀티스테이지, Spring Boot, 컨테이너 이미지]
repo: https://github.com/achul1026/traffic-stats-lab
repo_note: 실험 대상인 API 소스와 원래 Dockerfile이 들어 있다. 이 글의 비교용 Dockerfile은 저장소에 넣지 않고 본문에만 적었다.
date: 2026-11-09 09:00:00 +0900
---

traffic-stats-lab의 API 이미지는 멀티 스테이지로 빌드한다. 빌드 단계에서 jar를 만들고 실행 이미지에는 JRE와 jar만 둔다. 이 정도면 충분히 정리된 이미지라고 생각했다. 그런데 이미지가 정말 몇 바이트인지, 코드를 고칠 때마다 레지스트리로 얼마가 다시 올라가는지는 잰 적이 없었다. 그래서 같은 소스를 Dockerfile 네 가지로 빌드해서 쟀다.

## 어떻게 쟀나

- 대상: traffic-stats-lab의 `api` 디렉터리(Spring Boot 3.5.12, Java 21). 원본 저장소는 건드리지 않고 사본에서 빌드했다.
- 환경: macOS 15.8.2, Intel Core i5-8257U, Docker Desktop(엔진 29.4.3, x86_64, 메모리 4GB 할당). 이미지 저장소는 containerd 스냅샷터를 쓰는 설정이다.
- 베이스 이미지: `eclipse-temurin:21-jdk`, `21-jre`, `21-jre-alpine`. 2026년 11월 1일에 받은 것이고, 안에 든 JDK는 21.0.13, OS는 Ubuntu 26.04.1과 Alpine 3.24였다.
- 크기: `docker images`의 CONTENT SIZE(압축된 레이어 합)와 DISK USAGE(풀린 크기)를 같이 적었다. 레이어별 압축 크기는 로컬에 `registry:2`를 띄워 push한 뒤 매니페스트에서 읽었다.
- 측정은 각 1회다. 빌드 시간도 찍었지만 빌드끼리 캐시를 나눠 써서 비교할 수 있는 값이 아니었다. 그래서 표에 넣지 않았다.

## 네 가지 Dockerfile

| | 방식 |
|---|---|
| A | JDK 이미지 하나에서 빌드하고 그대로 실행 |
| B | 지금 저장소에 있는 것. JDK에서 빌드, `21-jre`로 jar만 복사 |
| C | B와 같고 실행 이미지만 `21-jre-alpine` |
| D | B와 같은 베이스에, jar를 Spring Boot 레이어별 디렉터리로 풀어서 복사 |

D의 실행 단계는 이렇다. 빌드 단계 끝에서 `java -Djarmode=tools -jar app.jar extract --layers --launcher --destination /x`로 jar를 풀어 두었다.

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /x/dependencies/ ./
COPY --from=build /x/spring-boot-loader/ ./
COPY --from=build /x/snapshot-dependencies/ ./
COPY --from=build /x/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

네 이미지 모두 컨테이너로 띄워서 `Started TrafficLabApplication` 로그가 찍히는 것까지 확인했다. DB와 Redis 없이 띄웠기 때문에 실제 질의는 돌리지 않았다.

## 이미지 크기

| | 압축(CONTENT SIZE) | 풀린 크기(DISK USAGE) | 레이어 수 |
|---|---|---|---|
| A 단일 JDK | 496MB | 1.34GB | 12 |
| B 멀티 스테이지 JRE | 152MB | 534MB | 8 |
| C 멀티 스테이지 JRE alpine | 111MB | 364MB | 7 |
| D 멀티 스테이지 JRE + 레이어 분리 | 152MB | 535MB | 11 |

A가 큰 이유를 컨테이너 안에서 `du`로 봤다. JDK가 295MB, Gradle 캐시(`/root/.gradle`)가 254MB, 빌드 산출물이 40MB였다. 앱을 돌리는 데는 하나도 필요 없는 것들이다. 멀티 스테이지로 바꾸는 것만으로 압축 기준 3.3배가 줄었다.

C는 B보다 41MB 작다. 줄어든 건 거의 OS 쪽이다. B의 Ubuntu 베이스 레이어가 42.1MB인데 Alpine은 3.9MB였다. JRE 레이어는 53.2MB와 53.4MB로 거의 같았다. Alpine은 glibc가 아니라 musl을 쓰는데, 이 앱이 거기서 문제없이 도는지는 기동만 확인했지 부하를 걸어 보지 않았다.

D는 크기로는 B와 같다. 차이는 다음 실험에서 나온다.

## 코드 한 줄을 바꾸면 얼마가 다시 올라가나

`TrafficLabApplication`에 `static final String BUILD = "2";` 한 줄을 넣고 B와 D를 다시 빌드했다. 그리고 두 버전을 차례로 로컬 레지스트리에 push했다. 두 번째 push에서 `Pushed`로 찍힌 레이어가 실제로 새로 올라간 것이다.

| | 바뀐 레이어 | 새로 올라간 레이어 크기(압축) |
|---|---|---|
| B | jar가 든 레이어 1개 / 8개 | 36,764,798B |
| D | `application/` 레이어 1개 / 11개 | 10,329B |

B는 jar 파일 하나가 레이어 하나다. 내 코드가 몇 바이트 바뀌어도 jar 전체, 그러니까 라이브러리 40MB(압축 36.8MB)를 통째로 다시 올린다. D에서는 라이브러리가 `dependencies/` 레이어에 따로 있어서 그대로 재사용됐고, 내 클래스가 든 10KB만 새로 올라갔다. 약 3,560배 차이다. 이미지 설정(config) blob 약 10KB는 두 경우 모두 따로 올라간다.

D의 레이어 크기를 압축 기준으로 보면 `dependencies` 36,567,918B, `spring-boot-loader` 132,949B, `snapshot-dependencies` 32B(비어 있다), `application` 10,329B다. 라이브러리 버전을 올리면 D도 36MB를 다시 올리게 된다. 레이어 분리가 아끼는 건 코드만 바뀌는 배포다.

## 모르는 것

- 이 차이가 배포 시간으로 얼마가 되는지는 재지 않았다. 로컬 레지스트리라서 네트워크 비용이 거의 없었고, 원격 레지스트리와 실제 서버의 pull은 해 보지 않았다.
- 빌드 시간 비교는 하지 않았다. 앞에서 적었듯 캐시가 섞여서 숫자를 믿을 수 없었다.
- 기동 시간은 B와 D를 같은 시각에 함께 띄워서 잰 값이라 비교에 쓰지 않았다.
- `jlink`로 필요한 모듈만 담은 런타임을 만드는 방법, distroless 베이스는 시도하지 않았다.

<!-- TODO(직접 채워 주세요): traffic-stats-lab의 Dockerfile을 D 방식으로 바꿀지, alpine으로 갈지에 대한 판단과 다음 계획 -->
