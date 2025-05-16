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