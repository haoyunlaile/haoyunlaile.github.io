---
title: Istio Sidecar注入原理
date: 2020-05-25
tags: ["istio","sidecar"]
categories:	istio
---

## 概念

简单来说，Sidecar 注入会**将额外容器**的配置**添加到 Pod 模板中**。这里特指将Envoy容器注应用所在Pod中。

Istio 服务网格目前所需的容器有：

`istio-init` 用于设置 iptables 规则，以便将入站/出站流量通过 Sidecar 代理。

初始化容器与应用程序容器在以下方面有所不同：

- 它在启动应用容器之前运行，并一直运行直至完成。
- 如果有多个初始化容器，则每个容器都应在启动下一个容器之前成功完成。

因此，您可以看到，对于不需要成为实际应用容器一部分的设置或初始化作业来说，这种容器是多么的完美。在这种情况下，`istio-init` 就是这样做并设置了 `iptables` 规则。

`istio-proxy` 这个容器是真正的 Sidecar 代理（基于 Envoy）。

<!-- more -->

下面的内容描述了向 pod 中注入 Istio Sidecar 的两种方法：

1. 使用 `istioctl`手动注入
2. 启用 pod 所属命名空间的 Istio Sidecar 注入器自动注入。

手动注入直接修改配置，如 deployment，并将代理配置注入其中。

当 pod 所属`namespace`启用自动注入后，自动注入器会使用准入控制器在创建 Pod 时自动注入代理配置。

通过应用 `istio-sidecar-injector` ConfigMap 中定义的模版进行注入。

### 自动注入

当你在一个`namespace`中设置了 `istio-injection=enabled` 标签，且 injection webhook 被启用后，任何新的 pod 都有将在创建时自动添加 Sidecar.  请注意，区别于手动注入，自动注入发生在 pod 层面。你将看不到 deployment 本身有任何更改 。

```shell
kubectl label namespace default istio-inhection=enabled
kubectl get namespace -L istio-injection
NAME           STATUS    AGE       ISTIO-INJECTION
default        Active    1h        enabled
istio-system   Active    1h
kube-public    Active    1h
kube-system    Active    1h
```

注入发生在 pod 创建时。杀死正在运行的 pod 并验证新创建的 pod 是否注入 sidecar。原来的 pod 具有 READY 为 1/1 的容器，注入 sidecar 后的 pod 则具有 READY 为 2/2 的容器 。

#### 自动注入原理

自动注入是利用了k8s  Admission webhook 实现的。 Admission webhook 是一种用于接收准入请求并对其进行处理的 HTTP 回调机制， 它可以更改发送到 API 服务器的对象以执行自定义的设置默认值操作。 具体细节可以查阅 [Admission webhook ](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/)文档。

istio 对应的istio-sidecar-injector webhook配置，默认会回调istio-sidecar-injector service的`/inject `地址。

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: istio-sidecar-injector
webhooks:
  - name: sidecar-injector.istio.io
    clientConfig:
      service:
        name: istio-sidecar-injector
        namespace: istio-system
        path: "/inject"
      caBundle: ${CA_BUNDLE}
    rules:
      - operations: [ "CREATE" ]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    namespaceSelector:
      matchLabels:
        istio-injection: enabled
```

回调API入口代码在 `pkg/kube/inject/webhook.go` 中

```go
// 创建一个用于自动注入sidecar的新实例
func NewWebhook(p WebhookParameters) (*Webhook, error) {
	
    // ...省略一万字...
		
	wh := &Webhook{
		Config:                 sidecarConfig,
		sidecarTemplateVersion: sidecarTemplateVersionHash(sidecarConfig.Template),
		meshConfig:             p.Env.Mesh(),
		configFile:             p.ConfigFile,
		valuesFile:             p.ValuesFile,
		valuesConfig:           valuesConfig,
		watcher:                watcher,
		healthCheckInterval:    p.HealthCheckInterval,
		healthCheckFile:        p.HealthCheckFile,
		env:                    p.Env,
		revision:               p.Revision,
	}
    
    //api server 回调函数,监听/inject回调
	p.Mux.HandleFunc("/inject", wh.serveInject)
	p.Mux.HandleFunc("/inject/", wh.serveInject)
    
    // ...省略一万字...

	return wh, nil
}
```

`serveInject`逻辑

```go
func (wh *Webhook) serveInject(w http.ResponseWriter, r *http.Request) {
	 
	// ...省略一万字...
    
	var reviewResponse *v1beta1.AdmissionResponse
	ar := v1beta1.AdmissionReview{}
	if _, _, err := deserializer.Decode(body, nil, &ar); err != nil {
		handleError(fmt.Sprintf("Could not decode body: %v", err))
		reviewResponse = toAdmissionResponse(err)
	} else {
         //执行具体的inject逻辑
		reviewResponse = wh.inject(&ar, path)
	}

    // 响应inject sidecar后的内容给k8s api server
	response := v1beta1.AdmissionReview{}
	if reviewResponse != nil {
		response.Response = reviewResponse
		if ar.Request != nil {
			response.Response.UID = ar.Request.UID
		}
	}

	// ...省略一万字...
}

