---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Introducción a AWS
description: Un breve repaso teoríco y práctico desde la línea de comandos de AWS.

# Author box
#author:
#    title: About Author
#    title_url: '#'
#    external_url: true
#    description: Author description

# Micro navigation
micro_nav: true

# Page navigation
page_nav:
    prev:
        content: Página Principal
        url: '/'
    next:
        content: Introducción a VPC
        url: '/vpc'
---

## **INTRODUCCIÓN A AWS**

Lo primero e imprescindible es tener una cuenta en AWS, Amazon te ofrece una capa gratuita el primer año después de haberte registrado, dispones de un tipo de instancia EC2, donde si no superas un máximo de 700 horas con la instancia encendida al mes, es `gratis`.

Por otro lado, también te dan de manera gratuita un máximo de 20.000 peticiones (gets) y 2.000 peticiones (puts) al servicio S3 con un máximo de almacenamiento en dicho servicio de 5 GiB, hasta 30 GiB de espacio en volúmenes con su servicio EBS, 750 horas de uso y 15 GiB de procesamiento de datos por el servicio ELB. Todo esto mensualmente durante el primer año desde la inscripción, donde es más que suficiente para aprender a utilizar los componentes principales.

Por lo tanto, el primer paso es registrarnos en AWS, en el proceso nos pedirán una tarjeta de crédito por si nos pasáramos en alguno de los límites de la capa “Free Tier”, hacernos el cargo correspondiente al
uso realizado en la plataforma.

Una vez registrados, ya podremos acceder a los servicios desde la consola de administración de aws con dicha cuenta, pero si queremos hacer uso de AWS CLI necesitamos generar unas Access Keys para realizarle las correspondientes llamadas a la API de AWS.


### **CREANDO UN USUARIO PARA AWS CLI**

Si no queremos buscar el servicio “IAM” en el panel frontal, podemos simplemente escribirlo en el buscador:

<p align="center">
    <img src="/images/Figura1.png" alt="Buscando el servicio IAM en el buscador">
</p>
<p align="center">
    <b>Figura 1 - Buscando el servicio IAM en el buscador</b>
</p>

Seguidamente, en la barra lateral izquierda, seleccionar en “Users” y luego a “Add User”:

<p align="center">
    <img src="/images/Figura2.png" alt="Accediendo a la ventana de Usuarios">
</p>
<p align="center">
    <b>Figura 2 - Accediendo a la ventana de Usuarios</b>
</p>

En el siguiente paso introducir el nombre para el nuevo usuario y seleccionar que dicho usuario sea del tipo “Programmatic access”:

<p align="center">
    <img src="/images/Figura3.png" alt="Creando un usuario con acceso programatico">
</p>
<p align="center">
    <b>Figura 3 - Creando un usuario con acceso programatico</b>
</p>

Luego importante, adjuntarle al usuario los permisos correspondientes, en este caso vamos a seleccionar una “policy” que tiene permiso absoluto sobre todos los servicios de AWS, pero esto no es una buena práctica, de todas formas, para el apartado de Introducción a los componentes vamos a dejarlo así,pero a la hora de crear un usuario para el despliegue de Kubernetes, si haremos esto correctamente.

<p align="center">
    <img src="/images/Figura4.png" alt="Asociando una politica al usuario IAM">
</p>
<p align="center">
    <b>Figura 4 - Asociando una politica al usuario IAM</b>
</p>

En la siguiente pantalla nos aparecerá una preview del usuario que vamos a crear, si todo está bien, procedemos a la creación del usuario.

<p align="center">
    <img src="/images/Figura5.png" alt="Claves de acceso del usuario IAM">
</p>
<p align="center">
    <b>Figura 5 - Claves de acceso del usuario IAM</b>
</p>

<p align="center">
    <img src="https://charliejsanchez.com/wp-content/uploads/2018/05/ProyectoAWS-5.png" alt="Access Keys IAM">
</p>

<div class="callout callout--info">
<p><strong>IMPORTANTE</strong> Anotar la Access Secret Key, ya que está, no será accesible posteriormente a su creación.</p>
</div>


## **LÍNEA DE COMANDOS AWS**

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