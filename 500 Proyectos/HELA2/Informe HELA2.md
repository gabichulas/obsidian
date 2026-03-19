# Optimización Logística para Distribución de Helados - Hela2
---
Código de proyecto: HELA2
---

## Índice
- [Introducción](#introducción)
- [Marco Teórico](#marco-teórico)
  - [Investigación Operativa y Logística](#investigación-operativa-y-logística)
  - [Problema de Ruteo de Vehículos (VRP)](#problema-de-ruteo-de-vehículos-vrp)
  - [Algoritmos de Optimización](#algoritmos-de-optimización)
  - [Tecnologías Utilizadas](#tecnologías-utilizadas)
- [Análisis del Problema](#análisis-del-problema)
  - [Descripción de la Red de Distribución](#descripción-de-la-red-de-distribución)
  - [Restricciones del Dominio (Cadena de Frío)](#restricciones-del-dominio)
- [Diseño Experimental e Implementación](#diseño-experimental-e-implementación)
  - [Arquitectura del Sistema](#arquitectura-del-sistema)
  - [Modelado de Datos](#modelado-de-datos)
  - [Implementación del Algoritmo](#implementación-del-algoritmo)
- [Pruebas y Resultados](#pruebas-y-resultados)
- [Conclusiones](#conclusiones)
- [Bibliografía](#bibliografía)

## Introducción
Presentá el contexto de Hela2. Explicá la problemática de las heladerías (costos de combustible, tiempos de entrega, sensibilidad de la temperatura) y cómo el software busca optimizar estos procesos.

## Marco Teórico

### Investigación Operativa y Logística
Definí los conceptos base de la optimización aplicados a la logística. Podés citar fuentes sobre gestión de cadena de suministro.

### Problema de Ruteo de Vehículos (VRP)
Si tu proyecto optimiza caminos, explicá qué es el VRP. Si es más de inventario, hablá de modelos de stock.

### Tecnologías Utilizadas
Justificá el uso de tu stack. Por ejemplo, si usás **FastAPI** para el backend por su alta performance y tipado.

## Análisis del Problema
Detallá los desafíos específicos de Hela2:
- Puntos de venta (nodos).
- Capacidad de los vehículos.
- Ventanas horarias de entrega.

## Diseño Experimental e Implementación

### Arquitectura del Sistema
Describí cómo se comunican los componentes (ej. Backend en Python, Base de Datos, Frontend). Podés mencionar el uso de **Docker** si lo aplicaste para el despliegue.

### Modelado de Datos
Explicá cómo representaste el mundo real en código (objetos Pedido, Camión, Ruta, etc.).

### Implementación del Algoritmo
Explicá la lógica core. ¿Es una heurística? ¿Usaste librerías de optimización? Al igual que en el ejemplo de tu compañero, documentá las funciones principales y su complejidad.

## Pruebas y Resultados
Mostrá métricas:
- Reducción de kilómetros recorridos.
- Tiempo de ejecución del algoritmo.
- Comparativa entre una ruta manual y una optimizada por Hela2.

## Conclusiones
Resumí si se cumplieron los objetivos y qué mejoras podrías añadir (ej. predicción de demanda con IA en el futuro).

## Bibliografía
No olvides citar libros de algoritmos y documentación oficial de las herramientas usadas (FastAPI, Python, etc.).