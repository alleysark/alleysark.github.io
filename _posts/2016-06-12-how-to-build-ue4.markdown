---
layout: post
title: "UE4 빌드하기"
date: 2016-06-12 21:11:27
author: "alleysark"
image: '/assets/images/'
tags: [ue4, game programming]
---

개인적으로 여러 게임 엔진들 중 언리얼 엔진이 가장 세련되었다고 생각합니다. 그런데 언리얼 엔진을 다뤄볼 일이 많지 않아 엔진 구조나 API를 잘 아는것도 아니고, 엔진 소스를 공개하고 있음에도 제대로 뜯어보지도 못해 애매하게 좋아하는 상태입죠. 그래서 시간이 나는 틈에 언리얼 엔진을 조금 깊이있게 파헤쳐보려합니다.

우선 언리얼 엔진에 가입을 하고, GitHub 계정을 연동시킨 후, 언리얼 엔진 저장소에 접근해서 코드를 내려받습니다. 자세한 내용은 [언리얼 엔진 소스 코드 내려받기](https://docs.unrealengine.com/latest/KOR/GettingStarted/DownloadingUnrealEngine/index.html) 페이지에서 확인하실 수 있습니다. 소스코드는 포크를 떠도 괜찮고 zip 파일로 다운로드 받아도 상관 없습니다. 저는 최근 릴리즈된 4.12 버전을 내려받았습니다.

# Unreal Engine 4 빌드하기
## Setup.bat 실행
언리얼 엔진을 빌드하기 위한 dependency file들을 설치해주는 배치 프로그램입니다. 실행하면 관련 파일들의 다운로드를 시작하고, 이는 몇 분 내로 끝납니다.

## GenerateProjectFiles.bat 실행
소스 코드를 열어보려면 Visual studio나 Xcode 프로젝트 파일을 실행해야겠죠? 그런데 초기 상태의 언리얼 엔진 저장소에는 프로젝트 파일이 포함되어있지 않습니다. 대신 GenerateProjectFiles.bat 배치 프로그램을 실행해서 환경에 알맞은 프로젝트 파일을 생성하도록 하고있죠. 자세한 내용은 [자동 프로젝트 파일 생성](https://docs.unrealengine.com/latest/KOR/Programming/UnrealBuildSystem/ProjectFileGenerator/index.html) 페이지에서 확인할 수 있습니다.

보통 CMake 파일을 제공해서 사용자가 프로젝트파일을 생성하는경우가 많은데, 언리얼은 독자적인 배치파일을 만들어 번거로움을 줄이고, 설정 실수로부터 오는 혼란을 막았습니다.

이 배치 프로그램은 초기에 한번 실행해주고, 이 후 새로운 모듈이나 코드 파일이 추가될 때에도 실행해줘야 합니다. 번거롭다고 생각하시는 분이 계실지도 모르겠습니다만, 고전적인 방법으로 동일한 일을 하려면 몇 배는 더 번거롭습니다 =ㅅ=

> Visual studio 2015를 기본 세팅으로 설치하시면 c++ 도구가 함께 설치되지 않습니다. 그러면 GenerateProjectFiles.bat 실행시 visual studio를 찾지 못한다는 에러를 뿜으니, 설치시 c++ 도구를 체크해주시거나, 이미 설치된 상태라면 '새 프로젝트 > Visual C++ > Windows 데스크탑용 Visual C++ 2015 도구'를 선택하여 따로 설치해주세요.

> GenerateProjectFiles를 실행할때 Avast가 간혹 문제를 일으키기도 한다고 합니다.이때는 Avast의 DeepScreen 기능을 꺼버리면 어찌 해결된다고는 하네요..

## 빌드하기
이제 UE4.sln을 실행해줍니다. 빌드 구성이 굉장히 다양해서, 이를 제대로 이해하려면 시행착오가 조금 필요해 보입니다. 일단 [소스에서 언리얼 엔진 빌드하기](https://docs.unrealengine.com/latest/KOR/Programming/Development/BuildingUnrealEngine/index.html) 페이지를 따라 솔루션 환경설정을 Development Editor로 설정하고, Win64 세팅으로 UE4 프로젝트를 빌드해줍니다. 빌드는 굉장히 오랜 시간이 소요되므로, 참지 못하고 빌드를 중지하는 불상사가 없으시길..