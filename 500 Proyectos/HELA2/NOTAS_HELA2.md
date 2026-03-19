# Informe técnico inicial del algoritmo de ruteo

## Marco de cumplimiento respecto del anteproyecto
El anteproyecto planteó una línea multiobjetivo como alternativa deseable y, en paralelo, estableció ACO como vía de implementación válida si no se cerraba un enfoque multiobjetivo formal. Bajo ese criterio, el estado actual del proyecto es consistente con lo comprometido: el núcleo implementado es ACO y cubre las restricciones operativas principales definidas al inicio.

Para evitar ambigüedades en la defensa, este informe distingue entre alcance comprometido y alcance deseable. Cuando se menciona una brecha, debe leerse como extensión pendiente o mejora futura, no como incumplimiento del objetivo principal.

## Estado del proyecto y alcance del documento
Este documento resume el estado actual de la implementación de ruteo en la rama de trabajo vigente, con foco en el algoritmo ACO y en los parámetros operativos que afectan el resultado. Está escrito a partir del código que hoy ejecuta la aplicación, no como una descripción teórica idealizada.

El sistema toma una zona (por ciudad o coordenadas), descarga o reutiliza el grafo vial de OpenStreetMap, detecta puntos de interés con amenity=ice_cream y calcula un tour que parte de un centro de distribución, visita todas las heladerías alcanzables y vuelve al origen. La capa de API está en FastAPI y la visualización se resuelve en la plantilla web con JavaScript nativo.

## Arquitectura funcional
La entrada de la aplicación se expone en src/main.py y delega en src/api/routes.py. Allí conviven los endpoints de grafo, consulta de heladerías, gestión de centros y ejecución del ACO.

La generación y preparación de grafos está en src/core/graph.py. Este módulo incorpora una etapa explícita de normalización de street_time en cada arista para evitar inconsistencias de cache. El descubrimiento de heladerías se implementa en src/core/osm.py. El núcleo de optimización está en src/core/algorithms.py.

La persistencia de centros de distribución usa SQLModel con SQLite en data/centros.db. El modelo se define en src/models/database.py.

## Flujo completo del cálculo ACO
El endpoint POST /aco-heladerias en src/api/routes.py ejecuta este pipeline:

1. Construye el grafo vial según modo city o coords.
2. Proyecta el grafo y mapea el origen al nodo más cercano.
3. Consulta heladerías en el radio solicitado.
4. Mapea cada heladería a su nodo más cercano.
5. Filtra destinos para quedarse solo con nodos de la componente conectada del origen.
6. Si corresponde, genera ventanas temporales aleatorias por destino.
7. Invoca aco_tour_through_nodes con los parámetros de optimización.
8. Arma salida enriquecida: ruta completa, matriz par a par, tramos del tour, métricas y log de llegadas.

## Definición del problema que resuelve la implementación actual
La implementación trabaja sobre una variante de TSP con extensiones operativas:

- Ventanas de tiempo blandas, con penalización por llegada temprana o tardía.
- Capacidad agregada del vehículo, modelada como penalización global por exceso de demanda total.
- Tiempo máximo de operación, modelado como penalización por excedente.

No hay, en la versión actual, múltiples vehículos ni una estrategia de reparto por rutas independientes. El tour resultante es único y cerrado (sale y vuelve al origen).

## Alcance comprometido y alcance deseable
En términos de trazabilidad con el anteproyecto, la implementación vigente cumple el alcance comprometido para la línea ACO: minimización de distancia con tratamiento de restricciones de ventanas, capacidad y tiempo de operación mediante penalizaciones. También dispone de un flujo completo de datos geográficos reales (red vial y puntos de interés) y una interfaz para ejecutar y analizar corridas.

Quedan como alcance deseable y no obligatorio de esta etapa: una formulación multiobjetivo estricta con frente de Pareto, métricas específicas de calidad multiobjetivo como hypervolume, tráfico en tiempo real, estrategia de múltiples vehículos y reabastecimiento operativo en depósitos intermedios.

## Núcleo del algoritmo (src/core/algorithms.py)
La función principal es aco_tour_through_nodes. El método construye primero una matriz completa de caminos mínimos entre nodos relevantes (origen + destinos) usando Dijkstra con el peso seleccionado.

Con esa matriz precalculada, cada hormiga construye una secuencia visitando nodos no visitados mediante una regla probabilística dependiente de feromona, heurística por distancia y un factor de urgencia temporal. La mejor solución por iteración deposita feromona y luego se conserva el mejor histórico global.

