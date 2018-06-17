---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Introducción al servicio VPC
description: Conoce mejor los subservicios y como funciona las redes en AWS

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
        content: Introducción a AWS
        url: '/intro'
    next:
        content: Introducción a EC2
        url: '/ec2'
---


## **NETWORKING - VPC**

La traducción de las siglas corresponde a “Virtual Private Cloud”, lo que podríamos decir que este componente simula una red privada o como si fuera nuestra propia red dentro de un centro de datos. Sería un segmento de red aislado de las otras VPC’s que tuviéramos creadas.

Por lo que cabe deducir es inevitable tener al menos 1 VPC en nuestra cuenta de AWS y tal es así, que
por defecto nos trae una VPC creada. Nosotros mismos podemos escoger el direccionamiento de red privado hasta un máximo de un CIDR /16, dentro de la VPC podemos segmentar y crear subnets según nuestra necesidad.

Las subnets pueden ser privada o públicas, por ejemplo, supongamos que nuestra aplicación tiene una parte web y otra con la base de datos. Para mayor seguridad y mejor práctica se colocaría la base de datos en una subnet privada, donde solo tendría permiso de acceso las instancias que se crearán en una subnet pública en concreto y prohibiendo el acceso remoto desde fuera hacia la base de datos. A estas últimas instancias que se levantarán dentro de la subnet pública se les repartiría de forma automática direccionamiento IP público.

Otro detalle es que las VPC se ubican dentro de una región y las subnets se localizan dentro de una zona
de disponibilidad. Se pueden crear subnets siempre y cuando no superemos el límite del direccionamiento de red elegido a
la hora de crear la VPC, por lo tanto, es algo bastante importante ya que una mala elección nos limitaría en el futuro. Como hemos comentado antes, tienen un alcance zonal, lo que quiere decir es que, si hemos creado nuestra VPC en la región de Irlanda, la subnet se deberá crear en una de las zonas de disponibilidad disponibles en esta región. Una subnet solo puede ubicarse en una zona de disponibilidad y esto no puede ser modificado después de su creación.

> **A crear:** 1 VPC con 2 Subnets públicas y 2 Subnets privadas en dos AZ diferentes:

<p align="center">
    <img src="https://raw.githubusercontent.com/carlosjsanch3z/proyectoaws/master/assets/img/VPC%26Subnets.png" alt="Diagrama VPC - Subnets">
</p>

Partiendo del diagrama anterior, vamos a proceder a su creación desde la línea de comandos:

Primero crear la VPC:

```
# aws ec2 create-vpc --cidr-block 10.0.0.0/16 --output json
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
# aws ec2 create-tags --resources "vpc-fd6f139b" \
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
# aws ec2 create-subnet --cidr-block "10.0.3.0/24" \ 
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

# aws ec2 create-tags --resources "subnet-6db1030b" \
    --tags Key=Name,Value="private1"
```
**Subnet - privada2:**

```
# aws ec2 create-subnet --cidr-block "10.0.4.0/24" \ 
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

# aws ec2 create-tags --resources "subnet-4a61e202" \
    --tags Key=Name,Value="private2"
```

Ahora para que realmente las subnets públicas, sean públicas de verdad, tendremos que habilitarles a ambas la opción de que automaticamente a los recursos que se creen en la subnet, se le asignen IP's públicas:

```
# aws ec2 modify-subnet-attribute \
--subnet-id "subnet-88c87aee" --map-public-ip-on-launch

# aws ec2 modify-subnet-attribute \
--subnet-id "subnet-2364e76b" --map-public-ip-on-launch
```

Por defecto las **subnets** comparten una misma tabla de enrutamiento de manera predeterminada, que se crea automáticamente con la VPC por defecto. Pero es posible crear una **tabla de enrutamiento** para una subnet específica.

### **Routing Table**

Las tablas se utilizan para enrutar el tráfico dentro de las VPC’s, por defecto se crea una de manera predeterminada, es posible crear varias **tablas de enrutamiento** y asociarlas de manera independiente a algunas subnets y a otras no.

A mi parecer es super importante etiquetar bien los componentes en AWS, ya que de lo contrario, nos podemos volver locos en el momento que la infraestructura escala y tenemos varios elementos de cada servicio.

Por ejemplo si queremos sacar alguna información de una subnet, podemos filtrar por el nombre que le hayamos puesto en su creación de la siguiente manera:

```
# aws ec2 describe-subnets --filters Name=tag:nombre-subnet
```
Salida, consultando la subnet **public1**:
```
{
    "Subnets": [
        {
            "AvailabilityZone": "eu-west-1a", 
            "Tags": [
                {
                    "Value": "public1", 
                    "Key": "Name"
                }
            ], 
            "AvailableIpAddressCount": 251, 
            "DefaultForAz": false, 
            "Ipv6CidrBlockAssociationSet": [], 
            "VpcId": "vpc-fd6f139b", 
            "State": "available", 
            "MapPublicIpOnLaunch": true, 
            "SubnetId": "subnet-88c87aee", 
            "CidrBlock": "10.0.1.0/24", 
            "AssignIpv6AddressOnCreation": false
        }
    ]
}
```

