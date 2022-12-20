# target-STM32F401-drivers-GPIO-interrupt
This project uses STM32CubeIDE and it's a program created to practice my C habilities during the course 'Mastering Microcontroller and Embedded Driver Development' from FastBit Embedded Brain Academy. I am using a NUCLEO-F401RE board.


//EXT se referen a interrupções, conectadas nos registradores NVIC

GPIO pin interrupt configuration
1. configure pin as input
2. configure the edge trigger (RF, FT, RFT)
3. enable interrupt delivery from peripheral to the processor (on peripheral side)
4. identify the IRQ number on which the processor accepts the interrupt from that pin
5. configure the IRQ priority for the identified IRQ number (processor side)
6. enable interrupt reception on that IRQ number (processor side)
7. implement IRQ handler 


adc em xx.h

Since we are using SYSCFG and EXTI, we are going to add the base addresses for the APB2 peripherals (and AHB2 to be completed).
```
/*
    Base addresses of peripherals which are hanging on APB2 bus
*/
#define TIM1_BASEADDR 		(APB2PERIPH_BASE + 0x0000)
#define USART1_BASEADDR 	(APB2PERIPH_BASE + 0x1000)
#define USART6_BASEADDR 	(APB2PERIPH_BASE + 0x1400)
#define ADC1_BASEADDR 		(APB2PERIPH_BASE + 0x2000)
#define SDIO_BASEADDR 		(APB2PERIPH_BASE + 0x2C00)
#define SPI1_BASEADDR 		(APB2PERIPH_BASE + 0x3000)
#define SPI4_BASEADDR 		(APB2PERIPH_BASE + 0x3400)
#define SYSCFG_BASEADDR 	(APB2PERIPH_BASE + 0x3800)
#define EXTI_BASEADDR 		(APB2PERIPH_BASE + 0x3C00)
#define TIM9_BASEADDR 		(APB2PERIPH_BASE + 0x4000)
#define TIM10_BASEADDR 		(APB2PERIPH_BASE + 0x4400)
#define TIM11_BASEADDR 		(APB2PERIPH_BASE + 0x4800)

/*
    Base addresses of peripherals which are hanging on AHB2 bus
*/
#define USB_OTG_FS_BASEADDR (AHB2PERIPH_BASE + 0x0000)
```


```
/*
    Peripheral definitions (Peripheral base addresses typecasted to xxx_RegDef_t)
*/
#define EXTI		((EXTI_RegDef_t*)EXTI_BASEADDR)
#define SYSCFG		((SYSCFG_RegDef_t*)SYSCFG_BASEADDR) 

/*
    This macro returns a code (between 0 to 5) for given GPIO base address (x)
*/
 //if its A, else B (C conditional operator) - this comment in next line bugged my macro
#define GPIO_BASEADDR_TO_CODE(x)       ((x==GPIOA) ? 0:\ 
					(x==GPIOB) ? 1:\
					(x==GPIOC) ? 2:\
					(x==GPIOD) ? 3:\
					(x==GPIOE) ? 4:\
					(x==GPIOH) ? 5:0)
			
/*
    Peripheral register definition structure for EXTI
*/
typedef struct{
	volatile uint32_t IMR; // Address offset: 0x00
	volatile uint32_t EMR; // Address offset: 0x04
	volatile uint32_t RTSR; // Address offset: 0x08
	volatile uint32_t FTSR; // Address offset: 0x0C
	volatile uint32_t SWIER; // Address offset: 0x10
	volatile uint32_t PR; // Address offset: 0x14
}EXTI_RegDef_t;

typedef struct{
	volatile uint32_t MEMRMP; // Address offset: 0x00
	volatile uint32_t PMC; // Address offset: 0x04
	volatile uint32_t EXTICR[4]; // Address offset: 0x08~0x14
	//SYSCFG_EXTICR decides from which port the EXT will be used
	//by defaut it is considered port A
	//SYSCFG_EXTICR4 -> EXT13[3:0] -> used to configur PC13 -> 0b0010 PC[x] pin
	//SYSCFG_EXTICR4 = EXTICR[3]
	uint32_t RESERVED1[2]; // Address offset: 0x08~0x1C
	volatile uint32_t CMPCR; // Address offset: 0x20
	uint32_t RESERVED2[2]; // Address offset: 0x28
	volatile uint32_t CFGR; // Address offset: 0x2C
}SYSCFG_RegDef_t;

void GPIO_IRQConfig()
```




