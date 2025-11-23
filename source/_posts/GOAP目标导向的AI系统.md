---
title: GOAP目标导向的AI系统
photos: images/GOAPfengmina.png
date: 2025-11-23 17:14:35
tags: [学习,AAA]
categories: 游戏技术
---
# 关于GOAP
首先要说明的是GOAP是一个目标导向的行为规划框架，你可以理解为把行为树中，每次判断是否执行当前行为的if条件，以及要执行的条件部分，两个拆开来管理的框架。

更多详细的介绍可以参考这位给的解释
>https://www.cnblogs.com/FlyingZiming/p/17274602.html#%E9%9A%BE%E7%82%B9%E4%B8%8E%E6%8C%91%E6%88%98

# 其次是关于实现，倒不如说本文侧重阐述实现的苦难....
首先要说明的是，GOAP本身的资料相对比较少，教程貌似只有一个在Unity上的实现，所以要参考网上的实现就只有一种方式，阅读源码并理解，但很可惜的项目组并没有给我太多的时间，而且我要在UE上实现，所以我的实现其实是我自己想的，目前也有些问题，但总归思路是对的，在此提醒。

![images1](images/个人goap.PNG "goap")  

## 大概的流程是这样的

1. GoapState具有所有的世界状态
2. 通过GoapSensor去感知到所有的状态
3. GoapPlanner根据所有的状态，通过当前的Goapgoal即目标，以及可以使用的GoapAction去规划要执行的plan
4. GoapAgentComponent根据找到的plan去执行详细的路径

## 实现部分，目前的代码是无法正常跑的，所以有感兴趣的朋友可以斧正下，因为本身是项目需要，也并非本人兴趣所在，所以最后转交给其他人来做这部分了，烂尾系列！我在下面标注了实际问题在哪
### GoapState
```C++
#include "GOAP/GoapState.h"

void FGoapState::SetState(const FName& Key, bool Value)
{
	Values.Add(Key, Value);
}

bool FGoapState::GetState(const FName& Key) const
{
	if (const bool* Found = Values.Find(Key))
	{
		return true;
	}
	return false;
}

bool FGoapState::HasState(const FName& Key) const
{
	return Values.Contains(Key);
}

void FGoapState::RemoveState(const FName& Key)
{
	Values.Remove(Key);
}

void FGoapState::Clear()
{
	Values.Empty();
}

bool FGoapState::Contains(const FGoapState& Other) const
{
	for (const TPair<FName, bool>& Pair : Other.Values)
	{
		const bool* Value = Values.Find(Pair.Key);
		if (!Value || *Value != Pair.Value)
		{
			return false;
		}
	}
	return true;
}

void FGoapState::Merge(const FGoapState& Other)
{
	for (const TPair<FName, bool>& Pair : Other.Values)
	{
		Values.Add(Pair.Key, Pair.Value);
	}
}
```
### GoapSensor
```C++
void UGoapSensor::InitWorldState(UGoapAgentComponent* Agent, FGoapState& WorldState)
{
//	CurrentWorldState.SetState("HasDrink", false);
//	CurrentWorldState.SetState("IsThirsty", true);
//	CurrentWorldState.SetState("HasFood", true);
//	CurrentWorldState.SetState("IsHungry", true);
//	CurrentWorldState.SetState("IsBored", false);
//	CurrentWorldState.SetState("HasFun", true);
//	CurrentWorldState.SetState("FullEnergy", false);
//
	if (WorldState != initWorldState)
	{
		WorldState = initWorldState;
	}
}
```

### GoapGoal

```C++
#include "GOAP/GoapGoal.h"

bool UGoapGoal::IsGoalSatisfied(const FGoapState& CurrentWorldState) const
{
	return CurrentWorldState.Contains(DesiredStates);
}


void UGoapGoal::AddDesiredState(FName Key, bool Value)
{
	DesiredStates.SetState(Key, Value);
}

void UGoapGoal::RemoveDesiredState(FName Key)
{
	DesiredStates.RemoveState(Key);
}
```

### GoapPlan

```C++
#include "GOAP/GoapPlan.h"

void GoapPlan::AddAction(UGoapAction* Action)
{
	if (Action)
	{
		ActionSequence.Insert(Action, 0); 
	}
}

void GoapPlan::Reset()
{
	ActionSequence.Reset();
	CurrentActionIndex = 0;
}

UGoapAction* GoapPlan::GetCurrentAction() const
{
	if (IsEmpty())
	{
		UE_LOG(LogTemp, Warning, TEXT("GOAP: EMPTY!!!!!!!!!!!!!!!!!!!!!!!!!!!!"));
	}
	if (ActionSequence.IsValidIndex(CurrentActionIndex))
	{
		return ActionSequence[CurrentActionIndex];
	}
	UE_LOG(LogTemp, Warning, TEXT("GOAP: Plan %s does not have Value ActionIndex."));
	return nullptr;
}

bool GoapPlan::IsEmpty() const
{
	return ActionSequence.Num() == 0;
}

bool GoapPlan::IsComplete() const
{
	return CurrentActionIndex >= ActionSequence.Num();
}

void GoapPlan::AdvanceAction()
{
	if (!IsComplete())
	{
		++CurrentActionIndex;
	}
}
```

