---
layout: post
title: "PrimitiveDrawInterface로 시각화도구 만들기"
date: 2017-04-25 21:50:54
author: "alleysark"
image: '/assets/images/'
tags: [ue4, game programming, rendering]
---

게임을 만들다 보면 배포버전에 들어갈 것은 아니지만, 간단하게 디버깅을 위해서 혹은 에디터에서 시각적 도움을 주기위한 목적으로 3차원 정보를 표시하고싶을 경우가 있습니다.
보통의 경우라면 *DrawDebugLine*과 같은 기능을 사용하게됩니다. 블루프린트에서도 쓸 수 있고 사용하기도 간단하죠. 하지만 이 기능에는 문제도 많고 안되는 것도 많습니다. 복잡한 도형은 그리기 벅차기도 하죠.
이번 포스팅에서는 이를 대체할 수 있는 방법을 알아보려합니다.

## DrawDebug-류 기능의 실체
말씀드렸던 바와 같이 DrawDebug-류 함수는 사용하기 쉬워 자주 사용되죠, 하지만 몇가지 문제점이 있습니다.

* 우선 에디터상에서는 볼 수 없습니다. 무조건 플레이를 해봐야하죠.
* 또한 디버그빌드에서만 그려집니다. 배포버전에서는 그려지지 않습니다.
* 게임속도를 굉장히 느리게 만듭니다. 한 두개 그리는건 관계 없지만 스피어 몇십개만 그려도 랙이 생깁니다.

특히 마지막 속도저하 문제의 경우, DrawDebug- 류의 함수는 **ULineBatchComponent** 컴포넌트를 통해 그려지게 됩니다. 선/삼각형들을 먼저 추가하고 렌더 프록시에서 이들을 일괄 렌더링합니다. 
하지만 라인 하나하나에 대해 라이프타임을 관리하며 매 틱마다 이들을 확인해 시간이 다 된 라인들을 지우는 처리를 하게됩니다.

정말로 짧은 시간만 보여주고 말 경우라면 좋은 선택이지만, 에디터에서 혹은 게임씬에서 지속적으로 보여야 할 프리미티브에 대해 이런 방식의 선 관리는 너무 비효율적입니다.

## 그리기의 본질로
이런 문제가 많음에도 공식 문서에서 제시하는 대안은 없습니다.
그런데 한가지 의문이 드는것은, 에디터에 보이는 트리거박스의 와이어프레임이나, 스플라인 컴포넌트의 스플라인등은 어떻게 가시화되는걸까요? 스태틱 메시를 와이어프레임으로 렌더링하는건 아닌것같습니다.

이들 컴포넌트들을 살펴보면 **FPrimitiveSceneProxy**인 렌더스레드를 위한 프리미티브 렌더링 프록시 인터페이스 혹은 **FComponentVisualizer**와 같은 가시화 헬퍼 클래스를 통해 원하는 바를 그리고 있었습니다.
이것들은 단지 잘 포장된 껍데기일 뿐이니 정말로 그리기 요청을 수행해 주는 부분을 찾아야했습니다. 다행히 조금만 더 들어가보니 그 답이 나왔습니다.

바로 **PrimitiveDrawInterface**(PDI) 입니다. 프리미티브에 대한 그리기 요청을 수행해주는 인터페이스입니다.

> **FPrimitiveDrawInterface**는 Engine/Source/Runtime/Engine/Public/SceneManagement.h에 정의되어있는 추상 클래스 인터페이스입니다.

그런데 엔진 문서를 살펴봐도 PDI에 대한 언급은 찾아보기 힘듭니다. 소스를 파보며 뭐하는 아이인지 알아봐야겠습니다.

### Primitive Draw Interface
우선 PDI를 보니 요청받은 프리미티브를 그릴 대상 씬(scene)을 참조하고있습니다.
이와 함께 다이나믹 리소스를 등록하기 위한 *RegisterDynamicResource*함수와 그리기 요청을 위한 *DrawPoint*, *DrawLine*, *DrawSprite*, 그리고 *DrawMesh*와 같은 함수를 인터페이스로 제시합니다.
이 외에도 그려진 프리미티브에 대한 히트 판정을 위해 *SetHitProxy* HitProxy 등록함수 인터페이스도 제시하고있습니다.

PDI는 이름 그대로 그리기 '인터페이스'입니다. 씬 매니저는 이의 구현체인 **FSimpleElementCollector**를 제공합니다.
이는 그리기 요청된 프리미티브들을 다이나믹 리소스로써 관리하고, *DrawBatchedElements* 요청을 통해 씬 렌더러에서 일괄 렌더링됩니다.

PDI로 복잡한 도형을 그리고싶을 땐 SceneManagement.h에 정의되어있는 유틸리티 함수를 사용할 수 있습니다.
박스, 콘, 스피어, 캡슐 등 간단한 3D 프리미티브부터 탄젠트를 정의할 수 있는 디스크, 플랫 애로우, 아크, 점선 등등 굉장히 다양한 그리기 함수를 제공하고 있습니다.

