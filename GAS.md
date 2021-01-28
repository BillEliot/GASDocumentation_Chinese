**【更新日期：2021/01/28】**  

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
* 执行基于等级的角色能力(ability)或技能(skill), 该能力或技能可选花费和冷却时间. (GameplayAbilities)
* 管理属于actor的数值`属性`(Attributes). (Attributes)
* 为actor应用状态效果. (GameplayEffects)
* 为actor应用`GameplayTags`. (GameplayTags)
* 生成视觉或声音效果. (GameplayCues)
* 为以上提到的所有应用Replication.

在多人游戏中, GAS提供客户端预测(client-side prediction)支持:  
* 能力激活.
* 播放蒙太奇.
* 对`属性`(Attributes)的修改.
* 应用`GameplayTags`.
* 生成`GameplayCues`.
* 通过连接于`CharacterMovementComponent`的`RootMotionSource`函数形成的移动.

**GAS必须由C++创建**, 但是`GameplayAbilities`和`GameplayEffects`可由设计师在蓝图中创建.  
GAS中的现存问题:
* `GameplayEffect`延迟调节(latency reconciliation).(不能预测能力冷却时间, 导致高延迟玩家相比低延迟玩家, 对于短冷却时间的能力有更低的激活速率.)
* 不能预测`GameplayEffects`的移除(removal). 然而我们可以预测使用相反的效果添加`GameplayEffects`, 从而高效的移除它们. 但是这不总是合适或者可行的, 因此这仍然是个问题.
* 缺乏样例模板项目, 多人联机样例和文档. 希望这篇文档会有所帮助.

## 2. 样例项目

该文档包含一个支持多人联机的第三人称射击游戏模板项目, 其目标受众为初识GameplayAbilitySystem插件, 但并不是Unreal Engine 4新手. 用户应该了解C++, 蓝图, UMG, Replication和其他UE4的中间件. 该项目提供了一个样例, 其向你展示了如何使用GameplayAbilitySystem插件建立一个基础的支持多人联机的第三人称射击游戏, 其中AbilitySystemComponent(ASC)分别位于PlayerState类代表玩家/AI控制的人物和位于Character类代表AI控制的小兵.  
我在保证展现GAS基础和带有完整注释的代码所表示的一些普遍技能的同时, 尽力使这个样例足够简单. 由于该文档专注于初学者, 因此这个样例不包含像predicting projectiles这样的高级技术.  
概念说明:  
* `ASC`位于`PlayerState`还是`Character`.
* 网络同步的`Attributes`.
* 网络同步的蒙太奇(animation montages).
* `GameplayTags`.
* 在GameplayAbilities内部和外部应用和移除`GameplayEffects`.
* 应用被护甲防御后的伤害值来修改角色生命值.
* `GameplayEffectExecutionCalculations`.
* 眩晕效果.
* 死亡和重生.
* 在服务器上使用能力(ability)生成抛射物(projectile).
* 在瞄准和奔跑时, 预测性的修改本地玩家速度.
* 不断消耗耐力来奔跑.
* 消耗魔法值来使用能力(ability).
* 被动能力(ability).
* 堆栈`GameplayEffects`.
* 锁定actor.
* 在蓝图中创建`GameplayAbilities`.
* 在C++中创建`GameplayAbilities`.
* 实例化每个actor的`GameplayAbilities`.
* 非实例化的`GameplayAbilities`(Jump).
* 静态`GameplayCues`(子弹撞击粒子效果).
* Actor `GameplayCues`(奔跑和眩晕粒子效果).

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

`GameplayAbilities`无论由蓝图还是C++创建都没关系. 这里我们使用蓝图和C++混合创建, 意在展示每种方式的使用方法.  
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
3. 刷新/重新生成Visual Studio project项目文件.
4. 从4.24开始, 需要强制调用`UAbilitySystemGlobals::InitGlobalData()`来使用`TargetData`, 样例项目在`UEngineSubsystem::Initialize()`中调用该函数. 参阅`InitGlobalData()`获取更多信息.

这就是你启用GAS所需做的全部了. 从这里开始, 添加一个`ASC`和`AttributeSet`到你的`Character`或`PlayerState`, 并开始着手`GameplayAbilities`和`GameplayEffects`!

## 4. GAS概念

### 4.1 Ability System Component

`AbilitySystemComponent(ASC)`是GAS的核心, 它是一个处理所有与该系统交互的`UActorComponent(UAbilitySystemComponent)`, 所有期望使用 `GameplayAbility`, 拥有`Attributes`, 或者接受`GameplayEffects的Actor`都必须附加ASC. 这些对象都存于ASC并由其管理和同步(除了由AttributeSet同步的Attributes). 开发者最好但不强求继承该组件.  

`ASC`附加的`Actor`被引用作为该`ASC`的`OwnerActor`, 该`ASC`的物理代表`Actor`被称为`AvatarActor`. `OwnerActor`和`AvatarActor`可以是同一个 `Actor`, 比如MOBA游戏中的一个简单AI小兵; 它们也可以是不同的`Actor`, 比如MOBA游戏中玩家控制的英雄, 其中`OwnerActor`是`PlayerState`, `AvatarActor`是英雄的`Character`类. 绝大多数Actor的ASC都附加在其自身, 如果你的Actor会重生并且重生时需要持久化`Attributes`或`GameplayEffects`(比如MOBA中的英雄), 那么ASC理想的位置就是`PlayerState`.  

