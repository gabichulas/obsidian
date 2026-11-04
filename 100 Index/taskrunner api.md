ya tengo:

- repo: set de funciones definidas para interactuar con la db
- workers funcionando
- etc

necesito la api

endpoints a implementar:

- GET /jobs/{id}: obtiene un job por id
- GET /jobs/: devuelve lista de todos los jobs del usuario
- GET /jobs/{status}: devuelve lista de jobs del usuario por estado (completed, failed, etc)
- POST /jobs/enqueue: encola un trabajo, enviando su respectivo payload y demás
- healthcheck
- un get para consultar el estado de un job mientras se está ejecutando?

la api funciona como un gateway porque los workers ya hacen todo el trabajo de obtener los jobs. la api solo toma el input el usuario e interactua con la db.

dentro de /api, mi idea es la siguiente estructura de carpetas:

- routes: rutas y definicion del mux
- handlers: funciones usadas por los endpoints
- ... ?

