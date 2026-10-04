# Basic Vacuum Cleaner
## Realizado por Manuel Martínez Arranz (m.martineza.2024@alumnos.urjc.es)
## Enunciado
Programa una aspiradora de gama baja para que limpie una casa movíendose pseudoaleatoriamente por ella. 
- Se programará un autómata con al menos 3 estados (avanzando, retrocediendo, girando).
- Se puede incluir un estado haciendo-espiral, que aumenta la eficiencia del barrido. Con él se logra una navegación pseudoaleatoria que estocásticamente cubre toda la casa, limpiándola.
- La aspiradora, al ser de gama baja no incorpora algoritmos robustos y precisos de autolocalización. La posición estimada por la odometría **no se puede usar** como posición verdadera, pues acumula ruido. Sí se puede utilizar la orientación para medir giros de modo aproximado.
- El autómata se implementará como un bucle infinito. **No se admiten *sleeps*** que rompan la reactividad del bucle principal.

[![[Unibotics] RoboticsAcademy - Basic Vacuum Cleaner - YouTube](https://i.ytimg.com/vi/qwBQ1B-05xU/maxresdefault.jpg)](https://www.youtube.com/watch?v=qwBQ1B-05xU "[Unibotics] RoboticsAcademy - Basic Vacuum Cleaner - YouTube")

## Robot API
- `import HAL` (Hardware Abstraction Layer). Funciones que envían y reciben información de y al hardware (Gazebo).
- `import WebGUI` (Web Graphical User Interface). Debug e imágenes.
- `HAL.setV(v)` y `HAL.setW(velocity)`. Establece la velocidad lineal y angular respectivamente.
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
---
