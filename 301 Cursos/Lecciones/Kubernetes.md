10-7-2026
**Clase**: [[Cloud Engineering]]
**Temas**: #k8s #docker #containers #devops

---
# Índice

- [Ganado](#Ganado)
- [Arquitectura del Cluster](#Arquitectura%20del%20Cluster)
- [Namespaces](#Namespaces)
- [Pods](#Pods)
	- [env](#env)
	- [resources](#resources)
	- [probes](#probes)
- [Deployments](#Deployments)
	- [Anatomía del Deployment](#Anatom%C3%ADa%20del%20Deployment)
- [DaemonSet](#DaemonSet)
- [StatefulSet](#StatefulSet)
- [Networking](#Networking)
	- [CNI y Calico](#CNI%20y%20Calico)
	- [Pod Networking](#Pod%20Networking)
- [Service](#Service)
	- [Definición de un Service](#Definici%C3%B3n%20de%20un%20Service)
	- [Services sin selectores](#Services%20sin%20selectores)
	- [Tipos de Service](#Tipos%20de%20Service)
	- [¿Para qué?](#%C2%BFPara%20qu%C3%A9?)
		- [Cluster IP](#Cluster%20IP)
		- [NodePort](#NodePort)
		- [LoadBalancer](#LoadBalancer)
- [Ingress](#Ingress)

---
# Ganado

El problema con Docker puro o Docker Compose es cuando tenemos que escalar. Correr 10 o 20 instancias a mano o con scripts y mantenerlas se vuelve problemático. Para solucionar esto, pasamos a pensar en las instancias como ganado, en vez de mascotas.

- **Mascotas:** Servidores que cuidás, tienen nombre propio. Si se enferman o se caen, no querés que mueran y tratás de arreglarlos.
- **Ganado:** Instancias efímeras y descartables. Si una vaca (instancia) muere, mucho no te importa porque simplemente levantás otra. Kubernetes es un orquestador que automatiza esto.

---

# Arquitectura del Cluster

Un cluster se divide en dos grandes partes: el Control Plane y los Workers.

- **Control Plane**: Tiene la API de K8s, el scheduler (componente que decide dónde van los contenedores) y `etcd`. `etcd` es una base de datos clave-valor donde se guarda todo el estado del cluster.
- **Workers**: Los nodos donde corren los contenedores. Tienen un agente llamado `kubelet` que conecta el worker con la API de K8s  y gestiona los contenedores del nodo (como un capataz), y `kube-proxy` que gestiona el networking.


![[Pasted image 20260710134205.png]]

---
# Namespaces

Son divisiones lógicas dentro del cluster. Sirven para separar recursos de distintas aplicaciones o entornos. Por defecto, K8s trae algunos preconfigurados, como `default` o `kube-system`. `kube-system` es súper importante porque está reservado para los procesos internos del propio Kubernetes.

---
# Pods

En K8s NO corremos contenedores de Docker directamente, sino que la unidad mínima es el Pod. Un Pod es un set de contenedores. Es muy probable que corramos un solo contenedor por Pod, pero se pueden poner más. 

> [!important] 
> Todos los contenedores adentro de un mismo Pod comparten la misma IP y red, por lo que no pueden colisionar en puertos.

Un pod se define a través de un **manifiesto**, un archivo YAML donde declaramos el estado deseado.

Un pod básico puede verse así:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:alpine
```

Podemos interactuar con este. Para aplicar este manifiesto y desplegar el pod:

```bash
kubectl apply -f pod.yaml
```

Si vemos el estado:

```bash
kubectl get pods
```

![[Pasted image 20260710135226.png]]

Podemos meternos dentro del pod y ejecutar comandos. Notar que esto es practicamente igual a `docker exec`

```bash
kubectl exec -it nginx -- sh
```

> [!note] 
> Si matamos el pod manualmente (`kubectl delete pod nginx`), se muere y listo. Como lo creamos como un Pod crudo mediante un YAML simple y no con un `Deployment`, K8s no sabe que tiene que mantener copias vivas, así que no lo levanta de nuevo automáticamente.

Otro pod un poco más avanzado sería:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    env:
    - name: MI_VARIABLE
      value: "pelado"
    - name: MI_OTRA_VARIABLE
      value: "pelade"
    - name: DD_AGENT_HOST
      valueFrom:
        fieldRef:
          fieldPath: status.hostIP
    resources:
      requests:
        memory: "64Mi"
        cpu: "200m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 20
    ports:
    - containerPort: 80
```


Podemos notar muchas diferencias con el anterior manifiesto:

## env

```yaml
# ... (previous pod specs)
    env:
    - name: MI_VARIABLE
      value: "pelado"
    - name: MI_OTRA_VARIABLE
      value: "pelade"
    - name: DD_AGENT_HOST
      # ---> NOTE: Esto captura dinámicamente el valor de la IP del host. <---
      valueFrom:
        fieldRef:
          fieldPath: status.hostIP
```

En este bloque definimos variables de entorno dentro del container. Podemos hardcodear los valores u obtenerlos dinámicamente como en `DD_AGENT_HOST`.

## resources

```yaml
# ... (previous config)
    resources:
      # Guaranteed resources for the container
      requests:
        memory: "64Mi"
        cpu: "200m"
      # Maximum allowed resources
      limits:
        memory: "128Mi"
        cpu: "500m"
```

En el bloque `resources` definimos, valga la redundancia, los recursos asignados a este contenedor. Vemos dos formas de hacerlo:

- **requests**: Lo que el contenedor tiene garantizado. No puede tener menos de 64MB de RAM y 200 Milicores.
- **limits**: El máximo que puede utilizar. Si usara más de esto, el propio kernel de Linux mataría estos contenedores.

> [!note]
> 1 Core = 1000 Milicores


## probes

```yaml
# ... (previous config)
    readinessProbe:
      httpGet:
        path: /
        port: 80
      # Wait 5 seconds before checking
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 20
```

K8s necesita saber cómo y cuándo determinar que el contenedor está *READY* y *ALIVE*. 

Con `readinessProbe` le decimos qué hacer para saber si el container está listo. En este caso, K8s espera 5 segundos (`initialDelaySeconds`) desde que el container se deployea y envía requests HTTP (`httpGet`) cada 10 segundos (`periodSeconds`) hasta obtener un código 200 OK.

Para saber si el container está vivo, establecemos `livenessProbe`, que le ordena a K8s intentar establecer una conexión TCP (`tcpSocket`) con el contenedor cada 20 segundos, luego de haber esperado 15 segundos desde el despliegue.

---

# Deployments

Desplegar pods sueltos no es una buena práctica. De esta forma, sería muy engorroso y no podemos definir réplicas ni decirle a K8s que recree un pod si este muere.

Para solucionar esto, usamos un **Deployment**. Un Deployment es, básicamente, un template de los pods que se van a crear. La diferencia con un Pod es que podemos indicar cuántas réplicas queremos, entre otras cosas.

Ejemplo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        env:
        - name: MI_VARIABLE
          value: "pelado"
        - name: MI_OTRA_VARIABLE
          value: "pelade"
        - name: DD_AGENT_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        resources:
          requests:
            memory: "64Mi"
            cpu: "200m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
        ports:
        - containerPort: 80
```


Como vemos, el archivo es *muy* similar al anterior `pod.yaml`.

## Anatomía del Deployment

**1. Cabecera y Metadata del Deployment**

- `apiVersion: apps/v1` y `kind: Deployment`: Le decimos a la API de K8s exactamente qué tipo de objeto queremos crear y a qué grupo de la API pertenece. Los Deployments viven en `apps/v1`, a diferencia de un Pod simple que vive en `v1`.

- `metadata`: Contiene los datos para identificar al controlador. En este caso, si hacés un `kubectl get deployments`, lo vas a ver listado como `nginx-deployment`.

**2. El Spec del Deployment** 
A partir de `spec:` empiezan las instrucciones para el Controller Manager de K8s.

- `replicas: 2`: Acá está la magia de la alta disponibilidad. Le estamos diciendo: _"No me importa qué pase, asegurate de que SIEMPRE haya exactamente 2 Pods corriendo"_. Si borrás uno a mano, el Deployment levanta otro.

**3. Selector**

- `selector` -> `matchLabels`: Un Deployment en realidad no "posee" a los Pods, sino que los "vigila" basándose en labels. Acá le decimos al Deployment: _"Tu trabajo es vigilar y contar todos los Pods en el clúster que tengan la etiqueta `app: nginx`"_. Si cuenta menos de 2, levanta más. Si cuenta 3, mata uno.

**4. Template**

- `template`: Define la estructura de los Pods que van a crearse.

- `metadata` -> `labels`: Al crear el Pod usando este molde, le asigna la etiqueta `app: nginx`.

---

# DaemonSet

Este `kind` es muy similar a un [Deployment](#Deployments). El objetivo de un DaemonSet es desplegar solo UN Pod en cada uno de los nodos.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        env:
        - name: MI_VARIABLE
          value: "pelado"
        - name: MI_OTRA_VARIABLE
          value: "pelade"
        - name: DD_AGENT_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        resources:
          requests:
            memory: "64Mi"
            cpu: "200m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
        ports:
        - containerPort: 80
```

Vemos que, en el archivo YAML, la única diferencia con el Deployment es el `kind` y la ausencia del parámetro `replicas`. Esto se debe a que no nos interesan las réplicas, solo queremos exactamente un Pod en cada nodo.

---

# StatefulSet

Los Pods que creamos antes son *Stateless*. Si quisieramos que esto no fuera así, igual que en Docker, necesitamos asignarle un **volúmen**. Esto podemos hacerlo mediante un **StatefulSet**.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-csi-app-set
spec:
  selector:
    matchLabels:
      app: mypod
  serviceName: "my-frontend"
  replicas: 1
  template:
    metadata:
      labels:
        app: mypod
    spec:
      containers:
      - name: my-frontend
        image: busybox
        args:
        - sleep
        - infinity
        volumeMounts:
        - mountPath: "/data"
          name: csi-pvc
  volumeClaimTemplates:
  - metadata:
      name: csi-pvc
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
      storageClassName: do-block-storage
```

Vamos a desglosar esto:

**1. `serviceName`**

A diferencia de un Deployment, el StatefulSet necesita mantener un orden y una identidad de red estable para cada réplica.

```yaml
# ... (metadata and selector)
  # Links the StatefulSet to a Headless Service
  serviceName: "my-frontend"
  replicas: 1
```

- `serviceName`: Los StatefulSets se apoyan en un tipo especial de Service llamado "Headless Service". Esto permite que los Pods no tengan nombres aleatorios como `mypod-7db6d8-5z7qx`, sino nombres secuenciales y predecibles: `my-csi-app-set-0`, `my-csi-app-set-1`, etc. Este parámetro vincula el StatefulSet con ese servicio para que cada Pod obtenga un registro DNS interno único (ej. `my-csi-app-set-0.my-frontend.default.svc.cluster.local`).

**2. `volumeMounts`**

Esto le indica al contenedor dónde va a inyectar el disco físico que Kubernetes le asigne.

```yaml
# ... (inside container spec)
        volumeMounts:
        # The path inside the container
        - mountPath: "/data"
          # The name of the volume to mount
          name: csi-pvc
```

- `mountPath`: Es la ruta interna del sistema de archivos de tu contenedor donde van a vivir los archivos persistentes. Si el contenedor escribe algo en `/data`, se estará guardando físicamente en el disco externo.

- `name`: Es el identificador lógico del volumen. Debe coincidir exactamente con el nombre que definimos más abajo en el template.


**3. `volumeClaimTemplates`** 

Es un template que Kubernetes usa para pedirle almacenamiento a la infraestructura.

```yaml
  volumeClaimTemplates:
  - metadata:
      name: csi-pvc
    spec:
      # Defines how the volume can be accessed
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          # Disk size requested
          storage: 5Gi
      # The cloud provider's storage driver
      storageClassName: do-block-storage
```

- `accessModes` -> `ReadWriteOnce`: Define los permisos físicos del disco. `ReadWriteOnce` significa que el disco permite lectura y escritura, pero solo puede estar montado en **un único nodo físico a la vez**.

- `storage`: Estamos declarando que necesitamos un disco de 5 Gigabytes de capacidad.

- `storageClassName`: Un _StorageClass_ actúa como un driver para un proveedor específico. `do-block-storage` le indica a Kubernetes que se conecte automáticamente a la API de DigitalOcean, cree un disco virtual real de 5GB y lo asigne a la instancia donde va a correr el Pod.

---

# Networking

![[Pasted image 20260711142647.png]]

En Docker, los contenedores viven en una red interna aislada dentro del servidor físico. Si queremos que dos contenedores en distintos servidores se comuniquen, hay que exponer puertos a las IPs públicas, lo cual no es escalable. K8s soluciona esto imponiendo una regla estricta:

1. Cada Pod tiene su propia IP única en todo el clúster.

2. Cualquier Pod debe poder comunicarse con otro Pod, en cualquier nodo, sin usar NAT.

## CNI y Calico 

K8s exige esta regla, pero no trae la herramienta para implementarla por defecto. Para esto existe el CNI (*Container Network Interface*), un estándar que permite instalar plugins de red de terceros. En el ejemplo usamos Calico.

## Pod Networking

Para lograr que los Pods se comuniquen entre distintos workers Calico despliega un agente en todos los nodos (osea, un [DaemonSet](#DaemonSet)). Cuando K8s levanta un Pod, el agente local le asigna una IP privada única (ej: 10.0.10.34). Para conectar dos Pods en distintos servidores, Calico manipula las tablas de ruteo del sistema operativo (el host físico). Crea rutas (`ip route`) que le dicen a la máquina física: "todo paquete que busque la IP 10.0.10.37, mandalo directo a la IP pública/privada del Worker de enfrente". Toda esta topología (quién tiene qué IP y dónde) se guarda y sincroniza a través de `etcd`.

> [!tip]
El CNI básicamente arma una red virtual por encima de la red física. Los Pods sienten que están conectados al mismo switch, sin importar en qué máquina física estén corriendo realmente.

---

# Service

Un Service es el objeto de la API de K8s que describe cómo se accede a las aplicaciones, tal como un conjunto de [Pods](#Pods), y que puede describir puertos y balanceadores de carga.

Cada Pod obtiene su propia dirección IP, sin embargo, en un Deployment, el conjunto de Pods corriendo en un momento dado puede ser diferente al conjunto de Pods corriendo esa aplicación un momento después.

Esto conlleva un problema: si un conjunto de Pods (llamémoslos "backends") provee funcionalidad a otros Pods (llamémoslos "frontends") dentro de tu cluster, ¿de qué manera los frontends encuentran y tienen seguimiento de cuál dirección IP conectarse, para que el frontend pueda usar la parte del backend de la carga de trabajo?

En Kubernetes, un Service es una abstracción que define un conjunto lógico de Pods y una política por la cual acceder a ellos. El conjunto de Pods a los que apunta un Servicio se determina usualmente por un Selector.

## Definición de un Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-servicio
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

A este Service se le asigna una IP virtual fija (`ClusterIP`). El controlador escanea continuamente qué Pods coinciden con el `selector` definido y actualiza una lista en un objeto llamado `Endpoints`.

Esta especificación crea un nuevo objeto Service llamado "mi-servicio", que apunta via TCP al puerto 9376 de cualquier Pod con la etiqueta `app.kubernetes.io/name=MyApp`.

## Services sin selectores

Se utilizan para abstraer el acceso a recursos que no son Pods nativos de K8s (por ejemplo, un clúster de base de datos externo o un servicio legacy en otra red). Como no existe un `selector`, el objeto `Endpoints` no se crea automáticamente y el mapeo de la dirección IP y el puerto debe hacerse de forma manual.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-servicio
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
apiVersion: v1
kind: Endpoints
metadata:
  name: mi-servicio
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

> [!note]
> Por restricciones de seguridad, el servidor API de Kubernetes no permite hacer proxy (ej. `kubectl port-forward`) a Endpoints que no estén mapeados a Pods legítimos.

## Tipos de Service

Los valores `Type` y sus comportamientos son:

- `ClusterIP`: Expone el Service en una dirección IP interna del clúster. Al escoger este valor el Service solo es alcanzable desde el clúster. Este es el `ServiceType` por defecto.
    
- [`NodePort`](https://kubernetes.io/es/docs/concepts/services-networking/service/#tipo-nodeport): Expone el Service en cada IP del nodo en un puerto estático (el `NodePort`). Automáticamente se crea un Service `ClusterIP`, al cual enruta el `NodePort`del Service. Podrás alcanzar el Service `NodePort` desde fuera del clúster, haciendo una petición a `<NodeIP>:<NodePort>`.
    
- [`LoadBalancer`](https://kubernetes.io/es/docs/concepts/services-networking/service/#loadbalancer): Expone el Service externamente usando el balanceador de carga del proveedor de la nube. Son creados automáticamente Services `NodePort`y `ClusterIP`, a los cuales el apuntará el balanceador externo.

![[Pasted image 20260711174206.png]]

## ¿Para qué?

Si el Pod A quiere hablar con el Pod B, podría usar su IP. Pero si el Pod B muere, la réplica nace con otra IP. El código del Pod A se rompería constantemente. Necesitamos los Services para tener un **punto de encuentro estático**. El Service tiene un nombre DNS y una IP que nunca cambian; su único trabajo es recibir el tráfico y redirigirlo a los Pods reales. Dependiendo de _quién_ necesite acceder a esos Pods, elegimos el tipo de Service.

### Cluster IP

- **Utilidad:** Para la comunicación interna entre tus propios componentes. El mundo exterior no tiene forma de acceder a esta IP.

- **Caso típico:** Tu contenedor de Frontend necesita consultar la API del Backend. Creás un Service `ClusterIP` para el Backend. El Frontend simplemente llama a `http://backend-service` y listo.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80        # Service port internal to the cluster
      targetPort: 8080 # App port inside the container
```

### NodePort 

Expone el servicio hacia el **exterior** del clúster abriendo un puerto estático (en el rango 30000-32767) en la IP física de absolutamente todos los Workers.

- **Utilidad:** Cuando necesitás que un cliente externo al cluster le pegue a la aplicación directamente usando la IP del nodo donde corre Minikube o tus servidores virtuales.

- **Cómo funciona:** Si el puerto asignado es el `30123`, podés ir a tu navegador y escribir `http://IP-DEL-NODO:30123` y el tráfico entrará al clúster directo hacia tus Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30123 # Exposed external port on all nodes
```


### LoadBalancer

Es la evolución de NodePort para entornos de producción.

- **Utilidad:** Para exponer tu aplicación web con una única IP pública dedicada.

- **Cómo funciona:** Al declarar `type: LoadBalancer`, Kubernetes habla con la API del proveedor de nube y contrata un Balanceador de Carga físico en tu cuenta. Este balanceador recibe una IP pública y redirige todo el tráfico hacia los puertos de tus nodos (`NodePort`), que a su vez lo mandan al `ClusterIP` y finalmente a los Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: public-ingress-service
spec:
  type: LoadBalancer
  selector:
    app: gateway
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

---

# Ingress

En una arquitectura de microservicios, usar el service [LoadBalancer](#LoadBalancer) sería un error. Deberíamos crear un LoadBalancer por cada microservicio, lo cual sería muy ineficiente en términos de:

- **Costos:** Por cada Service, K8s se conecta a la API de tu proveedor de nube y contrata un balanceador.

- **Manejo de IPs:** $N$ IPs públicas distintas para componentes del mismo sistema.

- **Capa 4:** El LoadBalancer crudo trabaja en la Capa 4. No entiende de URLs, dominios, paths ni certificados SSL.

Para solucionar esto, podemos usar un nuevo `kind`: Ingress. 

Su ventaja fundamental es que opera en la Capa 7. Esto te permite tener **un único** punto de entrada público que recibe todo el tráfico. Para que K8s entienda este recurso, hay que instalar un Ingress Controller en el cluster, siendo NGINX el más estándar. Este controlador lee los manifiestos Ingress y autoconfigura sus reglas de ruteo.

> [!important]
> Cuando se instala el Ingress Controller, también se crea un [LoadBalancer](#LoadBalancer), el cual tiene una IP pública asignada. Al aplicar un manifiesto de tipo Ingress, se le asigna esa misma IP pública, ya que este hará uso de ese LB.

El pro de poder usar la Capa 7 es que ahora no es necesario contratar $N$ LBs, ya que ingress puede acceder a los pods usando rutas HTTP:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-app
spec:
  rules:
  - http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: hello-v1
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: hello-v2
            port:
              number: 8080
```

Si quisieramos acceder a un Pod de la `v1`:

```sh
curl http://{IP-PUBLICA}/v1
```

Si quisieramos acceder a un Pod de la `v2`:

```sh
curl http://{IP-PUBLICA}/v2
```

---



```dataview
TABLE WITHOUT ID
  map(file.inlinks, (x) => link(x)) AS "🔗 Referencias"
WHERE file.name = this.file.name
```