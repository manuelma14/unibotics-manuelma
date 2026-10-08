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
- OpenCV: `import cv2`. Procesamiento de imágenes.

## Desarrollo
Tras obtener la imagen mediante `HAL`, aplicaremos un **filtro HSV**. Como tratamos de aislar el color rojo, debemos realizar dos máscaras, una para el rango inferior del rojo y otra para el rango superior, ya que este color se encuentra en ambos extremos del canal de tono (`H`). Posteriormente, uniremos ambas máscaras para obtener una única imagen binaria en la que quede aislada la línea.

Una vez obtenida la máscara, utilizaremos los **momentos de OpenCV** para obtener el **centroide** de la región filtrada. A partir de este centroide podremos calcular el error, ya que podemos obtener la posición del centro de la imagen mediante `image.shape[1] // 2` y calcular cuántos píxeles se encuentra desplazado el centroide respecto a dicho punto.

Con estos datos somos capaces de ajustar la **velocidad angular** mediante un controlador PID. Para este controlador no es necesario modelar la planta del sistema, ya que podemos realizar el ajuste de forma empírica. Iremos escalando progresivamente desde el controlador **proporcional**, añadiendo posteriormente la componente **derivativa** y, por último, la **integral**.

Una vez controlada la velocidad angular, realizaremos el control de la **velocidad lineal** mediante otro controlador PID. Toda la evolución del proceso y el comportamiento final quedarán reflejados en el **vídeo**.

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

## Resultado
### Vídeo de la solución:

## Conclusión
