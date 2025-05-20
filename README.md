Despliegue personalizado de AWX con manifestos para instalacion personalizada con kustomize

INSTALACIÓN PERSONALIZADA
Las fases del procedimiento completo para instalar AWX usando K3s en Distribución RHEL en versión 9, con enfoque personalizado.

Fase PREVIA: Desacople de la DB
DESACOPLE de la DB de POD a Servidor Independiente:

Configuración de Servidor de DB

1. Instalar PostgreSQL en el Host
Habilitar el repositorio PostgreSQL 15 (Por compatibilidad del Kubernetes de AWX):

sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
Deshabilitar el módulo PostgreSQL predeterminado:

sudo dnf -qy module disable postgresql
Instalar PostgreSQL 15:

- NOTA: Este se instala acorde a la versión del Pod de PostegreSQL de AWX

sudo dnf install -y postgresql15-server postgresql15-contrib
Inicializar la base de datos:

sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
Iniciar y habilitar el servicio PostgreSQL:

sudo systemctl enable --now postgresql-15
Configurar PostgreSQL para permitir conexiones externas:

Editar pg_hba.conf:

 - NOTA: Es necesario realizar una configuración optima en el Archivo de configuración de PostgreSQL "Tunning"

sudo vim /var/lib/pgsql/15/data/pg_hba.conf
Modificar:

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5 #scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 md5 #scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            md5 #scram-sha-256
host    replication     all             ::1/128                 md5 #scram-sha-256
 
# Allow connections from AWX host
host    all             all             [IP/rango]      md5
  ** [IP/rango]: Valor del servidor de la DB

Editar postgresql.conf:

sudo vim /var/lib/pgsql/15/data/postgresql.conf
Descomentar y establecer:

listen_addresses = '*'
Reiniciar el servicio PostgreSQL 15:

sudo systemctl restart postgresql-15
Configurar el firewall:

sudo firewall-cmd --add-service=postgresql --permanent
sudo firewall-cmd --reload
Crear un usuario y base de datos para AWX:

sudo su - postgres
/usr/pgsql-15/bin/psql
 
CREATE DATABASE awx;
CREATE USER awx WITH PASSWORD '[PASSWORD DE LA DB]';
ALTER USER awx WITH SUPERUSER;
  ** [PASSWORD DE LA DB]: Valor del servidor de la DB

Fase 1: Preparar el Sistema
Actualizar el sistema:

sudo yum -y update
sudo reboot
Instalar dependencias:

sudo yum -y install jq git make tar
SELinux en Modo Permisivo:

sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
cat /etc/selinux/config | grep SELINUX=
Habilitar puertos de Firewalld para, véase documentación aquí:

Aunque la documentación de K3S sugiere desactivarlo, asociado a la documentación habilite los puertos y zonas seguras descritas en la documentación

Protocol	Port	Source	Destination	Description
TCP	2379-2380	Servers	Servers	Required only for HA with embedded etcd
TCP	6443	Agents	Servers	K3s supervisor and Kubernetes API Server
UDP	8472	All nodes	All nodes	Required only for Flannel VXLAN
TCP	10250	All nodes	All nodes	Kubelet metrics
UDP	51820	All nodes	All nodes	Required only for Flannel Wireguard with IPv4
UDP	51821	All nodes	All nodes	Required only for Flannel Wireguard with IPv6
TCP	5001	All nodes	All nodes	Required only for embedded distributed registry (Spegel)
TCP	6443	All nodes	All nodes	Required only for embedded distributed registry (Spegel)
Protocol	Port	Description
Rango Red	10.42.0.0/16	Rango de red asignado a los Pods, configurado por defecto en K3s.
Rango Red	10.43.0.0/16	Rango de red asignado a los Servicios, configurado por defecto en K3s
Puertos:

firewall-cmd --permanent --add-port=[puerto]/[protocolo]
Zona Segura:

