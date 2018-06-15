---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Introducción a EC2
description: Conoce mejor el servicio de computing y maneja tus instancias

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
        content: Introducción a VPC
        url: '/vpc'
    next:
        content: Auto Scaling Groups
        url: '/asg'
---


## **COMPUTING - EC2**

### **SECURITY GROUPS**

Un security group o grupo de seguridad no es más que un firewall lógico que nos proporciona AWS, para poder controlar los permisos a la hora de interconectar las instancias o servicios independientemente de la subnet que se encuentren. Por ejemplo como se vió en el apartado anterior de **Networking** la idea es que las instancias públicas sean las únicas que puedan acceder al puerto **3306** donde se encuentra escuchando un servidor mysql. El caso sería que el **Security Group** de donde se encuentra la base de datos, tendría que permitir la entrada de tráfico a ese puerto a los **grupos de seguridad** asociados a las instancias públicas.

En primer lugar crearíamos el **security group**, donde seleccionamos también la VPC donde se crearía este:
```
# aws ec2 create-security-group --group-name "SG-Web" \
    --description "SG para los servidores web frontales" \
    --vpc-id "vpc-fd6f139b"
{
    "GroupId": "sg-0d035270"
}
```
Habilitar el puerto 80 TCP para el público en este **security group** para que se puedan realizar peticiones http al servidor web que se encuentre alojados en la instancia que tengamos:
```
# aws ec2 authorize-security-group-ingress --group-id "sg-0d035270" \
    --protocol tcp --port 80 --cidr 0.0.0.0/0
```
<div class="callout callout--info">
<p><strong>Nota:</strong> Si quisiéramos acceder por SSH a la instancia EC2, debemos habilitar el puerto tcp 22 en este grupo de seguridad.</p>
</div>

También podemos añadirle una etiqueta a dicho security group:
```
aws ec2 create-tags --resources "sg-0d035270" \
    --tags Key=Name,Value="SG-Web"
```
Por otro lado, crear el security group para las bases de datos **privadas**, donde se encontraría la base de datos:
```
# aws ec2 create-security-group --group-name "SG-BBDD" \
    --description "SG para los servidores de bases de datos" \
    --vpc-id "vpc-fd6f139b"
{
    "GroupId": "sg-2f3b6a52"
}
```
Ahora tocaría permitir que el **security group** de instancias públicas, pueda acceder al puerto **3306** de la base de datos que se encuentra en su security group correspondiente:
```
aws ec2 authorize-security-group-ingress --group-id "sg-2f3b6a52" \
    --protocol tcp --port 3306 --source-group "sg-0d035270"
```
Etiquetar el nuevo grupo de seguridad para reconocerlo más fácilmente desde la consola:
```
aws ec2 create-tags --resources "sg-2f3b6a52" \
    --tags Key=Name,Value="SG-BBDD"
```
Llegados a este punto ya tendríamos perfertamente nuestos security groups para las instancias que serán servidores web (públicos) y otro para los servidores de base de datos (privados).


### **Instancias EC2**

Es como denomina AWS a sus máquinas virtuales, las instancias se cobran por el % de uso de la cpu y el número de horas que se encuentren iniciadas.Si la instancia se encuentra en estado de **stop**, no se cobrará por ella.

Por otro lado comentar a favor de AWS, la gran variedad de tipos de instancias que nos pone a nuestra disposición. Desde tipos de instancias orientadas a grandes cargas de trabajo por parte de la CPU hasta el número de I/O en disco.

 Luego independientemente del tipo de instancia, hay unos modos de uso para las instancias EC2, que son básicamente:

 * **Bajo demanda**:  se pagan por horas levantadas, pudiéndose apagar en cualquier momento.
 * **Reservadas**: Bajo demanda: Se reservan de 1 a 3 años, Y suelen ser entre un 50-70% del costo si hubiera estado levantado todo el tiempo bajo el modo normal por así decirlo.
 * **Spot**: son las más baratas, hay un precio mínimo mucho más bajo que en el modo uso de baja demanda, y nosotros podemos elegir lo que deseamos pagar.

 Una instancia obviamente necesita de un sistema operativo, que se puede ser un sistema básico o un sistema operativo con una aplicación ya corriendo que se denominan AMI’s. Amazon tiene un marketplace que abarcan muchas AMI’s personalizadas en las que por ejemplo nos encontramos con servicios como: **nginx,php o apache** instaladas y configuradas para que se inicien automáticamente en el arranque del sistema.

 Así que podemos identificar varios elementos mínimo y necesarios para levantar una **instancia EC2**:

 * Una **AMI** bien sea un sistema operativo base o una AMI personalizada.
 * Una **VPC** que como ya se ha comentado, se crea una por defecto en la creación de la cuenta. Aún así tiene sentido que sea necesario al menos una ¿no?. Ya que es nuestro "CPD físico".
 * Un **security group** , nuestro volumen ```firewall lógico```.
 * Un **volumen o disco duro** como queráis llamarlo, también algo lógico ya que la instancia necesitaría albergar el sistema operativo y sus datos en algún lado :). La solicitud del nuevo volumen se hace al subservicio **EBS**, que veremos tranquilamente en el siguiente apartado.
 * Por último **Access Keys** o par de claves para conectarnos remotamente a la instancia posteriormente a su creación por el servicio SSH, podemos crear un par de claves nuevos en el momento de la creación de la instancia EC2 o elegir un par ya creado previamente en nuestra cuenta.

