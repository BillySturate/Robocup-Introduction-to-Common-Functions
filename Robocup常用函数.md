# 		Robocup3d常用函数简介

## WorldModel类

WorldModel类保存了场上许多的信息，是机器人利用场上信息的重要途径，它在机器人NaoBehavior中实例化了一个对象指针WorldModel。

在机器人策略中使用时，应使用worldmodel->方法名([参数列表])的形式。

```c++
// 返回当前机器人的编号
inline int getUNum() const {
  return uNum;
}
```

常用于判断自己是几号，可以在策略中设定类似“如果是几号球员，就执行什么动作”

例如在初始化beam函数中，根据unum来分配球员位置

```c++
// 如果是一号球员则它的位置是
if (worldModel -> getUNum() == 1)
{
  beamX = -15;
  beamY = 0;
  beamAngel = 0;
}
else if ... (其他情况)
```

~~~c++
// 返回比赛当前时间(距离开始的秒数)(返回的是一个浮点数)
inline double getGameTime() const {
  return gametime;
}
~~~

```c++
// 返回当前比赛的周期数(每一秒是20个周期，一个周期agent和server进行通讯,server进行仿真处理后将场上的信息返回发送给agent)
inline unsigned long getCycle() const {
  return cycle;
}
```

~~~c++
// 获取编号为i的对象的位置(以下两函数仅名称不同，效果是一样的)
inline VecPosition getOpponent(const int &i) const {
	return worldObjects[i].pos;
}
inline VecPosition getTeammate(const int &i) const {
	return worldObjects[i].pos;
}
~~~

如何利用这个函数来检索指定的球员的位置呢?

在头文件WorldObject.h中，定义了一个枚举worldObjType内容如下

~~~c++
enum WorldObjType {   // Types of objects
    WO_BALL,

    // Self and Teammates
    WO_TEAMMATE1,
    WO_TEAMMATE2,
    WO_TEAMMATE3,
    WO_TEAMMATE4,
    WO_TEAMMATE5,
    WO_TEAMMATE6,
    WO_TEAMMATE7,
    WO_TEAMMATE8,
    WO_TEAMMATE9,
    WO_TEAMMATE10,
    WO_TEAMMATE11,

    // Opponents
    WO_OPPONENT1,
    WO_OPPONENT2,
    WO_OPPONENT3,
    WO_OPPONENT4,
    WO_OPPONENT5,
    WO_OPPONENT6,
    WO_OPPONENT7,
    WO_OPPONENT8,
    WO_OPPONENT9,
    WO_OPPONENT10,
    WO_OPPONENT11,
  	
  
  	// 后面还有很多比如关节什么的枚举量
};

~~~

众所周知枚举类其实就是给整数起了个别名，例如此处第一个枚举WO_BALL

在代码中和0作用一样，此后的每一个枚举在前一个的基础上加1，所以检索球员的代码可以这样写：

~~~c++
for (int i = WO_OPPONENT; i<WO_OPPONENT + OPPONENT_NUM; i++){
  VecPosition pos = worldModel->getOpponent(i);
  // ...
}
~~~

~~~c++
// 获取编号为index的WorldObject的指针
inline WorldObject* getWorldObject(int index){
  return &worldObjects[index];
}
~~~



上文的getOpponent与此函数内容类似，都是在worldObjects数组中获取对应下标的对象

WorldObjects类描述了某物体(球，球员等)的一些基本属性，可以说，是WorldObject们构成了WorldModel。

关于WorldObjects类此处不展开介绍，其方法与属性都浅显易懂，有兴趣自行学习。

~~~c++
// 以下还有一些常用函数

// 获取此球员的位置
inline VecPosition getMyPosition() const {
  return myPosition;
};

// 获取球的位置
inline VecPosition getBall() const {
  return worldObjects[WO_BALL].pos;
};

// 获取当前比赛模式(比赛进行到不同阶段会有不同模式，比如任意球，门球...有兴趣可以查看源码自行观看)
inline VecPosition getPlayMode() const {
  return playMode;
}

// 获取机器人是否摔跤
inline bool isFallen() const {
  return fFallen;
}
~~~



## Naobehaviors类

