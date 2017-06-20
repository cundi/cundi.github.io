---
layout: post
title: "工作日志"
categories: Django Channels
tags: Ansible ESlint Visual\ Studio
---

* content
{toc}



出现每次连接虚拟机都需要输入私钥的口令：  

```
➜  Ansible ansible -i hosts localvm -a 'date'
Enter passphrase for key '/Users/ml/.ssh/local_vm_rsa': 
192.168.16.35 | SUCCESS | rc=0 >>
Fri Jun 16 06:00:48 UTC 2017
```

解决办法：  

```
ssh-add /Users/ml/.ssh/local_vm_rsa
Enter passphrase for /Users/ml/.ssh/local_vm_rsa: 
Identity added: /Users/ml/.ssh/local_vm_rsa (/Users/ml/.ssh/local_vm_rsa)
```

验证下：  

```
ansible -i hosts localvm -a 'date' 
192.168.16.35 | SUCCESS | rc=0 >>
Fri Jun 16 06:05:37 UTC 2017
```


Visual Studio在保存js文件时报错： ESLint configuration is invalid: - Unexpected top-level property "ecmaFeatures".  

解决办法： 修改`.eslintrc`，在parserOptions中嵌套 ecmaFeaturs 。  

关于webpack 2 no longer allows custom properties in configuration：  

```
gulp package                                
[16:12:11] Requiring external module babel-register
[16:12:12] Using gulpfile ~/Downloads/txtComponents/gulpfile.babel.js
[16:12:12] Starting 'clean'...
[16:12:12] Finished 'clean' after 9.44 ms
[16:12:12] Starting 'package'...
[16:12:12] 'package' errored after 77 ms
[16:12:12] WebpackOptionsValidationError: Invalid configuration object. Webpack has been initialised using a configuration object that does not match the API schema.
 - configuration has an unknown property 'historyApiFallback'. These properties are valid:
   object { amd?, bail?, cache?, context?, dependencies?, devServer?, devtool?, entry, externals?, loader?, module?, name?, node?, output?, performance?, plugins?, profile?, recordsInputPath?, recordsOutputPath?, recordsPath?, resolve?, resolveLoader?, stats?, target?, watch?, watchOptions? }
   For typos: please correct them.
   For loader options: webpack 2 no longer allows custom properties in configuration.
     Loaders should be updated to allow passing options via loader options in module.rules.
     Until loaders are updated one can use the LoaderOptionsPlugin to pass these options to the loader:
     plugins: [
       new webpack.LoaderOptionsPlugin({
         // test: /\.xxx$/, // may apply this only for some modules
         options: {
           historyApiFallback: ...
         }
       })
     ]
 - configuration.resolve has an unknown property 'fallback'. These properties are valid:
   object { alias?, aliasFields?, cachePredicate?, descriptionFiles?, enforceExtension?, enforceModuleExtension?, extensions?, fileSystem?, mainFields?, mainFiles?, moduleExtensions?, modules?, plugins?, resolver?, symlinks?, unsafeCache?, useSyncFileSystemCalls? }
 - configuration.resolve.extensions[0] should not be empty.
 - configuration.resolveLoader has an unknown property 'fallback'. These properties are valid:
   object { alias?, aliasFields?, cachePredicate?, descriptionFiles?, enforceExtension?, enforceModuleExtension?, extensions?, fileSystem?, mainFields?, mainFiles?, moduleExtensions?, modules?, plugins?, resolver?, symlinks?, unsafeCache?, useSyncFileSystemCalls? }
    at webpack (/Users/ml/Downloads/txtComponents/node_modules/webpack/lib/webpack.js:19:9)
    at Gulp.pack (/Users/ml/Downloads/txtComponents/tasks/package.js:8:2)
    at module.exports (/Users/ml/Downloads/txtComponents/node_modules/orchestrator/lib/runTask.js:34:7)
    at Gulp.Orchestrator._runTask (/Users/ml/Downloads/txtComponents/node_modules/orchestrator/index.js:273:3)
    at Gulp.Orchestrator._runStep (/Users/ml/Downloads/txtComponents/node_modules/orchestrator/index.js:214:10)
    at /Users/ml/Downloads/txtComponents/node_modules/orchestrator/index.js:279:18
    at finish (/Users/ml/Downloads/txtComponents/node_modules/orchestrator/lib/runTask.js:21:8)
    at /Users/ml/Downloads/txtComponents/node_modules/orchestrator/lib/runTask.js:43:4
```


解决办法： 按照官网文档 ，将`fallback: path.join(__dirname, 'node_modules')`改为：`modules: [path.resolve(__dirname, "node_modules")]`。暂时注释了`//historyApiFallback: true,`，还未找到热加载时启用它的方法。  



