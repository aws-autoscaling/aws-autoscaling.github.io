---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Auto Scaling Groups
description: Escalando automáticamente tus instancias EC2

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
        content: Introducción a AWS EC2
        url: '/ec2'
    next:
        content: Kubernetes Operations
        url: '/kops'
---

## **Auto Scaling Groups**

En primer lugar para proceder a las siguientes explicaciones, vamos a lanzar una instancia EC2, la cual vamos a pasarle un **user data** que viene a ser un pequeño script bash que asociamos a la instancia, para que en el arranque ejecute el código del interior del script. Consiguiendo así realizar cambios dentro de nuestra instancia al ser lanzada.

En este caso utilizaremos como imagen base del sistema, la imagen **Amazon Linux AMI**, y posteriormente vamos a instalarle un servidor web (apache) dentro. A partir de esta configuración, podremos guardar el sistema de la instancia EC2 como una AMI nuestra personalizada de la que partir, para proceder al auto escalado de nuestros servidores webs en este caso.

Primero crear el script que vamos a lanzar con el arranque de nuestra EC2:
```
#!/bin/bash
yum update -y
yum install -y httpd24 php56
service httpd start
chkconfig httpd on
```

Una vez tenemos el fichero de texto creado, procedemos a ejecutar la siguiente instrucción desde el mismo directorio:

```
aws ec2 run-instances --image-id ami-ca0135b3 --count 1 \
    --instance-type t2.micro \
    --key-name aws-charlie --subnet-id subnet-88c87aee \
    --security-group-ids sg-0d035270 \
    --user-data file://script.txt
```
Ahora si escribimos la dirección IP Pública que nos ha asignado la subnet pública a nuestra instancia y hemos configurado en los apartados anteriores la apertura del puerto 80 tcp a a dicha subnet. Deberíamos ser capaces de ver la página predeterminada del apache de la AMI **Amazon Linux**.

### **Launch Configuration**

Para poder auto escalar una aplicación en amazon, lo primero que necesitamos configurar es el **Launch Configuration** que no es más que una plantilla de como queremos que se levanten nuestras instancias en el Auto Scaling. Permitiéndonos así poder levantar nuevas instancias identicas, sin necesidad de tener que configurar manualmente una por una.

Así que para crear un launch configuration, primero necesitamos una AMI base, la cual vamos a utilizar la AMI de Amazon Linux donde hemos instalado el servidor apache en nuestra instancia EC2 anterior:

```
# aws ec2 create-image --instance-id i-0bdf383ea378fae47 \
    --name "Apache Server" --no-reboot
{
    "ImageId": "ami-1414176d"
}
```
 

<div class="callout callout--info">
<p><strong>Nota:</strong>El parametro --no-reboot, es básicamente para indicarle a aws que no es necesario reiniciar la instancia antes de proceder a crear la imagen. Este proceso de creación puede tardar varios minutos.</p>
</div>

Una vez este lista la AMI, procedemos a crear un **Launch Configuration**, es necesario crearlo antes, ya que el Auto Scaling Groups necesita de uno de estos.

A la hora de crearlo, indicarle la ami-id de la imagen creada antes:

```
aws autoscaling create-launch-configuration \
    --launch-configuration-name launch-web \
    --security-groups sg-0d035270 \
    --image-id ami-1414176d \
    --instance-type t2.micro \
    --key-name aws-charlie
```

<div class="callout callout--info">
<p><strong>Observación:</strong> A la hora de crear el "Launch Configuration" podemos indicarle como parametro la opción de "user data" por si quisiéramos que las nuevas instancias hicieran algo especial a la hora de iniciarse o realizar algún tipo de comprobación, como por ejemplo, descargar el código de la aplicación de un repositorio externo o algo similar.</p>
</div>

Ahora si que si procedemos a crear el **Auto Scaling Group** a partir del launch anterior, dsdsdsds:

