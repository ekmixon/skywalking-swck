# Java agent injector Manual

To use the java agent more natively, we propose the java agent injector to inject the agent sidecar into a pod.

When enabled in a pod's namespace, the injector injects the java agent container at pod creation time using a mutating webhook admission controller. By rendering the java agent to a shared volume, containers within the pod can use the java agent.
 
The following sections describe how to configure the agent, if you want to try it directly, please see [Usage](examples/java-agent-injector-usage) for more details.

## Install Injector

The java agent injector is a component of the operator, so you need to follow [Operator installation instrument](../README.md#operator) to install the operator firstly.

## Active the java agent injection

We have two granularities here: namespace and pod.

| Resource  | Label               | Enabled value | Disabled value |
| --------- | ------------------- | ------------- | -------------- |
| Namespace | swck-injection      | enabled       | disabled       |
| Pod       | swck-java-agent-injected | "true"        | "false"        |

The injector is configured with the following logic:

1. If either label is disabled, the pod is not injected.
2. If two labels are enabled, the pod is injected.

Follow the next steps to active java agent injection.

* Label the namespace with `swck-injection=enabled` 

```shell
$ kubectl label namespace default(your namespace) swck-injection=enabled
```

* Add label `swck-java-agent-injected: "true"` to the pod, and get the result as below.

```shell
$ kubectl get pod -l swck-java-agent-injected=true
NAME          READY   STATUS    RESTARTS   AGE
inject-demo   1/1     Running   0          2d2h
```

## The ways to configure the agent

The java agent injector supports a precedence order to configure the agent:

``` Annotations > Configmap > Default configmap```

### Annotations

Annotations are described in [kubernetes annotations doc](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/).

We support annotations in [agent annotations](#Use-annotations-to-overlay-default-agent-configuration) and [sidecar annotations](#configure-sidecar).

### Configmap

Configmap is described in [kubernetes configmap doc](https://kubernetes.io/docs/concepts/configuration/configmap/).

We need to use configmap to set [agent.config](https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/configurations/#table-of-agent-configuration-properties) so that we can modify the agent configuration without entering the container. 

If there are different configmap in the namepsace, you can choose a configmap by setting [sidecar annotations](#configure-sidecar); If there is no configmap, the injector will create a default configmap. 

### Default configmap

The injector will create the default configmap to overlay the `agent.config` in the agent container. 

The default configmap is shown as below, one is `agent.service_name` and the string can't be empty; the other is `collector.backend_service` and it needs to be a legal IP address and port, the other fields need to be guaranteed by users themselves. Users can change it as their default configmap.

```
data:
  agent.config: |
    # The service name in UI
    agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}

    # Backend service addresses.
    collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}

    # Please refer to https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/configurations/#table-of-agent-configuration-properties to get more details.
```

To avoid the default configmap deleting by mistake, we use a configmap controller to watch the default configmap. In addition, if the user applies an invalid configuration, such as a malformed `backend_service`, the controller will use the default configmap.

## Configure the agent

The injector supports two methods to configure agent:

* Only use the default configuration.
* Use annotations to overlay the default configuration.

### Use the default agent configuration

After activating the java agent injection, if not set the annotations, the injector will use the default agent configuration directly as below.

```
initContainers:
  - args:
    - -c
    - mkdir -p /sky/agent && cp -r /skywalking/agent/* /sky/agent
    command:
    - sh
    image: apache/skywalking-java-agent:8.8.0-java8
    name: inject-skywalking-agent
    volumeMounts:
    - mountPath: /sky/agent
      name: sky-agent
volumes:
  - emptyDir: {}
    name: sky-agent
  - configMap:
      name: skywalking-swck-java-agent-configmap
    name: java-agent-configmap-volume
```

### Use annotations to overlay default agent configuration

The injector can recognize five kinds of annotations to configure the agent as below.

#### 1. strategy configuration

The strategy configuration is the annotation as below.

| Annotation key                                    | Description                                                  | Annotation Default value |
| ------------------------------------------------- | ------------------------------------------------------------ | ------------------------ |
| `strategy.skywalking.apache.org/inject.Container` | Select the injected container, if not set, inject all containers. | not set                  |
| `strategy.skywalking.apache.org/agent.Overlay`    | Whether to overlay the agent configuration. If set true, then you can see the next details to configure the agent. If set false, then you can skip [Configure agent](#configure-agent). | `false`                  |

#### 2. agent configuration

The agent configuration is the annotation like `agent.skywalking.apache.org/{option}: {value}`, and the option support  `agent.xxx 、osinfo.xxx 、collector.xxx 、 logging.xxx 、statuscheck.xxx 、correlation.xxx 、jvm.xxx 、buffer.xxx 、 profile.xxx 、 meter.xxx 、 log.xxx` in [agent.config](https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/configurations/#table-of-agent-configuration-properties), such as `agent.skywalking.apache.org/agent.namespace`, `agent.skywalking.apache.org/meter.max_meter_size`, etc.

#### 3. plugins configuration

The plugins configuration is the annotation like `plugins.skywalking.apache.org/{option}: {value}`, and the option only support `plugin.xxx` in the [agent.config](https://skywalking.apache.org/docs/skywalking-java/latest/en/setup/service-agent/java-agent/configurations/#table-of-agent-configuration-properties), such as `plugins.skywalking.apache.org/plugin.mount`, ``plugins.skywalking.apache.org/plugin.mongodb.trace_param``, etc. 

#### 4. optional plugin configuration

The optional plugin configuration is the annotation as below.

| Annotation key                   | Description                                                  | Annotation value |
| -------------------------------- | ------------------------------------------------------------ | ---------------- |
| `optional.skywalking.apache.org` | Select the optional plugin which needs to be moved to the directory(/plugins). Users can select several optional plugins by separating from `｜`, such as `trace｜webflux｜cloud-gateway-2.1.x`. | not set          |

#### 5. optional reporter plugin configuration

The optional reporter plugin configuration is the annotation as below.

| Annotation key                            | Description                                                  | Annotation value |
| ----------------------------------------- | ------------------------------------------------------------ | ---------------- |
| `optional-reporter.skywalking.apache.org` | Select the optional reporter plugin which needs to be moved to the directory(/plugins). Users can select several optional reporter plugins by separating from `｜`, such as `kafka`. | not set          |

## Configure sidecar

The injector can recognize the following annotations to configure the sidecar:

| Annotation key                                               | Description                                                  | Annotation Default value                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `sidecar.skywalking.apache.org/initcontainer.Name`           | The name of the injected java agent container.               | `inject-skywalking-agent`                                    |
| `sidecar.skywalking.apache.org/initcontainer.Image`          | The container image of the injected java agent container.    | `apache/skywalking-java-agent:8.8.0-java8`                    |
| `sidecar.skywalking.apache.org/initcontainer.Command`        | The command of the injected java agent container.            | `sh`                                                         |
| `sidecar.skywalking.apache.org/initcontainer.args.Option`    | The args option of the injected java agent container.       | `-c`                                                         |
| `sidecar.skywalking.apache.org/initcontainer.args.Command`   | The args command of  the injected java agent container.      | `mkdir -p /sky/agent && cp -r /skywalking/agent/* /sky/agent` |
| `sidecar.skywalking.apache.org/sidecarVolume.Name`           | The name of sidecar Volume.                                  | `sky-agent`                                                  |
| `sidecar.skywalking.apache.org/sidecarVolumeMount.MountPath` | Mount path of the agent directory in the injected container. | `/sky/agent`                                                 |
| `sidecar.skywalking.apache.org/configmapVolume.Name`         | The name of configmap volume.                                | `java-agent-configmap-volume`                                |
| `sidecar.skywalking.apache.org/configmapVolumeMount.MountPath` | Mount path of the configmap in the injected container        | `/sky/agent/config`                                          |
| `sidecar.skywalking.apache.org/configmapVolume.ConfigMap.Name` | The name pf configmap used in the injected container as `agent.config ` | `skywalking-swck-java-agent-configmap`                       |
| `sidecar.skywalking.apache.org/env.Name`                     | Environment Name used by the injected container (application container). | `AGENT_OPTS`                                                 |
| `sidecar.skywalking.apache.org/env.Value`                    | Environment variables used by the injected container (application container). | `-javaagent:/sky/agent/skywalking-agent.jar`                 |

## The ways to get the final injected agent's configuration

Please see [javaagent introduction](javaagent.md) for details.