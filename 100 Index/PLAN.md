

**Objetivo:** Comparar arquitecturas bajo el mismo presupuesto computacional (30 horas semanales en Kaggle) y metodológico, sorteando el trade-off entre el límite de 200 GB del dataset y el tiempo de convergencia.

## Fase 1: Calibración Empírica del Tiempo (Semana 1)
1. **El Micro-Dataset:** Subir solo de 2 GB a 5 GB de archivos `.h5` a Kaggle como prueba inicial.
2. **Benchmark de Epoch:** Correr 2 o 3 *epochs* reales con el Vision Transformer en la GPU P100.
3. **La Ecuación del Presupuesto:** Medir cuánto tarda una *epoch*. Calcular el tamaño máximo del dataset (ej. 40 GB) para que una *epoch* dure entre 30 y 45 minutos. Esto asegura entre 15 y 20 *epochs* en un *run* de 10 horas.
4. **Generación del Subset Final:** Crear el subset calibrado asegurando la distribución representativa de géneros musicales y subirlo a Kaggle como el dataset definitivo.

## Fase 2: Desacoplar "Epochs" del Tamaño del Dataset
Abandonar la definición clásica de *epoch* (ver todos los datos una vez). Forzar validaciones y guardado de *checkpoints* frecuentes usando `steps_per_epoch` fijos para no perder progreso si la sesión se corta abruptamente.

```python
import math
from tensorflow.keras.callbacks import ModelCheckpoint, ReduceLROnPlateau, EarlyStopping

def train_demix_models(model, model_name, train_dataset, val_dataset, total_train_samples, batch_size):
    # Define an "epoch" as processing a fixed number of samples
    steps_per_epoch = 10000 
    val_steps = 500 

    checkpoint_path = f"/kaggle/working/{model_name}_weights.h5"
    
    callbacks_list = [
        ModelCheckpoint(
            filepath=checkpoint_path, 
            save_best_only=True, 
            monitor='val_loss',
            mode='min'
        ),
        ReduceLROnPlateau(
            monitor='val_loss', 
            factor=0.5, 
            patience=4, 
            min_lr=1e-6
        ),
        EarlyStopping(
            monitor='val_loss', 
            patience=10, 
            restore_best_weights=True
        )
    ]
    
    # train_dataset must be set to .repeat() so it doesn't exhaust
    history = model.fit(
        train_dataset,
        epochs=150, # Let Early Stopping halt it
        steps_per_epoch=steps_per_epoch,
        validation_data=val_dataset,
        validation_steps=val_steps,
        callbacks=callbacks_list
    )
    
    return history
```

## Fase 3: Estrategia de Ejecución Semanal (Cuota de 30 Horas)
* **Viernes (10 horas):** Lanzar el entrenamiento de la U-Net en segundo plano ("Save & Run All"). Configurar el código para que se detenga automáticamente antes de las 10 horas.
* **Sábado (12 horas):** Lanzar el Vision Transformer. Asignar el tiempo máximo de sesión permitido porque los ViTs requieren más tiempo de convergencia inicial (warm-up).
* **Domingo (8 horas de reserva):** Inferencia estricta. Cargar los mejores pesos `.h5` de ambos modelos y calcular las métricas finales (MSE, MAE, SI-SDR) en el conjunto de test.

## Fase 4: Justificación Académica para el Informe
* **Restricción como Parámetro:** Documentar que se evaluó empíricamente el *trade-off* entre volumen de datos y tasa de actualización de gradientes para ajustarse al hardware disponible.
* **Metodología de Epochs Fijas:** Explicar el desacople del tamaño del dataset usando `steps_per_epoch` constantes para garantizar una evaluación de alta frecuencia y la captura de pesos óptimos.
* **Comparación Justa:** Destacar que bajo un presupuesto temporal estrictamente idéntico y cantidad de parámetros equivalentes, se analizó qué arquitectura minimiza la función de pérdida y maximiza el SI-SDR de las cuatro fuentes separadas (drums, bass, other, vocals).