#### **Public Routing Table**

Posteriormente creamos una **tabla de enrutamiento** asociada a nuestra vpc:
```
# aws ec2 create-route table --vpc-id "vpc-fd6f139b"
```
Salida:
```
{
    "RouteTable": {
        "Associations": [], 
        "RouteTableId": "rtb-a2eb72db", 
        "VpcId": "vpc-fd6f139b", 
        "PropagatingVgws": [], 
        "Tags": [], 
        "Routes": [
            {
                "GatewayId": "local", 
                "DestinationCidrBlock": "10.0.0.0/16", 
                "State": "active", 
                "Origin": "CreateRouteTable"
            }
        ]
    }
}
```
Etiquetamos la tabla de enrutamiento con un nombre, en este caso **route-public**:
```
# aws ec2 create-tags \
    --resources "rtb-a2eb72db" --tags Key=Name,Value="route-public"
```
Y asociamos la tabla de enrutamiento a las dos subnets públicas:
```
# aws ec2 associate-route-table \
    --subnet-id "subnet-88c87aee" \
    --route-table-id "rtb-a2eb72db"

# aws ec2 associate-route-table \
    --subnet-id "subnet-2364e76b" \
    --route-table-id "rtb-a2eb72db
```

#### **Private Routing Table**

Haríamos lo mismo, creando una tabla de enrutamiento, que asociariamos a las subnets privadas:

```
# aws ec2 create-route-table --vpc-id "vpc-fd6f139b"
{
    "RouteTable": {
        "Associations": [], 
        "RouteTableId": "rtb-71f66f08", 
        "VpcId": "vpc-fd6f139b", 
        "PropagatingVgws": [], 
        "Tags": [], 
        "Routes": [
            {
                "GatewayId": "local", 
                "DestinationCidrBlock": "10.0.0.0/16", 
                "State": "active", 
                "Origin": "CreateRouteTable"
            }
        ]
    }
}
```

Etiquetar la **route-table**
```
# aws ec2 create-tags \
    --resources "rtb-71f66f08" --tags Key=Name,Value="route-private"
```

Asociar a las redes privadas:
```
# aws ec2 associate-route-table \
    --subnet-id "subnet-6db1030b" \
    --route-table-id "rtb-71f66f08"

# aws ec2 associate-route-table \
    --subnet-id "subnet-4a61e202" \
    --route-table-id "rtb-71f66f08"
```

### **Internet Gateway**

Es el punto de conexión entre los recursos de VPC para conectar las instancias a internet, como mínimo tenemos que tener una subnet pública, con una tabla de routas que tengan asociada un internetgat como ruta default.

Además de tener IG, las instancias que creemos dentro la subnet deben tener IP Públicas, el IG es automaticamente gestionado y escalado por Amazon y lo único que hay que hacer es asociarlo a una Route Table.

<div class="callout callout--info">
<p><strong>Nota:</strong> solamente puede haber un **IG** asocicado a una **VPC determinada**</p>
</div>

Primero crear el IG:

```
# aws ec2 create-internet-gateway
{
    "InternetGateway": {
        "Tags": [], 
        "Attachments": [], 
        "InternetGatewayId": "igw-fc2f6b9b"
    }
}
```

Etiquetarlo:
```
# aws ec2 create-tags \
    --resources "igw-fc2f6b9b" --tags Key=Name,Value="external-gateway"
```

Asociarlo a la VPC:
```
# aws ec2 attach-internet-gateway \
    --internet-gateway-id "igw-fc2f6b9b" \
    --vpc-id "vpc-fd6f139b""
```

Por último, asociar el IG como ruta por defecto en la "route table" de las subnets públicas.

```
# aws ec2 create-route --route-table-id "rtb-a2eb72db" --destination-cidr-block 0.0.0.0/0 --gateway-id "igw-fc2f6b9b"
{
    "Return": true
}
```

### **Elastic IP**

Es como AWS llama a las IP's públicas, el motivo particular por el cual es necesario utilizar estas IP's es cuando por ejemplo queremos que la dirección IP Públicas de una instancia no varie nunca.

Suponiendo que estamos levantando una instancia en una subnet pública donde habilitamos de manera predeterminada la opción de que se le asigne una ip pública ha dicha instancia cuando se levante dentro de esa subnet. Es cierto que la dirección que le asigna es de acceso público, pero hay un problema y es que cuando se reinicie la máquina o cambie de estado, se le re-asigna otra dirección IP pública diferente.

Esto es un problema si necesitamos que la direccion IP no cambie, y aquí es cuando entra en juego este tipo de direcciones IP, AWS te ofrece la primera **Elastic IP** gratuitamente siempre y cuando se encuentre asociada a una instancia. Si no fuera así nos cobraría por tener esa dirección "reservada".

Otra particularidad es que podemos desasociar la Elastic IP de una instancia a otra.

