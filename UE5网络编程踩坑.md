# 关于网络宏的使用

## UPROPERTY

### 	1. UPROPERTY(ReplicatedUsing = OnRep_NetV)（继承AActor）

由于继承AActor，必须在构造函数做如下声明`	bReplicates = true;`

此标识变量会网络同步，ReplicateUsing的特点是，当变量的值发生改变时，会调用指定的OnRep也叫RepNotify 函数。

```
	UPROPERTY(ReplicatedUsing = OnRep_NetV)
	FVector NetV;
```

必须要注意，函数名要与说明符等号后的指定的函数名相同`ReplicatedUsing=OnRep_NetV`
```
void AOPWProjectileBase::OnRep_NetV()
{
	MovementComponent->Velocity = NetV;
}
```
也必须要实现GetLifetimeReplicatedProps函数，否则编译报错

使用宏DOREPLIFETIME，必须包含头文件` #include "Net/UnrealNetwork.h"`

```
void AOPWProjectileBase::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);
	DOREPLIFETIME(AOPWProjectileBase, NetV);
}
```

### 2. UPROPERTY(ReplicatedUsing = OnRep_Health)（继承UActorComponent）

由于继承UActorComponent，必须在构造函数做如下声明`	SetIsReplicatedByDefault(true);`

其余与上文无异

### 3. UPROPERTY(Replicated)

此标识变量会默认网络同步，但同上文一样需要在构造函数中添加`相应语句`，并实现`GetLifetimeReplicatedProps`函数



 [注]`GetLifetimeReplicatedProps`函数中也可以使用宏DOREPLIFETIME_CONDITION，指定变量的同步条件

```
void AActor::GetLifetimeReplicatedProps( TArray< FLifetimeProperty > & OutLifetimeProps ) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    //同步Actor类中的变量Health，仅同步给模拟玩家
    DOREPLIFETIME_CONDITION( AActor, Health，COND_SimulatedOnly);
}
```
这里介绍一下宏`DOREPLIFETIME`和宏`DOREPLIFETIME_CONDITION`
1.宏`DOREPLIFETIME`，参数只需要传入类名和需要同步的变量

2.宏`DOREPLIFETIME_CONDITION`，参数需要传入类名和需要同步的变量和同步条件

| 条件                    | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| COND_InitialOnly        | 该属性仅在初始数据组尝试发送                                 |
| COND_OwnerOnly          | 该属性仅发送至 actor 的所有者                                |
| COND_SkipOwner          | 该属性将发送至除所有者之外的每个连接                         |
| COND_SimulatedOnly      | 该属性仅发送至模拟 actor                                     |
| COND_AutonomousOnly     | 该属性仅发送给自治 actor                                     |
| COND_SimulatedOrPhysics | 该属性将发送至模拟或 `bRepPhysics`的 actor                   |
| COND_InitialOrOwner     | 该属性将发送初始数据包，或者发送至 actor 所有者              |
| COND_Custom             | 该属性没有特定条件，但需要通过 `SetCustomIsActiveOverride `得到开启/关闭能力 |



## UFUNCTION

### 	1.UFUNCTION(NetMulticast, Reliable)

```
UFUNCTION(NetMulticast, Reliable)
void ASkillBase::ProjectileHitting_Multicast(AActor* Inst, const TArray<AActor*>& HitActors, FVector ExplosionPoint, float CauseDamage);
```

这是一个带有多播功能的函数声明。具体解释如下：

- `UFUNCTION(NetMulticast, Reliable)`: 这是一个宏定义，用于声明具有多播功能的网络函数。`NetMulticast`表示该函数将在服务器端调用，并在所有相关的客户端上进行多播。`Reliable`表示这个多播是可靠的，即确保所有客户端都会接收到调用。这意味着该函数将在所有客户端和服务器上执行，并且执行顺序是不确定的。
- `void ASkillBase::ProjectileHitting_Multicast(AActor* Inst, const TArray<AActor*>& HitActors, FVector ExplosionPoint, float CauseDamage)`: 这是函数的声明部分，它接受几个参数：
  - `Inst`: 发起技能的角色（Actor）实例。
  - `HitActors`: 被技能击中的角色（Actor）实例的数组。
  - `ExplosionPoint`: 技能爆炸的位置。
  - `CauseDamage`: 技能造成的伤害值。

接下来是函数的实现部分：

```
cppCopy codevoid ASkillBase::ProjectileHitting_Multicast_Implementation(AActor* Inst, const TArray<AActor*>& HitActors, FVector ExplosionPoint, float CauseDamage)
{
	// 创建一个Transform，设置其位置为爆炸点
	FTransform Transform;
	Transform.SetLocation(ExplosionPoint);
	// 在该位置生成一个特效
	UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), ImpactEffect, Transform);

	// 遍历被技能击中的角色数组
	for (auto& HitActor : HitActors)
	{
		// 判断被击中的角色是否实现了ISkillHitInterface接口
		ISkillHitInterface* HitI = Cast<ISkillHitInterface>(HitActor);
		if (HitI) HitI->OnSkillHitting(Inst); // 调用被击中角色的OnSkillHitting函数

		// 对被击中的角色应用伤害
		UGameplayStatics::ApplyDamage(HitActor, CauseDamage, nullptr, Inst, UDamageType::StaticClass());
	}
}
```