firewall-cmd --permanent --zone=trusted --add-source=[rango/xx]
** En caso de deshabilitar, hágase así:

sudo systemctl disable firewalld --now
Fase 2: Instalar K3s
Instalar K3s:

curl -sfL https://get.k3s.io | sudo bash -
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
Verificar el estado del servicio K3s:

systemctl status k3s.service
Fase 3: Configurar Kubectl
Crear el directorio de configuración de kubectl:

mkdir -p ~/.kube
Copiar el archivo de configuración de K3s:

sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
Exportar la variable de entorno KUBECONFIG:

export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
Verificar la configuración:

kubectl config view
kubectl get nodes
Fase 4: Implementar el Operador AWX
Instalar kustomize:

En la ruta /usr/local/bin instalar el binario de kustomize.

curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
Copiar el binario a /usr/local/bin/

sudo cp -pa kustomize /usr/local/bin/
Clonar el repositorio del operador AWX:

git clone https://github.com/ansible/awx-operator.git
Ir al directorio del operador AWX:

cd awx-operator/
Estructura de Directorios personalizada del Operador de AWX

[deploy_operator_awx]$ tree
.
├── cert
│   ├── awx.colcomercio.com.co.cer
│   ├── awx.colcomercio.com.co.key
│   ├── awx.colcomercio.com.co.csr
│   └── CAinterna.cer
├── config
│   ├── crd
│   │   ├── bases
│   │   │   ├── ...
│   │   └── ...
│   ├── default
│   │   ├── ...
│   ├── manager
│   │   ├── ...
│   └── rbac
│       ├── ...
├── awx.yaml
├── ingress.yaml
├── kustomization.yaml
└── service.yaml
 
7 directories, 36 files
Crear un espacio de nombres para AWX:

export NAMESPACE=awx
kubectl create ns ${NAMESPACE} --save-config


Personalización de directorio awx-operator.

Dado que la instalación es personalizada mediante kustomize, en la estructura del directorio 

Creación del kustomization.yaml

---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  # - github.com/ansible/awx-operator/config/default?ref=2.19.1
  - config/default
  - awx.yaml
  - service.yaml
  - ingress.yaml
 
# Set the image tags to match the git version from above
images:
  - name: creu2awxautomationprd.azurecr.io/awx-operator
    newTag: 2.19.1
 
# Specify a custom namespace in which to install AWX
namespace: awx
Creación del awx.yaml

---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: ClusterIP # nodeport
 
  projects_persistence: true
  projects_storage_access_mode: ReadWriteOnce
  projects_storage_size: 5Gi
 
  admin_password_secret: awx-admin-password
 
  postgres_configuration_secret: awx-postgres-configuration
  image: creu2awxautomationprd.azurecr.io/awx
  image_version: "24.6.1"
  image_pull_policy: IfNotPresent
  image_pull_secret: acrregistry
  redis_image: creu2awxautomationprd.azurecr.io/redis:7
  redis_image_version: "7"
 
  ee_images:
    - name: "AWX EE (latest)"
      image: "creu2awxautomationprd.azurecr.io/awx-ee:24.6.1"
    - name: "AWX EE Python (2.15)"
      image: "creu2awxautomationprd.azurecr.io/awx-builder-ee-corbeta:2.15_inv"
    - name: "AWX EE (2.15)"
      image: "creu2awxautomationprd.azurecr.io/awx-builder-ee-corbeta:2.15"
    - name: "AWX EE (2.12)"
      image: "creu2awxautomationprd.azurecr.io/awx-builder-ee-corbeta:2.12"
 
  control_plane_ee_image: creu2awxautomationprd.azurecr.io/awx-ee:24.6.1
  init_container_image: creu2awxautomationprd.azurecr.io/awx-ee
  init_container_image_version: 24.6.1
  init_projects_container_image: creu2awxautomationprd.azurecr.io/centos:stream9
 
  # Requerimientos de recursos para el servicio web de AWX
  web_resource_requirements:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
  
  # Requerimientos de recursos para las tareas de AWX
  task_resource_requirements:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "1"
      memory: "2Gi"
  
  # Requerimientos de recursos para el entorno de ejecución de AWX
  ee_resource_requirements:
    requests:
      cpu: "250m"
      memory: "512Mi"
    limits:
      cpu: "500m"
      memory: "1Gi"
  
  # Requerimientos de recursos para Redis de AWX
  redis_resource_requirements:
    requests:
      cpu: "50m"
      memory: "64Mi"
    limits:
      cpu: "150m"
      memory: "256Mi"
  
  # Requerimientos de recursos para RSYSLOG de AWX
  rsyslog_resource_requirements:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "250m"
      memory: "512Mi"
  ** [ee_images]: Corresponde a la creación de imágenes personalizadas según necesidades. URL  

  ** LDAPS (Seguro): Si desea configurar LDAP seguro LDAPS, siga los siguientes pasos. URL  

