 Dokuwiki 可以在安装插件后实现侧边栏、主体、命名空间的管理


侧边栏
```
~~NOCACHE~~




<minimap suppress="甜新科技 - |start">

<minimap suppress="wiki - |playground - ">


#### head4
   * foo:sidebar  
   [[甜新科技:首页|如何使用React]]

#### simple example page
   * [[wiki:命名空间和页面管理sample]]

#### Bing.com
   * www.bing.com.cn
```

分别对上面的标签语法进行说明：  

不要缓存页面内容
```
~~NOCACHE~~
```

配合bootstrap3风格的minimap，可以在插件中心安装后，使用下面的语法调用：

```xml
<minimap suppress="wiki - |playground - ">
```

使用minimap最侧边栏时主要在系统设置中默认的开始页面命名为：start。否则侧边栏的panel上显示："No Home Page found" 。  

具体使用：

## Syntax

```xml
<minimap suppress="regular expression pattern" includeDirectory="false" debug="false">
```

where:

the suppress option will suppress the "regular expression pattern" part of the page title. It uses the function preg_replace. Actually in the pattern, letters, digits and the following characters are allowed: `space, -, _, |, *, `. The use case is when you add to the title of your page already a namespace.
the includeDirectory permits to include the subdirectories in the list (Default=false)
the debug parameter prints debug information if set to true below the panel header and in the link title (Default=false)

## Example

```xml
<minimap suppress="Dokuwiki - |The Doku - ">
```

With the following page title,  

```
Dokuwiki - Plugin Mini Map
The Doku - Syntax
```

The mini-map will show the following page title:  

```
Plugin Mini Map
Syntax
```

要求不高的话，直接上markdown写。
  
<br/>


另外，页面内的索引也会直接通过`#`标签的层次关系解析好。


sudo docker run --name txt-mediawiki  -p 8078:80 -v /var/data/mediawiki:data:rw -d wikimedia/mediawiki
