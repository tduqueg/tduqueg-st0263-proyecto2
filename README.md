# Implementación de WordPress en GKE con Persistent Disk y Cloud SQL

Este instructivo describe cómo configurar WordPress en Google Kubernetes Engine (GKE) utilizando PersistentVolumes (PV) con Persistent Disk y una base de datos MySQL con Cloud SQL. Gran parte de esta guía fue basada en la guía ya existente de google: https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk?hl=es-419

## Autores

- Tomás Duque
- Juan Felipe Ortiz
- David Ruiz

## Pre-requisitos

- Una cuenta de Google Cloud con la facturación habilitada.
- Google Cloud SDK configurado con el proyecto de Google Cloud.
- `kubectl` configurado para comunicarse con tu clúster de GKE.
- Acceso a Cloud Shell o a una terminal local con acceso a tu proyecto de Google Cloud.

## Configuración

### Paso 1: Crear un clúster de GKE

1. En Cloud Shell, habilita las APIs necesarias:

```sh
   gcloud services enable container.googleapis.com sqladmin.googleapis.com
```

2. Configura la región predeterminada:

```sh
gcloud config set compute/region tu-region
```

3. Configura la variable de entorno PROJECT_ID como el ID de tu proyecto de Google Cloud (project-id).

```sh
export PROJECT_ID=project-id
```

Descarga los archivos de manifiesto de la app desde el repositorio de GitHub:

```sh
git clone https://github.com/tduqueg/tduqueg-st0263-proyecto2.git
```

4. Cambia al directorio con el archivo wordpress-persistent-disks:

```sh
cd tduqueg-st0263-proyecto2
```

5. Establece la variable de entorno `WORKING_DIR`:

```sh
WORKING_DIR=$(pwd)
```

6. Crea el clúster:

```sh
CLUSTER_NAME=nombre-de-tu-cluster
gcloud container clusters create-auto $CLUSTER_NAME
```

7. Obtén las credenciales para tu clúster:

```sh
gcloud container clusters get-credentials $CLUSTER_NAME --region tu-region
```

### Paso 2: Configurar PersistentVolumes y PersistentVolumeClaims

1. En Cloud Shell, implementa el archivo de manifiesto:

```sh
kubectl apply -f $WORKING_DIR/wordpress-volumeclaim.yaml
```

2. Verifica el estado del pv:

```sh
kubectl get persistentvolumeclaim
```

### Paso 3: Crear una instancia de Cloud SQL para MySQL

1. Crea una instancia de Cloud SQL:

```sh
INSTANCE_NAME=tu-instancia-mysql
gcloud sql instances create $INSTANCE_NAME
```

2. Establece las variables de entorno necesarias:

```sh
export INSTANCE_CONNECTION_NAME=$(gcloud sql instances describe nombre-instancia-mysql --format='value(connectionName)')
```

3. Crea una base de datos y un usuario para WordPress:

```sh
gcloud sql databases create wordpress --instance $INSTANCE_NAME
```

4. Crea un usuario de base de datos llamado wordpress y una contraseña para que WordPress se autentique en la instancia:

```sh
CLOUD_SQL_PASSWORD=$(openssl rand -base64 18)
gcloud sql users create wordpress --host=% --instance $INSTANCE_NAME --password $CLOUD_SQL_PASSWORD
```

### Paso 4: Implementar WordPress

1. Configura una cuenta de servicio para el proxy de Cloud SQL y crea los secretos necesarios en Kubernetes:

```sh
SA_NAME=cloudsql-proxy
gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME
```

2. Agrega la dirección de correo electrónico de la cuenta de servicio como una variable de entorno:

```sh
SA_EMAIL=$(gcloud iam service-accounts list \
    --filter=displayName:$SA_NAME \
    --format='value(email)')
```

3. Agrega la función cloudsql.client a tu cuenta de servicio:

```sh
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --role roles/cloudsql.client \
    --member serviceAccount:$SA_EMAIL
```

4. Crea una clave para la cuenta de servicio:

```sh
gcloud iam service-accounts keys create $WORKING_DIR/key.json \
    --iam-account $SA_EMAIL
```

Este comando descarga una copia del archivo key.json.

5. Crea un secreto de Kubernetes para las credenciales de MySQL:

```sh
kubectl create secret generic cloudsql-db-credentials \
    --from-literal username=wordpress \
    --from-literal password=$CLOUD_SQL_PASSWORD
```

6. Crea un secreto de Kubernetes para las credenciales de la cuenta de servicio:

```sh
kubectl create secret generic cloudsql-instance-credentials \
    --from-file $WORKING_DIR/key.json
```

### Paso 5: Exponer WordPress

1. Para preparar el archivo de implementación, reemplaza la variable de entorno `INSTANCE_CONNECTION_NAME`:

```sh
cat $WORKING_DIR/wordpress_cloudsql.yaml.template | envsubst > \
    $WORKING_DIR/wordpress_cloudsql.yaml
```

2. Implementa el archivo de manifiesto `wordpress_cloudsql.yaml`:

```sh
kubectl create -f $WORKING_DIR/wordpress_cloudsql.yaml
```

3. Mira la implementación para ver el cambio de estado a _running_:

```sh
kubectl get pod -l app=wordpress --watch
```

4. Crea un Service de type:LoadBalancer:

```sh
kubectl create -f $WORKING_DIR/wordpress-service.yaml
```

5. Mira la implementación y espera a que el servicio tenga asignada una dirección IP externa:

```sh
kubectl get svc -l app=wordpress --watch
```

### Paso 6: Configurar tu blog de WordPress

Accede a tu implementación de WordPress a través de la dirección IP externa asignada y completa la configuración inicial de tu blog.

### Limpieza

Para evitar costos adicionales, elimina los recursos que ya no necesites:

1. Borrar el clúster de GKE:

```sh
gcloud container clusters delete nombre-del-cluster
```

2. Elimina la instancia de Cloud SQL:

```sh
gcloud sql instances delete nombre-instancia-mysql
```

### Soporte

Si necesitas ayuda, puedes revisar la documentación oficial de GKE.
