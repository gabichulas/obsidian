**Por Escenario y Peso**

- E1 small, length: mejor configuración global en calidad es length_alpha0.8, seguida por length_ants40 y length_beta2.5.
- E1 small, street time: ganan street_time_iters120 y street_time_beta2.5 en costo objetivo; street_time_iters50 corre mucho más rápido pero pierde calidad.
- E2 medium, length: vuelve a dominar length_alpha0.8; luego length_beta2.5 y length_base. En tamaño medio, el beneficio de alpha bajo es más fuerte.
- E2 medium, street time: gana street_time_alpha0.8, luego street_time_ants40 y street_time_beta2.5.



- En boxplot_objective_cost_scenario_E2_medium_weight_street_time.png, street_time_alpha0.8 concentra mediana baja; alpha1.2 desplaza hacia arriba con mayor costo.
- En boxplot_penalty_total_scenario_E1_small_weight_length.png, hay bastante solapamiento, pero alpha0.8 y beta2.5 quedan en zona competitiva sin colas extremas sistemáticas.
- En boxplot_penalty_total_scenario_E2_medium_weight_street_time.png, beta2.5 tiende a buen compromiso entre mediana y dispersión.
- En boxplot_distance_total_scenario_E2_medium_weight_street_time.png, alpha0.8 y ants40 quedan entre los más bajos en distancia, coherente con su buen objetivo.
- En boxplot_distance_total_scenario_E1_small_weight_length.png, alpha0.8 no minimiza siempre distancia pura, pero sí mejora objetivo total por balance con penalización.
- En boxplot_penalty_total_scenario_E2_medium_weight_length.png, length_rho0.3 muestra cola alta de penalización; no parece opción robusta.

**Tradeoff**

- Más rápidos: street_time_iters50, street_time_ants20, length_iters50, length_ants20.
- Mejores en puntualidad mediana: varios configs empatados arriba (especialmente beta2.5 y varios street time).
- Pero los más rápidos no son los mejores en costo objetivo. Hay tradeoff real y visible en boxplot_exec_s_all_configs.png y boxplot_pct_on_time_all_configs.png.

**Sensibilidad De Parámetros**

- alpha tiene correlación positiva con costo en los 4 grupos analizados: subir alpha empeora.
- n ants y n iters suelen tener correlación levemente negativa: subirlos ayuda un poco, pero con rendimiento decreciente.
- beta ayuda más en street time que en length.
- gamma y rho tienen efecto menor y menos estable.

**Resumen**

1. Si priorizas calidad: usa alpha 0.8 como base en ambos pesos.
2. Si priorizas tiempo de cómputo: usa iters50 o ants20, aceptando pérdida de calidad.
3. Si quieres balance robusto en street time: beta2.5 o alpha0.8.
4. Si quieres balance robusto en length: alpha0.8, con beta2.5 como alternativa.