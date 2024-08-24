# Introducción

Este repositorio contiene el código y el detalle del desarrollo del laboratorio ***Creación de microservicios y una canalización de CI/CD con AWS*** de AWS Academy. El proyecto propuesto por el laboratorio consiste en llevar a cabo una serie de pasos con dos objetivos principales: implementar los cambios que permitan pasar de una applicación ejecutándose de forma *monolítica* a una que lo haga utiizando *microservicios*, y llevarlo a cabo creando para ello un entorno de desarrollo que incorpore prácticas de *integración y despliegue continuo (CI/CD)*.  

Respecto al primero, excluyendo la DB (que se funciona en una instancia de RDS), la aplicación en su estado original se ejecuta íntegramente en una instancia EC2. Por un lado, ello conlleva dificultades en la escalabilidad ante variaciones en la demanda, dado que para ampliar la capacidad de una parte de la aplicación debe escalarse la totalidad de la misma, teniéndose una relación de compromiso: si se escala para el servicio más demandado, se incurre en costo adicional innecesario dado que el otro no lo requiere; si se mantiene la escala para el servicio menos requerido se generará un cuello de botella en el más solicitado. A su vez, una estructura monolítica tiene menor *fiablilidad* o *reliabiity* dado que siemple implica un *single point of failure*. Ambos elementos se evitan con una solución basada en microservicios; tal arquitectura admite la posibilidad de escalar los servicios de forma independiente para adaptarse a la demanda de cada uno y al mismo tiempo da más robustez al permitir que si uno de los servicios está fuera de operación el resto permanezca funcional. 

En relación el segundo objetivo, cada vez que se efectuan cambios en el código de la aplicación, para que los mismos lleguen a impactar en el entorno productivo deben ejecutarse una sere de tareas de compilación, testeo y deploymen. Implementar prácticas de CI/CD en el entorno implica precisamente automatizar estas tareas de forma tal de agilizar el ciclo de desarrollo haciendo posible iterar con mejoras continuas que resultan en actualizaciones frecuentes y confiables de la aplicación.

A continuación se describen a alto nivel los pasos efectuados durante el laboratorio. No se lo hace en la secuencia propuesta en las instrucciones del mismo, sino agrupándolos como se considera más conveniente y resaltando algunos detalles que se estiman especialmente relevantes para entender el funicionamiento de la arquitectura propuesta. Más adelante se incluyen diagramas de la arquitectura y el flujo de CI/CD implementado y, finalmente, se agrega un pequeño ejemplo ilusrativo de estimación de costos.


# Pasos

### 1. Creación del entorno

**a.** En primer lugar, se crea un entorno en el servicio ***AWS Cloud9*** que levanta una instancia de EC2, donde se utilizará el editor de texto durante el desarrollo y se ejecutarán contenedores para evaluar las imágenes antes de enviarlas a producción. Como detalles a tener en cuenta, nótese que:
- La AMI elegida (*Amazon Linux 2*) incluye la *AWS CLI* que se utilizará para crear programáticamente algunos recursos. y también el engine de *Docker,* necesario precisamente para efectuar esas pruebas.
- La instancia se crea en la misma ***Amazon VPC*** (*LabVPC*) en que se está ejecutando originalmente la aplicación, lo cual hace posible conectarse con todo el resto de los recursos a utilizar.
- A su vez, dentro de esa VPC, la instancia está dentro de la subred *Public Subnet 1*; junto con otras, esa es una condición necesaria para que la instancia sea accesible desde internet, algo requerido para esar Docker durante el desarrollo (ver más adelante). 

**b.** Además, se crean dos repositorios en el servicio ***AWS CodeCommit***, uno para hostear el codebase de la aplicación y el otro para el código relacionado al deployment. Obsérvese que el ambos se encuentran incluidos como directorios en este repositorio.

**c.** Asimismo, se crean dos *registries* de imágenes de contenedores en el servicio ***AWS ECR*** con el objetivo de alojar, en cada uno, las distintas versiones de las imágenes de los dos microservicios a crear.


### 2. Migración a microservicios

**a.** Con el objetivo de pasar a ejecutar dos microservicios, uno para cada tipo de usuario (*customer* y *employee*), que puedan correr de forma independiente en contenedores, se modifica el código de la aplicación *monolítica* original. Para ello se generan dos copias análogas pero con las diferencias correspondientes a lo requerido para cada tipo de usuario.