**Note:** 如果ASC位于PlayerState, 那么你需要增加PlayerState的`NetUpdateFrequency`, 其默认值是一个很低的值, 因此在客户端上发生像`Attributes`和`GameplayTags`改变时会造成延迟或卡顿. 确保启用`Adaptive Network Update Frequency`, Fortnite就启用了该项.  

如果OwnerActor和AvatarActor是不同的Actor, 应该执行`IAbilitySystemInterface`, 该接口有一个必须重写的函数, `UAbilitySystemComponent* GetAbilitySystemComponent() const`, 其返回一个指向ASC的指针, ASC通过寻找该接口函数来和系统内部进行交互.  

ASC在`FActiveGameplayEffectsContainer ActiveGameplayEffects`中保存其当前活跃的`GameplayEffects`.  

ASC在`FGameplayAbilitySpecContainer ActivatableAbilities`中保存其使用的`GameplayAbilities`. 当你想遍历`ActivatableAbilities.Items`时, 确保在循环体之上添加`ABILITYLIST_SCOPE_LOCK();`来锁定列表以防其改变(比如移除一个ability). 每个域中的`ABILITYLIST_SCOPE_LOCK();`会增加`AbilityScopeLockCount`, 之后出域时会减量. 不要尝试在`ABILITYLIST_SCOPE_LOCK();`域中移除某个ability(ability删除函数会在内部检查`AbilityScopeLockCount`以防在列表锁定时移除ability).  

#### 4.1.1 同步模式

ASC定义了三种不同的同步模式用于同步`GameplayEffects`, `GameplayTags`和`GameplayCues` - `Full`, `Mixed`和`Minimal`. Attributes由其AttributeSet同步.  

|同步模式|何时使用|描述|
|:-:|:-:|:-:|
|Full|单人|每个GameplayEffect同步到客户端.|
|Mixed|多人, 玩家控制的Actor|GameplayEffect只同步到其所属客户端, 只有GameplayTags和GameplayCues同步到所有客户端.|
|Minimal|多人, AI控制的Actor|GameplayEffect从不同步到任何客户端, 只有GameplayTags和GameplayCues同步到所有客户端.|

**Note:** Mixed同步模式需要OwnerActor的Owner是Controller. PlayerState的Owner默认是Controller但是Character不是. 如果OwnerActor不是PlayerState时使用Mixed同步模式, 那么需要在OwnerActor中调用SetOwner()设置Controller.  

从4.24开始, 需要使用PossessedBy()设置新的Controller为Pawn的Owner.  

#### 4.1.2 设置和初始化

ASC一般在OwnerActor的构建函数中构建并且需要明确标记为replicated. **这必须在C++中完成.**  

```c++
AGDPlayerState::AGDPlayerState()
{
	// Create ability system component, and set it to be explicitly replicated
	AbilitySystemComponent = CreateDefaultSubobject<UGDAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
	AbilitySystemComponent->SetIsReplicated(true);
	//...
}
```

OwnerActor和AvatarActor的ASC在服务端和客户端上均需初始化, 你应该在Pawn的Controller设置之后初始化(possession之后), 单机游戏只需考虑服务端的做法.  

对于玩家控制的Character且ASC位于Pawn, 我一般在服务端Pawn的PossessedBy()函数中初始化, 在客户端PlayerController的AcknowledgePossession()函数中初始化.  

```c++
void APACharacterBase::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	if (AbilitySystemComponent)
	{
		AbilitySystemComponent->InitAbilityActorInfo(this, this);
	}

	// ASC MixedMode replication requires that the ASC Owner's Owner be the Controller.
	SetOwner(NewController);
}

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

对于玩家控制的Character且ASC位于PlayerState, 我一般在服务端Pawn的PossessedBy()函数中初始化, 在客户端PlayerController的OnRep_PlayerState()函数中初始化, 这确保PlayerState存在于客户端上.  

```c++
void AGDHeroCharacter::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the ASC on the Server. Clients do this in OnRep_PlayerState()
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// AI won't have PlayerControllers so we can init again here just to be sure. No harm in initing twice for heroes that have PlayerControllers.
		PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
	}
	
	//...
}

// Client only
void AGDHeroCharacter::OnRep_PlayerState()
{
	Super::OnRep_PlayerState();

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the ASC for clients. Server does this in PossessedBy.
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// Init ASC Actor Info for clients. Server will init its ASC when it possesses a new Actor.
		AbilitySystemComponent->InitAbilityActorInfo(PS, this);
	}

	// ...
}
```

如果你得到了错误消息`LogAbilitySystem: Warning: Can't activate LocalOnly or LocalPredicted ability %s when not local!`, 那么表明ASC没有在客户端中初始化.  

### 4.2 Gameplay Tags

`FGameplayTags`是由`GameplayTagManager`注册的形似`Parent.Child.Grandchild...`的层级Name, 这些标签对于分类和描述对象的状态非常有用, 例如, 如果某个Character处于眩晕状态, 我们可以给一个`State.Debuff.Stun`的`GameplayTag`.  

