---
layout: post
title: "游戏开发之网络同步技术"
date: 2023-10-11
last_modified_at: 2023-10-11
categories: [游戏开发]
---

* 目录  
{:toc}
<br/>
 
---

## 适用范围

帧同步与状态同步的区别，可以这么打比方：帧同步是所有客户端播放同一部电影，视觉上完全一样；状态同步是所有客户端拿着相同的剧本（状态一样），但是电影画面略有差异。而快照同步，由于所有的仿真都发生在服务器，所以各个客户端视觉上是一致的，也是在播放同一部电影。    

弱客户端，像小游戏，云游戏，都可以使用快照同步。    

对于面画有精确要求的，比如格斗类游戏，玩家的出招强依赖于对手的出招动作，所以保持双方视觉上的一致很有必要，此时就适用帧同步，此外，体育类、rts 等需要精细操作的也都得用帧同步。    

其他类型的，基本上都可以使用状态同步。    

王者荣耀使用的是帧同步，这个分享《王者技术修炼之路》[11]说明了理由：   

>第一，它的开发效率比较高。如果你开发思路的整体框架是验证可行的，如果你把它的缺点解决了，那么你的开发思路完全就跟写单机一样，你只需要遵从这样的思路，尽量保证性能，程序该怎么写就怎么写。比如我们以前要在状态同步下面做一个复杂的技能，有很多段位的技能，可能要开发好几天，才能有一个稍微过得去的结果，而在帧同步下面，英雄做多段位技能很可能半天就搞定了。   
第二，它能实现更强的打击感，打击感强除了我们说的各种反馈、特效、音效外，还有它的准确性。利用帧同步，游戏里面看到这些挥舞的动作，就能做到在比较准确的时刻产生反馈，以及动作本身的密度也可以做到很高的频率，这在状态同步下是比较难做的。   
第三，它的流量消耗是稳定的。    

moba 类型游戏其实是可以使用状态同步的，王者荣耀采用帧同步算是一种个性化的选择，也付出了很多努力来克服帧同步的弱点。不过看到分享里的这一段的时候，就知道王者荣耀很强调act，画面上追求打击感，那要这种效果，用帧同步显然是更适合的： 

>第二，它能实现更强的打击感，打击感强除了我们说的各种反馈、特效、音效外，还有它的准确性。利用帧同步，游戏里面看到这些挥舞的动作，就能做到在比较准确的时刻产生反馈，以及动作本身的密度也可以做到很高的频率，这在状态同步下是比较难做的。   


守望先锋使用的是状态同步，这个分享《《守望先锋》回放技术-阵亡镜头、全场最佳和亮眼表现》[8]说明了理由:   

>Overwatch最终没有选择确定性帧同步方案，原因如下：首先，我们不希望程序员和策划的因为偶现的不确定性而产生心理负担；除此之外，要支持中途加入游戏也不简单。模拟操作本身就已经很耗了，如果5分钟的游戏过程，每次都需要重头开始模拟的话，计算量太大。   
最后，实现一个阵亡镜头也会很困难，因为需要客户端快照以及快播机制。按理说如果游戏支持录像文件的话，快播自然就能做到，不过Overwatch并没有这一点。

---

## 帧同步的实现及优化手段

帧同步的确定性，要求各种平台之上的客户端计算都是确定的，这些都可能导致不确定计算：浮点数，随机数，执行顺序，调用时序，排序的稳定性，物理引擎。 
浮点数可以使用定点数替代。  
随机数可以统一随机数种子。  
执行顺序，要保持一致，需要所有的逻辑要有一个统一的入口，每次 tick update 进入一个统一的入口，依次调用各个模块的逻辑。   
排序的稳定性，可以指定统一的稳定排序算法。  
物理引擎，要求确定性的模拟，需要选用保证确定性的物理引擎。  

帧同步的挑战很大，由于误差累积会变大，基本上只要有一次计算不一致，那后续结果就都不一致了，游戏也就玩不下去了，王者荣耀的这个分享[11]就讲了很多这一方面的努力。   

可以使用的优化手段包括以下这些：     

#### 乐观帧

现在事实意义上的帧同步算法都是用的乐观帧了，即每帧固定时长，超时不等待。  

