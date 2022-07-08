# QotD Deployment with Helm 3

A helm 3 chart has been prepared to deploy the app to a single kubernetes cluster. If the target cluster is not OpenShift then the UI services (main app, load generator and anomaly generator) must use NodePorts to provide access to external clients.

It is assumed that [Helm 3](https://v3.helm.sh/) is already installed on the workstation.

## Install Helm Chart

Add the QotD chart to the local repository. By default the latest version of the chart is used.  

```bash
helm repo add qotd https://gitlab.com/api/v4/projects/26143345/packages/helm/stable

helm repo update

```

The currently published versions of this chart are found here: [QotD Package Registry](https://gitlab.com/quote-of-the-day/quote-of-the-day/-/packages).

| Chart version  |  QotD app version |
|----------------|-------------------|
| 1.0.0          |  3.1.0            |
| 2.0.0          |  4.0.0            |
| 2.0.1          |  4.0.1            |
| 2.0.2          |  4.0.1 (default)  |

The basic command to install a helm chart is:

`helm install [RELEASE NAME] [REPO NAME/CHART NAME] [flags]`

Where in our case the `[REPO NAME/CHART NAME]` value is `qotd/qotd`.  If you choose a release name like `qotd` to be the instance name then the install command will look a little redundant, but is nonetheless correct:

```shell
helm install qotd qotd/qotd \
...
```

## Instana 

QotD has been instrumented for Instana.  If you expect Instana will be collecting information, then you should enable the integration with the following flag:

```bash
helm install qotd qotd/qotd \
    --set host=apps.mycluster.com \
    --set enableInstana=true

```

If the Instana server is configured for QotD as a Web Application, you must pass in the reporting URL, JavaScript URL, and the Instana key as provided by the Web Application definition in the Instana server.

```bash
helm install qotd qotd/qotd \
    --set host=apps.mycluster.com \
    --set instanaReportingUrl=http://instana-server.example.com:2999/ \
    --set instanaEnumMinJsUrl=http://instana-server.example.com:2999/eum.min.js \
    --set instanaKey=8nGj0E7iT6C2ZmVkP7UfgQ \
    --set enableInstana=true

```

## Namespaces/projects

The default projects for the application and load/anomaly generators are `qotd` and `qotd-load` respectively.  These can be changed with the `appNamespace` and `loadNamespace` helm variables (notice the release name is `qotd2` this is just in case you still have a release called `qotd` created with the above examples).  For example:

```bash
helm install qotd2 qotd/qotd \
    --set host=apps.mycluster.com \
    --set appNamespace=qotd2 \
    --set loadNamespace=load2 

```

## Branding

You can add a "branding string" to the user interfaces with the `branding` helm variable.

```bash
helm install qotd qotd/qotd \
    --set host=apps.mycluster.com \
    --set branding="QotD - A demo app for the literate"

```


## Independent loggers

The flag `setDefaultLoggers` when set to true will define two indepenent loggers, one with the ratings service and the other with the PDF service.

The format of the logger in the rating service is:
```txt
Heartbeat status: {{STATUS}}
```
Where the value of `{{STATUS}}` is either [ `OK` | `SLOW` ] with the value `OK` occuring 80% of the time, every 30 seconds.

The format of the PDF service logger is 

```txt
incident error code: E{{v1}} epid={{v2}} dsteuid={{v3}} dstepid={{v4}} vd={{user}} type={{type}} subtype={{subtype}}

```

Where the fields `v1` through `v4` are randomly generated 4 digit integers, the `type` field is one of [ `access` | `update` | `weight` ], and `subtype` is one of [ `read` | `write` | `delete` ].

If you want to have other default loggers, or change the default "factory settings" in any way, then all you have to do is modify this configmap after the chart is installed with the (`setDefaultLoggers` option set to true) and restart all the pods.  They will pick up this new default configuration and start generating logs defined within them.

## Additional requirements for OpenShift 3.x/4.x 

When installing Before installing with Helm, you must create the namespaces and set the security contexts to allow root access (a current requirement for the database and Selenium based load generator).  The default namespaces for the chart are `qotd` and `qotd-load`.

```bash
oc new-project qotd
oc adm policy add-scc-to-user anyuid -z default

oc new-project qotd-load
oc adm policy add-scc-to-user anyuid -z default

```

For OpenShift deployments the web and anomaly genertor services have exposed routes. Therefore you need to set the base *apps* host URL while installing the chart.  For example if the OCP cluster `apps` base URL is `apps.mycluster.com` then you can install the chart with:

```bash
helm install qotd-chart qotd/qotd \
    --set enableInstana=true \
    --set host=apps.mycluster.com 

```

### ROKS installation

On IBM RedHat Openshift Kubernetes Service (ROKS) installations the base url for apps do not begin with `apps.`

**TODO: get instructions for determining base url**

In order to get the Instana collector to connect to the agent properly you need to include an additional parameter to the helm install command to set `roksCluster` to `true`.

```bash
helm install qotd-chart qotd/qotd \
    --set host=myhost-0d255d996fcd6ab-0000.us-south.containers.appdomain.cloud \
    --set roksCluster=true

```


## Kubernetes NodePort Installation

Before installing with Helm, you must create the namespaces and set the security contexts to allow root access (a current requirement for the database and Selenium based load generator).  The default namespaces for the chart are `qotd` and `qotd-load`.

```bash
kubectl create namespace qotd
kubectl create namespace qotd-load

# optionally set the default namespace to qotd
kubectl config set-context --current --namespace=qotd

```

For Generic Kubernetes deployments the three UI services; main app, load and anomaly generators must use a NodePort service.  The chart makes some assumptions on the allocated port numbers, that should be verified after deployment.

To install first use `kubectl` to log into the kubernetes cluster.  The `-n` argument is the namespace on the target cluster to begin the install with.

```bash
helm install qotd qotd/qotd  --set useNodePort=true -n qotd 
 
```

QotD has been instrumented for Instana.  If you expect Instana will be collecting information, then you should enable the integration with the following flag:

```bash
helm install qotd qotd/qotd \
    -n qotd \
    --set useNodePort=true \
    --set enableInstana=true

```

If the Instana server is configured for QotD as a Web Application, you must pass in the reporting URL, JavaScript URL, and the Instana key as provided by the Web Application definition in the Instana server.

```bash
helm install qotd qotd/qotd \
    -n qotd \
    --set useNodePort=true \
    --set instanaReportingUrl=http://instana-server.example.com:2999/ \
    --set instanaEnumMinJsUrl=http://instana-server.example.com:2999/eum.min.js \
    --set instanaKey=8nGj0E7iT6C2ZmVkP7UfgQ \
    --set enableInstana=true

```

The default projects for the application and load/anomaly generators are `qotd` and `qotd-load` respectively.  These can be changes with the `appNamespace` and `loadNamespace` helm variables.  For example:

```bash
helm install qotd qotd/qotd \
    -n qotd2 \
    --set useNodePort=true \
    --set appNamespace=qotd2 \
    --set loadNamespace=load2 

```

## Examples


