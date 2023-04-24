---
title: "쿠버네티스로 비-루트 .NET 컨테이너 실행하기 | Richard Lander"
datePublished: Mon Apr 24 2023 00:01:41 GMT+0000 (Coordinated Universal Time)
cuid: clgu2pkmj000009mg5uni72qa
slug: net-richard-lander
tags: net, dotnet

---

> Richard Lander님의 [Running non-root .NET containers with Kubernetes](https://devblogs.microsoft.com/dotnet/running-nonroot-kubernetes-with-dotnet/)을 번역하였습니다.

---

루트리스 또는 비-루트 Linux 컨테이너는 .NET 컨테이너 팀에서 가장 많이 요청한 기능입니다. 최근에 모든 [.NET 8 컨테이너 이미지를 코드 한 줄로 비-루트로 구성](https://devblogs.microsoft.com/dotnet/securing-containers-with-rootless/)할 수 있게 되었다고 발표했습니다. 이 변경은 보안 태세를 개선하는 환영할 만한 변화입니다. 지난 포스팅에서 쿠버네티스로 비-루트 호스팅에 접근하는 방법에 대한 후속 포스팅을 약속드린 바 있습니다. 오늘은 그 내용을 다루겠습니다.

[비-루트 쿠버네티스 샘플](https://github.com/dotnet/dotnet-docker/blob/main/samples/kubernetes/non-root/README.md)을 사용하여 클러스터에서 비-루트 컨테이너를 호스팅해 볼 수 있습니다. 이 샘플은 현재 작업 중인 더 큰 [쿠버네티스 샘플](https://github.com/dotnet/dotnet-docker/blob/main/samples/kubernetes/README.md) 세트의 일부입니다.

이 포스팅은 [쿠버네티스 "제한적" 강화 모범 사례](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted)를 따르는 데 도움이 될 것입니다. 비-루트 호스팅은 이 가이드의 핵심 요구 사항입니다.

## `runAsNonRoot`

우리가 논의할 대부분의 내용은 쿠버네티스 매니페스트의 [`SecurityContext`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.24/#securitycontext-v1-core) 섹션과 관련이 있습니다. 여기에는 쿠버네티스가 적용하는 보안 구성이 들어있습니다.

```yaml
    spec:
      containers:
      - name: aspnetapp
        image: dotnetnonroot.azurecr.io/aspnetapp
        securityContext:
          runAsNonRoot: true
```

이 `securityContext` 객체는 컨테이너가 루트가 아닌 사용자로 실행되는지 확인합니다. 정말 간단합니다.

[non-root.yaml](https://github.com/dotnet/dotnet-docker/blob/23b937b9aecfcbffdbb4d1cc25049abe38377f76/samples/kubernetes/non-root/non-root.yaml#L20-L22)의 더 넓은 컨텍스트에서 이를 확인할 수 있습니다.

`runAsNonRoot`는 사용자가 (UID를 통해) 루트가 아닌 사용자(`> 0`)인지 테스트하며, 그렇지 않으면 파드 생성이 실패합니다. 쿠버네티스는 이 테스트를 위해 컨테이너 이미지 메타데이터만 읽습니다. 컨테이너를 시작해야 하므로(테스트의 목적에 어긋나므로) `/etc/passwd`는 읽지 않습니다. 즉, Docker파일에서 `USER`는 UID로 설정되어야 합니다. `USER`가 이름으로 설정되어 있으면 실패합니다.

`docker inspect`로 동일한 테스트를 시뮬레이션할 수 있습니다.

```bash
% docker inspect dotnetnonroot.azurecr.io/aspnetapp -f "{{.Config.User}}"
64198
```

보시다시피 샘플 이미지에서는 UID로 사용자를 설정합니다. 그러나 `whoami` 는 여전히 사용자를 `app`으로 보고합니다.

위의 예제에서는 사용되지 않았지만 `runAsUser`도 관련 설정입니다. 컨테이너 이미지의 `USER`가 설정되지 않았거나, UID가 아닌 이름으로 설정되었거나, 기타 원하지 않는 경우에만 `runAsUser`를 사용해야 합니다. 새 `app` 사용자를 UID로 사용하기가 매우 쉬워졌기 때문에 .NET 앱에는 `runAsUser`가 필요하지 않습니다.

## `USER` 모범 사례

Dockerfile에서 `USER`를 설정할 때는 다음 패턴을 사용하는 것이 좋습니다.

```plaintext
USER $APP_UID
```

`USER` 명령은 순서는 중요하지 않지만 `ENTRYPOINT` 바로 앞에 배치되는 경우가 많습니다.

이 패턴을 사용하면 Dockerfile에서 매직넘버를 피하면서 `USER`가 UID로 설정됩니다. 환경 변수는 이미 .NET 이미지에서 선언한 대로 UID 값을 정의합니다.

.NET 이미지에서 환경 변수 설정을 확인할 수 있습니다.

```bash
% docker run mcr.microsoft.com/dotnet/runtime-deps:8.0-preview bash -c "export | grep UID"
declare -x APP_UID="64198"
```

비-루트 샘플은 이 패턴에 따라 [UID로 사용자를 설정](https://github.com/dotnet/dotnet-docker/blob/6f8bc3bf46a990fb954506383ec360c6ac5b7d91/samples/aspnetapp/Dockerfile.alpine-non-root#L29)합니다. 결과적으로 [`runAsNonRoot`](https://github.com/dotnet/dotnet-docker/blob/23b937b9aecfcbffdbb4d1cc25049abe38377f76/samples/kubernetes/non-root/non-root.yaml#L22)와 잘 작동합니다.

## 액션에 비-루트 호스팅

[비-루트 쿠버네티스 샘플](https://github.com/dotnet/dotnet-docker/blob/main/samples/kubernetes/non-root/README.md)을 사용하여 비루트 컨테이너 호스팅의 경험을 살펴보겠습니다.

저는 로컬 클러스터에 [minikube](https://minikube.sigs.k8s.io/docs/start/) 를 사용하고 있지만, 모든 쿠버네티스 호환 환경은 [kubectl](https://kubernetes.io/docs/tasks/tools/)과 잘 작동해야 합니다.

```bash
$ kubectl apply -f https://raw.githubusercontent.com/dotnet/dotnet-docker/main/samples/kubernetes/non-root/non-root.yaml
deployment.apps/dotnet-non-root created
service/dotnet-non-root created
$ kubectl get po
NAME                           READY   STATUS    RESTARTS   AGE
dotnet-non-root-68f4cd45c-687zp   1/1     Running   0          13s
```

앱이 실행 중입니다. 사용자를 확인해 보겠습니다.

```bash
$ kubectl exec dotnet-non-root-68f4cd45c-687zp -- whoami
app
```

앱에서 엔드포인트를 호출할 수도 있습니다. 먼저 엔드포인트에 대한 프록시를 만들어야 합니다.

```bash
% kubectl port-forward service/dotnet-non-root 8080
```

이제 사용자를 `app`으로 보고하는 엔드포인트를 호출할 수 있습니다.

```bash
% curl http://localhost:8080/Environment
{"runtimeVersion":".NET 8.0.0-preview.3.23174.8","osVersion":"Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022","osArchitecture":"Arm64","user":"app","processorCount":4,"totalAvailableMemoryBytes":4124512256,"memoryLimit":0,"memoryUsage":35004416}
```

리소스를 삭제합니다.

```bash
$ kubectl delete -f https://raw.githubusercontent.com/dotnet/dotnet-docker/main/samples/kubernetes/non-root/non-root.yaml
deployment.apps "dotnet-non-root" deleted
service "dotnet-non-root" deleted
```

이 글을 쓰는 시점에서 [공식 샘플](https://mcr.microsoft.com/product/dotnet/samples/about)은 아직 루트 사용자가 아닌 사용자를 사용하도록 전환하지 않았습니다. 샘플을 .NET 8로 옮길 때, 아마도 .NET 8 RC1에서 그렇게 할 것입니다. `aspnetapp` 이미지를 사용하여 루트를 사용하는 이미지에 `runAsNonRoot`를 사용할 때 어떤 일이 발생하는지 보여드릴 수 있습니다. 실패해야 하겠죠?

매니페스트를 조금 변경하겠습니다. `securityContext`섹션 없이 시작하겠습니다.

```yaml
    spec:
      containers:
      - name: aspnetapp
        image: mcr.microsoft.com/dotnet/samples:aspnetapp
```

사용자를 확인해 보겠습니다.

```bash
$ kubectl apply -f non-root.yaml
deployment.apps/dotnet-non-root created
service/dotnet-non-root created
$ kubectl get po
NAME                            READY   STATUS    RESTARTS   AGE
dotnet-non-root-85768f6c55-pb5gh   1/1     Running   0          1s
$ kubectl exec dotnet-non-root-85768f6c55-pb5gh -- whoami
root
```

예상했던 결과입니다. 이제 `runAsNonRoot`를 다시 추가해 보겠습니다.

```yaml
    spec:
      containers:
      - name: aspnetapp
        image: mcr.microsoft.com/dotnet/samples:aspnetapp
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
```

이 검사가 어떻게 작동하는지 살펴봅시다.

```bash
$ kubectl apply -f non-root.yaml
deployment.apps/dotnet-non-root created
service/dotnet-non-root created
$ kubectl get po
NAME                            READY   STATUS                       RESTARTS   AGE
dotnet-non-root-6df9cb77d8-74t96   0/1     CreateContainerConfigError   0          5s
```

저희가 원했던 대로 실패했습니다. 그 이유에 대해 좀 더 자세한 정보를 얻을 수 있습니다.

```bash
$ kubectl describe po | grep Error
      Reason:       CreateContainerConfigError
  Warning  Failed     7s (x2 over 8s)  kubelet            Error: container has runAsNonRoot and image will run as root (pod: "dotnet-non-root-6df9cb77d8-74t96_default(d4df0889-4a69-481a-adc4-56f41fb41c63)", container: aspnetapp)
```

파드에 `kubectl exec`을 시도할 수 있지만 실패합니다. 이는 쿠버네티스가 컨테이너가 생성되는 것을 차단했음을 보여줍니다(오류에 명시된 대로).

```bash
$ kubectl exec dotnet-non-root-6df9cb77d8-74t96 -- whoami
error: unable to upgrade connection: container not found ("aspnetapp")
```

## `dotnet-monitor`

[`dotnet-monitor`](https://github.com/dotnet/dotnet-monitor)는 실행 중인 애플리케이션에서 진단 아티팩트를 캡처하기 위한 진단 도구입니다. 저희는 이를 위한 [dotnet/monitor](https://hub.docker.com/_/microsoft-dotnet-monitor/) 컨테이너 이미지를 제공합니다. 비루트 호스팅에서도 잘 작동하나요? 예

[`hello-dotnet`](https://github.com/dotnet/dotnet-docker/blob/main/samples/kubernetes/hello-dotnet/README.md) 쿠버네티스 샘플은 비루트로 실행되는 ASP.NET과 `dotnet-monitor` 를 모두 보여줍니다. 또한 [클라우드](https://github.com/dotnet/dotnet-docker/blob/main/samples/kubernetes/hello-dotnet/README.md#monitor-with-prometheus)와 [로컬](https://github.com/dotnet/dotnet-docker/blob/main/samples/kubernetes/dotnet-monitor/README.md#monitor-with-prometheus) 모두에서 Prometheus 메트릭 데이터를 수집하는 데모도 이어집니다.

## 요약

몇 가지 간단한 구성 변경으로 쿠버네티스에서 비-루트 호스팅으로 전환할 수 있습니다. 앱의 보안이 강화되고 공격에 대한 복원력이 향상됩니다. 이 접근 방식은 또한 앱이 쿠버네티스 포드 강화 모범 사례를 준수하도록 합니다. 심층적인 방어에 큰 영향을 미치는 작은 변화입니다.

컨테이너 보안 계획을 통해 전체 .NET 컨테이너 에코시스템이 비-루트 호스팅으로 전환될 수 있기를 바랍니다. Microsoft는 클라우드의 .NET 앱이 고성능과 안전성을 갖출 수 있도록 투자하고 있습니다.

.NET 8 및 .NET에 제공되는 기타 기능에 대해 자세히 알아보려면 [dot.net/next](https://dotnet.microsoft.com/ko-kr/next) 에서 확인하세요.

---

%[https://devblogs.microsoft.com/dotnet/running-nonroot-kubernetes-with-dotnet/]