你会发现自己用GameplayTag替换了过去使用布尔值或枚举值处理的事情, 并且需要对对象有无特定的GameplayTag做布尔逻辑判断.  

当给某个对象设置标签时, 如果它有ASC的话, 我们一般添加标签到ASC, 因此GAS可以与其交互. UAbilitySystemComponent执行IGameplayTagAssetInterface接口的函数来访问其拥有的GameplayTag.  

多个GameplayTag可被保存于一个FGameplayTagContainer中, 相比`TArray<FGameplayTag>`, 最好使用GameplayTagContainer, 因为GameplayTagContainer做了一些很有效率的优化. 因为标签是标准的FName, 所以当在project setting中启用`Fast Replication`后, 它们可以高效地打包进FGameplayTagContainers以用于同步. `Fast Replication`要求服务端和客户端有相同的GameplayTag列表, 这通常不是问题, 因此你应该启用该选项. GameplayTagContainer也可以返回`TArray<FGameplayTag>`以用于遍历.  

保存于`FGameplayTagCountContainer`的GameplayTag有一个保存该GameplayTag实例数的`TagMap`. FGameplayTagCountContainer可能存有TagMapCount为0的GameplayTag, 如果ASC仍有一个GameplayTag, 你可能在debug时遇到这种情况. 任何HasTag()或HasMatchingTag()或其他相似的函数会检查TagMapCount, 如果GameplayTag不存在或者其TagMapCount为0就会返回false.  

GameplayTag必须在`DefaultGameplayTags.ini`中提前定义, UE4编辑器在project setting中提供了一个界面用于让开发者管理GameplayTag而无需手动编辑DefaultGameplayTags.ini, 该GameplayTag编辑器可以创建, 重命名, 搜索引用和删除GameplayTag.  

![](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/gameplaytageditor.png)  

搜索GameplayTag引用会弹出一个类似Reference Viewer的窗口来显示所有引用该GameplayTag的资源, 但这不会显示任何引用该GameplayTag的C++类.  

重命名GameplayTag会创建重定向, 因此仍引用原来GameplayTag的资源会重定向到新的GameplayTag. 如果可以的话, 我更倾向于创建新的GameplayTag, 手动更新所有引用到新的GameplayTag, 之后删除旧的GameplayTag以避免创建新的重定向.  

除了`Fast Replication`, GameplayTag编辑器还有一个选项来填充通常需要同步的GameplayTag以对其深度优化.  

如果GameplayTag由GameplayEffect添加, 那么其就是可同步的. ASC允许你添加不可同步的LooseGameplayTag且必须手动管理. 样例项目对`State.Dead`使用了LooseGameplayTag, 因此当生命值降为0时, 其所属客户端会立即响应. 重生时需要手动将TagMapCount设置回0, 当使用LooseGameplayTag时只能手动调整TagMapCount, 相比纯手动调整TagMapCount, 最好使用UAbilitySystemComponent::AddLooseGameplayTag()和UAbilitySystemComponent::RemoveLooseGameplayTag().  

C++中获取GameplayTag引用:  

```c++
FGameplayTag::RequestGameplayTag(FName("Your.GameplayTag.Name"))
```

对于像获取父或子GameplayTag的高级操作, 请查看GameplayTagManager提供的函数. 为了访问GameplayTagManager, 请引用GameplayTagManager.h并使用UGameplayTagManager::Get().FunctionName调用函数. 相比使用常量字符串进行操作和比较, GameplayTagManager实际上使用关系节点(父, 子等等)保存GameplayTag以获得更快的处理速度.  

GameplayTag和GameplayTagContainer有可选UPROPERTY宏`Meta = (Categories = "GameplayCue")`用于在蓝图中过滤标签而只显示父标签为GameplayCue的GameplayTag, 当你知道GameplayTag或GameplayTagContainer变量应该只用于GameplayCue时, 这将是非常有用的.  

可选地, 有一单独的FGameplayCueTag结构体可以包裹FGameplayTag并且可以在蓝图中自动过滤GameplayTag而只显示父标签为GameplayCue的标签.  

如果你想过滤函数中的GameplayTag参数, 使用UFUNCTION宏`Meta = (GameplayTagFilter = "GameplayCue")`. GameplayTagContainer参数不能过滤, 如果你想编辑引擎来允许过滤GameplayTagContainer参数, 查看`SGameplayTagGraphPin::ParseDefaultValueData()`是如何从`Engine\Plugins\Editor\GameplayTagsEditor\Source\GameplayTagsEditor\Private\SGameplayTagGraphPin.cpp`中调用`FilterString = UGameplayTagsManager::Get().GetCategoriesMetaFromField(PinStructType);`的, 还有是如何在`SGameplayTagGraphPin::GetListContent()`中将`FilterString`传递给`SGameplayTagWidget`的, `Engine\Plugins\Editor\GameplayTagsEditor\Source\GameplayTagsEditor\Private\SGameplayTagContainerGraphPin.cpp`中这些函数的`GameplayTagContainer`版本并没有检查meta域属性和传递过滤器.  

样例项目广泛地使用了GameplayTag.  

#### 4.2.1 响应Gameplay Tags的变化

