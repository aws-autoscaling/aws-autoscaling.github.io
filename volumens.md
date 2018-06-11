---
layout: default
keywords:
comments: false
title: Volumenes Persistentes
description: In progress
micro_nav: true
page_nav:
    prev:
        content: AutoScaler
        url: '/auto'
    next:
        content: EKS
        url: '/eks'
---

## Mantener el estado de nuestra aplicación

Si queremos mantener el estado de nuestra aplicación, frente a un fallo en las instancias, que por el motivo que fuera se cayerán dichas instancias. Y nos vieramos en la situación que automáticamente nuestros contenedores se vuelven a replicar a nuestro estado deseado, pero que ocurre si los datos de la aplicación se estaban almacenando dentro del propio contenedor por poner un ejemplo.¿No es mejor almacenar esos datos de manera replicada o en un servicio externo?

La opción de replicar el almacenamiento de forma local y replicandose en múltiples servidores es compleja de implementar para incluirla en este apartado.

Así que vamos a tirar por la opción de utiizar un servicio externo ya que estamos utilizando AWS, podemos elegir entre:
* S3
* Elastic File System (EFS)
* Elastic Block Store

La opción de **S3** la descartamos ya que **no vamos a sustituir** el almacenamiento de un disco local, por llamadas a una API para poder almacenar los datos.

La segunda opción, EFS tiene una muy clara ventaja y es que puede montarse en múltiples instancias de EC2 repartidos en múltiples AZ. Y es lo más cercano a lo que tolerancia a fallos se refiere. Pero es muy costo y nos penalizaría en este caso la ```latencia```.

Por lo tanto la última opción que nos queda es **EBS** donde el almacenamiento es mucho más rápido. La latencia de acceso a los datos sería muy baja. Pero tiene un pequeño inconveniente y es la disponibilidad. Este servicio no funciona en **múltiples zonas de disponibilidad**, si este fallará habría un tiempo de inactividad hasta que se restaurará dicha zona en caso de fallo.

Aún asi es la mejor opción que tenemos y más a la hora de utilizar en este caso **Jenkis** que dependerá mucho de I/O, y necesitamos que el acceso a los datos sea super ligero. Otra dos razones por la que decantarse por esta opción es la compatible con Kubernetes y que es mucho mas económico.

Como hemos comentado en el que caso de fallo en la AZ donde se encuentre el volumen, sería necesario migrar a otra zona **saludable**.

Así que vamos a introducirnos un poco en la creación de los volumenes en AWS con el servicio **EBS**:

### **Crear los volumenes en AWS**

Lo primero que necesitamos saber es en que zona de disponibilidad se encuentran nuestros **nodos workers**, lo podemos conseguir de la siguiente forma:

```
➜  aws ec2 describe-instances \
| jq -r \
".Reservations[].Instances[] \
| select(.SecurityGroups[]\
.GroupName==\"nodes.$NAME\")\
.Placement.AvailabilityZone"

eu-west-1c
eu-west-1b
```

Nos deberan aprecer las dos AZ en las que se encuetran los nodos.

<div class="callout callout--info">
<p><strong>NOTA:</strong> puede causar confusión que ahora los nodos están en AZ's diferentes en comparación con apartados anteriores, esto se debe a que modifique el valor del InstanceGroup de los workers.</p>
</div>

Vamos a almacenar las dos zonas de disponibilidad en un fichero y posteriormente seleccionar cada una de ellas y almacenarlas en una variable de entorno:

```
➜  aws ec2 describe-instances \
    | jq -r \
    ".Reservations[].Instances[] \
    | select(.SecurityGroups[]\
    .GroupName==\"nodes.$NAME\")\
    .Placement.AvailabilityZone" \
    | tee zones

AZ_1=$(cat zones | head -n 1)

AZ_2=$(cat zones | tail -n 1)
```
Ahora vamos a proceder a crear los volumenes en las zonas de disponibilidad, por ejemplo, dos volumenes en una AZ y uno en otra. Tendrán un espacio de almacenamiento de 10GB, y serán del tipo **gp2**, por otro lado los etiquetaremos para que sea más fácil reconocerlos posteriormente:

```
➜  VOLUME_ID_1=$(aws ec2 create-volume \
    --availability-zone $AZ_1 \
    --size 10 \
    --volume-type gp2 \
    --tag-specifications "ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=$NAME}]" \ 
    | jq -r '.VolumeId')

➜  VOLUME_ID_2=$(aws ec2 create-volume \
    --availability-zone $AZ_1 \
    --size 10 \
    --volume-type gp2 \
    --tag-specifications "ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=$NAME}]" \
    | jq -r '.VolumeId')

➜  VOLUME_ID_3=$(aws ec2 create-volume \
    --availability-zone $AZ_2 \
    --size 10 \
    --volume-type gp2 \
    --tag-specifications "ResourceType=volume,Tags=[{Key=KubernetesCluster,Value=$NAME}]" \
    | jq -r '.VolumeId')
```

Si por ejemplo miramos el valor de la variable de uno de ellos, debemos tener almacenado el id del volumen en AWS:

```
➜  echo $VOLUME_ID_1
vol-07b9ce41b602ada73
```
Por último si nos queremos asegurar de que se ha creado el volumen, podemos ejecutar la instrucción:

```
➜  aws ec2 describe-volumes \
    --volume-ids $VOLUME_ID_1
{
    "Volumes": [
        {
            "AvailabilityZone": "eu-west-1c", 
            "Attachments": [], 
            "Tags": [
                {
                    "Value": "proyecto.k8s.local", 
                    "Key": "KubernetesCluster"
                }
            ], 
            "Encrypted": false, 
            "VolumeType": "gp2", 
            "VolumeId": "vol-07b9ce41b602ada73", 
            "State": "available", 
            "Iops": 100, 
            ...
            "Size": 10
        }
    ]
}
```

Ahora que tenemos los volumenes disponibles en las mismas zonas de disponibilidad que las instancias EC2 workers, podemos proceder a crear los volumenes persistentes en Kubernetes.

En la siguiente figura podemos ver lo que acabamos de hacer:

<p align="center">
    <img src="/images/Figura21.png" alt="Volumenes EBS creado en la misma zona que los workers">
</p>
<p align="center">
    <b>Figura 21 - Volumenes EBS creado en la misma zona que los workers</b>
</p>



## **Volumenes Persistentes**

El hecho de que tengamos los volumenes creado en EBS y disponibles, no significa que nuestro Clúster de Kubernetes sepa que estos existen. Para ello es necesario agregar los **volumenes persistentes** que actuan de puente entre el clúster de kubernetes y los volumenes creado con EBS.

Los volumenes persistentes son también recursos del clúster, pero su **principal diferencia** es que el ciclo de vida de estos es independiente de los **Pods** por los que estén siendo utilizados.

Dicho esto, vamos a crear la definición para dichos volumenes persistentes en formato YAML:

```

```


### **Los volumenes persistentes en el Clúster**

### **Solicitando dichos volumenes a AWS**