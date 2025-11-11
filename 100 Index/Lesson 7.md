15-05-2025 23:37
**Clase**: [[Practical DL for Coders]]
**Temas**: #dl #gradient-accumulation #multi-target-model #collaborative-filtering #l2

**Estado**: #child

---

# Índice

- [Gradient Accumulation](#Gradient%20Accumulation)
- [Multi-target model](#Multi-target%20model)
	- [Modelo típico](#Modelo%20t%C3%ADpico)
	- [Modelo multi-objetivo](#Modelo%20multi-objetivo)
	- [Conclusiones](#Conclusiones)
- [Collaborative Filtering](#Collaborative%20Filtering)
	- [DataLoaders](#DataLoaders)
	- [From Scratch](#From%20Scratch)
	- [L2 Regularization](#L2%20Regularization)

# Gradient Accumulation

[Notebook](https://www.kaggle.com/code/jhoward/scaling-up-road-to-the-top-part-3)

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

[Notebook](https://www.kaggle.com/code/jhoward/multi-target-road-to-the-top-part-4)

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

```python
learn = vision_learner(dls, arch, n_out=20).to_fp16()
```

Vemos que ahora `n_out` es 20. Esto se debe a que ya no solo queremos predecir las 10 enfermedades, sino tambien los 10 tipos de planta.

Luego, redefinimos las funciones de loss:

```python
def disease_loss(inp,disease,variety): return F.cross_entropy(inp[:,:10],disease)
```

```python
def variety_loss(inp,disease,variety): return F.cross_entropy(inp[:,10:],variety)
```

Ver [[Manipulación de listas en Python]].

Luego, combinamos ambas funciones en una sola:

```python
def combine_loss(inp,disease,variety): return disease_loss(inp,disease,variety) + variety_loss(inp,disease,variety)
```

Seria bueno ver los error_rate de cada output por separado, por lo que:

```python
def disease_err(inp,disease,variety): return error_rate(inp[:,:10],disease)
def variety_err(inp,disease,variety): return error_rate(inp[:,10:],variety)

err_metrics = (disease_err,variety_err)
```

Nos gustaria tambien ver las metricas por separado:

```python
all_metrics = err_metrics+(disease_loss,variety_loss) 
# (disease_err,variety_err,disease_loss,variety_loss)
```

Finalmente, creamos el learner:

```python
learn = vision_learner(dls, arch, loss_func=combine_loss, metrics=all_metrics, n_out=20).to_fp16()

learn.fine_tune(5, lr)
```

![[Pasted image 20250519155433.png]]

## Conclusiones

Esta es una técnica muy útil para estas situaciones, pero tambien tiene una característica muy especial. Irónicamente, en algunos casos, puede mejorar la precisión de features individuales con respecto a modelos convencionales que estudiarían únicamente esa feature. Esto se debe a que, al analizar no solo esa, el modelo puede ser capaz de encontrar patrones que serían imposibles de encontrar con un modelo single-target. Por ejemplo, "la enfermedad *x* es más propensa a desarrollarse en la planta *y*".

---

# Collaborative Filtering

[Notebook](https://www.kaggle.com/code/jhoward/collaborative-filtering-deep-dive/notebook)

Supongamos que tenemos un dataset que describe los ratings dados por ciertos usuarios a ciertas películas. Se vería algo así:

```python
ratings.head()
```

| user | movie | rating | timestamp   |
|------|-------|--------|-------------|
| 196  | 242   | 3      | 881250949   |
| 186  | 302   | 3      | 891717742   |
| 22   | 377   | 1      | 878887116   |
| 244  | 51    | 2      | 880606923   |
| 166  | 346   | 1      | 886397596   |

Podemos organizar esta información de forma más legible e intuitiva de la siguiente forma:

![[Untitled.png]]

Podemos observar que hay ciertos valores que no existen (el usuario no vio la película), los cuales son los que queremos predecir.

Para buscar una forma de "rellenar" estos vacíos, debemos empezar buscando una forma de procesar esta matriz para que nuestro modelo pueda entender el input que le demos. Una forma de hacer esto es la siguiente:

Si supieramos qué tan importante es un aspecto de una película (género, año, director, estilo) podríamos crear un vector de valores entre -1 y 1 indicando la importancia de cada uno de estos valores. 

```python
force_awakens = np.array([0.98,0.9,-0.9])
```

En este caso, por ejemplo, [The Force Awakens](https://www.imdb.com/es/title/tt2488496/?ref_=ext_shr_lnk) tiene un puntaje de 0.98 en "Ciencia Ficción", 0.9 en "Acción" y -0.9 en "Clásico" (ya que es de 2017).

Podemos representar un usuario al que le gustan las películas de ciencia ficción de la siguiente forma:

```python
user1 = np.array([0.9,0.8,-0.6])
```

Si observamos, componente a componente, los valores se acercan mucho, por lo que es muy probable que a `user1` le guste la película.

Luego, calculamos el producto punto entre estos dos vectores:

```python
(user1*force_awakens).sum()

# Output: 2.1420000000000003
```


> [!NOTE] Nota
> El producto punto es **muy** importante en el ML, ya que constituye la base de la multiplicación de matrices. Básicamente, multiplicar dos vectores o matrices nx1 y 1xn es exactamente igual a calcular el producto punto. 


Si hacemos lo mismo para [Casablanca](https://www.imdb.com/es/title/tt0034583/?ref_=ext_shr_lnk):

```python
casablanca = np.array([-0.99,-0.3,0.8])
(user1*casablanca).sum()

# Output: -1.611
```

Para llevar esto a la práctica, podemos asignar estos valores a cada uno de los usuarios y películas y hacer nuestros cálculos para crear nuestras primeras predicciones. Los vectores asignados son inicializados aleatoriamente, por lo que las predicciones serán muy malas.

![[Untitled 1.png]]

Por ejemplo, el rating estimado que le dio el usuario 14 a la película 27 es:

$$
0.21 \times (-1.69) + 1.61 \times 1.01 + 2.89 \times 0.82 - 1.26 \times 1.89 + 0.82 \times 2.39
$$

Suponiendo que la primer componente simboliza qué tan romántica es una película, si el usuario tiene un valor muy alto y la película también o lo contrario (ambos scores muy bajos), el valor calculado va a ser muy alto, es decir, hay una gran correlación. Por otro lado, si al usuario le gustan mucho las películas románticas y la película no lo es, el valor será muy bajo,

Luego, debemos elegir una funcion de *loss*. Vamos a elegir [MSE](https://en.wikipedia.org/wiki/Mean_squared_error). Con esto podemos entrenar nuestro modelo minimizando la función mediante SGD.

> At each step, the stochastic gradient descent optimizer will calculate the match between each movie and each user using the dot product, and will compare it to the actual rating that each user gave to each movie. It will then calculate the derivative of this value and will step the weights by multiplying this by the learning rate. After doing this lots of times, the loss will get better and better, and the recommendations will also get better and better.


## DataLoaders

Vamos a implementar esto. Primero, creamos los DataLoaders (suponiendo que ya procesamos el dataset). 

```python
dls = CollabDataLoaders.from_df(ratings, item_name='title', bs=64)
dls.show_batch()
```

![[Pasted image 20250519212631.png]]

Luego, inicializamos las matrices necesarias para que PyTorch pueda operar sobre ellas:

```python
n_users  = len(dls.classes['user'])
n_movies = len(dls.classes['title'])
n_factors = 5

user_factors = torch.randn(n_users, n_factors)
movie_factors = torch.randn(n_movies, n_factors)
```

Para realizar los productos punto, debemos buscar los índices de la película y el usuario sobre los que queremos operar. Lamentablemente, esto no es algo que pueda hacer un modelo por si solo, pero podemos usar el siguiente "truco":

```python
one_hot_3 = one_hot(3, n_users).float()
user_factors.t() @ one_hot_3

# Output: tensor([-1.2493, -0.3099,  1.4229,  0.0840,  0.4132])
```

Lo que estamos haciendo es **Embedding**. Si hacemos esto repetidas veces, estaríamos usando muchísima memoria y tomaría mucho tiempo. Por suerte, librerías como PyTorch tienen estas operaciones totalmente optimizadas, por lo que no tenemos que preocuparnos por eso.


Si este fuera un problema de *computer vision*, sería sencillo caracterizar cada pixel, ya que cada uno tiene un valor que determina qué tan rojo, verde y azul es. Para nuestro caso, no existe tal información, por lo que nuestra única alternativa es dejar que el modelo la aprenda por sí mismo, determinando qué características son más o menos importantes.

Es decir, por cada iteración, calculamos nuestra función loss comparando nuestras predicciones con sus valores reales. Luego, calculamos el gradiente de loss respecto a los embeddings y los actualizamos optimizándolos con SGD.


## From Scratch

En PyTorch, podemos crear nuestro propio modelo a través de una clase de Python siguiendo los siguientes requisitos:

- hacer que nuestra clase herede de `Module`, provisto por PyTorch,
- crear la función `forward`, la cual llamará PyTorch a la hora de entrenar el modelo

```python
class DotProduct(Module):
    def __init__(self, n_users, n_movies, n_factors):
        self.user_factors = Embedding(n_users, n_factors)
        self.movie_factors = Embedding(n_movies, n_factors)
        
    def forward(self, x):
        users = self.user_factors(x[:,0])
        movies = self.movie_factors(x[:,1])
        return (users * movies).sum(dim=1)
```

Notar que el input es una matriz con shape `batch x 2`, donde la primer columna son los IDs de los usuarios y la segunda columna las IDs de las películas.

Creamos nuestro Learner y lo entrenamos:

```python
model = DotProduct(n_users, n_movies, 50)
learn = Learner(dls, model, loss_func=MSELossFlat())
learn.fit_one_cycle(5, 5e-3)
```

| epoch | train_loss | valid_loss | time  |
|:-----:|:----------:|:----------:|:-----:|
| 0     | 1.391418   | 1.281064   | 00:10 |
| 1     | 1.071581   | 1.059740   | 00:10 |
| 2     | 0.925082   | 0.955736   | 00:10 |
| 3     | 0.789137   | 0.877227   | 00:09 |
| 4     | 0.742079   | 0.862848   | 00:09 |

Nuestro modelo no es tan bueno, pero podemos seguir mejorándolo.

Para esto, vamos a implementar dos cambios fundamentales:

- una sigmoide que prevenga valores mayores a 5,
- biases para los usuarios y películas, indicando si un usuario es más propenso a ser más positivo o negativo con sus reviews y si una película es, según las reseñas, mejor o peor

```python
class DotProductBias(Module):
    def __init__(self, n_users, n_movies, n_factors, y_range=(0,5.5)):
        self.user_factors = Embedding(n_users, n_factors)
        self.user_bias = Embedding(n_users, 1)
        self.movie_factors = Embedding(n_movies, n_factors)
        self.movie_bias = Embedding(n_movies, 1)
        self.y_range = y_range
        
    def forward(self, x):
        users = self.user_factors(x[:,0])
        movies = self.movie_factors(x[:,1])
        res = (users * movies).sum(dim=1, keepdim=True)
        res += self.user_bias(x[:,0]) + self.movie_bias(x[:,1])
        return sigmoid_range(res, *self.y_range)

model = DotProduct(n_users, n_movies, 50)
learn = Learner(dls, model, loss_func=MSELossFlat())
learn.fit_one_cycle(5, 5e-3)
```

| epoch | train_loss | valid_loss | time  |
|:-----:|:----------:|:----------:|:-----:|
| 0     | 0.943364   | 0.932987   | 00:08 |
| 1     | 0.824034   | 0.864243   | 00:08 |
| 2     | 0.586939   | 0.860469   | 00:09 |
| 3     | 0.409691   | 0.886897   | 00:08 |
| 4     | 0.303943   | 0.894082   | 00:08 |

Nuestro modelo empeoró. Podemos ver que a partir de la tercera iteración empieza a empeorar; este es un claro indicador de *overfitting*. 


## L2 Regularization

La Regularización [[L2]] consiste en agregar la suma de los cuadrados de los pesos a la función loss, con el objetivo de que, al calcular el gradiente, los pesos se hagan más pequeños.

Esto previene el overfitting ya que mientras más altos son los coeficientes, más "agudos" son los "cañones" de *loss*, causando que nuestro modelo se ajuste a una función demasiado compleja y con cambios muy abruptos, causando que overfittee.

Siendo `wd` -sigla para Weight Decay, sinónimo de L2- un parámetro que controla la suma de los cuadrados de los pesos:

```python
loss_with_wd = loss + wd * (parameters**2).sum()
```

`wd` es un parámetro que podemos elegir y pasar como argumento cuando ajustamos nuestro modelo:

```python
model = DotProductBias(n_users, n_movies, 50)
learn = Learner(dls, model, loss_func=MSELossFlat())
learn.fit_one_cycle(5, 5e-3, wd=0.1)
```

| epoch | train_loss  | valid_loss  | time  |
|:-----:|:-----------:|:-----------:|:-----:|
| 0     | 0.964149    | 0.947329    | 00:08 |
| 1     | 0.853209    | 0.862615    | 00:09 |
| 2     | 0.734107    | 0.828079    | 00:08 |
| 3     | 0.595621    | 0.812455    | 00:08 |
| 4     | 0.490830    | 0.812497    | 00:08 |

El rendimiento mejoró mucho.
