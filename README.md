# Proyecto Final – Plataforma de E-commerce (Despliegue y Monitorización)

Este proyecto implementa el backend de un sistema de comercio electrónico basado en **Node.js**, **PostgreSQL**, contenedores **Docker**, orquestación con **Kubernetes**, y un stack de monitoreo basado en **Prometheus + Grafana + Node Exporter**.  
Se desarrollaron tres fases principales: Docker Compose, k3s y finalmente kind con Nginx reverse proxy.

---

## Backend del Proyecto

El backend utilizado se clonó desde:  
- https://github.com/devdbrandy/restful-ecommerce

Incluye:

- API REST con **Node.js + Express**
- Módulos de usuarios, productos, categorías y pedidos
- Base de datos **PostgreSQL**
- Migraciones y seeders con **Sequelize**

Ejemplos de endpoints:

- GET /api/v1/products
- POST /auth/login
- GET /api/v1/users

# ✅ FASE 1 – Docker Compose + Stack de Monitoreo

## 1. Clonación del backend

```bash
git clone https://github.com/devdbrandy/restful-ecommerce
```
Cambios realizados:

- Actualización de Alpine en Dockerfile.
- Ajustes al docker-compose.yml.
- Volumen persistente nombrado.

## 2. Integración de Prometheus + Grafana + Node Exporter

- Se creó prometheus_conf.yml
- Se agregaron servicios al Docker Compose ( Node Exporter y Prometheus y Grafana)

Accesos:

- Prometheus → http://<EC2>:9090
- Grafana → http://<EC2>:3001

En Grafana:

- Se configuró Prometheus como datasource (http://prometheus:9090)
- Se importó dashboard 1860 – Node Exporter Full

Migraciones (Agrgar datos a la db):
```bash
docker exec -it restful-ecommerce_api_1 npx sequelize db:migrate
docker exec -it restful-ecommerce_api_1 npx sequelize db:seed:all
```
# ✅ FASE 2 – Migración a Kubernetes con K3s

En esta fase se migró el despliegue desde Docker Compose hacia Kubernetes utilizando **k3s**, una distribución ligera ideal para EC2.

## 1. Creación de Namespaces

```bash
sudo k3s kubectl apply -f namespace.yaml
sudo k3s kubectl apply -f namespace_m.yaml
```

Namespaces creados:
- `ecommerce` → API + Postgres  
- `monitoring` → Prometheus + Grafana + Node Exporter

---

## 2. Despliegue de API y Base de Datos

```bash
sudo k3s kubectl apply -f postgres-deployment.yaml
sudo k3s kubectl apply -f api-deployment.yaml
```

Verificar recursos:

```bash
sudo k3s kubectl get all -n ecommerce
```

Acceso a la API mediante NodePort:
```
http://<EC2_PUBLIC_IP>:32032/api/v1/products
```

---

## 3. Migraciones dentro del Pod de la API

```bash
sudo k3s kubectl exec -it -n ecommerce ecommerce-api-XXXX -- sh
npx sequelize db:migrate
npx sequelize db:seed:all
```

---

## 4. Despliegue del Stack de Monitoreo

```bash
sudo k3s kubectl apply -f prometheus-configmap.yaml
sudo k3s kubectl apply -f prometheus-deployment.yaml
sudo k3s kubectl apply -f grafana-deployment.yaml
sudo k3s kubectl apply -f node-exporter.yaml
```

Verificar:

```bash
sudo k3s kubectl get all -n monitoring
```

Puertos NodePort:
- Prometheus → **30090**
- Grafana → **30300**

Acceso desde navegador:
```
http://<EC2_PUBLIC_IP>:30300
```

### Configuración en Grafana
1. Login: `admin / admin`  
2. Agregar Prometheus como DataSource:
   ```
   http://prometheus:9090
   ```
3. Importar dashboard **ID 1860 – Node Exporter Full**

---

## 5. Escalamiento de la API

Editar `api-deployment.yaml`:

```yaml
replicas: 1
```

Cambiar por:

```yaml
replicas: 2
```

o más, según pruebas.

---

# ✅ FASE 3 – Migración a KIND + Reverse Proxy con Nginx

Para un entorno más limpio y reproducible, se eliminó el despliegue anterior y se migró a **kind**, un clúster Kubernetes dentro de Docker.

## 1. Limpieza del entorno K3s

```bash
sudo k3s kubectl delete -f kubernetes/api-deployment.yaml -n ecommerce
sudo k3s kubectl delete -f kubernetes/postgres-deployment.yaml -n ecommerce
sudo k3s kubectl delete -f kubernetes/prometheus-deployment.yaml -n monitoring
sudo k3s kubectl delete -f kubernetes/grafana-deployment.yaml -n monitoring
sudo k3s kubectl delete ns ecommerce
sudo k3s kubectl delete ns monitoring
```

---

## 2. Creación del clúster KIND

```bash
kind create cluster --name ecommerce-cluster-proyectosf3 --config kind.yaml
kubectl get nodes
```

Resultado esperado:
- 1 nodo control-plane  
- 1 nodo worker  

---

## 3. Adaptación de Services para KIND

KIND **NO soporta LoadBalancer**, por lo que todos los servicios deben ser:

```yaml
type: NodePort
```

Esto permite exponer los servicios hacia fuera usando Nginx como reverse proxy.

---

## 4. Configuración del Reverse Proxy con Nginx

Se crearon archivos individuales para evitar mezclas de tráfico:

- `/etc/nginx/sites-available/api`
- `/etc/nginx/sites-available/grafana`
- `/etc/nginx/sites-available/prometheus`

Ejemplo de configuración:

```nginx
server {
    listen 8081;
    location / {
        proxy_pass http://172.18.0.2:30300;  # Grafana NodePort
    }
}
```

Activación:

```bash
sudo ln -s /etc/nginx/sites-available/api /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

---

## 5. Acceso final a los servicios expuestos mediante Nginx

- API → `http://<EC2_PUBLIC_IP>:8080`  
- Grafana → `http://<EC2_PUBLIC_IP>:8081`  
- Prometheus → `http://<EC2_PUBLIC_IP>:8082`  

---


# Créditos

**Proyecto desarrollado por:**  
👤 **Richard Pérez**
👤 **Alejandro Gómez**
👤 **Luis Toscano**

**Asistencia técnica y documentación generada con:**  
🤖 *ChatGPT (OpenAI), como herramienta de apoyo para explicación, depuración y generación de documentación técnica.*

**Backend utilizado:**  
📦 https://github.com/devdbrandy/restful-ecommerce

**Tecnologías:**  
- Docker  
- Docker Compose  
- K3s  
- KIND  
- Kubernetes  
- Prometheus  
- Grafana  
- Node Exporter  
- Nginx Reverse Proxy  
- Node.js  
- PostgreSQL  
