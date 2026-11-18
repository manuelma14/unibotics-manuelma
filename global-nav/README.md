# Navegación global de un taxi autónomo con Gradient Path Planning (GPP)
## Realizado por Manuel Martínez Arranz (m.martineza.2024@alumnos.urjc.es)
## Enunciado
Programa un coche autónomo, equipado con un sensor GPS y un mapa de la ciudad, para que ejerza de taxi autónomo siendo capaz de planificar su navegación desde el lugar en el que se encuentra hasta otro punto de la ciudad como destino, utilizando el algoritmo de Gradient Path Planning. Este algoritmo es un ejemplo de plan como recurso, proporcionando un campo escalar que el robot puede emplear en cualquier momento para pedir consejo de hacia dónde moverse para navegar hacia el destino. El campo será la suma de dos componentes, el subcampo de propagación a destino y el subcampo de penalización por cercanía a obstáculos. Además de la planificación utilizando el mapa la aplicación robótica deberá pilotar el coche siguiendo el gradiente de ese campo, gobernando reactivamente la velocidad lineal del coche (V) y la velocidad angular (W) en cada iteración. El pilotaje se implementará como un bucle reactivo. Idealmente, una vez alcanzado un destino remoto, el coche autónomo esperará un nuevo destino que marque el usuario humano picando en otro punto del mapa de la ciudad.

### video

## API
- Básicos: `import WebGUI, import HAL, import Frequency`.
- Concretos: `HAL.getPose3d(), HAL.setV(), HAL.setW(), WebGUI.getMap(), WebGUI.getTargetPose(), WebGUI.worldToGrid(), WebGUI.gridToWorld(), WebGUI.showNumpy(), WebGUI.showPath()`.
- Matemáticas: `import math, import numpy as np`.

## Desarrollo
El algoritmo GPP construye un **campo escalar** sobre la rejilla del mapa (400 x 400 celdas, cada una de 1.25 m) en el que el destino es el punto más bajo y las celdas cercanas a las paredes son más altas. El coche solo tiene que ir buscando el **mínimo** por ese campo para llegar al destino. El programa sigue estos pasos:

### 1. Mapa y coordenadas
`WebGUI.getMap()` devuelve el mapa en escala de grises: **255** es carretera (transitable) y **0** es obstáculo. Para trabajar con la rejilla, tanto el destino (`WebGUI.getTargetPose()`) como la posición del coche (`HAL.getPose3d()`) se pasan de coordenadas del **mundo** a coordenadas de **celda** con `WebGUI.worldToGrid()`, y al revés con `WebGUI.gridToWorld()` cuando hay que convertir una celda en un punto al que apuntar.

`worldToGrid()` devuelve (columna, fila), mientras que la matriz se indexa como `[fila][columna]`, por lo que hay que **invertir el orden** de las coordenadas al acceder al mapa (`mapa[b][a]`).

Cada celda tiene **8 vecinos** (N, S, E, O y las 4 diagonales), y solo se consideran los que son **transitables**.

### 2. Subcampo de penalización
Antes de planificar, y **una sola vez** al principio (solo depende del mapa), se calcula la penalización por cercanía a obstáculos:
- Las celdas de carretera **pegadas a una pared** (les falta algún vecino) reciben el valor máximo, `radio * paso`.
- Desde ellas se avanza capa a capa hacia el centro de la calle, y en cada capa el valor baja `paso`.
- A partir de `radio` celdas de la pared, la penalización es 0.

> [!NOTE]
> Con `radio = 6` y `paso = 2`, la penalización según la distancia a la pared es:
>
> | Distancia a la pared (celdas) | Penalización |
> |:---:|:---:|
> | 1 | 12 |
> | 2 | 10 |
> | 3 | 8 |
> | 4 | 6 |
> | 5 | 4 |
> | 6 | 2 |
> | ≥ 7 | 0 |
>
> Las calles miden unas 14 celdas de ancho, así que el centro de la calle queda sin penalización y el coche tiende a circular por él.

### 3. Subcampo de propagación a destino
La propagación es un **frente de onda** que sale desde el destino (valor 0) y se expande ola a ola por la carretera. Cada vez que la onda pasa de una celda a una vecina suma su coste:
- **1** en las direcciones N, S, E, O (distancia 1).
- **1.4** en las diagonales (distancia √2).

Si a una celda que ya tenía valor le llega uno más pequeño, se actualiza y vuelve a entrar en el frente. La onda no se detiene justo al alcanzar al coche, sino que sigue **30 olas más**, para que el campo también tenga valor alrededor y por detrás del coche.