// 注入逻辑实现
func (wh *Webhook) inject(ar *v1beta1.AdmissionReview, path string) *v1beta1.AdmissionResponse {
    
    // ...省略一万字...
    
    // injectRequired判断是否有设置自动注入
	if !injectRequired(ignoredNamespaces, wh.Config, &pod.Spec, &pod.ObjectMeta) {
		log.Infof("Skipping %s/%s due to policy check", pod.ObjectMeta.Namespace, podName)
		totalSkippedInjections.Increment()
		return &v1beta1.AdmissionResponse{
			Allowed: true,
		}
	}

	// ...省略一万字...
    
	// 返回需要注入Pod的对象
	spec, iStatus, err := InjectionData(wh.Config.Template, wh.valuesConfig, wh.sidecarTemplateVersion, typeMetadata, deployMeta, &pod.Spec, &pod.ObjectMeta, wh.meshConfig, path) // nolint: lll
	if err != nil {
		handleError(fmt.Sprintf("Injection data: err=%v spec=%v\n", err, iStatus))
		return toAdmissionResponse(err)
	}

    // 执行容器注入逻辑
	patchBytes, err := createPatch(&pod, injectionStatus(&pod), wh.revision, annotations, spec, deployMeta.Name, wh.meshConfig)
	if err != nil {
		handleError(fmt.Sprintf("AdmissionResponse: err=%v spec=%v\n", err, spec))
		return toAdmissionResponse(err)
	}

	reviewResponse := v1beta1.AdmissionResponse{
		Allowed: true,
		Patch:   patchBytes,
		PatchType: func() *v1beta1.PatchType {
			pt := v1beta1.PatchTypeJSONPatch
			return &pt
		}(),
	}

	return &reviewResponse
}
```

`injectRequired`函数

```go
func injectRequired(ignored []string, config *Config, podSpec *corev1.PodSpec, metadata *metav1.ObjectMeta) bool { 
    // HostNetwork模式直接跳过注入
	if podSpec.HostNetwork {
		return false
	}

    // k8s系统命名空间(kube-system/kube-public)跳过注入
	for _, namespace := range ignored {
		if metadata.Namespace == namespace {
			return false
		}
	}

	annos := metadata.GetAnnotations()
	if annos == nil {
		annos = map[string]string{}
	}

    
	var useDefault bool
	var inject bool
    // 优先判断是否申明了`sidecar.istio.io/inject` 注解，会覆盖命名配置
	switch strings.ToLower(annos[annotation.SidecarInject.Name]) {
	case "y", "yes", "true", "on":
		inject = true
	case "":
        // 使用命名空间配置
		useDefault = true
	}

	// 指定Pod不需要注入Sidecar的标签选择器
	if useDefault {
		for _, neverSelector := range config.NeverInjectSelector {
			selector, err := metav1.LabelSelectorAsSelector(&neverSelector)
			if err != nil {
			} else if !selector.Empty() && selector.Matches(labels.Set(metadata.Labels))
                // 设置不需要注入
				inject = false
				useDefault = false
				break
			}
		}
	}

	// 总是将 sidecar 注入匹配标签选择器的 pod 中，而忽略全局策略
	if useDefault {
		for _, alwaysSelector := range config.AlwaysInjectSelector {
			selector, err := metav1.LabelSelectorAsSelector(&alwaysSelector)
			if err != nil {
				log.Warnf("Invalid selector for AlwaysInjectSelector: %v (%v)", alwaysSelector, err)
			} else if !selector.Empty() && selector.Matches(labels.Set(metadata.Labels)){ 				  // 设置需要注入
				inject = true
				useDefault = false
				break
			}
		}
	}

	// 如果都没有配置则使用默认注入策略
	var required bool
	switch config.Policy {
	default: // InjectionPolicyOff
		log.Errorf("Illegal value for autoInject:%s, must be one of [%s,%s]. Auto injection disabled!",
			config.Policy, InjectionPolicyDisabled, InjectionPolicyEnabled)
		required = false
	case InjectionPolicyDisabled:
		if useDefault {
			required = false
		} else {
			required = inject
		}
	case InjectionPolicyEnabled:
		if useDefault {
			required = true
		} else {
			required = inject
		}
	}

	return required
}
```

从上面我们可以看出，是否注入Sidecar的优先级为

>  Pod Annotations → NeverInjectSelector → AlwaysInjectSelector → Default Policy 

`createPath`函数

```go
func createPatch(pod *corev1.Pod, prevStatus *SidecarInjectionStatus, revision string, annotations map[string]string,
	sic *SidecarInjectionSpec, workloadName string, mesh *meshconfig.MeshConfig) ([]byte, error) {

	var patch []rfc6902PatchOperation

	// ...省略一万字...

    // 注入初始化启动容器
	patch = append(patch, addContainer(pod.Spec.InitContainers, sic.InitContainers, "/spec/initContainers")...)
    // 注入Sidecar容器
	patch = append(patch, addContainer(pod.Spec.Containers, sic.Containers, "/spec/containers")...)
    // 注入挂载卷
	patch = append(patch, addVolume(pod.Spec.Volumes, sic.Volumes, "/spec/volumes")...)
	patch = append(patch, addImagePullSecrets(pod.Spec.ImagePullSecrets, sic.ImagePullSecrets, "/spec/imagePullSecrets")...)
    // 注入新注解
	patch = append(patch, updateAnnotation(pod.Annotations, annotations)...)

    // ...省略一万字...
	return json.Marshal(patch)
}
```

**总结：可以看到，整个注入过程实际就是原本的Pod配置反解析成Pod对象，把需要注入的Yaml内容(如:Sidecar)反序列成对象然后append到对应Pod (如：Container)上，然后再把修改后的Pod重新解析成yaml 内容返回给k8s的api server，然后k8s 拿着修改后内容再将这两个容器调度到同一台机器进行部署，至此就完成了对应Sidecar的注入。**

#### 卸载 sidecar 自动注入器

```shell
kubectl delete mutatingwebhookconfiguration istio-sidecar-injector
kubectl -n istio-system delete service istio-sidecar-injector
kubectl -n istio-system delete deployment istio-sidecar-injector
kubectl -n istio-system delete serviceaccount istio-sidecar-injector-service-account
kubectl delete clusterrole istio-sidecar-injector-istio-system
kubectl delete clusterrolebinding istio-sidecar-injector-admin-role-binding-istio-system
```

上面的命令不会从 pod 中移除注入的 sidecar。需要进行滚动更新或者直接删除对应的pod，并强制 deployment 重新创建新pod。 

### 手动注入 sidecar

手动注入 deployment ，需要使用 使用 `istioctl kube-inject`

使用手动注入前先关闭自动注入

```shell
kubectl label namespace default istio-injection=disabled
```

使用istioctl手动注入

```shell
istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -
```

我们可以查看对应的`deployment` 明细

```shell
describe deployment sleep
Name:                   sleep
Namespace:              default
CreationTimestamp:      Wed, 27 May 2020 10:45:23 +0800
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=sleep
Pod Template:
  Labels:           app=sleep
                    istio.io/rev=
                    security.istio.io/tlsMode=istio
  Annotations:      sidecar.istio.io/interceptionMode: REDIRECT
                    sidecar.istio.io/status:
                      {"version":"d36ff46d2def0caba37f639f09514b17c4e80078f749a46aae84439790d2b560","initContainers":["istio-init"],"containers":["istio-proxy"]...
                    traffic.sidecar.istio.io/excludeInboundPorts: 15020
                    traffic.sidecar.istio.io/includeOutboundIPRanges: *
  Service Account:  sleep
  Init Containers:
   istio-init:
    Image:      docker.io/istio/proxyv2:1.6.0
    Port:       <none>
    Host Port:  <none>
    Args:
      istio-iptables
      -p
      15001
      -z
      15006
      -u
      1337
      -m
      REDIRECT
      -i
      *
      -x
      
      -b
      *
      -d
      15090,15021,15020
  Containers:
   sleep:
    Image:      governmentpaas/curl-ssl
    Port:       <none>
    Host Port:  <none>
    Command:
      /bin/sleep
      3650d
    Environment:  <none>
    Mounts:
      /etc/sleep/tls from secret-volume (rw)
   istio-proxy:
    Image:      docker.io/istio/proxyv2:1.6.0
    Port:       15090/TCP
    Host Port:  0/TCP
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      --serviceCluster
      sleep.$(POD_NAMESPACE)
      --proxyLogLevel=warning
      --proxyComponentLogLevel=misc:error
      --trust-domain=cluster.local
      --concurrency
      2
```

可以看到，相比原始的deployment.yaml文件多出了两个容器，这两个容器的作用后面单独写一篇文章来分析：

1. `Init Containers`下的 `istio-init`
2. `Containers`下的`istio-proxy`

上面两个容器的注入，**默认情况下**将使用集群内的配置，或者使用该配置的本地副本来完成注入。 

```shell
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}' > inject-config.yaml
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.values}' > inject-values.yaml
kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}' > mesh-config.yaml
```

 指定输入文件，运行 `kube-inject` 并部署 

```shell
istioctl kube-inject \
    --injectConfigFile inject-config.yaml \
    --meshConfigFile mesh-config.yaml \
    --valuesFile inject-values.yaml \
    --filename samples/sleep/sleep.yaml \
    | kubectl apply -f -
