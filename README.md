# Proyecto-DevOps-EP3-backend

## 1. Arquitectura Detallada del Sistema

La capa de backend opera de forma desacoplada y asíncrona dentro del ecosistema, interactuando directamente con el motor de base de datos y respondiendo de manera eficiente a la capa de presentación.

### A. Componentes de Cómputo e Infraestructura (Kubernetes)
* **API Microservicio (`tienda-backend`):** Desarrollado sobre un entorno de Node.js/Express (o la tecnología homóloga del proyecto). Expone una API RESTful encargada de procesar las transacciones y persistir el catálogo en la base de datos. Escucha nativamente en el puerto `3001`.
* **Capa de Persistencia Relacional (`tienda-db`):** Motor de base de datos **MySQL 8.0**. Este componente se ejecuta de manera interna en el clúster con un almacenamiento persistente simulado y mapeado de forma lógica. Opera en el puerto estándar `3306`.

### B. Nodos / Fargate / Capacity Providers
Para el clúster maestro de Kubernetes (`devopseks`), la computación se administra bajo el modelo de **Nodos EC2 Administrados (EC2 Managed Node Groups)** en lugar de servidores Serverless como AWS Fargate. 
* **Aprovisionamiento:** AWS se encarga de manera automatizada del aprovisionamiento, actualización de parches del sistema operativo y mantenimiento de las instancias EC2 subyacentes.
* **Distribución de Carga:** Estas instancias actúan como los nodos trabajadores (*Workers*) del clúster de Kubernetes. Cuando el pipeline ejecuta un despliegue, el clúster distribuye homogéneamente los Pods (contenedores) a lo largo del hardware disponible para evitar la sobrecarga de un único servidor.
* **Resiliencia:** Al estar el clúster distribuido en un grupo de nodos, si una máquina EC2 física falla, Kubernetes migra los Pods de la API al nodo restante de forma automática en pocos segundos.

### C. Networking, Aislamiento y Seguridad (VPC)
Toda la infraestructura de red fue provista mediante la plantilla de AWS CloudFormation (`duoc-devops-act3.yaml`), implementando una segmentación de seguridad estricta:
* **Privacidad de Red:** Tanto la base de datos como la API se despliegan exclusivamente dentro de las **Subredes Privadas** de la VPC. Al carecer de direccionamiento IP público asignado, es físicamente imposible conectarse a la base de datos o al backend directamente desde Internet, previniendo ataques de inyección SQL externos o denegación de servicio (DDoS).
* **Alta Disponibilidad Multi-AZ:** Las subredes privadas se encuentran distribuidas entre múltiples Zonas de Disponibilidad físicas de AWS (ej: `us-east-1a` y `us-east-1b`). Si un centro de datos físico de Amazon sufre un corte de energía, los Pods del backend se mantendrán corriendo en el nodo de la otra zona sin interrupciones.
* **Firewalls Virtuales (Security Groups):** * El Pod de la base de datos MySQL (`tienda-db`) tiene un grupo de seguridad que restringe el tráfico entrante de forma estricta: **únicamente acepta conexiones en el puerto 3306 si provienen de la IP interna del Backend**.
  * El Backend solo procesa peticiones en el puerto `3001` orientadas desde el proxy o LoadBalancer del Frontend.

### D. Resiliencia, Disponibilidad y Alta Elasticidad
* **Métricas de Hardware (Metrics Server):** El clúster monitorea constantemente los recursos mediante sondas integradas.
* **Autoscaling Horizontal (HPA):** Se configuró el recurso `horizontalpodautoscaler` bajo la métrica de CPU. Si el promedio de cómputo del backend supera el **50% de su límite solicitado** (`requests: cpu: 100m`), Kubernetes gatilla la replicación elástica. El umbral se parametrizó con un mínimo de **1 Pod** y un techo máximo de **3 Pods** concurrentes distribuidos automáticamente entre los nodos EC2.
* **Ciclo de Vida (Probes de Salud):**
  * `Readiness Probe`: Monitorea el endpoint `/api/health` en el puerto `3001` con un retardo inicial de 5 segundos. Evita que el tráfico de los usuarios se envíe a un Pod que aún se está inicializando.
  * `Liveness Probe`: Ejecuta pruebas cíclicas cada 10 segundos en la misma ruta. Si detecta un bloqueo o congelamiento interno (*deadlock*), destruye y recrea el Pod de forma automatizada (*Self-Healing*).

---

## 2. Estructura Completa del Proyecto

```text
├── .github/
│   └── workflows/
│       └── deploy.yml          # Pipeline de CI/CD automatizado en GitHub Actions
├── backend/
│   ├── app.js                  # Lógica principal de la API REST
│   ├── package.json            # Dependencias del proyecto y scripts de ejecución
│   └── Dockerfile              # Instrucciones de compilación de la imagen Docker
├── k8s/
│   ├── backend-deployment.yaml # Definición del deployment y especificación de Pods
│   ├── mysql-deployment.yaml   # Despliegue del motor de base de datos MySQL
│   ├── frontend-hpa.yaml       # Configuración del autoescalado elástico del backend
│   └── namespace.yaml          # Aislamiento lógico del entorno ("tienda")
└── README.md                   # Documentación técnica principal del repositorio

