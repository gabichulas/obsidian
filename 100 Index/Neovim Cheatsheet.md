### Movimiento y Navegación Básica

| Comando | Acción |
| :--- | :--- |
| `h`, `j`, `k`, `l` | Izquierda, Abajo, Arriba, Derecha |
| `w`, `b`, `e` | Inicio de siguiente palabra, Inicio de palabra anterior, Fin de palabra |
| `0`, `$` | Inicio absoluto de la línea, Fin de la línea |
| `gg`, `G` | Primera línea del archivo, Última línea del archivo |
| `Ctrl + u`, `Ctrl + d` | Scroll media página hacia arriba, Scroll media página hacia abajo |
| `%` | Saltar entre llaves `{}`, corchetes `[]` o paréntesis `()` correspondientes |
| `f [letra]`, `t [letra]` | Saltar hacia la letra en la misma línea, Saltar justo antes de la letra |

### Edición y Operadores

| Comando | Acción |
| :--- | :--- |
| `i`, `I` | Modo Insertar antes del cursor, Insertar al inicio de la línea |
| `a`, `A` | Modo Insertar después del cursor, Insertar al final de la línea |
| `o`, `O` | Insertar nueva línea abajo, Insertar nueva línea arriba |
| `x`, `s` | Borrar un carácter, Borrar carácter y entrar a modo Insertar |
| `d`, `c`, `y` | Borrar (Delete/Cortar), Cambiar (Borrar y modo Insertar), Copiar (Yank) |
| `dd`, `cc`, `yy` | Borrar línea completa, Cambiar línea completa, Copiar línea completa |
| `p`, `P` | Pegar después del cursor, Pegar antes del cursor |
| `u`, `Ctrl + r` | Deshacer, Rehacer |
| `Ctrl + a`, `Ctrl + x` | Incrementar número bajo el cursor, Decrementar número |

### Objetos de Texto (Combinar con `d`, `c`, o `y`)

| Comando | Acción |
| :--- | :--- |
| `i w`, `a w` | Dentro de la palabra (Inside word), Alrededor de la palabra (Around word) |
| `i {`, `a {` | Dentro de las llaves, Alrededor de las llaves (incluye las llaves) |
| `i (`, `i "` | Dentro de paréntesis, Dentro de comillas (aplica a cualquier bloque) |

*Ejemplo: `c i w` borra la palabra entera y entra a modo insertar. `d i {` vacía el contenido de una función.*

### LazyVim: Atajos Clave (Tecla Líder: `<Space>`)

| Comando | Acción |
| :--- | :--- |
| `<Space> e` | Abrir/Cerrar explorador de archivos (Neo-tree) |
| `<Space> f f` | Buscar archivos por nombre (Telescope) |
| `<Space> s g` | Buscar texto en todo el proyecto (Live Grep) |
| `<Space> c a` | Acciones de código del linter (Code Actions) |
| `<Space> c f` | Formatear el archivo actual (Go fmt) |
| `<Space> x x` | Mostrar panel de errores y diagnósticos (Trouble) |
| `s` | Búsqueda rápida y salto visual (Flash) |
| `K` | (Shift+k) Mostrar documentación bajo el cursor (Hover) |
| `g d` | Ir a la definición de la función o variable (Go to definition) |
| `Ctrl + o`, `Ctrl + i` | Volver a la posición anterior, Ir a la posición siguiente |

### Gestión de Ventanas y Pestañas

| Comando | Acción |
| :--- | :--- |
| `:vsp`, `:sp` | Dividir pantalla verticalmente, Dividir pantalla horizontalmente |
| `Ctrl + w` seguido de `h/j/k/l` | Mover el foco a la ventana izquierda/abajo/arriba/derecha |
| `Ctrl + w` seguido de `q` | Cerrar la ventana actual |