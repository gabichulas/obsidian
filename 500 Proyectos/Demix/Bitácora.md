
### 12/3

Hasta el momento, logré establecer la línea base del proyecto Demix. Descargué y organicé el dataset completo de aproximadamente 100 GB en local. A nivel de código, ya implementé la arquitectura de la U-Net en TensorFlow/Keras, diseñada para separar los 4 stems (drums, bass, other, vocals) estimando máscaras sobre los espectrogramas. 

También dejé configurado el pipeline de entrenamiento inicial, incluyendo el guardado de checkpoints y el cálculo de métricas de error (MSE y MAE). Realicé las primeras pruebas de entrenamiento en mi 1050 Ti, completando ciclos de hasta 20 horas continuas, lamentablemente sin avances considerables. 

Estas ejecuciones iniciales demostraron que la convergencia es mínima y confirmaron que la restricción de hardware (4 GB de VRAM) imposibilita iterar sobre el dataset completo. Este cuello de botella metodológico es lo que motivó el cambio de estrategia actual: pausar el entrenamiento local por fuerza bruta y migrar hacia un diseño experimental basado en un subset estratificado en Kaggle, lo que garantizará un entorno reproducible y justo para la inminente comparación contra el Vision Transformer.


### 18/3

Dividí el set de train 80/20 para crear un set de validación a partir de este. Creé un script de segmentacion porcentual del dataset para facilitar la subida y los benchmarks en Kaggle. Hice una primera iteración de esto creando un dataset con el 20% del original y lo subí a Kaggle.

En una primer experimento, entrené una U-Net idéntica a la existente con los siguientes filtros iniciales: [8, 16, 32, 64]. El entrenamiento, de 10 epochs, duró 41 minutos en total con la GPU T4, es decir, unos 4 minutos por epoch. IMPORTANTE: el entrenamiento fue **mucho** más fructífero que el desarrollado localmente aún con menos epochs. val_loss pasó de 0.18343 a 0.14758, cuando localmente, con el dataset completo y unas 40 epochs (20 horas) se había quedado en aprox. 0.18.

El mejor modelo quedó guardado, pero voy a analizar qué hacer en cuanto al tamaño del dataset. Quizás está overfitteando.


### 28/3

Después de batallar con Kaggle porque no seguía el entrenamiento a partir del checkpoint, lo logré solucionar y el modelo va por buen camino. Lo que estaba pasando era que el optimizador debía ser inicializado previamente, ya que al cargar el chackpoint sin hacerlo se producía un error de diferencia de parámetros entre el optimizador nuevo y el del checkpoint. Al no matchear, el modelo empezaba a entrenar de cero.

Otro cambio que hice fue cambiar los filtros. Ahora estoy usando [8, 16, 32, 64].

Luego de 105 epochs y unas 10 horas de entrenamiento, los numeros son los siguientes:

![[Pasted image 20260328173001.png]]

Creo que sería bueno hacer una prueba para ver como funciona porque no mejoró demasiado en las ultimas 50 epochs (de 0.126 a 0.121).

### 4/4

Hoy hice una pruebita con el modelo de arriba y funciona sorprendentemente bien. Está verde así que lo voy a seguir entrenando, pero anda bien.

Hice un test con "Fade Into You" y lo que mejor separó con diferencia fue la voz. El bajo va bastante encaminado. Le cuesta bastante separar la batería. El stem de "other" elimina casi por completo la batería.

Todavía no testeé métricas, pero a simple vista va bien.