ASC提供了一个委托(delegate)用于GameplayTag添加或移除时触发, 其中EGameplayTagEventType参数可以明确只有GameplayTag添加/移除还是该GameplayTag的TagMapCount发生变化时触发.  

```c++
AbilitySystemComponent->RegisterGameplayTagEvent(FGameplayTag::RequestGameplayTag(FName("State.Debuff.Stun")), EGameplayTagEventType::NewOrRemoved).AddUObject(this, &AGDPlayerState::StunTagChanged);
```

回掉函数有一个GameplayTag参数和新的TagCount.  

```c++
virtual void StunTagChanged(const FGameplayTag CallbackTag, int32 NewCount);
```

### 4.3 Attributes

#### 4.3.1 Attribute定义

Attribute是由`FGameplayAttributeData`结构体定义的浮点值, 其可以表示从角色生命值到角色等级再到一瓶药水的剂量的任何事物, 如果某项数值是属于某个Actor且游戏相关的, 你就应该考虑使用Attribute. Attribute一般应该只能由GameplayEffect修改, 这样ASC才能预测(predict)其改变.  

Attribute也可以由AttributeSet定义并存于其中. AttributeSet用于同步那些标记为replication的Attribute. 参阅AttributeSets部分来了解如何定义Attribute.  

**Tip**: 如果你不想某个Attribute显示在编辑器的Attribute列表, 可以使用`Meta = (HideInDetailsView)`属性宏.  

#### 4.3.2 BaseValue vs. CurrentValue

一个Attribute是由两个值 —— 一个BaseValue和一个CurrentValue 组成的, BaseValue是Attribute的永久值而CurrentValue是BaseValue加上GameplayEffect给的临时修改值后得到的. 例如, 你的Character可能有一个BaseValue为600u/s的移动速度Attribute, 因为还没有GameplayEffect修改移动速度, 所以CurrentValue也是600u/s, 如果Character获得了一个临时50u/s的移动速度加成, 那么BaseValue仍然是600u/s而CurrentValue是600+50=650u/s, 当该移动速度加成消失后, CurrentValue就会变回BaseValue的600u/s.  

初识GAS的新手经常将BaseValue误认为Attribute的最大值并以这样的认识去编码, 这是错误的, 可以改变或引用在Ability/UI中的Attribute最大值应该是另外单独的Attribute. 对于硬编码的最大值和最小值, 有一种方法是使用可以设置最大值和最小值的FAttributeMetaData定义一个DataTable, 但是Epic在该结构体上的注释称之为"work in progress", 参阅`AttributeSet.h`获得更多信息. 为了避免这种疑惑, 我建议引用在Ability或UI中的最大值应该单独定义Attribute, 只用于限制(clamp)Attribute大小的硬编码最大值和最小值应该在AttributeSet中定义为硬编码浮点值. 关于Attribute值的限制(clamp)在PreAttributeChange()中讨论了修改CurrentValue, 在PostGameplayEffectExecute()中讨论了修改GameplayEffect的BaseValue.  

即时(Instant)GameplayEffect可以永久性的修改BaseValue, 而持续(Duration)和无限(Infinite)GameplayEffect可以修改CurrentValue. 周期性(Periodic)GameplayEffect被视为GameplayEffect实例并且可以修改BaseValue.  

#### 4.3.3 元(Meta)Attributes

一些Attribute被视为占位符, 其用于预计和Attribute交互的临时值, 这些Attribute被叫做`Meta Attribute`. 例如, 我们通常定义伤害值为`Meta Attribute`, 使用伤害值Meta Attribute作为占位符, 而不是使用GameplayEffect直接修改生命值Attribute, 使用这种方法, 伤害值就可以在GameplayEffectExecutionCalculation中由buff和debuff修改, 并且可以在AttributeSet中进一步操作, 例如, 在最终将生命值减去伤害值之前, 要将伤害值减去当前的护盾值. 伤害值Meta Attribute在GameplayEffect之间不是持久化的, 并且可以被任何一方重写. Meta Attribute一般是不可同步的.  

Meta Attribute对于在"我们应该造成多少伤害?"和"我们该如何处理伤害值?"这种问题之中的伤害值和治疗值做了很好的解构, 这种解构意味着GameplayEffect和ExecutionCalculation无需了解目标是如何处理伤害值的. 继续看伤害值的例子, GameplayEffect确定造成多少伤害, 之后AttributeSet决定如何使用该伤害值, 不是所有的Character都有相同的Attribute, 特别是使用了AttributeSet子类的话, AttributeSet基类可能只有一个生命值Attribute, 但是它的子类可能增加了一个护盾值Attribute, 拥有护盾值Attribute的子类AttributeSet可能会以不同于AttributeSet基类的方式分配收到的伤害.  

尽管Meta Attribute是一个很好的设计模式, 但它们并不是强制使用的. 如果你只有一个用于所有伤害实例的Execution Calculation和一个所有Character共用的AttributeSet类, 那么你就可以在Exeuction Calculation中分配伤害到生命, 护盾等等, 并直接修改那些Attribute, 这种方式你只会丢失灵活性, 但总体上并无大碍.  

#### 4.3.4 响应Attribute变化

为了监听Attribute何时变化以便更新UI和其他游戏逻辑, 可以使用`UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate(FGameplayAttribute Attribute)`, 该函数返回一个委托(Delegate), 你可以将其绑定一个当Attribute变化时需要自动调用的函数. 该委托提供一个`FOnAttributeChangeData`参数, 其中有`NewValue`, `OldValue`和`FGameplayEffectModCallbackData`. **Note**: `FGameplayEffectModCallbackData`只能在服务端上设置.  

```c++
AbilitySystemComponent->GetGameplayAttributeValueChangeDelegate(AttributeSetBase->GetHealthAttribute()).AddUObject(this, &AGDPlayerState::HealthChanged);

virtual void HealthChanged(const FOnAttributeChangeData& Data);
```

样例项目将其绑定到了`GDPlayerState`用于更新HUD, 当生命值下降为0时, 也可以响应玩家死亡.  

样例项目中有一个将上述逻辑包裹进`ASyncTask`的自定义蓝图节点, 其在`UI_HUD(UMG Widget)`中用于更新生命值, 魔法值和耐力值. 该AsyncTask会一直响应直到手动调用`EndTask()`, 就像在UMG Widget的`Destruct`事件中调用. 参阅`AsyncTaskAttributeChanged.h/cpp`.  

![](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/attributechange.png)  

#### 4.3.5 自动推导Attribute

为了使一个Attribute的部分或全部值继承自一个或更多Attribute, 可以使用基于一个或多个Attribute或`MMC Modifiers`的无限(Infinite)GameplayEffect. 当自动推导Attribute依赖的某个Attribute更新时它也会自动更新.  

在自动推导Attribute上的所有Modifier形成的最终公式和`Modifier Aggregators`的公式是一样的. 如果你需要计算式要按一定的顺序进行, 在`MMC`中做就是了.  

```c++
((CurrentValue + Additive) * Multiplicitive) / Division
```

**Note**: 如果在PIE中打开多个窗口, 你需要在编辑器首选项中禁用`Run Under One Process`, 否则当自动推导Attribute所依赖的Attribute更新时, 除了第一个窗口外其不会更新.  

在这个例子中, 我们有一个无限(Infinite)GameplayEffect, 其从TestAttrB和TestAttrC Attribute以`TestAttrA = (TestAttrA + TestAttrB) * ( 2 * TestAttrC)`公式继承得到TestAttrA, 每次TestAttrB和TestAttrC更新时, TestAttrA都会自动重新计算.  

