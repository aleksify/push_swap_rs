

## Introducción

Este proyecto comenzó como una asignatura de la School 42: escribir un programa que ordene una pila de enteros utilizando un conjunto limitado de 11 operaciones, en la menor cantidad de movimientos posible.

Intenté ir más allá. El recorrido fue aproximadamente así:

1. **Múltiples algoritmos en paralelo.** El solucionador ejecuta distintos algoritmos de ordenamiento concurrentemente en cada entrada y selecciona aquel que produzca la salida más corta. Diferentes algoritmos ganan en distintas distribuciones de entrada.
2. **Un optimizador de tipo peephole.** Algunos algoritmos emiten secuencias con redundancias locales: `ra` seguido de `rra` se cancela, `ra` seguido de `rb` puede colapsar a `rr`, etc. Escribí un optimizador de tipo peephole que post-procesa la salida, reescribiendo estos patrones para eliminarlos. La primera versión usó un puñado de reglas escritas a mano.
3. **Un superoptimizador para generar las reglas.** Escribir reglas de reescritura a mano es tedioso e incompleto; siempre faltará algún patrón. Por eso construí un superoptimizador: una búsqueda BFS exhaustiva sobre el espacio de estados de configuraciones de pilas que descubre cada secuencia de operaciones reducible hasta una profundidad dada. La tabla de reglas del optimizador se genera en tiempo de compilación a partir de esta búsqueda.
4. **Golpear el muro de escalabilidad.** Más allá de cierta profundidad, la cantidad de reglas y el tamaño del binario explotan mientras las ganancias reales disminuyen. Esto llevó a pensar en optimización específica por algoritmo en lugar de reglas universales, véase [Problemas Actuales](#current-issues).
5. **¿Qué tan bajo puede llegar?** Derivamos el límite inferior teórico-informático a partir del factor de ramificación efectivo del conjunto de operaciones, luego extrapolamos el óptimo real con búsqueda BFS. Véase [Mínimo Teórico](#theoretical-minimum).

### Lista de TODO para investigación futura
- Comportamiento en cada nivel de desorden, en particular cómo cambia la eficiencia del superoptimizador.
- n > 500.
- Reemplazar el algoritmo BFS (`sort_bfs`, no `exact_bfs`) con A*, pero encontrar una buena heurística es la parte difícil (quizás bases de datos de patrones). También mejorar BFS/A* usando encuentro por la mitad (meet-in-the-middle).

Tabla de Contenidos
=================

* [Cómo ejecutarlo](#how-to-run)
* [El juego](#the-game)
* [Optimizador](#optimizer)
* [Superoptimizador](#superoptimizer)
* [Problemas Actuales](#current-issues)
* [Más Reflexiones](#more-thoughts)
* [Mínimo Teórico](#theoretical-minimum)
* [Cómo compilarlo](#how-to-build)

## Cómo ejecutarlo

Puedes compilarlo localmente (instrucciones al final), o si no tienes Rust y cargo instalados, puedes descargar un binario desde la página de Releases. Fueron generados usando GitHub Actions, para lo cual existe un registro, así que puedes verificar que no haya sido manipulado.

Usa esto para descargar el binario de Linux y asignarle permisos de ejecución:
```
curl -L -o push_swap https://github.com/aleksify/pushswap-optimizer/releases/download/v0.4/push_swap && chmod +x push_swap
```

El binario por defecto es un binario genérico de Linux que debería funcionar en cualquier distribución, ya que está vinculado estáticamente con musl. El binario `_mac` está compilado para Macs con Apple Silicon, pero para ejecutarlo tendrías que ejecutar algunos comandos ya que Apple por defecto prohíbe ejecutar binarios no firmados, así que para ser honestos es más fácil compilar localmente. Pero puedes usar estos comandos para ejecutar ese binario: `xattr -cr push_swap_mac` para eliminarlo de cuarentena, y `codesign --force --deep -s - push_swap_mac` para firmarlo tú mismo.

## El juego

`push_swap` es un proyecto de la School 42. El desafío: dada una pila A de enteros, ordénala en orden ascendente utilizando solo un conjunto limitado de operaciones, y hazlo en la menor cantidad de movimientos posible.

Las reglas:
- Tienes dos pilas, A y B. A comienza con todos los valores de entrada; B comienza vacía.
- Solo puedes manipular las pilas a través de 11 operaciones: intercambios (`sa`, `sb`, `ss`), empujes (`pa`, `pb`), rotaciones (`ra`, `rb`, `rr`) y rotaciones inversas (`rra`, `rrb`, `rrr`).
- El objetivo es conseguir que todos los valores estén ordenados en A usando la menor cantidad de operaciones.

Aunque se llaman "pilas", las operaciones de rotación (mover el elemento superior al inferior o viceversa) significan que en realidad se comportan más como colas dobles (deques).

Las operaciones compuestas (`ss`, `rr`, `rrr`) se aplican a ambas pilas simultáneamente por el costo de un solo movimiento; por ejemplo, `ss` ejecuta `sa` + `sb` en una sola operación, por lo que combinar dos operaciones independientes de una sola pila en una compuesta ahorra un movimiento.

Por defecto, el binario `push_swap` ejecuta todos los algoritmos de ordenamiento disponibles en paralelo (selección, inserción, k_chunk, turk) y selecciona aquel que produzca la solución más corta. Puedes seleccionar un algoritmo específico con `--turk`, `--selection`, `--insertion` o `--k_chunk`. Usa `--bench` para una salida de benchmark que compare la cantidad de operaciones, y `--no-opt` para deshabilitar el optimizador.

`--n1` añade corredores de reoptimización con conocimiento de valores: la salida de cada variante quick3 se post-procesa reemplazando sub-ordenamientos recursivos pequeños con secuencias de operaciones demostrablemente más cortas encontradas mediante búsqueda bidireccional sobre un gráfico de ventanas con rangos comprimidos (`src/reopt.rs`). Ahorra ~70 ops/entrada en n=500 (mejor-media ~3717 → ~3645) pero cuesta ~0.4s de búsqueda por variante, por lo que está desactivado por defecto.

```
./push_swap 3 1 2                   # ordenar, elegir mejor algoritmo
./push_swap --turk 3 1 2            # usar solo el algoritmo turk
./push_swap --bench 3 1 2           # benchmark de todos los algoritmos
./push_swap --bench --turk 3 1 2    # benchmark solo de turk
./push_swap --n1 3 1 2              # añadir los corredores de reoptimización N1 (más lentos)
```

## Optimizador

El optimizador (`src/optimizer.rs`) es un optimizador de tipo peephole universal que post-procesa la secuencia de operaciones producida por cualquier algoritmo de ordenamiento, reescribiéndola para usar menos movimientos. Aplica pases repetidamente hasta que no se encuentren más reducciones.

Opera en dos pases:

1. **Paso de normalización**: Entre operaciones barrera (`pa`, `pb`, `ss`, `rr`, `rrr`), las operaciones en la pila A y la pila B son independientes y pueden reordenarse libremente. Este paso agrupa las operaciones de A y B dentro de cada bloque libre de barreras, llevando las operaciones de la misma pila a estar adyacentes entre sí. Esto expone cancelaciones y fusiones que no serían visibles en el orden intercalado original. Se prueban los ordenamientos con prioridad a A y con prioridad a B, y se conserva el resultado más corto.

2. **Paso de tipo peephole**: Escanea con ventanas de ancho variable (primero las más largas, codiciosamente) y aplica reglas de reescritura desde una tabla de búsqueda. Las reglas vienen en dos variedades:
   - **Aniquiladores**: secuencias que se cancelan por completo (por ejemplo, `ra rra` -> vacío).
   - **Reducciones**: secuencias reemplazables por equivalentes más cortos (por ejemplo, `ra rb` -> `rr`, o `ra pb rra` -> `sa pb`).

   En caso de coincidencia, la ventana retrocede para capturar reducciones en cascada.

Las propias reglas de reescritura son generadas por el superoptimizador (véase abajo) y se incrustan en tiempo de compilación desde `superopt_cache.json`.

## Superoptimizador

Un [superoptimizador](https://en.wikipedia.org/wiki/Superoptimization) es una técnica originaria de la investigación en compiladores: en lugar de aplicar reglas de reescritura escritas a mano, busca exhaustivamente el espacio de todas las secuencias posibles de instrucciones para encontrar reemplazos demostrablemente óptimos. Los compiladores tradicionales usan la superoptimización para descubrir reglas de tipo peephole para la selección y programación de instrucciones.

Nuestro superoptimizador (`src/bin/superopt.rs`) genera la tabla de reglas de reescritura utilizada por el optimizador. Funciona mediante **búsqueda en grafo BFS** sobre el espacio de estados de configuraciones de pilas:

1. Partiendo de un estado canónico de dos pilas, explora todas las secuencias posibles de las 11 operaciones, nivel por nivel (profundidad 1, profundidad 2, ..., hasta profundidad N).
2. Un **oráculo** (mapa hash de estado de pila a secuencia de operaciones más corta conocida) rastrea la primera vez que se alcanza cada estado.
3. Cuando una secuencia más larga alcanza un estado ya conocido, la diferencia es una regla de reescritura: la secuencia más larga puede reemplazarse por la más corta (una **reducción**), o eliminarse por completo si el estado es el estado inicial (un **aniquilador**).
4. Un **conjunto de sufijos reducibles** poda la búsqueda: cualquier secuencia que termine en un patrón conocido como reducible se omite, ya que nunca podría ser óptima.
5. Todas las reglas descubiertas se **verifican con fuzzing** contra 1,000 configuraciones de pilas aleatorias para protegerse contra errores.

El estado canónico usa pilas de tamaño `2N+1` (con un mínimo de 3). Este tamaño es el mínimo necesario para garantizar que las 11 operaciones produzcan transiciones de estado distintas; con menos elementos, algunas operaciones se vuelven degeneradas (por ejemplo, el intercambio es idéntico a la rotación en una pila de 2 elementos), y las reglas descubiertas podrían no generalizarse a pilas más grandes.

Los resultados se almacenan en caché en `superopt_cache.json`, y la búsqueda puede reanudarse desde la última profundidad explorada. La caché se incrusta en el binario del optimizador en tiempo de compilación mediante `include_str!`.

## Problemas Actuales

El enfoque exhaustivo del superoptimizador choca con tres muros de escalabilidad a medida que N crece:

- **Memoria**: El oráculo BFS crece exponencialmente con la profundidad. Empaquetar en bits las secuencias de operaciones y estados podría ayudar, pero solo retrasa lo inevitable: más allá de N=10 aproximadamente, el conjunto de trabajo necesitaría respaldarse en una base de datos en disco en lugar de mantenerse en RAM.
- **Tamaño del binario**: Todas las reglas descubiertas se incrustan en el binario final mediante `include_str!`. En N=8, el binario se aproxima a 400 MB; en N=9, está cerca de 2 GB.
- **Retornos decrecientes**: El número de reglas explota con la profundidad, pero la mayoría nunca se activan en la práctica. Las reglas de mayor profundidad coinciden con patrones cada vez más raros que la mayoría de los algoritmos raramente producen.

En resumen: el uso de RAM, el tamaño del binario y la cantidad de reglas explotan, mientras que las ganancias reales de optimización disminuyen rápidamente.

**¿Hacia dónde ir desde aquí?** En lugar de descubrir reglas universalmente a través de todos los estados posibles de pilas, una dirección más prometedora sería la optimización específica por algoritmo: generar reglas solo para patrones que un algoritmo dado realmente produce. Dos enfoques:

1. **Búsqueda impulsada por corpus**: Ejecutar fuzzing a cada algoritmo con miles de entradas aleatorias, recolectar las secuencias de operaciones y ejecutar el superoptimizador solo sobre ese corpus. Esto reduce drásticamente el espacio de búsqueda al enfocarse en estados que el algoritmo realmente visita.
2. **Podado ex post**: Ejecutar el superoptimizador completo, luego hacer pruebas de fuzzing para identificar qué reglas se aplicaron realmente y descartar el resto.

Dicho esto, incluso estos enfoques enfrentan retornos decrecientes. Los algoritmos heurísticos como Turk ya son lo suficientemente inteligentes en su selección de movimientos que queda menos margen para que un optimizador ex post pueda mejorarlos.

## Más Reflexiones

Hasta ahora, cada enfoque en este proyecto es либо un algoritmo de ordenamiento diseñado a mano либо un optimizador ex post sobre uno de ellos. Pero existe toda otra clase de técnicas que vale la pena considerar: aquellas que *buscan* soluciones directamente en lugar de construirlas procedimentalmente:

- **Algoritmos genéticos**: Evolucionar una población de secuencias de operaciones. El cruce une secuencias, la mutación cambia o inserta operaciones, y la selección conserva los más aptos. A lo largo de las generaciones, la población converge hacia soluciones más cortas.
- **Aprendizaje por refuerzo**: Tratar el ordenamiento como un juego. Estado = pilas actuales, acciones = las 11 operaciones, recompensa = finalización del ordenamiento (menos un costo por operación). Entrenar una red de política (por ejemplo, PPO, MCTS estilo AlphaZero) para elegir movimientos. La red aprende a navegar el espacio de estados sin reglas explícitas.
- **Búsqueda heurística avanzada**: Búsqueda de haz (Beam search) o Búsqueda de Árbol Monte Carlo con horizonte acotado. En cada paso, expandir secuencias de movimientos candidatas hasta la profundidad K, puntuar los estados resultantes y comprometerse con el primer movimiento de la mejor ruta.

**El obstáculo común: la función de aptitud (fitness).** Los tres enfoques necesitan una forma de puntuar qué tan "cerca de estar ordenado" está un estado (A, B) dado. La opción ingenua: contar inversiones en A, o medir el desplazamiento desde el orden ordenado, fundamentalmente no funciona para push_swap.

La razón: **ordenar a menudo requiere primero *aumentar* el desorden.** Para ordenar una pila con push_swap, típicamente empujas valores a B, los reorganizas allí y los empujas de vuelta en el orden correcto. Durante este proceso, A se ve más desordenado que al principio; se han eliminado valores, las rotaciones han mezclado lo que queda. Una función de aptitud ingenua castigaría exactamente los movimientos que un buen algoritmo necesita hacer. Es como la Torre de Hanói: el progreso requiere estados intermedios que parecen regresiones.

Una función de aptitud funcional necesitaría modelar la *estructura* de un ordenamiento válido, no solo la apariencia del orden. Algunas ideas:

- **Descomposición con conocimiento de valores**: Permitir que B esté "ordenada descendente" y A "ordenada ascendente"; medir inversiones dentro de cada una, pero tratar la división misma como gratuita. Penalizar solo cuando los valores están en la pila equivocada *y* en el orden relativo incorrecto.
- **Distancia a un canónico alcanzable**: Precomputar (vía BFS estilo superoptimizador) la ruta más corta desde cualquier estado pequeño hasta el objetivo ordenado, y usar eso como una métrica de distancia aprendida para estados más grandes.
- **Fitness aprendido**: Permitir que el agente de RL aprenda su propia función de valor solo a partir de recompensas terminales (estilo AlphaZero). Esto evita diseñar la aptitud a mano, pero paga el costo de un problema de entrenamiento mucho más difícil.

## Mínimo Teórico

Una pregunta natural: olvidémonos de algoritmos ingeniosos, ¿cuál es la *menor* cantidad de operaciones que *cualquier* solucionador podría usar en una entrada aleatoria? Para n=500 existe un límite inferior duro que ningún algoritmo puede superar, y proviene de la teoría de la información. Se mantiene no solo en el peor caso, sino para *casi toda* entrada: una permutación aleatoria de 500 elementos se sitúa por encima del límite con probabilidad → 1.

Toda la historia en un gráfico: cada tipo de redundancia que tenemos en cuenta reduce el factor de ramificación `b` y eleva el límite, y la búsqueda exhaustiva luego muestra cuán por encima de él se sitúa el *óptimo real*:

![El escalón del límite: desde el ingenuo 1089 hasta el riguroso 1644, el óptimo real ~2700, y lo que logran los algoritmos reales](docs/img/floor-hierarchy.png)

El resto de esta sección explica de dónde vienen esos números.

**El argumento de conteo.** Las 11 operaciones son *ciegas a los valores*: mueven elementos por posición, sin mirar nunca los valores. Por lo tanto, una secuencia fija de operaciones realiza el mismo mezclado posicional en cualquier entrada, lo que significa que **cada secuencia ordena exactamente una de las `500!` permutaciones posibles**. Para manejar cada entrada, un solucionador necesita `500!` secuencias distintas. Hay `log₂(500!) ≈ 3767 bits` de "¿cuál es esta permutación?" por resolver, y cada operación resuelve como máximo `log₂(b)` bits, donde `b` es el *factor de ramificación efectivo*; por lo tanto, la longitud mínima es `≥ log_b(500!)`.

**El factor de ramificación no es 11.** Ingenuamente, cada paso tiene 11 opciones, pero muchas son redundantes. Dos efectos distintos reducen el número real:

- **Cancelaciones y fusiones** (*reductores de longitud*): `ra` luego `rra` se deshace a sí mismo; `ra` luego `rb` colapsa a `rr`. Una solución más corta nunca contiene estas. Contar solo las secuencias que las evitan da el **crecimiento de palabras**, ω ≈ 7.8.
- **Confluencias de igual longitud**: secuencias distintas de la *misma* longitud pueden alcanzar un estado *idéntico*, por ejemplo `ra rb` y `rb ra`, ya que las dos pilas son independientes. Esto no acorta nada, por lo que las reglas de reducción normal del superoptimizador son ciegas a ellas. Capturarlas requirió extender `src/bin/superopt.rs` para registrar colisiones de igual longitud, no solo reducciones. Incorporándolas obtenemos el **crecimiento de estados**, b* ≈ 4.9; y la brecha ω/b* ≈ 1.6 es el número promedio de caminos más cortos distintos por estado.

**Medición de b*.** El BFS del superoptimizador ya enumera el grafo de estados alcanzables, por lo que simplemente podemos contar los estados *nuevos* alcanzados por primera vez en cada profundidad `d` (el "tamaño de esfera"); la razón entre profundidades consecutivas *es* `b`, y comienza cerca de 11, luego colapsa hacia ~5:

![Razón de tamaño de esfera cayendo desde 6.26 en profundidad 3 hacia 4.79 en profundidad 8, muy por debajo del ingenuo 11](docs/img/branching-bstar.png)

*(Tamaños de esfera medidos con `make superopt N=8`).* La razón aún está descendiendo; la extrapolación sitúa el verdadero b* ≈ 4.4–4.5. Para un número *riguroso*, las ~116,000 patrones prohibidos encontrados hasta longitud 8 (reducciones + colisiones de igual longitud) definen un conjunto restringido de secuencias permitidas cuya tasa de crecimiento es el mayor autovalor de una matriz de transferencia; en profundidad 8 ese autovalor es **4.894**, un límite superior probado para b*. Como verificación de sentido, un modelo construido a partir de esos patrones reproduce los tamaños de esfera anteriores *exactamente*, por lo que no está ajustado artificialmente.

Cada capa de estructura despega la ramificación y eleva el límite `log_b(500!)`; el escalón en el gráfico de la parte superior de esta sección: ingenuo **11 → 1089**, luego solo conmutación A∥B **10.110 → 1129**, más cancelaciones/fusiones **7.823 → 1270**, y más confluencias de igual longitud **4.894 → 1644** (el límite riguroso). El b* extrapolado ≈ 4.45 ajusta esa estimación a ~1750.

La fila de solo conmutación tiene una forma cerrada limpia. Las operaciones de A `{sa, ra, rra}` y las de B `{sb, rb, rrb}` conmutan (tocal pilas independientes), formando un [monoide de trazas](https://en.wikipedia.org/wiki/Trace_monoid) cuya tasa de crecimiento es `1/r`, donde `r` es la raíz más pequeña del *polinomio de clique* `μ(t) = 1 − 11t + 9t²`; dando `(11 + √85) / 2 ≈ 10.11`.

**El límite inferior.** **Ningún solucionador puede ordenar una pila aleatoria de 500 en menos de ~1644 operaciones**; un límite inferior duro y riguroso (extrapolar la tasa de crecimiento de estados da una estimación de conteo similar, ~1750). Un argumento complementario coincide en el orden de magnitud: cada elemento empujado a B debe volver (≥ 2 ops cada uno), y solo una ejecución ya creciente — aproximadamente `2√500 ≈ 45` elementos — puede quedarse en su lugar en A, lo que por sí solo fuerza ≥ ~900 operaciones.

**¿Pero qué tan ajustado es ese límite? La búsqueda exacta lo resuelve.** Un límite de conteo es solo un *límite inferior*; nunca dice qué tan cerca está el *óptimo real*. Para averiguarlo, construí un solucionador exacto (`src/bin/exact_bfs.rs`). El truco: cada operación tiene una inversa de una sola operación (`sa⁻¹ = sa`, `pa⁻¹ = pb`, `ra⁻¹ = rra`, …), por lo que el grafo de configuraciones es **no dirigido**. Una sola búsqueda en anchura desde el estado ordenado, por lo tanto, etiqueta cada una de las `(n+1)·n!` configuraciones con su longitud de solución *exacta* y óptima de una vez; la verdad base real, sin heurísticas. Esto es viable hasta **n = 11** (479 millones de configuraciones; una reimplementación independiente reproduce cada número).

El resultado es estable en todo el rango: la longitud óptima típica sigue `≈ 1.07 · ln(n!)`, una base efectiva *alcanzada* `b_eff = (n!)^{1/L} ≈ 2.55`; muy por debajo del 4.894 del límite de conteo, y apenas se mueve:

![Izquierda: peor caso y óptimo típico frente al límite riguroso para n≤11, un salto de ~1.7×. Derecha: la base alcanzada b_eff ≈ 2.55, plana y muy por debajo del 4.894 del límite](docs/img/exact-tightness.png)

Extrapolar `b_eff ≈ 2.55` a n=500 sitúa el **óptimo típico real cerca de ~2700 operaciones**; por lo tanto, en los tamaños que podemos verificar, el límite riguroso, aunque insuperable, corre aproximadamente **1.6–1.7× suelto** (pero véase la advertencia abajo sobre confiar en ese salto a n=500).

**Por qué el límite no puede ajustarse más.** El límite de conteo mide *información*: cuántos estados distintos existen dentro de `L` pasos, el **volumen** del grafo. Pero una permutación aleatoria se sitúa muy lejos cerca del **diámetro** del grafo, mucho después del radio donde ese volumen ya supera `500!`. Los datos exactos de n=11 hacen la brecha concreta:

![Distribución de longitud óptima sobre todas las entradas 11! se agrupa alrededor de 18–20, mientras que el volumen de conteo ya satura de regreso en la distancia 14](docs/img/volume-vs-diameter.png)

El conteo "se llena" en la distancia 14 (suficientes estados existen para etiquetar cada entrada), pero las entradas mismas promedian ~18.7; esa brecha es pura geometría. No importa cuán cuidadosamente se estime `b*`, el método de conteo alcanza un tope cerca de ~1644; la distancia extra hasta ~2700 es pura geometría, que solo la búsqueda exhaustiva puede medir. Los buenos algoritmos aterrizan en ~4000–5500; ahora solo ~1.5–2× por encima del óptimo real, no el ~2.5× que sugería el límite por sí solo.

**¿Qué tan sólido es el ~2700?** Mucho menos que el límite; es el único número suave aquí. El límite 1644 está *probado* y se mantiene en cada n que medimos; ~2700 es una **extrapolación de 45×** (n=11 → n=500) desde un ajuste fenomenológico, por lo que léelo como un rango aproximado, no una cifra precisa. El defecto principal: `b_eff` no es realmente plano; se desvía *hacia arriba* (2.50 → 2.55 en n=8→11) y su límite es desconocido. Existe una posibilidad real de que siga subiendo hacia el b* del límite de conteo (~4.4–4.9) a medida que los efectos de borde de n pequeños se desvanecen, en cuyo caso el óptimo real se sitúa *por debajo* de 2700; quizás mucho más cerca del límite, haciendo que el límite esté casi ajustado. No tenemos una ley de escalado *derivada*, solo un ajuste sobre una ventana minúscula donde la geometría del problema aún está cambiando, por lo que ni la regla `L ≈ 1.07·ln(n!)` ni la holgura de ~1.7× están garantizadas para sobrevivir a n=500. Rango honesto: el óptimo real de n=500 es al menos el límite probado (~1644) y probablemente bajo ~2900, con ~2700 como mejor suposición que la desviación hacia arriba sugiere que es alta. Fijarlo significa empujar la búsqueda exacta más allá de n=11; viable hasta aproximadamente n=14–16 muestreando instancias aleatorias y encontrándose por la mitad, pero fuera de alcance en 500.

## Cómo compilarlo

El proyecto usa un Makefile para tareas comunes:

```sh
# 1. Generar reglas del optimizador con el superoptimizador.
#    Recomendado: N=5. No pases de 8 (OOM).
make superopt N=5

# 2. Compilar el binario de release (usa las reglas generadas).
make release

# 3. Ejecutarlo.
./push_swap 4 2 7 1 3
```

Otros objetivos del Makefile:

| Objetivo | Descripción |
|--------|-------------|
| `make build` | Compilación de depuración, copia binarios a la raíz del proyecto |
| `make release` | Compilación optimizada de release |
| `make test` | Ejecutar suite de pruebas |
| `make fmt` | Formatear código con rustfmt |
| `make lint` | Ejecutar clippy |
| `make superopt N=5` | Ejecutar superoptimizador hasta profundidad N |
| `make clean-cache` | Restablecer `superopt_cache.json` a vacío |
| `make clean` | Eliminar binarios compilados de la raíz |
| `make fclean` | Limpieza completa (incluyendo `target/`) |
