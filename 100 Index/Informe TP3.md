**Integrantes**: Azurduy Giovanni, Lopez Romero Gabriel


## Características del cluster

El cluster utilizado en la facultad tenía, al momento de la entrega del TP, seis máquinas con cuatro núcleos cada uno, osea, veinticuatro en total.

---
## Ejercicios

A continuación, se mostrarán los ejercicios corridos sobre el cluster con su respectivo speedup.


### Ejercicio 1: Test

![[WhatsApp Image 2025-11-11 at 17.06.42.jpeg]]


### Ejercicio 2: Cálculo de Logaritmo Natural por Serie de Taylor

![[Pasted image 20251112170232.png]]

Utilizaremos, a modo de comparación, los tiempos obtenidos en el TP1.

| Implementación      | Resultado          | Tiempo (ms) | Speedup |
| :------------------ | :----------------- | :---------- | :------ |
| Secuencial          | 14.731801283197943 | 0.020245    | 1       |
| Paralelo (4 hilos)  | 14.731801283197943 | 0.011851    | 1.7     |
| Paralelo (8 hilos)  | 14.731801283197943 | 0.006011    | 3.37    |
| Paralelo (12 hilos) | 14.731801283197943 | 0.004480    | 4.52    |
| Paralelo (20 hilos) | 14.731801283197943 | 0.005155    | 3.92    |
| Cluster             | 14.64549295614092  | 0.00273     | 7.41    |


### Ejercicio 3: Búsqueda de patrones

![[WhatsApp Image 2025-11-11 at 17.06.44.jpeg]]

| Implementación      | Tiempo (s) | Speedup |
| :------------------ | :--------- | :------ |
| Secuencial          | 4.30556    | 1       |
| Paralelo (12 hilos) | 0.971355   | 4.43    |
| Paralelo (32 hilos) | 0.887331   | 4.85    |
| Cluster             | 0.588957   | 7.31    |


### Ejercicio 4: Multiplicación de matrices

