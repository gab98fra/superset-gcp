
# Apache Superset en Google Cloud Platform (GKE + Cloud SQL)

Este repositorio contiene la configuración completa para desplegar [Apache Superset](https://github.com/apache/superset) en **Google Kubernetes Engine (GKE)** usando una base de datos **PostgreSQL en Cloud SQL** como metastore, incluyendo:

- Imagen personalizada de Superset (con soporte para PostgreSQL)
- Autenticación segura con Cloud SQL Auth Proxy
- Configuración con `superset_config.py` para usar `SQLALCHEMY_DATABASE_URI`
- Archivos YAML necesarios (deployment, service, configmap, secret)

---

##  Servicios de GCP utilizados

| Servicio | Descripción |
|----------|-------------|
| GKE (Google Kubernetes Engine) | Cluster para ejecutar Superset y el proxy |
| Cloud SQL (PostgreSQL) | Base de datos para los metadatos de Superset |
| Cloud SQL Auth Proxy | Acceso seguro desde GKE a Cloud SQL |
| Artifact Registry | Almacenamiento de la imagen personalizada de Superset |
| IAM (Service Account) | Acceso autenticado al proxy |
| Kubernetes Secrets | Manejo seguro del `credentials.json` |
| ConfigMaps | Montaje de configuraciones personalizadas (`superset_config.py`) |

---

## ✅ Pre-requisitos

1. Tener un proyecto de GCP activo y facturación habilitada
2. Tener habilitados los siguientes servicios:

```bash
gcloud services enable \
  container.googleapis.com \
  sqladmin.googleapis.com \
  artifactregistry.googleapis.com \
  iam.googleapis.com
```

3. Crear un bucket en Artifact Registry para la imagen:

```bash
gcloud artifacts repositories create superset \
  --repository-format=docker \
  --location=us-central1 \
  --description="Superset images"
```

4. Crear instancia PostgreSQL en Cloud SQL con:
   - Base de datos: `superset`
   - Usuario: `usuario`
   - Contraseña: `password`
   - Conexión: `up-computo:us-central1:superset`

5. Crear un clúster en GKE:

```bash
gcloud container clusters create superset-cluster \
  --zone us-central1-c \
  --num-nodes 3 \
  --enable-ip-alias

gcloud container clusters get-credentials superset-cluster --zone us-central1-c

```

6. Crear una **Service Account** con el rol `Cloud SQL Client`, exportar la credencial generada `credentials.json`

```bash
kubectl create secret generic cloudsql-instance-credentials \
  --from-file=credentials.json=/ruta/a/credentials.json \
  --namespace superset
```

---

## 📁 Estructura del proyecto

```
/
├── Dockerfile
├── superset-deployment.yaml
├── superset-service.yaml
├── superset-configmap.yaml
├── README.md (este archivo)
```

---

##  Construcción de la imagen personalizada

### Dockerfile

```dockerfile
FROM apache/superset:latest

USER root
RUN pip install psycopg2-binary
USER superset
```

### Build & Push

```bash
docker build -t us-central1-docker.pkg.dev/up-computo/superset/superset:latest .
docker push us-central1-docker.pkg.dev/up-computo/superset/superset:latest
```

---

##  Archivos Kubernetes

### 1. `superset-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: superset-config
  namespace: superset
data:
  superset_config.py: |
    import os
    SQLALCHEMY_DATABASE_URI = os.getenv("SQLALCHEMY_DATABASE_URI", "sqlite:////app/superset_home/superset.db")
```

### 2. `superset-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: superset-deployment
  namespace: superset
spec:
  replicas: 1
  selector:
    matchLabels:
      app: superset
  template:
    metadata:
      labels:
        app: superset
    spec:
      containers:
      - name: superset
        image: us-central1-docker.pkg.dev/up-computo/superset/superset:latest
        ports:
        - containerPort: 8088
        env:
        - name: SUPERSET_SECRET_KEY
          value: "mysecretkey"
        - name: SQLALCHEMY_DATABASE_URI
          value: postgresql+psycopg2://<usuario>:<password>@127.0.0.1:5432/superset
        command: ["/bin/sh", "-c"]
        args:
          - >
            superset db upgrade &&
            superset fab create-admin \
              --username admin \
              --firstname Superset \
              --lastname Admin \
              --email admin@example.com \
              --password admin &&
            superset init &&
            superset run -h 0.0.0.0 -p 8088
        volumeMounts:
        - name: superset-config
          mountPath: /app/pythonpath/superset_config.py
          subPath: superset_config.py
      - name: cloudsql-proxy
        image: gcr.io/cloudsql-docker/gce-proxy:1.33.1
        command:
          - "/cloud_sql_proxy"
          - "-instances=up-computo:us-central1:superset=tcp:5432"
          - "-credential_file=/secrets/cloudsql/credentials.json"
        volumeMounts:
        - name: cloudsql-instance-credentials
          mountPath: /secrets/cloudsql
          readOnly: true
      volumes:
      - name: cloudsql-instance-credentials
        secret:
          secretName: cloudsql-instance-credentials
      - name: superset-config
        configMap:
          name: superset-config
```

### 3. `superset-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: superset-service
  namespace: superset
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8088
  selector:
    app: superset
```

---

##  Desplegar todo en GKE

```bash
kubectl create namespace superset
kubectl apply -f superset-configmap.yaml
kubectl apply -f superset-deployment.yaml
kubectl apply -f superset-service.yaml
```

---

##  Acceder a Superset

```bash
kubectl get svc -n superset
```

Abrir el navegador: `http://<EXTERNAL-IP>`

- Usuario: `admin`
- Contraseña: `admin`

---


> Este proyecto está diseñado para entornos de prueba, evaluación y desarrollo. Para producción, se recomienda integrar balanceadores, seguridad y alta disponibilidad adecuadas.