**b.** Para evaluar durante el desarrollo y solucionar problemas de forma temprana, antes de enviar las nuevas versiones de las imágeneas a producción se las ejecuta en Docker. Para ello, para cada microservicio se crea el Dockerfile, se constrye la imagen y se levanta un contenedor con ella. Como se dijo, los contenedores deben ser públicamente accesibles de modo que las imágenes que corren puedan ser evaluadas via intenet usnado un navegador. Para ello:
- A la hora de ejecutar el contenedor se lo hace con un mapeo del puerto de la instancia de desarrollo (aquí el Docker host) al puerto correspondiente del contenedor (aquí 8080 u 8081), donde la aplicación está escuchando.
- A su vez, debe habilitarse el tráfico TCP en tales puertos en el security group asociado a la instancia de desarrollo de modo que la misma pueda recibir ese tráfico. 

Cumplidas ambas condiciones, con un navegador desde internet se puede acceder al servicio ejecutándose en el contenedor a través de los puertos 8080 / 8081 de la instancia de desarrollo.

**c.** Una vez completado el desarrollo deseado, para cada microservicio, se hace un *push* de la imagen al registry correspndiente creado en ECR en el paso previo.


### 3. Creación del Application Load Balancer

**a.** Hecho eso, se crea un Application Load Balancer (ALB) de tipo *internet-facing* en *LabVPC*. Como detalles a tener en cuenta, nótese que:
- Se lo asocia a las dos subredes públicas (*Public Subnet 1* y *Public Subnet 2*), lo cual es necesario para que reciba tráfico de internet y pueda enviarlo a los contenedores de ECS que correrán en esas subredes.
- Se le asigna un security group en el que se crean reglas de entrada que permiten el tráfico TCP desde cualquier IPv4 en los puertos 80 y 8080, que son los que se usaran para la aplicación.

**b.** Dado que se pretende utilizar la estrategia de deployment *blue/green* que implica que, ante un deployment, pasaremos de usar la aplicación del entorno *blue* (con la versión en uso) al entorno *green* (con la nueva versión) de modo de garantizar su disponibilidad en todo momento, se tiene que:
  - Para cada microservicio se requieren dos *target groups (TG)*, por lo cual se crean los cuatro TG necesarios.
  - Se requieren dos *listeners* (en 80 y 8080, que son los puertos a utilizar). A su vez, para enrutar el tráfico al microservicio correspondiente, cada uno de ellos debe tener dos *rules* (que a su vez envian a cada uno de esos TG):
    - Una regla default que enviará tráfico al microservico ***customer***.
    - Una regla específica que enviará el tráfico que incluya `/admin` en el path al microservico ***employee***.  


### 4. Creación del ECS cluster

**a.** Utilizando el servicio **AWS ECS** y empleando **AWS Fargate** como compute engine, se crea un cluster *serverless* 
(*microservices-serverlesscluster*) dentro de la *LabVPC* y configurado para usar *Public Subnet 1* y *Public Subnet 2*, que se utilizará para ejecutar los microservicios.

**b.** Para cada microservicio se crea la *task definition*, que funcionará como "molde" a la hora de instanciar nuevas *tasks* de acuerdo a la escala requerida; a su vez se registra en ECS utilizando el comando `aws ecs register-task-definition` de la AWS CLI. Como detalle, obsérvese que se deja parametrizado el nombre de la imagen para que durante el deployment se interpole con la versión adecuada.

**c.** Se crean los dos *ECS services*, especificando detalles como el cluster donde correrá, la task definition para crear las tasks, los puertos a exponer por el contenedor, el target group y una `desired-count` inicial de 1, lo cual implica que al iniciar el servicio se ejecute una única réplica. Se lo hace utilizando el comando `aws ecs create-service` de la AWS CLI.


### 5. Incorporación de prácticas de CI/CD

**a.** Se crea una "applicación" del servicio **AWS Code Deploy** con un *deployment group* para cada servicio, indicando en cada caso los TG correspondientes y el ECS service a donde se debe hacer el deployment.

**b.** Para cada microservicio se crea el *application specification (AppSpec) file* donde se establece cómo efectuar el deploy. Como con la task definition, se deja paramerizado el value de la task definition para que pueda ser evaluado dinámicamente al momento del deployment.

**c.** Con el servicio **AWS CodePipeline**, para cada microservicio se crea un pipeline en el que se establecen las sources de los artifacts y ECS como compute para las tareas asociadas al deployment. En particular, se especifican dos sources: CodeCommit, que proveerá un primer *input artifact* conteniendo la task definition del microservicio y el correspondiente appspec file, y ECR, que proveerá un segundo input artifact que incluye la imagen que deben correr las tasks. Ambas source actions tienen *change detection* automatizado por lo que, aún cuando no se especifiquen *triggers*, el pipeline se iniciará ante cambios en cualquiera de ellas. 

**d.** Finalmente se examina el funcionamiento del deployment, desencadenando los pipelines manualmente y verificando los cambios en la configuración del ALB (en los listeners y en las vinculaciones de los TG) consistentes con la estrategia de depoyment blue/green en uso. 


### 6. Evaluación de situaciones de uso real

