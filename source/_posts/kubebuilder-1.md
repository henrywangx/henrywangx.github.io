---
title: kubebuilder(1)-安装和使用
categories:
  - Cloud
tags:
  - Kubernetes
  - CRD
  - kubebuilder
comments: true
date: 2019-12-29 00:28:30
---

> **导读：** 作为云原生的从业者，多多少少也会听说CRD(Custom Resource Definition)和operator，但是说实话开发起来挺繁琐的，而且对初学者也不友好。还好社区目前有好使的脚手架：[kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)和[operator-sdk](https://github.com/operator-framework/operator-sdk)，目前个人感觉大家会往前者站队，毕竟k8s自家的，索性就学着用它来尝试开发operator了。考虑到篇幅和观感问题，这里就只介绍最基本的使用，像**finalizer**和**webhook**建议直接查官方文档，code逻辑另外写一篇心得。

## 1.安装
安装参考[官方指南](https://book.kubebuilder.io/quick-start.html)， 由于github下载太慢我就用码云fork了源文件来安装的：

```bash
git clone https://gitee.com/henrywangx/kubebuilder.git
cd kubebuilder
make build
cp bin/kubebuilder $GOPATH/bin
```
## 2.使用
kubebuilder依赖go module所以要打开go module环境变量:`export GO111MODULE=on`, 另外proxy或者墙的原因，先设一下go mod的proxy：`export GOPROXY=https://goproxy.io`, 然后就可以开始使用了。总结就是要：

```bash
export GO111MODULE=on
export GOPROXY=https://goproxy.io
```

### 2.1创建project
```bash
mkdir $GOPATH/src/demo
cd $GOPATH/src/demo
kubebuilder init --domain demo.com --license apache2 --owner "xiong"
```
然后呢check一下当前文件夹：

```bash
tree .
.
├── Dockerfile
├── Makefile
├── PROJECT
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   └── role_binding.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go
```
这里kubebuilder帮我们生成了一下模板文件夹，包括解决crd的rbac, cert, webhook的文件。暂时不用管他们，这时需要保证你的终端能访问k8s的测试集群，简单就是用`kubectl cluster-info`看看是否出错，如果不出错，就可以run起来main.go了

```bash
kubectl cluster-info
go run main.go
```
可以看到终端输出的log：

```bash
2019-12-28T00:22:09.789+0800	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": ":8080"}
2019-12-28T00:22:09.789+0800	INFO	setup	starting manager
2019-12-28T00:22:09.790+0800	INFO	controller-runtime.manager	starting metrics server	{"path": "/metrics"}
```
从`main.go`里面可以看出其实kubebuilder帮我们生成一个管理controller的manager的代码，但是还没添加controller（controller是指管理crd的控制器）：

```go
func main() {
...
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme:             scheme,
		MetricsBindAddress: metricsAddr,
		LeaderElection:     enableLeaderElection,
		Port:               9443,
	})
...
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
...
	}
}
```

ok, 接下来我们就可以用kubebuilder帮我们创建一个我们想要的crd，我就叫这个crd为**Object**吧：

```bash
kubebuilder create api --group infra --version v1 --kind Object
```
这里简单注意一下， `group`， `version`, `kind`这三个属性组合起来来标识一个k8s的crd。另外就是`kind`要首字母大写而且不能有特殊符号。

执行上面的命令之后，kubebuilder就帮我们创建了两个文件`api/v1/object_types.go`和`controllers/object_controller.go`, 前者是这个crd需要定义哪些属性，而后者是对crd的reconsile的处理逻辑（也就是增删改crd的逻辑）, 我们后面再讲这两个文件。最后呢，在main.go里面，我们定义的**Object**对应的controller会注册到之前生成的manager里：

```go
function main(){
...
// 注册Object的controller到manager里
	if err = (&controllers.ObjectReconciler{
		Client: mgr.GetClient(),
		Log:    ctrl.Log.WithName("controllers").WithName("Object"),
		Scheme: mgr.GetScheme(),
	}).SetupWithManager(mgr); err != nil {
...
	}
...
```

聪明的你一定能猜到，我们反复执行`kubebuilder create api xxx`这条命令就会帮我们创建和注册不同的controller到manager里面。

回过头我们再看一下`api/v1/object_types.go`，这里我在spec里面加一个`Detail`，在status里面加一个`Created`：

```go
// ObjectSpec defines the desired state of Object
type ObjectSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of Object. Edit Object_types.go to remove/update
	Foo string `json:"foo,omitempty"`
	Detail string `json:"detail,omitempty"`
}

// ObjectStatus defines the observed state of Object
type ObjectStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	Created bool `json:"created,omitempy"`
}
```
实际上kubebuilder就是帮我们生成Object的spec和status的模板，从注释就也可以看出来spec是我们期望的crd状态，而status就是观测到的状态，具体也可以参见k8s对一个[对象的定义](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)。可以看到默认定义下面，kubebuilder会为我们生成对应的yaml文件在`config/samples/infra_v1_object.yaml`:

```yaml
---
apiVersion: infra.demo.com/v1
kind: Object
metadata:
  name: object-sample
spec:
  # Add fields here
  foo: bar
  detail: "detail for demo"
```
在`controllers/object_controller.go`，也就是kubebuilder帮我们生成的Reconcile代码里面，我添加了打印`Detail`的信息，并且把`Created`改成true:

```go
//此method在controllers/object_controller.go
func (r *ObjectReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	ctx := context.Background()
	_ = r.Log.WithValues("object", req.NamespacedName)
	// your logic here

	// 1. Print Spec.Detail and Status.Created in log
	obj := &infrav1.Object{}
	if err := r.Get(ctx, req.NamespacedName, obj); err != nil {
		fmt.Errorf("couldn't find object:%s", req.String())
	} else {
	//打印Detail和Created
		r.Log.V(1).Info("Successfully get detail", "Detail", obj.Spec.Detail)
		r.Log.V(1).Info("", "Created", obj.Status.Created)
	}
	// 2. Change Created
	if !obj.Status.Created {
		obj.Status.Created = true
		r.Update(ctx, obj)
	}

	return ctrl.Result{}, nil
}
```


由于kubebuilder要用到[kustomize](https://github.com/kubernetes-sigs/kustomize)，所以先确保先装好了, mac的安装方式:

```bash
brew install kustomize
```

然后就install生成的crd并且运行修改代码后的manager:

```bash
make install
go run main.go
```

现在我们就可以用`config/samples/infra_v1_object.yaml`创建一个Object:

```bash
kubectl create -f config/samples/infra_v1_object.yaml
```

在运行manager的终端里面可以看到我们刚才添加的代码打印出来的log:

```bash
2019-12-28T23:39:34.963+0800	DEBUG	controllers.Object	Successfully get detail	{"Detail": "detail for demo"}
2019-12-28T23:39:34.963+0800	DEBUG	controllers.Object	{"Created": false}
2019-12-28T23:39:35.019+0800	DEBUG	controller-runtime.controller	Successfully Reconciled	{"controller": "object", "request": "default/object-sample"}
2019-12-28T23:39:35.019+0800	DEBUG	controllers.Object	Successfully get detail	{"Detail": "detail for demo"}
2019-12-28T23:39:35.019+0800	DEBUG	controllers.Object	{"Created": true}
```

再看看从k8s里面看到的这个Object的状态:

```bash
kubectl get object object-sample -o yaml
apiVersion: infra.demo.com/v1
kind: Object
metadata:
  creationTimestamp: "2019-12-28T15:39:34Z"
  generation: 2
  name: object-sample
  namespace: default
  resourceVersion: "433782"
  selfLink: /apis/infra.demo.com/v1/namespaces/default/objects/object-sample
  uid: 3f4b368d-2988-11ea-a544-080027d6a71e
spec:
  detail: detail for demo
  foo: bar
status:
  Created: true   #此处是我们修改成true的状态
```
可以看到这个`object-sample`的`status.Created`已经被修改为true了

## 3.其他
关于没有介绍的crt，webhook和finalizer，官网手册有比较详细的用法介绍。我就简单谈一下我的理解，如果有错误请纠正：

1. **webhook**，其实就是当我们添加和修改一个**Object**时，我们需要对Object的合法性进行判断，所以可以通过webhook的framework来进行合法性的判定，所以kubebuilder可以生成对应的webhook代码；
2. **crt**，用于解决webhook访问k8s时所需要的证书问题，官网也建议使用[crt-manager](https://github.com/jetstack/cert-manager)解决证书问题；
3. **finalizer**，就是在删除Object时，由于这个**Object**可能创建一些其他的resource比如pod之类的，又或者在删除之前，我们需要做一些清理工作，**finalizer**就是实现这个清理的framework代码；

另外kubebuilder也是支持把这个manager部署为deployment，但是调试起来比较麻烦，所以就只用`go run`的形式演示了。

## 小结
这篇blog简单的介绍了一下kubebuilder开发crd的基本过程，没有深入过多的代码原理，可能也有不少错误地方，麻烦帮忙纠正。另外大家可能跟我刚学习kubebuilder的时候一样，只能照着官网教程敲命令，kubebuilder生成的代码就像一个黑盒一样，接下来目标就是专门整理一下kubebuilder生成crd代码流程和结构。