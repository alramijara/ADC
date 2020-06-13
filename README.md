# Ejercicio del elevador Elevador
En este ejercicio de va a hacer uso de los perifericos de la tarjeta STM32L476RG para crear un elevador sencillo. Este elevador va a funcionar usando el botón y el led que estan en la tarjeta, lo que se busca es crear un programa que lleve un ascensor de un piso a otro, el piso estara determinado por el número de veces que se presione el botón de manera consecutiva y la posición del ascensor estara representada por los pines 1 a 3 del puerto A (sindo cada bit un piso, cuando el ascensor NO este en un piso el bit estara en cero), el led funcionara como una señal que indica cuando el elevador esta moviendose (apagado) y cuando ha llegado al piso destino (encendido).
Para esta aplicación se usara la interrupción externa y el timer del microcontrolador.
## Interrupción externa
Las interrupciones externas tal como su nombre lo indica son interrupciones que se dan en la ejecución del código como respuesta a una señal externa al microcontrolador, esto permite llevar a cabo un procedimiento especifico en cualquier momento que se detecte esta señal. Para el ejercicio la señal de interrupción provendrá del boton de la tarjeta, que al ser presionado aumentara un contador que se va a usar para indicar el piso al que se quiere llegar. Esto significa que se debe escoger el pin 13 del puerto C como la entrada de la señal y crear un manejador que aumente el contador.
### Registros necesarios
#### SYSCFG_EXTICR4
External interrupt configuration register 4, con este registro se escoge la fuente(entrada) que usará para la interrupción. Existen CR1, CR2, CR3 y CR4, cada unocorresponde a un grupo de pines distintos y tienen offsets diferentes. El registroEXTICR4 corresponde a los pines: 12, 13, 14 y 15 de todos los puertos. Los bits seagrupan de a 4 y dependiendo del valor del grupo de bits se elige el puerto alcual pertenece el pin de entrada.

![relojes](https://github.com/alramijara/ADC/master/syscfgexti.PNG)
![relojes](https://github.com/Valeria0212/Interrupcion-externa/blob/master/Imagenes/relojes.jpg)
