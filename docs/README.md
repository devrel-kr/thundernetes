---
layout: default
title: 홈
nav_order: 1
description: "Thundernetes는 여러분의 게임 서버를 Kubernetes에서 쉽게 실행하도록 해줍니다."
permalink: /
---

# Thundernetes 

Thundernetes에 오신 것을 환영합니다! Azure/XBOX 팀에서 출발한 오픈 소스 프로젝트로, Linux 게임 서버들을 Kubernetes 클러스터에서 실행하는 것을 가능하게 해 줍니다!

:exclamation: 최신 릴리스 정보: [![GitHub release](https://img.shields.io/github/release/playfab/thundernetes.svg)](https://github.com/playfab/thundernetes/releases)

## 사전에 필요한 지식

Kubernetes 또는 컨테이너가 새롭게 느껴진다면 [사전 지식](prerequisites.md) 문서를 통해 Thundernetes 내 동작하는 기술을 사용하기 위한 지식 틈을 줄여볼 수 있을 것입니다.

## 요구 사항

Thundernetes를 위해 필요한 사항입니다:

- Kubernetes 클러스터 (온프레미스 또는 퍼블릭 클라우드 프로바이터). 이상적으로는, 클러스터에는 외부에서 들어오는 접속을 허용하기 위해 노드 당 공용 IP 하나를 지원해야 합니다.
- 게임 서버 
  - 오픈 소스인 [Game Server SDK](https://github.com/playfab/gsdk) (GSDK)와 통합되어야 합니다. GSDK는 몇 해 동안 [Azure PlayFab Multiplayer Servers service](https://docs.microsoft.com/gaming/playfab/features/multiplayer/servers/) 에서 여러 AAA 타이틀로 배틀 테스트를 거쳤으며, Unity, Unreal, C#, C++, Java, Go와 같은 여러 대중적인 프로그래밍 언어 및 게임 엔진을 지원합니다.
  - Linux 컨테이너 이미지로 빌드되어야 합니다. 해당 이미지는 Kubernetes 클러스터가 액세스 가능한 컨테이너 레지스트리에 배포가 되어야 합니다.

> **_참고_**: [Wrapper 샘플](usingwrapper.md)을 사용하여 GSDK와 통합하는 단계를 피할 수도 있습니다. 이 샘플은 Thundernetes와 실험적으로 통합할 때 훌륭하지만, 적절한 GSDK 통합을 매우 권장합니다.

## 빠른 시작(Quickstart)

[빠른 시작](quickstart.md) 문서를 통해 클러스터에 Thundernetes를 설치하고 샘플 게임 서버를 실행하여 Thundernetes가 정상적으로 동작하는지 확안하세요.

아래 이미지를 클릭하여 Thundernetes를 설치하고 사용하는 것이 얼마나 쉬운지 확인하세요 (영문):

[![asciicast](https://asciinema.org/a/438455.svg)](https://asciinema.org/a/438455)

## 추천하는 링크

- [Wrapper 사용하기](usingwrapper.md) - GSDK와 통합하지 않고 Thundernetes에서 게임 서버를 실행하기 위해 프로세스를 래핑하는 방법
- [게임 서버](yourgameserver.md) - Thundernetes를 전용 게임 서버로 사용하는 방법
- [GameServerBuild 정의하기](gameserverbuild.md) - GameServerBuild를 YAML로 정의하는 방법
- [게임 서버 수명주기](gameserverlifecycle.md) - 게임 서버 프로세스 수명주기(lifecycle)
- [아키텍처](architecture.md) - Thundernetes 아키텍처 개요
- [트러블슈팅 가이드](troubleshooting/README.md) - Thundernetes 트러블슈팅 가이드에 대한 모든 퍼블릭 저장소
- [개발 노트](development.md) - Thundernetes에 컨트리뷰션하고자 할 때 유용한 개발 노트입니다
- [자주 묻는 질문 (FAQ)](FAQ.md) - 자주 묻는 질문 모음입니다.

## 기여하기

본 저장소는 한글 번역에 대한 기여만을 받습니다. Thundernetes에 대한 기여는 아래 영문 내용을 참고하세요.

If you are interested in contributing to Thundernetes, please read our [Contributing Guide](contributing.md) and open a PR. We'd be more than happy to help you out!
