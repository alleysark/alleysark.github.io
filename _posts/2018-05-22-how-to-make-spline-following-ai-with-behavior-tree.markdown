---
layout: post
title: "Behavior Tree로 스플라인 팔로잉 AI 만들기"
date: 2018-05-22 19:42:15
author: "alleysark"
image: '/assets/images/'
tags: [ue4, game programming, ai]
---

AI 에이전트가 정해진 경로를 패트롤하는 기능은 게임에서 굉장히 자주 요구되는 기능입니다. Unreal Engine에서는 [AIMoveTo][1] 기능을 활용해 정해진 포인트를 순차적으로 움직이는 AI 행동을 구현할 수 있습니다.

그런데 단순히 포인트와 포인트 사이를 직진이동하는 게 아니라, 디자이너가 정한 곡선을 부드럽게 따라 움직이는 AI는 어떻게 구현해야할까요?

이런 요구 조건에 흔히 Bezier curve, 혹은 Spline을 떠올릴 수 있습니다. 이번 포스팅에서는 스플라인을 따라 움직이는 AI를 만들어 볼텐데, Behavior Tree에 사용 가능한 태스크 형태로 작성하고 블랙보드를 통해 어떻게 이를 처리하는지 직접 구현해보겠습니다.

## Path 정의
가장 먼저 에이전트가 따라 움직일 경로인 Path 객체를 만들어야 합니다. 새 블루프린트 액터를 생성하고 이름을 **Path**라 지정합니다. 그리고 [*Spline Component*][2]를 추가해줍니다.

![Add USplineComponent into Path Actor]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/Path-component-list.png)

사실 Path 정의는 이 정도로 충분하지만, 스플라인 팔로잉에 필요한 기능을 Path에 구현해두면 앞으로 사용하기 더 편할 것입니다. 그래서 Path 액터에 몇 가지 기능을 추가하겠습니다.

### 특정 스플라인 시간의 위치 구하기
![GetPathLocationAtTime]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/GetPathLocationAtTime.png)

[USplineComponent][2]의 기능 중 [GetLocationAtTime][3]은 주어진 스플라인 시간으로 스플라인의 패스 포인트들을 보간한 위치를 반환합니다. 그런데 이 함수는 Coordinate Spate나 Constant Velocity 여부를 지정해줘야해서 번거롭습니다. 또한 외부에서 Spline 컴포넌트로의 접근을 제한하는 목적으로 Path 액터에 GetLocationAtTime 함수를 작성해줍니다.

### 특정 위치에 가장 인접한 스플라인 위치 구하기

![FindTimeClosestToWorldLocation]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/FindTimeClosestToWorldLocation.png)

스플라인 팔로잉을 수행하기 위한 중요한 기능이 하나 있습니다. 바로 월드 위치에서 가장 인접한 스플라인의 위치를 찾는 기능입니다. [USplineComponent][2]에는 [FindInputKeyClosestToWorldLocation][4] 기능이 있습니다만, 이는 스플라인의 InputKey를 반환합니다. 

> 스플라인의 InputKey는 인접한 두 스플라인 패스 포인트로 이루어진 패스 세그먼트의 키를 보간한 값입니다. 반면 스플라인 시간은 0에서 부터 전체 스플라인 커브의 Duration까지를 범위로 하는 조금 더 포괄적인 값입니다.

InputKey를 사용해 스플라인을 보간하면 각 패스 세그먼트의 길이에 따라 균질하지 않은 보간이 됩니다. 따라서 인접 위치의 InputKey를 스플라인 시간으로 역연산하여 관리하고, [GetLocationAtTime][3]시 *bUseConstantVelocity* 옵션을 주어 보간 속도가 연속적일 수 있도록 해야합니다.

> UE4의 USplineComponent는 스플라인 세그먼트 시퀀스의 보간 속도 연속성을 유지하는 방법으로 [Reparameterization Table][5] 기법을 사용합니다. 

### 팔로잉 에이전트와 적정 거리 유지하기
![ComputeTimeSeparatedFrom]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/ComputeTimeSeparatedFrom.png)