### GoapAction

```C++
#include "GOAP/GoapAction.h"
#include "GOAP/GoapAgentComponent.h"
#include "Components/ACFActionsManagerComponent.h"
#include "GameplayTags.h"

void UGoapAction::InitAction()
{
	bIsDone = false;
}

bool UGoapAction::CheckProceduralPrecondition(AActor* Agent)
{
	if (!Agent) return false;

	UGoapAgentComponent* GoapComponent = Agent->FindComponentByClass<UGoapAgentComponent>();
	if (!GoapComponent)
	{
		UE_LOG(LogTemp, Warning, TEXT("GOAP:Agent %s does not have GoapAgentComponent."), *Agent->GetName());
		return false;
	}
	const FGoapState& WorldState = GoapComponent->GetCurrentWorldState();

	if (WorldState.Contains(Preconditions))
	{
		UE_LOG(LogTemp, Log, TEXT("GOAP: Action %s - All preconditions satisfied."), *GetName());
		return true;
	}

	UE_LOG(LogTemp, Log, TEXT("GOAP: Action %s -Failed Preconditions Actions."), *GetName());
	return false;
}

bool UGoapAction::Perform(AActor* Agent, float DeltaTime)
{
	UE_LOG(LogTemp, Log, TEXT("GOAP: Performing Action -> %s"), *GetName());
	UACFActionsManagerComponent* ActionsManager = Agent->FindComponentByClass<UACFActionsManagerComponent>();
	if (!ActionsManager)
	{
		UE_LOG(LogTemp, Error, TEXT("GOAP: Agent %s does not have ActionsManagerComponent."), *Agent->GetName());
		return false;
	}

	//FGameplayTag ActionTag = FGameplayTag::RequestGameplayTag(FName("Action.Attack"));
	//FGameplayTag ActionTag = FGameplayTag::RequestGameplayTag(FName("Action.Jumpback"));
	//FGameplayTag ActionTag = FGameplayTag::RequestGameplayTag(FName("Action.Hit"));
	//FGameplayTag ActionTag = FGameplayTag::RequestGameplayTag(FName("Action.Attack"));
	//ActionsManager->TriggerAction(ActionTag, EActionPriority::EMedium, false, FString(TEXT("GOAP_Action")));

	//UE_LOG(LogTemp, Log, TEXT("GOAP: Triggered action [%s] for agent %s"), *ActionTag.ToString(), *Agent->GetName());

	UE_LOG(LogTemp, Log, TEXT("GOAP: Triggered action  for agent %s"), *Agent->GetName());

	//ActionsManager->TriggerAction();

	bIsDone = true;
	return true;
}
```
### GoapPlanner
### 我写的版本其实问题就出在这，对于他的建图A*实现的不对，有很多错误
```C++

#include "GOAP/GoapPlanner.h"
#include "Algo/Reverse.h"

bool GoapPlanner::Plan(const TArray<UGoapAction*>& AvailableActions,
	UGoapGoal* Goal,
	const FGoapState& CurrentWorldState,
	AActor* Agent,
	GoapPlan& OutPlan)
{
	if (!Goal) return false;

	TArray<SearchNode> OpenList;
	TSet<FGoapState> ClosedSet;

	TArray<UGoapAction*> UsableActions;
	GetUsableActions(Agent, AvailableActions, UsableActions);

	SearchNode StartNode;
	StartNode.State = CurrentWorldState;
	StartNode.Cost = 0;
	OpenList.Heapify();
	OpenList.HeapPush(StartNode);

	while (!OpenList.IsEmpty())
	{
		SearchNode CurrentNode;
		OpenList.HeapPop(CurrentNode);

		if (Goal->IsGoalSatisfied(CurrentNode.State))
		{
			OutPlan.Reset();
			for (UGoapAction* Action : CurrentNode.Actions)
			{
				OutPlan.AddAction(Action);
			}

			UE_LOG(LogTemp, Warning, TEXT("Found plan for goal %s:"), *Goal->GetName());
			for (UGoapAction* Action : OutPlan.GetActions())
			{
				UE_LOG(LogTemp, Warning, TEXT("  %s"), *Action->GetName());
			}
			return true;
		}

		ClosedSet.Add(CurrentNode.State);

		for (UGoapAction* Action : UsableActions)
		{
			if (!CurrentNode.State.Contains(Action->GetPreconditions()))
			{
				continue;
			}

			FGoapState NewState = CurrentNode.State;
			NewState.Merge(Action->GetEffects());

			if (ClosedSet.Contains(NewState))
			{
				continue;
			}

			SearchNode NewNode;
			NewNode.State = NewState;
			NewNode.Actions = CurrentNode.Actions;
			NewNode.Actions.Add(Action);
			NewNode.Cost = CurrentNode.Cost + Action->GetCost();

			OpenList.HeapPush(NewNode);
		}
	}

	return false;
}

void GoapPlanner::GetUsableActions(AActor* Agent,const TArray<UGoapAction*>& AllActions, TArray<UGoapAction*>& OutUsable)
{
	for (UGoapAction* Action : AllActions)
	{
		if (Action && Action->CheckProceduralPrecondition(Agent))
		{
			OutUsable.Add(Action);
		}
	}
}

```