> [!IMPORTANT]
> En una primera versión, el campo era la suma directa de la propagación y la penalización. Con una penalización fuerte (que suba más de 1 por celda) aparecían **mínimos locales** en las esquinas interiores de los cruces: el coche bajaba por el gradiente y se quedaba atascado antes del destino. Con una penalización suave no había mínimos, pero el camino **cortaba las esquinas** pegado a la pared y el coche rozaba con ella.

### 4. Campo final
El campo que consulta el coche es la **suma de los dos subcampos**, como indica el enunciado: `suma = propagación + penalización`

Las celdas a las que no ha llegado la onda (paredes o zonas lejanas) se marcan con un valor muy alto (`SIN_VALOR = 10000`), y el destino se fuerza a **0** para asegurar que siempre es el punto más bajo del campo, aunque se haya picado cerca del bordillo.

El campo se muestra con `WebGUI.showNumpy()` (blanco cerca del destino, negro lejos o sin valor) y, bajando el gradiente paso a paso desde el coche hasta el destino, se dibuja el camino resultante con `WebGUI.showPath()`. Este camino es **solo visual**: el coche no lo sigue, sino que consulta el campo en cada iteración.

### 5. Pilotaje siguiendo el gradiente
En cada iteración del bucle (a 50 Hz) se localiza la celda del coche y se baja por el gradiente (en cada paso se elige el vecino con menor valor) para obtener **dos objetivos locales**:
- **Punto cercano (5 pasos):** marca hacia dónde girar.
- **Punto lejano (15 pasos):** permite anticipar las curvas antes de llegar a ellas.

Con cada punto se calcula el **error angular** entre hacia dónde quiere ir el coche y hacia dónde mira (`yaw`), normalizado entre -π y π. A partir de ellos:
- **Velocidad angular (W):** proporcional al error del punto cercano (`W = 1.75 * error`).
- **Velocidad lineal (V):** baja poco a poco según lo cerrada que sea la curva, `V = 12 - 12 * curva`, donde `curva` es el mayor de los dos errores, sin bajar nunca de 2.5. Así el coche va rápido en las rectas y frena **antes** de las curvas gracias al punto lejano.

### 6. Llegada al destino y nuevo destino
- En los **últimos 8 m** el coche deja de apuntar al centro de la celda y apunta directamente al punto exacto que marcó el usuario, para que no dé bandazos al llegar.
- La velocidad nunca supera la distancia que queda hasta el destino (`V ≤ dist`), de modo que el coche **frena progresivamente** al acercarse.
- Cuando está a menos de **1.5 m**, se detiene (`V = 0`, `W = 0`) y queda a la espera de que el usuario pique un nuevo destino en el mapa, momento en el que se vuelve a calcular el campo.

> [!TIP]
> Para ver la propagación de la onda **iteración a iteración**, dentro del bucle de propagación se llama a `WebGUI.showNumpy()` cada 5 olas, de modo que se ve cómo el frente avanza desde el destino por las calles hasta alcanzar al coche:
> ```python
> if ver_onda and ola % 5 == 0:
>     WebGUI.showNumpy((255 - np.minimum(matriz, 1000) * 255 / 1000).astype(np.uint8))
> ```
> También resulta útil imprimir el valor del mapa en la celda del destino (`mapa[b][a]`, que debe ser 255) para comprobar que las coordenadas están bien invertidas.

## Resultado
### Vídeo de la solución:

En el vídeo se marcan varios destinos seguidos en distintos puntos de la ciudad. Para cada uno se ve cómo se propaga la onda, el campo resultante y el camino dibujado, y cómo el coche llega al destino, se detiene y espera al siguiente.

## Conclusión
La solución lleva el taxi hasta el destino marcado por el camino más corto **por el centro de las calles**, sin chocar con las paredes, y queda a la espera de un nuevo destino al llegar. Gracias al punto lejano del gradiente, el coche reduce la velocidad antes de las curvas y acelera en las rectas, y gracias al frenado progresivo se detiene sobre el punto marcado.

La parte más delicada fue la **combinación de los dos subcampos**: una penalización demasiado débil hace que el camino corte esquinas, y una demasiado fuerte crea mínimos locales donde el coche se queda atascado. Incluir la penalización en el coste de la propagación resolvió ambos problemas.

El ajuste de parámetros (radio y paso de la penalización, número de pasos de los dos puntos del gradiente y multiplicadores de V y W) se hizo de forma **empírica**.

Como limitación, la propagación de la onda se recalcula entera cada vez que se marca un destino, lo que hace que el coche tarde un momento en arrancar, y los parámetros están ajustados para este mapa concreto. En una ciudad con calles de otra anchura habría que reajustar sobre todo el radio de la penalización.

