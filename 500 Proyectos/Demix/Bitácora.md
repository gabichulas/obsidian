
### 12/3

Hasta el momento, logré establecer la línea base del proyecto Demix. Descargué y organicé el dataset completo de aproximadamente 100 GB en local. A nivel de código, ya implementé la arquitectura de la U-Net en TensorFlow/Keras, diseñada para separar los 4 stems (drums, bass, other, vocals) estimando máscaras sobre los espectrogramas. 

También dejé configurado el pipeline de entrenamiento inicial, incluyendo el guardado de checkpoints y el cálculo de métricas de error (MSE y MAE). Realicé las primeras pruebas de entrenamiento en mi 1050 Ti, completando ciclos de hasta 20 horas continuas, lamentablemente sin avances considerables. 

Estas ejecuciones iniciales demostraron que la convergencia es mínima y confirmaron que la restricción de hardware (4 GB de VRAM) imposibilita iterar sobre el dataset completo. Este cuello de botella metodológico es lo que motivó el cambio de estrategia actual: pausar el entrenamiento local por fuerza bruta y migrar hacia un diseño experimental basado en un subset estratificado en Kaggle, lo que garantizará un entorno reproducible y justo para la inminente comparación contra el Vision Transformer.