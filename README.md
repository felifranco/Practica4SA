# Kubernetes On-premises

## AWS

Se deben tener instaladas las herramientas de `AWS CLI` en el equipo. Se puede verificar con el siguiente comando:

```shell
$ aws --version
aws-cli/2.15.2 Python/3.12.2 Linux/6.7.9-200.fc39.x86_64 source/x86_64.fedora.39 prompt/off
```

### [Usuario kops de IAM](https://kops.sigs.k8s.io/getting_started/aws/#setup-iam-user)

Crear el usuario `kops` en IAM con los siguientes permisos:

```
AmazonEC2FullAccess
AmazonRoute53FullAccess
AmazonS3FullAccess
IAMFullAccess
AmazonVPCFullAccess
AmazonSQSFullAccess
AmazonEventBridgeFullAccess
```

Luego de creado el usuario se debe crear también su clave de acceso para la _Interfaz de línea de comandos (CLI)_ y descargarla en algún lugar seguro; configuraremos éste usuario en el `AWS CLI` instalado en nuestro equipo, lo único a lo que podemos acceder es a lo permitido en el listado anterior. Iniciamos la configuración de AWS CLI con el siguiente comando:

```shell
aws configure
```

Ingresamos la información solicitada

```shell
AWS Access Key ID [None]: <access_key_id>
AWS Secret Access Key [None]: <secret_access_key>
Default region name [None]: <region_id>
Default output format [None]:
```

comprobamos que todo se encuentre correcto ejecutando algún comando, el siguiente es un ejemplo:

```shell
aws route53 list-hosted-zones
```

### Configure DNS

Crear una zona alojada privada en `Route 53 > Hosted zones`

### [Crear buckets en S3](https://kops.sigs.k8s.io/getting_started/aws/#cluster-state-store)

Según la [documentación](https://kops.sigs.k8s.io/getting_started/aws/#cluster-state-store), en S3 se almacena el estado del clúster por lo que será necesario tener un bucket dedicado para kops: _In order to store the state of your cluster, and the representation of your cluster, we need to create a dedicated S3 bucket for kops to use_. Se recomienda crear el bucket en la misma región:

```shell
aws s3api create-bucket \
    --bucket prefix-example-com-state-store \
    --region us-east-1
```

Se recomienda versionar el bucket S3 en caso de revertir cambios y regresar a una versión estable. Para este caso no se versionará:

```shell
aws s3api put-bucket-versioning --bucket prefix-example-com-state-store  --versioning-configuration Status=Enabled
```

Creamos otros tres buckets en donde se almacenará otra información, se recomienda utilizar buckets distintos al anterior:

```shell
aws s3api create-bucket \
    --bucket prefix-example-com-oidc-store \
    --region us-east-1 \
    --object-ownership BucketOwnerPreferred
aws s3api put-public-access-block \
    --bucket prefix-example-com-oidc-store \
    --public-access-block-configuration BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
aws s3api put-bucket-acl \
    --bucket prefix-example-com-oidc-store \
    --acl public-read
```

## Kops

Instalación para Linux

```shell
curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops
sudo mv kops /usr/local/bin/kops
```

## Configurar clúster

Mostrar las zonas habilitadas para nuestra región

```shell
aws ec2 describe-availability-zones --region us-east-1
```

Definir las siguientes variables que se utilizarán para crear la configuración del clúster:

```shell
export NAME=myfirstcluster.k8s.local
export KOPS_STATE_STORE=s3://prefix-example-com-state-store
```

elejir una zona de disponibilidad y ejecutar el siguiente comando para crear la configuración

```shell
kops create cluster \
    --name=${NAME} \
    --cloud=aws \
    --zones=us-east-1a \
    --discovery-store=s3://prefix-example-com-oidc-store/${NAME}/discovery
```

Al finalizar la ejecución del comando se muestra información útil como la siguiente:

```shell
Suggestions:
 * list clusters with: kops get cluster
 * edit this cluster with: kops edit cluster myfirstcluster.k8s.local
 * edit your node instance group: kops edit ig --name=myfirstcluster.k8s.local nodes-us-east-1a
 * edit your control-plane instance group: kops edit ig --name=myfirstcluster.k8s.local control-plane-us-east-1a

Finally configure your cluster with: kops update cluster --name myfirstcluster.k8s.local --yes --admin
```

Si queremos editar la configuración del cluster el siguiente comando desplegará el archivo YAML con la configuración actual el cuál podemos editar:

```shell
kops edit cluster --name ${NAME}
```

Editaremos el `node instance group`, al archivo yaml que se despliega el cambiaremos el atributo `machineType: t3.medium` a `machineType: t2.micro` porque `t2.micro` está dentro la capa gratuita y es lo que necesitamos para este momento. Configurar el que se desee.

```shell
kops edit ig --name=myfirstcluster.k8s.local nodes-us-east-1a
```

## Crear clúster

para crear el clúster ejecutamos el siguiente comando:

```shell
kops update cluster --name myfirstcluster.k8s.local --yes --admin
```

## Probar el clúster

Para probar el clúster se debe tener instalado `kubectl`, esta herramienta será útil también para trabajar con Kubernetes:

```shell
kubectl get nodes
```

El nombre de los nodos es el ID de la instancia EC2, eso quiere decir que el clúster se encuentra funcionando. También se puede utilizar `kops` para validar que el clúster se encuentra funcionando correctamente:

```shell
kops validate cluster --wait 10m
```

Y finalmente con el siguiente comando se pueden ver todos los componentes del sistema:

```shell
kubectl -n kube-system get po
```

## Eliminar el clúster

Utilizaremos el siguiente comando para ver todos los recursos que se eliminarán junto al clúster

```shell
kops delete cluster --name ${NAME}
```

Si se está de acuerdo entonces ejecutar:

```shell
kops delete cluster --name ${NAME} --yes
```

# NOTAS

```shell

```

```shell

```

# Referencias

- [kOps - Kubernetes Operations - Installing](https://kops.sigs.k8s.io/getting_started/install/)
- [Kubernetes Installers for Cloud and Bare Metal](https://medium.com/@m.k.joerg/overview-of-kubernetes-installers-8f06437d215a)
- [Interfaz de línea de comandos de AWS](https://aws.amazon.com/es/cli/)
- [Install or update to the latest version of the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#cliv2-linux-install)
- [Getting Started with kOps on AWS](https://kops.sigs.k8s.io/getting_started/aws/)
- [How to install Kubernetes in AWS using KOPS - Part 1](https://www.youtube.com/watch?v=0wB0MfYRBA4)
- []()