### 最后是Agent执行部分，GoapAgent作为一个Component挂在任意actor即可
这部分收Planner的影响其实已经有点烂掉了，写的非常混沌，而且这部分也有问题，要处理他的优先级部分，这部分没写对
```C++
#include "GOAP/GoapAgentComponent.h"

UGoapAgentComponent::UGoapAgentComponent()
{

	PrimaryComponentTick.bCanEverTick = true;

}


void UGoapAgentComponent::BeginPlay()
{
	Super::BeginPlay();

	Instantiate();
}


void UGoapAgentComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
	Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

	if (Sensor && bShouldSenseEnvironment)
	{
		Sensor->InitWorldState(this, CurrentWorldState);
		bShouldSenseEnvironment = false;
	}
	else if(!Sensor)
	{
		UE_LOG(LogTemp, Warning, TEXT("UGoapAgentComponent: Sensor is null"));
	}

	if (CheckPlanNeeded())
	{
		SelectGoal();	
		if (CurrentGoal)  
		{
			PerformPlanning();
		}
	}
	PerformAction(DeltaTime);
}


void UGoapAgentComponent::Instantiate()
{
	if (SensorClasses)
	{
		UGoapSensor* DefaultSensor = SensorClasses->GetDefaultObject<UGoapSensor>();
		if (DefaultSensor)
		{
			Sensor = DuplicateObject<UGoapSensor>(DefaultSensor, this);
		}
	}

	for (const auto& ActionClass : ActionClasses)
	{
		if (ActionClass)
		{
			UGoapAction* DefaultAction = ActionClass->GetDefaultObject<UGoapAction>();
			if (DefaultAction)
			{
				UGoapAction* Action = DuplicateObject<UGoapAction>(DefaultAction, this);
				if (Action)
				{
					ActionList.Add(Action);
				}
			}
		}
	}

	for (const auto& GoalClass : GoalClasses)
	{
		if (GoalClass)
		{
			UGoapGoal* DefaultGoal = GoalClass->GetDefaultObject<UGoapGoal>();
			if (DefaultGoal)
			{
				UGoapGoal* Goal = DuplicateObject<UGoapGoal>(DefaultGoal, this);
				if (Goal)
				{
					GoalList.Add(Goal);
					GoalAvailabilityMap.Add(Goal, true);
				}
			}
		}
	}

}

void UGoapAgentComponent::SelectGoal()
{
	TArray<UGoapGoal*> SortedGoals = GoalList;

	SortedGoals.Sort([](const UGoapGoal& A, const UGoapGoal& B)
		{
			return A.GetPriority() > B.GetPriority();
		});

	for (UGoapGoal* Goal : SortedGoals)
	{
		if (Goal
			&& !Goal->IsGoalSatisfied(CurrentWorldState)
			&& GoalAvailabilityMap.Contains(Goal)
			&& GoalAvailabilityMap[Goal])
		{
			CurrentGoal = Goal;
			return;
		}
	}
	UE_LOG(LogTemp, Warning, TEXT("GOAP:All Goal Can not find a plan Or Finsihed!!!!  in current State!!!!!!!!!!!!"));
	CurrentGoal = nullptr;
	return;
}

void UGoapAgentComponent::PerformPlanning()
{
	for (UGoapAction* Action : ActionList)
	{
		if (Action)
		{
			Action->SetIsDone(false);
		}
	}

	CurrentPlan.Reset();
	CurrentAction = nullptr;

	if (Planner.Plan(ActionList, CurrentGoal, CurrentWorldState, GetOwner(), CurrentPlan))
	{
		CurrentAction = CurrentPlan.GetCurrentAction();
		UE_LOG(LogTemp, Log, TEXT("GOAP: Planning succeeded. action: %s"),
			*GetNameSafe(CurrentAction));
		LastWorldState = CurrentWorldState;
		LastGoal = CurrentGoal;
	}
	else
	{
		UE_LOG(LogTemp, Warning, TEXT("GOAP: Planning failed for goal: %s"),
			*GetNameSafe(CurrentGoal));
		GoalAvailabilityMap[CurrentGoal] = false;
		return;
	}
}

void UGoapAgentComponent::PerformAction(float DeltaTime)
{

    if (!CurrentAction) return;

	if (CurrentAction->IsDone())
	{
		ApplyActionEffect(CurrentAction);

		for (UGoapGoal* Goal : GoalList)
		{
			GoalAvailabilityMap[Goal] = true;
		}
	
		AdvanceToNextAction();
		return;
	}

	if (!CurrentAction->CheckProceduralPrecondition(GetOwner()))
	{
		UE_LOG(LogTemp, Warning, TEXT("GOAP: Action precondition failed: %s"),
			*GetNameSafe(CurrentAction));

		CurrentPlan.Reset();
		CurrentAction = nullptr;
		return;
	}

	if (!CurrentAction->Perform(GetOwner(), DeltaTime))
	{
		UE_LOG(LogTemp, Warning, TEXT("GOAP: Action execution failed: %s"),
			*GetNameSafe(CurrentAction));

		CurrentPlan.Reset();
		CurrentAction = nullptr;
		return;
	}
}

void UGoapAgentComponent::ApplyActionEffect(UGoapAction* Action)
{
	if (!Action || !Sensor) return;

	const FGoapState& Effects = Action->GetEffects();

	for (const auto& Elem : Effects.GetValues())
	{
		CurrentWorldState.SetState(Elem.Key, Elem.Value);
	}

	FString StateLog = TEXT("CurrentWorldState after ApplyActionEffect:");
	for (const auto& Pair : CurrentWorldState.GetValues())
	{
		StateLog += FString::Printf(TEXT(" [%s: %s]"),
			*Pair.Key.ToString(),
			Pair.Value ? TEXT("true") : TEXT("false"));
	}

	UE_LOG(LogTemp, Warning, TEXT("%s"), *StateLog);
}

bool UGoapAgentComponent::CheckPlanNeeded()
{
	if (CurrentPlan.IsEmpty() )
	{
		return true;
	}

	if (CurrentPlan.IsComplete())
	{
		if (!CurrentGoal->IsGoalSatisfied(CurrentWorldState))
		{
			return true;
		}
		else
		{
			return false;
		}
	}
	if (LastWorldState != CurrentWorldState)
	{
		return true;
	}
	if (LastGoal != CurrentGoal)
	{
		return true;
	}
	return false;
}

void UGoapAgentComponent::AdvanceToNextAction()
{
	CurrentPlan.AdvanceAction();
	CurrentAction = CurrentPlan.GetCurrentAction();
}

```