```

 验证 sidecar 已经被注入到 READY 列下 `2/2` 的 sleep pod 中 

```shell
kubectl get pod  -l app=sleep
NAME                     READY   STATUS    RESTARTS   AGE
sleep-64c6f57bc8-f5n4x   2/2     Running   0          24s
```

查看对应pod中的容器

```
kubectl get pods  -o jsonpath="{.items[*].spec.containers[*].image}" | tr -s '[[:space:]]' '\n' | sort
docker.io/istio/proxyv2:1.6.0
governmentpaas/curl-ssl
```

手动注入的代码入口在 `istioctl/cmd/kubeinject.go`

手工注入跟自动注入还是有些差异的。手动注入是改变了`Deployment`。我们可以看下它具体做了哪些动作：

`Deployment`注入前配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      serviceAccountName: sleep
      containers:
      - name: sleep
        image: governmentpaas/curl-ssl
        command: ["/bin/sleep", "3650d"]
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: /etc/sleep/tls
          name: secret-volume
      volumes:
      - name: secret-volume
        secret:
          secretName: sleep-secret
          optional: true
```

`Deployment`注入后配置：

```shell
kubectl get deployment sleep -o yaml > sleep.yaml
less sleep.yaml
```

这里只保留跟注入容器有关的部分内容

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  generation: 1
  name: sleep
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      annotations:
        sidecar.istio.io/interceptionMode: REDIRECT
      labels:
        app: sleep
        istio.io/rev: ""
        security.istio.io/tlsMode: istio
    spec:
      containers:
      - command:
        - /bin/sleep
        - 3650d
        image: governmentpaas/curl-ssl
        imagePullPolicy: IfNotPresent
        name: sleep
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/sleep/tls
          name: secret-volume
      - args:
        - proxy
        - sidecar
        - --domain
        - $(POD_NAMESPACE).svc.cluster.local
        - --serviceCluster
        - sleep.$(POD_NAMESPACE)
        - --proxyLogLevel=warning
        - --proxyComponentLogLevel=misc:error
        - --trust-domain=cluster.local
        - --concurrency
        - "2"
        image: docker.io/istio/proxyv2:1.6.0
        imagePullPolicy: Always
        name: istio-proxy
        ports:
        - containerPort: 15090
          name: http-envoy-prom
          protocol: TCP
      dnsPolicy: ClusterFirst
      initContainers:
      - args:
        - istio-iptables
        - -p
        - "15001"
        - -z
        - "15006"
        - -u
        - "1337"
        - -m
        - REDIRECT
        - -i
        - '*'
        - -x
        - ""
        - -b
        - '*'
        - -d
        - 15090,15021,15020
        env:
        - name: DNS_AGENT
        image: docker.io/istio/proxyv2:1.6.0
        imagePullPolicy: Always
        name: istio-init
      restartPolicy: Always
```

可见新增了一个容器镜像

```yaml
 image: docker.io/istio/proxyv2:1.6.0
```

那么注入的内容模板从哪里获取，这里有两个选项。

1.  —injectConfigFile  指定对应的注入文件
2.  —injectConfigMapName  注入配置的 ConfigMap 名称 



如果在操作时发现Sidecar没有注入成功可以根据注入的方式查看上面的注入流程来查找问题。



## 参考文献

 https://preliminary.istio.io/zh/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection

 https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/ 

 https://istio.io/zh/docs/reference/commands/istioctl/#istioctl-kube-inject 