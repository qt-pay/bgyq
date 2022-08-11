## Golang Code Demo

### 第一个Golang代码:平滑迁移pod

目标：平滑迁移指定node上pod，并将该node踢出集群

思路：

1. 将node设置为cordon
2. 修改该node上所有pod的关键label，使其和rs/deployment无法匹配，利用k8s的reconcile在其他node自动创建出来新的pod
3. 删除cordon node上的多余pod
4. 删除节点

差个 pod list 的判断，将同一个deployment的pod，挪到第一个slice中，这样避免一个应用的pod全在一个node上时，出现业务中断。

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"encoding/json"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"os"
	"path/filepath"
	"strings"
	"log"
	"sync"
	"time"
)



const (
	maxPods = 110
	maxRetry = 60
	interval = 5
)

var (
	nodeName = ""
	wg sync.WaitGroup
	podNamespace = ""
)

type PodBaseInfo struct {
	podName string
	deploymentName string
}

type PatchStringValue struct {
	Op    string      `json:"op"`
	Path  string      `json:"path"`
	Value string `json:"value"`
}



func getPodsWithNodeName(nodeName string, client *kubernetes.Clientset) (pods []PodBaseInfo) {
	pods = make([]PodBaseInfo, 0, maxPods)
	opts := metav1.ListOptions{
		FieldSelector: fmt.Sprintf("spec.nodeName=%s", nodeName),
	}
	podsInfo, err := client.CoreV1().Pods(podNamespace).List(context.TODO(), opts)
	if err != nil {
		panic(err.Error())
	}
	log.Printf("There are %d pods in the node.\n", len(podsInfo.Items))
	for _, pod := range podsInfo.Items {
		if len(pod.OwnerReferences) != 0{
			li := strings.Split(pod.OwnerReferences[0].Name, "-")
			deploymentName := strings.Join(
				li[:len(li)-1],
				"-",
			)
			pods = append(
				pods,
				PodBaseInfo{
					podName: pod.Name,
					deploymentName: deploymentName,
				},
			)
		}
	}
	return
}


func drainPod(podInfo PodBaseInfo, client *kubernetes.Clientset) {
	defer func() {
		if err := recover() ; err != nil{
			log.Printf("drain %s pod error: %v\n", podInfo.podName, err)
		}
		wg.Done()
	}()
	patchValues := make([]PatchStringValue, 0)
	patchString := PatchStringValue{
		Op: "replace",
        // 修改pod label，让deployment在其他节点自动创建新的pod
		Path: "/metadata/labels/app_type",
		Value: "darin",
	}
	patchValues = append(patchValues, patchString)

	patchData, _ := json.Marshal(patchValues)

	_, err := client.CoreV1().Pods(podNamespace).Patch(
		context.TODO(),
		podInfo.podName,
		types.JSONPatchType,
		patchData,
		metav1.PatchOptions{},
	)
	if err != nil {
		panic(err.Error())
	}

	ret := verifyDeploymentStatus(podInfo.deploymentName, client)
	if ret{
		deletePod(podInfo.podName, client)
	}
	return
}

func verifyDeploymentStatus(deploymentName string, client *kubernetes.Clientset) bool {
	log.Println("begin to watch deployment...")

	for i:=0; i<maxRetry;i++{
		// avoid to can not catch deployment change
		time.Sleep(time.Second * interval)
		deploymentInfo, err := client.AppsV1().Deployments(podNamespace).Get(
			context.TODO(),
			deploymentName,
			metav1.GetOptions{},
		)
		if err != nil {
			log.Printf("get deploymemt status error: %v\n",err)
			return false
		}
		readyReplicas := deploymentInfo.Status.ReadyReplicas
		replicas := deploymentInfo.Status.Replicas
		fmt.Printf("%+v\n",deploymentInfo.Status)
		if readyReplicas == replicas{
			log.Printf("%s is ready!\n",deploymentName)
			return true
		}
	}
	log.Printf("%s is not ready!\n", deploymentName)
	return false

}

func deletePod(podName string, client *kubernetes.Clientset)  {
	log.Printf("delete %s pod...\n", podName)
	err := client.CoreV1().Pods(podNamespace).Delete(
		context.TODO(),
		podName,
		metav1.DeleteOptions{},
	)
	if err != nil{
		log.Printf("delete pod %s failed.\n", podName)
	}
}

func main() {

	var podNamespaceFlag, kubeConfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeConfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeConfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
	flag.StringVar(&nodeName, "nodeName", "cn-xxx.16.0.42", "specific node name")
	podNamespaceFlag = flag.String("namespace", "testing","pod namespace")
	podNamespace = *podNamespaceFlag
	flag.Parse()
	if nodeName == ""{
		fmt.Println("Not found node name")
		os.Exit(-1)
	}
	// use the current context in kubeconfig
	config, err := clientcmd.BuildConfigFromFlags("", *kubeConfig)
	if err != nil {
		panic(err.Error())
	}

	// create the clientset
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}
	podsInfo := getPodsWithNodeName(nodeName, clientset)
	log.Printf("pods info :%v\n", podsInfo)
	podNumber := len(podsInfo)
	wg.Add(podNumber)
	for _, v := range podsInfo{
		log.Println("begin to drain:", v.podName)
		pod := v
		go drainPod(pod, clientset)
	}
	wg.Wait()

}


```

end