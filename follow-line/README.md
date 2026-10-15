# Follow Line
## Realizado por Manuel Martínez Arranz (m.martineza.2024@alumnos.urjc.es)
## Enunciado
Programa un coche de Fórmula1, equipado con una cámara frontal, para que recorra un circuito de carreras que tiene una **línea roja** en el medio de la pista. 
- Se programará uno o varios controladores PID para gobernar la velocidad lineal del coche (V) y la velocidad angular (W).
- El coche debe funcionar en varios circuitos, completarlos en el menor tiempo posible y sin alejarse de la línea roja.
- Cuantas menos oscilaciones tenga mejor, y conviene que sea robusto **si en algún momento pierde la línea**.

La aplicación robótica se implementará como un bucle infinito. Cada iteración incluirá las instrucciones para materializar el procesamiento de la imagen y para la toma de decisiones sobre los dos actuadores V y W. 
[![[Unibotics] RoboticsAcademy - Follow Line - YouTube](https://i.ytimg.com/vi/HRZC1-tGW-s/maxresdefault.jpg)](https://www.youtube.com/watch?v=HRZC1-tGW-s&t=1s "[Unibotics] RoboticsAcademy - Follow Line - YouTube")
## API
- Básicos: `import WebGUI, import HAL, import Frequency`.
- Concretos: `HAL.getImage(), HAL.setV(), HAL.setW(), WebGUI.showImage()`.
- OpenCV: `import cv2`. Procesamiento de imágenes.

## Desarrollo
Tras obtener la imagen mediante `HAL`, aplicaremos un **filtro HSV**. Como tratamos de aislar el color **rojo**, debemos realizar dos máscaras, una para el rango inferior del rojo y otra para el rango superior, ya que este color se encuentra en ambos extremos del canal de tono (`H`). Posteriormente, uniremos ambas máscaras para obtener una única imagen binaria en la que quede aislada la línea.
<p align="center">
  <img src="https://github.com/manuelma14/unibotics-manuelma/blob/main/follow-line/img/filtro-hsv.png?raw=true" alt="Filtro HSV" width="650" />
</p>
Una vez obtenida la máscara, utilizaremos los **momentos de OpenCV** para obtener el **centroide** de la región filtrada. A partir de este centroide podremos calcular el error, ya que podemos obtener la posición del centro de la imagen mediante `image.shape[1] // 2` y calcular cuántos píxeles se encuentra desplazado el centroide respecto a dicho punto.

Con estos datos somos capaces de ajustar la **velocidad angular** mediante un controlador PID. Para este controlador no es necesario modelar la planta del sistema, ya que podemos realizar el ajuste de forma empírica. Iremos escalando progresivamente desde el controlador **proporcional**, añadiendo posteriormente la componente **derivativa** y, por último, la **integral**.

<p align="center">
  <img width="450" alt="image" src="https://github.com/user-attachments/assets/4b41d255-6dcc-492b-a8e2-eb58f3ebde8d" />
</p>
Una vez controlada la velocidad angular, realizaremos el control de la **velocidad lineal** mediante otro controlador PID.

> [!TIP]
> Para el procesamiento de la imagen utilizaremos las siguientes funciones:
> ```python
> image = HAL.getImage()
> hsv = cv2.cvtColor(image, cv2.COLOR_BGR2HSV) # BGR -> HSV
> mask1 = cv2.inRange(hsv, low, upper)
> mask2 = cv2.inRange(hsv, low, upper)
> mask = cv2.bitwise_or(mask1, mask2) # combina ambas
> moments = cv2.moments(mask) # para el centroide
> ```
> En cuanto a los momentos, se puede seguir las indicaciones de: https://learnopencv.com/find-center-of-blob-centroid-using-opencv-cpp-python/

> [!NOTE]
> Si el robot pierde la línea, o arranca en un punto donde la máscara no detecta **ningún píxel rojo**, se activa un **comportamiento de recuperación**. Una sentencia `ìf` comprueba si se ha detectado la línea (es decir, si el momento `m00` de la **máscara es distinto de cero**). En caso contrario, mediante `else` se hace que el robot retroceda con una velocidad lineal constante negativa mientras gira lentamente con una velocidad angular pequeña, de modo que vuelva a **encuadrar la línea en la cámara**. Puede verse el ejemplo al inicio del vídeo.
> 
> En la versión PID, además, se reinicia la integral acumulada para que el error previo no afecte al control cuando se recupere la línea.

## Resultado
### Vídeo de la solución:
[![P2 - Follow Line - YouTube](https://res.cloudinary.com/marcomontalbano/image/upload/v1792080831/video_to_markdown/images/youtube--J6l0p0m2j0o-c05b58ac6eb4c4700831b2b3070cd403.jpg)](https://www.youtube.com/watch?v=J6l0p0m2j0o "P2 - Follow Line - YouTube")

<p align="center">
  <img src="https://github.com/manuelma14/unibotics-manuelma/blob/main/follow-line/img/Captura%20desde%202026-10-15%2018-51-00.png?raw=true" alt="" width="650" />
</p>

<p align="center">
  <img src="https://github.com/manuelma14/unibotics-manuelma/blob/main/follow-line/img/Captura%20desde%202026-10-15%2018-54-45.png?raw=true" alt="" width="650" />
</p>


## Conclusión
El rendimiento de la simulación depende en buena medida del ordenador en el que se ejecuta. En mi caso he observado ejecuciones con muchas oscilaciones y otras prácticamente perfectas con exactamente el mismo código, un efecto que se nota sobre todo al grabar, ya que la grabación consume recursos y altera la frecuencia real del bucle de control, que está diseñado para funcionar a 50 Hz.

Aun así, el robot adapta tanto su velocidad lineal como la angular en función del error y de su variación, frenando en las curvas y acelerando en las rectas, lo que lo hace robusto ante distintos circuitos y no solo ante el circuito inicial.

La solución mostrada en el vídeo no incorpora control integral (PD), ya que con ella se cumple el objetivo planteado en clase de completar el circuito inicial en menos de 60 segundos. No obstante, en la entrega se incluyen dos archivos, uno con control PD y otro con control PID, por si la integral fuese un requisito explícito.

Por último, cabe señalar que esta solución está implementada para modelos holónomos, es decir, aquellos capaces de girar sobre sí mismos, lo que permite la maniobra de recuperación al perder la línea. No he tratado de implementarla ni de adaptarla a modelos Ackermann.
