# appium

## Android Testing
- 安装JDK
- 安装Android Studio
- 配置.bash_profile

```python
```

#### Tips
1. appium会话覆盖：

```shell
appium --session-override 
```

2. appium禁止应用重新安装

```
desired_caps['noReset'] = True
```

or 

```
--no-reset for appium
```

## 使用 uiautomatorviewer
Mac OS上的路径在SDK中：  

```shell
/Users/ml/Library/Android/sdk/tools/bin/uiautomatorviwer
```

可以链接为全局命令：  

```shell
sudo ln -s /Users/ml/Library/Android/sdk/tools/bin/uiautomatorviwer /usr/local/bin/uiautomaterviewer
```

点击Device Screenshot(uiautomater dump),稍后便会连接模拟器中正在运行的应用。  