**a.** Se evalúa la posibilidad de gestionar las listener rules y con ellas el ruteo y acceso a los microservicios. En particular, se limita el acceso a un solo IP para el microservicio ***employee*** utilizando una listener rule más restrictiva y se verifica que el microservicio queda inaccesible para otras direcciones IP.

**b.** Se ensaya la situación diaria de desarrollo en la que se modifica el código, se "rebuildea" la imagen y se hace un "push" al registry de ECR. Como es esperable, eso dispara de forma automatizada una ejecución del pipeline correspondiente a ese microservicio que resulta en un deployment con los cambios introducidos sin afectación de la disponibilidad del servicio en producción.

**c.** También se experimenta el escalado de un servicio de forma independiente del otro, en este caso de forma manual.
- Para ello, primero se utiliza el comando `aws ecs update-service` de la AWS CLI indicando el cluster y servicio correspondiente, y selecionando un valor para el parámetro `desired-count` mayor al valor 1 seteado inicialmente al registrar el servicio. Inmediatamete ECS comienza a provisionar las task necesarias para llegar al número elegido y luego de unos minutos están totalmente operativas en estado *running*.
- Luego, de forma inversa, se hace un *scale in*; se disminuye el numero de replicas deseadas y las excedentes inician un proceso de desactivación que luego de unos minutos las lleva al estado *stopped*.
- Finalmente, definiendo el valor 0 para el parámetro `desired-count`, se detienen todas las tasks del servicio.

***

# Diagramas


## Arquitectura 

A continuación se ilustra la arquitectura de microservicios que resulta de los cambios detalllados. Se incluyen la AWS region, las Availability Zones, la VPC, las subnets con sus CIDR blocks, el Application Load balancer, el Interet gateway, las Route tables, las instancias EC2 (la de la appliación original[^1] y la utilizada para desarrollo), la instancia RDS, el cluster ECS con sus servicios y contenedores ejecutando las tasks[^2], los security groups, etc.

<p align="center">
    <img src="img/imgA.svg" alt="Arquitectura de microservicios" width="1000" title="Arquitectura de microservicios"/>
</p>

[^1]: Se incluye la EC2 instance donde corre la aplicación monolítica original porque como parte del laboratorio no se ha desaprovisionado, pero en el caso real se eliminaría ya que no forma parte de la nueva arquitectura.

[^2]: La cantidad de contenedores ilustrada para cada servicio en el ECS cluster (2 y 3) es solo esquemática, ya que depende de la escala deseada y precisamente no deben necesariamente ser iguales para los dos microservicio ni para un mismo microservicio a lo largo del tiempo.


## Flujo de CI/CD 

En el diagrama siguiente se esquematiza el flujo utilizado para incorporar CI/CD al ciclo de desarrollo[^3]. Se incluye la instancia de desarrollo creada por Cloud9 junto con los repositorios remotos en CodeCommit y los image registries en ECR. Los logos de rayo ilustran cómo, al enviarse cambios (*push*) hacia el remoto *deployment* o hacia alguno de esos registries, se dispara una ejecución del pipeline correspondiente al servicio modificado. Ambas fuentes proveen input artifacts que luego utiliza CodeDeploy para hacer un deployment blue/green utilizando el compute de ECS.

<p align="center">
    <img src="img/imgB.svg" alt="Flujo de CI/CD" width="800" title="Flujo de CI/CD"/>
</p>

[^3]: Por razones de espacio se omiten o simplifican parcialmente algnos elementos.

***

# Estimación de costo