```
// put this inside, after the if(pGPIOHandle->GPIO_PinConfig.GPIO_PinMode <= GPIO_MODE_ANALOG)
else{ // >= 4
	if(pGPIOHandle->GPIO_PinConfig.GPIO_PinMode == GPIO_MODE_IT_FT){
		//1. configure the FTSR
		EXTI->FTSR |= (1 << pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber);
		// Clear the corresponding RTSR bit
		EXTI->RTSR &= ~(1 << pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber);

	}else if (pGPIOHandle->GPIO_PinConfig.GPIO_PinMode == GPIO_MODE_IT_RT){
		//1. configure the RTSR
		EXTI->RTSR |= (1 << pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber);
		// Clear the corresponding RTSR bit
		EXTI->FTSR &= ~(1 << pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber);

	}else if(pGPIOHandle->GPIO_PinConfig.GPIO_PinMode == GPIO_MODE_IT_RFT){
		//1. configure both FTSR and RTSR
		EXTI->FTSR |= (1 << pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber);
		EXTI->RTSR |= (1 << pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber);
	}

	//2. configure the GPIO port selection in SYSCFG_EXTICR
	uint8_t temp1 = pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber /4;
	uint8_t temp2 = pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber %4;
	uint8_t portcode = GPIO_BASEADDR_TO_CODE(pGPIOHandle->pGPIOx);
	SYSCFG_PCLK_EN();
	SYSCFG->EXTICR[temp1] = portcode << (temp2 *4);

	//3. enable the EXTI interrupt delivery using IMR
	EXTI->IMR |= (1 << pGPIOHandle->GPIO_PinConfig.GPIO_PinNumber);
} 
```

The information about the position of the IRQ is found in the 'Table 38: Vector table for STM32F401xB/CSTM32F401xD/E' of user manual (RM0368). Here we are creating some makros to help with the code. Later, it can be completed the makros for all on the table.