```
aws autoscaling create-auto-scaling-group --auto-scaling-group-name "web-servers" \
    --launch-configuration-name "launch-web" \
    --min-size 1 \
    --max-size 2 \
    --desired-capacity 1 \
    --vpc-zone-identifier "subnet-88c87aee,subnet-2364e76b" \
    --tags "ResourceId=web-servers,ResourceType=auto-scaling-group,Key=Name,Value=web-server,PropagateAtLaunch=true"
```

Ya tendríamos creado nuestro primero ASG, pero si es verdad que tendríamos que cambiar los valores manualmente, para que el número de instancias dentro del **Auto Scaling Group** variase, ya que hemos definido que el mínimo de instancia que haya siempre levantada de ese ASG, es de una instancia, como estado deseado 1 y máximo 2. Pero la realidad es que nunca va a escalar a 2 a no ser que cambiemos nosotros este valor manualmente.

La idea siguiente es, la de que esto escale de manera automática pudiéndose basar en algún tipo de información como por ejemplo dependiendo de el % de uso de CPU de la instancia, si esta se ve comprometida a más del 70% se vaya levantando paralelamente una segunda instancia. También es posible añadir una notificación y que nos avise a nuestro correo en caso de que se dispare el auto escalado de nuestro **ASG**.

### **Actualizar el Auto Scaling Group**

Imaginemos que necesitamos cambiar algo del código o algún paquete en el sistema que estamos utilizando como base en nuestra AMI que utiliza el Launch Configuration.

Debemos realizar los siguientes pasos, vamos a partir con que nuestro servidor apache que corre en las instancias EC2 de nuestro Auto Sscaling Group, esta utilizando la página predeterminada. La idea es cambiar esto y que las nuevas instancias que se levanten en el sistema ya tengan la nueva plantilla html.

Los pasos a realizar serian los siguientes:

* Levantar o conectarnos a una instancia con nuestra AMI personalizada y realizar los cambios.
* Crear una nueva AMI a partir de esta instancia.
* Modificar el Launch Configuration, para que utilice la nueva AMI personalizada.

En mi caso voy a utilizar la instancia EC2 levantada con la AMI personalizada con apache y php, vamos a modificar el **index.html** y reiniciamos el servicio:

```
ssh -i "aws-charlie.pem" ec2-user@34.244.137.253
[root@ip-10-0-1-45 ec2-user]# echo "Esto es una página de prueba" > /var/www/html/index.html
[root@ip-10-0-1-45 ec2-user]# service httpd restart
```

Crear la AMI a partir de la instancia modificada:

```
aws ec2 create-image --instance-id i-08336f4d0db03ef76 \
    --name "Apache Server v2" \
    --no-reboot
{
    "ImageId": "ami-911e1de8"
}
```

El siguiente paso es crear un nuevo **Launch Configuration** que utilice la nueva AMI:
```
aws autoscaling create-launch-configuration \
    --launch-configuration-name launch-web-v2 \
    --security-groups sg-0d035270 \
    --image-id ami-911e1de8 \
    --instance-type t2.micro \
    --key-name aws-charlie
```
Una vez creado, vamos actualizar el **Auto Scaling Group** para que utilice el nuevo **Launch Configuration**:
```
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name "web-servers" \
    --launch-configuration-name "launch-web-v2"
```
Y por último eliminamos el primer **Launch Configuration** para evitar equivocaciones:
```
aws autoscaling delete-launch-configuration \
    --launch-configuration-name "launch-web"
```

A partir de ahora  todas las instancias nuevas que levantarán el **Auto Scaling Group** tendrían como imagen base la nueva AMI con la modificación en el fichero html. Para llevar a cabo esto y sin perder la disponibilidad de la primera instancia, necesitamos cambiar el número deseado de instancias en el ASG.

En el caso que se esta siguiendo en este proyecto, fijamos los valores de la siguiente forma:
* Min: 1
* Desired: 1
* Max: 2

Solamente tendríamos que cambiar el valor de **Desired** a 2, y se nos lanzaría automáticamente una nueva instancia EC2 con la nueva plantilla:

```
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name "web-servers" \
    --desired-capacity 2
```

