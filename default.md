---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Descripción del Proyecto
description: ¿Pero de que trata el proyecto realmente? o que se abarca en él

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
        content: Página principal
        url: '/'
    next:
        content: Introducción a AWS
        url: '/intro'
---

### **DESCRIPCIÓN**

En este documento vamos abarcar el problema habitual con el que se encuentran la mayoría de los responsables que tienen a su cargo una aplicación web que tengan picos de demandas puntuales. En los que es muy difícil predecir cuándo va a ocurrir dicho evento, por lo que necesitamos de alguna manera que nuestra infraestructura sea capaz de escalar automáticamente según la necesidad y sin escalar de manera desproporcionada elevando los costos de forma innecesaria.

En este caso siempre vamos a partir como base, como proveedor de nube pública AWS (Amazon Web Services), ya que a día de hoy es una solución muy utilizada tanto como por personas individuales, como por startups.

Partiendo como base con AWS, posteriormente necesitamos otra solución para tener nuestros “servidores”, en la que básicamente nos podemos decantar por dos opciones:

* **Máquinas Virtuales:** es la forma tradicional y donde AWS nos ofrece su propio servicio denominado EC2, donde las usaremos junto con algo que se hace llamar `Auto Scaling Groups`.
* **Contenedores:** esto que se está poniendo tan de moda ahora y escuchamos en todos lados, para los usuarios que se hayan decantado o quieran pasar su aplicación a contenedores, también veremos unos ejemplos prácticos con:
    * **Kops:** para crear y administrar tu propio clúster `Kubernetes` sobre AWS, y pudiendo así auto-escalar tu aplicación con contenedores.
    * **EKS:** Proximamente.

Así que, para poder entender mejor las diferentes soluciones, me parece más que interesante empezar por una introducción con los principales componentes de AWS, desde la línea de comandos a través de la AWS CLI.
Posteriormente desplegar un Clúster de Kubernetes sobre AWS, gracias a la herramienta Kops. Donde también vamos a auto escalar a nivel de infraestructura (nodos).
Y por último el otro extremo sería utilizando AWS Fargate, desplegar un chat en tiempo real con socket.io y un redis sobre contenedores. Auto escalando estos en horizontal.