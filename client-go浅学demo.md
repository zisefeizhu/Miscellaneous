- client-go是kubernetes官方提供的go语言的客户端库，go应用使用该库可以访问kubernetes的API Server，这样我们就能通过编程来对kubernetes资源进行增删改查操作；

- 除了提供丰富的API用于操作kubernetes资源，client-go还为controller和operator提供了重要支持，如下图，client-go的informer机制可以将controller关注的资源变化及时带给此controller，使controller能够及时响应变化：

![](https://img-blog.csdnimg.cn/20210208214902260.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvbGluZ19jYXZhbHJ5,size_16,color_FFFFFF,t_70)

- 官方仓库：https://github.com/kubernetes/client-go
- 通过client-go提供的客户端对象与kubernetes的API Server进行交互，而client-go提供了以下四种客户端对象

1. RESTClient：这是最基础的客户端对象，仅对HTTPRequest进行了封装，实现RESTFul风格API，这个对象的使用并不方便，因为很多参数都要使用者来设置，于是client-go基于RESTClient又实现了三种新的客户端对象；

2. ClientSet：把Resource和Version也封装成方法了，用起来更简单直接，一个资源是一个客户端，多个资源就对应了多个客户端，所以ClientSet就是多个客户端的集合了，这样就好理解了，不过ClientSet只能访问内置资源，访问不了自定义资源；

3. DynamicClient：可以访问内置资源和自定义资源，个人感觉有点像java的集合操作，拿出的内容是Object类型，按实际情况自己去做强制转换，当然了也会有强转失败的风险；

4. DiscoveryClient：用于发现kubernetes的API Server支持的Group、Version、Resources等信息；

### 进入demo 实战

- 本次实战的目录结构

```markdown
.
├── cmd
│   └── root.go
├── config
│   ├── config.go
│   └── config.yaml
├── go.mod
├── go.sum
├── main.go                                           
├── pkg
│   └── tool.go
└── service
    └── demo.go

4 directories, 8 files
```
调用链：main.go --> root.go --> config.go --> namespace.go --> total.go

本次demo 重点演示``ClientSet`` 客户端对象

代码如下：
#### main.go
>> 主程序入口
```markdown
/*
* @Author: zisefeizhu
* @Description: student operator
* @File:  main.go
* @Version: 1.0.0
* @Date: 2021/8/301 13:37
*/

package main

import "operator/cmd"

func main()  {
	//入口
	cmd.Execute()
}
```
#### cmd
root.go
>> 程序初始化操作
```markdown
package cmd

import (
	"fmt"
	"operator/config"
	"operator/service"
	"os"

	"github.com/sirupsen/logrus"
	"github.com/spf13/cobra"
)

var (
	cfgFile string
	serverPort int
)


var rootCmd = &cobra.Command{
	Use:   "operator",
	Short: "Learning Project Operator",
	Long:  "Learning project Operator from zisefeizhu",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("启动参数: ", args)
		httpServer()
	},
}

func init() {
	logrus.Infoln("init root.go...")
	cobra.OnInitialize(initConifg)
	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $CURRENT_DIR/config/config.yaml)")
	rootCmd.Flags().IntVarP(&serverPort, "port", "p", 9002, "port on which the server will listen")
}

// 初始化配置
func initConifg() {
	config.Loader(cfgFile) // cfgFile string
	service.Namespace()
}

func httpServer() {
	logrus.Infoln("server start...")
	defer func() {
		logrus.Infoln("server exit..")
	}()
}

// Execute rootCmd
func Execute() {
	if err := rootCmd.Execute(); err != nil {
		logrus.Fatalln(err)
		os.Exit(1)
	}
}
```
#### config
config.yaml
>> 动态配置 对应k8s的configmap
```markdown
#  DeploymentMethod 部署方式 1为k8s集群外部 0为k8s集群内部
DeploymentMethod: 1
```
config.go
>> 目前主要是对k8s的配置文件处理
```markdown
package config

import (
	"flag"
	"fmt"
	"github.com/spf13/viper"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"path/filepath"

	"os"
	"strings"
)

// Loader 加载配置文件
func Loader(cfgFile string) {
	if cfgFile == "" {
		path, _ := os.Getwd()
		cfgFile = path + "/config/config.yaml"
		fmt.Println(cfgFile)
	}

	viper.SetConfigFile(cfgFile)              //用来指定配置文件的名称
	viper.SetEnvPrefix("ENV")                 //SetEnvPrefix会设置一个环境变量的前缀名
	viper.AutomaticEnv()                      //会获取所有的环境变量，同时如果设置过了前缀则会自动补全前缀名
	replacer := strings.NewReplacer(".", "_") //NewReplacer() 使用提供的多组old、new字符串对创建并返回一个*Replacer
	viper.SetEnvKeyReplacer(replacer)

	if err := viper.ReadInConfig(); err != nil {
		fmt.Printf("config file error: %s\n", err)
		os.Exit(1)
	}
}

var (
	// 1. 声明三个变量
	err error
	config *rest.Config
	kubeconfig *string
)

// homeDir 2.定义一个函数用来在操作系统中获取目录路径
func homeDir() string {
	if h := os.Getenv("HOME");h != ""{
		return h
	}
	return os.Getenv("USERPROFILE")  //windows
}

// DeployAndKuExternal 部署与k8s外部
func DeployAndKuExternal() *kubernetes.Clientset {
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

	// 5.在获取到使用Kubeonfig文件配置的Config对象之后，创建Clientset对象，并对其进行操作
	// 已经获得了rest.Config对象
	// 创建Clientset对象
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
	return clientset
}

// DeployAndKuInternal 部署与k8s内部
func DeployAndKuInternal() *kubernetes.Clientset {
	if config, err = rest.InClusterConfig(); err != nil {
		panic(err.Error())
	}
	// 创建Clientset对象
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
	return clientset
}

// KubeConfig k8s的config加载
func KubeConfig() *kubernetes.Clientset  {
	var clientset *kubernetes.Clientset
	switch choose := viper.GetInt("DeploymentMethod"); choose {
	case 1 :
		clientset = DeployAndKuExternal()
	case 0 :
		DeployAndKuInternal()
	}
	return clientset
}
```
####service
demo.go
>> 以名称空间、deployment、service 为例 学习增删改查
```markdown
package service


/*
参考文章：
	https://github.com/kubernetes/client-go
	https://blog.csdn.net/boling_cavalry/article/details/113487087?spm=1001.2014.3001.5501
 */


import (
	"context"
	"fmt"
	"github.com/sirupsen/logrus"
	appsV1 "k8s.io/api/apps/v1"
	coreV1 "k8s.io/api/core/v1"
	metaV1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"operator/config"
	"operator/pkg"
	"time"
)

func Namespace()  {
	clientSet := config.KubeConfig()
	// 1. namespace 列表
	fmt.Println("namespace list: ")
	namespaceClient := clientSet.CoreV1().Namespaces()
	namespaceResult, err := namespaceClient.List(context.TODO(), metaV1.ListOptions{}); if err != nil {
		logrus.Fatal(err)
	}
	now := time.Now()
	namespaces := []string{}
	fmt.Println("namespace: ")
	for _, namespace := range namespaceResult.Items {
		namespaces = append(namespaces, namespace.Name)
		fmt.Println(namespace.Name, now.Sub(namespace.CreationTimestamp.Time))
	}
	fmt.Println("namespaces\t", namespaces)

	// 2. namespace 创建
	fmt.Println("namespace create: ")
	namespace := &coreV1.Namespace{
		ObjectMeta: metaV1.ObjectMeta{
			Name: "test",
		},
	}
	namespace, err = namespaceClient.Create(context.TODO(), namespace, metaV1.CreateOptions{}); if err != nil {
		logrus.Println(err)
	} else {
		fmt.Println(namespace.Status)
	}

	// 2. deployment 列表
	fmt.Println("deployment list: ")
	for _, namespace := range namespaces {
		deploymentClient := clientSet.AppsV1().Deployments(namespace)
		deploymentResult, err := deploymentClient.List(context.TODO(), metaV1.ListOptions{}); if err != nil {
			logrus.Fatal(err)
		}else {
			for _, deployment := range deploymentResult.Items {
				fmt.Println(deployment.Name, deployment.Namespace,*deployment.Spec.Replicas)
			}
		}
	}

	// 3. deployment 创建
	fmt.Println("deployments create: ")
	deploymentClient := clientSet.AppsV1().Deployments("test")
	deployment := &appsV1.Deployment{
		ObjectMeta: metaV1.ObjectMeta{
			Name: "test-dev-nginx",
			Labels: map[string]string{
				"app":       "nginx",
				"env":       "test",
				"by":        "zisefeizhu",
				"version":   "v0.1.0",
			},
		},
		Spec: appsV1.DeploymentSpec{
			Replicas: pkg.Int32Ptr(3),
			Selector: &metaV1.LabelSelector{
				MatchLabels: map[string]string{
					"app":       "nginx",
					"env":       "test",
					"by":        "zisefeizhu",
					"version":   "v0.1.0",
				},
			},
			Template: coreV1.PodTemplateSpec{
				ObjectMeta: metaV1.ObjectMeta{
					Labels: map[string]string{
						"app":       "nginx",
						"env":       "test",
						"by":        "zisefeizhu",
						"version":   "v0.1.0",
					},
				},
				Spec: coreV1.PodSpec{
					Containers: []coreV1.Container{
						{
							Name:  "nginx",
							Image: "nginx:latest",
							Ports: []coreV1.ContainerPort{
								{
									Name:          "http",
									ContainerPort: 80,
									Protocol:      coreV1.ProtocolTCP,
								},
							},
						},
					},
				},
			},
		},
	}
	fmt.Println("create deployment: ")
	deployment, err = deploymentClient.Create(context.TODO(), deployment, metaV1.CreateOptions{}); if err != nil {
		logrus.Println(err)
	} else {
		fmt.Println(deployment.Status.Conditions)
	}

	// 4。 deployment 修改
	fmt.Println("deployment update: ")
	deployment, err = deploymentClient.Get(context.TODO(), "test-dev-nginx", metaV1.GetOptions{})
	if *deployment.Spec.Replicas > 3 {
		deployment.Spec.Replicas = pkg.Int32Ptr(1)
	} else {
		deployment.Spec.Replicas = pkg.Int32Ptr(*deployment.Spec.Replicas + 1)
	}
	// 1 => nginx:1.19.1
	// 2 => nginx:1.19.2
	// 3 => nginx:1.19.3
	// 3 => nginx:1.19.4
	deployment.Spec.Template.Spec.Containers[0].Image = fmt.Sprintf("nginx:1.19.%d", *deployment.Spec.Replicas); if err != nil {
		logrus.Println(err)
	}
	deployment, err = deploymentClient.Update(context.TODO(), deployment, metaV1.UpdateOptions{}); if err != nil {
		logrus.Println(err)
	}else {
		fmt.Println(deployment.Status)
	}

	// 5. service 列表
	fmt.Println("services list: ")
	for _, namespace := range namespaces {
		serviceClient := clientSet.CoreV1().Services(namespace)
		serviceResult, err := serviceClient.List(context.TODO(), metaV1.ListOptions{}); if err != nil {
			logrus.Println(err)
		}else {
			for _, service := range serviceResult.Items {
				fmt.Println(service.Name, service.Namespace, service.Labels, service.Spec.Selector, service.Spec.Type, service.Spec.ClusterIP, service.Spec.Ports, service.CreationTimestamp)
			}
		}
	}

	// 6. service 创建
	fmt.Println("services create: ")
	serviceClient := clientSet.CoreV1().Services("test")
	service := &coreV1.Service{
		ObjectMeta: metaV1.ObjectMeta{
			Name: "test-dev-nginx",
			Labels: map[string]string{
				"app":       "nginx",
				"env":       "test",
				"by":        "zisefeizhu",
				"version":   "v0.1.0",
			},
		},
		Spec: coreV1.ServiceSpec{
			Selector: map[string]string{
				"app":       "nginx",
				"env":       "test",
				"by":        "zisefeizhu",
				"version":   "v0.1.0",
			},
			Type: coreV1.ServiceTypeNodePort,
			Ports: []coreV1.ServicePort{
				{
					Name: "http",
					Port: 80,
					Protocol: coreV1.ProtocolTCP,
				},
			},
		},
	}
	service, err = serviceClient.Create(context.TODO(), service, metaV1.CreateOptions{})
	if err != nil {
		logrus.Println(err)
	} else {
		fmt.Println(service.Status)
	}

	// 7. service 修改
	fmt.Println("services update: ")
	service, err = serviceClient.Get(context.TODO(), "test-dev-nginx", metaV1.GetOptions{}); if err != nil {
		logrus.Println(err)
	}
	if service.Spec.Type == coreV1.ServiceTypeNodePort {
		service.Spec.Ports[0].NodePort = 30900
	}
	service, err = serviceClient.Update(context.TODO(), service, metaV1.UpdateOptions{}); if err != nil {
		logrus.Println(err)
	}else {
		fmt.Println(service.Spec.ClusterIP)
	}

	// 8. deployment 删除
	fmt.Println("deployment delete: ")
	err = deploymentClient.Delete(context.TODO(), "test-dev-nginx", metaV1.DeleteOptions{}); if err != nil {
		logrus.Println(err)
	}

	// 补充逻辑 判断deployment删除完毕 --> 再删除service

	// 9. service 删除
	fmt.Println("service delete: ")
	err = serviceClient.Delete(context.TODO(),"test-dev-nginx", metaV1.DeleteOptions{}); if err != nil {
		logrus.Println(err)
	}

	// 补充逻辑 判断所有资源均被删除完毕后 --> 再删除namespace

	// 10. namespace 删除
	fmt.Println("namespace delete: ")
	err = namespaceClient.Delete(context.TODO(),"test", metaV1.DeleteOptions{}); if err != nil {
		logrus.Println(err)
	}
}
```
#### pkg
tool.go
>> 工具包
```markdown
package pkg

func Int32Ptr(n int32) *int32 {
	i32 := int32(n)
	p32 := &i32
	return p32
}
```
### 演示
请自行测试

本demo 环境
- k8s 1.18.3
- client.go k8s.io/client-go@v0.18.3