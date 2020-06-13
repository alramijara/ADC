# Ejercicio del elevador Elevador
En este ejercicio de va a hacer uso de los perifericos de la tarjeta STM32L476RG para crear un elevador sencillo. Este elevador va a funcionar usando el botón y el led que estan en la tarjeta, lo que se busca es crear un programa que lleve un ascensor de un piso a otro, el piso estara determinado por el número de veces que se presione el botón de manera consecutiva y la posición del ascensor estara representada por los pines 1 a 3 del puerto A (sindo cada bit un piso, cuando el ascensor NO este en un piso el bit estara en cero), el led funcionara como una señal que indica cuando el elevador esta moviendose (apagado) y cuando ha llegado al piso destino (encendido).
Para esta aplicación se usara la interrupción externa y el timer del microcontrolador.
## Interrupción externa
Las interrupciones externas tal como su nombre lo indica son interrupciones que se dan en la ejecución del código como respuesta a una señal externa al microcontrolador, esto permite llevar a cabo un procedimiento especifico en cualquier momento que se detecte esta señal. Para el ejercicio la señal de interrupción provendrá del boton de la tarjeta, que al ser presionado aumentara un contador que se va a usar para indicar el piso al que se quiere llegar. Esto significa que se debe escoger el pin 13 del puerto C como la entrada de la señal y crear un manejador que aumente el contador.
### Registros necesarios
#### RCC_APB2ENR
Este registro se usa para activar el reloj de periféricos distintos a los puertos.En este ejercicio se hará uso de SYSCFG por lo cual se debe activar el reloj para este con el bit SYSCFGEN.
![](https://github.com/alramijara/ADC/blob/master/syscfgen.jpg)
```
      // habilitar el reloj SYSCFG 
    	RCC->APB2ENR |= 0x1;
```
#### SYSCFG_EXTICR4
External interrupt configuration register 4, con este registro se escoge la fuente(entrada) que usará para la interrupción. Existen CR1, CR2, CR3 y CR4, cada unocorresponde a un grupo de pines distintos y tienen offsets diferentes. El registroEXTICR4 corresponde a los pines: 12, 13, 14 y 15 de todos los puertos. Los bits seagrupan de a 4 y dependiendo del valor del grupo de bits se elige el puerto alcual pertenece el pin de entrada.

![](https://github.com/alramijara/ADC/blob/master/syscfgexti.jpg)
```
    	// Escribir 0b0010 en EXTI13 para escoger el pin13 del puerto C
    	SYSCFG->EXTICR[3] |= 0x20; // 
```
### EXTI_IMR1
Interrup mask register, este registro determina si se hará enmascaramiento a la solicitud de interrupción proveniente de un pin especifico, el pin depende de la ubicación del bit en el registro (0-15), por ejemplo para enmascarar la solicitud de interrupción del pin 13 se escribe un cero en el bit 13.

![](https://github.com/alramijara/ADC/blob/master/exti_imr.jpg)
```
    	EXTI->IMR1 |= 0x2000;    // Enmascarar IM13.
```

###   EXTI_RTSR1
Rising trigger selection register 1. La solicitud de interrupción se da cuando enuna entrada se detecta un cambio, sin embargo este cambio puede ser un flanco de subida o un flanco de bajada, por lo tanto se hace necesario escoger si uno o ambos de ellos indican una solicitud de interrupción. Con este registro se escoge que pines hacen una solicitud de interrupción cuando se identifica un flanco desubida.

![](https://github.com/alramijara/ADC/blob/master/exti_rtsr1.jpg)


void TIM2_IRQHandler(void)
{

    // clear interrupt status
    if (TIM2->DIER & 0x01) {
        if (TIM2->SR & 0x01) {
            TIM2->SR &= ~(1U << 0);
        }
    }

    	if ((pos<piso) & (con==1)){
    		pos+=1;
    		con=1;
    	}
    	else{
    		if((pos>piso)& (con==1)){
    			pos-=1;
    			con=1;
    		}
    		else{
    			con=0;
    			GPIOA->ODR &=0x0;
    			GPIOA->ODR |=(1<<(pos-1));
    			piso=0;
    		}

    	}


}
void EXTI15_10_IRQHandler(void)
{
	//Check if the interrupt came from exti13
	if(EXTI->PR1 & (1 <<13)) {
		piso += 1;
		con=1;
		if (piso>3){
			piso=0;

		}
		// Clear pending bit
		EXTI->PR1 = 0x00002000;
	}
}
int main(void)
{
	RCC->AHB2ENR |= 0x00000005;
	// Enable GPIOA and GPIOC Peripheral Clock (bit 0 and 2 in AHB2ENR register)
	// Make GPIOA Pin5 as output pin (bits 1:0 in MODER register)
	GPIOA->MODER &= 0xABFFFFFF;		// Clear bits 11, 10 for P5
	GPIOA->MODER &= 0xFFFFF755;		// Write 01 to bits 11, 10 for P5
	GPIOA->ODR &=0x0000;
	// Make GPIOD Pin13 as input pin (bits 27:26 in MODER register)
	GPIOC->MODER &= 0xFFFFFFFF;		// Clear bits 27, 26 for P13
	GPIOC->MODER &= 0xF3FFFFFF;		// Write 00 to bits 27, 26 for P13
    // enable TIM2 clock (bit0)
    RCC->APB1ENR1 |= (1<<0);


    	// Writing a 0b0010 to pin13 location ties PC13 to EXT4
    	SYSCFG->EXTICR[3] |= 0x20; // Write 0002 to map PC13 to EXTI4
    	// Choose either rising edge trigger (RTSR1) or falling edge trigger (FTSR1)
    	EXTI->RTSR1 |= 0x2000;   // Enable rising edge trigger on EXTI4
    	// Mask the used external interrupt numbers.
    	EXTI->IMR1 |= 0x2000;    // Mask EXTI4
    	// Set Priority for each interrupt request
    	NVIC->IP[EXTI15_10_IRQn] = 0x10; // Priority level 1
    	// enable EXT0 IRQ from NVIC
    	NVIC_EnableIRQ(EXTI15_10_IRQn);

    TIM2->PSC = 8399;

    TIM2->ARR = 1000;

    // Update Interrupt Enable
    TIM2->DIER |= (1 << 0);

    NVIC_SetPriority(TIM2_IRQn, 2); // Priority level 2
    // enable TIM2 IRQ from NVIC
    NVIC_EnableIRQ(TIM2_IRQn);

    // Enable Timer 2 module (CEN, bit0)
    TIM2->CR1 |= (1 << 0);

	while(1)
	{

		if (con==0){
			GPIOA->ODR |=(1<<5);
		}
		else {
			GPIOA->ODR &=(0<<5);
		}


	}

	__asm__("NOP"); // Assembly inline can be used if needed
	return 0;
}
void delay(volatile uint32_t s)
{
    for(s; s>0; s--){
        // Do nothing
    }
}
