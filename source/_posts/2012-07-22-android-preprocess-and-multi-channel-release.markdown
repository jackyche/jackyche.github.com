---
layout: post
title: "Android编译预处理及多渠道发布"
date: 2012-07-22 11:06
comments: true
categories: Android
---

Android游戏在国内的渠道实在太乱了，像UC、91、QQ、应用汇、安智市场、N多网这样的平台，大大小小几十个。一款游戏如果需要在所有的渠道上发布，还得为每个渠道定制相应的版本。这种活如果人肉一个渠道一个渠道去做，有点劳民伤财。考虑到不同渠道的版本在代码和资源上99%都是相同的，这种事情应该用脚本去完成。

五六年前J2ME游戏经常需要为上百款设备做适配，当时比较流行的发布策略是[Ant](http://ant.apache.org/) + [Antenna](http://antenna.sourceforge.net/)。考虑了一下，这种策略对于应付Android的多渠道也是绰绰有余的。

这篇文章分享一下基于Ant + Antenna的Android游戏多渠道发布策略。

<!-- more -->


不同渠道的版本，通常情况下差异在两个方面：少量代码 + 资源。

###首先处理代码的差异性：

在C/C++里面，因为有#ifdef，#ifndef这种条件预编译宏的存在，可以很方便为每个渠道做代码的定制化，但是Java不支持这种方式。那就是说，一份Java代码只能编译出一个执行文件(不要跟我说动态运行这件事，这不是编译期的事情)，那我们如何来解决这个问题呢。换一种思路来考虑问题：事实上我们并不排斥多份编译代码的存在，我们关心的是能否只维护一份代码。

Antenna用一种取巧的方式解决了这个问题：每次通过对原始代码做预处理，得到一份中间代码，然后对这份中间代码进行编译发布。

Antenna提供了类似C/C++里的条件预编译宏，具体支持的宏定义可以参考[这里](http://antenna.sourceforge.net/wtkpreprocess.php)。比较大的差别在于，Antenna的预编译宏是以`//`开头的，它以注释的形式存在于代码中，不会导致原始代码的编译错误。

下面是一个示例：
{% codeblock lang:java %}
	//#ifdef CHANNEL_91
	//@ setContentView(R.layout.hello_activity_91);
	//#elifdef CHANNEL_UC
		setContentView(R.layout.hello_activity_uc);
	//#elifdef CHANNEL_QQ
	//@ setContentView(R.layout.hello_activity_qq);
	//#endif
{% endcodeblock %}
然后我们可以定义Ant任务
{% codeblock lang:xml %}
	<!-- wtk is not neccessory ,but still need to define wtk.home -->
	<property name="wtk.home" value="/tmp" />
	<path id="antenna.lib">
	 	<pathelement location="antenna/antenna-bin-1.2.1-beta.jar"/>
	 </path>
	 <!-- Defining Antenna's WTK tasks hard coded, do NOT ask -->
	<taskdef name="preprocess" classname="de.pleumann.antenna.WtkPreprocess">
		<classpath refid="antenna.lib"/>
	</taskdef>


	<!-- 
	  input: macros.definition 
	  environment: srcdir, destdir
	-->
	<target name="preproc">
		<echo message="proproc: ${version.definition}" />
		<preprocess srcdir="${srcdir}" destdir="${outdir-preprocess}" symbols="${macros.definition}" />
	</target>
	
	<target name="preproc_uc" description="release for uc platform">
		<antcall target="preproc">
			<param name="macros.definition" value="CHANNEL_UC" />
		</antcall>
	</target>
	
	<target name="preproc_qq" description="release for qq platform" >
		<antcall target="preproc">
			<param name="macros.definition" value="CHANNEL_QQ" />
		</antcall>
	</target>
{% endcodeblock %}	
这里有两个Ant任务：`preproc_uc`和`preproc_qq`。当我们执行`preproc_uc`的时候，会得出以下的中间代码：
{% codeblock lang:java %}
	//#ifdef CHANNEL_91
	//@ setContentView(R.layout.hello_activity_91);
	//#elifdef CHANNEL_UC
		setContentView(R.layout.hello_activity_uc);
	//#elifdef CHANNEL_QQ
	//@ setContentView(R.layout.hello_activity_qq);
	//#endif
{% endcodeblock %}		
而当我们执行`preproc_qq`的时候，则会得出以下的中间代码：
{% codeblock lang:java %}
	//#ifdef CHANNEL_91
	//@ setContentView(R.layout.hello_activity_91);
	//#elifdef CHANNEL_UC
	//@	setContentView(R.layout.hello_activity_uc);
	//#elifdef CHANNEL_QQ
	    setContentView(R.layout.hello_activity_qq);
	//#endif
{% endcodeblock %}		
之后我们就可以根据生成的不同中间代码，编译不同的渠道版本。

Antenna支持多个编译宏的形式，在这段示例里`macros.definition`的编译宏参数以逗号分隔就可以了，如：
{% codeblock lang:xml %}
	<target name="preproc_91" description="release for qq platform" >
		<antcall target="preproc">
			<param name="macros.definition" value="CHANNEL_91, FLAG_91, PARAMS_91" />
		</antcall>
	</target>
{% endcodeblock %}

同样它也支持一系列的条件判断语句，具体请参考[这里](http://antenna.sourceforge.net/wtkpreprocess.php)。

这里需要注意的是，Antenna在执行的时候，会把不需要的相应代码块用`//@`注释掉。

###接下来解决资源打包问题：
Android打包应用，一般会经过以下几个步骤：

1. 用aapt命令生成R.java文件
- 用javac命令编译java源文件生成class文件
- 用dx将class文件转换成classes.dex文件
- 用aapt命令生成资源包文件resources.ap_
- 用apkbuilder打包资源和classes.dex文件,生成unsigned.apk
- 用jarsinger命令对apk认证,生成signed.apk
- 用jarsinger的verify功能对apk进行校验
- 做相应的发布清理工作

我们可以用Ant脚本定义每一个步骤：
{% codeblock 生成R.java文件 lang:xml %}
    <target name="gen-R">  
        <echo>Generating R.java from the resources...</echo>  
        <exec executable="${aapt}" failonerror="true">  
            <arg value="package" />  
            <arg value="-f" />  
            <arg value="-m" />  
            <arg value="-J" />  
            <arg value="${outdir-gen}" />  
            <arg value="-S" />  
            <arg value="${resource-dir}" />  
            <arg value="-M" />  
            <arg value="${manifest-xml}" />  
            <arg value="-I" />  
            <arg value="${android-jar}" />  
        </exec>  
    </target>
{% endcodeblock %}

{% codeblock 编译java源文件 lang:xml %}
    <target name="compile" depends="gen-R">  
        <echo>Compiling java source code...</echo>  
        <javac encoding="utf-8" target="1.6" destdir="${outdir-classes}" bootclasspath="${android-jar}">  
        	<src path="${outdir-preprocess}"/>
        	<src path="${outdir-gen}"/>
        	
        	<classpath>  
                <fileset dir="${external-lib}" includes="*.jar"/>  
            </classpath>  
        </javac>  
    </target>
{% endcodeblock %}

这里需要编译生成的R.cpp，以及经过Antenna处理过的中间代码。

{% codeblock 将class文件转换成classes.dex文件 lang:xml %}
    <target name="dex" depends="compile">  
        <echo>Converting compiled files and external libraries into a .dex file...</echo>  
        <exec executable="${dx}" failonerror="true">  
            <arg value="--dex" />    
            <arg value="--output=${dex-ospath}" />  
            <arg value="${outdir-classes-ospath}" />  
            <arg value="${external-lib-ospath}"/>  
        </exec>  
    </target>	
{% endcodeblock %}

{% codeblock 生成资源包文件resources.ap_ lang:xml %}
    <target name="package-res-and-assets">  
        <echo>Packaging resources and assets...</echo>  
        <exec executable="${aapt}" failonerror="true">  
            <arg value="package" />  
            <arg value="-f" />  
            <arg value="-M" />  
            <arg value="${manifest-xml}" />  
            <arg value="-S" />  
            <arg value="${resource-dir}" />  
            <arg value="-A" />  
            <arg value="${asset-dir}" />  
            <arg value="-I" />  
            <arg value="${android-jar}" />  
            <arg value="-F" />  
            <arg value="${resources-package}" />  
        </exec>  
    </target>
{% endcodeblock %}

{% codeblock 打包未签名apk lang:xml %}
    <target name="package" depends="dex, package-res-and-assets">  
        <echo>Packaging unsigned apk for release...</echo>  
        <exec executable="${apkbuilder}" failonerror="true">  
            <!--<arg value="${out-unsigned-package-ospath}" />-->
        	<arg value="${out-unsigned-package-ospath}" />
            <arg value="-u" />  
            <arg value="-z" />  
            <arg value="${resources-package-ospath}" />  
            <arg value="-f" />  
            <arg value="${dex-ospath}" />  
            <arg value="-rf" />  
            <arg value="${srcdir-ospath}" />  
        </exec>  
        <echo>It will need to be signed with jarsigner before being published.</echo>  
    </target>
{% endcodeblock %}

{% codeblock 对apk进行认证 lang:xml %}
    <target name="jarsigner" depends="package">  
        <echo>Packaging signed apk for release...</echo>  
        <exec executable="${jarsigner}" failonerror="true">  
            <arg value="-keystore" />  
            <arg value="${keystore-file}" />  
            <arg value="-storepass" />  
            <arg value="qwer1234" />  
            <arg value="-keypass" />  
            <arg value="qwer1234" />  
            <arg value="-signedjar" />  
            <arg value="${outdir-release}/${release.name}.apk" />  
            <arg value="${out-unsigned-package-ospath}"/>  
            <!-- don't forget alis of certification -->  
            <arg value="release.keystore"/>  
        </exec>  
    </target> 
{% endcodeblock %}

如果你没有生成keystore文件，需要用以下命令生成：

	keytool -genkey -v -keystore <file_name> -alias <alias_name> -keyalg RSA -validity 10000
照着提示生成就可以了。

这里需要注意的是jarsigner需要提供keystore的别名。

{% codeblock 校验认证 lang:xml %}
    <target name="verifysign" depends="jarsigner">  
        <echo>Verfify signed apk for release...</echo>  
        <exec executable="${jarsigner}" failonerror="true">  
            <arg value="-verify" />  
            <arg value="-verbose" />  
            <arg value="${outdir-release}/${release.name}.apk" />  
        </exec>  
    </target>  
{% endcodeblock %}

{% codeblock 做发布清理 lang:xml %}
    <target name="release" depends="verifysign">  
        <delete file="${out-unsigned-package-ospath}"/>  
        <echo>APK is released. path:${outdir-release}/${release.name}.apk</echo>
    </target> 
{% endcodeblock %}

有了这些脚本，我们就可以对资源打包做灵活性控制了。例如我们对于不同的渠道，可以用不同的`AndroidManifest.xml`，打包不同的`res`文件夹，诸如此类。


最后我在GitHub上写了一个示例的工程，仅供参考：[AndroidAntennaExample](https://github.com/jackyche/AndroidAntennaExample)

###参考资料
- [http://antenna.sourceforge.net/wtkpreprocess.php](http://antenna.sourceforge.net/wtkpreprocess.php)
- [http://stackoverflow.com/questions/1797930/how-to-get-eclipse-to-recognize-preprocessor-statements](http://stackoverflow.com/questions/1797930/how-to-get-eclipse-to-recognize-preprocessor-statements)
- [http://www.cnblogs.com/qianxudetianxia/archive/2012/07/04/2573687.html](http://www.cnblogs.com/qianxudetianxia/archive/2012/07/04/2573687.html)