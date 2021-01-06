# taboolib-quickstart-archetype  
## 前言
<details><summary>如果你不想听我唠嗑可以直接跳过</summary>

编写原因: [传送门](https://www.mcbbs.net/thread-1146527-1-1.html)  

TabooLib的作者配置的 [Taboolib SDK](https://github.com/taboolib/taboolib-sdk) 是gradle项目  
我是maven钉子户, 就是不肯搞gradle, 打算研究一下怎么用maven整活  

TabooLib原贴中提供了代替TabooLib SDK的方法  

先添加仓库  
```xml
<repository>  
	<id>taboolib</id>  
	<url>http://ptms.ink:8081/repository/maven-public/</url>  
</repository>  
```

然后添加两个依赖
当时是这么写的 **(这个是错的)**
```xml
<dependency>
	<groupId>io.izzel.taboolib</groupId>
	<artifactId>TabooLib</artifactId>
	<version>5.45</version>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>io.izzel.taboolib</groupId>
	<artifactId>TabooLibLoader</artifactId>
	<version>2.9</version>
	<scope>provided</scope>
</dependency>
```

用shade插件进行relocate, 顺便把META-INF给去了
```xml
<plugin>
	<artifactId>maven-shade-plugin</artifactId>
	<version>3.2.4</version>
	<configuration>
		<relocations>
			<relocation>
				<pattern>io.izzel.taboolib.loader</pattern>
				<shadedPattern>${project.groupId}.${project.artifactId}.boot</shadedPattern>
			</relocation>
		</relocations>
		<filters>
			<filter>
				<artifact>*:*</artifact>
				<excludes>
					<exclude>META-INF/**</exclude>
				</excludes>
			</filter>
		</filters>
	</configuration>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>shade</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

这样连编译都过不了, 于是开始排错

- 首先发现TabooLibLoader应当是被打包的, 把scope改成compile
- 然后死活不能从仓库下载, 一直提示"找不到XXX版本"
- 后来去ptms的nexus里浏览, 发现TabooLib和TabooLibLoader都有个"all"的classifier...  

至此"io.izzel.taboolib.loader.Plugin"终于能被识别到了, 并且编译通过  
但是放到本地服去跑会报错:  
<details><summary>报错信息</summary>

```
[09:50:24] [Server thread/ERROR]: Could not load 'plugins\ProjectName v0.0.1-SNAPSHOT.jar' in folder 'plugins'
org.bukkit.plugin.InvalidPluginException: main class `io.freeze_dolphin.test.test' does not extend JavaPlugin
	at org.bukkit.plugin.java.PluginClassLoader.<init>(PluginClassLoader.java:91) ~[paper1122.jar:git-Paper-1618]
	at org.bukkit.plugin.java.JavaPluginLoader.loadPlugin(JavaPluginLoader.java:127) ~[paper1122.jar:git-Paper-1618]
	at org.bukkit.plugin.SimplePluginManager.loadPlugin(SimplePluginManager.java:329) ~[paper1122.jar:git-Paper-1618]
	at org.bukkit.plugin.SimplePluginManager.loadPlugins(SimplePluginManager.java:251) ~[paper1122.jar:git-Paper-1618]
	at org.bukkit.craftbukkit.v1_12_R1.CraftServer.loadPlugins(CraftServer.java:318) ~[paper1122.jar:git-Paper-1618]
	at net.minecraft.server.v1_12_R1.DedicatedServer.init(DedicatedServer.java:222) ~[paper1122.jar:git-Paper-1618]
	at net.minecraft.server.v1_12_R1.MinecraftServer.run(MinecraftServer.java:616) ~[paper1122.jar:git-Paper-1618]
	at java.lang.Thread.run(Unknown Source) [?:1.8.0_261]
Caused by: java.lang.ClassCastException: class io.freeze_dolphin.test.test
	at java.lang.Class.asSubclass(Unknown Source) ~[?:1.8.0_261]
	at org.bukkit.plugin.java.PluginClassLoader.<init>(PluginClassLoader.java:89) ~[paper1122.jar:git-Paper-1618]
	... 7 more
```
</details>

想起plugins.yml还要进行特殊配置...  
打上三项: 
```yaml
lib-version: ${taboolib.lib-version}
loader-version: ${taboolib.loader-version}
lib-download: ${taboolib.lib-download}
```
然后再在pom.xml里设定一下相关变量  

终于成功运行

---
之后就想做个archetype方便以后自己建项目, 顺便发出来方便别人
</details>  

## 从archetype创建项目
- 常见的IDE如Eclipse、IntelliJ在创建maven项目时都会让你选择archetype, 你可以直接用"Add Archetype"功能添加远程archetype到内部Nexus服务器; 也可以先添加archetype repository再从其中选取适当的archetype. 这里提供后一种方法, 以Eclipse为例: 
	1. 在设置(Preferences)中找到: Maven > Archetypes  
    	![1-1](https://raw.github.com/freeze-dolphin/taboolib-quickstart-archetype/master/images/1-1.png)
	2. 在右侧点击 "Add Remote Catalog" 并按照图中的方式进行配置  
    	![1-2](https://raw.github.com/freeze-dolphin/taboolib-quickstart-archetype/master/images/1-2.png)  
		`https://raw.github.com/freeze-dolphin/maven-repository/master/repository/archetype-catalog.xml`
- 你也可以从控制台创建一个基于此archetype的maven项目: 
	1. 打开console或者terminal, 切到你想要创建工程的目录下
	2. 输入以下指令:  
		`mvn org.apache.maven.plugins:maven-archetype-plugin:2.4:generate -DarchetypeGroupId=io.freeze-dolphin.archetypes -DarchetypeArtifactId=taboolib-quickstart-archetype -DarchetypeVersion=1.1.0 -DarchetypeRepository=https://raw.github.com/freeze-dolphin/maven-repository/master/repository/`
	3. 之后会让你配置项目的properties:  
		![2-3](https://raw.github.com/freeze-dolphin/taboolib-quickstart-archetype/master/images/2-3.png)  
		(properties中的"groupId"和""artifactId"以及"author"是必填项, 这些都可以在生成的pom.xml中修改)
	4. 从ide导入生成的maven项目并开始开发  

## 注意事项
- 默认去掉了test包, 如果有需要可以自己加回来
- 通过指令生成项目时请注意一定要用2.4版本的maven-archetype-plugin, 因为貌似只有2.x版本才能在命令行中指定archetypeRepository (当时费了我好久才发现这个= .=)  

## 开源相关
开源地址  
maven repository: [传送门](https://github.com/freeze-dolphin/maven-repository)  
taboolib-quickstart-archetype: [传送门](https://github.com/freeze-dolphin/taboolib-quickstart-archetype)  

欢迎发issue

---

如果这个骨架有帮到你的话请在github给个Star或者评个分叭