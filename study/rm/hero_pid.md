#英雄车云台PID调试总结
今天已经18号了，距离出车的deadline已经不远了，留给电控的时间不多了。

昨天和前天已经有好几波人轮流调了英雄车的云台，但都无功而返，眼看死期将至，决定自己还是亲自拼一把。

我想起了上一年比赛第一次接触PID的时候，连PID是啥都不知道，结果上手就是一个串级PID，六个参数直接玄学调参，调了一个月左右，最后发现反馈数据给错了！修正之后立马就调好了。

今天的情况其实也很类似，也算是反馈数据不准确吧。

今天我才知道串级PID的调试方法，之前我一直是六个参数同时变，然后波形也不看，就各种自闭，虽然我熬了很多次夜来调试，逃了好几节来调试，都无济于事，说白了就是没掌握方案，没有完全理解PID的含义。

串级PID先调内环的参数。我这里的串级PID分别是外环位置环和内环速度环，使用的是带输出上限和积分饱和上限的普通位置式PID函数。

首先只调速度环，我使用的是陀螺仪反馈的角速度信息，单位是`°/s`，输出的是电机的电流大小，也就是扭矩的大小。

但是自闭了一个早上之后一点进度都没有，调出来的yaw轴云台要么没有力，要么就是疯狂振荡，后来我使用J-Scope在线查看PID设定值和反馈值，以及电机的角度值的曲线才发现，陀螺仪的角速度反馈值存在两个问题。
1. 角速度数据反馈频率低，在J-Scope中显示的曲线呈很明显的阶梯状
2. 陀螺仪的角速度是根据加速度积分得到的，计算需要时间，所以反馈的数据在图像上显示存在相位差

上面两个特点就决定了陀螺仪的反馈数据不可用，因为角速度无法得到快速的反馈。速度环是属于内环的，内环是整个串级PID的基础，内环调不好，那整个系统就崩溃了。

在自闭了一会儿之后，我决定使用电机编码器的反馈的机械角度位置进行微分计算电机的转速。

在这里介绍一下电机的参数：
> 机械角度反馈范围为0~8191
反馈频率为1000Hz

从上面的信息可以算出，如果只采集相邻两次的编码器数据，那么对应的角速度大约为
$$\frac{1000\times 360}{8192}=43.94^\circ /s$$

这显然是不行的，所以我连续采集了24次编码器数据计算平均速度，这样子精度可以在上面的精度基础上再二四分之。
$$\frac{43.94}{24}=1.83^\circ /s$$

实现的代码如下：
```c
/**
* @brief CAN通信电机的反馈数据具体解析函数
* @param 电机数据结构体
* @retval None
*/
void CanDataEncoderProcess(struct CAN_Motor *motor)
{
  int temp_sum = 0;
  motor->last_fdbPosition = motor->fdbPosition;
	motor->fdbPosition = CanReceiveData[0]<<8|CanReceiveData[1];
	motor->fdbSpeed = CanReceiveData[2]<<8|CanReceiveData[3];
  motor->last_real_position = motor->real_position;
  
  /* 电机位置数据过零处理，避免出现位置突变的情况 */
  if(motor->fdbPosition - motor->last_fdbPosition > 4096)
  {
    motor->round --;
  }else if(motor -> fdbPosition - motor->last_fdbPosition < -4096)
  {
    motor->round ++;
  }
  
	motor->real_position = motor->fdbPosition + motor->round * 8192;
  
  motor->diff = motor->real_position - motor->last_real_position;
  motor->position_buf[motor->index] = motor->diff;
  motor->index ++;
  if(motor->index == 24)
  {
    motor->index = 0;
  }
  
  for(int i=0; i<24;i++)
  {
    temp_sum += motor->position_buf[i];
  }
  motor->velocity = temp_sum*1.83;        //默认单位是8192，转换为360，毫秒转化为秒，加上这里累加了最近的24个数据，所以算式为temp_sum*360*1000/8192/24 = 1.83
}
```
经过上述处理之后，获取到的角速度就与角度的变化曲线对应起来了，但是曲线上看还是有很多细微的锯齿，这就导致PID中微分基本无用，因为数据在不断地上升下降，微分的作用几乎为零。

使用上述的角速度反馈之后，我直接在速度环PID中只加入了一个P，整个系统就稳定了，再经过细微的调整，整个云台的yaw轴的阻尼就特别大，用手掰会感受到很多的阻力，甚至炮管在经受大力冲击后能立马减速停止，居然达到了类似位置环的效果，非常让人满意，整个yaw轴给人的感觉就像`非牛顿液体`一样。

然后位置环就更简单了，加一个P就行了。最后这个云台的yaw轴的串级PID只使用了两个P就稳定了，经过测试角度精度能达到`0.1度`左右，完全达到视觉识别瞄准所需要的精度。

经过这次PID的调试，我对PID的理解更加深刻了。

**每一层PID的计算就相当于求导，位置环PID输出的是速度，于是输出值作为内环速度环的输入值，速度环的输出是加速度，也就是力矩，所以速度环的输出直接输给电调。**

这下子整个串级PID的逻辑就基本捋清楚了，特此记录一下。