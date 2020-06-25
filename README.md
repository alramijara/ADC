# Ejercicio del Elevador
En este ejercicio de va a hacer uso de los perifericos de la tarjeta STM32L476RG para crear un elevador sencillo. Este elevador va a funcionar usando el botón y el led que estan en la tarjeta, lo que se busca es crear un programa que lleve un ascensor de un piso a otro, el piso estara determinado por el número de veces que se presione el botón de manera consecutiva y la posición del ascensor estara representada por los pines 1 a 3 del puerto A (sindo cada bit un piso, cuando el ascensor NO este en un piso el bit estara en cero), el led funcionara como una señal que indica cuando el elevador esta moviendose (apagado) y cuando ha llegado al piso destino (encendido).
Para esta aplicación se usara la interrupción externa y el timer del microcontrolador.
## Interrupción externa
Las interrupciones externas tal como su nombre lo indica son interrupciones que se dan en la ejecución del código como respuesta a una señal externa al microcontrolador, esto permite llevar a cabo un procedimiento especifico en cualquier momento que se detecte esta señal. Para el ejercicio la señal de interrupción provendrá del boton de la tarjeta, que al ser presionado aumentara un contador que se va a usar para indicar el piso al que se quiere llegar. Esto significa que se debe escoger el pin 13 del puerto C como la entrada de la señal y crear un manejador que aumente el contador.
### Inicialización
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
#### EXTI_IMR1
Interrup mask register, este registro determina si se hará enmascaramiento a la solicitud de interrupción proveniente de un pin especifico, el pin depende de la ubicación del bit en el registro (0-15), por ejemplo para enmascarar la solicitud de interrupción del pin 13 se escribe un cero en el bit 13.

