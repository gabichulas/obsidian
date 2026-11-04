### Ejercicio 1 - Diagnóstico de Incidentes mediante Responsabilidad Compartida

| Incidente | Determinación de responsabilidad                            | Capa de servicio                     | Fundamentación                                                                                                                                                                                                                                                                                                                                     | Medidas preventivas                                                                                                                                                                                                                                |
| :-------- | :---------------------------------------------------------- | :----------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1**     | **Cliente (Empresa)**                                       | PaaS                                 | En un modelo de base de datos administrada el cliente es responsable de los datos, la configuración de respaldos, la retención y la gestión de accesos (quién puede eliminar recursos).                                                                                                                                                            | Habilitar backups automatizados con políticas de retención claras. Activar la protección contra eliminación. Aplicar el principio de menor privilegio para evitar borrados accidentales.                                                           |
| **2**     | **Proveedor** (Falla física) / **Cliente** (Disponibilidad) | IaaS / PaaS (Infraestructura Global) | El proveedor es responsable de la "seguridad DE la nube" (hardware, redes físicas e instalaciones del centro de datos). El cliente es responsable de la "seguridad EN la nube", lo que implica diseñar arquitecturas tolerantes a fallos (Multi-AZ o Multi-Región) para garantizar la continuidad del negocio ante la caída de un centro de datos. | Implementar un Plan de Recuperación ante Desastres (DRP) en una región secundaria. Desplegar los recursos críticos en múltiples Zonas de Disponibilidad (Multi-AZ).                                                                                |
| **3**     | **Cliente (Empresa)**                                       | IaaS                                 | Al utilizar máquinas virtuales, el proveedor asegura la infraestructura subyacente, pero el cliente tiene control total y responsabilidad sobre el sistema operativo invitado, la gestión de credenciales (contraseñas predeterminadas) y la configuración del firewall perimetral de la instancia (puertos abiertos a Internet).                  | Configurar los Grupos de Seguridad (Security Groups) para restringir el acceso SSH solo a direcciones IP confiables (o usar AWS Systems Manager Session Manager). Deshabilitar contraseñas predeterminadas y forzar el uso de pares de claves SSH. |

---

### Ejercicio 2 - Plan de Recuperación ante Desastres (DRP)

**1. Optimización de la arquitectura hacia Pilot Light**
La arquitectura actual de Warm Standby incluye un DNS con failover hacia una región secundaria, donde se mantienen permanentemente activos un ALB y un grupo de instancias EC2 con capacidad reducida. Para transicionar a una estrategia Pilot Light y reducir costos manteniendo el RPO (<15 min) y RTO (<1 hora), se deben aplicar los siguientes cambios:
*   **Capa de Cómputo (EC2 y ALB):** Eliminar o apagar las instancias EC2 activas en la región secundaria. El Application Load Balancer también puede ser eliminado para ahorrar costos fijos. Estos recursos deben configurarse mediante Infraestructura como Código para ser aprovisionados o escalados únicamente en el momento en que se declare el desastre. Esto cumple con un RTO de 1 hora.
*   **Capa de Datos (RDS y S3):** Mantener la réplica de Amazon RDS y la replicación del bucket de S3 activas de forma continua. En Pilot Light, el núcleo de los datos siempre debe estar encendido y sincronizado para garantizar un RPO menor a 15 minutos. 

**2. Optimización de Backups en la cuenta primaria**
Para reducir costos y mejorar la gestión de los respaldos sin comprometer los objetivos de recuperación:
*   **Ciclo de vida y Almacenamiento:** Implementar políticas que trasladen automáticamente los respaldos antiguos (ej. mayores a 30 días) desde el almacenamiento estándar hacia capas de almacenamiento en frío de menor costo, como Amazon S3 Glacier o Glacier Deep Archive.
*   **Retención:** Definir reglas de expiración definitivas para eliminar los respaldos que superen el marco regulatorio o las necesidades del negocio (ej. eliminar después de 1 o 5 años).
*   **Pruebas de Restauración:** Automatizar rutinas mensuales o trimestrales que restauren un backup en un entorno aislado para validar la integridad de los datos y medir el tiempo real de recuperación (RTO).

---

### Ejercicio 3 - Diagnóstico de Resiliencia, Monitoreo y Pipelines CI/CD Inmutables

**Eje 1: Monitoreo Proactivo y Alarmas**
Para evitar la caída en cascada producida por el agotamiento de recursos al esperar respuestas de un microservicio secundario, se debió monitorear la **latencia de las peticiones (Request Duration)** y la **saturación del pool de conexiones (Connection Pool / Active Threads)**.
*   **Métrica a monitorear:** Peticiones pendientes o latencia en el percentil 95/99.
*   **Umbral de alerta:** Disparar una alarma crítica si el uso del pool de conexiones supera el 85% durante más de 1 minuto, o si el P99 de la latencia supera los 2 segundos. Adicionalmente, el microservicio de procesamiento de pagos debe implementar un patrón de resiliencia como *Circuit Breaker* o *Timeouts* estrictos para abortar las conexiones bloqueadas antes de agotar la memoria.

**Eje 2: Despliegues Inmutables y CI/CD**
La inmutabilidad establece que un contenedor, una vez construido, jamás debe ser modificado en tiempo de ejecución (como hizo el operador ingresando por SSH). Si hay un error, se debe generar una nueva versión del código, construir una nueva imagen y desplegarla reemplazando la anterior. El flujo documentado exige un proceso automatizado que va desde el push del desarrollador hasta el despliegue final.

**Pasos técnicos clave:**

**1. Docker:**
Generar la imagen inmutable utilizando el hash del commit de Git (o un identificador de pipeline) como etiqueta, garantizando trazabilidad y evitando sobreescribir la etiqueta `latest`.

```bash
docker build -t digitalpay-service:${GITHUB_SHA} .
```

**2. GitHub Actions (Flujo CI/CD):**
El workflow incluye la ejecución en GitHub Actions, la construcción y etiquetado de la imagen, la subida a un Container Registry, y el despliegue en Kubernetes.

```yaml
jobs:
  deploy_workflow:
    runs-on: ubuntu-latest
    steps:
      # Step 1: Checkout the repository source code
      - name: Checkout source code
        uses: actions/checkout@v4

      # Step 2: Build the Docker image and push to Container Registry
      - name: Build and push image
        run: |
          docker build -t myregistry/digitalpay-service:${{ github.sha }} .
          docker push myregistry/digitalpay-service:${{ github.sha }}

      # Step 3: Trigger the Kubernetes deployment
      - name: Update Kubernetes Deployment
        run: |
          kubectl set image deployment/payments-deployment payments-container=myregistry/digitalpay-service:${{ github.sha }}
```

**3. Kubernetes:**
Para actualizar los Pods sin perder disponibilidad, se debe asegurar que el manifiesto del `Deployment` contenga la estrategia `RollingUpdate`.

```yaml
# Deployment manifest strategy section
strategy:
  type: RollingUpdate
  rollingUpdate:
    # Max number of pods that can be unavailable during the update
    maxUnavailable: 25%
    # Max number of pods that can be scheduled above the desired amount
    maxSurge: 25%
```