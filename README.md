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

```
/*
    Peripheral definitions (Peripheral base addresses typecasted to xxx_RegDef_t)
*/
#define EXTI	((RCC_RegDef_t*)EXTI_BASEADDR)
#define SYSCFG	((SYSCFG_RegDef_t*)SYSCFG_BASEADDR) //check if it is done

#define GPIO_BASEADDR_TO_CODE(x) 	(x==GPIOA) ? 0:\ //if its A, else B (C conditional operator)
					(x==GPIOB) ? 1:
					(x==GPIOC) ? 2:
					(x==GPIOD) ? 3:
					(x==GPIOE) ? 4:
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
	volative uint32_t MEMRMP; // Address offset: 0x00
	volative uint32_t PMC; // Address offset: 0x04
	volative uint32_t EXTICR[4]; // Address offset: 0x08~0x14
	//SYSCFG_EXTICR decides from which port the EXT will be used
	//by defaut it is considered port A
	//SYSCFG_EXTICR4 -> EXT13[3:0] -> used to configur PC13 -> 0b0010 PC[x] pin
	//SYSCFG_EXTICR4 = EXTICR[3]
	uint32_t RESERVED1[2]; // Address offset: 0x08~0x1C
	volative uint32_t CMPCR; // Address offset: 0x20
	uint32_t RESERVED2[2]; // Address offset: 0x28
	volative uint32_t CFGR; // Address offset: 0x2C
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

PAREI em 112
