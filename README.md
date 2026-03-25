# PROYECTO-2.-GRUA-ROBOTICA
Proyecto consiste en el diseño e implementación de una grúa robótica de sobremesa que cuenta con 2 grados de libertad (GDL), controlados mediante servomotores.
Diseño e Implementación de una Grúa Robótica  con ESP32

SEGUIMIENTO 2 - DIGITAL 2

Grua Robotica

Laura Ximena Uribe Zapata

Jailler Dany Gaviria

Daniela Dıaz Ossa


1. Introducción

El presente proyecto consiste en el diseño e implementación de una grúa robótica de sobremesa con dos grados de libertad (GDL), controlada mediante un microcontrolador ESP32. El sistema permite simular el funcionamiento básico de una grúa industrial, integrando control manual y modos automáticos mediante el uso de sensores analógicos y mecanismos de interrupción.

2. Objetivos
Objetivo General

Desarrollar una grúa robótica funcional controlada por un ESP32, capaz de operar en modo manual y automático.

Objetivos Específicos
Implementar control de posición mediante potenciómetros.
Utilizar conversión ADC para lectura de señales analógicas.
Generar señales PWM para el control de servomotores.
Diseñar un sistema basado en interrupciones para modos automáticos.
Incorporar indicadores visuales y sonoros para el estado del sistema.

3. Descripción del Sistema

El sistema está compuesto por una grúa robótica con dos grados de libertad, controlados mediante servomotores. Los movimientos se gestionan a través de potenciómetros, los cuales permiten una interacción intuitiva similar a la de un operador real.

Además, se incorporan dos pulsadores con interrupciones para activar modos automáticos:

Retorno a posición inicial
Ejecución de una secuencia predefinida

El sistema también incluye mecanismos de señalización mediante LEDs y un buzzer.


<img width="693" height="538" alt="image" src="https://github.com/user-attachments/assets/cc8b9ec4-8d47-433d-b4b3-77c8e232e0da" />

<img width="853" height="550" alt="image" src="https://github.com/user-attachments/assets/82ca3e83-6448-4283-87bf-24ee7dcb198b" />

Codigo 

    from machine import Pin, ADC, PWM
    import time

    # POTENCIOMETROS 
    pot_base = ADC(Pin(34))
    pot_brazo = ADC(Pin(35))
    pot_base.atten(ADC.ATTN_11DB)
    pot_brazo.atten(ADC.ATTN_11DB)
    pot_base.width(ADC.WIDTH_12BIT)   # Potenciometro 1: 12 bits -> 0 a 4095
    pot_brazo.width(ADC.WIDTH_10BIT)  # Potenciometro 2: 10 bits -> 0 a 1023

    # SERVOS 
    servo_base = PWM(Pin(18))
    servo_base.freq(50)
    servo_brazo = PWM(Pin(19))
    servo_brazo.freq(50)

    # BOTONES 
    btn_retorno = Pin(25, Pin.IN, Pin.PULL_UP)
    btn_rutina  = Pin(26, Pin.IN, Pin.PULL_UP)

    # LEDS 
    led_verde = Pin(4, Pin.OUT)
    led_rojo  = Pin(5, Pin.OUT)

    # BUZZER 
    buzzer = PWM(Pin(23))
    buzzer.freq(2000)
    buzzer.duty(0)

    # VARIABLES GLOBALES 
    angulo_base = 90
    angulo_brazo = 0

    flag_retorno = False
    flag_rutina  = False
    ultimo_tiempo_retorno = 0
    ultimo_tiempo_rutina  = 0
    DEBOUNCE_MS  = 200

    # INTERRUPCIONES CON ANTIRREBOTE
    def isr_retorno(pin):
    global flag_retorno, ultimo_tiempo_retorno
    ahora = time.ticks_ms()
    if time.ticks_diff(ahora, ultimo_tiempo_retorno) > DEBOUNCE_MS:
        flag_retorno = True
        ultimo_tiempo_retorno = ahora

    def isr_rutina(pin):
    global flag_rutina, ultimo_tiempo_rutina
    ahora = time.ticks_ms()
    if time.ticks_diff(ahora, ultimo_tiempo_rutina) > DEBOUNCE_MS:
        flag_rutina = True
        ultimo_tiempo_rutina = ahora

    btn_retorno.irq(trigger=Pin.IRQ_FALLING, handler=isr_retorno)
    btn_rutina.irq(trigger=Pin.IRQ_FALLING, handler=isr_rutina)

    # FUNCION SERVO 
    def mover_servo(servo, angulo):
    duty = int((angulo / 180) * 75 + 40)
    servo.duty(duty)

    # MOVIMIENTO SUAVE 
    def mover_suave(servo, inicio, fin):
    paso = 1 if fin > inicio else -1
    for ang in range(inicio, fin, paso):
        mover_servo(servo, ang)
        time.sleep_ms(15)
    mover_servo(servo, fin)

    # BUZZER ON/OFF 
    def buzzer_on():
    buzzer.duty(800)

    def buzzer_off():
    buzzer.duty(0)

    # RETORNO AUTOMATICO 
    def rutina_retorno():
    global angulo_base, angulo_brazo
    led_verde.off()
    led_rojo.on()
    buzzer_on()
    mover_suave(servo_base,  angulo_base,  90)
    mover_suave(servo_brazo, angulo_brazo, 90)
    angulo_base  = 90
    angulo_brazo = 90
    buzzer_off()
    led_rojo.off()
    led_verde.on()

    # RUTINA AUTOMATICA 
    def rutina_robot():
    global angulo_base, angulo_brazo
    led_verde.off()
    led_rojo.on()
    buzzer_on()
    mover_suave(servo_base,  angulo_base,  0)
    mover_suave(servo_base,  0,            180)
    mover_suave(servo_brazo, angulo_brazo, 90)
    mover_suave(servo_brazo, 90,           0)
    angulo_base  = 180
    angulo_brazo = 0
    buzzer_off()
    led_rojo.off()
    led_verde.on()

    #  INICIO

    led_verde.on()
    led_rojo.off()
    mover_servo(servo_base,  angulo_base)
    mover_servo(servo_brazo, angulo_brazo)

    # LOOP PRINCIPAL

    while True:

    # BOTON RETORNO (via interrupcion)
    if flag_retorno:
        flag_retorno = False
        rutina_retorno()

    # BOTON RUTINA (via interrupcion)
    elif flag_rutina:
        flag_rutina = False
        rutina_robot()

    # MODO MANUAL
    else:
        led_verde.on()
        led_rojo.off()
        buzzer_off()

        v1 = pot_base.read()   # 0 a 4095 (12 bits)
        v2 = pot_brazo.read()  # 0 a 1023 (10 bits)

        nuevo_base  = int(v1 * 180 / 4095)
        nuevo_brazo = int(v2 * 180 / 1023)

        if abs(nuevo_base - angulo_base) > 2:
            mover_servo(servo_base, nuevo_base)
            angulo_base = nuevo_base

        if abs(nuevo_brazo - angulo_brazo) > 2:
            mover_servo(servo_brazo, nuevo_brazo)
            angulo_brazo = nuevo_brazo

    time.sleep_ms(50)


  <img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/72d6d73a-c51a-44dd-8241-aaa8ef179ab3" />