Vamos a lanzar una instancia EC2, con un imagen del sistema linux, más concretamente se le denomina "Amazon Linux" y digamos que es el resultado de una CentOS con una capa de personalización de AWS ;).

Creación de una instancia EC2 (web) desde línea de comandos:
```
aws ec2 run-instances --image-id ami-ca0135b3 --count 1 \
    --instance-type t2.micro --key-name aws-charlie \ 
    --security-group-ids sg-0d035270 --subnet-id subnet-88c87aee \
    --region eu-west-1
{
    "Instances": [
        {
            ..............
            "PrivateIpAddress": "10.0.1.185", 
            "ProductCodes": [], 
            "VpcId": "vpc-fd6f139b", 
            "CpuOptions": {
                "CoreCount": 1, 
                "ThreadsPerCore": 1
            }, 
            "StateTransitionReason": "", 
            "InstanceId": "i-0709d58d00766cc8c", 
            "ImageId": "ami-ca0135b3", 
            "PrivateDnsName": "ip-10-0-1-185......, 
            "KeyName": "aws-charlie", 
            "SecurityGroups": [
                {
                    "GroupName": "SG-Web", 
                    "GroupId": "sg-0d035270"
                }
            ], 
            "ClientToken": "", 
            "SubnetId": "subnet-88c87aee", 
            "InstanceType": "t2.micro", 
            "NetworkInterfaces": [
                {
                    "Status": "in-use", 
                    "MacAddress": "02:63:32:95:b5:b8", 
                    "SourceDestCheck": true, 
                    "VpcId": "vpc-fd6f139b", 
                    "Description": "", 
                    "NetworkInterfaceId": "eni-a1326082", 
                    "PrivateIpAddresses": [
                        {
                            "Primary": true, 
                            "PrivateIpAddress": "10.0.1.185"
                        }
                    ], 
                    "SubnetId": "subnet-88c87aee", 
                    "Attachment": {
                        "Status": "attaching", 
                        "DeviceIndex": 0, 
                        "DeleteOnTermination": true, 
                        "AttachmentId": "eni-attach-1e9be16d", 
                        "AttachTime": "2018-06-08T19:23:01.000Z"
                    }, 
                    "Groups": [
                        {
                            "GroupName": "SG-Web", 
                            "GroupId": "sg-0d035270"
                        }
                    ], 
                    "Ipv6Addresses": [], 
                    "OwnerId": "826764960316", 
                    "PrivateIpAddress": "10.0.1.185"
                }
            ], 
            "SourceDestCheck": true, 
            "Placement": {
                "Tenancy": "default", 
                "GroupName": "", 
                "AvailabilityZone": "eu-west-1a"
            }, 
            "Hypervisor": "xen", 
            "BlockDeviceMappings": [], 
            "Architecture": "x86_64", 
            "RootDeviceType": "ebs", 
            "RootDeviceName": "/dev/xvda", 
            "VirtualizationType": "hvm", 
            "AmiLaunchIndex": 0
            .........
}
```
Una vez se cree, como dijimos en el apartado anterior de security groups, debemos habilitar en dicho **SG-Web** que nos podamos conectar por SSH solamente con nuestra IP Pública
 
```
# aws ec2 authorize-security-group-ingress --group-id sg-0d035270 \
    --protocol tcp --port 22 --cidr 87.221.175.124/32
```
Ya solo nos faltaría para conectarnos remotamente, usar el endpoint que nos proporciona AWS para dicha instancia EC2, en este caso habria que utilizar el usuario **ec2-user** que es el usuario predeterminado de la AMI que he usado y tener la clave privada en nuestra máquina anfitriona.

<div class="callout callout--info">
<p><strong>Recordatorio:</strong> Si la instancia se reinicia, variará el endpoint, por lo tanto no nos podremos conectar a través del endpoint o la dirección IP pública dinamica que tuviera asociada dicha instancia. Si por cualquier motivo no quisiéramos que esto variara, deberiamos asociarle posteriormente a la creación de la máquina una <strong>Elastic IP</strong></p>
</div>

### **EBS**

EBS es el servicio que básicamente administra y crea los volúmenes necesarios para las instancias. Obviamente es posible asociar un nuevo disco a una instancia ya lanzada, como a una instancia que estamos creando. Eligiendo así en su creación tanto el **disco raíz** como otros volúmenes si quisiéramos.

Así que partiendo de la base que tenemos una instancia corriendo, vamos a re-utilizar la EC2 anterior, asociándole un nuevo volumen, para ello primero habrá que crearlo:

Así que vamos a proceder a crear dicho volumen, pero esta vez vamos a etiquetar el recurso a la par que lo creamos, hasta ahora estabamos etiquetando nuestro recurso después de su creación:

Este ejemplo crea un volumen magnetico (HDD) en la zona de disponibilidad **eu-west-1a** de 1GiB.
```
# aws ec2 create-volume --size 1 --region eu-west-1 \
    --availability-zone eu-west-1a --volume-type standard \
    --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=MyPendrive}]'
{
    "AvailabilityZone": "eu-west-1a", 
    "Tags": [
        {
            "Value": "MyPendrive", 
            "Key": "Name"
        }
    ], 
    "Encrypted": false, 
    "VolumeType": "standard", 
    "VolumeId": "vol-0795c601c460a306e", 
    "State": "creating", 
    "SnapshotId": "", 
    "CreateTime": "2018-06-09T09:19:35.000Z", 
    "Size": 1
}
```

<div class="callout callout--info">
<p><strong>IMPORTANTE:</strong> la zona de disponibilidad donde debe crearse dicho volumen, debe ser la misma en la que se encuentra la instancia.</p>
</div>

En el apartado de [Kubernetes Operations](/kops), hablaremos mejor sobre el asunto de las regiones y zonas de disponibilidad en AWS. Pero por ahora es más que suficiente.

Ahora sí, vamos a asociar el nuevo volumen creado a nuestra instancia y posteriormente comprobaremos que efectivamente la instancia o más bien el sistema operativo de esta, ha reconocido dicho volumen:

```
# aws ec2 attach-volume --volume-id vol-0795c601c460a306e \
    --instance-id i-0709d58d00766cc8c --device /dev/xvdb
{
    "AttachTime": "2018-06-09T09:27:52.980Z", 
    "InstanceId": "i-0709d58d00766cc8c", 
    "VolumeId": "vol-0795c601c460a306e", 
    "State": "attaching", 
    "Device": "/dev/xvdb"
}
```

Nos conectamos remotamente por el servicio SSH a nuestra instancia con nuestro par de claves asociado a dicha instancia:

```
# ssh -i "aws-charlie.pem" ec2-user@54.154.233.240
The authenticity of host '54.154.233.240 (54.154.233.240)' can't be established.
ECDSA key fingerprint is SHA256:qe+WZffrTjfldzhOUzEtmCdh0XRx44QkpK2nqAUAf0Y.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '54.154.233.240' (ECDSA) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2018.03-release-notes/
10 package(s) needed for security, out of 12 available
Run "sudo yum update" to apply all updates.
```

Si listamos los discos de nuestra instancia, vemos solamente el volumen raíz del sistema, pero no el que acabamos de asociarle:

```
[ec2-user@ip-10-0-1-185 ~]$ df -h
S.ficheros     Tamaño Usados  Disp Uso% Montado en
devtmpfs         484M    56K  484M   1% /dev
tmpfs            494M      0  494M   0% /dev/shm
/dev/xvda1       7,8G   1,1G  6,7G  14% /
```

Esto se debe a que el volumen asociado, esta conectado a la instancia pero no esta montado en ningún lugar del sistema, si ejecutamos la siguiente instrucción vemos los volúmenes disponibles en la instancia:

```
[ec2-user@ip-10-0-1-185 ~]$ lsblk -f
NAME    FSTYPE LABEL UUID                                 MOUNTPOINT
xvda                                                      
└─xvda1 ext4   /     36181dd4-d665-43b1-a236-8cb5c24125f6 /
xvdb       
```

Por lo tanto, antes de seguir y montar dicho volumen en un punto de montaje, tenemos que formatear el volumen con un sistema de ficheros:

```
[ec2-user@ip-10-0-1-185 ~]$ sudo mkfs -t ext4 /dev/xvdb
```
<div class="callout callout--info">
<p><strong>Nota:</strong> el volumen es solo necesario formatearlo después de su creación, una vez ya hayamos hecho esta acción en el volumen no será necesario, ya que a partir de ese momento podemos desasociar el volumen de la instancia y asociarlo a otra.</p>
</div>

Ya si lo montaríamos en algún lugar y comprobaríamos que efectivamente se ha detectado dicho volumen:

```
[ec2-user@ip-10-0-1-185 ~]$ sudo mount /dev/xvdb /mnt/
[ec2-user@ip-10-0-1-185 ~]$ df -h
S.ficheros     Tamaño Usados  Disp Uso% Montado en
devtmpfs         484M    60K  484M   1% /dev
tmpfs            494M      0  494M   0% /dev/shm
/dev/xvda1       7,8G   1,1G  6,7G  14% /
/dev/xvdb        976M   1,3M  908M   1% /mnt
```

En el próximo apartado, vamos a tocar el servicio **ELB** que se trata de un balanceador de carga, esto es más que recomendable en cualquier escenario que empecemos a tener más de 1 instancia ofreciendo un servicio públicamente, acompañado de la creación de una pequeña AMI personalizada y entrando en detalle como configurar un grupo de auto escalado de la instancia que hayamos creado, creando así replicas exactas de nuestra instancia EC2.