![](https://github.com/alramijara/ADC/blob/master/exti_imr.jpg)
```
    	EXTI->IMR1 |= 0x2000;    // Enmascarar IM13.
```

####   EXTI_RTSR1
Rising trigger selection register 1. La solicitud de interrupción se da cuando enuna entrada se detecta un cambio, sin embargo este cambio puede ser un flanco de subida o un flanco de bajada, por lo tanto se hace necesario escoger si uno o ambos de ellos indican una solicitud de interrupción. Con este registro se escoge que pines hacen una solicitud de interrupción cuando se identifica un flanco desubida.

![](https://github.com/alramijara/ADC/blob/master/exti_rtsr1.jpg)
```
EXTI->RTSR1 |= 0x2000;   // Cuando detecta un flanco de subida en el pin 13 se ejecuta la interrupción
```
####   NVIC->IP[EXTI15_10_IRQn] 
Establece la prioridad de una interrupción. 
![](https://github.com/alramijara/ADC/blob/master/nvic_ipr.jpg)
```
NVIC->IP[EXTI15_10_IRQn] = 0x10; // Prioridad nivel 1
```
#### NVIC_EnableIRQ(EXTI15_10_IRQn)
Habilita la interrupción externa, para que se ejecute una sección de código especifica cada vez que se detecte un flanco de subida en el pin 13.

```
NVIC_EnableIRQ(EXTI15_10_IRQn);
```
### Manejador (EXTI15_10_IRQHandler)
Cada vez que se presione el botón se ejecutara esta rutina, comienza verificando que la señal de interrupción proviene del pin 13 y no de otro, si es así aumenta en uno el contador "piso" para indicar a que piso se quiere llegar (si se oprime dos veces consecutivas el contador aumentará a dos y subira hasta el segundo piso y asi sucesivamente) y pone la variable con en 1 que funciona como un condicional para indicar que se ha hecho un nuevo llamado al elevador. Al finalizar se limpia la bandera de solicitud de interrupción.
```
void EXTI15_10_IRQHandler(void)
{
	//Check if the interrupt came from exti13
	if(EXTI->PR1 & (1 <<13)) {
		piso += 1;
		con=1;
		if (piso>3){
			piso=0;

		}
		
	EXTI->PR1 = 0x00002000;
	}
}
```
## TIMER2
El TIMER es un periferico que permite ejecutar una sección de código cada cierto perido de tiempo especificado por el programador (limitado por el reloj del sistema), para este ejercicio va a ejecutarse cada dos segundos aproximadamente los cuales representan el tiempo que le toma al ascensor subir un piso. El timer funciona como una interrupción, genera un segundo reloj derivado del reloj de microcontrolador despues de pasar por un peescalador, con cada ciclo de este nuevo reloj se aumenta en uno un contador y cuando llegue a un valor limite establecido el contador se reiniciará y ejecutara la interrupción del timer (llamada update interruption).

Es necesario entender como se genera esta base de tiempo a partir del TIMER, en primer lugar se usa un prescalador para generar un reloj de menor frecuencia que el del microcontrolador, cda vez que ocurre un ciclo de este nuevo reloj, se aumenta un contador interno en 1 hasta llegar a un valor limite establecido por el usuario y entonces el contador se reinicia al mismo tiempo que se hace una solicitud de interrupción (update interruption). Es en esta solicitud de interrupción que se aumenta un contador propio para indicar el tiempo de ejecución.

![](https://github.com/alramijara/ADC/blob/master/timer_example.JPG)

 Se explicaran las señales de cada fila:
 
- CK_PSC: Reloj interno del microcontrolador, su frecuencia depende de como este configurado y de la alimentación de la tarjeta.
     
- CNT_EN: Habilita el reloj CK_CNT, comienza el conteo n ciclos siguiente, siendo n el divisor del reloj (PSC[15:0]+1).
     
- CK_CNT: Reloj del contador, con una frecuencia igual a f_{CK\_PSC}/(PSC[15:0]+1), en cada ciclo se aumenta en 1 el contador del timer.
     
- Counter register: Registro que almacena el contador del TIMER, aumenta en 1 cada ciclo del CK_CNT, cuando alcanza el valor de ARR se reinicia al ciclo siguiente (para el caso ascendente) o si llega a 0 toma el valor de ARR en el ciclo siguiente (para el caso descendente).
    
- Counter overflow: Cuando el contador del TIMER se reinicia, se genera una señal de que ya se alcanzo el valor máximo.
     
- Update Event: Anteriormente se había habilitado la interrupción de actualización, este evento (señal) es el que genera esa interrupción para que se ejecute el código.
     
- Update Interrupt Flag: Es la bandera que indica la solicitud de interrupción de actualización, es limpiada usando el registro SR.

### Inicialización
#### RCC→APB1ENR1
Registro que habilita o deshabilita el reloj para ciertos periféricos, con este reg-istro se habilitara el reloj del TIMER 2 que se va a usar en el ejercicio.
![](https://github.com/alramijara/ADC/blob/master/apb1enr1.JPG)
```
RCC->APB1ENR1 |= (1<<0);

```
#### TIMx→PSC
TIMx prescaler, con este registro se establece el preescalador del TIMER x, el cual funciona como un divisor de frecuencia programable. 

![](https://github.com/alramijara/ADC/blob/master/psc.JPG)
```
TIM2->PSC = 8399; Prescalador 

```
####   TIMx→ARR
TIMx auto-reload register, almacena el valor hasta el cual va a contar el TIMER, funciona distinto si el contador trabaja de forma ascendente o descendente.

![](https://github.com/alramijara/ADC/blob/master/arr.JPG)
```
TIM2->ARR = 1000;

```
#### TIMx→DIER
TIMx DMA/interrupt enable register, habilita y deshabilita distintasinterrupciones del TIMER, entre las cuales se encuentra la interrupción update que habilitaremos para el proyecto.

![](https://github.com/alramijara/ADC/blob/master/dier.JPG)
```
TIM2->DIER |= (1 << 0);

```
#### TIMx→CR1
TIMx control register 1, este registro se usará para habilitar el contador de el TIMER.

![](https://github.com/alramijara/ADC/blob/master/cr1.JPG)
```
TIM2->DIER |= (1 << 0);

```
#### TIMx_SR
TIMx status register, contiene banderas de eventos del TIMER, para este ejercicio se hace uso de la interrupción de actualización y por lo tanto es la bandera que se deberá limpiar al ejecutar la interrupción para establecer como ejecutada una solicitud de interrupción. ESto se hace en el manejador del TIMER.

![](https://github.com/alramijara/ADC/blob/master/sr.JPG)

```
    if (TIM2->DIER & 0x01) {
        if (TIM2->SR & 0x01) {
            TIM2->SR &= ~(1U << 0);
        }
    }

```

####   NVIC_SetPriority(TIM2_IRQn,2) 
Establece la prioridad de una interrupción. 
![](https://github.com/alramijara/ADC/blob/master/nvic_ipr.jpg)
```
NVIC->IP[EXTI15_10_IRQn] = 0x10; // Prioridad nivel 1
```
#### NVIC_EnableIRQ(TIM2_IRQn)
Habilita el timer y su interrupción.
```
NVIC_EnableIRQ(TIM2_IRQn);
```
### Manejador
El manejador se va a encargar de mover un piso el ascensor cada dos segundos. Por cada interrupción solo se desplazara un piso, si el ascensor ya ha llegado al piso entonces no se hará nada durante esta interrupción. 

```
void TIM2_IRQHandler(void)
{
    if (TIM2->DIER & 0x01) {
        if (TIM2->SR & 0x01) {
            TIM2->SR &= ~(1U << 0);
        }
    }
     //Si el ascensor debe subir
    	if ((pos<piso) & (con==1)){
    		pos+=1;
    		con=1; //Bandera que indica si el ascensor esta en movimiento, 1=se mueve y 0=no se mueve
    	}
    	else{
	//Si el ascensor debe bajar
    		if((pos>piso)& (con==1)){
    			pos-=1;
    			con=1; 
    		}
		//Si se ha alcanzado el piso destino
    		else{
    			con=0;
    			GPIOA->ODR &=0x0;
    			GPIOA->ODR |=(1<<(pos-1)); // Carga en el puerto A la posicion del ascensor
    			piso=0; //Reestablece el contador de piso
    		}
    	}
}
```
## Tarea:
Inicializa el pin 5 del puerto A y el pin 13 de puerto C para usarlos para el ejercicio.


    

 
