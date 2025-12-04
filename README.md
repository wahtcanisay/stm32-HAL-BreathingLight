# stm32呼吸灯实验
使用定时器输出不同占空比的PWM，动态改变LED的亮度实现呼吸灯  
## 硬件部分
![电路接线图](https://github.com/wahtcanisay/stm32-HAL-BreathingLight/blob/main/%E7%94%B5%E8%B7%AF%E6%8E%A5%E7%BA%BF%E5%9B%BE.png)
电路接线图  
想实现呼吸灯效果，需要将亮度按照 `亮度=0.5*sin(2*pi*t)+0.5`的方式设置，而亮度正比于占空比。  
![占空比大小](https://github.com/wahtcanisay/stm32-HAL-BreathingLight/blob/main/%E5%8D%A0%E7%A9%BA%E6%AF%94%E5%A4%A7%E5%B0%8F%E5%8E%9F%E7%90%86.png)
占空比大小原理图
## 软件部分
### cubemx部分  
首先在`SYS`中选择`Serial Wire`   
选中`TIM1`,`Clock Source`选择Internal Clock，即来自内部时钟源RCC  
PSC选择7，上计数，ARR设置为999，RCR设为0，使能ARR预加载。  
`Channel 1`选择`PWM Generation CH1 CH1N`，输出PWM波形和互补的PWM波形（PA8被分配为正常输出，PA7被分配为互补输出）  
参数设置中，选用`PWM mode 1`(CNT<CCRx为高电压)，`pulse`选为0(CCR初始值设为0)  
`Output compare preload`设为`Enable`使能CCR预加载（只有CNTupdate事件发生后才会重装CCR值） 
`CH Polarity`和`CHN Polarity`均选择High，表示正极性。  
### keil5部分
**新接口:**
```c
HAL_StatuTypeDef HAL_TIM_PWM_Start(TIM_HandleTypeDef *htim， uint32_t Channel)

HAL_StatuTypeDef HAL_TIM_PWMN_Start(TIM_HandleTypeDef *htim， uint32_t Channel)
```
作用：启动PWM正常输出和互补输出  (分别闭合两个开关启动定时器和使能正常或者互补输出)  

```c
__HAL_TIM_SET_PRESCALER(__HANDLE__, __VAL__)
__HAL_TIM_GET_PRESCALER(__HANDLE__)

__HAL_TIM_SET_COUNTER(__HANDLE__, __VAL__)
__HAL_TIM_GET_COUNTER(__HANDLE__)

__HAL_TIM_SET_AUTORELOAD(__HANDLE__, __VAL__)
__HAL_TIM_GET_AUTORELOAD(__HANDLE__)

__HAL_TIM_SET_COMPARE(__HANDLE__, __CHANNEL__,  __VAL__)
__HAL_TIM_GET_COMPARE(__HANDLE__, __CHANNEL__)
```
作用：分别写入和读取PSC/CNT/ARR/CCR的值  

```c
#include "math.h"
int main(){

    HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);//启动TIM1_CH1的正常输出
    HAL_TIM_PWMN_Start(&htim1, TIM_CHANNEL_1);//启动TIM1_CH1的互补输出

    while(1){

        float t = HAL_GetTick() * 0.001;//获取当前时间(ms)并将其转化为s
        float duty = 0.5 * sin(2 * pi * t) + 0.5; //计算想要实现的占空比
        uint16_t arr = __HAL_TIM_GET_AUTORELOAD(%htim1); //获取ARR寄存器的值
        uint16_t ccr = duty * (arr + 1);//计算出arr此时对应的值
        __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, ccr);//计算结果写入ccr

    }
}
```