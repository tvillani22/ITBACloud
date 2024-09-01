# Diagramas

## Arquitectura 

A continuación se ilustra la arquitectura de microservicios que resulta de los cambios detalllados. Se incluyen la AWS region, las Availability Zones, la VPC, las subnets con sus CIDR blocks, el Application Load balancer, el Interet gateway, las Route tables, las instancias EC2 (la de la appliación original[^1] y la utilizada para desarrollo), la instancia RDS, el cluster ECS con sus servicios y contenedores ejecutando las tasks[^2], los security groups, etc.

<p align="center">
    <img src="img/Arq.svg" alt="Arquitectura de microservicios" width="1000" title="Arquitectura de microservicios"/>
</p>

[^1]: Se incluye la EC2 instance donde corre la aplicación monolítica original porque como parte del laboratorio no se ha desaprovisionado, pero en el caso real se eliminaría ya que no forma parte de la nueva arquitectura.

[^2]: La cantidad de contenedores ilustrada para cada servicio en el ECS cluster (2 y 3) es solo esquemática, ya que depende de la escala deseada y precisamente no deben necesariamente ser iguales para los dos microservicio ni para un mismo microservicio a lo largo del tiempo.

<br />

## Flujo CI/CD 

En el diagrama siguiente se esquematiza el flujo utilizado para incorporar CI/CD al ciclo de desarrollo[^3]. Se incluye la instancia de desarrollo creada por Cloud9 junto con los repositorios remotos en CodeCommit y los image registries en ECR. Los logos de rayo ilustran cómo, al enviarse cambios (*push*) hacia el remoto *deployment* o hacia alguno de esos registries, se dispara una ejecución del pipeline correspondiente al servicio modificado. Ambas fuentes proveen input artifacts que luego utiliza CodeDeploy para hacer un deployment blue/green utilizando el compute de ECS.

<p align="center">
    <img src="img/CICD.svg" alt="Flujo de CI/CD" width="800" title="Flujo de CI/CD"/>
</p>

[^3]: Por razones de espacio se omiten o simplifican parcialmente algnos elementos.

<br />

[Volver al README](../README.md)