Si volvemos a cambiar el valor anteriores a 1, automáticamente se borrará la instancia con la AMI antigua:
```
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name "web-servers" \
    --desired-capacity 1
```

Si listamos los tags de la nueva instancia EC2, veremos que esta utilizando la AMI nueva que era **ami-911e1de8**:
```
aws ec2 describe-instances --instance-id i-09cd718886fa5a4b1 \
    --query 'Reservations[*].Instances[*].[ImageId,Tags[*]]'
[
    [
        [
            "ami-911e1de8", 
            [
                {
                    "Value": "web-servers", 
                    "Key": "aws:autoscaling:groupName"
                }, 
                {
                    "Value": "web-server", 
                    "Key": "Name"
                }
            ]
        ]
    ]
```

Bueno y después de ver esto, creo que viene una pregunta lógica y es ¿y si tengo mi aplicación web corriendo en más de 1 instancia?, ¿qué tengo que introduccir la dirección IP Pública de todas las instancias manualmente?.

Pues no, para eso existe el servicio **Elastic Load Balancer**, que nos permitirá integrar un balanceador de carga en el frontal de nuestras instancias EC2, redireccionando las peticiones a nuestros servidores web y balanceando la carga de manera transparente a nosotros. Y al fin y al cabo solamente utilizar un **endpoint** para acceder a nuestra aplicación.


### **ELB**

Como hemos dicho necesitamos obligatoriamente tener un balanceador de carga cuando disponemos de más de 1 instancia en nuestro **Auto Scaling Group**.

Para empezar es necesario crear un **security group**, si el balanceador de carga también necesita tener asociado este concepto dentro de la VPC, para definir posteriormente los puertos que estará escuchando. Y finalmente seleccionar a que puertos de las instancias va a redireccionar esas peticiones provinientes del puerto que esta escuchando.

Crear el security group:
```
aws ec2 create-security-group --group-name "SG-ELB" \
    --description "SG para nuestro ELB" \
    --vpc-id "vpc-fd6f139b"
{
    "GroupId": "sg-15dd8e68"
}
```

Permitir a cualquier direccionamiento acceder al puerto 80 en este **security group**, donde ubicaremos el ELB que vamos a crear:
```
aws ec2 authorize-security-group-ingress --group-id "sg-15dd8e68" \
    --protocol tcp --port 80 --cidr 0.0.0.0/0
```
A día de hoy hay varios tipos de balanceadores de carga, cada uno especializado en un aspecto. Para esta introducción vamos a crear el balanceador de carga clásico.

En el siguiente ejemplo, redireccionariamos el tráfico entrante al puerto 80 del **ELB** hacia el puerto 80 de las instancias **EC2**:
```
aws elb create-load-balancer --load-balancer-name "elb-web" \
    --listeners "Protocol=HTTP,LoadBalancerPort=80,InstanceProtocol=HTTP,InstancePort=80" \
    --subnets "subnet-88c87aee" "subnet-2364e76b" \
    --security-groups sg-15dd8e68
{
    "DNSName": "elb-web-2079783846.eu-west-1.elb.amazonaws.com"
}
```
<div class="callout callout--info">
<p><strong>Importante:</strong> Introducir las dos subnets públicas como mínimo para poder aprovechar la alta disponibilidad en el balanceador de carga.</p>
</div>

Seguidamente debemos asociar el **balanceador de carga** con nuestro **Auto Scaling Group**:

```
aws autoscaling attach-load-balancers \
    --auto-scaling-group-name "web-servers" \
    --load-balancer-names "elb-web"
```
<div class="callout callout--info">
<p><strong>Observación:</strong> El balanceador de carga, tiene otra opción configurable y es el número de comprobaciones que tiene que hacer a las instancias durante X intervalos, para añadir dicha instancia como "saludable" para que el balanceador considere que puede redireccionar el tráfico hacia dicha instancia.</p>
</div>

Por este motivo, tardará unos minutos en que el **ELB** registre nuestras instancias como saludables y empiece a balancear el tráfico hacia ellas.