4. Componentes del Sistema
   
 
4.1 Entradas

2 potenciómetros lineales:
Rotación de la base
Elevación del brazo
2 pulsadores con interrupción

4.2 Procesamiento

Microcontrolador: ESP32
Conversión ADC:
Potenciómetro 1: 12 bits
Potenciómetro 2: 10 bits
Generación de señales PWM para servos
Gestión de interrupciones
Manejo de estados mediante variables globales

4.3 Salidas

2 servomotores:
Servo 1: rotación de la base
Servo 2: movimiento del brazo
LEDs:
Verde: modo manual
Rojo: modo automático
Buzzer: alerta sonora durante modo automático

5. Funcionamiento del Sistema
   
5.1 Modo Manual

Los potenciómetros generan señales analógicas proporcionales a su posición.
El ESP32 lee estas señales mediante el ADC.
Los valores se convierten a señales PWM.
Los servos se mueven en tiempo real.
LED verde encendido.

5.2 Modo Automático: Retorno a Posición Inicial

Activado mediante pulsador con interrupción.
Se detiene el control manual.
Los servos regresan gradualmente a su posición inicial.
LED rojo y buzzer se activan.
Al finalizar, vuelve al modo manual.

5.3 Modo Automático: Secuencia Predefinida

Activado mediante otro pulsador.
Se ejecuta una rutina de movimiento programada.
LED rojo y buzzer activos durante la ejecución.
Finaliza regresando al modo manual.

6. Conclusiones

El proyecto demuestra la integración efectiva de hardware y software en sistemas embebidos, utilizando el ESP32 para el control en tiempo real. Se logró implementar un sistema robusto con múltiples modos de operación, haciendo uso de interrupciones, control PWM y lectura analógica.

<img width="393" height="431" alt="image" src="https://github.com/user-attachments/assets/b76776fa-da33-43a7-a8a0-ceb58bb0fafb" />

<img width="356" height="461" alt="image" src="https://github.com/user-attachments/assets/3c9c39ae-d4c1-4a1a-85be-6b6c996ce717" />

<img width="396" height="362" alt="image" src="https://github.com/user-attachments/assets/ff59ecc5-10ac-4dca-9257-f77f6f58081d" />

