---
layout: post
title: Introduction
---

<div id='id1' />

### **1. INTRODUCCIÓN A AWS**

Lo primero e imprescindible es tener una cuenta en AWS, Amazon te ofrece una capa gratuita el primer año después de haberte registrado, dispones de un tipo de instancia EC2, donde si no superas un máximo de 700 horas con la instancia encendida al mes, es `gratis`.

Por otro lado, también te dan de manera gratuita un máximo de 20.000 peticiones (gets) y 2.000 peticiones (puts) al servicio S3 con un máximo de almacenamiento en dicho servicio de 5 GiB, hasta 30 GiB de espacio en volúmenes con su servicio EBS, 750 horas de uso y 15 GiB de procesamiento de datos por el servicio ELB. Todo esto mensualmente durante el primer año desde la inscripción, donde es más que suficiente para aprender a utilizar los componentes principales.

Por lo tanto, el primer paso es registrarnos en AWS, en el proceso nos pedirán una tarjeta de crédito por si nos pasáramos en alguno de los límites de la capa “Free Tier”, hacernos el cargo correspondiente al
uso realizado en la plataforma.

Una vez registrados, ya podremos acceder a los servicios desde la consola de administración de aws con dicha cuenta, pero si queremos hacer uso de AWS CLI necesitamos generar unas Access Keys para realizarle las correspondientes llamadas a la API de AWS.

<div id='id2' />

### **1.1. CREANDO UN USUARIO PARA AWS CLI**

Si no queremos buscar el servicio “IAM” en el panel frontal, podemos simplemente escribirlo en el buscador:

<p align="center">
    <img src="https://charliejsanchez.com/wp-content/uploads/2018/05/ProyectoAWS-1.png" alt="New User With IAM">
</p>
Seguidamente, en la barra lateral izquierda, seleccionar en “Users” y luego a “Add User”:

<p align="center">
    <img src="https://charliejsanchez.com/wp-content/uploads/2018/05/ProyectoAWS-2.png" alt="Add User With IAM">
</p>

En el siguiente paso introducir el nombre para el nuevo usuario y seleccionar que dicho usuario sea del tipo “Programmatic access”:

<p align="center">
    <img src="https://charliejsanchez.com/wp-content/uploads/2018/05/ProyectoAWS-3.png" alt="Programmatic Access IAM">
</p>

Luego importante, adjuntarle al usuario los permisos correspondientes, en este caso vamos a seleccionar una “policy”que tiene permiso absoluto sobre todos los servicios de AWS, pero esto no es una buena práctica, de todas formas, para el apartado de Introducción a los componentes vamos a dejarlo así,pero a la hora de crear un usuario para el despliegue de Kubernetes, si haremos esto correctamente.

<p align="center">
    <img src="https://charliejsanchez.com/wp-content/uploads/2018/05/ProyectoAWS-4.png" alt="Attach policy IAM">
</p>

En la siguiente pantalla nos aparecerá una preview del usuario que vamos a crear, si todo está bien, procedemos a la creación del usuario.

<p align="center">
    <img src="https://charliejsanchez.com/wp-content/uploads/2018/05/ProyectoAWS-5.png" alt="Access Keys IAM">
</p>

> **IMPORTANTE**: Anotar la Access Secret Key, ya que está, no será accesible posteriormente a su creación.


<div id='id3' />

### **1.2. LÍNEA DE COMANDOS AWS**