Finalmente si accedemos desde una navegador a la URL del balanceador de carga que nos proporciona AWS, en mi caso ```elb-web-2079783846.eu-west-1.elb.amazonaws.com``` podremos visualizar el contenido de nuestros servidores web.


### **Cloudwatch**

Es el servicio encargado de recopilar métricas y estadisticas del resto de servicios. Como por ejemplo, con sumo de cpu, latencia, métricas relacionadas con el balanceador de carga, todas estas métricas son importadas a CloudWatch. 

Una parte buena es la posibilidad de crear alertas que salten cuando se cumpla una condición sobre una métricas predeterminada. En relación con nuestro **Auto Scaling Group** podemos agregar un **disparador** para que cuando la media de uso de cpu de las instancias que se encuentren dentro del ASG se dispare por encima de un **porcentaje** determinado, automáticamente se levanten **más** instancias y en periodos de una carga baja de cpu escalemos hacía abajo en número de instancias.

Bien desde que para visualizar las métricas, necesitamos visualizar gráficos nos olvidamos de momento por la línea de comandos y obviamente tenemos que acceder a la consola de AWS. Por otro lado aunque **CloudWatch** dispone de su propio apartado para crear **dashboards** y personalizarlos con métricas de los diferentes servicios que estemos utilizando a nuestro gusto. Desde el apartado de **EC2**, si marcamos una instancia en concreto, en la pestaña **Monitoring** podemos ver una preview de algunas métricas de la instancia en cuestión.

<p align="center">
    <img src="/images/Figura6.png" alt="Monitoring Preview Instancia EC2">
</p>
<p align="center">
    <b>Figura 6 - previa Métricas de Monitorización</b>
</p>

<div class="callout callout--info">
<p><strong>Nota:</strong> Si os fijáis en la imagen anterior, tengo habilitado la opción **Detailed monitoring**. Esto simplemente graba y presenta los datos de la métrica, pudiéndose representar cada minuto y no cada 5 minutos que es como esta configurado de manera predeterminada.</p>
</div>

Al igual que en EC2 tenemos una previa de las métricas, en el apartado del **ELB** también disponemos de esa característica, donde se refleja la latencia, el número de peticiones, o el número de peticiones de un código http como (2XXs,4XXs y 5XXs). O el número de instancias saludables que tenemos detrás del balanceador para poder recibir peticiones.

<p align="center">
    <img src="/images/Figura7.png" alt="Monitoring Preview ELB">
</p>
<p align="center">
    <b>Figura 7 - Métricas de Monitorización del ELB</b>
</p>

Por supuesto desde **CloudWatch** tenemos acceso a algunas métricas que no están disponibles desde el panel de EC2 o ELB.

Pero si es cierto, que hay dos métricas a mi parecer, que no nos proporciona **CloudWatch** sobre nuestras instancias EC2 y son el **uso de memoria ram** y el **espacio en disco**. Aún así, podemos hacer uso de unos scripts programados en el lenguaje **perl** para que podamos recopilar esta información y mandársela a CloudWatch.

Estos scripts son compatibles tanto con la AMI de Amazon Linux, como de sistemas operativos como SUSE,Ubuntu o Red Hat.

En el siguiente enlace viene explicado lo necesitario para instalar dichos scripts:

[https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/mon-scripts.html](https://docs.aws.amazon.com/es_es/AWSEC2/latest/UserGuide/mon-scripts.html)

A continuación os dejo un video, donde hago uso de los servicios citados anteriormente en el proyecto y se puede ver el uso de un **Auto Scaling Group** y ver como automáticamente se levanta una nueva instancia exacta de la ya existente cuando el **uso de cpu de la instancia se eleva a un % determinado**.

### **Video Demostrativo**

[![Auto Scaling Group en AWS](http://img.youtube.com/vi/1rfjP2PjWpU/0.jpg)](https://www.youtube.com/watch?v=1rfjP2PjWpU "Auto Scaling Group en AWS")