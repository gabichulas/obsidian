### Reglas de diseño (qué NO vamos a escribir)

• Cero malloc / free: El parser corta el request in-place sobre char buf[2048] y los archivos de www/ salen por streaming en bloques de char chunk[512]. Cero fugas y cero fragmentación de RAM en el ESP32.

• Cero fork(), hilos o select(): Loop secuencial con timeout de lectura. Sumás concurrencia recién si un día medís que te queda corto.

• Cero archivos extra antes de tiempo: Todo vive en server.c hasta el Paso 5, donde recién partimos include/ y src/ para compartir el código con ESP-IDF.

---

### Paso 1: Limpiar server.c para portabilidad + Makefile mínimo

• Qué falta:
    • En server.c: borrar `<asm-generic/socket.h>`, `<signal.h>`, `<sys/wait.h>`, `sigchld_handler` y el bloque `fork()` (L103-L110) (inexistentes en FreeRTOS).
    • Cambiar `uint sin_size` (L35) a `socklen_t` y corregir los paréntesis de `getaddrinfo` (L47).
    • Configurar `setsockopt(..., SO_RCVTIMEO, ...)` (~3s) en `new_fd` para que un cliente mudo no cuelgue el servidor.
    • Escribir Makefile (3 líneas) compilando con `-Wall -Wextra -Werror -fsanitize=address,undefined`.

• Qué necesitás saber: Lo que ya escribiste en main.

• Qué ganás: Separar POSIX portable de APIs exclusivas de Linux, timeouts defensivos en sockets y detección automática de bugs de memoria con AddressSanitizer.

• Check: `nc localhost 3490` sin mandar datos se cierra solo a los 3 segundos.

---

### Paso 2: Construir y enviar Responses HTTP/1.1 reales

• Qué falta: Reemplazar `send(new_fd, "Hola :)", 7, 0)` (L105) por:
    1. `send_all(int fd, const void *data, size_t len)` con un `while` para garantizar que salgan todos los bytes.
    2. `send_response(int fd, int status, const char *reason, const char *type, const void *body, size_t len)` que formatea los headers en un `char hdr[512]` con `snprintf` (Content-Type, Content-Length, Connection: close) y llama a `send_all` para headers y body.

• Qué necesitás saber: `snprintf` y manejo de `const void *` + `size_t` (igual a tus ejercicios 21–22).

• Qué ganás: Manejar envíos parciales en TCP y estructurar respuestas HTTP/1.1 válidas tanto para texto como para binarios.

• Check: Abrir `http://localhost:3490` en el navegador y ver `<h1>Hola :)</h1>` renderizado sin errores de protocolo.

---

### Paso 3: Leer el stream TCP + Parser HTTP a mano (in-place)

• Qué falta:
    1. **Lectura:** Loop sobre `recv()` acumulando en `char buf[2048]` hasta encontrar el fin de headers `"\r\n\r\n"` (si llena el buffer sin encontrarlo → responder 413; si corta antes → 400).
    2. **Modelo:** Definir `struct HttpHeader` (key, value) y `struct HttpRequest` (method, path, version, headers[16], header_count, body, body_len).
    3. **Parser a mano:** `int parse_request(char *buf, size_t len, HttpRequest *req)` recorriendo `buf` con punteros:
        • Cortar method, path y version buscando `' '` y `"\r\n"`, validando formato y reemplazando delimitadores por `'\0'`.
        • Cortar cada header en `':'`, saltar espacios iniciales del valor (`' '` / `'\t'`) y cortar en `"\r\n"` hasta toparse con la línea vacía.
        • Apuntar `req->body` después de `"\r\n\r\n"`.
    4. **Ruteo básico:** GET contra rutas conocidas → 200, otro método → 405, ruta desconocida → 404, error del parser → 400.
    5. **Autotest mínimo:** Una función `test_parser()` con `assert` (corrible con `make test`) probando 5 casos: GET simple, múltiples headers, espacios raros, request truncado y basura.

• Qué necesitás saber: Aritmética de punteros `char *`, terminadores `'\0'` y paso de struct por puntero (`HttpRequest *`).

• Qué ganás: Dominar zero-copy parsing en C de sistemas, lectura correcta de streams TCP fragmentados, manejo de bordes maliciosos/rotos y códigos de estado HTTP.

• Check: `make test` en verde con sanitizers activos + pruebas con `curl -v` y `printf "BASURA\r\n\r\n" | nc localhost 3490`.

---

### Paso 4: Servir archivos estáticos desde www/ por streaming

• Qué falta:
    1. **Seguridad:** Rechazar con 403 Forbidden cualquier `req.path` que no empiece con `'/'` o contenga `".."` (evita ataques de Path Traversal como `/../src/server.c`).
    2. Mapear `"/"` a `"/index.html"`, armar la ruta en disco con `snprintf`, abrir con `fopen(..., "rb")` (NULL → 404) y sacar el tamaño con `stat()`.
    3. Elegir Content-Type según la extensión buscando el último `'.'` (.html, .css, .js, .png, .ico).
    4. Mandar los headers de respuesta y hacer un loop `fread(chunk, 1, sizeof chunk, f)` → `send_all()`.

• Qué necesitás saber: `fopen` / `fread` / `fclose` y `stat()`.

• Qué ganás: I/O de archivos binarios en C, defensa básica de servidores web y streaming con consumo de RAM constante O(1) (crítico en sistemas embebidos).

• Check: Servir un `www/index.html` con estilos/imagen en el navegador y comprobar que `curl --path-as-is http://localhost:3490/../Makefile` devuelva 403.

---

### Paso 5: Modularizar e integrar con ESP-IDF + FreeRTOS en el ESP32

• Qué falta:
    1. Extraer el servidor a `src/httpd.c` + `include/httpd.h` exponiendo `void httpd_run(const char *port, const char *web_root)`. En Linux, `main()` queda de una línea.
    2. Crear el proyecto ESP-IDF partiendo del ejemplo oficial de Wi-Fi Station, montar `www/` en flash con LittleFS (para que tus mismos `fopen`/`fread` funcionen sin tocarlos sobre `"/littlefs/..."`) y lanzar la tarea en `app_main`: `xTaskCreate(httpd_task, "httpd", 8192, NULL, 5, NULL);`
    3. Ajustar el tamaño de stack de la tarea en FreeRTOS (8192 bytes) contemplando tus buffers locales (`buf[2048]` + `chunk[512]`) y sumar un endpoint de hardware (ej. POST o GET `/led/toggle` moviendo un GPIO real).

• Qué necesitás saber: Uso básico de `idf.py` (build, flash, monitor), concepto de tarea/stack en FreeRTOS y tabla de particiones del ESP32.

• Qué ganás: Arquitectura C portable entre Linux y microcontrolador, Virtual Filesystem (VFS) en memoria flash, dimensionamiento real de stack en un RTOS y control de hardware por red.

• Check: Entrar desde el navegador del teléfono a la IP del ESP32, cargar la web desde la flash y controlar un pin físico de la placa.