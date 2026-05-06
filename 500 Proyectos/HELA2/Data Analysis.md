**Por Escenario y Peso**

- E1 small, length: mejor configuración global en calidad es `length_alpha0.8`, seguida por `length_ants40` y `length_gamma0.7`.
- E1 small, street time: gana `street_time_alpha0.8`, seguida por `street_time_gamma1.3` y `street_time_beta2.5`.
- E2 medium, length: vuelve a dominar `length_alpha0.8`; luego `length_rho0.7` y `length_iters120`.
- E2 medium, street time: gana `street_time_alpha0.8`, luego `street_time_rho0.3` y `street_time_beta2.5`.

**Gráficos**

- En boxplot_objective_cost_scenario_E2_medium_weight_street_time.png, `street_time_alpha0.8` concentra la mediana más baja (826.73) y `alpha1.2` la desplaza hacia arriba (883.69).
- En boxplot_penalty_total_scenario_E1_small_weight_length.png, `alpha0.8` (mediana 100.08, IQR 39.69) y `beta2.5` (mediana 104.88, IQR 28.06) quedan en zona competitiva por costo y dispersión.
- En boxplot_penalty_total_scenario_E2_medium_weight_street_time.png, `beta2.5` tiene buen compromiso (mediana 101.26, IQR 29.01) frente a alternativas con medianas mayores o IQR más altos.
- En boxplot_distance_total_scenario_E2_medium_weight_street_time.png, `alpha0.8` marca la menor mediana de distancia (739.05); `ants40` queda más alto (765.65).
- En boxplot_distance_total_scenario_E1_small_weight_length.png, `alpha0.8` también está entre las distancias más bajas (5194.12), coherente con su mejor objetivo total.
- En boxplot_penalty_total_scenario_E2_medium_weight_length.png, no se destaca un outlier extremo; `rho0.3` queda en el medio (mediana 109.64, IQR 59.29).

**Tradeoff**

- Más rápidos: `street_time_iters50`, `street_time_ants20`, `length_iters50`, `length_ants20`.
- Mejores en puntualidad mediana: los configs de E2 se mantienen en 85.71% en casi todos los casos; no hay un ganador claro por puntualidad.
- Pero los más rápidos no son los mejores en costo objetivo. Hay tradeoff real y visible en boxplot_exec_s_all_configs.png y boxplot_pct_on_time_all_configs.png.

**Sensibilidad De Parámetros**

- alpha tiene correlación positiva con costo en los 4 grupos analizados: subir alpha empeora.
- n ants y n iters suelen tener correlación levemente negativa: subirlos ayuda un poco, pero con rendimiento decreciente.
- beta ayuda más en street time que en length.
- gamma y rho tienen efecto menor y menos estable.

**Resumen**

1. Calidad: alpha 0.8 como base en ambos pesos.
2. Tiempo de cómputo: iters50 o ants20, aceptando pérdida de calidad.
3. Balance robusto en street time: beta2.5 o alpha0.8.
4. Balance robusto en length: alpha0.8, con gamma0.7 como alternativa.