## Pasos

&#9745; Añadir al menos un endpoint  
&#9745; Test unitarios para los nuevos endpoints  
&#9745; Creación de pipelines de CI/CD en cualquier plataforma (Github Actions, Jenkins, etc) que cuenten por lo menos con las fases: 
- &#9745; Testing: tests unitarios con cobertura. Se dispone de un [ejemplo con Github Actions en el repositorio actual](./.github/workflows/test.yaml)
- &#9745; Build & Push: creación de imagen docker y push de la misma a cualquier registry válido que utilice alguna estrategia de release para los tags de las vistas en clase, se recomienda GHCR ya incluido en los repositorios de Github. Se dispone de un [ejemplo con Github Actions en el repositorio actual](./.github/workflows/release.yaml)  

&#9745; Configurar monitorización mediante prometheus en los nuevos endpoints añadidos  
&#9745; Desplegar prometheus a través de Kubernetes mediante minikube y configurar alert-manager para por lo menos las siguientes alarmas, tal y como se ha realizado en el laboratorio del día 3 mediante el chart `kube-prometheus-stack`:  
- &#9745; Uso de CPU de un contenedor mayor al del límite configurado, se puede utilizar como base el ejemplo utilizado en el laboratorio 3 para mandar alarmas cuando el contenedor de la aplicación `fast-api` consumía más del asignado mediante request  

&#9745; Las alarmas configuradas deberán tener severity high o critical  
&#9745; Crear canal en slack `<nombreAlumno>-prometheus-alarms` y configurar webhook entrante para envío de alertas con alert manager  
&#9744; Alert manager estará configurado para lo siguiente: (Para poder comprobar si esta parte funciona se recomienda realizar una prueba de estres, como la realizada en el laboratorio 3 a partir del paso 8.)  
- &#9744; Mandar un mensaje a Slack en el canal configurado en el paso anterior con las alertas con label "severity" y "critical"  
- &#9744; Deberán enviarse tanto alarmas como recuperación de las mismas  
- &#9744; Habrá una plantilla configurada para el envío de alarmas  

&#9744; Creación de un dashboard de Grafana, con por lo menos lo siguiente:  
- &#9744; Número de llamadas a los endpoints
- &#9744; Número de veces que la aplicación ha arrancado