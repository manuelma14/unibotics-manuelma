# Navegación local esquivando obstáculos con algoritmo VFF
## Realizado por Manuel Martínez Arranz (m.martineza.2024@alumnos.urjc.es)
## Enunciado
Programa un coche de Fórmula1, equipado con un **sensor láser** y un **sensor GPS**, para que recorra un circuito de carreras con vallas en los laterales y con otros coches como obstáculos a lo largo del recorrido.
- Se programará el algoritmo de navegación local VFF con fuerzas virtuales para gobernar la velocidad lineal del coche (V) y la velocidad angular (W) en cada iteración.
- El coche debe completar el circuito en el menor tiempo posible y siempre manteniendo la seguridad de que no choca con ningún obstáculo.
- El sistema de navegación global ha planificado una secuencia de subobjetivos de modo que si se van alcanzando consecutivamente garantiza que se completa la vuelta al circuito.
- Se considerará un subobjetivo alcanzado cuando la posición del coche está razonablemente de la posición del subobjetivo. En ese caso se pasará a navegar localmente hacia el siguiente subobjetivo.

La aplicación robótica se implementará como un bucle infinito. Cada iteración incluirá las instrucciones para materializar el procesamiento del sensor laser, las fuerzas repulsivas, la fuerza atractiva, su combinación y su traducción a decisiones sobre los dos actuadores V y W. 

