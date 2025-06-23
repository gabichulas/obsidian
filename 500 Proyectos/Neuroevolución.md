## Objetivo del Proyecto

Reemplazar el componente de *Reinforcement Learning* (Q-Learning) en un autoscaler para workflows científicos en la nube, utilizando **Deep Learning** mediante **neuroevolución de una red neuronal que actúa como política**.

---

## 🧬 ¿Qué es una política?

> Una **política** es una función que, dado un estado `s`, devuelve una acción `a`.

En RL, la política define el comportamiento del agente en cada paso.

---

## 🧠 ¿Cómo una red neuronal representa una política?

- Entrada: vector de estado (en tu caso, 4 variables categóricas o discretizadas).
- Salida: vector de scores (uno por cada acción posible).
- La acción elegida es la de mayor score (`argmax`).

Ejemplo:

```text
Estado: [join=Low, fork=High, pipe=Mid, carga=Low]
→ Red neuronal → [0.2, 1.7, -0.1]
→ Acción elegida = Scaling_Type2
```

---

## 🧩 Enfoque actual vs tu propuesta

| Aspecto                  | RL tradicional (Q-Learning)      | Tu enfoque (Neuroevolución)              |
|--------------------------|----------------------------------|------------------------------------------|
| Representación de política | Q-table                         | Red neuronal                             |
| Aprendizaje              | Bootstrapping + TD learning      | Algoritmo evolutivo offline              |
| Entrenamiento            | Online, durante interacción      | Offline, basado en simulaciones          |
| Señal de entrenamiento   | Recompensa inmediata y acumulada | Makespan + Costo                         |
| Adaptabilidad online     | ✅                                | ❌ (pero posible híbrido en el futuro)    |
| Generalización           | limitada                         | mejor si red bien diseñada               |

---

## 📌 Flujo del algoritmo

```text
1. Inicializar población de vectores de pesos
2. Para cada individuo:
   - Construir red neuronal
   - Ejecutar simulación con esa red como política
   - Medir makespan y costo
   - Asignar fitness multiobjetivo
3. Aplicar selección, crossover, mutación
4. Repetir por N generaciones
5. Seleccionar la mejor política final (mínimo AggMC)
```

---

## 📚 Conceptos clave

- **Neuroevolución**: entrenamiento de redes neuronales mediante algoritmos evolutivos.
- **Política**: función que define el comportamiento del agente (π(s) = a).
- **AggMC**: norma L2 de (makespan, cost), utilizada como métrica de desempeño final.
- **Red neuronal como política**: input → hidden layers → scores por acción → acción elegida.

---

## 🧠 Posibles extensiones

- Evolución de arquitectura (no solo pesos).
- Integración con RL (política evolucionada como inicialización).
- Políticas específicas por tipo de workflow.
- Transferencia entre dominios de ejecución.

---