La utilización de la línea de comandos de AWS está disponible tanto para [Windows](https://docs.aws.amazon.com/es_es/cli/latest/userguide/awscli-install-windows.html), [macOS](https://docs.aws.amazon.com/es_es/cli/latest/userguide/cli-install-macos.html) y [Linux](https://docs.aws.amazon.com/es_es/cli/latest/userguide/awscli-install-linux.html). En este último existen varias maneras de instalarlo, la página web oficial explica la instalación con pip, las imágenes de Amazon Linux por ejemplo ya llevan instaladas la CLI de AWS.

Por otro lado, si utilizamos un sistema operativo como `Debian`, podemos instalar directamente el paquete desde el repositorio oficial de la distribución:

~~~~~~~~
# apt install awscli
~~~~~~~~

Posteriormente lanzamos la siguiente instrucción, que lo que hará será pedirnos las Access Keys, la región y el formato de salida de las instrucciones que ejecutemos con AWS CLI. Esto creará en segundo plano, un directorio en el home del usuario “.aws” con dos ficheros que albergarán la información que
le vamos a pasar, denominados “config” y “credentials”.

```
# aws configure
AWS Access Key ID [****************YRRA]: AKIAIMERB67B56OERWGQ
AWS Secret Access Key [****************L1vW]: XXXXXXXXXXXXXXXX
Default region name [eu-west-1]: eu-west-1
Default output format [None]: json
```

Si mostramos el contenido de los ficheros que se han nombrado antes, vemos como no era mentira. Pero cabe destacar que, si utilizamos variables de entorno para definir estos valores, se utilizaran estos antes que los definidos en los ficheros:

```
# cat ~/.aws/credentials
[default]
aws_secret_access_key = XXXXXXXXXXXXXX
aws_access_key_id = AKIAIMERB67B56OERWGQ
```
Y:

```
~ cat ~/.aws/config
[default]
region = eu-west-1
output = json
```

Si ahora probáramos a listar las instancias EC2 que tenemos lanzadas nos debería devolver la siguiente salida:

```
# aws ec2 describe-instances
{
   Reservations": []
}
```

<div id='id4' />

### **1.3. Redes - VPC**  
![VPC Image](https://raw.githubusercontent.com/carlosjsanch3z/proyectoaws/master/assets/img/VPC.png)

La traducción de las siglas corresponde a “Virtual Private Cloud”, lo que podríamos decir que este componente simula una red privada o como si fuera nuestra propia red dentro de un centro de datos. Sería un segmento de red aislado de las otras VPC’s que tuviéramos creadas.

Por lo que cabe deducir es inevitable tener al menos 1 VPC en nuestra cuenta de AWS y tal es así, que
por defecto nos trae una VPC creada. Nosotros mismos podemos escoger el direccionamiento de red privado hasta un máximo de un CIDR /16, dentro de la VPC podemos segmentar y crear subnets según nuestra necesidad.

Las subnets pueden ser privada o públicas, por ejemplo, supongamos que nuestra aplicación tiene una parte web y otra con la base de datos. Para mayor seguridad y mejor práctica se colocaría la base de datos en una subnet privada, donde solo tendría permiso de acceso las instancias que se crearán en una subnet publica en concreto y prohibiendo el acceso remoto desde fuera hacia la base de datos. A estas últimas instancias que se levantarán dentro de la subnet pública se les repartiría de forma automática direccionamiento IP público.

Otro detalle es que las VPC se ubican dentro de una región y las subnets se localizan dentro de una zona
de disponibilidad. Se pueden crear subnets siempre y cuando no superemos el límite del direccionamiento de red elegido a
la hora de crear la VPC, por lo tanto, es algo bastante importante ya que una mala elección nos limitaría en el futuro. Como hemos comentado antes, tienen un alcance zonal, lo que quiere decir es que, si hemos creado nuestra VPC en la región de Irlanda, la subnet se deberá crear en una de las zonas de disponibilidad disponibles en esta región. Una subnet solo puede ubicarse en una zona de disponibilidad y esto no puede ser modificado después de su creación.

**A crear:** 1 VPC con 2 Subnets públicas y 2 Subnets privadas en dos AZ diferentes:
<p align="center">
    <img src="https://raw.githubusercontent.com/carlosjsanch3z/proyectoaws/master/assets/img/VPC%26Subnets.png" alt="Diagrama VPC - Subnets">
</p>

Partiendo del diagrama anterior, vamos a proceder a su creación desde la línea de comandos:

Primero crear la VPC:

```
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --output json
```
Resultado:
```
{
    "Vpc": {
        "VpcId": "vpc-fd6f139b", 
        "InstanceTenancy": "default", 
        "Tags": [], 
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-8379e7e8", 
                "CidrBlock": "10.0.0.0/16", 
                "CidrBlockState": {
                    "State": "associated"
                }
            }
        ], 
        "Ipv6CidrBlockAssociationSet": [], 
        "State": "pending", 
        "DhcpOptionsId": "dopt-e9668e8f", 
        "CidrBlock": "10.0.0.0/16", 
        "IsDefault": false
    }
}
```
A continuación podemos etiquetar la VPC con un nombre, para identificarla más facilmente:

```
aws ec2 create-tags --resources "vpc-fd6f139b" \
    --tags Key=Name,Value="proyectoaws"
```

Ahora necesitamos crear las subnets:

**Subnet - public1:**

```
# aws ec2 create-subnet --cidr-block "10.0.1.0/24" \
    --availability-zone "eu-west-1a" --vpc-id "vpc-fd6f139b"
{
    "Subnet": {
        ...
        "AvailabilityZone": "eu-west-1a", 
        "VpcId": "vpc-fd6f139b",  
        "SubnetId": "subnet-88c87aee", 
        ...
    }
}
```
Etiquetamos dicha subnet con su nombre, cogiendo el id de la Subnet:
```
# aws ec2 create-tags --resources "subnet-88c87aee" \
    --tags Key=Name,Value="public1"
```
**Subnet - public2:**

En la subnet public2 debemos cambiar la zona disponibilidad, el direccionamiento de red y etiquetarla con su nombre correspondiente:

```
# aws ec2 create-subnet --cidr-block "10.0.2.0/24" \ 
    --availability-zone "eu-west-1b" --vpc-id "vpc-fd6f139b"
{
    "Subnet": {
        ...
        "AvailabilityZone": "eu-west-1b", 
        "VpcId": "vpc-fd6f139b",
        "SubnetId": "subnet-2364e76b", 
        ...
    }
}
# aws ec2 create-tags --resources "subnet-2364e76b" \
    --tags Key=Name,Value="public2"
```

**Subnet - privada1:**

```
aws ec2 create-subnet --cidr-block "10.0.3.0/24" \ 
    --availability-zone "eu-west-1a" --vpc-id "vpc-fd6f139b"

{
    "Subnet": {
        ...
        "AvailabilityZone": "eu-west-1a", , 
        "VpcId": "vpc-fd6f139b",  
        "SubnetId": "subnet-6db1030b", 
        ...
    }
}

aws ec2 create-tags --resources "subnet-6db1030b" \
    --tags Key=Name,Value="private1"
```
**Subnet - privada2:**

```
aws ec2 create-subnet --cidr-block "10.0.4.0/24" \ 
    --availability-zone "eu-west-1b" --vpc-id "vpc-fd6f139b"

{
    "Subnet": {
        ...
        "AvailabilityZone": "eu-west-1b", , 
        "VpcId": "vpc-fd6f139b",  
        "SubnetId": "subnet-4a61e202", 
        ...
    }
}

aws ec2 create-tags --resources "subnet-4a61e202" \
    --tags Key=Name,Value="private2"
```

Ahora para que realmente las subnets públicas, sean públicas de verdad, tendremos que habilitarles a ambas la opción de que automaticamente a los recursos que se creen en la subnet, se le asignen IP's públicas:

```
aws ec2 modify-subnet-attribute \
--subnet-id "subnet-88c87aee" --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
--subnet-id "subnet-2364e76b" --map-public-ip-on-launch
```

## **CONTINUARÁ**