----

### 2.UFUNCTION(NetMulticast, Unreliable)

```
UFUNCTION(NetMulticast, Unreliable)
void AGunBase::OnShotHitting_Multicast(AActor* Inst, AActor* HitActor, FVector ImpactPoint, float CauseDamage);
```

这是一个带有多播功能的函数声明，具体解释如下：

- `UFUNCTION(NetMulticast, Unreliable)`: 这是一个宏定义，用于声明具有多播功能的网络函数。`NetMulticast`表示该函数将在服务器端调用，并在所有相关的客户端上进行多播。`Unreliable`表示该多播是不可靠的，意味着可能会丢失某些调用。这意味着该函数将在所有客户端和服务器上执行，并且执行顺序是不确定的。
- `void AGunBase::OnShotHitting_Multicast_Implementation(AActor* Inst, AActor* HitActor, FVector ImpactPoint, float CauseDamage)`: 这是函数的实现部分，它接受几个参数：
  - `Inst`: 开枪的角色（Actor）实例。
  - `HitActor`: 被子弹击中的角色（Actor）实例。
  - `ImpactPoint`: 子弹击中的位置。
  - `CauseDamage`: 子弹造成的伤害值。

```
cppCopy codevoid AGunBase::OnShotHitting_Locally(AActor* Inst, AActor* HitActor, FVector ImpactPoint, float CauseDamage)
{
	FVector MuzzleLocation = GetMesh()->GetSocketLocation("Muzzle");

	if (HitActor)
		DrawDebugLine(GetWorld(), MuzzleLocation, ImpactPoint, FColor::Green, false, 2.0f, 0, 1.0f);
	else
		DrawDebugLine(GetWorld(), MuzzleLocation, ImpactPoint, FColor::Red, false, 2.0f, 0, 1.0f);

	IGunHitInterface* Impl = Cast<IGunHitInterface>(HitActor);
	if (Impl)
	{
		Impl->OnShotHitting(GetInstigator(), ImpactPoint);
	}
	else
	{
		DrawDebugPoint(GetWorld(), ImpactPoint, 10, FColor::Blue, false, 2);
	}

	UGameplayStatics::ApplyDamage(HitActor, CauseDamage, nullptr, this, UDamageType::StaticClass());
}
```

这是`AGunBase`类中两个函数的实现部分。具体解释如下：

- ```
  void AGunBase::OnShotHitting_Locally(AActor* Inst, AActor* HitActor, FVector ImpactPoint, float CauseDamage)
  ```

  : 这个函数用于处理本地的子弹击中效果。具体功能包括：

  - 获取枪支模型的"Muzzle"插槽的位置作为枪口位置。
  - 如果击中的角色存在，则绘制一条绿色的调试线段，表示子弹从枪口位置到击中位置的路径。否则，绘制一条红色的调试线段，表示子弹未击中目标。
  - 如果被击中的角色实现了`IGunHitInterface`接口，通过`Cast`将其转换为`IGunHitInterface`类型的指针，并调用其`OnShotHitting`函数，传递发起开枪的角色（Inst）和击中位置（ImpactPoint）作为参数。
  - 如果被击中的角色没有实现`IGunHitInterface`接口，通过`DrawDebugPoint`在击中位置绘制一个蓝色的调试点，表示子弹击中了非角色物体。
  - 使用`UGameplayStatics::ApplyDamage`对被击中的角色应用伤害，传递伤害值（CauseDamage）、发起开枪的角色（this）和伤害类型（UDamageType::StaticClass()）作为参数。

---

###  3.UFUNCTION(Server, Reliable)

```
UFUNCTION(Server, Reliable)
void CastSkill_ServerRPC();
```

这是一个服务器函数的声明，具体解释如下：

- `UFUNCTION(Server, Reliable)`: 这是一个宏定义，用于声明服务器函数。`Server`表示该函数只在服务器端执行，不能在客户端调用。`Reliable`表示该函数是可靠的，即确保服务器会接收到该函数的调用。这意味着该函数将在服务器上执行，并且执行顺序是确定的。

```
cppCopy codevoid USkillComponent::CastSkill_ServerRPC_Implementation()
{
	ACharacter* Character = GetCharacter();
	if (Character->GetLocalRole() == ROLE_Authority)
	{
		FTransform Transform(Character->GetControlRotation());
		Transform.SetLocation(Character->GetMesh()->GetSocketLocation("SkillSocket"));

		//GEngine->AddOnScreenDebugMessage(-1, 10, FColor::Red, Transform.ToString());
		AActor* Target = FindFirstTarget();

		SkillArray[0]->CastSkillWithLockedTarget(Transform, Target);
	}
}
```

这是`CastSkill_ServerRPC`函数的实现部分。具体解释如下：

- 获取角色（Character）的指针。
- 检查角色的本地角色（LocalRole），如果是服务器（ROLE_Authority），则执行以下操作：
  - 创建一个`FTransform`对象，设置其旋转为角色的控制旋转（Character->GetControlRotation()）。
  - 设置该`FTransform`对象的位置为角色模型的"SkillSocket"插槽的位置。
  - 找到第一个目标（Target）。
  - 调用`SkillArray[0]`中的技能对象的`CastSkillWithLockedTarget`函数，传递`Transform`和`Target`作为参数。

---