## 以上代码结束，这边开始总结败因

### 首先是最重要的一点也是最困难的一点，你要先想好所有的Action

在Planner时，他一定会根据你的所有Action开始建图，每一个Action相当于节点，如果你没有办法划分这个的话他是无法很好的执行的，比如说你在跟4个敌人一起作战，敌人后跳后切换最前方的敌人，但实际的执行层Action如果要包括一个后跳就不会这样子划分，可能是Action1：距离300,向后移动 Action2：距离500， 丢失目标

所以你需要把他每一个行为的最小单元想清楚，这是最重要的，也是我没搞出来的原因。

### 第二点是WorldState

这部分是变化，因为你的state不可能是一直不变的，但如果要让他变化，单纯的Action是可以的，但他是有限的，所以对于不同的物体应该有一个不同的state去触发

即WorldState为什莫叫WorldState

### 接下来是代码总结

首先在GoapPlanner的A*部分，没有去正确的建图，如何建图这部分其实更像是把每一个动作拆分成最小单元集，这是一种在项目组内不断拆解功能后的一种能力，因为我本身是做引擎的，最近才开始在项目组内实现需求，这点能力有欠缺，相信屏幕前的你应该比我强(●ˇ∀ˇ●)

其次是在WorldState你可以看到目前是一个很空的东西，因为我是从最小开始做的，即一开始能够实现找到一个GoapGoal，并执行一个Action即可，所以后面没有让所有的WorldActor去拥有这样的State

然后是在Agent的执行层面上，非常混乱了，前面也提到优先级的部分，本身这部分其实相当于一个权重，在A*找路径的时候左右他的节点选择，但这个也是后加的，所以最后因为时间紧张+资料少+没想清楚，最后越写越乱。

## 总结：如果需求你感觉明显有问题，跟上司提工期有问题，而不是跟他说你这星期做不完