한 가지 도움이 될 기능은 스플라인 위의 위치 중 팔로잉 에이전트와 적정 거리를 유지하는 스플라인 시간 어드밴스 기능입니다. 뒤에 자세히 설명할 패스 팔로잉 로직의 큰 흐름은 스플라인상 특정 위치를 구하고 해당 위치로 이동하는 것인데, 팔로잉 에이전트가 스플라인 위치에 너무 가까워지면 이동 명령이 종료되기 때문에 '적당한 간격을 유지하는' 스플라인 시간 갱신 기능이 필요합니다. 위의 블루프린트 노드는 이런 요구 사항을 구현한 것입니다.

에이전트별 현재 스플라인 시간은 각 에이전트의 블랙보드에 기록될 것이므로 Path 액터는 적정 거리를 유지한 채 다음 스플라인 시간을 구해주는 기능만을 작성합니다.

물론 스플라인 팔로잉을 위해 이런 구현이 꼭 필요한 것은 아닙니다. 그다지 효율적이지 못하기도 하고, 에이전트의 속력이 급격히 변하는 시스템 혹은 스플라인이 굉장히 복잡하게 꼬여있는 상황에는 제대로 대응하지 못합니다. 그렇지만 패트롤링 하는 수준의 AI에는 괜찮은 접근 방법이라고 생각합니다.

## AI 에이전트 정의
AI 에이전트는 앞으로 작성할 Behavior Tree를 통해 제어될 행위자 캐릭터입니다. *Character* 혹은 *Pawn* 클래스를 상속하는 새로운 블루프린트 액터를 만들어줍니다.

에이전트는 특별한 기능은 없습니다. 다만, 패스 팔로잉할 액터를 변수로 선언해줍니다.

![Agent Following Path Variable]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/Agent-variable-pannel.png)

그 다음 클래스 디폴트 디테일 패널에서 뒤에 정의할 AI Controller를 AI Controller Class로 지정합니다.

## Behavior Tree 작성
이제 본격적으로 스플라인 팔로잉을 수행하는 Behavior Tree를 작성해봅시다. 먼저 가장 큰 틀인 Behavior Tree를 생성합니다.

![Add New Behavior Tree]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/NewBT.png)

그리고 블랙보드도 같이 만들고 새로 만든 Behavior Tree의 블랙보드로 설정합니다.

여기서 작성할 Behavior Tree는 아래와 같은 흐름으로 동작합니다.

1. Following 상태인가?
2. 현재 에이전트 위치에서 가장 가까운 패스 지점을 찾는다
3. 해당 지점부터 패스를 따라간다 <- 다른 액션이 있기까지 이 태스크에 머무른다

![Patrol Behavior Tree]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/BT.png)

다르게 작성할 수도 있겠지만, 가급적 MoveTo 태스크와 유사하게 팔로잉 로직에 액션 상태가 머무를 수 있도록 했습니다.

### Blackboard 키 설정
Behavior Tree는 태스크간 정보 전달의 매개체로 블랙보드를 사용합니다. 블랙보드가 조금 쓰기 번거로워서 캐릭터나 컨트롤러에 변수를 넣기도 하는데, Behavior Tree를 제대로 활용하고자 한다면 블랙보드를 통해 정보 접근을 하는 것이 좋습니다.

> 다수의 에이전트끼리 서로 정보를 공유하는 집단 행동을 처리할 때 블랙보드의 진가를 맛볼 수 있습니다.

![Blackboard key settings]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/Blackboard-key-list.png)

주요 블랙보드 키는 UObject 타입의 PathToFollow와 float 타입의 CurSplineTime입니다. PathToFollow는 Behavior Tree를 이용할 에이전트가 따라갈 Path 액터 인스턴스를 저장하고, CurSplineTime은 에이전트가 현재 따라가고 있는 스플라인 위치의 스플라인 시간을 나타냅니다.

### FindClosestPathTime Task 작성
새 Behavior Tree 태스크를 작성해야 합니다. 
![Create new BT Task]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/Create-New-BTTask.png)

BTTask_BlueprintBase 클래스를 상속해 구현하면 됩니다. 이와 비슷하게 커스텀 서비스나 커스텀 데코레이터도 쉽게 만들 수 있습니다.

![BT Task FindClosestPathTime]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/BTTask_FindClosestPathTime.png)

