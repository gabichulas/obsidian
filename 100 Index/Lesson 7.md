15-05-2025 23:37
**Clase**: [[Practical DL for Coders]]
**Temas**: #dl #memoria #gradient-accumulation

**Estado**: #baby 

---

# Gradient Accumulation

Al correr modelos más grandes, podemos tener problemas de VRAM, por lo que muchos pensarían que modelos más grandes -> GPUs más potentes, pero no es necesariamente cierto. 

Una posible solución a esto es el [[Gradient Accumulation]].

El siguiente bloque de código simboliza una *epoch* en un loop de entrenamiento *sin* Gradient Accumulation:

```python
for x,y in dl:
    calc_loss(coeffs, x, y).backward()
    coeffs.data.sub_(coeffs.grad * lr)
    coeffs.grad.zero_()
```

Si agregamos este "truco", quedaría así (asumiendo *batch_length*=64):

```python
count = 0            # track count of items seen since last weight update
for x,y in dl:
    count += len(x)  # update count based on this minibatch size
    calc_loss(coeffs, x, y).backward()
    if count>=64:     # count is greater than accumulation target, so do weight update
        coeffs.data.sub_(coeffs.grad * lr)
        coeffs.grad.zero_()
        count=0      # reset count
```

Es decir, calcula los gradientes con `backward()` sobre los minibatches predefinidos según un índice en el trainer, y al no actualizar los pesos inmediatamente, obtenemos un resultado equivalente (NO siempre, pero particularmente si en convNext) consumiendo mucha menos memoria al operar con batches más pequeños.

Para dimensionar el efecto de Gradient Accumulation, calculamos el uso de memoria del siguiente modelo:

```python
train('convnext_small_in22k', 128, epochs=1, accum=1, finetune=False)
```

![[Pasted image 20250516162107.png]]

El modelo está usando unos 4GB de VRAM sin Gradient Accumulation. Si seteamos `accum=4`:

```python
train('convnext_small_in22k', 128, epochs=1, accum=4, finetune=False)
```

![[Pasted image 20250516162325.png]]

Bajamos el consumo de VRAM a prácticamente la mitad.

PD: Esto ya está implementado y optimizado, no es necesario hacerlo manualmente :)

---

# Multi-target model

Ya vimos cómo desarrollar un modelo que prediga *la* categoría de una imágen, pero nos interesa saber si podemos crear uno capaz de predecir más de una categoría.

Tomaremos como ejemplo el dataset de una [competencia](https://www.kaggle.com/competitions/paddy-disease-classification) de **Kaggle**. Los datos se ven así:

| image_id     | label                     | variety | age |
|:------------:|:-------------------------:|:-------:|:---:|
| 100330.jpg   | bacterial_leaf_blight     | ADT45   | 45  |
| 100365.jpg   | bacterial_leaf_blight     | ADT45   | 45  |
| 100382.jpg   | bacterial_leaf_blight     | ADT45   | 45  |
| 100632.jpg   | bacterial_leaf_blight     | ADT45   | 45  |
| 101918.jpg   | bacterial_leaf_blight     | ADT45   | 45  |

Suponiendo que ya creamos las funciones necesarias e importamos los datos:

```python
dls = DataBlock(
    blocks=(ImageBlock,CategoryBlock,CategoryBlock),
    n_inp=1,
    get_items=get_image_files,
    get_y = [parent_label,get_variety],
    splitter=RandomSplitter(0.2, seed=42),
    item_tfms=Resize(192, method='squish'),
    batch_tfms=aug_transforms(size=128, min_scale=0.75)
).dataloaders(trn_path)
```

Donde:

```python
n_inp=1,
```

indica el número de inputs que tendrá el modelo, es decir, el input es `ImageBlock`;

```python
get_y = [parent_label,get_variety],
```

ahora *y* no es una columna, ya que tenemos dos objetivos. `parent_label` es una función de fastai que devuelve el nombre de la carpeta que contiene a la imágen, es decir, la categoría (en este caso, la enfermedad de la planta). `get_variety` devuelve la especie de la planta. 


Luego, visualizamos un batch:

```python
dls.show_batch(max_n=6)
```

![[Pasted image 20250516175240.png]]


## Modelo típico

Creamos el learner para predecir **solo** la enfermedad:

```python
arch = 'convnext_small_in22k'
learn = vision_learner(dls, arch, loss_func=disease_loss, metrics=disease_err, n_out=10).to_fp16()
lr = 0.01 # learning rate
```

Notamos dos funciones desconocidas: `disease_loss` y `disease_err`. Las definiciones son las siguientes:

```python
def disease_err(inp,disease,variety): return error_rate(inp,disease)
def disease_loss(inp,disease,variety): return F.cross_entropy(inp,disease)
```

La creación de estas funciones se debe a que ahora, en vez de dos inputs, tenemos tres. Ambas funciones son equivalentes, solo que seleccionando solo uno de los targets. Ver [[Cross Entropy]].

Notar el parámetro `n_out=10`. Este indica el órden de la matriz resultado, o, equivalentemente, la cantidad de enfermedades únicas que hay.


## Modelo multi-objetivo

Para transformar lo que hicimos en un [[#Multi-target model]], hacemos lo siguiente:

