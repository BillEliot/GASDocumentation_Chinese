**【更新日期：2021/02/01】**  

原文地址: [tranek/GASDocumentation](https://github.com/BillEliot/GASDocumentation)  
翻译此文时间: 2020.12.21  

[TOC]

# GASDocumentation

我使用一个简单的多人游戏模板项目来阐述对Unreal Engine 4的GameplayAbilitySystem(GAS)插件的理解. 这不是官方文档并且这个项目和我自己也都不来自Epic Games. 我不能保证该文档的准确性.  

该文档的目的是阐明GAS中的主要概念和相关类, 并结合我的经验提供一些附加说明. 在社区用户中, 已经形成了大量有关GAS的"部落知识", 而我致力于将我了解的全部在这里分享.  

样例项目和文档目前基于`Unreal Engine 4.26`. 该文档拥有可用于旧版本Unreal Engine的分支, 但是它们不再受支持, 并且可能存在bug和过时信息.  
[GASShooter](https://github.com/tranek/GASShooter)是该样例项目的姐妹项目, 其演示了基于多人FPS/TPS的高级GAS技术.  

最好的文档永远是该插件的代码.  

## 1. 步入GameplayAbilitySystem插件

摘自官方文档:  
> Gameplay技能系统 是一个高度灵活的框架，可用于构建你可能会在RPG或MOBA游戏中看到的技能和属性类型。你可以构建可供游戏中的角色使用的动作或被动技能，使这些动作导致各种属性累积或损耗的状态效果，实现约束这些动作使用的"冷却"计时器或资源消耗，更改技能等级及每个技能等级的技能效果，激活粒子或音效，等等。简单来说，此系统可帮助你在任何现代RPG或MOBA游戏中设计、实现及高效关联各种游戏中的技能，既包括跳跃等简单技能，也包括你喜欢的角色的复杂技能集。
 
GameplayAbilitySystem插件由Epic Games开发, 随Unreal Engine 4 (UE4)发布. 它已经由3A商业游戏的严格测试, 例如帕拉贡(Paragon)和堡垒之夜(Fortnite)等等.  

该插件对于单人和多人游戏提供了开箱即用的解决方案:  

* 执行基于等级的角色能力(Ability)或技能(Skill), 该能力或技能可选花费和冷却时间. (`GameplayAbility`)
* 管理属于Actor的数值Attribute. (`Attribute`)
* 为Actor应用状态效果. (`GameplayEffect`)
* 为Actor应用GameplayTag. (`GameplayTag`)
* 生成视觉或声音效果. (`GameplayCue`)
* 为以上提到的所有应用同步(Replication).

在多人游戏中, GAS提供客户端预测(client-side prediction)支持:  

* 能力激活.
* 播放蒙太奇.
* 对`Attribute`的修改.
* 应用`GameplayTag`.
* 生成`GameplayCue`.
* 通过连接于`CharacterMovementComponent`的`RootMotionSource`函数形成的移动.

**GAS必须由C++创建**, 但是`GameplayAbility`和`GameplayEffect`可由设计师在蓝图中创建.  

GAS中的现存问题:

* `GameplayEffect`延迟调节(Latency Reconciliation).(不能预测能力冷却时间, 导致高延迟玩家相比低延迟玩家, 对于短冷却时间的能力有更低的激活速率.)
* 不能预测`GameplayEffect`的移除(Removal). 然而我们可以预测使用相反的效果添加`GameplayEffect`, 从而高效的移除它们. 但是这不总是合适或者可行的, 因此这仍然是个问题.
* 缺乏样例模板项目, 多人联机样例和文档. 希望这篇文档会有所帮助.

## 2. 样例项目

该文档包含一个支持多人联机的第三人称射击游戏模板项目, 其目标受众为初识`GameplayAbilitySystem`插件, 但并不是Unreal Engine 4新手. 用户应该了解C++, 蓝图, UMG, Replication和其他UE4的中间件. 该项目提供了一个样例, 其向你展示了如何使用`GameplayAbilitySystem`插件建立一个基础的支持多人联机的第三人称射击游戏, 其中`AbilitySystemComponent(ASC)`分别位于`PlayerState`类代表玩家/AI控制的人物和位于`Character`类代表AI控制的小兵.  
我在保证展现GAS基础和带有完整注释的代码所表示的一些普遍技能的同时, 尽力使这个样例足够简单. 由于该文档专注于初学者, 因此该样例不包含像`Predicting Projectiles`这样的高级技术.  
概念说明:  
* `ASC`位于`PlayerState`还是`Character`.
* 网络同步的`Attribute`.
* 网络同步的蒙太奇(Animation Montages).
* `GameplayTag`.
* 在`GameplayAbility`内部和外部应用和移除`GameplayEffect`.
* 应用被护甲防御后的伤害值来修改角色生命值.
* `GameplayEffectExecutionCalculations`.
* 眩晕效果.
* 死亡和重生.
* 在服务器上使用能力(Ability)生成抛射物(Projectile).
* 在瞄准和奔跑时, 预测性的修改本地玩家速度.
* 不断消耗耐力来奔跑.
* 消耗魔法值来使用能力(Ability).
* 被动能力(Ability).
* 堆栈`GameplayEffect`.
* 锁定Actor.
* 在蓝图中创建`GameplayAbility`.
* 在C++中创建`GameplayAbility`.
* 实例化每个Actor的`GameplayAbility`.
* 非实例化的`GameplayAbility`(Jump).
* 静态`GameplayCue`(子弹撞击粒子效果).
* Actor `GameplayCue`(奔跑和眩晕粒子效果).

角色类有如下能力:  
|能力|输入绑定|是否可预测|C++/Blueprint|描述|
|:-:|:-:|:-:|:-:|:-:|
|跳跃|空格键|Yes|C++|使角色跳跃.|
|枪|鼠标左键|No|C++|从角色的枪中发射投掷物, 发射动画是可预测的, 但是投掷物不能预测.|
|瞄准|鼠标右键|Yes|Blueprint|当按住鼠标右键时, 角色会走的更慢并且摄像机会拉近(zoom in)以获得更高的射击精度.|
|奔跑|左Shift|Yes|Blueprint|当按住左Shift时, 角色在消耗体力的同时跑得更快.|
|向前猛冲|Q|Yes|Blueprint|角色消耗体力的同时向前猛冲.|
|被动护盾叠加|被动|No|Blueprint|每过4s角色获得一个最大层数为4的护盾, 每次受到伤害时移除一层护盾.|
|流星坠落|R|No|Blueprint|角色锁定一个敌人召唤一个流星, 对其造成伤害和眩晕效果.|

`GameplayAbility`无论由蓝图还是C++创建都没关系. 这里我们使用蓝图和C++混合创建, 意在展示每种方式的使用方法.  
AI控制的小兵没有预先定义的`GameplayAbility`. 红方小兵有较多的生命回复, 蓝方小兵有较多的初始生命.  
对于`GameplayAbility`的命名, 我使用`_BP`后缀表示由蓝图创建的`GameplayAbility`逻辑, 没有后缀则表示由C++创建.  

**蓝图资源命名前缀**  
|Prefix|Asset Type|
|:-:|:-:|
|GA_|GameplayAbility|
|GC_|GameplayCue|
|GE_|GameplayEffect|

## 3. 使用GAS创建一个项目

使用GAS建立一个项目的基本步骤:  
1. 在编辑器中启用GameplayAbilitySystem插件.
2. 编辑`YourProjectName.Build.cs`, 添加`"GameplayAbilities"`, `"GameplayTags"`, `"GameplayTasks"`到你的`PrivateDependencyModuleNames`.
3. 刷新/重新生成Visual Studio项目文件.
4. 从4.24开始, 需要强制调用`UAbilitySystemGlobals::InitGlobalData()`来使用`TargetData`, 样例项目在`UEngineSubsystem::Initialize()`中调用该函数. 参阅`InitGlobalData()`获取更多信息.

这就是你启用GAS所需做的全部了. 从这里开始, 添加一个`ASC`和`AttributeSet`到你的`Character`或`PlayerState`, 并开始着手`GameplayAbility`和`GameplayEffect`!

## 4. GAS概念

### 4.1 Ability System Component

`AbilitySystemComponent(ASC)`是GAS的核心, 它是一个处理所有与该系统交互的`UActorComponent(UAbilitySystemComponent)`, 所有期望使用 `GameplayAbility`, 包含`Attribute`, 或者接受`GameplayEffect`的Actor都必须附加`ASC`. 这些对象都存于`ASC`并由其管理和同步(除了由`AttributeSet`同步的`Attribute`). 开发者最好但不强求继承该组件.  

`ASC`附加的`Actor`被引用作为该`ASC`的`OwnerActor`, 该`ASC`的物理代表`Actor`被称为`AvatarActor`. `OwnerActor`和`AvatarActor`可以是同一个 `Actor`, 比如MOBA游戏中的一个简单AI小兵; 它们也可以是不同的`Actor`, 比如MOBA游戏中玩家控制的英雄, 其中`OwnerActor`是`PlayerState`, `AvatarActor`是英雄的`Character`类. 绝大多数Actor的`ASC`都附加在其自身, 如果你的Actor会重生并且重生时需要持久化`Attribute`或`GameplayEffect`(比如MOBA中的英雄), 那么`ASC`理想的位置就是`PlayerState`.  

**Note:** 如果`ASC`位于PlayerState, 那么你需要增加PlayerState的`NetUpdateFrequency`, 其默认值是一个很低的值, 因此在客户端上发生像`Attribute`和`GameplayTag`改变时会造成延迟或卡顿. 确保启用`Adaptive Network Update Frequency`, Fortnite就启用了该项.  

如果OwnerActor和AvatarActor是不同的Actor, 应该执行`IAbilitySystemInterface`, 该接口有一个必须重写的函数, `UAbilitySystemComponent* GetAbilitySystemComponent() const`, 其返回一个指向`ASC`的指针, `ASC`通过寻找该接口函数来和系统内部进行交互.  

`ASC`在`FActiveGameplayEffectContainer ActiveGameplayEffect`中保存其当前活跃的`GameplayEffect`.  

`ASC`在`FGameplayAbilitySpecContainer ActivatableAbility`中保存其授予的`GameplayAbility`. 当你想遍历`ActivatableAbility.Items`时, 确保在循环体之上添加`ABILITYLIST_SCOPE_LOCK();`来锁定列表以防其改变(比如移除一个Ability). 每个域中的`ABILITYLIST_SCOPE_LOCK();`会增加`AbilityScopeLockCount`, 之后出域时会减量. 不要尝试在`ABILITYLIST_SCOPE_LOCK();`域中移除某个Ability(Ability删除函数会在内部检查`AbilityScopeLockCount`以防在列表锁定时移除Ability).  

#### 4.1.1 同步模式

`ASC`定义了三种不同的同步模式用于同步`GameplayEffect`, `GameplayTag`和`GameplayCue` - `Full`, `Mixed`和`Minimal`. `Attribute`由其`AttributeSet`同步.  

|同步模式|何时使用|描述|
|:-:|:-:|:-:|
|`Full`|单人|每个`GameplayEffect`同步到客户端.|
|`Mixed`|多人, 玩家控制的Actor|`GameplayEffect`只同步到其所属客户端, 只有`GameplayTag`和`GameplayCue`同步到所有客户端.|
|`Minimal`|多人, AI控制的Actor|`GameplayEffect`从不同步到任何客户端, 只有`GameplayTag`和`GameplayCue`同步到所有客户端.|

**Note:** `Mixed`同步模式需要`OwnerActor`的`Owner`是`Controller`. `PlayerState`的`Owner`默认是`Controller`但是`Character`不是. 如果`OwnerActor`不是`PlayerState`时使用`Mixed`同步模式, 那么需要在`OwnerActor`中调用`SetOwner()`设置`Controller`.  

从4.24开始, 需要使用`PossessedBy()`设置新的`Controller`为`Pawn`的Owner.  

#### 4.1.2 设置和初始化

`ASC`一般在`OwnerActor`的构建函数中构建并且需要明确标记为Replicated. **这必须在C++中完成.**  

```c++
AGDPlayerState::AGDPlayerState()
{
	// Create Ability system component, and set it to be explicitly replicated
	AbilitySystemComponent = CreateDefaultSubobject<UGDAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
	AbilitySystemComponent->SetIsReplicated(true);
	//...
}
```

`OwnerActor`和`AvatarActor`的`ASC`在服务端和客户端上均需初始化, 你应该在`Pawn`的`Controller`设置之后初始化(Possession之后), 单机游戏只需考虑服务端的做法.  

对于玩家控制的Character且`ASC`位于`Pawn`, 我一般在服务端`Pawn`的`PossessedBy()`函数中初始化, 在客户端`PlayerController`的`AcknowledgePossession()`函数中初始化.  

```c++
void APACharacterBase::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	if (AbilitySystemComponent)
	{
		AbilitySystemComponent->InitAbilityActorInfo(this, this);
	}

	// `ASC` MixedMode replication requires that the `ASC` Owner's Owner be the Controller.
	SetOwner(NewController);
}
```

```c++
void APAPlayerControllerBase::AcknowledgePossession(APawn* P)
{
	Super::AcknowledgePossession(P);

	APACharacterBase* CharacterBase = Cast<APACharacterBase>(P);
	if (CharacterBase)
	{
		CharacterBase->GetAbilitySystemComponent()->InitAbilityActorInfo(CharacterBase, CharacterBase);
	}

	//...
}
```

对于玩家控制的Character且`ASC`位于`PlayerState`, 我一般在服务端`Pawn`的`PossessedBy()`函数中初始化, 在客户端PlayerController的`OnRep_PlayerState()`函数中初始化, 这确保`PlayerState`存在于客户端上.  

```c++
void AGDHeroCharacter::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the `ASC` on the Server. Clients do this in OnRep_PlayerState()
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// AI won't have PlayerControllers so we can init again here just to be sure. No harm in initing twice for heroes that have PlayerControllers.
		PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
	}
	
	//...
}
```

```c++
// Client only
void AGDHeroCharacter::OnRep_PlayerState()
{
	Super::OnRep_PlayerState();

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the `ASC` for clients. Server does this in PossessedBy.
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// Init `ASC` Actor Info for clients. Server will init its `ASC` when it possesses a new Actor.
		AbilitySystemComponent->InitAbilityActorInfo(PS, this);
	}

	// ...
}
```

如果你得到了错误消息`LogAbilitySystem: Warning: Can't activate LocalOnly or LocalPredicted Ability %s when not local!`, 那么表明`ASC`没有在客户端中初始化.  

### 4.2 Gameplay Tags

`FGameplayTag`是由`GameplayTagManager`注册的形似`Parent.Child.Grandchild...`的层级Name, 这些标签对于分类和描述对象的状态非常有用, 例如, 如果某个Character处于眩晕状态, 我们可以给一个`State.Debuff.Stun`的`GameplayTag`.  

你会发现自己用`GameplayTag`替换了过去使用布尔值或枚举值处理的事情, 并且需要对对象有无特定的`GameplayTag`做布尔逻辑判断.  

当给某个对象设置标签时, 如果它有`ASC`的话, 我们一般添加标签到`ASC`, 因此GAS可以与其交互. UAbilitySystemComponent执行`IGameplayTagAssetInterface`接口的函数来访问其拥有的`GameplayTag`.  

多个`GameplayTag`可被保存于一个`F`GameplayTagContainer``中, 相比`TArray<FGameplayTag>`, 最好使用`GameplayTagContainer`, 因为`GameplayTagContainer`做了一些很有效率的优化. 因为标签是标准的`FName`, 所以当在project setting中启用`Fast Replication`后, 它们可以高效地打包进`F`GameplayTagContainer``以用于同步. `Fast Replication`要求服务端和客户端有相同的`GameplayTag`列表, 这通常不是问题, 因此你应该启用该选项. `GameplayTagContainer`也可以返回`TArray<FGameplayTag>`以用于遍历.  

保存于`FGameplayTagCountContainer`的`GameplayTag`有一个保存该`GameplayTag`实例数的`TagMap`. FGameplayTagCountContainer可能存有`TagMapCount`为0的`GameplayTag`, 如果`ASC`仍有一个`GameplayTag`, 你可能在Debug时遇到这种情况. 任何`HasTag()`或`HasMatchingTag()`或其他相似的函数会检查`TagMapCount`, 如果`GameplayTag`不存在或者其`TagMapCount`为0就会返回false.  

`GameplayTag`必须在`DefaultGameplayTag.ini`中提前定义, UE4编辑器在project setting中提供了一个界面用于让开发者管理`GameplayTag`而无需手动编辑DefaultGameplayTag.ini, 该`GameplayTag`编辑器可以创建, 重命名, 搜索引用和删除`GameplayTag`.  

![`GameplayTag` Editor in Project Settings](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/gameplaytageditor.png)  

搜索`GameplayTag`引用会弹出一个类似Reference Viewer的窗口来显示所有引用该`GameplayTag`的资源, 但这不会显示任何引用该`GameplayTag`的C++类.  

重命名`GameplayTag`会创建重定向, 因此仍引用原来`GameplayTag`的资源会重定向到新的`GameplayTag`. 如果可以的话, 我更倾向于创建新的`GameplayTag`, 手动更新所有引用到新的`GameplayTag`, 之后删除旧的`GameplayTag`以避免创建新的重定向.  

除了`Fast Replication`, `GameplayTag`编辑器还有一个选项来填充通常需要同步的`GameplayTag`以对其深度优化.  

如果`GameplayTag`由`GameplayEffect`添加, 那么其就是可同步的. `ASC`允许你添加不可同步的`LooseGameplayTag`且必须手动管理. 样例项目对`State.Dead`使用了`LooseGameplayTag`, 因此当生命值降为0时, 其所属客户端会立即响应. 重生时需要手动将`TagMapCount`设置回0, 当使用`LooseGameplayTag`时只能手动调整`TagMapCount`, 相比纯手动调整`TagMapCount`, 最好使用`UAbilitySystemComponent::AddLooseGameplayTag()`和`UAbilitySystemComponent::RemoveLooseGameplayTag()`.  

C++中获取`GameplayTag`引用:  

```c++
FGameplayTag::RequestGameplayTag(FName("Your.GameplayTag.Name"))
```

对于像获取父或子`GameplayTag`的高级操作, 请查看`GameplayTagManager`提供的函数. 为了访问`GameplayTagManager`, 请引用`GameplayTagManager.h`并使用`UGameplayTagManager::Get().FunctionName`调用函数. 相比使用常量字符串进行操作和比较, `GameplayTagManager`实际上使用关系节点(父, 子等等)保存`GameplayTag`以获得更快的处理速度.  

`GameplayTag`和`GameplayTagContainer`有可选UPROPERTY宏`Meta = (Categories = "GameplayCue")`用于在蓝图中过滤标签而只显示父标签为`GameplayCue`的`GameplayTag`, 当你知道`GameplayTag`或`GameplayTagContainer`变量应该只用于`GameplayCue`时, 这将是非常有用的.  

作为选择, 有一单独的`FGameplayCueTag`结构体可以包裹`FGameplayTag`并且可以在蓝图中自动过滤`GameplayTag`而只显示父标签为`GameplayCue`的标签.  

如果你想过滤函数中的`GameplayTag`参数, 使用UFUNCTION宏`Meta = (GameplayTagFilter = "GameplayCue")`. `GameplayTagContainer`参数不能过滤, 如果你想编辑引擎来允许过滤`GameplayTagContainer`参数, 查看`SGameplayTagGraphPin::ParseDefaultValueData()`是如何从`Engine\Plugins\Editor\GameplayTagEditor\Source\GameplayTagEditor\Private\SGameplayTagGraphPin.cpp`中调用`FilterString = UGameplayTagManager::Get().GetCategoriesMetaFromField(PinStructType);`的, 还有是如何在`SGameplayTagGraphPin::GetListContent()`中将`FilterString`传递给`SGameplayTagWidget`的, `Engine\Plugins\Editor\GameplayTagEditor\Source\GameplayTagEditor\Private\S`GameplayTagContainer`GraphPin.cpp`中这些函数的`GameplayTagContainer`版本并没有检查Meta域属性和传递过滤器.  

样例项目广泛地使用了`GameplayTag`.  

#### 4.2.1 响应Gameplay Tags的变化

`ASC`提供了一个委托(Delegate)用于`GameplayTag`添加或移除时触发, 其中`EGameplayTagEventType`参数可以明确只有`GameplayTag`添加/移除还是该`GameplayTag`的`TagMapCount`发生变化时触发.  

```c++
AbilitySystemComponent->RegisterGameplayTagEvent(FGameplayTag::RequestGameplayTag(FName("State.Debuff.Stun")), EGameplayTagEventType::NewOrRemoved).AddUObject(this, &AGDPlayerState::StunTagChanged);
```

回调函数有一个`GameplayTag`参数和新的`TagCount`.  

```c++
virtual void StunTagChanged(const FGameplayTag CallbackTag, int32 NewCount);
```

### 4.3 Attribute

#### 4.3.1 Attribute定义

`Attribute`是由`FGameplayAttributeData`结构体定义的浮点值, 其可以表示从角色生命值到角色等级再到一瓶药水的剂量的任何事物, 如果某项数值是属于某个Actor且游戏相关的, 你就应该考虑使用`Attribute`. `Attribute`一般应该只能由`GameplayEffect`修改, 这样`ASC`才能预测(predict)其改变.  

`Attribute`也可以由`AttributeSet`定义并存于其中. `AttributeSet`用于同步那些标记为replication的`Attribute`. 参阅`AttributeSet`部分来了解如何定义`Attribute`.  

**Tip**: 如果你不想某个`Attribute`显示在编辑器的`Attribute`列表, 可以使用`Meta = (HideInDetailsView)`属性宏.  

#### 4.3.2 BaseValue vs. CurrentValue

一个`Attribute`是由两个值 —— 一个`BaseValue`和一个`CurrentValue`组成的, `BaseValue`是`Attribute`的永久值而`CurrentValue`是`BaseValue`加上`GameplayEffect`给的临时修改值后得到的. 例如, 你的Character可能有一个`BaseValue`为600u/s的移动速度`Attribute`, 因为还没有`GameplayEffect`修改移动速度, 所以`CurrentValue`也是600u/s, 如果Character获得了一个临时50u/s的移动速度加成, 那么`BaseValue`仍然是600u/s而`CurrentValue`是600+50=650u/s, 当该移动速度加成消失后, `CurrentValue`就会变回`BaseValue`的600u/s.  

初识GAS的新手经常将`BaseValue`误认为`Attribute`的最大值并以这样的认识去编码, 这是错误的, 可以改变或引用在Ability/UI中的`Attribute`最大值应该是另外单独的`Attribute`. 对于硬编码的最大值和最小值, 有一种方法是使用可以设置最大值和最小值的`FAttributeMetaData`定义一个DataTable, 但是Epic在该结构体上的注释称之为"work in progress", 参阅`AttributeSet.h`获得更多信息. 为了避免这种疑惑, 我建议引用在Ability或UI中的最大值应该单独定义`Attribute`, 只用于限制(Clamp)`Attribute`大小的硬编码最大值和最小值应该在`AttributeSet`中定义为硬编码浮点值. 关于`Attribute`值的限制(Clamp)在PreAttributeChange()中讨论了修改CurrentValue, 在PostGameplayEffectExecute()中讨论了修改`GameplayEffect`的`BaseValue`.  

`即刻(Instant)GameplayEffect`可以永久性的修改`BaseValue`, 而`持续(Duration)`和`无限(Infinite)GameplayEffect`可以修改CurrentValue. 周期性(Periodic)`GameplayEffect`被视为`GameplayEffect`实例并且可以修改`BaseValue`.  

#### 4.3.3 元(Meta)Attribute

一些`Attribute`被视为占位符, 其用于预计和`Attribute`交互的临时值, 这些`Attribute`被叫做`Meta Attribute`. 例如, 我们通常定义伤害值为`Meta Attribute`, 使用伤害值`Meta Attribute`作为占位符, 而不是使用`GameplayEffect`直接修改生命值`Attribute`, 使用这种方法, 伤害值就可以在`GameplayEffectExecutionCalculation`中由buff和debuff修改, 并且可以在`AttributeSet`中进一步操作, 例如, 在最终将生命值减去伤害值之前, 要将伤害值减去当前的护盾值. 伤害值`Meta Attribute`在`GameplayEffect`之间不是持久化的, 并且可以被任何一方重写. `Meta Attribute`一般是不可同步的.  

`Meta Attribute`对于在"我们应该造成多少伤害?"和"我们该如何处理伤害值?"这种问题之中的伤害值和治疗值做了很好的解构, 这种解构意味着`GameplayEffect`和`ExecutionCalculation`无需了解目标是如何处理伤害值的. 继续看伤害值的例子, `GameplayEffect`确定造成多少伤害, 之后`AttributeSet`决定如何使用该伤害值, 不是所有的Character都有相同的`Attribute`, 特别是使用了`AttributeSet`子类的话, `AttributeSet`基类可能只有一个生命值`Attribute`, 但是它的子类可能增加了一个护盾值`Attribute`, 拥有护盾值`Attribute`的子类`AttributeSet`可能会以不同于`AttributeSet`基类的方式分配收到的伤害.  

尽管`Meta Attribute`是一个很好的设计模式, 但它们并不是强制使用的. 如果你只有一个用于所有伤害实例的`Execution Calculation`和一个所有Character共用的`AttributeSet`类, 那么你就可以在`Exeuction Calculation`中分配伤害到生命, 护盾等等, 并直接修改那些`Attribute`, 这种方式你只会丢失灵活性, 但总体上并无大碍.  

#### 4.3.4 响应Attribute变化

为了监听`Attribute`何时变化以便更新UI和其他游戏逻辑, 可以使用`UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate(FGameplayAttribute `Attribute`)`, 该函数返回一个委托(Delegate), 你可以将其绑定一个当`Attribute`变化时需要自动调用的函数. 该委托提供一个`FOnAttributeChangeData`参数, 其中有`NewValue`, `OldValue`和`FGameplayEffectModCallbackData`. **Note**: `FGameplayEffectModCallbackData`只能在服务端上设置.  

```c++
AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(AttributeSetBase->GetHealthAttribute()).AddUObject(this, &AGDPlayerState::HealthChanged);

virtual void HealthChanged(const FOnAttributeChangeData& Data);
```

样例项目将其绑定到了`GDPlayerState`用于更新HUD, 当生命值下降为0时, 也可以响应玩家死亡.  

样例项目中有一个将上述逻辑包裹进`ASyncTask`的自定义蓝图节点, 其在`UI_HUD(UMG Widget)`中用于更新生命值, 魔法值和耐力值. 该AsyncTask会一直响应直到手动调用`EndTask()`, 就像在UMG Widget的`Destruct`事件中调用那样. 参阅`AsyncTaskAttributeChanged.h/cpp`.  

![Listen for `Attribute` Change BP Node](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/attributechange.png)  

#### 4.3.5 自动推导Attribute

为了使一个`Attribute`的部分或全部值继承自一个或更多`Attribute`, 可以使用基于一个或多个`Attribute`或`MMC Modifiers`的`无限(Infinite)GameplayEffect`. 当自动推导`Attribute`依赖的某个`Attribute`更新时它也会自动更新.  

在自动推导`Attribute`上的所有`修改器(Modifier)`形成的最终公式和`Modifier Aggregators`的公式是一样的. 如果你需要计算式要按一定的顺序进行, 在`MMC`中做就是了.  

```c++
((CurrentValue + Additive) * Multiplicitive) / Division
```

**Note**: 如果在PIE中打开多个窗口, 你需要在编辑器首选项中禁用`Run Under One Process`, 否则当自动推导`Attribute`所依赖的`Attribute`更新时, 除了第一个窗口外其不会更新.  

在这个例子中, 我们有一个`无限(Infinite)GameplayEffect`, 其从TestAttrB和TestAttrC `Attribute`以`TestAttrA = (TestAttrA + TestAttrB) * ( 2 * TestAttrC)`公式继承得到TestAttrA, 每次TestAttrB和TestAttrC更新时, TestAttrA都会自动重新计算.  

![Derived `Attribute` Example](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/derivedattribute.png)  

### 4.4 AttributeSet

#### 4.4.1 定义AttributeSet

`AttributeSet`用于定义, 保存和管理`Attribute`的变化. 开发者应该继承UAttributeSet. 在OwnerActor的构建函数中创建`AttributeSet`会自动注册到其`ASC`. **这必须在C++中完成.**  

#### 4.4.2 设计AttributeSet

一个`ASC`可能有一个或很多`AttributeSet`, `AttributeSet`消耗的内存微不足道, 使用多少`AttributeSet`是留给开发人员决定的.  

有种方案是设置一个单一且巨大的`AttributeSet`, 共享于游戏中的所有Actor, 并且只使用需要的`Attribute`, 忽略不用的`Attribute`.  

作为选择, 你可以使用多个`AttributeSet`表示按需添加到Actor的`Attribute`分组, 例如, 你可以有一个生命相关的`AttributeSet`, 一个魔法相关的`AttributeSet`, 等等. 在MOBA游戏中, 英雄可能需要魔法, 但是小兵并不需要, 因此英雄就需要魔法`AttributeSet`而小兵就不需要.  

另外, 继承`AttributeSet`的另一种意义是可以选择一个Actor可以有哪些`Attribute`. `Attribute`在内部被引用为`AttributeSetClassName.AttributeName`, 当你继承`AttributeSet`时, 所有父类的`Attribute`将仍保留父类名作为前缀.  

尽管可以拥有多个`AttributeSet`, 但是不应该在同一`ASC`中拥有多个同一类的`AttributeSet`, 如果在同一`ASC`中有多个同一类的`AttributeSet`, `ASC`就不知道该使用哪个`AttributeSet`而随机选择一个.  

##### 4.4.2.1 使用单独Attribute的子组件

假设你在某个Pawn上有多个可被损害的组件, 像可被独立损害的护甲片, 如果可以明确可被损害组件的最大数量, 我建议把多个生命值`Attribute`放到一个`AttributeSet`中 —— DamageableCompHealth0, DamageableCompHealth1, 等等, 以表示这些可被损害组件在逻辑上的"slot", 在可被损害组件的类实例中, 指定可以被`GameplayAbility`和Execution读取的带slot编号的`Attribute`来表明该应用伤害值到哪个`Attribute`. 如果Pawn当前拥有0个或少于最大数量的可损害组件也无妨, 因为`AttributeSet`拥有一个`Attribute`, 并不意味着必须要使用它, 未使用的`Attribute`只占用很少的内存.  

如果每个子组件都需要很多`Attribute`且子组件的数量可以是无限的, 或者子组件可以分离被其他玩家使用(比如武器), 或者出于其他原因上述方法不适用于你, 那么我建议就不要使用`Attribute`, 而是在组件中保存普通的浮点数. 参阅Item `Attribute`.  

##### 4.4.2.2 运行时添加和移除AttributeSet

`AttributeSet`可以在运行时从`ASC`上添加和移除, 然而移除`AttributeSet`是很危险的, 例如, 如果某个`AttributeSet`在客户端上移除早于服务端, 而某个`Attribute`的变化又同步到了客户端, 那么`Attribute`就会因为找不到`AttributeSet`而使游戏崩溃.  

武器添加到Inventory:  

```c++
AbilitySystemComponent->SpawnedAttribute.AddUnique(WeaponAttributeSetPointer);
AbilitySystemComponent->ForceReplication();
```

武器从Inventory移除:  

```c++
AbilitySystemComponent->SpawnedAttribute.Remove(WeaponAttributeSetPointer);
AbilitySystemComponent->ForceReplication();
```

##### 4.4.2.3 Item Attribute(武器弹药)

有几种方法可以实现带有`Attribute`(武器弹药, 盔甲耐久等等)的可装备物品, 所有这些方法都直接在物品中存储数据, 这对于在生命周期中可以被多个玩家装备的物品来说是必须的.  

1. 在物品中使用普通的浮点数(推荐).
2. 在物品中使用单独的`AttributeSet`.
3. 在物品中使用单独的`ASC`.

###### 4.4.2.3.1 在物品中使用普通浮点数

在物品类实例中存储普通浮点数而不是`Attribute`, Fortnite和GASShooter就是这样处理枪械子弹的, 对于枪械, 在其实例中存储可同步的浮点数(COND_OwnerOnly), 比如最大弹匣量, 当前弹匣中弹药量, 剩余弹药量等等, 如果枪械需要共享剩余弹药量, 那么就将剩余弹药量移到Character中共享的弹药`AttributeSet`里作为一个`Attribute`(换弹Ability可以使用一个`Cost GE`从剩余弹药量中填充枪械的弹匣弹药量浮点). 因为没有为当前弹匣弹药量使用`Attribute`, 所以需要重写`UGameplayAbility`中的一些函数来检查和应用枪械中浮点数的花销(cost). 当授予Ability时将枪械在`GameplayAbilitySpec`中转换为`SourceObject`, 这意味着可以在Ability中访问授予Ability的枪械.  

为了防止在全自动射击过程中枪械会反向同步弹药量并扰乱本地弹药量(译者注: 通俗解释就是因为存在同步延迟且在连续射击这一高同步过程中, 所以客户端的弹药量会来不及和服务端同步, 造成弹药量减少后又突然变多的现象.), 如果玩家拥有`IsFiring`的`GameplayTag`, 就在PreReplication()中禁用同步, 本质上是要在其中做自己的本地预测.  

```c++
void AGSWeapon::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
	Super::PreReplication(ChangedPropertyTracker);

	DOREPLIFETIME_ACTIVE_OVERRIDE(AGSWeapon, PrimaryClipAmmo, (IsValid(AbilitySystemComponent) && !AbilitySystemComponent->HasMatchingGameplayTag(WeaponIsFiringTag)));
	DOREPLIFETIME_ACTIVE_OVERRIDE(AGSWeapon, SecondaryClipAmmo, (IsValid(AbilitySystemComponent) && !AbilitySystemComponent->HasMatchingGameplayTag(WeaponIsFiringTag)));
}
```

好处:  

1. 避免了使用`AttributeSet`的局限(见下).

局限:  

1. 不能使用现有的`GameplayEffect`工作流(弹药使用的Cost GEs等等).
2. 要求重写UGameplayAbility中的关键函数来检查和应用枪械中浮点数的花销(Cost).

###### 4.4.2.3.2 在物品中使用AttributeSet

在物品中使用单独的`AttributeSet`可以实现"将其添加到玩家的Inventory", 但还是有一定的局限性. 较早版本的GASShooter中的武器弹药是使用的这种方法, 武器类在其自身存储诸如最大弹匣量, 当前弹匣弹药量, 剩余弹药量等等到一个`AttributeSet`, 如果枪械需要共享剩余弹药量, 那么就将剩余弹药量移到Character中共享的弹药`AttributeSet`里. 当服务器上某个武器添加到玩家的Inventory后, 该武器会将它的`AttributeSet`添加到玩家的`ASC::SpawnedAttribute`, 之后服务器会将其同步下发到客户端, 如果该武器从Inventory中移除, 它也会将其`AttributeSet`从`ASC::SpawnedAttribute`中移除.  

当`AttributeSet`存于除了OwnerActor之外的对象上时(对于某个武器来说), 会得到一些关于`AttributeSet`的编译错误, 解决办法是在BeginPlay()中构建`AttributeSet`而不是在构造函数中, 并在武器类中实现`IAbilitySystemInterface`(当你添加武器到玩家Inventory时设置`ASC`的指针).  

```c++
void AGSWeapon::BeginPlay()
{
	if (!`AttributeSet`)
	{
		`AttributeSet` = NewObject<UGSWeaponAttributeSet>(this);
	}
	//...
}
```

你可以查看较早版本的GASShooter来实际地体会这种方案.  

好处:  
1. 可以使用已有的`GameplayAbility`和`GameplayEffect`工作流(弹药使用的Cost GEs等等).
2. 对于很小的物品集可以快速设置

局限:  
1. 必须为每个武器类型创建新的`AttributeSet`类, `ASC`实际上只能有一个该类的`AttributeSet`实例, 因为对`Attribute`的修改会在`ASC`的SpawnedAttribute数组中寻找其第一个`AttributeSet`类实例, 其他相同的`AttributeSet`类实例则会被忽略.  
2. 和第1条同样的原因(每个`AttributeSet`类一个`AttributeSet`实例), 在玩家的Inventory中每种武器类型只能有一个.
3. 移除`AttributeSet`是很危险的. 在GASShooter中, 如果玩家因为火箭弹而自杀, 玩家会立即从其Inventory中移除火箭弹发射器(包括其在`ASC`中的`AttributeSet`), 当服务端同步火箭弹发射器的弹药`Attribute`改变时, 由于`AttributeSet`在客户端`ASC`上不复存在而使游戏崩溃.

###### 4.4.2.3.3 在物品中使用单独的ASC

在每个物品上都创建一个`AbilitySystemComponent`是种很极端的方案. 我还没有亲自做过这种方案, 在其他地方也没见过. 这种方案应该会花费相当的开发成本才能正常使用.  

> Is it viable to have several AbilitySystemComponents which have the same owner but different avatars (e.g. on pawn and weapon/items/projectiles with Owner set to PlayerState)?

> The first problem I see there would be implementing the IGameplayTagAssetInterface and IAbilitySystemInterface on the owning Actor. The former may be possible: just aggregate the tags from all all ASCs (but watch out -HasAlMatchingGameplayTag may be met only via cross ASC aggregation. It wouldn't be enough to just forward that calls to each ASC and OR the results together). But the later is even trickier: which ASC is the authoritative one? If someone wants to apply a GE -which one should receive it? Maybe you can work these out but this side of the problem will be the hardest: owners will multiple ASCs beneath them.

> Separate ASCs on the pawn and the weapon can make sense on its own though. E.g, distinguishing between tags the describe the weapon vs those that describe the owning pawn. Maybe it does make sense that tags granted to the weapon also “apply” to the owner and nothing else (E.g, Attribute and GEs are independent but the owner will aggregate the owned tags like I describe above). This could work out, I am sure. But having multiple ASCs with the same owner may get dicey.

*[community questions #6](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)中来自Epic的Dave Ratti的回答.*  

好处: 
1. 可以使用已有的`GameplayAbility`和`GameplayEffect`工作流(弹药使用的Cost GEs等等).
2. 可以复用`AttributeSet`类(每个武器的`ASC`中各一个).

局限:  
1. 未知的开发成本.
2. 甚至方案可行么?

#### 4.4.3 定义Attribute

**Attribute只能使用C++在AttributeSet头文件中定义.** 建议把下面这个宏块加到每个`AttributeSet`头文件的顶部, 其会自动为每个`Attribute`生成getter和setter函数.  

```c++
// Uses macros from `AttributeSet`.h
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
```

一个可同步的生命值`Attribute`可能像下面这样定义:  

```c++
UPROPERTY(BlueprintReadOnly, Category = "Health", ReplicatedUsing = OnRep_Health)
FGameplayAttributeData Health;
ATTRIBUTE_ACCESSORS(UGDAttributeSetBase, Health)
```

同样在头文件中定义OnRep函数:  

```c++
UFUNCTION()
virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);
```

`AttributeSet`的.cpp文件应该用预测系统(prediction system)使用的`GAMEPLAYATTRIBUTE_REPNOTIFY`宏填充OnRep函数:  

```c++
void UGDAttributeSetBase::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(UGDAttributeSetBase, Health, OldHealth);
}
```

最后, `Attribute`需要添加到`GetLifetimeReplicatedProps`:  

```c++
void UGDAttributeSetBase::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME_CONDITION_NOTIFY(UGDAttributeSetBase, Health, COND_None, REPNOTIFY_Always);
}
```

`REPTNOTIFY_Always`告知OnRep函数如果本地值已经和从服务端同步下来的值相同时触发(预测的原因), 默认当本地值和从服务端同步下来的值相同时, OnRep函数是不会触发的.  

如果`Attribute`无需像`Meta Attribute`那样同步, 那么`OnRep`和`GetLifetimeReplicatedProps`步骤可以跳过.  

#### 4.4.4 初始化Attribute

有多种方法可以初始化`Attribute`(将BaseValue和CurrentValue设置为某初始值). Epic建议使用`即刻(Instant)GameplayEffect`, 这也是样例项目使用的方法.  

查看样例项目的GE_HeroAttribute蓝图来了解如何创建`即刻(Instant)GameplayEffect`以初始化`Attribute`, 该`GameplayEffect`应用是写在C++中的.  

如果在定义`Attribute`时使用了ATTRIBUTE_ACCESSORS宏, 那么在`AttributeSet`中会自动为每个`Attribute`生成一个初始化函数.  

```c++
// InitHealth(float InitialValue) is an automatically generated function for an `Attribute` 'Health' defined with the `ATTRIBUTE_ACCESSORS` macro
AttributeSet->InitHealth(100.0f);
```

查看`AttributeSet.h`获取更多初始化`Attribute`的方法.  

**Note**: 4.24之前, FAttributeSetInitterDiscreteLevels不能和FGameplayAttributeData协同使用, 它在`Attribute`是原生浮点数时创建, 并且会和FGameplayAttributeData不是Plain Old Data(POD)时冲突. 该问题在4.24中修复([https://issues.unrealengine.com/issue/UE-76557.](https://issues.unrealengine.com/issue/UE-76557)).  

#### 4.4.5 PreAttributeChange()

`PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)`是`AttributeSet`中主要的函数之一, 其在变化发生前响应`Attribute`的CurrentValue的变化, 其是通过引用参数NewValue限制(Clamp)CurrentValue即将发生的改变的理想位置.  

例如像样例项目那样限制移动速度`修改器`:  
```c++
if (`Attribute` == GetMoveSpeedAttribute())
{
	// Cannot slow less than 150 units/s and cannot boost more than 1000 units/s
	NewValue = FMath::Clamp<float>(NewValue, 150, 1000);
}
```

`GetMoveSpeedAttribute()`函数是由我们在`AttributeSet.h`中添加的宏块创建的(Defining Attribute).  

`PreAttributeChange()`可以由`Attribute`的任何改变触发, 无论是使用`Attribute`的setter(由`AttributeSet.h`中的宏块定义(Defining Attribute))还是使用`GameplayEffect`.  

**Note**: 在这里做的任何限制都不会永久性地修改`ASC`中的`修改器(Modifier)`, 只会修改查询`修改器`返回的值, 这意味着像`GameplayEffectExecutionCalculations`和`ModifierMagnitudeCalculations`这种从所有`修改器`处重新计算CurrentValue的函数需要再次执行限制(Clamp)操作.  

**Note**: Epic对于PreAttributeChange()的注释说明不要将该函数用于游戏逻辑事件, 而主要在其中做限制操作. 对于修改`Attribute`的游戏逻辑事件的建议位置是`UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate(FGameplayAttribute Attribute)`(Responding to Attribute Changes).  

#### 4.4.6 PostGameplayEffectExecute()

`PostGameplayEffectExecute(const FGameplayEffectModCallbackData & Data)`会在某个来自`即刻(Instant)GameplayEffect`的`Attribute`的BaseValue变化之后触发, 当修改是来自`GameplayEffect`时, 这就是一个处理更多`Attribute`操作的有效位置.  

例如, 在样例项目中, 我们在这里从生命值`Attribute`中减去了最终的伤害值`Meta Attribute`, 如果有护盾值`Attribute`的话, 我们也会在减除生命值之前从护盾值中减除伤害值. 样例项目也在这里应用被击打反应动画, 显示浮动的伤害数值, 和为击杀者分配经验值和赏金. 通过设计, 伤害值`Meta Attribute`总是会通过某个`即刻(Instant)GameplayEffect`而不会通过Attribute setter.  

其他只会由`即刻(Instant)GameplayEffect`修改BaseValue的`Attribute`, 像魔法值和耐力值, 也可以在这里被限制为它们对应`Attribute`的最大值.  

**Note**: 当PostGameplayEffectExecute()被调用时, 对`Attribute`的改变已经发生, 但是还没有被同步回客户端, 因此在这里限制值不会造成对客户端的二次同步, 客户端只会接收到限制后的值.  

#### 4.4.7 OnAttributeAggregatorCreated()

`OnAttributeAggregatorCreated(const FGameplayAttribute& Attribute, FAggregator* NewAggregator)`会在Aggregator为集合中的某个`Attribute`创建时触发, 它允许`FAggregatorEvaluateMetaData`的自定义设置, `AggregatorEvaluateMetaData`是Aggregator基于所有应用的`修改器(Modifier)`评估`Attribute`的CurrentValue的. 默认情况下, AggregatorEvaluateMetaData只由Aggregator用于确定哪些`修改器`是满足条件的, 以MostNegativeMod_AllPositiveMods为例, 其允许所有正(Positive)`修改器`但是限制负(Negative)`修改器`(仅最负的那一个), 这在Paragon中只允许将最负移动速度减速效果应用到玩家, 而不用管应用所有正移动速度buff时有多少负移动效果. 不满足条件的`修改器`仍存于`ASC`中, 只是不被总合进最终的CurrentValue, 一旦条件改变, 它们之后就可能满足条件, 就像如果最负`修改器`过期后, 下一个最负`修改器`(如果存在的话)就是满足条件的.  

为了在只允许最负`修改器`和所有正`修改器`的例子中使用`AggregatorEvaluateMetaData`:  

```c++
virtual void OnAttributeAggregatorCreated(const FGameplayAttribute& Attribute, FAggregator* NewAggregator) const override;

void UGSAttributeSetBase::OnAttributeAggregatorCreated(const FGameplayAttribute& Attribute, FAggregator* NewAggregator) const
{
	Super::OnAttributeAggregatorCreated(Attribute, NewAggregator);

	if (!NewAggregator)
	{
		return;
	}

	if (Attribute == GetMoveSpeedAttribute())
	{
		NewAggregator->EvaluationMetaData = &FAggregatorEvaluateMetaDataLibrary::MostNegativeMod_AllPositiveMods;
	}
}
```

你的自定义`AggregatorEvaluateMetaData`应该作为静态变量添加到`FAggregatorEvaluateMetaDataLibrary`.  

### 4.5 Gameplay Effects

#### 4.5.1 定义GameplayEffect

`GameplayEffect(GE)`是Ability修改其自身和其他`Attribute`和`GameplayTag`的容器, 其可以立即修改`Attribute`(像伤害或治疗)或应用长期的状态buff/debuff(像移动速度加速或眩晕). UGameplayEffect只是一个定义单一游戏效果的数据类, 不应该在其中添加额外的逻辑. 设计师一般会创建很多UGameplayEffect的子类蓝图.  

`GameplayEffect`通过`修改器(Modifier)`和`Execution(GameplayEffectExecutionCalculation)`修改`Attribute`.  

`GameplayEffect`有三种持续类型: `即刻(Instant)`, `持续(Duration)`和`无限(Infinite)`.  

额外地, `GameplayEffect`可以添加/执行`GameplayCue`, `即刻(Instant)GameplayEffect`可以调用`GameplayCue`, `GameplayTag`中的Execute而`持续(Duration)`或`无限(Infinite)`可以调用`GameplayCue`, `GameplayTag`中的Add和Remove.  

|类型|`GameplayCue`事件|何时使用|
|:-:|:-:|:-:|
|`即刻(Instant)`|Execute|对于`Attribute`中BaseValue的永久性的立即修改. `GameplayTag`不会被应用, 哪怕是一帧.|
|`持续(Duration)`|Add & Remove|对于`Attribute`中CurrentValue的临时修改和当`GameplayEffect`过期或手动移除时, 应用将要被移除的`GameplayTag`. 持续时间是在UGameplayEffect类/蓝图中明确的.|
|`无限(Infinite)`|Add & Remove|对于`Attribute`中CurrentValue的临时修改和当`GameplayEffect`移除时, 应用将要被移除的`GameplayTag`. 该类型自身永不过期且必须由某个Ability或`ASC`手动移除.|

`持续(Duration)`和`无限(Infinite)GameplayEffect`可以选择应用周期性的Effect, 其每过X秒(由周期定义)就应用一次`修改器`(Modifier)和Execution, 当周期性的Effect修改`Attribute`的BaseValue和执行`GameplayCue`时就被视为`即刻(Instant)GameplayEffect`, 这对于像随时间推移的持续伤害(damage over time, DOT)这种类型的Effect很有用. **Note**: 周期性的Effect不能被预测.  

如果`持续(Duration)`和`无限(Infinite)GameplayEffect`进行中的标签需求(Ongoing Tag Requirements)未满足的话, 那么它们在应用后就可以被暂时的关闭和打开(Gameplay Effect Tags), 关闭`GameplayEffect`会移除其`修改器`和已应用`GameplayTag`的效果, 但是不会移除该`GameplayEffect`, 重新打开`GameplayEffect`会重新应用其`修改器`和`GameplayTag`.  

如果你需要手动重新计算某个`持续(Duration)`或`无限(Infinite)GameplayEffect`的`修改器(Modifier)`(假设有一个使用非`Attribute`数据的`MMC`), 可以使用和`UAbilitySystemComponent::ActiveGameplayEffect.GetActiveGameplayEffect(ActiveHandle).Spec.GetLevel()`相同的evel`调用UAbilitySystemComponent::ActiveGameplayEffect.SetActiveGameplayEffectLevel(FActiveGameplayEffectHandle ActiveHandle, int32 NewLevel)`. 当支持(backing)`Attribute`更新时, 基于支持(backing)`Attribute`的`修改器`会自动更新. SetActiveGameplayEffectLevel()更新`修改器`的关键函数是:  

```c++
MarkItemDirty(Effect);
Effect.Spec.CalculateModifierMagnitudes();
// Private function otherwise we'd call these three functions without needing to set the level to what it already is
UpdateAllAggregatorModMagnitudes(Effect);
```

`GameplayEffect`一般是不实例化的, 当Ability或`ASC`想要应用`GameplayEffect`时, 其会从`GameplayEffect`的`ClassDefaultObject`创建一个`GameplayEffectSpec`, 之后成功应用的`GameplayEffectSpec`会添加到一个名为`FActiveGameplayEffect`的新结构体, 其是`ASC`在名为`ActiveGameplayEffect`的特殊结构体容器中追踪的内容.  

#### 4.5.2 应用GameplayEffect

`GameplayEffect`可以被`GameplayAbility`和`ASC`中的多个函数应用, 其通常是`ApplyGameplayEffectTo`的形式, 不同的函数本质上都是最终在目标上调用`UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf()`的方便函数.  

为了在`GameplayAbility`之外应用`GameplayEffect`, 例如从某个投掷物中, 你就需要获取到该目标的`ASC`并使用它的函数之一来`ApplyGameplayEffectToSelf`.  

你可以绑定`持续(Duration)`或`无限(Infinite)GameplayEffect`的委托来监听其应用到`ASC`:  

```c++
AbilitySystemComponent->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(this, &APACharacterBase::OnActiveGameplayEffectAddedCallback);
```

回调函数:  

```c++
virtual void OnActiveGameplayEffectAddedCallback(UAbilitySystemComponent* Target, const F`GameplayEffectSpec`& SpecApplied, FActiveGameplayEffectHandle ActiveHandle);
```

服务端总是会调用该函数而不管同步模式是什么, `Autonomous`代理只会在`Full`和`Mixed`同步模式下对于可同步的`GameplayEffect`调用该函数, `Simulated`代理只会在`Full`同步模式下调用该函数.  

#### 4.5.3 移除GameplayEffect

`GameplayEffect`可以被`GameplayAbility`和`ASC`中的多个函数移除, 其通常是`RemoveActiveGameplayEffect`的形式, 不同的函数本质上都是最终在目标上调用`FActiveGameplayEffectContainer::RemoveActiveEffects()`的方便函数.  

为了在`GameplayAbility`之外移除`GameplayEffect`, 你就需要获取到该目标的`ASC`并使用它的函数之一来`RemoveActiveGameplayEffect`.  

你可以绑定`持续(Duration)`或`无限(Infinite)GameplayEffect`的委托来监听其应用到`ASC`:  

```c++
AbilitySystemComponent->OnAnyGameplayEffectRemovedDelegate().AddUObject(this, &APACharacterBase::OnRemoveGameplayEffectCallback);
```

回调函数:  

```c++
virtual void OnRemoveGameplayEffectCallback(const FActiveGameplayEffect& EffectRemoved);
```

服务端总是会调用该函数而不管同步模式是什么, Autonomous代理只会在Full和Mixed同步模式下对于可同步的`GameplayEffect`调用该函数, Simulated代理只会在Full同步模式下调用该函数.  

#### 4.5.4 GameplayEffect修改器

`修改器`(Modifier)可以修改`Attribute`并且是唯一可以预测性修改`Attribute`的方法. 一个`GameplayEffect`可以有0个或多个`修改器`, 每个`修改器`通过某个指定的操作只能修改一个`Attribute`.  

|操作|描述|
|:-:|:-:|
|Add|将`修改器`指定的`Attribute`加上计算结果. 使用负数以实现减法操作.|
|Multiply|将`修改器`指定的`Attribute`乘以计算结果.|
|Divide|将`修改器`指定的`Attribute`除以计算结果.|
|Override|使用计算结果覆盖`修改器`指定的`Attribute`.|

`Attribute`的CurrentValue是其所有`修改器`与其BaseValue计算并总合后的结果, 像下面这样的`修改器`总合公式被定义在`GameplayEffectAggregator.cpp`中的`FAggregatorModChannel::EvaluateWithBase`:  

```c++
((InlineBaseValue + Additive) * Multiplicitive) / Division
```

Override`修改器`会优先覆盖最后应用的`修改器`得出的最终值.  

**Note**: 对于基于百分比的修改, 确保使用`Multiply`操作以使其在加法操作之后.  

**Note**: 预测(Prediction)对于百分比修改有些问题.  

有四种类型的`修改器`: Scalable Float, `Attribute` Based, Custom Calculation Class, 和 Set By Caller, 它们全都生成一些浮点数, 用于之后基于各自的操作修改指定`修改器`的`Attribute`.  

|`修改器`类型|描述|
|:-:|:-:|
|Scalable Float|FScalableFloats结构体可以指向某个横向为变量, 纵向为等级的Data Table, `Scalable Float`会以Ability的当前等级自动读取指定Data Table的某行值(或者在`GameplayEffectSpec`中重写的不同等级), 该值可以被系数处理, 如果没有指定Data Table/Row, 那么该值就会被视为1, 因此系数就可以被用来在所有等级硬编码为一个单一值.![ScalableFloat](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/scalablefloats.png)|
|`Attribute` Based|`Attribute` Based`修改器`将CurrentValue或BaseValue视为Source(谁创建的`GameplayEffectSpec`)或Target(谁接收`GameplayEffectSpec`)的支持(Backing)`Attribute`, 可以使用系数和前后系数之和来修改它. `Snapshotting`意味着当`GameplayEffectSpec`创建时支持`Attribute`被捕获(Captured), 而`no snapshotting`意味着当`GameplayEffectSpec`被应用时`Attribute`被捕获.|
|Custom Calculation Class|`Custom Calculation Class`为复杂的`修改器`提供了最大的灵活性, 该`修改器`使用了`ModifierMagnitudeCalculation`类, 且可以使用系数和前后系数之和处理浮点值结果.|
|Set By Caller|`SetByCaller`修改器是运行时由Ability或`GameplayEffectSpec`的创建者于`GameplayEffect`之外设置的值, 例如, 如果你想让伤害值随玩家蓄力技能的长短而变化, 那么就需要使用`SetByCaller`. `SetByCaller`本质上是存于`GameplayEffectSpec`中的`TMap<FGameplayTag, float>`, `修改器`只是告知`Aggregator`去寻找与提供的`GameplayTag`相关联的`SetByCaller`值. `修改器`使用的`SetByCaller`只能使用该概念的`GameplayTag`形式, `FName`形式在此处不适用. 如果`修改器`被设置为`SetByCaller`, 但是带有正确`GameplayTag`的`SetByCaller`在`GameplayEffectSpec`中不存在, 那么游戏会抛出一个运行时错误并返回0, 这可能在`Divide`操作中出现问题. 参阅`SetByCaller`s获取更多关于如何使用`SetByCaller`的信息.|

##### 4.5.4.1 Multiply和Divide修改器

默认情况下, 所有的`Multiply`和`Divide`修改器在对`Attribute`的BaseValue乘除前都会先加到一起.  

```c++
float FAggregatorModChannel::EvaluateWithBase(float InlineBaseValue, const FAggregatorEvaluateParameters& Parameters) const
{
	...
	float Additive = SumMods(Mods[EGameplayModOp::Additive], GameplayEffectUtilities::GetModifierBiasByModifierOp(EGameplayModOp::Additive), Parameters);
	float Multiplicitive = SumMods(Mods[EGameplayModOp::Multiplicitive], GameplayEffectUtilities::GetModifierBiasByModifierOp(EGameplayModOp::Multiplicitive), Parameters);
	float Division = SumMods(Mods[EGameplayModOp::Division], GameplayEffectUtilities::GetModifierBiasByModifierOp(EGameplayModOp::Division), Parameters);
	...
	return ((InlineBaseValue + Additive) * Multiplicitive) / Division;
	...
}
```

```c++
float FAggregatorModChannel::SumMods(const TArray<FAggregatorMod>& InMods, float Bias, const FAggregatorEvaluateParameters& Parameters)
{
	float Sum = Bias;

	for (const FAggregatorMod& Mod : InMods)
	{
		if (Mod.Qualifies())
		{
			Sum += (Mod.EvaluatedMagnitude - Bias);
		}
	}

	return Sum;
}
```

*摘自GameplayEffectAggregator.cpp*  

在该公式中`Multiply`和`Divide`修改器都有一个值为1的`Bias`值(加法的`Bias`值为0), 因此它看起来像:  

```c++
1 + (Mod1.Magnitude - 1) + (Mod2.Magnitude - 1) + ...
```

该公式会导致一些意料之外的结果, 首先, 它在对BaseValue乘除之前将所有的`修改器`都加到了一起, 大部分人都期望将其乘或除在一起, 例如, 你有两个值为1.5的`Multiply`修改器, 大部分人都期望将BaseValue乘上`1.5 x 1.5 = 2.25`, 然而, 这里是将两个1.5加在一起再乘以BaseValue(50%增量 + 另一个50%增量 = 100%增量).拿`GameplayPrediction.h`中的一个例子来说, 给基值速度500加上10%的加速buff就是550, 再加上另一个10%的加速buff就是600.  

其次, 该公式还有一些关于可以使用什么值的未说明的规则, 因为这是考虑Paragon的情况而设计的.  

对于`Multiply`和`Divide`中乘法加法公式的规则:  

* (最多不超过1个值 < 1) AND (任何值都位于区间[1, 2))
* OR (有一个值 >= 2)

公式中的Bias基本上会减去`[1, 2)`区间中的整数位, 第一个修改器的`Bias`会从最开始的`Sum`值减值(在循环体前设置Bias), 这就是某个值它本身的作用和某个小于1的值与`[1, 2)`区间中的值的作用.  

`Multiply`的一些例子:  
Multipliers: 0.5  
`1 + (0.5 - 1) = 0.5`, 正确.  

Multipliers: 0.5, 0.5  
`1 + (0.5 - 1) + (0.5 - 1) = 0`, 错误, 结果应该是1. 小于1的积数在`修改器`相加中不起作用. Paragon这样设计只是为了使用`对于Multiply`修改器`的最负值`, 因此最多只会有一个小于1的值乘到BaseValue.  

Multipliers: 1.1, 0.5  
`1 + (0.5 - 1) + (1.1 - 1) = 0.6`, 正确.  

Multipliers: 5, 5
`1 + (5 - 1) + (5 - 1) = 9`, 错误, 结果应该是10. 总会是`修改器值的和 - 修改器的数量 + 1`(译者注: 修改器此时只有`5`一个.).  

很多游戏会想要它们的`Modify`和`Divide`修改器在应用到BaseValue之前先乘或除到一起, 为了实现这种需求, 你需要修改`FAggregatorModChannel::EvaluateWithBase()`的引擎代码.  

```c++
float FAggregatorModChannel::EvaluateWithBase(float InlineBaseValue, const FAggregatorEvaluateParameters& Parameters) const
{
	...
	float Multiplicitive = MultiplyMods(Mods[EGameplayModOp::Multiplicitive], Parameters);
	...

	return ((InlineBaseValue + Additive) * Multiplicitive) / Division;
}
```

```c++
float FAggregatorModChannel::MultiplyMods(const TArray<FAggregatorMod>& InMods, const FAggregatorEvaluateParameters& Parameters)
{
	float Multiplier = 1.0f;

	for (const FAggregatorMod& Mod : InMods)
	{
		if (Mod.Qualifies())
		{
			Multiplier *= Mod.EvaluatedMagnitude;
		}
	}

	return Multiplier;
}
```

##### 4.5.4.2 修改器的GameplayTag

`SourceTag`和`TargetTag`可以为每个`修改器`设置, 它们的作用就像`GameplayEffect`的`Application Tag requirements`, 因此只有当Effect应用后才会考虑标签, 也可以说当有一个周期性(Periodic)的`无限(Infinite)`Effect时, 这些标签只会在第一次应用Effect时才会被考虑, 而不是在每次周期执行时.  

`Attribute Based`修改器也可以设置`SourceTagFilter`和`TargetTagFilter`. 当确定`Attribute Based`修改器的源`Attribute`的量值时, 这些过滤器就会用来排除该`Attribute`指定的`修改器`, `修改器`的Source或Target中没有过滤器所标记标签的就会被排除在外.  

这更详尽的意思是: 源`ASC`和目标`ASC`的标签都被`GameplayEffect`所捕获, 当`GameplayEffectSpec`创建时, 源`ASC`的标签被捕获, 当执行Effect时, 目标`ASC`的标签被捕获. 当确定`无限(Infinite)`或`持续(Duration)`Effect的`修改器`是否满足条件可以被应用(也就是总合条件)并且过滤器已经被设置时, 被捕获的标签就会和过滤器进行比对.  

#### 4.5.5 GameplayEffect堆栈

`GameplayEffect`默认会应用`GameplayEffectSpec`的新实例, 而不明确或不关心之前已经应用过的尚且存在的`GameplayEffectSpec`实例. `GameplayEffect`可以设置到堆栈中, `GameplayEffectSpec`的新实例不会添加到堆栈中, 而是修改当前已经存在的`GameplayEffectSpec`堆栈数. 堆栈只适用于`持续(Duration)`和`无限(Infinite)GameplayEffect`.  

有两种类型的堆栈: Aggregate by Source和Aggregate by Target.  

|堆栈类型|描述|
|:-:|:-:|
|Aggregate by Source|目标(Target)中的每个源`ASC`都有一个单独的堆栈实例, 每个源可以应用堆栈中的X个(`GameplayEffect`).|
|Aggregate by Target|目标(Target)上只有一个堆栈实例而不管源如何, 每个源可以将一个堆栈应用到共享堆栈极限(Limit).|

堆栈对过期, 持续刷新和周期性刷新也有一些处理策略, 这些在`GameplayEffect`蓝图中都有很友好的悬浮提示帮助.  

样例项目包含一个用于监听`GameplayEffect`堆栈变化的自定义蓝图节点, HUD UMG Widget使用它来更新玩家拥有的被动护盾堆栈(层数). 该`AsyncTask`将会一直响应直到手动调用`EndTask()`, 就像在UMG Widget的`Destruct`事件中调用那样. 参阅`AsyncTaskAttributeChanged.h/cpp`.  

![Listen for `GameplayEffect` Stack Change BP Node](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/gestackchange.png)

#### 4.5.6 授予(使用)的Ability

`GameplayEffect`可以授予(Grant)新的`GameplayAbility`到`ASC`. 只有`持续(Duration)`和`无限(Infinite)GameplayEffect`可以授予Ability.  

一个普遍用法是当想要强制另一个玩家做某些事的时候, 像击退或拉取时移动他们, 就会对他们应用一个`GameplayEffect`来授予其一个自动激活的Ability, 从而使其做出相应的动作.  

设计师可以选择一个`GameplayEffect`能够授予哪些Ability, 在什么等级时授予, 将其绑定在什么输入键上和授予Ability的移除策略.  

|移除策略|描述|
|:-:|:-:|
|立即取消Ability|当授予Ability的`GameplayEffect`从目标移除时, 授予的Ability就会立即取消并移除.|
|结束时移除Ability|允许授予的Ability完成, 之后将其从目标移除.|
|无|授予的Ability不受从目标移除的授予`GameplayEffect`的影响, 目标将会一直拥有该Ability直到之后被手动移除.|

#### 4.5.7 GameplayEffect标签

`GameplayEffect`可以带有多个`GameplayTagContainer`, 设计师可以编辑每个种类(Category)的`Added`和`Removed`GameplayTagContainer, 结果会在编译时显示在合并的`GameplayTagContainer`中. `Added`标签是该`GameplayEffect`新增的其父类之前没有的标签, `Removed`标签是其父类拥有但该类没有的标签.  

|分类|描述|
|:-:|:-:|
|Gameplay Effect Asset Tags|`GameplayEffect`拥有的标签, 它们自身没有任何功能且只用于描述`GameplayEffect`.|
|Granted Tags|存于`GameplayEffect`中但又用于`GameplayEffect`应用到的`ASC`的标签. 当`GameplayEffect`移除时它们也会从`ASC`中移除. 该标签只作用于`持续(Duration)`和`无限(Infinite)GameplayEffect`.|
|Ongoing Tag Requirements|一旦应用后, 这些标签将决定`GameplayEffect`是开启还是关闭. 一个`GameplayEffect`可以是关闭但仍然是应用的. 如果某个`GameplayEffect`由于不符合`Ongoing Tag Requirements`而关闭, 但是之后又满足需求了, 那么该`GameplayEffect`会重新打开并重应用它的`修改器`. 该标签只作用于`持续(Duration)`和`无限(Infinite)GameplayEffect`.|
|Application Tag Requirements|位于目标上决定某个`GameplayEffect`是否可以应用到该目标的标签, 如果不满足这些需求, 那么`GameplayEffect`就不可应用.|
|Remove Gameplay Effects with Tags|当该`GameplayEffect`被成功应用后, 位于目标上的`GameplayEffect`会从目标中删除它在`Asset Tag`或`Granted Tag`中的这些标签.|

#### 4.5.8 免疫

`GameplayEffect`可以基于`GameplayTag`实现免疫, 有效阻止其他`GameplayEffect`的应用. 尽管免疫可以由`Application Tag Requirements`等方式有效地实现, 但是使用该系统可以在`GameplayEffect`被免疫阻塞时提供`UAbilitySystemComponent::OnImmunityBlockGameplayEffectDelegate`委托(Delegate).  

`GrantedApplicationImmunityTags`会检查源`ASC`(包括源Ability的AbilityTag, 如果有的话)是否包含特定的标签, 这是一种基于特定Character或源的标签对其所有`GameplayEffect`提供免疫的方法.  

`Granted Application Immunity Query`会检查传入的`GameplayEffectSpec`是否与其查询条件相匹配, 从而阻塞或允许其应用.  

`GameplayEffect`蓝图中的查询条件都有友好的悬浮提示帮助.  

#### 4.5.9 GameplayEffectSpec

`GameplayEffectSpec`(GESpec)可以看作是`GameplayEffect`的实例, 它保存了一个其所代表的`GameplayEffect`类的引用, 创建时的等级和创建者, 它在应用之前可以在运行时(Runtime)自由的创建和修改, 不像`GameplayEffect`应该由设计师在运行前创建. 当应用`GameplayEffect`时, `GameplayEffectSpec`会自`GameplayEffect`创建并且会实际应用到目标.  

`GameplayEffectSpec`是由`UAbilitySystemComponent::MakeOutgoingSpec()(BlueprintCallable)`自`GameplayEffect`创建的. `GameplayEffectSpec`不必立即应用. 通常是将`GameplayEffectSpec`传递给创建自Ability的投掷物, 该投掷物可以应用到它之后击中的目标. 当`GameplayEffectSpec`成功应用后, 就会返回一个名为`FActiveGameplayEffect`的新结构体.  

`GameplayEffectSpec`的重要内容:  

* 创建该`GameplayEffectSpec`的`GameplayEffect`类.
* 该`GameplayEffectSpec`的等级. 通常和创建`GameplayEffectSpec`的Ability的等级一样, 但是可以是不同的.
* `GameplayEffectSpec`的持续时间. 默认是`GameplayEffect`的持续时间, 但是可以是不同的.
* 对于周期性的Effect中`GameplayEffectSpec`的周期. 默认是`GameplayEffect`的周期, 但是可以是不同的.
* 该`GameplayEffectSpec`的当前堆栈数. 堆栈限制取决于`GameplayEffect`.
* GameplayEffectContextHandle表明该`GameplayEffectSpec`由谁创建.
* `Attribute`在`GameplayEffectSpec`创建时由于Snapshot被捕获.
* `GameplayEffectSpec`授予给目标的`DynamicGrantedTag`添加到`GameplayEffect`授予的`GameplayTag`中.
* `GameplayEffectSpec`的`DynamicAssetTag`添加到`GameplayEffect`的`AssetTag`中.
* SetByCaller TMaps.

##### 4.5.9.1 SetByCaller

`SetByCaller`允许`GameplayEffectSpec`拥有和`GameplayTag`或`FName`相关联的浮点值, 它们存储在`GameplayEffectSpec`上其各自的`TMaps: TMap<FGameplayTag, float>`和`TMap<FName, float>`中, 可以作为`GameplayEffect`中的`修改器`或者传递浮点值的一般方法使用. 其普遍用法是经由`SetByCaller`传递某个Ability内部生成的数值数据到`GameplayEffectExecutionCalculations`或`ModifierMagnitudeCalculations`.  

|SetByCaller使用|说明|
|:-:|:-:|
|`修改器`|必须提前在`GameplayEffect`类中定义. 只能使用`GameplayTag`形式. 如果在`GameplayEffect`类中定义而`GameplayEffectSpec`中没有相应的标签/浮点值对, 那么游戏在`GameplayEffectSpec`应用时会抛出运行时错误并返回0, 对于`Divide`操作这是个潜在问题, 参阅`Modifier`.|
|其他位置|无需提前定义. 读取`GameplayEffectSpec`中不存在的`SetByCaller`会返回一个由开发者定义的可带有警告信息的默认值.|

为了在蓝图中指定`SetByCaller`值, 请使用相应形式(`GameplayTag`或`FName`)的蓝图节点.  

![Assigning SetByCaller](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/`SetByCaller`.png)