![image](https://user-images.githubusercontent.com/58916022/208267413-03ed1832-e0cb-4f10-9055-eabc06528235.png)

Copy in the MCU header file:

```
#define IRQ_NO_EXTI0 6
#define IRQ_NO_EXTI1 7
#define IRQ_NO_EXTI2 8
#define IRQ_NO_EXTI3 9
#define IRQ_NO_EXTI4 10
#define IRQ_NO_EXTI9_5 23
#define IRQ_NO_EXTI15_10 40 
```

Or, the complete macros:

```
// Generic IRQ macros 		Table 38. Vector table for STM32F401xB/CSTM32F401xD/E
#define IRQ_NO_WWDG 				0
#define IRQ_NO_EXTI16 				1
#define IRQ_NO_PVD 					1
#define IRQ_NO_EXTI21 				2
#define IRQ_NO_TAMP_STAMP			2
#define IRQ_NO_EXTI22 				3
#define IRQ_NO_RTC_WKUP				3
#define IRQ_NO_FLASH 				4
#define IRQ_NO_RCC	 				5
#define IRQ_NO_EXTI0 				6
#define IRQ_NO_EXTI1 				7
#define IRQ_NO_EXTI2 				8
#define IRQ_NO_EXTI3 				9
#define IRQ_NO_EXTI4 				10
#define IRQ_NO_DMA1_STREAM0			11
#define IRQ_NO_DMA1_STREAM1			12
#define IRQ_NO_DMA1_STREAM2			13
#define IRQ_NO_DMA1_STREAM3			14
#define IRQ_NO_DMA1_STREAM4			15
#define IRQ_NO_DMA1_STREAM5			16
#define IRQ_NO_DMA1_STREAM6			17
#define IRQ_NO_ADC					18
#define IRQ_NO_EXTI9_5 				23
#define IRQ_NO_TIM1_BRK_TIM9		24
#define IRQ_NO_TIM1_UP_TIM10		25
#define IRQ_NO_TIM1_TRG_COM_TIM11	26
#define IRQ_NO_TIM1_CC	 			27
#define IRQ_NO_TIM2	 				28
#define IRQ_NO_TIM3 				29
#define IRQ_NO_TIM4 				30
#define IRQ_NO_I2C1_EV 				31
#define IRQ_NO_I2C1_ER 				32
#define IRQ_NO_I2C2_EV 				33
#define IRQ_NO_I2C2_ER 				34
#define IRQ_NO_SPI1 				35
#define IRQ_NO_SPI2 				36
#define IRQ_NO_USART1				37
#define IRQ_NO_USART2	 			38
#define IRQ_NO_EXTI15_10 			40
#define IRQ_NO_EXTI17	 			41
#define IRQ_NO_RTC_ALARM 			41
#define IRQ_NO_EXTI18	 			42
#define IRQ_NO_OTG_FS_WKUP 			42
#define IRQ_NO_DMA1_STREAM7			47
#define IRQ_NO_SDIO		 			49
#define IRQ_NO_TIM5		 			50
#define IRQ_NO_SPI3		 			51
#define IRQ_NO_TIM5		 			50
#define IRQ_NO_DMA2_STREAM0			56
#define IRQ_NO_DMA2_STREAM1			57
#define IRQ_NO_DMA2_STREAM2			58
#define IRQ_NO_DMA2_STREAM3			59
#define IRQ_NO_DMA2_STREAM4			60
#define IRQ_NO_OTG_FS		 		67
#define IRQ_NO_DMA2_STREAM5			68
#define IRQ_NO_DMA2_STREAM6			69
#define IRQ_NO_DMA2_STREAM7			70
#define IRQ_NO_USART6	 			71
#define IRQ_NO_I2C3_EV	 			72
#define IRQ_NO_I2C3_ER		 		73
#define IRQ_NO_FPU		 			81
#define IRQ_NO_SPI4		 			84
```

Now using the [Cortex -M4 Devices Generic User Guide](https://developer.arm.com/documentation/dui0553/latest/), we are going to complete the GPIO_IRQConfig by the processor specification side.

Note: EXTI is MCU side, NVIC is processor side.

For enabling and disabling those registers, we need to check both registers:

![image](https://user-images.githubusercontent.com/58916022/208692704-f424f4c9-9a72-41a0-83da-f09e49a5e09f.png)

And we also will need to set its priority:

![image](https://user-images.githubusercontent.com/58916022/208693039-55459a79-f655-4759-b930-3c8938cc51b7.png)

Lets create some macros at the top of stm32f401xx.h file:
```
/********************* [START] Processor Specific Details ************/
/*
 *   ARM Cortex M4 Processor NVIC ISERx register addresses
 */
#define NVIC_ISER0		((volatile uint32_t*)0xE000E100)
#define NVIC_ISER1		((volatile uint32_t*)0xE000E104)
#define NVIC_ISER2		((volatile uint32_t*)0xE000E108)
/*
 *   ARM Cortex M4 Processor NVIC ICERx register addresses
 */
#define NVIC_ICER0		((volatile uint32_t*)0xE000E180)
#define NVIC_ICER1		((volatile uint32_t*)0xE000E184)
#define NVIC_ICER2		((volatile uint32_t*)0xE000E188)
/********************* [END] Processor Specific Details ************/
```

And in stm32f401xx_gpio_driver.c we can complete our functions. But note the we just enable or disable the IRQ number, taking out the Priority (uint8_t IRQPriority) f our already created function (same must happen in .h file).
We also chaged the name to  GPIO_IRQInterruptConfig and created a new one for priority.

```
void GPIO_IRQInterruptConfig(uint8_t IRQNumber, uint8_t EnOrDi){
	if (EnOrDi == ENABLE){
		if(IRQNumber <= 32){
			//program ISER0 register
			*NVIC_ISER0 |= (1 << IRQNumber);
		}else if (IRQNumber > 31 &&  IRQNumber < 64){ //32 to 63
			//program ISER1 register
			*NVIC_ISER1 |= (1 << IRQNumber % 32);
		}else if (IRQNumber >= 64 &&  IRQNumber < 96){ // 64 to 95 (it ends in 81 for our MCU)
			//program ISER2 register
			*NVIC_ISER2 |= (1 << IRQNumber % 64);
		}
	}else{
		if(IRQNumber <= 32){
			//program ICER0 register
			*NVIC_ICER0 |= (1 << IRQNumber);
		}else if (IRQNumber > 31 &&  IRQNumber < 64){
			//program ICER1 register
			*NVIC_ICER1 |= (1 << IRQNumber % 32);
		}else if (IRQNumber >= 64 &&  IRQNumber < 96){
			//program ICER2 register
			*NVIC_ICER2 |= (1 << IRQNumber % 64);
		}
	}
}


```

For priority, the Interrupt Priority Registers (NVIC_IPR0-NVIC_IPR59) are divided by sections of 8 bits (IRQx_PRI), being 4 IRQ for register. 