> 이제와서 말하지만, DrawDebug-류 함수도 계속 들어가다보면 결국 PDI로 그리고 있습니다.

### PDI 사용하기
그렇다면 이 좋은걸 어떻게 써야할까요?

언리얼 엔진에서는 그려져야할 컴포넌트를 **UPrimitiveComponent**로 관리합니다. 씬 렌더러가 등록된 프리미티브 컴포넌트를 렌더 쓰레드에 그려달라고 요청하는 방식이죠.
프리미티브 컴포넌트의 경우 렌더링을 위한 **FPrimitiveSceneProxy** 타입의 씬 프록시(scene proxy)를 정의할 수 있습니다.

    virtual FPrimitiveSceneProxy* CreateSceneProxy();

이 함수는 씬이 새로운 씬 프록시를 요구할 때 호출됩니다. 씬 매니저의 판단에 따라 호출되거나 *MarkRenderStateDirty* 와 같은 함수로 명시적으로 재생성을 요구할 수 있습니다.

씬 프록시는 *GetDynamicMeshElements* 함수를 통해 이 프리미티브 컴포넌트의 그려져야 할 다이나믹 메시를 **FMeshElementCollector**에게 전달할 수 있습니다.
이 때 메시를 직접 전달하는 대신, 메시 콜렉터의 PDI를 받아 PDI를 통해 다이나믹 프리미티브 리소스를 생성해줄 수 있는것입니다.

### Draw3DAgentComponent
알았습니다. PDI로 그리면 짱 좋겠군요. 그런데 쓰기가 너무 불편합니다. 매번 프리미티브 컴포넌트의 씬 프록시를 정의해야한다니 불편함이 하늘을 뚫습니다.

그래서 3D 프리미티브를 그려주는 에이전트 컴포넌트를 만들어봤습니다. 이름하야 **Draw3DAgentComponent**! (열파참을 외치듯)

이 에이전트 컴포넌트의 역할은 복잡하지 않습니다. 프리미티브 컴포넌트를 상속하고 씬 프록시를 정의하지만, 다이나믹 메시를 '어떻게 정의해야하는가'를 사용하에게 위임하는 형태입니다.

이 컴포넌트를 사용하는 액터에서 *CreateDynMeshElemDelegate*에 어떻게 그릴지를 바인딩합니다. 그 후 전달받은 PDI로 원하는 프리미티브를 그리면 되는것입니다.

{% gist alleysark/cbf37b7e049fa3bfa35471dcc2efa354 %}

여기에 더해 C++ 코드를 사용하지 않고 블루프린트에서 그리는 방식을 정의할 수 있도록 *CreateDynamicElem* 함수를 BlueprintNativeEvent 함수로 만들어봤습니다. 사용할 때 Draw3DAgentComponent를 상속하여 컴포넌트를 만들고, 해당 함수를 오버라이딩한 뒤 인자로 주어지는 **FPrimitiveDrawProxy** PDI 프록시 인자를 통해 Draw 요청을 할 수 있습니다. 저는 블루프린트에서 쓸 일이 없어 기본적인 Draw함수만 감싸뒀습니다만, 원한다면 앞서 말했던 SceneManager에 정의되어있는 다양한 그리기 함수를 노출시킬 수 있을것입니다.

혹 이런기능이 필요하실 분을 위해 Draw3DAgentComponent의 코드 전문을 올려둡니다.

{% gist alleysark/b4a8f743ea4ea79715d0211a301f352b %}
{% gist alleysark/05e6077fedd78043a85f22607e2f0581 %}

## Component Visualizer
PDI를 직접 사용해 프리미티브를 그리는것 꽤나 흥미롭습니다. 그런데 이걸 사용해 복잡한 시각화 도구를 만드려니 머리가 아픕니다. 다행히 언리얼 엔진이 이러한 니즈를 충족시켜주기 위해 **FComponentVisualizer**를 제공하고있습니다. 이는 PDI를 잘 감싸 에디터 전용 컴포넌트 시각화 도구를 만들기 쉽게 해주는 에디터 모듈 컴포넌트입니다.

> FComponentVisualizer는 Engine/Source/Editor/UnrealEd/Public 아래 위치해있습니다.

스플라인 컴포넌트나 여타 에디터용 가시화 도구를 갖는 컴포넌트들이 이를 활용하고 있습니다. 씬 프록시를 통해 PDI를 직접 사용하는것에 비해 마우스나 키보드 커맨드도 관리해주고, 연결된 본래의 컴포넌트 프로퍼티의 갱신도 처리해주기 때문에 프리미티브의 렌더링이 목적이 아니라 컴포넌트의 시각화된 보조 도구를 구현해야하는 경우라면 Component visualizer를 추천합니다.

[Component visualizer 사용법](https://wiki.unrealengine.com/Component_Visualizers) 페이지를 보시면 많은 참고가 될 것 같습니다.

.. 사실 위 페이지만으로는 설명이 조금 많이 부실해서 Component visualizer에 대해 따로 글을 써볼 생각입니다.