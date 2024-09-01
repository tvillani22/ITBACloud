# TP Final Módulo 1

Este repositorio contiene el código del laboratorio ***Creación de microservicios y una canalización de CI/CD con AWS*** de AWS Academy y el detalle del desarrollo mismo.

En relación al primero, en el directorio [Cloud9Environment](Cloud9Environment) se incluye el último estado del código de los dos repositorios utilizados durante el laboratorio, junto con los JSONs empleados para crear los servicios en ECS.

En cuanto al desarrollo, el proyecto propuesto por el laboratorio consiste en llevar a cabo una serie de pasos con dos objetivos principales: implementar los cambios que permitan pasar de una applicación ejecutándose de forma *monolítica* a una que lo haga utiizando *microservicios*, y llevarlo a cabo creando para ello un entorno de desarrollo que incorpore prácticas de *integración y despliegue continuo (CI/CD)*.  

Respecto al primer objetivo, excluyendo la DB (que funciona en una instancia de RDS), la aplicación en su estado original se ejecuta íntegramente en una instancia EC2. Por un lado, ello conlleva dificultades en la escalabilidad ante variaciones en la demanda, dado que para ampliar la capacidad de una parte de la aplicación debe escalarse la totalidad de la misma, teniéndose una relación de compromiso: si se escala para el servicio más demandado, se incurre en costo adicional innecesario dado que el otro no lo requiere; si se mantiene la escala para el servicio menos requerido se generará un cuello de botella en el más solicitado. A su vez, una estructura monolítica tiene menor *fiablilidad* o *reliabiity* dado que siemple implica un *single point of failure*. Ambos elementos se evitan con una solución basada en microservicios; tal arquitectura admite la posibilidad de escalar los servicios de forma independiente para adaptarse a la demanda de cada uno y al mismo tiempo da más robustez al permitir que si uno de los servicios está fuera de operación el resto permanezca funcional. 

En relación el segundo objetivo, cada vez que se efectuan cambios en el código de la aplicación, para que los mismos lleguen a impactar en el entorno productivo deben ejecutarse una serie de tareas de compilación, testeo y deployment. Implementar prácticas de CI/CD en el entorno implica precisamente automatizar estas tareas de forma tal de agilizar el ciclo de desarrollo haciendo posible iterar con mejoras continuas que resultan en actualizaciones frecuentes y confiables de la aplicación.

De acuerdo a lo solicitado, dentro del repo se incluye:
- [Detalle de los pasos efectuados durante el laboratorio](TPF/Steps.md).
- [Diagramas de la arquitectura y el flujo de CI/CD implementado](TPF/Diagrams.md).
- [Ejemplo ilustrativo de estimación de costos](TPF/Quote.md).