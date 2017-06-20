
取消devtool的注释，避免打包出来的js代码包裹在eval（字符串）。  
同时指明导出库的类型和名称。  

```js
//devtool: 'cheap-module-eval-source-map',
    //devtool: 'eval',
    //historyApiFallback: true,
    entry: {
        app: [
            './' + config.src.main
        ]
    },
    output: {
        libraryTarget: "commonjs-module",
        library: 'PublicComponents',
        //path: path.join(__dirname, config.dest),
        filename: 'txtcomponents.js',
        path: path.resolve(__dirname, "./static/"),
        //publicPath: "/static/",

    },
```   