### Función objetivo
La función objetivo combina costo de distancia y penalizaciones:

objective_cost = lambda_weight * distance_cost + mu_weight * penalty_total

con

penalty_total = penalty_alpha * penalty_window + penalty_beta * penalty_capacity + penalty_gamma * penalty_time

El valor que retorna como cost corresponde al objective_cost, no a la distancia pura.

### Modelo de tiempo de viaje
El tiempo operativo no es idéntico al peso usado en Dijkstra. En la construcción del tour se estima tiempo de viaje en minutos así:

- Si weight = street_time, se interpreta distance como segundos y se convierte a minutos.
- Si weight = length, se interpreta distance como metros y se convierte a minutos con una velocidad urbana configurable.
- Sobre ambos casos se aplica un factor multiplicativo operativo y un piso mínimo por tramo.

Parámetros actuales del modelo temporal dentro del algoritmo:

- time_factor = 5.0
- min_travel_minutes = 1.5
- length_speed_kph = 18.0

Esto desacopla la métrica de optimización (el peso de camino mínimo) del reloj operativo usado para ventanas y tiempos de servicio.

### Ventanas de tiempo y estado de llegada
Para cada destino, el sistema calcula arrival_time y evalúa:

- TEMPRANO, si llega antes de earliest.
- A TIEMPO, si cae dentro de [earliest, latest].
- TARDÍO, si supera latest.

Además registra wait_minutes, service_start_minutes y depart_minutes. En API se formatea a HH:MM:SS para evitar pérdidas de lectura en tramos cortos.

## Parámetros expuestos por el endpoint de ACO
La API recibe los siguientes parámetros de optimización en src/api/routes.py:

| Parámetro | Tipo | Default | Uso principal |
|---|---:|---:|---|
| weight | str | length | Peso para Dijkstra: length o street_time |
| n_ants | int | 30 | Cantidad de hormigas por iteración |
| n_iters | int | 80 | Número de iteraciones |
| alpha | float | 1.0 | Exponente de feromona |
| beta | float | 2.0 | Exponente heurístico |
| gamma | float | 0.7 | Exponente del factor de urgencia |
| rho | float | 0.5 | Tasa de evaporación |
| q | float | 100.0 | Intensidad de depósito |
| start_time | float | 0 | Minutos desde medianoche |
| unload_time | int | 10 | Tiempo de descarga por visita (min) |
| generate_windows | str | false | Genera ventanas aleatorias |
| capacity | float? | null | Capacidad total del vehículo |
| default_demand | float | 1.0 | Demanda por heladería |
| max_operation_time | float? | null | Límite de tiempo operativo total |
| penalty_alpha | float | 1.0 | Peso de penalización por ventanas |
| penalty_beta | float | 1.0 | Peso de penalización por capacidad |
| penalty_gamma | float | 1.0 | Peso de penalización por tiempo total |
| lambda_weight | float | 1.0 | Peso del término distancia |
| mu_weight | float | 1.0 | Peso del término penalizaciones |

## Construcción del peso street_time en el grafo
En src/core/graph.py el atributo street_time por arista se calcula como:

street_time = length / (maxspeed / 3.6)

Aquí length está en metros y maxspeed en km/h, por lo que street_time queda en segundos. El código aplica esta normalización tanto en grafos recién descargados como en grafos cargados de cache, para evitar valores heredados inconsistentes.

## Salidas relevantes para análisis experimental
La respuesta de /aco-heladerias devuelve:

- order y order_labels para secuencia de visita.
- path con nodos del trazado completo.
- pairwise con matriz de distancias entre nodos relevantes.
- tour_legs con segmentos del tour.
- arrival_log con horas de llegada, inicio de servicio y salida.
- metrics con descomposición del objetivo: distancia, penalizaciones, tiempo total y parámetros de pesos.

Esto permite construir análisis comparativos entre configuraciones sin reformatear el backend.

## Observaciones técnicas del estado actual
Para el informe, el núcleo matemático y operativo está concentrado en dos archivos: src/core/algorithms.py y src/core/graph.py.

En este contexto, la ausencia de MOACO estricto no invalida la solución desarrollada, porque el anteproyecto explicitó ACO como alternativa válida. La diferencia actual debe presentarse como decisión de alcance y priorización técnica, no como desalineación del proyecto.