Naobehaviors类中有许许多多机器人可以执行的高级动作，怎么使用只要关注他的返回值即可，返回值为SkillType的可以直接在selectskill()中return（或者在其他返回值为SkillType的函数中return，比如gototarget()调用了gototargetrelative(),gototargetrelative调用了getwalk()等），返回值为VecPosition的可以用表达式和判断条件中，Bool类型的判断情况的函数，多用于条件判断。下面是一些常用的：

~~~c++
void NaoBehavior::beam( double& beamX, double& beamY, double& beamAngle );
~~~

决定球员上场位置的函数，可以依次对不同的number给其beamX和beamY赋值，也可以都设为default

~~~c++
double trim(const double& value,const double& min, const double& max);
~~~

trim:修剪。返回值为经过判断修改的value，满足

1.return_value = min, value ∈ (-∞, min)

1.return_value = value, value ∈ [min, max]

1.return_value = min, value ∈ (max,+∞)





~~~c++
// Currently untuned. For example, there's no slow down...
SkillType NaoBehavior::goToTargetRelative(const VecPosition& targetLoc, const double& targetRot, const double speed, bool fAllowOver180Turn, WalkRequestBlock::ParamSet paramSet)
{
    double walkDirection, walkRotation, walkSpeed;

    walkDirection = targetLoc.getTheta();
    walkRotation = targetRot;
    walkSpeed = speed;

    walkSpeed = trim(walkSpeed, 0.1, 1);

    if (targetLoc.getMagnitude() == 0)
        walkSpeed = 0;

    return getWalk(paramSet, walkDirection, walkRotation, walkSpeed, fAllowOver180Turn);
}
~~~

也是一个走向目标的函数，在Naobehaviors.cc里面，一般调用形式为

return goToTargetRelative(relativeTarget, turnAngle);

turnAngle是一个角度，让机器人转动这个角度从而保证可以用maximum speed走向目标，可以为turnAngle赋值为0，就表示walk directly to the target

~~~c++
//Assumes target = z-0. Maybe needs further tuning
SkillType NaoBehavior::goToTarget(const VecPosition &target) {
    double distance, angle;
    getTargetDistanceAndAngle(target, distance, angle);

    const double distanceThreshold = 1;
    const double angleThreshold = getLimitingAngleForward() * .9;
    VecPosition relativeTarget = VecPosition(distance, angle, 0, POLAR);

    // Turn to the angle we want to walk in first, since we want to walk with
    // maximum forwards speeds if possible.
    /*if (abs(angle) > angleThreshold)
    {
    return goToTargetRelative(VecPosition(), angle);
    }*/

    // [patmac] Speed/quickness adjustment
    // For now just go full speed in the direction of the target and also turn
    // toward our heading.
    SIM::AngDeg turnAngle = angle;

    // If we are within distanceThreshold of the target, we walk directly to the target
    if (me.getDistanceTo(target) < distanceThreshold) {
        turnAngle = 0;
    }

    // Walk in the direction that we want.
    return goToTargetRelative(relativeTarget, turnAngle);
}
~~~

此函数在Naobehaviors.cc，表示走向目标位置。





~~~c++
// Determines whether a collision will occur while moving to a target, adjusting accordingly when necessary
VecPosition NaoBehavior::collisionAvoidance(bool avoidTeammate, bool avoidOpponent, bool avoidBall, double PROXIMITY_THRESH, double COLLISION_THRESH, VecPosition target, bool fKeepDistance)；
~~~

此函数为避免碰撞函数。

demoKickingCircle()中使用了这个函数，具体为

~~~c++
target = collisionAvoidance(true /*teammate*/, false/*opponent*/, true/*ball*/, 1/*proximity thresh*/, .5/*collision thresh*/, target, true/*keepDistance*/);
~~~

前三个参数分别对应三个避免的对象（队友，对手，球），true就是避免，false就是不避免，最后一个参数为要去的target,函数通过一些算法取调整并返回target.





~~~c++
SkillType kickBall(const int kickTypeToUse, const VecPosition &target);
~~~

踢球的函数。第一个参数为怎么踢，KICK_DRRIBLE带球，KICK_FORWARD大力踢，KICK_IK快速踢，第二个参数为目标位置。



除此之外也可以自己添加函数，在Naobehaviors这个类中添加他的声明（比如最下面的demoKickingCircle(),大约在230行附近），然后在naobehaviors.cc或strategy.cc中定义即可。







先写这么多吧，码字很累，等这阵子忙完别的事再继续吧😂

--------------------------------------分割线--------------------------------------------------------------