Para concluir, se incluye un pequeño ejericio de estimación de costos efectuado con la herramienta [***AWS Pricing Calculator***](https://calculator.aws/).

Nótese que se trata de un ejercicio ilustrativo ya que una estimación concreta requiere conocer la demanda real que se prevé tendrá la aplicación. En función de ello se estimarían lo más certeramente posible los parámetros que afectan de forma más significativa al costo total (carga que debe manejar el load balancer, cantidad de tasks que debe correr cada servicio en ECS, memoria y CPU que debe usar cada contenedor e incluso el tipo de instancia a usar para alojar la DB) y con ello se podria efectuar una estimación del costo real que puede tener la aplicación en producción.

A falta de ello y con fines didácticos, se realiza una estimación ([disponible aquí](https://calculator.aws/#/estimate?id=0e626006f323e4d8c10f2b74b3d6017a616624b4)) basada en un uso propuesto de la aplicación de hasta 900 conexiones activas. Como complemento, a continuación se listan los recursos para los que se estimaron costos, acompañados de las principales consierdaciones empleadas o detalles relevantes a tener en cuenta:

**a. VPC**:
- La VPC en sí misma no tiene costo, pero sí algunos de sus servicios, entre ellos las direcciones IPv4 públicas. Aquí se emplean varias:
  - Una utilizada por la instancia EC2 de DEV, necesaria para acceder a los servicios que ejecutamos en docker containers durante el desarrollo, asignada automáticamente por iniciarse la instancia en una subnet (*Public Subnet 1*) que tiene activada la opción de *auto-assign public IPv4 address*.
  - Dos *service managed public IPv4* utilzadas por el Application Load Balancer, y assignadas al momento de crearlo de tipo *internet-facing*, una por cada subnet a la que está asociado.
  - Una asignada a cada contenedor ejecutándose en el ECS cluster. La asignación ocurre automáticamente al iniciarse cada uno porque los *ECS services* fueron creados con la opción `assignPublicIp` habilitada, y es necesaria porque los contenedores deben poder acceder a servicios de AWS cuyo acceso es via internet (eg ECR).[^4]

  Dado que se prevé ejecutar un promedio de cinco contenedores (ver más adelante), se tienen en total ocho direcciones.

[^4]: Obsérvese que podría evitarse la asignación de estas IP públicas, pero debería garantizarse de alguna forma alternativa que los servcios de AWS requeridos (eg ECR) siguen siendo accesibles. Por ejemplo, podrían desplegarse los contenedores en una subnet privada y utilizar un NAT Gateway o directamente hacierse esos servicios accesibles via *VPC endponts* usando ***AWS PrivateLink***. 

**b. Application Load Balancer**:
- Tiene un costo fijo y un costo varable dado por las load balancer capacity units (LCUs), que se estiman en base al promedio de conexiones activas proyectado.

**c. RDS**
- Se utilizan los parámetros de la instancia RDS ya creada por el lab (familia de instancia, storage, single-AZ), pero para manejar el mayor número de conexiones máximo se modifica el tipo de instancia a **db.t3.large**. A su vez, dado que debe funcionar de forma permanente, resulta conveniente cambiar el pricing modely pasar a usar una instancia reservada, tomándose un compromiso de un año sin pago por delantado que implica un ahorro de alrededor de un tercio del costo.
- No se considera el uso de RDS Proxy, Perfomance Insights, Extended support ni extra storage para back up por fuera del incluido (de igual tamaño que el storage principal de la DB).

**d. EC2 (instancia DEV)**
- Se utiliza el tipo de instancia creada siguiendo las instrucciones del lab (tipo de instancia, storage EBS, pricing model) y estimando un uso prinicpal de 9 horas los dias de semana. Cabe notar que en este caso, dado el uso no permanente, no resulta conveniente tomar un compromiso de más largo plazo con *saving plans*.
- Nótese que la instancia es una **t3.small**, pero observando los logs de los períodos en que se trabajó con ella para el labratorio, su poder de cómputo está muy subutiizado, por lo que podría usarse una **t3.micro** que tiene un costo 50% menor.

**e. ECS (microservicios)**
- Se utilizan los parámetros de CPU y memoria definidos en las task definitions.
- Se estiman un *promedio* de cinco tasks funcionando, una para el servicio ***employee*** y cuatro para ***customer***, dada la mayor demanda esperada para este úlitmo.

**f. ECS (deployments)**
- Se utilizan los parámetros de CPU y memoria definidos para los servicios en las task definitions y se estiman 3 deploys por día, con una duración de 5 min cada uno.

**g. CodePipeline**
- Se consideran los dos pipelines creados. AWS incluye uno sin cargo por lo que solo uno genera un costo adicional.

**h. ECR**
- Se consideran 10 Gb de almacenamiento en los registries que, dado el pequeño tamaño de las imágenes (< 30 Mb) es suficiente para conservar unas 150 versiones de cada una de las dos.


Notar que **no se incluyen**:
- La instancia EC2 en la que corre la aplicación "monolítica" original. Como de dijo, si bien no se desaprovisionó durante el laboratorio en una situación real se eliminaría ya que su función es reemplazada por el cluster ECS. Por la misma razón no se considera su dirección IPv4 pública en la cotización de la VPC.
- El servicio de CodeDeploy, dado que tiene costo adicional solo en caso de ejecutarse deployments usando máquinas on-premise como compute. En este caso el compute utilizado para ello es ECS, incluido en el listado previo.  
- El servicio Cloud9, puesto que no tiene costo adicional por fuera de la instancia EC2 y el EBS asociado a la misma, ya incluidos.
- El servicio de CodeCommit, dado que es sin cargo hasta 5 usuarios activos y con límites de storage y git requests por encima de lo requerido en un caso como este.
- Otros servicios que se utilizan indirectamente, e implican costo adicional, pero que son de dimensión despreciable para esta situación (eg uso de S3 para almacenar los input artifacts que se generan durante el deployment a partir del repositorio deployment de CodeCommit).