Para solicitar una Elastic IP en nuestra cuenta, más concretamente en la región **eu-west-1**, habría que ejecutar la siguiente instrucción:

```
# aws ec2 allocate-address --domain "vpc" --region eu-west-1
{
    "PublicIp": "34.247.238.137", 
    "Domain": "vpc", 
    "AllocationId": "eipalloc-dcc786e1"
}
```
Si quisiéramos asociar dicha **IP** a una instancia sería de la siguiente forma:

```
aws opsworks --region eu-west-1 associate-elastic-ip \
    --instance-id dff32xd-32323-ddsdsds-232-2323-3232323232 \
    --elastic-ip 34.247.238.137
```

O si de lo contrario nos gustaria liberar esa **Elastic IP**:
```
# aws ec2 release-address --allocation-id "eipalloc-dcc786e1" \
    --public-ip "34.247.238.137"
```

### **NAT Gateway**

Como hacer que los recursos que se creen en subnets privadas, incluso sin tener ip públicas, puedan ser capaces de acceder a internet. Por ejemplo si tenemos instancias dentro de subnets privadas y necesitamos actualizar el sistema, paquetes, etc.. O si necesitamos que la instancia que se encuentra en la subnet privada necesite conectarse a un endpoint o a una api.

Evitando así tener que contratar IPs Públicas para estos casos.

Tiene que crearse en una subnet pública que tenga en su Route table, la ruta por defecto el IG.

Si por ejemplo vamos a consumir un recurso de terceros y este válida la dirección Ip pública, si tuvieramos un grupo de auto-escalado y pasáramos de tener de 2 instancias a tener hasta 30. Por ejemplo nos dejaría de funcionar un servicio externo porque algunos incluso tienen un limite de validación de 2-3 Ips Públicas.

Esto se evita utilizando el NAT GAteway, ya que tiene una ip pública y estatica.


Crear el NAT Gateway dentro de una subnet pública, debemos coger el valor de **allocation-id** que nos devolvió cuando creamos antes la Elastic IP, para poder asociarla con el nuevo **NAT Gateway**:
```
# aws ec2 create-nat-gateway --subnet-id subnet-88c87aee \
    --allocation-id eipalloc-dcc786e1
{
    "NatGateway": {
        "NatGatewayAddresses": [
            {
                "AllocationId": "eipalloc-dcc786e1"
            }
        ], 
        "VpcId": "vpc-fd6f139b", 
        "State": "pending", 
        "NatGatewayId": "nat-04779f5584e71f247", 
        "SubnetId": "subnet-88c87aee", 
        "CreateTime": "2018-06-07T22:37:32.000Z"
    }
}
```
<div class="callout callout--info">
<p><strong>Nota:</strong> la subnet pública, debe tener en su tabla de encaminamiento como ruta por defecto el Internet Gateway.</p>
</div>

Ahora nos tocará añadir como ruta por defecto a la tabla de enrutamiento de una subnet privada, el NAT Gateway:

```
# aws ec2 create-route --route-table-id "rtb-71f66f08" \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id "nat-04779f5584e71f247"
{
    "Return": true
}
```

### **NACL**

Esto son listas de control de acceso dentro de la VPC, podemos restringir el acceso a un rango de direcciones IP o a una dirección IP en concreta. Por defecto, cuando se auto crea la **VPC** de manera adjunta se crea una regla o entrada del tipo **Allow all**.

Se puede restringir el tanto de entrada como de salida, podemos crear varias NACL y asociarlas a diferentes VPC's.

Crear una nueva NACL:

```
# aws ec2 create-network-acl --vpc-id "vpc-fd6f139b"
{
    "NetworkAcl": {
        "Associations": [], 
        "NetworkAclId": "acl-c1a455b8", 
        "VpcId": "vpc-fd6f139b", 
        "Tags": [], 
        "Entries": [
            {
                "RuleNumber": 32767, 
                "Protocol": "-1", 
                "Egress": true, 
                "CidrBlock": "0.0.0.0/0", 
                "RuleAction": "deny"
            }, 
            {
                "RuleNumber": 32767, 
                "Protocol": "-1", 
                "Egress": false, 
                "CidrBlock": "0.0.0.0/0", 
                "RuleAction": "deny"
            }
        ], 
        "IsDefault": false
    }
}
```
Lo importante a la hora de agregar nuevas rules, es definir bien el orden de la prioridad(ya que estas se procesan en orden ascendente).

Añadir una nueva regla de entrada a la NACL:

```
# aws ec2 create-network-acl-entry --network-acl-id "acl-0fa45176" \
    --ingress --rule-number 10 --protocol tcp \
    --port-range From=80,To=80 --cidr-block 0.0.0.0/0 \
    --rule-action deny
```
<div class="callout callout--info">
<p><strong>La regla anterior:</strong> Bloquearía todo el trafico entrante http al puerto 80 desde cualquier direccionamiento.</p>
</div>


Ver la información sobre una NACL determinada:

```
# aws ec2 describe-network-acls --network-acl-ids "acl-0fa45176"
```