![](https://raw.githubusercontent.com/tranek/GASDocumentation/master/Images/derivedattribute.png)  

### 4.4 Attribute Set

#### 4.4.1 定义Attribute Set

AttributeSet用于定义, 保存和管理Attribute的变化. 开发者应该继承UAttributeSet. 在OwnerActor的构建函数中创建AttributeSet会自动注册到其ASC. **这必须在C++中完成.**  

#### 4.4.2 设计Attribute Set

一个ASC可能有一个或很多AttributeSet, AttributeSet消耗的内存微不足道, 使用多少AttributeSet是留给开发人员决定的.  

有种方案是设置一个单一且巨大的AttributeSet, 共享于游戏中的所有Actor, 并且只使用需要的Attribute, 忽略不用的Attribute.  

作为选择, 你可以使用多个AttributeSet表示按需添加到Actor的Attribute分组, 例如, 你可以有一个生命相关的AttributeSet, 一个魔法相关的AttributeSet, 等等. 在MOBA游戏中, 英雄可能需要魔法, 但是小兵并不需要, 因此英雄就需要魔法AttributeSet而小兵就不需要.  

另外, 继承AttributeSet的另一种意义是可以选择一个Actor可以有哪些Attribute. Attribute在内部被引用为`AttributeSetClassName.AttributeName`, 当你继承AttributeSet时, 所有父类的Attribute将仍保留父类名作为前缀.  

尽管可以拥有多个AttributeSet, 但是不应该在同一ASC中拥有多个同一类的AttributeSet, 如果在同一ASC中有多个同一类的AttributeSet, ASC就不知道该使用哪个AttributeSet而随机选择一个.  

##### 4.4.2.1 使用单独Attribute的子组件

假设你在某个Pawn上有多个可被损害的组件, 像可被独立损害的护甲片, 如果可以明确可被损害组件的最大数量, 我建议把多个生命值Attribute放到一个AttributeSet中 —— DamageableCompHealth0, DamageableCompHealth1, 等等, 以表示这些可被损害组件在逻辑上的"slot", 在可被损害组件的类实例中, 指定可以被GameplayAbility和Executions读取的带slot编号的Attribute来表明该应用伤害值到哪个Attribute. 如果Pawn当前拥有0个或少于最大数量的可损害组件也无妨, 因为AttributeSet拥有一个Attribute, 并不意味着必须要使用它, 未使用的Attribute只占用很少的内存.  

如果每个子组件都需要很多Attribute且子组件的数量可以是无限的, 或者子组件可以分离被其他玩家使用(比如武器), 或者出于其他原因上述方法不适用于你, 那么我建议就不要使用Attribute, 而是在组件中保存普通的浮点数. 参阅Item Attributes.  

##### 4.4.2.2 运行时添加和移除AttributeSet

AttributeSet可以在运行时从ASC上添加和移除, 然而移除AttributeSet是很危险的, 例如, 如果某个AttributeSet在客户端上移除早于服务端, 而某个Attribute的变化又同步到了客户端, 那么Attribute就会因为找不到AttributeSet而使游戏崩溃.  

武器添加到Inventory:  

```c++
AbilitySystemComponent->SpawnedAttributes.AddUnique(WeaponAttributeSetPointer);
AbilitySystemComponent->ForceReplication();
```

武器从Inventory移除:  

```c++
AbilitySystemComponent->SpawnedAttributes.Remove(WeaponAttributeSetPointer);
AbilitySystemComponent->ForceReplication();
```

##### 4.4.2.3 Item Attributes(武器弹药)

有几种方法可以实现带有Attribute(武器弹药, 盔甲耐久等等)的可装备物品, 所有这些方法都直接在物品中存储数据, 这对于在生命周期中可以被多个玩家装备的物品来说是必须的.  

1. 在物品中使用普通的浮点数(推荐).
2. 在物品中使用单独的AttributeSet.
3. 在物品中使用单独的ASC.

###### 4.4.2.3.1 在物品中使用普通浮点数

在物品类实例中存储普通浮点数而不是Attribute, Fortnite和GASShooter就是这样处理枪械子弹的, 对于枪械, 在其实例中存储可同步的浮点数(COND_OwnerOnly), 比如最大弹匣量, 当前弹匣中弹药量, 剩余弹药量等等, 如果枪械需要共享剩余弹药量, 那么就将剩余弹药量移到Character中共享的弹药AttributeSet里作为一个Attribute(换弹ability可以使用一个`Cost GE`从剩余弹药量中填充枪械的弹匣弹药量浮点). 因为没有为当前弹匣弹药量使用Attribute, 所以需要重写UGameplayAbility中的一些函数来检查和应用枪械中浮点数的花销(cost). 当使用ability时将枪械在GameplayAbilitySpec中转换为SourceObject, 这意味着可以在ability中访问使用ability的枪械.  

为了防止在全自动射击过程中枪械会反向同步弹药量并扰乱本地弹药量(译者注: 通俗解释就是因为存在同步延迟且在连续射击这一高同步过程中, 所以客户端的弹药量会来不及和服务端同步, 造成弹药量减少后又突然变多的现象.), 如果玩家拥有IsFiring的GameplayTag, 就在PreReplication()中禁用同步, 本质上是要在其中做自己的本地预测.  

```c++
void AGSWeapon::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
	Super::PreReplication(ChangedPropertyTracker);

	DOREPLIFETIME_ACTIVE_OVERRIDE(AGSWeapon, PrimaryClipAmmo, (IsValid(AbilitySystemComponent) && !AbilitySystemComponent->HasMatchingGameplayTag(WeaponIsFiringTag)));
	DOREPLIFETIME_ACTIVE_OVERRIDE(AGSWeapon, SecondaryClipAmmo, (IsValid(AbilitySystemComponent) && !AbilitySystemComponent->HasMatchingGameplayTag(WeaponIsFiringTag)));
}
```

好处:  

1. 避免了使用AttributeSet的局限(见下).

局限:  

1. 不能使用现有的GameplayEffect工作流(弹药使用的Cost GEs等等).
2. 要求重写UGameplayAbility中的关键函数来检查和应用枪械中浮点数的花销(cost).

###### 4.4.2.3.2 在物品中使用AttributeSet

在物品中使用单独的AttributeSet可以实现"将其添加到玩家的Inventory", 但还是有一定的局限性. 较早版本的GASShooter中的武器弹药是使用的这种方法, 武器类在其自身存储诸如最大弹匣量, 当前弹匣弹药量, 剩余弹药量等等到一个AttributeSet, 如果枪械需要共享剩余弹药量, 那么就将剩余弹药量移到Character中共享的弹药AttributeSet里. 当服务器上某个武器添加到玩家的Inventory后, 该武器会将它的AttributeSet添加到玩家的`ASC::SpawnedAttributes`, 之后服务器会将其同步下发到客户端, 如果该武器从Inventory中移除, 它也会将其AttributeSet从ASC::SpawnedAttributes中移除.  

当AttributeSet存于除了OwnerActor之外的对象上时(对于某个武器来说), 会得到一些关于AttributeSet的编译错误, 解决办法是在BeginPlay()中构建AttributeSet而不是在构造函数中, 并在武器类中实现IAbilitySystemInterface(当你添加武器到玩家Inventory时设置ASC的指针).  

```c++
void AGSWeapon::BeginPlay()
{
	if (!AttributeSet)
	{
		AttributeSet = NewObject<UGSWeaponAttributeSet>(this);
	}
	//...
}
```

你可以查看较早版本的GASShooter来实际地体会这种方案.  

好处:  
1. 可以使用已有的GameplayAbility和GameplayEffect工作流(弹药使用的Cost GEs等等).
2. 对于很小的物品集可以快速设置

局限:  
1. 必须为每个武器类型创建新的AttributeSet类, ASC实际上只能有一个该类的AttributeSet实例, 因为对Attribute的修改会在ASC的SpawnedAttributes数组中寻找其第一个AttributeSet类实例, 其他相同的AttributeSet类实例则会被忽略.  
2. 和第1条同样的原因(每个AttributeSet类一个AttributeSet实例), 在玩家的Inventory中每种武器类型只能有一个.
3. 移除AttributeSet是很危险的. 在GASShooter中, 如果玩家因为火箭弹而自杀, 玩家会立即从其Inventory中移除火箭弹发射器(包括其在ASC中的AttributeSet), 当服务端同步火箭弹发射器的弹药Attribute改变时, 由于AttributeSet在客户端ASC上不复存在而使游戏崩溃.

###### 4.4.2.3.3 在物品中使用单独的ASC

在每个物品上都创建一个AbilitySystemComponent是种很极端的方案. 我还没有亲自做过这种方案, 在其他地方也没见过. 这种方案应该会花费相当的开发成本才能正常使用.  

> Is it viable to have several AbilitySystemComponents which have the same owner but different avatars (e.g. on pawn and weapon/items/projectiles with Owner set to PlayerState)?

> The first problem I see there would be implementing the IGameplayTagAssetInterface and IAbilitySystemInterface on the owning actor. The former may be possible: just aggregate the tags from all all ASCs (but watch out -HasAlMatchingGameplayTags may be met only via cross ASC aggregation. It wouldn't be enough to just forward that calls to each ASC and OR the results together). But the later is even trickier: which ASC is the authoritative one? If someone wants to apply a GE -which one should receive it? Maybe you can work these out but this side of the problem will be the hardest: owners will multiple ASCs beneath them.

> Separate ASCs on the pawn and the weapon can make sense on its own though. E.g, distinguishing between tags the describe the weapon vs those that describe the owning pawn. Maybe it does make sense that tags granted to the weapon also “apply” to the owner and nothing else (E.g, attributes and GEs are independent but the owner will aggregate the owned tags like I describe above). This could work out, I am sure. But having multiple ASCs with the same owner may get dicey.

*[community questions #6](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)中来自Epic的Dave Ratti的回答.*  

好处: 
1. 可以使用已有的GameplayAbility和GameplayEffect工作流(弹药使用的Cost GEs等等).
2. 可以复用AttributeSet类(每个武器的ASC中各一个).

局限:  
1. 未知的开发成本.
2. 甚至方案可行么?

#### 4.4.3 定义Attribute

**Attribute只能使用C++在AttributeSet头文件中定义.** 建议把下面这个宏块加到每个AttributeSet头文件的顶部, 其会自动为每个Attribute生成getter和setter函数.  

```c++
// Uses macros from AttributeSet.h
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
```

一个可同步的生命值Attribute可能像下面这样定义:  

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

AttributeSet的.cpp文件应该用预测系统(prediction system)使用的GAMEPLAYATTRIBUTE_REPNOTIFY宏填充OnRep函数:  

```c++
void UGDAttributeSetBase::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(UGDAttributeSetBase, Health, OldHealth);
}
```

最后, Attribute需要添加到GetLifetimeReplicatedProps:  

```c++
void UGDAttributeSetBase::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME_CONDITION_NOTIFY(UGDAttributeSetBase, Health, COND_None, REPNOTIFY_Always);
}
```

`REPTNOTIFY_Always`告知OnRep函数如果本地值已经和从服务端同步下来的值相同时触发(预测的原因), 默认当本地值和从服务端同步下来的值相同时, OnRep函数是不会触发的.  

如果Attribute无需像Meta Attribute那样同步, 那么OnRep和GetLifetimeReplicatedProps步骤可以跳过.  

#### 4.4.4 初始化Attribute

有多种方法可以初始化Attribute(将BaseValue和CurrentValue设置为某初始值). Epic建议使用即刻(instant)GameplayEffect, 这也是样例项目使用的方法.  

查看样例项目的GE_HeroAttributes蓝图来了解如何创建即刻(instant)GameplayEffect以初始化Attribute, 该GameplayEffect应用是写在C++中的.  

如果在定义Attribute时使用了ATTRIBUTE_ACCESSORS宏, 那么在AttributeSet中会自动为每个Attribute生成一个初始化函数.  

```c++
// InitHealth(float InitialValue) is an automatically generated function for an Attribute 'Health' defined with the `ATTRIBUTE_ACCESSORS` macro
AttributeSet->InitHealth(100.0f);
```

查看AttributeSet.h获取更多初始化Attribute的方法.  

**Note**: 4.24之前, FAttributeSetInitterDiscreteLevels不能和FGameplayAttributeData协同使用, 它在Attribute是原生浮点数时创建, 并且会和FGameplayAttributeData不是Plain Old Data(POD)时冲突. 该问题在4.24中修复([https://issues.unrealengine.com/issue/UE-76557.](https://issues.unrealengine.com/issue/UE-76557)).  

#### 4.4.5 PreAttributeChange()

`PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)`是AttributeSet中主要的函数之一, 其在变化发生前响应Attribute的CurrentValue的变化, 其是通过引用参数NewValue限制(clamp)CurrentValue即将发生的改变的理想位置.  

例如像样例项目那样限制移动速度修改器:  
```c++
if (Attribute == GetMoveSpeedAttribute())
{
	// Cannot slow less than 150 units/s and cannot boost more than 1000 units/s
	NewValue = FMath::Clamp<float>(NewValue, 150, 1000);
}
```

GetMoveSpeedAttribute()函数是由我们在AttributeSet.h中添加的宏块创建的(Defining Attributes).  

PreAttributeChange()可以由Attribute的任何改变触发, 无论是使用Attribute的setter(由AttributeSet.h中的宏块定义(Defining Attributes))还是使用GameplayEffects.  

**Note**: 在这里做的任何限制都不会永久性地修改ASC中的修改器(Modifier), 只会修改查询修改器返回的值, 这意味着像`GameplayEffectExecutionCalculations`和`ModifierMagnitudeCalculations`这种从所有修改器处重新计算CurrentValue的函数需要再次执行限制(Clamp)操作.  

**Note**: Epic对于PreAttributeChange()的注释说明不要将该函数用于游戏逻辑事件, 而主要在其中做限制操作. 对于修改Attribute的游戏逻辑事件的建议位置是`UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate(FGameplayAttribute Attribute)`(Responding to Attribute Changes).  

#### 4.4.6 PostGameplayEffectExecute()

`PostGameplayEffectExecute(const FGameplayEffectModCallbackData & Data)`会在某个来自即刻(Instant)GameplayEffect的Attribute的BaseValue变化之后触发, 当修改是来自GameplayEffect时, 这就是一个处理更多Attribute操作的有效位置.  

例如, 在样例项目中, 我们在这里从生命值Attribute中减去了最终的伤害值`Meta Attribute`, 如果有护盾值Attribute的话, 我们也会在减除生命值之前从护盾值中减除伤害值. 样例项目也在这里应用被击打反应动画, 显示浮动的伤害数值, 和为击杀者分配经验值和赏金. 通过设计, 伤害值`Meta Attribute`总是会通过某个即刻(Instant)GameplayEffect而不会通过Attribute setter.  

其他只会由即刻(Instant)GameplayEffect修改BaseValue的Attribute, 像魔法值和耐力值, 也可以在这里被限制为它们对应Attribute的最大值.  

**Note**: 当PostGameplayEffectExecute()被调用时, 对Attribute的改变已经发生, 但是还没有被同步回客户端, 因此在这里限制值不会造成对客户端的二次同步, 客户端只会接收到限制后的值.  

#### 4.4.7 OnAttributeAggregatorCreated()

`OnAttributeAggregatorCreated(const FGameplayAttribute& Attribute, FAggregator* NewAggregator)`会在Aggregator为集合中的某个Attribute创建时触发, 它允许FAggregatorEvaluateMetaData的自定义设置, AggregatorEvaluateMetaData是Aggregator基于所有应用的Modifier评估Attribute的CurrentValue的. 默认情况下, AggregatorEvaluateMetaData只由Aggregator用于确定哪些Modifier是合格的, 以MostNegativeMod_AllPositiveMods为例, 其允许所有正(Positive)修改器但是限制负(Negative)修改器(仅最负的那一个), 这在Paragon中只允许将最负移动速度减速效果应用到玩家, 而不用管应用所有正移动速度buff时有多少负移动效果. 不合格的Modifier仍存于ASC中, 只是不被聚合进最终的CurrentValue, 一旦条件改变, 它们之后就可能合格, 就像如果最负Modifier过期后, 下一个最负Modifier(如果存在的话)就是合格的.  

为了在只允许最负Modifier和所有正Modifier的例子中使用AggregatorEvaluateMetaData:  
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

你的自定义AggregatorEvaluateMetaData应该作为静态变量添加到FAggregatorEvaluateMetaDataLibrary.  

### 4.5 Gameplay Effects

#### 4.5.1 定义Gameplay Effect

GameplayEffect(GE)是Ability修改其自身和其他Attribute和GameplayTag的容器, 其可以立即修改Attribute(像伤害或治疗)或应用长期的状态buff/debuff(像移动速度加速或眩晕). UGameplayEffect只是一个定义单一游戏效果的数据类, 不应该在其中添加额外的逻辑. 设计师一般会创建很多UGameplayEffect的子类蓝图.  

GameplayEffect通过Modifier和Executions(GameplayEffectExecutionCalculation)修改Attribute.  

GameplayEffect有三种持续类型: 即刻(Instant), 持续(Duration)和无限(Infinite).  

额外地, GameplayEffect可以添加/执行GameplayCue, 即刻(Instant)GameplayEffect可以调用GameplayCue, GameplayTag中的Execute而持续(Duration)或无限(Infinite)可以调用GameplayCue, GameplayTag中的Add和Remove.  

|类型|GameplayCue事件|何时使用|
|:-:|:-:|:-:|
|即刻(Instant)|Execute|对于Attribute中BaseValue的永久性的立即修改. GameplayTag不会被应用, 哪怕是一帧.|
|持续(Duration)|Add & Remove|对于Attribute中CurrentValue的临时修改和当GameplayEffect过期或手动移除时, 应用将要被移除的GameplayTag. 持续时间是在UGameplayEffect类/蓝图中明确的.|
|无限(Infinite)|Add & Remove|对于Attribute中CurrentValue的临时修改和当GameplayEffect移除时, 应用将要被移除的GameplayTag. 该类型自身永不过期且必须由某个Ability或ASC手动移除.|


