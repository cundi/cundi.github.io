# appium+python进行手机app自动化测试时send_keys报错 :

```
Message: Parameters were incorrect. We wanted {"required":["value"]} and you sent ["text","sessionId","id","value"]
```

降级selenium后解决：  

1. pip uninstall selenium
2. pip install selenium==3.3.1
