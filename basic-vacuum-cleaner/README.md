# Basic Vacuum Cleaner
## Realizado por Manuel Martínez Arranz (m.martineza.2024@alumnos.urjc.es)
## Enunciado
Programa una aspiradora de gama baja para que limpie una casa movíendose pseudoaleatoriamente por ella. 
- Se programará un autómata con al menos 3 estados (avanzando, retrocediendo, girando).
- Se puede incluir un estado haciendo-espiral, que aumenta la eficiencia del barrido. Con él se logra una navegación pseudoaleatoria que estocásticamente cubre toda la casa, limpiándola.
- La aspiradora, al ser de gama baja no incorpora algoritmos robustos y precisos de autolocalización. La posición estimada por la odometría **no se puede usar** como posición verdadera, pues acumula ruido. Sí se puede utilizar la orientación para medir giros de modo aproximado.
- El autómata se implementará como un bucle infinito. **No se admiten *sleeps*** que rompan la reactividad del bucle principal.

[![[Unibotics] RoboticsAcademy - Basic Vacuum Cleaner - YouTube](https://i.ytimg.com/vi/qwBQ1B-05xU/maxresdefault.jpg)](https://www.youtube.com/watch?v=qwBQ1B-05xU "[Unibotics] RoboticsAcademy - Basic Vacuum Cleaner - YouTube")

## API
- `import HAL` (Hardware Abstraction Layer). Funciones que envían y reciben información de y al hardware (Gazebo).
- `import WebGUI` (Web Graphical User Interface). Debug e imágenes.
- `import random`.
- `HAL.setV(v)` y `HAL.setW(w)`. Establece la velocidad lineal y angular respectivamente.
- `HAL.getLaserData()`. Para obtener información del sensor láser, que contiene la siguiente información:
> [!IMPORTANT]
> ``` python
> LaserData: {
> minAngle: 0.0007962743428091557
> maxAngle: 3.140796379246984
> minRange: 0.07999999821186066
> maxRange: 10.0
> timeStamp: 1.6
> values: array('f', [...])
> }
> ```
> Para esta práctica usaremos `HAL.getLaserData().values[]`.

> [!WARNING]
> Durante la ejecución será importante comprobar que el array de valores **no se encuentra vacío** mediante una comprobación de su longitud con `len()`.

---

## Desarrollo
> [!NOTE]
> Usaremos una lectura en el rango de 60-120º **mediante un for** de la siguiente forma:
> <p align="center">
>   <img src="https://github.com/manuelma14/unibotics-manuelma/blob/main/basic-vacuum-cleaner/img/1000109229.png" width="250">
> </p>

Implementaremos una máquina de estados (FSM) con **tres estados: AVANZAR, RETROCEDER, GIRAR**. Seguirán la siguiente secuencia:
<p align="center">
  <img src="https://github.com/manuelma14/unibotics-manuelma/blob/main/basic-vacuum-cleaner/img/1000109230.png" width="250">
</p>

El robot avanza de forma continua hasta que el sensor láser detecta un obstáculo en su trayectoria. En ese momento, transita al estado **RETROCEDER**, durante el cual se desplaza hacia atrás durante un número aleatorio de ciclos comprendido entre 40 y 80.

Una vez finalizado el retroceso, el robot pasa al estado **GIRAR**. En este estado, mediante una selección aleatoria, determina la dirección del giro —izquierda o derecha— y establece aleatoriamente su duración, entre 50 y 150 ciclos. Tras completar el giro, el robot vuelve al estado **AVANZAR**, repitiendo este comportamiento de forma iterativa.

### Visual del movimiento:
[![P1. Navegación pseudoaleatoria con FSM en una aspiradora de gama baja - YouTube](https://i.ytimg.com/vi/h5e498e6fZo/maxresdefault.jpg)](https://www.youtube.com/watch?v=h5e498e6fZo "P1. Navegación pseudoaleatoria con FSM en una aspiradora de gama baja - YouTube")

---
## Resultado
### Vídeo de la solución:
[![P1 - Ejemplo de ejecución - YouTube](https://i.ytimg.com/vi/cdsCuL_GSuE/hqdefault.jpg)](https://www.youtube.com/watch?v=cdsCuL_GSuE "P1 - Ejemplo de ejecución - YouTube")

Al fin y al cabo, estamos tratando con una ejecución pseudoaleatoria, y cada ejecución puede ser muy distinta. Tras los 5 minutos que se ven en el vídeo, hemos conseguido limpiar un 25% de nuestro entorno, por lo que sería de esperar que a los 20 minutos hayamos llegado aproximadamente al 80%, como se especifica en los enunciados, aunque no siempre es así. En la siguiente imagen se muestra cómo, realmente, tomando tiempo se llega al porcentaje deseado.

<p align="center">
  <img src="https://github.com/manuelma14/unibotics-manuelma/blob/main/basic-vacuum-cleaner/img/Captura%20desde%202026-10-04%2018-46-26.png" width="700">
</p>

En otro orden de cosas, el uso de la espiral aumenta considerablemente el porcentaje inicial, es decir, es más rápido, pero a la larga recorre muchos espacios innecesarios, por lo que he decidido dejar el sistema de manera sencilla con los 3 estados.