이 태스크는 패스 팔로잉 수행 전 초기 팔로잉 위치를 갱신하는 역할을 수행합니다. 앞서 구현해둔 Path 액터의 *FindTimeClosestToWorldLocation* 함수를 사용해 PathFollow 태스크 수행 전 초기 스플라인 위치를 구해두는 것입니다. 스플라인 시간을 잘 찾았으면 Finish Execute하여 Behavior Tree의 부모 노드가 다음 처리를 수행할 수 있도록 합니다.

태스크를 수행하기 위해 앞서 정의한 블랙보드 값을 가져와야 하므로 PathToFollow와 SplineTime에 대한 BlackboardKeySelector 변수를 선언하고 '편집 가능' 토글을 활성화 해둡니다.

![BT Task Variables]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/BTTask-variables.png)

함수의 DistanceThreshold 값과 UpdatePrecision 값은 적당히 넣어주거나, 에이전트에 정의해둘 수 있습니다.

이 태스크를 구현함으로써 PathFollow 태스크가 모종의 이유로 중단되었다 해도, 다시 팔로잉을 수행할 때 현재 위치로부터 가까운 패스 위치로 복귀할 수 있습니다.

### FollowPath Task 작성
이제 스플라인 팔로잉의 핵심 태스크인 FollowPath 태스크를 작성합니다.

![BT Task FollowPath Execute]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/BTTask_FollowPath_Execute.png)

![BT Task FollowPath Tick]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/BTTask_FollowPath_Tick.png)

Execute 이벤트에서는 사용할 Path 인스턴스를 저장해둡니다. (사실 Tick에서 블랙보드 키로 매번 구해와도 됩니다. 이게 더 좋습니다.)

Tick 이벤트에서는 블랙보드에 기록된 스플라인 시간의 위치를 구해와 [SimpleMoveToLocation][6] 하였습니다. AIMoveTo 노드와 다르게 폰을 타겟 위치로 이동시키는 아주 기초적인 기능을 수행하는 노드입니다. 복잡한 행동이 없기 때문에 고도화된 팔로잉 로직을 직접 관리하는 현재 상황에 가장 적합합니다.

이 후 다음 틱의 팔로잉 처리를 위해 팔로잉 에이전트로부터 적정 거리 떨어뜨린 스플라인 시간을 구하고 블랙보드의 값을 갱신합니다.

## AI Controller 정의
열심히 만든 Behavior Tree를 사용하기 위해서는 새로운 AI Controller를 정의할 필요가 있습니다. *AIController*를 부모 클래스로 하여 새 블루프린트 액터를 생성해줍니다.

*OnPossess* 이벤트를 오버라이드하여 아래와 같이 Behavior Tree를 실행해주면 됩니다.

![AI Controller OnPossess]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/AIController-OnPossess.png)

마지막으로 블랙보드의 PathToFollow를 현재 포즈된 에이전트가 참조하는 패스 인스턴스로 설정해주고,

![AI Controller Set Path]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/AIController-SetFollowingPath-BB.png)

패트롤이 필요한 시점에 IsFollowing 블랙보드 값을 설정하면 동작할 것입니다.

![AI Controller Set IsFollowing]({{site.url}}/assets/posts/how-to-make-spline-following-ai-with-behavior-tree/AIController-SetIsFollowing-BB.png)

## 테스트
적당히 맵을 만들고 내비메시 바운드 볼륨을 설치합니다. 그리고 Path 액터를 맵에 배치해 에이전트가 패트롤 할 경로를 만들어줍니다. 에이전트를 월드에 배치하고 팔로잉할 패스를 지정한 뒤 실행을 하면, 짜잔~

https://youtu.be/bDpmNHPPrKg

위 코드를 기반으로 조금 더 다양한 패트롤 행동들을 구현할 수도 있을 것입니다.



[1]: http://api.unrealengine.com/INT/BlueprintAPI/AI/AIMoveTo/index.html
[2]: http://api.unrealengine.com/INT/API/Runtime/Engine/Components/USplineComponent/index.html
[3]: http://api.unrealengine.com/INT/API/Runtime/Engine/Components/USplineComponent/GetLocationAtTime/index.html
[4]: http://api.unrealengine.com/INT/API/Runtime/Engine/Components/USplineComponent/FindInputKeyClosestToWorldLocati-/index.html
[5]: https://www.worldscientific.com/doi/abs/10.1142/S0218654396000075
[6]: http://api.unrealengine.com/INT/BlueprintAPI/AI/Navigation/SimpleMovetoLocation/index.html