# PRÁCTICA DEL MÓDULO LIBERANDO PRODUCTOS - SRE
### Autor: Aimar Lauzirika


## El Proyecto

Se trata de un servidor web muy sencillo que dispone de 3 endpoints: `/`, `/health` y `/bye` y devuelven un json. También dispone de test unitarios que cubren más del 80% del codigo que es el requerido.

## Pipelines

Tiene dos pipelines que se ejecutan automáticamente:

- Test: Cada vez que el repositorio de GitHub se actualiza se ejecuta el pipeline `./.github/workflows/test.yaml`, que realiza los test unitarios del código.
- Release: Este pipeline consiste en construir la imagen de la app y después subirlo repositorios remotos. Para que se ejecute este pipeline debemos asignar tags a nuestro repositorio. Esto lo conseguiremos si después de asignar un tag al commit con `git tag "v.1.0.1"` lo subimos con `git push --tags`.

## Requisitos

- docker
- docker compose
- minikube
- kubectl
- helm

## Construyendo el entorno

### Slack

Utilizaremos slack para recibir las alertas configuradas. Para ello, crearemos un espacio de trabajo en slack y dentro se crea un canal, por donde recibiremos las alertas.

Además, en slack añadiremos una aplicación llamada `incoming-webhook`, que lo asignaremos al canal recién creado y anotaremos la url que nos indica.

Ahora editaremos el archivo `./kube-prometheus-stack/values.yaml` con dos cambios. En la línea 109 debemos completar el campo `api_url:` con la url que anotamos anteriormente de slack entre comillas simples. Y por otro lado, dos líneas más abajo, en el campo `channel:` pondremos el nombre del canal de slack que habiamos asignado a `incoming-webhook` empezando con el caracter almohadilla.

```yaml
. . .
alertmanager:
    config:
        global:
        resolve_timeout: 5m
        route:
        group_by: ['job']
        group_wait: 30s
        group_interval: 5m
        repeat_interval: 12h
        receiver: 'slack'
        routes:
        - match:
            alertname: Watchdog
            receiver: 'null'
        # This inhibt rule is a hack from: https://stackoverflow.com/questions/54806336/how-to-silence-prometheus-alertmanager-using-config-files/54814033#54814033
        inhibit_rules:
        - target_match_re:
            alertname: '.+Overcommit'
            source_match:
            alertname: 'Watchdog'
            equal: ['prometheus']
        receivers:
        - name: 'null'
        - name: 'slack'
        slack_configs:
        - api_url: 'https://hooks.slack.com/services/YOUR_SLACK_WEBHOOK_HERE' # <--- AÑADIR EN ESTA LÍNEA EL WEBHOOK CREADO
            send_resolved: true
            channel: '#notif-channel' # <--- AÑADIR EN ESTA LÍNEA EL CANAL
. . .
```


### Minikube

Crearemos un perfil de minikube para este proyecto:
```
minikube start --kubernetes-version='v1.21.1' \
    --memory=4096 \
    --addons="metrics-server,default-storageclass,storage-provisioner" \
    -p monitoring-demo
```

#### Prometheus stack

Instalaremos los elementos necesarios para la monitorización con helm:
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

```

```
helm -n monitoring upgrade --install prometheus prometheus-community/kube-prometheus-stack -f kube-prometheus-stack/values.yaml --create-namespace --wait --version 34.1.1
```

## Instalar la aplicación

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

```
helm dep up fast-api-webapp
helm -n fast-api upgrade my-app --wait --install --create-namespace fast-api-webapp
```

Port-forward de los servicios:
```
kubectl -n monitoring port-forward svc/prometheus-grafana 3000:http-web
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090
kubectl -n fast-api port-forward svc/my-app-fast-api-webapp 8081:8081
```

## Monitorización

Ya podemos acceder a la dirección `http://localhost:3000` en el navegador para acceder a Grafana, las credenciales por defecto son `admin` para el usuario y `prom-operator` para la contraseña.

Importamos el dashboard `./dashboard-grafana-aimar.json` y seleccionamos el namespace `fast-api` y el pod `my-app-fast-api-...`.

![](/practica-copia/img/dashboard_grafana.png)

Podemos empezar a realizar diferentes peticiones al servidor de fastapi, es posible ver los endpoints disponibles y realizar peticiones a los mismos a través de la URL `http://localhost:8081/docs` utilizando swagger y así ver las peticiones en grafana.

## Prueba de estrés

Para la prueba de estrés podemos ejecutar los siguientes comandos:
```
export POD_NAME=$(kubectl get pods --namespace fast-api -l "app.kubernetes.io/name=fast-api-webapp,app.kubernetes.io/instance=my-app" -o jsonpath="{.items[0].metadata.name}")
kubectl -n fast-api exec --stdin --tty $POD_NAME -c fast-api-webapp -- /bin/sh
```

Una vez dentro del pod:
```
apk update && apk add git go
```

```
git clone https://github.com/jaeg/NodeWrecker.git
cd NodeWrecker
go build -o extress main.go
```

Empezaremos con la prueba de estrés con el siguiente comando:
```
./extress -abuse-memory -escalate -max-duration 10000000
```

Podemos ver como se empieza a consumir más recursos en grafana y también por la terminal con:
```
kubectl -n fast-api get hpa -w
```

Se debería recibir una notificación como la siguiente en el canal de Slack configurado para el envío de notificaciones sobre alarmas:

  ```
  [FIRING:1] Monitoring Event Notification
  Alert: fastApiConsumingMoreThanRequest - critical
  Description: Pod my-app-fast-api-webapp-585bf945cc-lvvpv is consuming more than limit
  Graph: :gráfico_con_tendencia_ascendente: Runbook: <|:cuaderno_de_espiral:>
  Details:
  • alertname: fastApiConsumingMoreThanRequest
  • pod: my-app-fast-api-webapp-585bf945cc-lvvpv
  • prometheus: monitoring/prometheus-kube-prometheus-prometheus
  • severity: critical
  ```

El Horizontal Pod Autoscaler escalará el número de pods para mitigar este pico y por lo tanto pasado un tiempo debería recibirse una notificación de que la alarma se ha mitigado, tal y como se puede ver en la siguiente captura.

## Terminar

Parar el cluster de minikube creado para esta parte del laboratorio:

```
minikube -p monitoring-demo stop
```