Creación del service.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: awx-service
  namespace: awx
spec:
  selector:
    app.kubernetes.io/name: awx-web
    app.kubernetes.io/component: awx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8052
  type: ClusterIP
Creación del ingress.yaml

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awx-ingress
  namespace: awx
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  tls:
    - hosts:
        - [url.com]
      secretName: awx-tls
  rules:
    - host: [url.com]
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: awx-service
                port:
                  number: 80
  ** [url.com]: Es el valor URL que se debe asignar en concordancia al DNS 

Configuración de los SECRETS:

Secret "awx-postgres-configuration" de la configuración del PostgreSQL

kubectl create secret generic awx-postgres-configuration --from-literal=host='[IP SERVIDOR POSTGRESQL]' --from-literal=port='[PUERTO]' --from-literal=database='[NOMBRE DB]' --from-literal=username='[USER DB]' --from-literal=password='[PASSWORD DB]' -n awx
Secret "acrregistry" del ACR (Si existe)

kubectl create secret docker-registry acrregistry --docker-server=[server] --docker-username=[username] --docker-password=[password] -n awx
Secret "awx-tls"
--cert: Es la concatenación del → ( Certificado de AWX + CA Interna )
--key: Es el certificado .key

kubectl create secret tls awx-tls --cert=cert/[nombre].crt --key=cert/[nombre].key -n awx
Secret "awx-admin-password"

kubectl create secret generic awx-admin-password --from-literal=password='[PASSWORD DE AWX]' -n awx
  ** [PASSWORD DE AWX]: Valor asignado al usuario admin de AWX

Establecer Imágenes al ACR (Si aplica):

Si el objetivo es realizar despliegue de AWX mediante imágenes (no publicas), se debe hacer el proceso respectivo de [ pull, tag, push ].
docker.io/library/redis:7
docker.io/rancher/klipper-lb:v0.4.9
docker.io/rancher/klipper-helm:v0.9.2-build20240828
docker.io/rancher/local-path-provisioner:v0.0.28
docker.io/rancher/mirrored-coredns-coredns:1.11.3
docker.io/rancher/mirrored-library-traefik:2.11.8
docker.io/rancher/mirrored-library-busybox:1.36.1
docker.io/rancher/mirrored-pause:3.6
gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0
quay.io/ansible/awx-ee:24.6.1
quay.io/ansible/awx-operator:2.19.1
quay.io/ansible/awx:24.6.1
registry.k8s.io/metrics-server/metrics-server:v0.7.2
quay.io/centos/centos:stream9
EJEMPLO:

Pull
podman pull quay.io/ansible/awx:24.6.1
Tag
podman tag quay.io/ansible/awx:24.6.1 creu2awxautomationprd.azurecr.io/awx:24.6.1
Push
podman push creu2awxautomationprd.azurecr.io/awx:24.6.1
Establecer ruta de Imágenes Podman/Docker:

