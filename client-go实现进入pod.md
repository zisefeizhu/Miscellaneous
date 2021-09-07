实验目的：使用client-go进入任一pod执行命令，模拟终端.
> 比较简单 直接上代码
```go
/*
    模拟 ``ctl exec -it pods -n namespace -- /bin/sh `` 命令

    # ctl get po -n a | grep client
	elasticsearch-client-7bf748d697-bfd9p   1/1     Running   1          36d
 */

func ExecPodCom()  {
	config  := config.KubeConfig()

	clientSet,err  := kubernetes.NewForConfig(config); if err != nil {
		logrus.Println(err.Error())
	}

	// 初始化pod所在的corev1资源组，发送请求
	// PodExecOptions struct 包括Container stdout stdout  Command 等结构
	// scheme.ParameterCodec 应该是pod 的GVK （GroupVersion & Kind）之类的
	req := clientSet.CoreV1().RESTClient().Post().
		Resource("pods").Name("elasticsearch-client-7bf748d697-bfd9p").
		Namespace("a").
		SubResource("exec").
		VersionedParams(&coreV1.PodExecOptions{
			Command: []string{"bash","-c","/bin/sh"},
			Stdin: true,
			Stdout: true,
			Stderr: true,
			TTY: true,       // 打开linux终端
	},scheme.ParameterCodec)

	// remotecommand 主要实现了http 转 SPDY 添加X-Stream-Protocol-Version相关header 并发送请求
	exec, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())

	// 检查是不是终端
	if !terminal.IsTerminal(0) || !terminal.IsTerminal(1) {
		fmt.Errorf("stdin/stdout should be terminal")
	}

	// 这个应该是处理Ctrl + C 这种特殊键位
	oldState, err := terminal.MakeRaw(0); if err != nil {
		fmt.Println(err.Error())
	}

	// 读取当前状态
	fd := int(os.Stdin.Fd())
	oldState, err = terminal.MakeRaw(fd); if err != nil {
		fmt.Println(err.Error())
	}

	defer terminal.Restore(fd, oldState)

	// 用IO读写替换 os stdout
	screen := struct {
		io.Reader
		io.Writer
	}{os.Stdin, os.Stdout}

	// 建立链接之后从请求的sream中发送、读取数据
	if err = exec.Stream(remotecommand.StreamOptions{
		Stdin: screen,
		Stdout: screen,
		Stderr: screen,
		Tty: true,
	}); err != nil {
		fmt.Print(err)
	}
}
```
### 结果
```markdown
 zisefeizhu@zisefeizhudeMacBook-Pro  ~/linkun/goproject/operator  go run main.go                         
INFO[0000] init root.go...                              
/Users/zisefeizhu/linkun/goproject/operator/config/config.yaml
kubeConfig /Users/zisefeizhu/.kube/config
sh-4.2# ls
LICENSE.txt  README.asciidoc  config  jdk  logs     plugins
NOTICE.txt   bin              data    lib  modules  repository-s3-7.8.0.zip
sh-4.2# cd logs/
sh-4.2# ls
gc.log  gc.log.00
```