但这里有个细节问题，客户端发送给服务端的 input 数据包都是带有客户端帧号的，那么服务端是否要抛弃客户端过时的 input 数据包，即客户端帧号小于当前服务端帧号的数据包？   

比如这个 demo 项目（[https://github.com/JiepengTan/Lockstep-Tutorial](https://github.com/JiepengTan/Lockstep-Tutorial)）就是会抛弃客户端 input 数据包的。 [https://github.com/JiepengTan/Lockstep-Tutorial/blob/master/Server/Src/SimpleServer/Src/Server/Game.cs](https://github.com/JiepengTan/Lockstep-Tutorial/blob/master/Server/Src/SimpleServer/Src/Server/Game.cs):    

```cs
void C2G_PlayerInput(Player player, BaseMsg data){
    ...
    if (input.Tick < Tick) {
        return;
    }
    ...
}
```

这样抛弃是否会带来问题？似乎是有问题的，即一个延迟高的客户端，它的 input 永远不会被服务端应用。我认为这样是不妥的，那么如何实现才是好的呢？   

参考另一个 demo ( [https://github.com/Enanyy/Frame](https://github.com/Enanyy/Frame) )，这个实现不会抛弃客户端过时的 input 数据包，代码在此（ [https://github.com/Enanyy/Frame/blob/master/FrameServer/FrameServer/Program.cs](https://github.com/Enanyy/Frame/blob/master/FrameServer/FrameServer/Program.cs) ）：

```cs
private void OnOptimisticFrame(Session client, GM_Frame recvData)
{

    int roleId = recvData.roleId;

    long frame = recvData.frame;

    Debug.Log(string.Format("Receive roleid={0} serverframe:{1} clientframe:{2} command:{3}", roleId, mCurrentFrame, frame,recvData.command.Count),ConsoleColor.DarkYellow);
    
    if (mFrameDic.ContainsKey(mCurrentFrame) == false)
    {
        mFrameDic[mCurrentFrame] = new Dictionary<int, List<Command>>();
    }
    for (int i = 0; i < recvData.command.Count; ++i)
    {
        //乐观模式以服务器收到的时间为准
        Command frameData = new Command(recvData.command[i].frame, recvData.command[i].type, recvData.command[i].data, mFrameTime);
        if (mFrameDic[mCurrentFrame].ContainsKey(roleId) == false)
        {
            mFrameDic[mCurrentFrame].Add(roleId, new List<Command>());
        }
        mFrameDic[mCurrentFrame][roleId].Add(frameData);
    }
}
```


#### buffering

针对延迟以及网络抖动，可以通过增加缓冲区的方式来对抗：
输入 -> 缓冲区 -> 渲染   
缓冲区的问题在于会增加延迟。  


#### 预测回滚

不止是状态同步，帧同步也是可以 “预测回滚” 的，但叫法是 timewarp。大体做法都是记录快照，然后出现冲突的时候回滚到快照点。韦易笑的这篇文章《帧同步游戏中使用 Run-Ahead 隐藏输入延迟》[14]介绍过这种做法。   

---

## 快照同步的实现及优化手段

快照同步的优化措施，也就是 buffering 了，通过增加缓冲区来对抗网络延迟及抖动。另外，就是根据业务的特点，压缩快照的数据量，减少带宽压力，Glenn Fielder 的这篇文章《Snapshot Compression》[6]提供了一次很好的展示。   

这篇文章《Cocos 技术派｜实时竞技小游戏技术实现分享》[9]还讲到使用 DR 来对快照同步做优化，很奇怪，既然都已经选择快照同步了，为何还使用 DR？不应该是 buffering + 逻辑与显示分离+内插值吗？  

---

## 状态同步的实现及优化手段

#### 客户端预测回滚 (client-side predition and rollback)

客户端预测是为了更及时的反馈，如果玩家自己的每一个动作都要等待服务器的回应才执行，那么会严重受到延迟的影响，体验很糟糕，所以一般都会采取客户端预测，即客户端先行，客户端在把 input 发给服务器的同时，自己先执行动作，待等到服务器回包的时候再根据情况，如果状态不一致，则需要回滚（Gabriel Gambetta 的这篇文章里也称为和解），这个过程是这样的，客户端回滚到状态不一致的那一帧，然后再重新应用这一帧之后的所有 inputs，此为最新的预测状态，之后再通过插值平滑的过度到此状态。  


#### 延迟补偿 (lag compensation)

这个是针对命中判断来说的，比如说 fps 类型的游戏，由于网络延迟以及一些优化手段，导致你在画面上看到的景象实际上是几帧之前发生的事情，这时候你进行射击，此刻出现在你画面中的玩家在服务器处可能已经跑远了，当你的射击指令到达服务器时，是不会被判断命中的，这时候服务器需要把画面回滚（rewind）到你射击时候播放的那一帧，然后再进行判断。用这张经典的图片来展示一下：  

![valve-Lag_compensation](https://blog.antsmallant.top/media/blog/2023-06-27-game-networking/valve-Lag_compensation.jpg)  
<center>图2：延迟补偿[15]</center>

关于延迟补偿，在《网络多人游戏架构与编程》[22]中有具体的实现指导:  
>* 远程玩家使用客户端插值，而不是航位推测。  
>* 使用本地客户端移动预测和移动重放。 
>* 发送给服务器的每个移动数据包中保存客户端视角。客户端应该在每个发送的数据包中记录客户端当前插值的两个帧的ID，以及插值进度百分比。这给服务器提供了客户端当时所感知世界的精确指示。 
>* 在服务器端，存储每个相关对象最近几帧的位置。  


#### Dead Reckning

缩写为 DR，中文叫导航预测算法，这篇文章[12]对 DR 下了一个定义：  
>Using a predetermined set of algorithms to extrapolate entity behavior, you can hide some of the effects that latency has on fast-action games.   

翻译过来就是使用一系列算法去 extrapolate，是在特定场景下进行 extrapolate，是对 extrapolate 的一种特定应用。extrapolate 就是外插值，在游戏中就是根据已知的离散点，去推测出未来的点，也就是推算出未来的路径。与外插值相对的是内插值(interpolation), 内插在游戏中的应用就是根据已经的离散的点，在此范围内拟合出一条曲线，即移动路径。    

DR 实际上就是对于本玩家之外的其他物体进行预测的一种手段，本玩家直接使用预测回滚的方式进行预测了。  

#### 区域裁剪

比如通过 AOI 算法，裁剪需要下发给客户端的消息包。  

AOI 是 area of interest 的缩写，它代表的是游戏单位的视野，可以大到整个场景，也可以小到相当于屏幕大小的场景范围，主要目的就是减少需要广播的消息量，游戏单位只需要接收它当前视野范围的其他单位的变化信息。当然，这只是一种尽力而为的优化措施，当同屏有大量单位的时候，这个算法是起不了作用的，此时需要考虑其他的优化手段来减少消息广播量。     

常见的 aoi 算法有九宫格和十字链表，这两个都对应的是 2D 地图，3D 地图的话相对应的就是二十七宫格和三轴十字链表了，但其实原理类似，没有多大不同。     

九宫格的思路大概是这样:   
* 把场景地图分成 n 个小方格，每个小方格大约四分之一屏幕大小，九个格子刚好是比一个屏幕大一些
* 进入场景时，计算所处的格子，将自己加入该格子，给周围9个格子的单位广播 enter 消息
* 离开场景时，将自己从格子删除，给周围9个格子的单位广播 leave 消息
* 属性变化时，给周围9个格子的单位广播 change 消息
* 移动时
    * 如果没跨越，则只给周围9个格子的单位广播 move 消息；
    * 如果跨越了，则需要：1、给旧9格与新9格的差集广播 leave 消息；2、给新9格与旧9格的差集广播 enter 消息；3、给旧9格与新9格的交集广播 move 消息；

![aoi-九宫格](https://blog.antsmallant.top/media/blog/2023-06-27-game-networking/aoi-9-grid.webp)
<center>图3：aoi-九宫格算法[18] </center>

九宫格相当于在场景中创建了一个全局变量来记录每个格子中的单位，这样一来每个单位就不需要自己维护一个观察者列表或被观察者列表了。九宫格实现起来很简单，但是如果想要做到不同单位可变视野，就会很费劲了，需要很多额外的遍历，这时候可以考虑使用十字链表法。  

十字链表法的思路大概是这样：   
* X、Y 轴各维护一个按坐标值大小排列的有序链表；
* 进入场景时，游戏单位根据自己的坐标值，分别遍历X轴、Y轴，找到自己的位置把自己插入进去链表中；
* 每个单位在同一条轴上也插入两个哨兵节点，这两个哨兵节点到自己的距离刚好是视野半径；
* 单位移动的时候，哨兵节点也跟着移动，保持相对距离不变；
* 每个游戏单位维护一个观察者集合，被观察者集合，当有其他单位越过哨兵节点或自己的时候，就根据距离判断是加入观察者集合，还是移出观察者集合；

十字链表法看起来挺巧妙的，利用了两个哨兵节点来感应周围的变化，并且由于两个哨兵间的距离可以随自己的需要进行调整，所以可以很方便的做成可变视野的。  

源码方面，kbengine 的 aoi 实现是三轴十字链表，可以参考一下。   

AOI 算法还可以参考以下两篇文章，写得挺好的：   
* [游戏服务器AOI的实现](https://www.cnblogs.com/coding-my-life/p/14256640.html)
* [深入探索AOI算法](https://zhuanlan.zhihu.com/p/201588990)

#### 障眼法-隐藏延迟的 trick

通过前摇之类的方式来隐藏网络延迟的做法，我这里统称为障眼法。下面举一些实战的例子

halo 2011 年的这个 GDC 分享，展示一种如何让扔手雷看起来更流畅的做法，这里面的关键是：在合适的地方隐藏网络延迟。  

尝试一，按下按键，等待服务器回应之后再播放扔的动画，这种体验非常差：
![](https://blog.antsmallant.top/media/blog/2023-06-27-game-networking/halo-grenade-throw-attempt-1.png)  
<center>halo-grenade-throw-attempt-1[21]</center>

尝试二，按下按键，播放扔的动画，同时发消息给服务器，客户端播放完动画不等服务器响应直接扔出手雷，这种做法虽然没有延迟，但是违背了服务器权威的原则：
![halo-grenade-throw-attempt-2](https://blog.antsmallant.top/media/blog/2023-06-27-game-networking/halo-grenade-throw-attempt-2.png)  
<center>halo-grenade-throw-attempt-2[21]</center>

尝试三，这也是 halo 的最终实现方案，按下按键立即播放扔的动画，同时发消息给服务器，等收到回包再实际扔出手雷：  
![halo-grenade-throw-attempt-3](https://blog.antsmallant.top/media/blog/2023-06-27-game-networking/halo-grenade-throw-attempt-3.png)  
<center>halo-grenade-throw-attempt-3[21]</center>


#### 具体实现

预测回滚跟延迟补偿，对于客户端管理数据的能力提出了很高的要求，客户端要能够记录最近n帧的快照（世界状态），然后在检测到自身与服务端数据有冲突时进行和解，所谓和解，即回滚到发生冲突的那一帧，先把状态修改为服务端的权威状态，然后再应用本地的预测 input。   

守望先锋团队抱着试一试的心态采用的 ECS 架构，恰好可以很好的 “管理快速增长的代码复杂性” [13]，并且由于数据与逻辑完全分离，对于做数据回滚特别的方便。云风也分析到 ECS 架构对于做预测回滚会有很大帮助 [17]。

有精细的实现，也有粗糙的实现，我在 github 上看过一份源码（ [https://github.com/tsymiar/TheLastBattle](https://github.com/tsymiar/TheLastBattle) ），这款 moba 游戏里面也实现了客户端“预测先行”，但它只是把 local player 的朝向修改了，并没有真正的先移动。代码在此：
[https://github.com/tsymiar/TheLastBattle/blob/main/Client/Assets/Scripts/GameEntity/Iselfplayer.cs](https://github.com/tsymiar/TheLastBattle/blob/main/Client/Assets/Scripts/GameEntity/Iselfplayer.cs):     

```cs
public override void OnExecuteEntityAdMove()
{
    base.OnExecuteEntityAdMove();
    Quaternion DestQuaternion = Quaternion.LookRotation(EntityFSMDirection);
    Quaternion sMidQuater = Quaternion.Lerp(RealEntity.GetTransform().rotation, DestQuaternion, 10 * Time.deltaTime);
    RealEntity.GetTransform().rotation = sMidQuater;
    this.RealEntity.PlayerRunAnimation();
}
```  

这做法算是一种很经济的实现了 ：），应用范围也很广，比如在释放技能的时候，经常会搞一段很长时间的前摇，比如说 100 毫秒，等前摇完的时候，服务器回包也基本上收到了（当然，这里也要求服务端以比较高的帧率运行，否则 100 毫秒还不足以覆盖 RTT + 服务器帧时长），然后就可以播放真正的技能特效了。    


---

## 通用的优化手段

### udp 代替 tcp

用 udp 替代 tcp，是一种很有效的优化。可以使用可靠 udp (reliable udp) 比如 kcp，也可以使用带冗余信息的不可靠 udp。也可以把二者结合起来，比如这样的一个方案：kcp+fec。  


### 逻辑和显示的分离

这个主要是为了做插值使得视觉平滑，减少抖动感。客户端在实现上区分了“逻辑帧”与“显示帧”，比如玩家的位置会有个逻辑上的位置 position，会有个显示上的位置 view_position，显示帧 tick 的时候，通过插值算法，将 view_position 插值到 position，比如这样：  

```cs
player.view_pos = Vector3.Lerp(player.view_pos, player.pos, 0.5f);
player.view_rot = Quaternion.Slerp(player.view_rot, player.rot, 0.5f);
```

---

## 常见游戏的网络同步方案

### MMO 游戏的网络同步

一般的 MMO 对于网络同步的要求不高，最重要的是做好 AOI，因为场景里面需要同步的单位可能会特别多。  

### SLG 游戏的网络同步

SLG 这类游戏，对于网络同步的要求可以说是很低的，基本的 rpc 调用就够了，不需要复杂的网络同步。  

### MOBA 游戏的网络同步

MOBA 对于动作的同步的要求比较高，对于延迟也是相对敏感一些的。像王者荣耀使用的是帧同步，可以实现很精细的打击效果，更多的是使用带各种优化的状态同步，比如客户端先行，预测回滚、延迟补偿等。  

也有实现上很粗糙的，比如这款游戏 ( [https://github.com/tsymiar/TheLastBattle](https://github.com/tsymiar/TheLastBattle) )，它的客户端先行只是先简单的改变移动朝向，并不会真正的先移动。  

### FPS 游戏的网络同步

可以用帧同步，但没必要，用守望先锋的那种带各种优化的状态同步就可以达到很好的效果了。  

---

## 一些问题的探讨

### 王者荣耀使用帧同步明智吗？

这个很难评，成王败寇，它成功了，它就是明智的。但是帧同步带来的心智负担还是很重的，他们的分享里面也提到他们花了很大的功夫去解决不一致问题。   

个人更喜欢守望先锋的做法，虽然可能开发量更大，但至少没有埋下不一致这种大深坑。  

### 服务端预测

在 FPS 游戏中，会采用服务端预测的技术，服务端在没有收到输入的情况下，依据客户端当前的状态作出预测。那么问题来了，与客户端预测相比，它有何不同？  

客户端预测是当状态冲突时，以服务端为权威。那服务端预测，也是一样的原则，要以服务端为权威，所以即使服务端预测的是错的，客户端也要以服务端为权威，修正客户端的状态。  

比如 《无畏契约》的这个分享里介绍的 [19]：  
>What do we do when the server and client get out of sync?
>
>When one client disagrees, our top priority is to minimize the impact to the other nine players. The server commits its prediction as truth, and that client is told to adjust their simulation state back to match the server. This usually means instantly adjusting the positions or state of mispredicted characters back to where they should be. These corrections are rare, small in magnitude, and only seen by the player who encountered the underlying network issues & misprediction. The other nine players continue to see smooth movement.  

<br/>

这种做法结果就是牺牲网络卡的那个玩家，让其他网络正常玩家的体验保持丝滑，而网络卡的玩家体验到拉扯感。不过上文也说了，这种情况并不多见。  


### fps 中 Peeker’s advantage

资料主要参考自天美工作室的分享[20]，以及拳头游戏的这个文章[19]。  

玩家在转角处来回晃悠可以获得先手优势，这种优势是由于网络延迟造成的。静止不动的玩家，它在世界上的位置是相对不变的，而移动的玩家要被静止的玩家看到，至少要经历 RTT + 服务器缓存延迟 + 客户端缓存延迟，所以留给静止玩家的反应时间就要少得多。下面这张图很好的说明了问题。  

![valorant peeker's advantage](https://blog.antsmallant.top/media/blog/2023-06-27-game-networking/valorant-peeker-advantage.png)  
<center>图4：valorant peeker's advantage[19]</center>



### 客户端先行如何处理服务端同步过来的 “过时状态”

前面已经提到了，通过记录历史帧状态来做预测回滚，但是好多 MMO 在设计之初就没有使用这一套机制来做。所以现在就有一个问题，在一个老项目中，如何做到客户端先行。  

在前面也提到了，即使在 Moba 游戏中，也有使用障眼法来对付过去的，并没有真正的客户端先行，而是客户端先旋转方向。  

---

## todo

* mmo 关于同步粒度过于粗糙的问题，同步线段的问题
* mmo 中的预测回滚问题
* mmo 应该采用一个怎么样的相对适中的状态同步策略？
* 补充关于 buffering 的小节
* 处理过时状态的问题


---

## 推荐阅读

韦易笑对于网络同步有深入研究，是比较权威的：    
* [关于 “帧同步” 说法的历史由来](https://zhuanlan.zhihu.com/p/165293116)
* [再谈网游同步技术](https://www.skywind.me/blog/archives/1343)
* [服务端十二小时（百度网盘提取码:2j9b）](https://pan.baidu.com/s/1oBvmdQgsUWKrmU8g9o3u5Q)    
* [帧锁定同步算法](http://www.skywind.me/blog/archives/131)
* [帧同步游戏中使用 Run-Ahead 隐藏输入延迟](https://www.skywind.me/blog/archives/2746)
* [影子跟随算法（2007年老文一篇）](https://www.skywind.me/blog/archives/1145)


Jerish 的这几篇文章对于网络同步的发展史考察得很深入，值得仔细阅读：     
* [细谈网络同步在游戏历史中的发展变化（上）](https://zhuanlan.zhihu.com/p/130702310)
* [细谈网络同步在游戏历史中的发展变化（中）](https://zhuanlan.zhihu.com/p/164686867)
* [细谈网络同步在游戏历史中的发展变化（下）](https://zhuanlan.zhihu.com/p/336869551)   


Glen Fielder 是网络同步的世界级专家，在 gdc 上做过多次技术分享，这几篇文章写得很不错：   
* [Introduction to Networked Physics](https://gafferongames.com/post/introduction_to_networked_physics/)
* [What Every Programmer Needs To Know About Game Networking](https://gafferongames.com/post/what_every_programmer_needs_to_know_about_game_networking/)
* [Deterministic Lockstep](https://gafferongames.com/post/deterministic_lockstep/)
* [Snapshot Interpolation](https://gafferongames.com/post/snapshot_interpolation/)
* [Snapshot Compression](https://gafferongames.com/post/snapshot_compression/)
* [State Synchronization](https://gafferongames.com/post/state_synchronization/)


Gabriel Gambetta 这几篇文章关于状态同步相关优化手段的文章写得很通俗易懂，甚至还在文章里内嵌里一个 js 写的 demo：  
* [Fast-Paced Multiplayer (Part I): Client-Server Game Architecture](https://www.gabrielgambetta.com/client-server-game-architecture.html)
* [Fast-Paced Multiplayer (Part II): Client-Side Prediction and Server Reconciliation](https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html)
* [Fast-Paced Multiplayer (Part III): Entity Interpolation](https://www.gabrielgambetta.com/entity-interpolation.html)
* [Fast-Paced Multiplayer (Part IV): Lag Compensation](https://www.gabrielgambetta.com/lag-compensation.html)
* [Fast-Paced Multiplayer: Sample Code and Live Demo](https://www.gabrielgambetta.com/client-side-prediction-live-demo.html)

<br/>

---

## 总结

* 纸上谈兵容易，真做项目很难，会遇到各种各样的细节问题。自己多练手真正写写 demo 才更清楚其中的细节。  

---

## 参考

[1] Glenn Fiedler. Deterministic Lockstep. Available at https://gafferongames.com/post/deterministic_lockstep/, 2014-11.    

[2] Glenn Fiedler. Snapshot Interpolation. Available at https://gafferongames.com/post/snapshot_interpolation/, 2014-11.  

[3] Glenn Fiedler. State Synchronization. Available at https://gafferongames.com/post/state_synchronization/, 2015-1.   

[4] 韦易笑. 关于 “帧同步”说法的历史由来. Available at https://zhuanlan.zhihu.com/p/165293116, 2020-08.   

[5] 韦易笑. 帧锁定同步算法. Available at https://www.skywind.me/blog/archives/131, 2007-2.    

[6] Glenn Fiedler. Snapshot Compression. Available at https://gafferongames.com/post/snapshot_compression/, 2015-1.    

[7] Philip Orwig. Replay Technology in 'Overwatch': Kill Cam, Gameplay, and Highlights. Available at https://gdcvault.com/play/1024053/Replay-Technology-in-Overwatch-Kill, 2017.   

[8] kevinan. 《守望先锋》回放技术-阵亡镜头、全场最佳和亮眼表现. Available at https://www.sohu.com/a/162289484_483399, 2017-8.   

[9] 李清. Cocos 技术派｜实时竞技小游戏技术实现分享. Available at https://indienova.com/indie-game-development/real-time-mini-game-explained/, 2019-9.   

[10] 烟雨迷离半世殇. 基于行为树的MOBA技能系统：基于状态帧的战斗，技能编辑器与录像回放系统设计. Available at https://www.lfzxb.top/nkgmoba-framestepstate-architecture-battle-design/, 2021-11.   

[11] 邓君. 王者技术修炼之路. Available at https://youxiputao.com/articles/11842, 2017-5.   

[12] Jesse Aronson. Dead Reckoning: Latency Hiding for Networked Games. Available at https://www.gamedeveloper.com/programming/dead-reckoning-latency-hiding-for-networked-games#close-modal, 1997-9.        

[13] kevinan. 暴雪Tim Ford：《守望先锋》架构设计与网络同步. Available at https://www.sohu.com/a/148848770_466876, 2017-6.        

[14] 韦易笑. 帧同步游戏中使用 Run-Ahead 隐藏输入延迟. Available at https://www.skywind.me/blog/archives/2746, 2023-10.        

[15] Valve. Source Multiplayer Networking. Available at https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking.    

[16] Gabriel Gambetta. Fast-Paced Multiplayer (Part II): Client-Side Prediction and Server Reconciliation. Available at https://www.gabrielgambetta.com/client-side-prediction-server-reconciliation.html.      

[17] 云风. 浅谈《守望先锋》中的 ECS 构架. Available at https://blog.codingnow.com/2017/06/overwatch_ecs.html, 2017-6-26.        

[18] co lin. 深入探索AOI算法. Available at https://zhuanlan.zhihu.com/p/201588990, 2020-8-28.        

[19] RiotGames. PEEKING INTO VALORANT'S NETCODE. Available at https://technology.riotgames.com/news/peeking-valorants-netcode, 2020-7-28.        

[20] 腾讯天美工作室群. FPS游戏中，在玩家的延时都不一样的情况下是如何做到游戏的同步性的. Available at https://www.zhihu.com/question/29076648/answer/1946885829, 2021-6-18.       

[21] David Aldridge. I Shot You First: Networking the Gameplay of Halo: Reach. Available at https://www.youtube.com/watch?v=h47zZrqjgLc, 2011.           

[22] [美]Joshua Glazer, Sanjay Madhav. 网络多人游戏架构与编程. 王晓慧, 张国鑫. 北京: 人民邮电出版社, 2017-10(1): 244-245.           