[![[Unibotics] RoboticsAcademy - ObstacleAvoidance - YouTube](https://res.cloudinary.com/marcomontalbano/image/upload/v1793302242/video_to_markdown/images/youtube--8DiCRFsrxUg-c05b58ac6eb4c4700831b2b3070cd403.jpg)](https://www.youtube.com/watch?v=8DiCRFsrxUg&t=1s "[Unibotics] RoboticsAcademy - ObstacleAvoidance - YouTube")

## API
- Básicos: `import WebGUI, import HAL, import Frequency`.
- Concretos: `HAL.getPose3d(), HAL.getLaserData(), HAL.setV(), HAL.setW(), WebGUI.getNextTarget(), WebGUI.showForces()`.
- Matemáticas: `import math`.

## Desarrollo
El algoritmo VFF asigna una **fuerza de atracción** hacia el subobjetivo actual y una **fuerza de repulsión** generada por los obstáculos detectados. La dirección que sigue el coche es la suma de ambas. En cada iteración del bucle (a 50 Hz) se siguen estos pasos:

### 1. Vector atractivo
El GPS proporciona la posición y la orientación del coche en el sistema de coordenadas **absoluto** (origen en la línea de meta), igual que el subobjetivo. Como el vector debe expresarse desde el punto de vista del coche, se transforma la posición del subobjetivo al sistema de coordenadas **relativo** al robot (X hacia delante, Y hacia la izquierda), restando la posición del coche y rotando según su orientación (`yaw`).

El vector resultante se **normaliza**, de modo que solo aporta dirección y su módulo es siempre 1. Además, se calcula la distancia al subobjetivo: cuando baja de un umbral (1.5 m), se marca como alcanzado y el coche pasa a perseguir el siguiente.

### 2. Vector repulsivo
El láser entrega 180 lecturas, una por grado, que cubren de 0º a 180º. El índice 90 corresponde a la parte frontal del coche, por lo que a cada lectura se le resta 90º para centrar el ángulo en cero delante del coche, positivo a la izquierda y negativo a la derecha. Con la **distancia** y el **ángulo** se obtiene la lectura en **coordenadas polares**.

Cada lectura genera un vector repulsivo que:
- Apunta en sentido **contrario** al obstáculo.
- Tiene un módulo **inversamente proporcional** a la distancia (`1/d`), de modo que cuanto más cerca está el obstáculo, más repele.
- Se ignora si la lectura es demasiado cercana (`< 0.05 m`) o demasiado lejana (`> 4.5 m`), porque no es relevante para la navegación.

> [!NOTE]
> La magnitud de la fuerza repulsiva de cada lectura del láser es inversamente proporcional a la distancia (1/d), por lo que un obstáculo tiene mucha más influencia cuanto más cerca está:
>
> | Distancia al obstáculo (d) | Magnitud repulsiva (1/d) |
> |:---:|:---:|
> | 4.00 m | 0.25 |
> | 1.00 m | 1.00 |
> | 0.25 m | 4.00 |
> | 0.10 m | 10.00 |
>
> Una lectura a 0.10 m aporta **40 veces más repulsión** que una a 4 m, así que un obstáculo cercano domina la suma aunque haya muchas lecturas lejanas.

Los 180 vectores se suman en un único vector repulsivo.

> [!IMPORTANT]
> Como indica la teoría, el vector repulsivo suele apuntar casi siempre hacia atrás, porque hay **"paredes" a ambos lados**, y sumado directamente al atractivo daría un resultado poco útil. Para evitarlo, el vector repulsivo **solo se tiene en cuenta** cuando su módulo supera un **umbral ajustado empíricamente (45)**. Si no lo supera, **se descarta** y el coche se guía solo por el **vector atractivo**. Cuando lo supera, se normaliza igual que el atractivo. Este comportamiento se podrá observar en el vídeo donde al inicio el vector atractivo es superpuesto por el vector resultante, ya que **no existe un vector repulsivo** (`0, 0`) debido a este ajuste.

### 3. Vector resultante
El vector final es la suma del atractivo y del repulsivo multiplicado por un **peso** (`1.55`). Que el peso sea mayor que 1 hace que, cuando hay peligro cercano, la evitación domine sobre la atracción al subobjetivo, lo que da prioridad a la seguridad. Ambos pesos se ajustaron por prueba y error.

### 4. Control de V y W
A partir del vector resultante se calculan sus dos características:
- **Velocidad lineal (V):** proporcional al **módulo** del vector, limitada entre un mínimo y un máximo. Cuando el repulsivo se opone al atractivo (obstáculo delante), el módulo disminuye y el coche **frena de forma natural**. Cuando el camino está despejado, el módulo es mayor y el coche acelera.
- **Velocidad angular (W):** proporcional al **ángulo** del vector resultante respecto al frente del coche, saturada a un valor máximo. Para evitar giros bruscos y oscilaciones se aplica un **filtro de suavizado** que mezcla el giro deseado con el giro de la iteración anterior (factor `alpha 0.18`).

### 5. Organización del código
Para que el bucle principal sea legible, la lógica se ha dividido en funciones de una única responsabilidad: transformación de coordenadas, lectura del láser, cálculo del vector atractivo, cálculo del vector repulsivo, combinación de vectores y cálculo de velocidades. Las constantes de ajuste se han agrupado al inicio del programa para facilitar su ajuste.

Por último, en cada iteración se visualizan los tres vectores (atractivo, repulsivo y resultante) en la interfaz web para poder depurar el comportamiento.

> [!TIP]
> Para ajustar los parámetros de forma empírica resulta muy útil **imprimir trazas** en cada iteración, de modo que se vea cómo evolucionan los vectores y las velocidades. Es especialmente importante observar el módulo del vector repulsivo **antes** de aplicar el umbral, ya que es el valor que permite decidir su filtro:
> ```python
> print(
>   f"att=({att_x:.2f}, {att_y:.2f}) "
>   f"rep=({rep_x:.2f}, {rep_y:.2f}) "
>   f"norm_rep={norm_rep:.2f} "
>   f"result=({result_x:.2f}, {result_y:.2f}) "
>   f"angle={angle:.2f} "
>   f"module={module:.2f} "
>   f"v={v:.2f} "
>   f"w={w:.2f}"
> )
> ```
> Con estas trazas se puede ver, por ejemplo, qué valores toma `norm_rep` en una recta despejada, entre paredes o ante un coche obstáculo, y fijar el umbral en consecuencia. Conviene imprimir con pocos decimales para que la salida sea legible, y comentar la traza cuando no se esté depurando, ya que imprimir en cada iteración a 50 Hz puede ralentizar el bucle.

## Resultado
### Vídeo de la solución:
[![P3 - Navegación local esquivando obstáculos con algoritmo VFF - YouTube](https://res.cloudinary.com/marcomontalbano/image/upload/v1793303990/video_to_markdown/images/youtube--hOtptHaE51I-c05b58ac6eb4c4700831b2b3070cd403.jpg)](https://www.youtube.com/watch?v=hOtptHaE51I "P3 - Navegación local esquivando obstáculos con algoritmo VFF - YouTube")

El código de la entrega corresponde con la primera ejecución mostrada en el video, las otras 3 son previas al ajuste final, aunque también cumplen el circuito sin colisionar, pero quizás con una oscilación más notable.

## Conclusión
La solución completa el circuito en aproximadamente **2 minutos** sin colisiones, esquivando los coches obstáculo y manteniéndose alejado de las "paredes". Al frenar de forma natural cuando la repulsión se opone a la atracción y acelerar en tramos despejados, el coche adapta su velocidad al entorno sin necesidad de un controlador específico para V.

El ajuste de parámetros (umbral de repulsión, peso de la repulsión, factor de suavizado y multiplicadores de V y W) se hizo de forma **empírica**.

Como limitación, el ajuste del **umbral de repulsión** (y del resto de parámetros) se ha realizado de forma empírica sobre el circuito proporcionado, por lo que la solución **está adaptada a ese circuito concreto**. En otros circuitos, con otra anchura de pista, otra disposición u otra densidad de obstáculos, el módulo del vector repulsivo tomaría valores distintos y los parámetros ajustados podrían no ser robustos, haciendo necesario reajustarlos.