Si el despliegue se va a realizar en base a las imágenes que reposan en el Container Independiente, se modifica en las rutas del awx-operator.

Archivo kustomization.yaml
quay.io/ansible/awx-operator:2.19.1            →            creu2awxautomationprd.azurecr.io/awx-operator:2.19.1
Archivo awx.yaml
quay.io/ansible/awx:24.6.1            →      creu2awxautomationprd.azurecr.io/awx:24.6.1
docker.io/library/redis:7                                  →           creu2awxautomationprd.azurecr.io/redis:7
quay.io/ansible/awx-ee:24.6.1         →      creu2awxautomationprd.azurecr.io/awx-ee:24.6.1
quay.io/ansible/awx-operator:2.19.1             →            creu2awxautomationprd.azurecr.io/awx-operator:2.19.1
quay.io/centos/centos:stream9                       →           creu2awxautomationprd.azurecr.io/centos:stream9
docker.io/library/redis:7                                   →          creu2awxautomationprd.azurecr.io/redis:7
Archivo config/manager/manager.yaml
En ImagePullSecrets:
- name: [NOMBRE DEL SECRET DEL ACR]


Archivo: default/manager_auth_proxy_patch.yaml
- Modificar ruta imagen: 
gcr.io/kubebuilder/kube-rbac-proxy:v0.15.0  →         creu2awxautomationprd.azurecr.io/kube-rbac-proxy:v0.15.0


Archivo config/manager/kustomization.yaml
quay.io/ansible/awx-operator:latest                →         creu2awxautomationprd.azurecr.io/awx-operator:2.19.1
Fase 5: Instalar AWX
Deploy mediante kustomize:

Una vez efectuado los puntos anteriores, se procede a realizar el Deploy mediante kustomize.

kustomize build . | kubectl apply -f -
Fase 6: Comprobar Registros del Contenedor AWX
Verificar el estado de la PVC:

kubectl get pvc -n awx
Monitorear los pods hasta que estén en estado Running:

kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator" -n awx -w
Verificar los registros de los despliegues:

kubectl -n awx logs deploy/awx-web
Listar todos los pods y sus contenedores:

kubectl get pod -n awx
kubectl -n awx get pod <pod_name> -o jsonpath='{.spec.containers[*].name}';echo
Fase 7: Acceso al Contenedor del Pod AWX
Acceder a los contenedores:

kubectl -n awx exec -ti deploy/awx-web -c redis -- /bin/bash
kubectl -n awx exec -ti deploy/awx-web -c awx-rsyslog -- /bin/bash
Fase 8: Acceso a la Interfaz Web de AWX
Obtener el puerto del servicio web de AWX:

kubectl get service -n awx
Identifica el puerto del nodo del servicio, por ejemplo:

NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
awx-service        NodePort   10.43.197.41    <none>        80:30614/TCP   16m
Acceder a la consola web de AWX:

http://<k3s-server-ip-address>:<port-number>
Obtener la contraseña de administrador:

kubectl -n awx get secret awx-admin-password -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
Ejemplo de salida:
password: ZMXFxKxPpDq10iZDyZNhY2cBLxs32Ohu

Iniciar sesión en la interfaz web de AWX: Usa el nombre de usuario admin y la contraseña obtenida.

Fase 9: Cambiar la Contraseña del Administrador
Obtener el nombre del pod de AWX:

kubectl get pods -n awx
Identifica el pod del servidor web de AWX.

Acceder al pod del servidor web de AWX:

kubectl exec -it <awx-web-pod-name> -n awx -- /bin/bash
Cambiar la contraseña usando awx-manage:
awx-manage changepassword admin

Se te pedirá que ingreses y confirmes la nueva contraseña.

Salir del pod:

exit
Fase 10: Validación Final
Verificar que los contextos están configurados correctamente:

kubectl config get-contexts
Verificar que puedes listar los pods en el namespace awx:

      imagePullSecrets:
      - name: redhat-operators-pull-secret
