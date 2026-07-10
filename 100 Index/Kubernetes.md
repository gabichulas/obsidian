10-7-2026
**Clase**: [[Cloud Engineering]]
**Temas**: #k8s #docker #containers #devops

**Estado**: #wip

---
# Índice

- [Ganado](#Ganado)
- [Arquitectura del Cluster](#Arquitectura%20del%20Cluster)

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

### env

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

### resources

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


### probes

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




```dataview
TABLE WITHOUT ID
  map(file.inlinks, (x) => link(x)) AS "🔗 Referencias"
WHERE file.name = this.file.name
```