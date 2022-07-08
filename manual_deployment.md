# Manual Deployment

All of the components of the QotD application are built as simple containers.  This means that they can run on Kubernetes or simply as Docker containers on a VM.  This flexibility of deployment options means that components can be easily distributed across hosts.  In fact it is even possible to download the Node.js source and run as a native application with a process manager like [PM2](https://pm2.keymetrics.io/).  This however is beyond the scope of the documentation at this point, and this page will focus on manual deployment to Kubernetes.

If the target cluster is OpenShift then Routes are used to expose the services, otherwise NodePort services are used for the external components with a user interface.  Example sets of kubernetes resources for Openshift and generic Kubernetes can be found in the `resources/k8s/ocp_okd/v3.1.0` and `resources/k8s/generic_k8s/v3.1.0` folders of this Git repository.

## Deployments

Each file contains all the kubernetes resources required for that particular component.  For example the main front end web component, configured for Openshift, has the following deployment configuration:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: qotd-web
  namespace: qotd
  labels:
    app: qotd
    tier: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qotd-web
  template:
    metadata:
      labels:
        app: qotd-web
    spec:
      volumes:
        - name: qotd-web-loggers
          configMap:
            name: qotd-web-loggers
            defaultMode: 420
        # - name: qotd-web-call-distribution-overrides
        #   configMap:
        #     name: qotd-web-call-distribution-overrides
        #     defaultMode: 420
      restartPolicy: Always
      containers:
        - name: qotd-web
          image: registry.gitlab.com/quote-of-the-day/qotd-web:3.1.0
          imagePullPolicy: Always
          volumeMounts:
            - name: qotd-web-loggers
              readOnly: true
              mountPath: /app/loggers
            # - name: qotd-web-call-distribution-overrides
            #   readOnly: true
            #   mountPath: /app/config
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
            - name: svc
              containerPort: 8080
              protocol: TCP
          env:
            - name: LOCAL_LOGGING
              value: "false"
            - name: ENABLE_INSTANA
              value: "false"
            - name: QUOTE_SVC
              value: "qotd-quote.qotd.svc.cluster.local:3001"
            - name: AUTHOR_SVC
              value: "qotd-author.qotd.svc.cluster.local:3002"
            - name: RATING_SVC
              value: "qotd-rating.qotd.svc.cluster.local:3004"
            - name: PDF_SVC
              value: "qotd-pdf.qotd.svc.cluster.local:3005"
            - name: ENGRAVING_SVC
              value: "qotd-engraving.qotd.svc.cluster.local:3006"
            - name: INSTANA_REPORTING_URL
              value: ""
            - name: INSTANA_ENUM_MIN_JS_URL
              value: ""
            - name: INSTANA_KEY
              value: ""
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            requests:
              cpu: "50m"
              memory: "200Mi"
            limits:
              cpu: "200m"
              memory: "800Mi"

```

You'll notice that there is a config map reference that is commented out (and one that is not). These will be discussed later.

This web service exposes two ports; 3000 and 8080.  The first port; 3000,  is for all the business application logic.  The second port; 8080, is used by the anomaly generator to tell this web application to induce some type of anomaly.  All the main services of the QotD (except the DB) application work this way.

Since the web service communicates directly with the quote, author, rating, pdf and engraving services.  The endpoint of each of these needs to be set in an environment variable, so that the web component knows how/where to connect to the other service.  By specifying service connections this way, the components can be placed anywhere (even outside of the cluster).  

If all the components are deployed to the same kubernetes namespace, then the internal URLs can be used.  These are of the form:

`qotd-<component>.<app namespace>.svc.cluster.local:<port>`

The component names are; `web`, `quote`, `author`, `image`, `rating`, `pdf`, `engraving`, and `db`.

The default port numbers for each of the components are different.  This is intentional and indended to make it easier to debug all components simultaneuosly on the same workstation.  You can find the port numbers of each service by looking at the service definition resource for the main app.  

The default set of values assumes all components are running in the same namespace on the same cluster.

```yaml
- name: QUOTE_SVC
  value: "qotd-quote.qotd.svc.cluster.local:3001"
- name: AUTHOR_SVC
  value: "qotd-author.qotd.svc.cluster.local:3002"
- name: RATING_SVC
  value: "qotd-rating.qotd.svc.cluster.local:3004"
- name: PDF_SVC
  value: "qotd-pdf.qotd.svc.cluster.local:3005"
- name: ENGRAVING_SVC
  value: "qotd-engraving.qotd.svc.cluster.local:3006"
```

### Instana

The `ENABLE_INSTANA` variable is a boolean that turns on the Instana instrumentation and should be set to true if an Instana agent is configured to collect data from this component.

If an Instana Web Application is defined, then we need to inject some JavaScript into the applications web pages.  This is done with the three environment variables; `INSTANA_REPORTING_URL`, `INSTANA_ENUM_MIN_JS_URL` and `INSTANA_KEY` which is obtained from the Web Application definition on the server.

Some example values are:
```yaml
- name: INSTANA_REPORTING_URL
  value: "http://instana-server.example.com:2999/"
- name: INSTANA_ENUM_MIN_JS_URL
  value: "http://instana-server.example.com:2999/eum.min.js"
- name: INSTANA_KEY
  value: "8nGj0E7iT6C2ZmVkP7UfgQ"

```

### Load Generator

The Load generator invokes the application functionality as a user.  An actual browser instance (currently Firefox on Linux) is used by [Selenium Webdriver](https://www.selenium.dev/documentation/webdriver/) to invoke actions as a user would.  A fixed suite of actions covering all of the applications functionality are exercised over and over again.

The Load generator has three envirionment variables that should be set.  When `AUTOSTART` is set to true (default), then the load generator will start as soon as it is loaded.  This is useful when you are running multiple replicas to generate a larger volume of logs per time.

```yaml
- name: AUTOSTART
  value: "true"
```

The `QOTD_WEB_HOST` variable is the main home page of the application.  When the application is deployed on OCP/OKD where the default network policies prevent direct communication across namespaces, the form of this value should be the actual external URL for the web app. For deployments on generic kubernetes the external services might be defined as NodePort services, so the URL would be the URL of the cluster with the specific port for the web service.

```yaml
- name: QOTD_WEB_HOST
  value: "http://qotd-web-qotd.apps.{{apps host}}/"
- name: RUNS_PER_HOUR
  value: "90"

```

Finally the `RUNS_PER_HOUR` variable sets the number of runs of the full suite of scenarios on the application.  Valid values are anything from 0 to 90.  The rate of 90 runs of the suite per hour is about the maximum that can be expected.  Each of the runs include realistic time delays of about two seconds before the next click is made.  This makes each suite take from 38 to 85 seconds to complete, depending on local conditions and expected variations in application behavior.  Each run of the suite generates approximately 190 log entries accross all the components.

### Anomaly Generator

The anomaly generator (`qotd-usecase` component) makes connections to all the main application components, but using the second exposed port.  So for generic Kubernetes where connection across namespaces are allowed the following values would wor when the application is installed in the `qotd` namespace. 

```yaml
- name: QOTD_WEB_SVC
  value: "http://qotd-web-svc.qotd.svc.cluster.local:8080"
- name: QOTD_QUOTE_SVC
  value: "http://qotd-quote-svc.qotd.svc.cluster.local:8081"
- name: QOTD_AUTHOR_SVC
  value: "http://qotd-author-svc.qotd.svc.cluster.local:8082"
- name: QOTD_IMAGE_SVC
  value: "http://qotd-image-svc.qotd.svc.cluster.local:8083"
- name: QOTD_RATING_SVC
  value: "http://qotd-rating-svc.qotd.svc.cluster.local:8084"
- name: QOTD_PDF_SVC
  value: "http://qotd-pdf-svc.qotd.svc.cluster.local:8085"
- name: QOTD_ENGRAVING_SVC
  value: "http://qotd-engraving-svc.qotd.svc.cluster.local:8085"

```

For OCP/OKD deployments the anomaly generator should use the externally acessible URL as defined by the Route resources.  If the OCP cluster uses `apps.mycluster.com` for its routes, then a sample set of calues for the anomaly generator would be the following.  Note that the URL points to the second (svc) endpoint, not the main application one.

```yaml
- name: QOTD_WEB_SVC
  value: "http://qotd-web-svc-qotd.apps.mycluster.com/"
- name: QOTD_QUOTE_SVC
  value: "http://qotd-quote-svc-qotd.apps.mycluster.com/"
- name: QOTD_AUTHOR_SVC
  value: "http://qotd-author-svc-qotd.apps.mycluster.com/"
- name: QOTD_IMAGE_SVC
  value: "http://qotd-image-svc-qotd.apps.mycluster.com/"
- name: QOTD_RATING_SVC
  value: "http://qotd-rating-svc-qotd.apps.mycluster.com/"
- name: QOTD_PDF_SVC
  value: "http://qotd-pdf-svc-qotd.apps.mycluster.com/"
- name: QOTD_ENGRAVING_SVC
  value: "http://qotd-engraving-svc-qotd.apps.mycluster.com/"

```

### Engraving Service

The engraving service is a special service that is intended to connect to a REST service and delivery a JSON payload that will "order an engraving of the selected quote".  In the demo environment it is expected that this REST endpoint is provded by IBM App Connect Enterprise and orchestrates a IBM MQ message push.  All of this should be visible in an Instana monitored trace.  The setup and configuration of the ACE and MQ components is beyond the scope of this document.

If the ACE and MQ flows are configured then the `SUPPLY_CHAIN_URL` should be set with the ACE URL, and the `SUPPLY_CHAIN_SIMULATE` value set to false.  If there is no ACE/MQ configured this action can be simulated by setting the `SUPPLY_CHAIN_SIMULATE` to true.

```yaml
- name: SUPPLY_CHAIN_URL
  value: ""
- name: SUPPLY_CHAIN_SIMULATE
  alue: "true"
```

## Default Loggers

Default loggers are log entries that are generated by a component even if that component is not experiencing any load.  These logs, like all logs, can be configured to display text along with randomly generated fields of different types, and at any frequency.  Typical examples of log information this would be "heartbeat" logs which get generated regardless of user load of the service.  

By default two components; ratings and web come with default logger definitions. These are defined in ConfigMaps.  You can add or remove content from these as you like.

The ConfigMap that defines the default loggers for the web service is:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: qotd-web-loggers
  namespace: qotd
data:
  incident.json: |
    {
        "id": "incident",
        "name": "Simulated incident record",
        "template": "Web service disk status notifier: code=D{{`{{v1}}`}} dstepid={{`{{v2}}`}} vd={{`{{user}}`}}",
        "fields": {
            "v1": {
                "type": "randomInt",
                "min": 1000,
                "max": 9999
            },
            "v2": {
                "type": "randomInt",
                "min": 1000,
                "max": 9999
            },
            "user": {
                "type": "userName"
            }
        },
        "repeat": {
            "mean": 13000,
            "stdev": 200,
            "min": 11000,
            "max": 15000
        }
    }

  heartbeat.json: |
    {
      "id": "heartbeat",
      "name": "A regular log message indicating the health with a simple status.",
      "template": "Web pulse status: {{`{{STATUS}}`}}",
      "fields": {
          "STATUS": {
              "type": "pickWeighted",
              "options": [
                  { "value": "GOOD", "weight": 8 },
                  { "value": "POOR", "weight": 2 }
              ]            
          }
      },
      "repeat": {
          "mean": 4000,
          "stdev": 1000,
          "min": 3000,
          "max": 5000
      }
    }

```

This ConfigMap defines two JSON files; `incident.json` and `heartbeat.json`.  The heartbeat file (simpler of the two), defines a regularly occuring log, about every 4 seconds (4000ms), with a slight variation (std: 1s).  The log format itself is a simple string with one variable; `{{STATUS}}`

```
Web pulse status: {{STATUS}}
```

The variable can be one of two values; `GOOD` or `POOR`, where the value `GOOD` appears 8 our of 10 times, and the value `POOR` 2 out of 10 times.

Full details of how to define a logger can be found in the [Anomaly Generator](anomaly_generator.md) documentation.

The ConfigMap must be mounted to the pod's `/app/loggers` directory, in order for the component to pick it up and create the loggers.

Default loggers can be created for any of the main application components (web, quote, author, image, rating, pdf, and engraving).

## Call Distribution Overrides

A call service override definition allows us override the normal call status (HTTP status) return for any endpoint used in the application.  A special identifier is associated with each service's endpoints that can be overridden.  

Only the Rating service comes with a call distribution override in the default packaging. You can add as many as you like by defining new ConfigMaps and mounting them to the pod's `/app/config` directory.

The defautl rating service call override is:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: qotd-rating-call-distribution-overrides
  namespace: qotd
data:
  overrides.json: |
    {
      "ratings_id": [
        {
            "code": 0,
            "weight": 99
        },
        {
            "code": 404,
            "weight": 1,
            "payload": "404 Not found",
            "log": {
                "template": "Resource not found: {{`{{resourceType}}`}}",
                "fields": {
                    "resourceType": {
                        "type": "pickWeighted",
                        "options": [
                            {
                                "value": "disk",
                                "weight": 6
                            },
                            {
                                "value": "memory",
                                "weight": 4
                            }
                        ]
                    }
                }
            }
        }
      ]
    }
```

This ConfigMap defines one json file called `overrides.json`.  This is a fixed filename, and must stay the same for all components.  The number of overrides in each however can vary.

In this example there is only one call override defined for the Rating endpoint; `ratings_id`. In this definition there are two overrides defined, the first is a special one with a code value of `0`.  The weight of this call response is 99 out of every 100 times (the total is obtained by adding all the weights of all the overrides for this endpoint).  The status of 0 means that the call should not be overriden, and instead processed as normal.  The second value in this array defines a code of `404`, with a weight of 1 out of 100.

The `payload` property defines the value that the message body should return.  The `log` property defines the log entry that will be added to the logs each time this call response is activated.  The `template` and `fields` property of `log` are of the same form as the default loggers, and the induced loggers as part of the anomaly generator.

The full listing of internal and external endpoints that can be overriden are listed in the table below.

**Table: Call Service Overrides Endpoint IDs**

| Service      | Method | Endpoint ID     |  Endpoint URL     |
|--------------|--------|-----------------|-------------------|
| Rating       | GET    | `rating_id`     | `/ratings/:id`    |
| Web          | GET    | `daily`         | `/`               |
| Web          | GET    | `random`        | `/random`         |
| Web          | GET    | `author_id`     | `/author/:id`     |
| Web          | GET    | `images_id`     | `/images/:id`     |
| Web          | GET    | `pdf_id`        | `/pdf/:id`        |
| Web          | GET    | `order_id`      | `/order/:id`      |
| Web          | POST   | `post_order_id` | `/order/:id`      |
| Quote        | GET    | `daily`         | `/daily`          |
| Quote        | GET    | `random`        | `/random`         |
| Quote        | GET    | `quotes_id`     | `/quotes/:id`     |
| Author       | GET    | `authors_id`    | `/authors/:id`    |
| Author       | GET    | `images_id`     | `/images/:id`     |
| Engraving    | POST   | `order`         | `/order`          |
| PDF          | GET    | `pdf_id`        | `/pdf/:id`        |
| Image        | GET    | `images_id`     | `/images/:id`     |

