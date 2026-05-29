# zg_sdk技巧

## test log

**场景**：日志组件单元测试——校验默认 **zap** 引擎、落盘配置、多路输出注册（容器里常配合 stdout + 采集 agent、或双写集中日志平台）。

```go
func Test_Build(t *testing.T) {
	li, err := CreateLogger().Build()
	if li == nil || err != nil {
		t.Errorf("create logger error.")
	}
	// test default config
	if li.(*LoggerImp).Config.RotateFileName != ConfigDefaultFileName {
		t.Errorf("default file error.")
	}
	if li.(*LoggerImp).Config.Level != ConfigDefaultLevel {
		t.Errorf("default level error.")
	}

	if _, ok := li.(*LoggerImp).fixedLogger[ConsoleOutput]; !ok {
		t.Errorf("there has no [%s] logger.", ConsoleOutput)
	}
	if _, ok := li.(*LoggerImp).fixedLogger[FileOutput]; !ok {
		t.Errorf("there has no [%s] logger.", FileOutput)
	}
}

func Test_RegisterOutput(t *testing.T) {
	li, err := CreateLogger().Build()
	if li == nil || err != nil {
		t.Errorf("create logger error.")
	}
	if li.RegisterOutput(FileOutput, os.Stdout) == nil {
		t.Errorf("repeated outputname error.")
	}
	if li.RegisterOutput("Test", os.Stdout) != nil {
		t.Errorf("repeated outputname error.")
	}
}

func Test_reviseConfig(t *testing.T) {
	defaultConfig := reviseConfig(nil)
	if defaultConfig.RotateFileName != ConfigDefaultFileName {
		t.Errorf("use default config error.")
	}

	customConfig := reviseConfig(&Config{
		Format:           "some-format",
		RotateMaxSize:    -1,
		RotateMaxBackups: -1,
		RotateMaxAge:     30,
		EngineName:       "logrus",
	})
	if customConfig.Format != ConfigDefaultFormat {
		t.Errorf("use custom format config error.")
	}
	if customConfig.RotateFileName != ConfigDefaultFileName {
		t.Errorf("use custom filename config error. actual=%s expected=%s", customConfig.RotateFileName, ConfigDefaultFileName)
	}
	if customConfig.RotateMaxSize != ConfigDefaultMaxSize {
		t.Errorf("use custom maxsize config error.")
	}
	if customConfig.RotateMaxBackups != ConfigDefaultMaxBackups {
		t.Errorf("use custom backups config error.")
	}
	if customConfig.RotateMaxAge != 30 {
		t.Errorf("use custom maxage config error.")
	}
	if customConfig.EngineName != "zap" {
		t.Errorf("use custom engine name config error.")
	}
}

func Test_createLoggerWithConfig(t *testing.T) {
	testDir := "/var/log/test.log"
	_ = os.Setenv(meshLogFilePath, testDir)
	defer func() { _ = os.Unsetenv(meshLogFilePath) }()
	li, err := CreateLogger().Build()
	if li == nil || err != nil {
		t.Errorf("create logger error.")
	}
	if li.(*LoggerImp).Config.RotateFileName != testDir {
		t.Errorf("get from env error.")
	}
}
```

# config

### config 实现

**场景**：进程启动时读取 `conf/zego-micro.yaml`，与 **12-factor** 中「配置与代码分离」一致；可先注入测试 YAML 再合并磁盘文件，便于本地/CI 覆盖端口与日志级别。

```go
var data = []byte(`
service: a-brand-new-service
server:
  http:
    port: 9000
  grpc:
    port: 9001
logger:
  format: "json"
  level: "error"
  max-size: 125
  max-age: 7
`)

type typ struct {
	Service string `yaml:"service"`
	Server  struct {
		Http struct {
			Port int `yaml:"port"`
		} `yaml:"http"`
		Grpc struct {
			Port int `yaml:"port"`
		} `yaml:"grpc"`
	} `yaml:"server"`
	Logger struct {
		FileName string `yaml:"file-name"`
		Format   string `yaml:"format"`
		Level    string `yaml:"level"`
		MaxSize  int    `yaml:"max-size"`
		MaxAge   int    `yaml:"max-age"`
	} `yaml:"logger"`
}

var config typ

func init() {
	const path = "./conf/zego-micro.yaml"

	abs, err := filepath.Abs(path)
	if err != nil {
		panic(err)
	}
	bytes, err := ioutil.ReadFile(abs)
	if err != nil {
		panic("Can't find configuration file:" + err.Error())
	}

	err = yaml.Unmarshal(data, &config)
	if err != nil {
		panic(err)
	}

	err = yaml.Unmarshal(bytes, &config)
	if err != nil {
		panic(err)
	}
}

func Copy() typ {
	return config
}
```

[src: raw/ingested/3项目/服务网格-即构/zg_sdk技巧-test-log.md]