---
title: 了解xcodeproj
---

​	在工作中遇到一个需求，比较两个target的不同——编译资源和资源文件的不同。我们都知道.m文件会参与编译；xib，assets文件会作为包的资源文件。头文件在预处理时使用。那xcode是如何组织这些内容的。本文的主角xcodeproj做了这些。

<!--more-->

### xcodeproj结构

```
{
	archiveVersion = 1;
	classes = {
		...
	};
	objectVersion = 48;
	objects = {
		...
	};
	rootObject = 22FAD22721553C5F007B05D8 /* Project object */;
}
```

​	从pbxproj文件的结构上来看，pbxproj文件看起来就像是一个庞大的map。proj文件主要包含了主要两块内容，一、工程的目录结构；二、target的组织。

​	首先，说一下工程目录结构xcodeproj是怎么维护的。在文件中，目录结构是一个树结构。从根目录开始作为根节点，每个文件或者group都是一个节点，依次延展开。每个节点都有自己唯一的uuid作为标识。

<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/了解xcodeproj_1.jpg">

​	然后，就是target，很容易联想到，proj文件中也是已树的结构组织了多个target。对于每个target结构，都会有不同的section。

```xml
		22FAD22E21553C5F007B05D8 /* TestObject */ = {
			isa = PBXNativeTarget;
			buildConfigurationList = 22FAD25B21553C5F007B05D8 /* Build configuration list for PBXNativeTarget "TestObject" */;
			buildPhases = (
				22FAD22B21553C5F007B05D8 /* Sources */,
				22FAD22C21553C5F007B05D8 /* Frameworks */,
				22FAD22D21553C5F007B05D8 /* Resources */,
			);
			buildRules = (
			);
			dependencies = (
			);
			name = TestObject;
			productName = TestObject;
			productReference = 22FAD22F21553C5F007B05D8 /* TestObject.app */;
			productType = "com.apple.product-type.application";
		};
```

​	以上是一个最简单工程中一个target结构。它有一个uuid，用于标识自己。isa表示这个结构的类型，很显然是一个target。然后是buildConfigurationList，从名称上可以很容易看的出来是一个list，list中涵盖了这个target的所有编译模式，例如Debug、Release，每个编译模式都会有一些自己的编译设置，如果有设置的话，默认情况下Release会有Release这个预处理宏。然后是buildPhases，这个section是target所有资源组织的section，第一个Sources，指向的是PBXSourcesBuildPhase对象，里面的是所有需要参与编译的资源，绝大部分是.m文件。然后是Frameworks，指向的是PBXFrameworksBuildPhase对象，这个对象的内容是target所需要的静态库和framework。最后是Resources，指向是的PBXResourcesBuildPhase section的中一个对象，包含了需要打包的文件，比如说图片、xib文件等等。当然一个实际项目中的build phase还远不止这些，还会有PBXCopyFilesBuildPhase, PBXHeadersBuildPhase,PBXShellScriptBuildPhase。接下来是buildRules，一些编译规则；dependencies，target的依赖项。接下来，是描述target的name，targetName，编译包的引用信息等等。

<img src="https://raw.githubusercontent.com/shiroJin/image-storage/master/了解xcodeproj_2.jpg">

## target diff

​	了解了target的文件组织后，比较两个target的文件不同，严谨一些是需要比较所以phase不同，这里简化一下比较PBXSourcesBuildPhase、PBXResourcesBuildPhase、PBXFrameworksBuildPhase的不同。有请我的好伙伴cocoapod出来，这次的主角不是pod，是xcodeproj工具，一个ruby编写的工具。很好的解析了pbxproj文件，面向对象，使用起来也很方便。每个buildfile都有一个file reference，指向一个具体的资源。只需要比较出两个target不同的资源即可。这里我们遍历target的所有资源，然后对于每个资源做target自己的标记，然后输出所有标记只有一个target的文件就是想要的不同结果了。