## kubelet 使用docker login config.json

### 实现kubelet使用私有仓库

#### 使用docker config

```bash
docker login harbor-local.unicloudsrv.com -u 'muye' -p "XXX"
## 这样就可以访问私有仓库了，不用配置secret
cp ~/.docker/config.json /var/lib/kubelet/config.json
```

#### 实现原理

一个大佬发现了kubelet启动日志的怪东西

```bash
$ journalctl -u kubelet |grep looking
Dec 23 15:59:00 node1 kubelet[727393]: I1223 15:59:00.059831  727393 config.go:144] looking for config.json at /var/lib/kubelet/config.json
Dec 23 15:59:00 node1 kubelet[727393]: I1223 15:59:00.059849  727393 config.go:144] looking for config.json at /config.json
Dec 23 15:59:00 node1 kubelet[727393]: I1223 15:59:00.059856  727393 config.go:144] looking for config.json at /root/ /.docker/config.json
Dec 23 15:59:00 node1 kubelet[727393]: I1223 15:59:00.059862  727393 config.go:144] looking for config.json at /.docker/config.json
Dec 23 15:59:00 node1 kubelet[727393]: I1223 15:59:00.059936  727393 config.go:110] looking for .dockercfg at /var/lib/kubelet/.dockercfg
Dec 23 15:59:00 node1 kubelet[727393]: I1223 15:59:00.059948  727393 config.go:110] looking for .dockercfg at /.dockercfg
Dec 23 15:59:00 node1 kubelet[727393]: I1223 15:59:00.059955  727393 config.go:110] looking for .dockercfg at /root/ /.dockercfg
Dec 23 15:59:00 node1 kubelet[727393]: I1223 15:59:00.059960  727393 config.go:110] looking for .dockercfg at /.dockercfg

```

根据提示信息，定位下config.go的代码

```go

var (
	preferredPathLock sync.Mutex
	preferredPath     = ""
	workingDirPath    = ""
	homeDirPath, _    = os.UserHomeDir()
	rootDirPath       = "/"
	homeJSONDirPath   = filepath.Join(homeDirPath, ".docker")
	rootJSONDirPath   = filepath.Join(rootDirPath, ".docker")

	configFileName     = ".dockercfg"
	configJSONFileName = "config.json"
)
```

 if searchPaths is empty, the default paths are used.因为默认没有指定searchPaths，所以就在默认的`/var/lib/kubelet`目录了

```go
// ReadDockerConfigJSONFile attempts to read a docker config.json file from the given paths.
// if searchPaths is empty, the default paths are used.
func ReadDockerConfigJSONFile(searchPaths []string) (cfg DockerConfig, err error) {
	if len(searchPaths) == 0 {
		searchPaths = DefaultDockerConfigJSONPaths()
	}
	for _, configPath := range searchPaths {
		absDockerConfigFileLocation, err := filepath.Abs(filepath.Join(configPath, configJSONFileName))
		if err != nil {
			klog.Errorf("while trying to canonicalize %s: %v", configPath, err)
			continue
		}
		klog.V(4).Infof("looking for %s at %s", configJSONFileName, absDockerConfigFileLocation)
		cfg, err = ReadSpecificDockerConfigJSONFile(absDockerConfigFileLocation)
		if err != nil {
			if !os.IsNotExist(err) {
				klog.V(4).Infof("while trying to read %s: %v", absDockerConfigFileLocation, err)
			}
			continue
		}
		klog.V(4).Infof("found valid %s at %s", configJSONFileName, absDockerConfigFileLocation)
		return cfg, nil
	}
	return nil, fmt.Errorf("couldn't find valid %s after checking in %v", configJSONFileName, searchPaths)

}
```





#### 使用docker login config的好处

只用在node上执行一次docker login即可，或者直接下发docker login conifg，不用在每个namespace注入secret还得检测是否有新的namespace创建。



### 引用

1. https://github.com/kubernetes/kubernetes/issues/107202