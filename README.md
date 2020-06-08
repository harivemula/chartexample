# Creating a Helm Chart using helm3 for an application.

Install helm (version >3.x) cli, Follow the steps described in this [link](https://helm.sh/docs/intro/install/).
Run the below command to ensure the version.
```
$ helm version --short
```
Go to the directory where you would like to create the chart.
Run the below command, which creates chart with pre-defined templates. Use the name of the chart as your application/module name.
```
$ helm create <name>

$ helm create chartexample
```
This will create the folder with the name of the chart in current working directory.
Change to the newly created directory (chartexample) and view the contents

```
$ cd chartexample
```
```
$ tree
.
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```
- Chart.yaml
  - This defines the chart and its key properties.
- values.yaml
  - Define all the configurable properties for the chart deployment.
- charts
  - You may add any dependent chart(s) into this directory
- templates
  - A directory of templates that, when combined with values, will generate valid Kubernetes manifest files. The templates are written using [Go Template Language](https://golang.org/pkg/text/template/).

## Chart.yaml

`apiVersion: v2`  - v2 represents helm3 compatible charts
`name: chartexample` - name of the chart
`type: application` - Default is application. Other supported value is `Library`
`version: 0.1.0` - This is the chart version, increment this version if you change the app version or any change to the deployment templates.
`appVersion: 1.16.0` - Your application version. Change whenver there is any change to the application.

Apart from the above there are many optional fields available.

## values.yaml
The default configuration values for this chart. This comes with pre-defined set of properties which are used in the templates. You may add or modify them, remember to change the templates which depends on these default properties.

In my example, i would like to change the below in values.yaml
```yaml
image:
  repository: harivemula/kubeconfigexample
...
serviceAccount:
  create: false
...

resources:
  limits:
    cpu: 500m
    memory: 750Mi
  requests:
    cpu: 500m
    memory: 750Mi

...
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 2
  targetCPUUtilizationPercentage: 80
```

I'm introducing few more properties in values.yaml
```yaml

port:
  containerPort: 8080

volumeMounts:
  - name: cache-volume
    mountPath: /cache
    
volumes:
  - name: cache-volume
    emptyDir: {}
    
env:
  - name: DEMO_ENV_PROP
    value: mydemoenvproperty
  - name: ENV_PROP
    value: justenvprop
```

In the above content, i have used the same structure of **kubernetes manifest** for adding new properties, however it is not manadatory. It all depends on how you want to use the  property in templates.

**Example**, we have defined `port.containerPort` in the above example, and going to use this in the `templates/deployments.yaml` as 
```yaml
ports:
- name: http
  containerPort: {{ .Values.port.containerPort }}
```
Instead you may define simply as `container_port: 8080` in *values.yaml* and refer it in `templates/deployment.yaml` like below
```yaml
ports:
- name: http
  containerPort: {{ .Values.container_port }}
```

## templates
By default templates directory contains templates for below kubernetes objects.
* deployment
* service
* service account
* ingress
* horizontal pod autoscaling

You may add any other objects you need, like configmap.

The `_helpers.tpl` file defines named templates. 
**Example**, it defines a templete to derive name of the chart. Go to the [link]([https://helm.sh/docs/chart_template_guide/named_templates/](https://helm.sh/docs/chart_template_guide/named_templates/)).
``` go
{{- define  "chartexample.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc  63 | trimSuffix  "-" }}
{{- end }}
```

### Modifying the templates
Modify the provided templates to suite to your requirement. In my case i have modified the `deployment.yaml`  with highlighted fields below.

```yaml
...
          ports:
            - name: http
              containerPort: {{ .Values.port.containerPort }}
              protocol: TCP
...
          env:
            {{- toYaml .Values.env | nindent 12 }}
...
          volumeMounts:
            - name: config-volume
              mountPath: /workspace/config
            {{- toYaml .Values.volumeMounts | nindent 12 }}
...
      volumes:
        - name: config-volume
          configMap:
            name: {{ .Release.Name }}-samplespringconfig
        {{- toYaml .Values.volumes | nindent 8 }}

...

```

In the above lets take couple of examples
```yaml
env:
  {{- toYaml .Values.env | nindent 12 }}
```
**env**: As in my values.yaml, i have defined the same structure it should be in kubernetes manifest, now i am trying to simply get the `env` property values and copying here as yaml output with indent 12.


```yaml
volumes:
  - name: config-volume
    configMap:
      name: {{ .Release.Name }}-samplespringconfig
  {{- toYaml .Values.volumes | nindent 8 }}
```
**volumes**: I have defined a volume with configmap as source here in the template directly and if any other volumes defined on values.yaml, all will be added here as yaml with indent 8.
And the name of the configmap is based on the chart release name.

### Adding new object template
I would like to create configmap to pass some properties to my application and use the configmap as volume to deployment. For that, i am adding a new file in templates directory as `configmap.yaml`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
name: {{ .Release.Name }}-samplespringconfig
data:
application.properties: |
demo.file.prop=ValueFromConfig
```
In the configmap, i am using the name as Release.Name prefix, which is the name we give when we deploy the chart. So, the configmap object name is dynamic. 
You can use templates in the data section as well.

## Package Chart

Once you have completed with your modifications, you can package the chart.
Before packaging, verify if the chart is well formed by running 
```sh
helm lint
```

Run the below to package chart
```sh
helm package .
```
This will package your chart with name < chartname >-< chartversion >.tgz. 
**Example**, chartexample-0.1.0.tgz


