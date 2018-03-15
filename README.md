# 裸奔STM32 系统时钟 + NVICC

## 认识时钟树  
在配置系统时钟之前先观摩一下STM32时钟树。比起51的时钟配置，STM32的时钟配置看起来挺吓人的.来，关门放狗。。。看树。 
![image](./img/1.png)  
看图，怕了吧，怕个卵。我带你们砍树。我们比较关注的是系统时钟，当然使用M3也想要它效率最大嘛，你开个超跑走的拖拉机还慢有意思么。那我们直接奔着72M去。  
首先根据时钟树我规划了4条路线，调调大路通罗马嘛。   
![image](./img/2.png)    

画的有点丑，将就将就。( ╯□╰ )   
**路线1:**  
HSI直接进来作为系统时钟。
HSI是内部8M时钟，明显不够。PASS.   
注意：STM32上电默认走这条路。   
**路线2:**   
HSI 8M -> 2分频 4M -> PLLMULL 2-16倍频 max=64M   差点。PASS
**路线3:**    
HSE OSC直接进来，HSE OSE是外部晶振时钟，晶振范围4-16M，也不够。   PASS   
**路线4:**     
HSE OSC 外部晶振 4-16M进入 -> 是否/2 -> PLLMUL 2-16倍频   
要得到72M时钟，只有8*9=72；9可以从PLLMUL倍频得到，8可以从HSE OSE直接得到，或者16M / 2得到。嗯，路线规划完毕，老司机开车，准备出发咯。

**路线整理**    
别急，先确定一下路线和硬件,把其它时钟也配置一下。回头看看时钟树，我们大部分外设都不是直接使用系统时钟的，一般使用AHB时钟和APB1时钟，配置的时候
我们也顺便将它们也配置好。当然也是安最高速度来配置啦。AHB和系统时钟同步72M，APB1时钟因为最大为36M，我们就2分频配置为36M即可。

1.首先STM32上电使用的是HSI，必须切换回来使用HSE OSC   
2.然后进入PLLXPRE，不分屏   
3.接着经过PLLSR后进入PLLMUL倍频，最后通过SW选择PLLSCK作为系统时钟    
4.配置AHB时钟72M    
5.配置APB1时钟36M    

来来来，终于开车了：    
车型KEIL:   

	void SysClk_Init(void)
	{
		RCC->CR|=1<<16;					//开启HSE OSC外部晶振时钟 8M
		while((RCC->CR&(1<<17))==0);	//等待HSE晶振就绪
		RCC->CFGR &= ~(1 << 17);		//PLLXPRE HSE不分频  8M
		RCC->CFGR|=1<<16;				//PLLSRC选择HSE最为PLL时钟。		
		RCC->CFGR|=7<<18;				//PLLMUL 9倍频 系统时钟达到72M
		RCC->CFGR&=0xffffff0c;			//AHB不分频 72M
		RCC->CFGR &= 0xFFFFF8FF;		//清除APB1分频位
		RCC->CFGR|=  0x00000400;		//APB1 2分频 36M
		RCC->CFGR|=0x00000002;			//PLL作为系统时钟
		
		/*
		这里蹦出了一条奇怪的配置。别慌，先了解一下。
		根据FLASH编程手册，时钟频率在(48,72]MHz时需要配置两个等待周期。
		为什么要在这里配置呢？
		其实很简单，CPU处理速度快，FLASH操作速度最高24M,因此必须增加等待时钟来适应FLASH速度。
		上电配置到现在都是使用HSI内部8M晶振，因此不需要配置FLASH等待时钟，然而
		后面两条指令执行后，系统时钟将变为72M,因此现在还不改变FLASH就来不及了。
		*/
		FLASH->ACR|=0x32;			
		
		
		RCC->CR|=1<<24;					//使能PLL
		while((RCC->CR & (1<<25))==0);	//等待PLL就绪
		
			
	}