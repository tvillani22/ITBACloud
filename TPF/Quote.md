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

<br />

[Volver al README](../README.md)