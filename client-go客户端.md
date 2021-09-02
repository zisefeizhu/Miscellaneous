> 此部分是对紫色飞猪的研发之旅--02golang：client-go浅学demo[https://www.cnblogs.com/zisefeizhu/p/15207204.html]的补充

### 对02的改动点如下：

#### cmd/root.go
```go
// 初始化配置
func initConifg() {
	config.Loader(cfgFile) // cfgFile string
    //dservice.Demo()
	//service.RESTClient()
	//service.ClientSet()
	//service.DynamicClient()
	service.DiscoveryClient()
}
```
#### config/config.go
```go
// DeployAndKuExternal 部署与k8s外部
func DeployAndKuExternal() *rest.Config {
	// 3. 在k8s的环境中kubectl配置文件一般放在用户目录的.kube文件中
	if home := homeDir(); home != ""{
		kubeconfig = flag.String("kubeconfig",filepath.Join(home,".kube","config"),"(可选)kubeconfig 文件的绝对路径")
		fmt.Println("kubeConfig", *kubeconfig)
	}else {
		kubeconfig = flag.String("kubeconfig","","kubeconfig 文件的绝对路径")
		fmt.Println(kubeconfig)
		fmt.Println("##################")
	}
	flag.Parse()
	// 4.创建集群配置,首先使用 inCluster 模式(需要区配置对应的RBAC 权限,默认的sa是default-->是没有获取deployment的List权限)
	if config, err = rest.InClusterConfig(); err != nil {
		// 使用Kubeconfig文件配置集群Config对象
		if config,err = clientcmd.BuildConfigFromFlags("",*kubeconfig); err != nil {
			panic(err.Error())
		}
	}

	return config
}

// DeployAndKuInternal 部署与k8s内部
func DeployAndKuInternal() *rest.Config {
	// 使用当前上下文环境
	kubeconfig := filepath.Join(
		os.Getenv("KUBECONFIG"),
		)
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		logrus.Fatal(err)
	}
	return config
}

// KubeConfig k8s的config加载
func KubeConfig() *rest.Config  {
	switch choose := viper.GetInt("DeploymentMethod"); choose {
	case 1 :
		config = DeployAndKuExternal()
	case 0 :
		config = DeployAndKuInternal()
	}
	return config
}
```
#### service/demo.go
```go
package service

/*
	参考文档：
	Kubernetes的Group、Version、Resource学习小记： https://xinchen.blog.csdn.net/article/details/113715847
	Client-go 实战：https://xinchen.blog.csdn.net/article/details/113753087

	注：上面的链接博主写的已经十分详细，本着好记性不如烂笔头的目的 跟着敲了一边
*/

import (
	"context"
	"flag"
	"fmt"
	appsV1 "k8s.io/api/apps/v1"
	coreV1 "k8s.io/api/core/v1"
	metaV1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/kubernetes/scheme"

     rschema "k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"operator/config"
	"operator/pkg"
)

func RESTClient()  {

	//RestClient demo : 查询kube-system这个namespace下的所有pod，然后在控制台打印每个pod的几个关键字段；

	config := config.KubeConfig()
	// 参考path : /api/v1/namespaces/{namespace}/pods
	config.APIPath = "api"
	// pod的group是空字符串
	config.GroupVersion = &coreV1.SchemeGroupVersion
	// 指定序列化工具
	config.NegotiatedSerializer = scheme.Codecs

	// 根据配置信息构建restClient实例
	restClient, err := rest.RESTClientFor(config)

	if err!=nil {
		panic(err.Error())
	}

	// 保存pod结果的数据结构实例
	result := &coreV1.PodList{}

	//  指定namespace
	namespace := "kube-system"
	// 设置请求参数，然后发起请求
	// GET请求
	err = restClient.Get().
		//  指定namespace，参考path : /api/v1/namespaces/{namespace}/pods
		Namespace(namespace).
		// 查找多个pod，参考path : /api/v1/namespaces/{namespace}/pods
		Resource("pods").
		// 指定大小限制和序列化工具
		VersionedParams(&metaV1.ListOptions{Limit:100}, scheme.ParameterCodec).
		// 请求
		Do(context.TODO()).
		// 结果存入result
		Into(result)

	if err != nil {
		panic(err.Error())
	}

	// 表头
	fmt.Printf("namespace\t status\t\t name\n")

	// 每个pod都打印namespace、status.Phase、name三个字段
	for _, d := range result.Items {
		fmt.Printf("%v\t %v\t %v\n",
			d.Namespace,
			d.Status.Phase,
			d.Name)
	}
}

const (
	NAMESPACE = "test-clientset"
	DEPLOYMENT_NAME = "client-test-deployment"
	SERVICE_NAME = "client-test-service"
)

func ClientSet()  {
	/*
		本次编码实战的需求如下：
		写一段代码，检查用户输入的operate参数，该参数默认是create，也可以接受clean；
		如果operate参数等于create，就执行以下操作：
		新建名为test-clientset的namespace
		新建一个deployment，namespace为test-clientset，镜像用tomcat，副本数为2
		新建一个service，namespace为test-clientset，类型是NodePort
		如果operate参数等于clean，就删除create操作中创建的service、deployment、namespace等资源：
		以上需求使用Clientset客户端实现，完成后咱们用浏览器访问来验证tomcat是否正常；
	*/

	// 获取用户输入的操作类型，默认是create，还可以输入clean，用于清理所有资源
	operate := flag.String("operate", "clean", "operate type : create or clean")
	// 把用户传递的命令行参数解析为对应变量的值
	flag.Parse()

	fmt.Printf("operation is %v\n", *operate)

	// 实例化clientset对象
	clientset, err := kubernetes.NewForConfig(config.KubeConfig()); if err != nil {
		panic(err.Error())
	}

	// 如果要执行清理操作
	if "clean" == *operate {
		clean(clientset)
	} else {
		// 创建namespace
		createNamespace(clientset)

		// 创建deployment
		createDeployment(clientset)

		// 创建service
		createService(clientset)
	}
}

// 清理本次实战创建的所有资源
func clean(clientset *kubernetes.Clientset)  {
	emptyDeleteOptions := metaV1.DeleteOptions{}

	// 删除service
	if err := clientset.CoreV1().Services(NAMESPACE).Delete(context.TODO(), SERVICE_NAME,emptyDeleteOptions); err != nil {
		panic(err.Error())
	}

	// 删除deployment
	if err := clientset.AppsV1().Deployments(NAMESPACE).Delete(context.TODO(), DEPLOYMENT_NAME, emptyDeleteOptions); err != nil {
		panic(err.Error())
	}

	// 删除namespace
	if err := clientset.CoreV1().Namespaces().Delete(context.TODO(),NAMESPACE, emptyDeleteOptions); err != nil {
		panic(err.Error())
	}
}

// 新建namespace
func createNamespace(clientset *kubernetes.Clientset)  {
	namespaceClient := clientset.CoreV1().Namespaces()
	namespace := &coreV1.Namespace{
		ObjectMeta: metaV1.ObjectMeta{
			Name: NAMESPACE,
		},
	}

	result, err := namespaceClient.Create(context.TODO(), namespace, metaV1.CreateOptions{}); if err != nil{
		panic(err.Error())
	}

	fmt.Printf("Create namespace %s \n", result.GetName())
}

// 新建service
func createService(clientset *kubernetes.Clientset)  {
	// 得到service的客户端
	serviceClient := clientset.CoreV1().Services(NAMESPACE)

	// 实例化一个数据结构
	service := &coreV1.Service{
		ObjectMeta: metaV1.ObjectMeta{
			Name: SERVICE_NAME,
		},
		Spec: coreV1.ServiceSpec{
			Ports: []coreV1.ServicePort{
				{
					Name: "http",
					Port: 8080,
					NodePort: 30080,
				},
			},
			Selector: map[string]string{
				"app" : "tomcat",
			},
			Type: coreV1.ServiceTypeNodePort,
		},
	}
	result, err := serviceClient.Create(context.TODO(), service, metaV1.CreateOptions{}); if err != nil {
		panic(err.Error())
	}
	fmt.Printf("Create service %s \\n", result.GetName())
}

// 新建deployment
func createDeployment(clientset *kubernetes.Clientset)  {
	// 得到deployment的客户端
	deploymentClient := clientset.AppsV1().Deployments(NAMESPACE)

	// 实例化一个数据结构
	deployment := &appsV1.Deployment{
		ObjectMeta: metaV1.ObjectMeta{
			Name: DEPLOYMENT_NAME,
		},
		Spec: appsV1.DeploymentSpec{
			Replicas: pkg.Int32Ptr(2),
			Selector: &metaV1.LabelSelector{
				MatchLabels: map[string]string{
					"app": "tomcat",
				},
			},
			Template: coreV1.PodTemplateSpec{
				ObjectMeta: metaV1.ObjectMeta{
					Labels: map[string]string{
						"app": "tomcat",
					},
				},
				Spec: coreV1.PodSpec{
					Containers: []coreV1.Container{
						{
							Name:            "tomcat",
							Image:           "tomcat:latest",
							ImagePullPolicy: "IfNotPresent",
							Ports: []coreV1.ContainerPort{
								{
									Name:          "http",
									Protocol:      coreV1.ProtocolTCP,
									ContainerPort: 8080,
								},
							},
						},
					},
				},
			},
		},
	}
	result, err := deploymentClient.Create(context.TODO(), deployment, metaV1.CreateOptions{}); if err != nil {
		panic(err.Error())
	}
	fmt.Printf("Create deployment %s \n", result.GetName())
}

func DynamicClient()  {
	// 查询指定namespace下的所有pod，然后在控制台打印出来，要求用dynamicClient实现
	dynamicClient, err := dynamic.NewForConfig(config.KubeConfig()); if err != nil {
		panic(err.Error())
	}

	// 从dynamicClient 的唯一关联方法所需的入参
	gvr := rschema.GroupVersionResource{Version: "v1", Resource: "pods"}
	// 使用dynamicClient的查询列表方法，查询指定namespace下的所有pod，
	// 注意此方法返回的数据结构类型是UnstructuredList
	unstructObj, err := dynamicClient.
		Resource(gvr).
		Namespace("kube-system").
		List(context.TODO(), metaV1.ListOptions{Limit: 100}); if err != nil {
			panic(err.Error())
	}

	// 实例化一个PodList数据结构，用于接收从unstructObj转换后的结果
	podList := &coreV1.PodList{}

	// 转换
	err = runtime.DefaultUnstructuredConverter.FromUnstructured(unstructObj.UnstructuredContent(), podList); if err != nil {
		panic(err.Error())
	}

	// 表头
	fmt.Printf("namespace\t status\t\t name\n")

	// 每个pod都打印namespace、status.Phase、name三个字段
	for _, d := range podList.Items {
		fmt.Printf("%v\t %v\t %v\n",
			d.Namespace,
			d.Status.Phase,
			d.Name)
	}
}

func DiscoveryClient()  {
	// 从kubernetes查询所有的Group、Version、Resource信息，在控制台打印出来
	discoveryClient, err := discovery.NewDiscoveryClientForConfig(config.KubeConfig()); if err != nil{
		panic(err.Error())
	}

	// 获取所有分组和资源数据
	APIGroup, APIResourceListSlice, err := discoveryClient.ServerGroupsAndResources(); if err != nil {
		panic(err.Error())
	}

	// 先看Group信息
	fmt.Printf("APIGroup :\n\n %v\n\n\n\n",APIGroup)

	// APIResourceListSlice是个切片，里面的每个元素代表一个GroupVersion及其资源
	for _, singleAPIResourceList := range APIResourceListSlice {

		// GroupVersion是个字符串，例如"apps/v1"
		groupVerionStr := singleAPIResourceList.GroupVersion

		// ParseGroupVersion方法将字符串转成数据结构
		gv, err := rschema.ParseGroupVersion(groupVerionStr)

		if err != nil {
			panic(err.Error())
		}

		fmt.Println("*****************************************************************")
		fmt.Printf("GV string [%v]\nGV struct [%#v]\nresources :\n\n", groupVerionStr, gv)

		// APIResources字段是个切片，里面是当前GroupVersion下的所有资源
		for _, singleAPIResource := range singleAPIResourceList.APIResources {
			fmt.Printf("%v\n", singleAPIResource.Name)
		}
	}
}

```