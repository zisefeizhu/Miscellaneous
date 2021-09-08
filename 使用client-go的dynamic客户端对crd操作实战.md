lientSet的使用在此篇博文已有详细案例：[紫色飞猪的研发之旅--02golang：client-go浅学demo](https://www.cnblogs.com/zisefeizhu/p/15207204.html) 对于dynamicClient的使用将在本篇案例.

本篇有改动的目录结构为：
```markdown
├── cmd
│   └── root.go
├── pkg
│   ├── dynamic-crd
│   │   ├── crd.yaml
│   │   └── crontab.yaml
└── service
      └── demo.go

3 directories, 4 files
```
### cmd
root.go
```markdown
// 初始化配置
func initConifg() {
	config.Loader(cfgFile) // cfgFile string
	service.Crd()
}
```
### dynamic-crd
crd.yaml
```markdown
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  # 对应 Group 字段的值
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    # 对应 Version 字段的可选值
    - name: v1
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    # 对应 Resource 字段的值
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```
crontab.yaml
```markdown
---
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: cron-1
  namespace: default
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image-1

---
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: cron-2
  namespace: default
spec:
  cronSpec: "* * * * */8"
  image: my-awesome-cron-image-2

---
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: cron-3
  namespace: default
spec:
  cronSpec: "* * * * */10"
  image: my-awesome-cron-image-3
```
### service
demo.go
```markdown
package service

/*
	注：在实际借助client-go 开发时最常用的时clientSet和dynamicClient客户端
        clientSet的使用在此篇博文已有详细案例：https://www.cnblogs.com/zisefeizhu/p/15207204.html
        对于dynamicClient的使用将在本篇案例
*/

import (
	"context"
	"fmt"
	"github.com/sirupsen/logrus"
	metaV1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/runtime/serializer/yaml"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/json"
	"k8s.io/client-go/dynamic"
	"operator/config"
	"strings"
)

// Crd client-go 对crd的有关操作
func Crd()  {
	dynamicClient, err := dynamic.NewForConfig(config.KubeConfig()); if err != nil {
		logrus.Println(err)
	}
	searchWorld := "list"
	// 删除空格
	search := strings.TrimSpace(searchWorld)
	switch search {
	case "list":
		list, err := listCrontabs(dynamicClient,"default"); if err != nil {
			logrus.Println(err)
		}
		for _, t := range list.Items {
			fmt.Printf("%s %s %s %s\n", t.Namespace, t.Name, t.Spec.CronSpec, t.Spec.Image)
		}
	case "get":
		ct, err := getCrontab(dynamicClient, "default","cron-1" ); if err != nil {
			logrus.Println(err)
		}
		fmt.Printf("%s %s %s %s\n", ct.Namespace, ct.Name, ct.Spec.CronSpec, ct.Spec.Image)
	case "create":
		createData := `
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: cron-5
  namespace: default
spec:
  cronSpec: "* * * * */20"
  image: my-awesome-cron-image-5`
		ct,err := createCrontabWithYaml(dynamicClient,"default",createData)
		if err != nil {
			logrus.Println(err.Error())
		}
		fmt.Printf("%s %s %s %s\n", ct.Namespace, ct.Name, ct.Spec.CronSpec, ct.Spec.Image)
	case "update":
		upData := `
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: cron-2
  namespace: default
spec:
  cronSpec: "* * * * */15"
  image: my-awesome-cron-image-2`
		ud, err := updateCrontabWithYaml(dynamicClient,"default",upData); if err != nil {
			logrus.Println(err.Error())
		}
		fmt.Printf("%s %s %s %s\n", ud.Namespace, ud.Name, ud.Spec.CronSpec, ud.Spec.Image)
	case "patch":
		patchData := []byte(`{"spec": {"image": "my-awesome-cron-image-2-patch"}}`)
		pt,err := pathCrontab(dynamicClient, "default", "cron-1", types.MergePatchType, patchData); if err != nil {
			logrus.Println(err.Error())
		}
		fmt.Printf("%s %s %s %s\n", pt.Namespace, pt.Name, pt.Spec.CronSpec, pt.Spec.Image)
	case "delete":
		if err := deleteCrontab(dynamicClient,"default","cron-1"); err != nil {
			logrus.Println(err.Error())
		}
	}
}

var gvr = schema.GroupVersionResource{
	Group: "stable.example.com",
	Version: "v1",
	Resource: "crontabs",
}

type CrontabSpec struct {
	CronSpec string `json:"cronSpec"`
	Image    string `json:"image"`
}

type Crontab struct {
	metaV1.TypeMeta   `json:",inline"`
	metaV1.ObjectMeta `json:"metadata,omitempty"`
	Spec CrontabSpec `json:"spec,omitempty"`
}

type CrontabList struct {
	metaV1.TypeMeta `json:",inline"`
	metaV1.ListMeta `json:"metadata,omitempty"`
	Items []Crontab `json:"items"`
}

// listCrontabs 资源列表
func listCrontabs(client dynamic.Interface, namespace string) (*CrontabList, error)  {
	list, err := client.Resource(gvr).Namespace(namespace).List(context.TODO(),metaV1.ListOptions{}); if err != nil {
		return nil,err
	}
	data, err := list.MarshalJSON(); if err != nil {
		return nil,err
	}
	var ctList CrontabList
	err = json.Unmarshal(data, &ctList); if err != nil {
		return nil, err
	}
	return &ctList, nil

}

// getCrontab 获取资源
func getCrontab(client dynamic.Interface, namespace string, name string) (*Crontab, error)  {
	result, err := client.Resource(gvr).Namespace(namespace).Get(context.TODO(),name,metaV1.GetOptions{}); if err != nil {
		return nil, err
	}
	data , err := result.MarshalJSON(); if err != nil {
		return nil, err
	}
	var ct Crontab
	err = json.Unmarshal(data, &ct);  if err != nil {
		return nil, err
	}
	return &ct, nil
}

// createCrontabWithYaml 创建资源
func createCrontabWithYaml(client dynamic.Interface, namespace string, yamlData string) (*Crontab, error) {
	decoder := yaml.NewDecodingSerializer(unstructured.UnstructuredJSONScheme)
	obj := &unstructured.Unstructured{}
	 _, gvk, err := decoder.Decode([]byte(yamlData), nil, obj); if err != nil {
	 	logrus.Println(err.Error())
	}
	// Get the common metadata, and show GVK
	fmt.Println(obj.GetName(), gvk.String())

	utd, err := client.Resource(gvr).Namespace(namespace).Create(context.TODO(),obj, metaV1.CreateOptions{})
	if err != nil {
		return nil, err
	}
	data, err := utd.MarshalJSON()
	if err != nil {
		return nil, err
	}
	var ct Crontab
	if err := json.Unmarshal(data, &ct); err != nil {
		return nil, err
	}
	return &ct, nil
}

// updateCrontabWithYaml 更新资源
func updateCrontabWithYaml(client dynamic.Interface, namespace string, yamlData string) (*Crontab, error)  {
	decoder := yaml.NewDecodingSerializer(unstructured.UnstructuredJSONScheme)
	obj := &unstructured.Unstructured{}
	if _, _, err := decoder.Decode([]byte(yamlData),nil,obj); err != nil {
		return nil,err
	}

	utd, err := client.Resource(gvr).Namespace(namespace).Get(context.TODO(),obj.GetName(), metaV1.GetOptions{}); if err != nil {
		return nil, err
	}
	obj.SetResourceVersion(utd.GetResourceVersion())
	utd, err = client.Resource(gvr).Namespace(namespace).Update(context.TODO(),obj,metaV1.UpdateOptions{}); if err != nil {
		return nil, err
	}

	data, err := utd.MarshalJSON(); if err != nil {
		return nil, err
	}
	var ct Crontab
	if err := json.Unmarshal(data, &ct); err != nil {
		return nil, err
	}
	return &ct, nil
}

// pathCrontab 布丁资源
func pathCrontab(client dynamic.Interface, namespace, name string, pt types.PatchType, data []byte) (*Crontab ,error) {
	resource, err := client.Resource(gvr).Namespace(namespace).Patch(context.TODO(),name,pt,data, metaV1.PatchOptions{}); if err != nil {
		return nil, err
	}
	data, err = resource.MarshalJSON();if err != nil {
		return nil, err
	}
	var ct Crontab
	if err := json.Unmarshal(data, &ct); err != nil {
		return nil, err
	}
	return &ct, nil
}

// deleteCrontab删除资源
func deleteCrontab(client dynamic.Interface, namespace , name string) error  {
	fmt.Printf("将要删除%s名称空间的%s资源",namespace,name)
	err := client.Resource(gvr).Namespace(namespace).Delete(context.TODO(),name,metaV1.DeleteOptions{})
	return err
}
```
### 执行
```markdown
1、部署crd
2、切换不同的``searchWorld``
```
