#Control PID con Arduino

Se trataba en este caso de probar una regulación PID mediante la tarjeta Arduino NANO.

El sistema consistía en una fiambrera, que sería el recinto cuya temperatura interior deseaba controlar mediante un ventilador que disiparía el calor producido por una lámpara incandescente, por medio de la cual se podría calentar el aire interior. Dicha lámpara iría conectada a la red convencional mediante un potenciómetro para poder regular su intensidad luminosa y, por ende, el calor emitido.

La cosa tenía esta pinta... Donde se puede apreciar el ventilador situado en la parte superior de la tapa de la fiambrera y junto a éste, se encuentra un módulo amplificador que tuve que construir para poder gobernar el ventilador mediante señales TTL de 5V de la placa Arduino. Dado que el ventilador era capaz de funcionar a 24V y se alimentaba con una fuente de unos 15V en vacío (la historia de esta fuente está en otra entrada de este mismo blog (Fuente de alimentación regulada).

![](imagenes/PIDarduino/fotoPIDarduino01.jpg)

La fiambrera disponía en su parte inferior de unos pequeños agujeros laterales por los que el aire impulsado por el ventilador podía salir. Se puede ver también en la foto una placa de prototipos y, conectada a esta, la estupenda placa Arduino NANO, con su cable de conexión al ordenador, directamente llegados desde Hong Kong.

Dentro de la fiambrera está el sensor de temperatura LM35 que comunicaría con Arduino.  El citado sensor tiene esta pinta, una vez montado en su módulo correspondiente...

![](imagenes/PIDarduino/fotoPIDarduino02.jpg)

Volviendo al módulo amplificador, también se ha luchado lo propio... El módulo hubo que añadirlo para poder hacer funcionar un ventilador de 24V a partir de una fuente de 15V de f.e.m. mediante un impulso de 5V... He aquí el esquema del módulo conectado al ventilador y un boceto del montaje.

![](imagenes/PIDarduino/fotoPIDarduino03.jpg)

Nótese que en el ventilador hay un diodo conectado en antiparalelo para evitar las corrientes de retorno producidas en la bobina del propio motor y que podrían dañar, a la larga, el circuito.

Concluido el montaje, quedaban dos tareas pendientes: una, la estimación de los parámetros de la regulación, a saber, las constantes proporcional, integral y derivativa. La segunda, la elaboración de un sketch para Arduino.

Empezando con la estimación del orden de magnitud de los parámetros característicos dinámicos del sistema: kp, ki, kd a partir de la curva de reacción (lazo abierto). Se tomaron datos experimentales sobre el enfriamiento del recinto mediante el ventilador al 100% de su potencia, después de ser calentado con la lámpara, también esta al 100% de su intensidad luminosa.

![](imagenes/PIDarduino/fotoPIDarduino04.jpg)

La siguiente figura muestras los resultados obtenidos. En el eje de ordenadas se representa la temperatura en grados centígrados del interior del recinto y en el eje de abcisas el tiempo en segundos.

![](imagenes/PIDarduino/fotoPIDarduino05.png)

En la figura también se puede apreciar una curva en rojo que representa el modelo teórico de decaimiento exponencial que se empleó para ajustar los puntos. Se trata de la Ley del enfriamiento de Newton. Una expresión del tipo:
T=Tm + C·e^(-k·t)

Donde T es la temperatura medida del sistema, Tm la temperatura del medio (del baño térmico se entiende), C y K son constantes y t es el tiempo. Es importante notar que no se ha realizado ninguna prueba estadística sobre la calidad del ajuste por no ser relevante en esta situación ya que se trata solo de una estimación para conocer el orden de magnitud de las constantes kp, ki y kd y tener algún punto de partida para abordar el problema.

Aproximadamente, los datos de partida fueron los siguientes:
Tm = 24.41ºC, temperatura del entorno
Tt = 33.20ºC, temperatura del sistema poco después de empezar el enfriamiento
Tt+50 = 27.83ºC, temperatura del sistema 50 segundos después de la anterior medida Tt.

Normalizando Tt=0s y por consiguiente también Tt+50=50s obtenemos los siguientes valores para las constantes de la expresión anterior:
C = 8.79 ºC
k = 0.0189 1/ºC

A partir de la derivada de esta función y mediante diversos cálculos, se pudieron encontrar el tiempo de retraso Tu, el tiempo de regulación intrínseca TG y el coeficiente de transferencia. De tal forma que a partir de estos valores pudieron estimarse de forma aproximada el orden de magnitud de las constantes dinámicas kp = 50, ki = 0.025 y kd = 1.

Con estos valores de partida se pudo realizar el sketch para Arduino que a continuación se muestra:

    /* Control proporcional+integral+derivativo de la temperatura:
     Proyecto Fiambrera
     Juan Pedro Perez Moreno
     abril 2016
    */

    // Definicion de E/S
    int sensorPIN = 0;      //sensor LM35 en la entrada analogica 0
    int ventiladorPIN = 5;  //salida digital PWM 5 al ventilador

    //Definicion de variables del control proporcional
    int TiempoMuestreo = 1;    //Tiempo de muestreo en milisegundos
    unsigned long pasado = 0;  //Tiempo pasado (Se hace para asegurar tiempo de muestreo)
    unsigned long ahora;
    float tempConsigna = 27.0;  //referencia
    double tempMedida;       //Salida medida por el sensor LM35
    double error;           //error en tiempo t
    double error1;
    double errorAnterior = 0;  //error en tiempo t-1
    double errorIntegrado = 0;  //error integrado, el errore se va sumando
    double errorDerivativo =0; //error derivativo
    double control;       //Señal control Proporcional e Integral

    float kp = 50.0;        //constante proporcional del controlador 
    float ki = 0.025;    //constante integral del controlador
    float kd = 1.0;       //constante derivativa del controlador

    float controlP;
    float controlI;
    float controlD;
    float controlPID;

    void setup(){
       Serial.begin(9600);
    }

    void loop() {
  
      //lectura de datos del sensor
      int lectura=analogRead(sensorPIN);
      float voltaje= 5.0/1024.0 * lectura;
      float tempMedida= voltaje*100;
      delay(1);
  
      ahora=millis();                     // tiempo actual 
      int CambioTiempo= ahora-pasado;    // diferencia de tiempo actual- pasado
  
      if(CambioTiempo >= TiempoMuestreo){    // si se supera el tiempo de muestreo
    error1 = tempMedida - tempConsigna;
    error = abs(tempMedida - tempConsigna);  // error Lazo cerrado
    errorIntegrado= error*TiempoMuestreo+errorIntegrado;  //es la integral del error
    errorDerivativo=(error-errorAnterior)/TiempoMuestreo;  //es la derivada del error
    
    if(error1>0){
      controlP = kp*error;      //señal control P
      controlI = ki*errorIntegrado;   //señal control I
      controlD = kp*errorDerivativo;  //señal contrl D
      controlPID = controlP + controlI + controlD;  //Señal control PID
    }
    
    if(error1<=0){
      error=0;
      errorIntegrado=0;
      errorDerivativo=0;
      controlPID=0;
    }
    pasado=ahora;            // actualizar tiempo
    errorAnterior=error1;    //actualizo error
    
    if (controlPID > 255){  // límite de saturación de la señal de control
      controlPID=255;

    }
    }
    analogWrite(ventiladorPIN,controlPID);  //envio al ventilador la velocidad
  
    Serial.print("   Actuador%= ");
    Serial.print(controlPID/255.0*100);
    Serial.print("   ");
    Serial.print("   Error% = ");
    Serial.print(error*100/tempConsigna);
    Serial.print("   ");
    Serial.print("   T_Medida= ");
    Serial.print(tempMedida);
    Serial.print(" *C     ");
    Serial.print("     T_Consigna= ");
    Serial.print(tempConsigna);
    Serial.print(" *C     ");
    Serial.print("       ");
    Serial.print("controlP= ");
    Serial.print(controlP);
    Serial.print("       ");
    Serial.print("controlI= ");
    Serial.print(controlI);
    Serial.println("   ");
    }
 
##Conclusiones:

Se ha realizado una regulación PID de un sistema formado por un depósito calentado interiormente por una lámpara y refrigerado por un ventilador manteniendo los valores de temperatura elegidos como consigna dentro de un rango de 25ºC a 30ºC.

En la realización del sketch para la placa controladora Arduino no se ha precisado el empleo de librerías PID para tal efecto. Se investigará en el futuro el uso de dichas librerías.

Para poder realizar la regulación, la intensidad luminosa de la lámpara tuvo que mantenerse sobre el 25-35% de su máximo. Por encima de estos niveles el ventilador utilizado no tenía suficiente capacidad de regulación.

En cuanto a la temperatura elegida como consigna, no resultó efectiva la regulación de este sistema por debajo de 24ºC.

Es importante tener en cuenta que para realizar un control PID efectivo en este sistema no deben realizarse cambios ni muy grandes ni muy rápidos en la temperatura de consigna puesto que el sistema carece de filtro anti windup y en ese caso la acción integral saturará el actuador.

En la presente entrada PID con Arduino han resultado clarificadoras, como siempre, las conversaciones con mi amigo el profesor José Luís Fernández Albéndiz y fundamental la aportación del profesor Javier Fernández Reyes. Gracias a ambos.
