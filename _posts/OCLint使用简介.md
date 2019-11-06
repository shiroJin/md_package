---
title: OCLint使用简介
---

OCLint是iOS中代码质量控制的工具，支持自定义规则，适用于团队协作开发，规范代码。

<!--more-->

## 一、OCLint 安装

macOS用户可以直接使用homebrew安装。

#### Installing OCLint

```ruby
$ brew tap oclint/formulae
$ brew install oclint
```

#### Updating OCLint

```ruby
$ brew update
$ brew upgrade oclint
```

#### Installation Path

安装路径使用brew命令查看

```ruby
$ brew list oclint
```

#### Finally

项目分析需要用到xcpretty工具。安装方式如下：

```ruby
$ gem install xcpretty
```



## 二、在Xcode中使用OCLint

#### Setting up Target

* 在项目中新增一个新的target，选择<b><kbd><font color="red">Aggregate</font></kbd></b>。<img src="http://docs.oclint.org/en/stable/_images/xcode_screenshot_1.png" width="70%" height="70%">
* 给新的target命名。这里简单的命令为OCLint，你可以创建多个target，每个target在不同的方面对代码进行分析。
* 在创建的target中选择<font color="Blue">`Build Phase`</font>，新增Run Script，输入一段脚本，用于创建分析文件和分析使用。<img src="http://docs.oclint.org/en/stable/_images/xcode_screenshot_4.png" width="70%" height="70%">
* <kbd><font color="Blue">Command+B</font></kbd>编译程序就可以看到分析结果了。<img src="http://docs.oclint.org/en/stable/_images/xcode_screenshot_8.png" width="70%" height="70%">

##### 补充：

oclint-json-compilation-database命令有过滤项和配置两部分选项可以配置。过滤选项用于过滤一些不需要分析的文件或者文件夹，例如执行oclint-json-compilation-database -e Pod命令，就不会分析pod目录下的代码，-e支持链式使用，即可以过滤过个规则。过滤和配置两部分用符号<font color='blue'>`--`</font>分隔，下文会更详细的介绍oclint的选项配置。

## 三、OCLint选项配置

#### Customize Rule Thresholds

在使用<font color="Blue">oclint-json-compilation-database</font>命令的时候可以自定义规则临界值(即定义规则的参数)。在命令后面添加<kbd><font color="red">-rc <threshold_name>=<new_value></font></kbd>选项。多个自定义规则可以使用连续的-rc选项。如下：

```ruby
-rc <threshold_name>=<new_value> -rc <threshold_name>=<new_value>
```

<b>Available Thresholds</b>	

Here is a list of all thresholds defined in OCLint 0.13:

| Name                    | Description                                                  | Default |
| ----------------------- | ------------------------------------------------------------ | ------- |
| CYCLOMATIC_COMPLEXITY   | Cyclomatic complexity of a method                            | 10      |
| LONG_CLASS              | Number of lines for a C class or Objective-C interface, category, protocol, and implementation | 1000    |
| LONG_LINE               | Number of characters for one line of code                    | 100     |
| LONG_METHOD             | Number of lines for a method or function                     | 50      |
| LONG_VARIABLE_NAME      | Number of characters for a variable name                     | 20      |
| MAXIMUM_IF_LENGTH       | Number of lines for the `if` block that would prefer an early exists | 15      |
| MINIMUM_CASES_IN_SWITCH | Count of case statements in a switch statement               | 3       |
| NPATH_COMPLEXITY        | NPath complexity of a method                                 | 200     |
| NCSS_METHOD             | Number of non-commenting source statements of a method       | 30      |
| NESTED_BLOCK_DEPTH      | Depth of a block or compound statement                       | 5       |
| SHORT_VARIABLE_NAME     | Number of characters for a variable name                     | 3       |
| TOO_MANY_FIELDS         | Number of fields of a class                                  | 20      |
| TOO_MANY_METHODS        | Number of methods of a class                                 | 30      |
| TOO_MANY_PARAMETERS     | Number of parameters of a method                             | 10      |

<b>.oclint文件</b>

oclint还支持配置文件配置的方式。在工程项目下创建<font color="red">.oclint</font>文件。在文件中输入想要的选项即可达到命令行方式同样的效果。

```
rule-configurations:
  - key: THRESHOLD_1
    value: new_value_1
  - key: THRESHOLD_2
    value: new_value_2
  ...
```

#### Select OCLint Rules for Inspection

在oclint的安装文件<kbd>${installation-path}/lib/oclint/rules</kbd>目录下可以看到oclint默认的分析规则。使用命令行可以指定或者过滤某条规则，命令如下：

```ruby
oclint -R /path/to/rules -disable-rule GotoStatement
```

推荐使用<font color="blue">.oclint</font>配置文件。

```
rule-paths:
  - /path/to/rules
rules:
disable-rules:
  - GotoStatement
```

#### Pick Up the Right Reporter

oclint现在支持多种输出方式。如果需要在代码中直接显示信息，<font color="blue">-report-type</font>设置为xcode。如果想要汇总输出，可以选择html格式。同样可以在<font color="blue">.oclint</font>配置文件中配置：

```
report-type: html
output: oclint.html	
```

html的文件输出可以直观的看到一个总览。效果如下：

![screenshot](/Users/remain/Library/Group Containers/Q79WDW8YH9.com.evernote.Evernote/Evernote/quick-note/13630831-personal-app.yinxiang.com/quick-note-nJN2un/attachment--5mZIE6/screenshot.png)

#### .oclint文件

.oclint文件是oclint的配置文件，前面也有所提及。.oclint文件可以放在工程项目的根目录，也放在全局的文件路径下，例如<font color="blue">~/.oclint </font>或 <font color="blue">/etc/.oclint</font>。这里详细的描述下有哪些选项可以配置。

| Option                       | Type                       | Mapping Command Option        |
| ---------------------------- | -------------------------- | ----------------------------- |
| rules                        | List of strings            | -rule                         |
| disable-rules                | List of strings            | -disable-rule                 |
| rule-paths                   | List of strings            | -R                            |
| rule-configurations          | List of associative arrays | -rc                           |
| output                       | String                     | -o                            |
| report-type                  | String                     | -report-type                  |
| max-priority-1               | Integer                    | -max-priority-1               |
| max-priority-2               | Integer                    | -max-priority-2               |
| max-priority-3               | Integer                    | -max-priority-3               |
| enable-global-analysis       | Boolean                    | -enable-global-analysis       |
| enable-clang-static-analyzer | Boolean                    | -enable-clang-static-analyzer |

看到最后两个配置。oclint的分析还支持了clang的静态分析，也就是在这个选项打开的情况下，结果输出还会包含clang的静态分析结果。

### 四、OCLint部分规则

这里简单介绍几个实用的oclint规则，现在xcode自己的编译分析已经覆盖很多方面了，oclint中的一些规则也有些过时，想了解全部规则可以参考[OCLint官网规则目录](http://docs.oclint.org/en/stable/rules/index.html)。如果想自定义规则可以参考[编写自定义规则](http://docs.oclint.org/en/stable/devel/rules.html)。

1、MissingHashMethod

> 简单解释：`isEqual` 方法被重写, `hash` 方法也应该被重写。

```objc
@implementation BaseObject 
- (BOOL)isEqual:(id)obj { 
 return YES; 
} 
/*
- (int)hash is missing; If you override isEqual you must override hash too.
*/ 
@end
```

2、CallingProtectedMethod

> 简单解释：在`Objective-C` 中虽然没有`protected`这个概念。但是在设计的角度，我们有时希望一个方法只希望被它自己或者它的子类调用。这个方法可以模仿`protected`在有人调用的时候给一个警告。

```objc
@interface A : NSObject
- (void)foo __attribute__((annotate("oclint:enforce[protected method]")));
@end

@interface B : NSObject
@property (strong, nonatomic) A* a;
@end

@implementation B
- (void)bar {
    [self.a foo]; // calling protected method foo from outside A and its subclasses
}
@end
```

3、MissingAbstractMethodImplementation

> 简单解释：由于`Objective`的`runtime`特性，抽象方法可以被声明，可以不实现，该规则验证子类是否正确实现抽象方法。

```objc
@interface Parent

- (void)anAbstractMethod __attribute__((annotate("oclint:enforce[abstract method]")));

@end

@interface Child : Parent
@end

@implementation Child

/*
// Child, as a subclass of Parent, must implement anAbstractMethod
- (void)anAbstractMethod {}
*/

@end
```

4、UnnecessaryDefaultStatement

> 简单解释：如果`switch`覆盖了所有的条件，`default`是不需要的应该被移除。如果不是`default`还是需要的。

```objc
typedef enum {
    value1 = 0,
    value2 = 1
} eValues;

void aMethod(eValues a)
{
    switch(a)
    {
        case value1:
            break;
        case value2:
            break;
        default:          // this break is obsolete because all
            break;        // values of variable a are already covered.
    }
}
```

5、MisplacedDefaultLabel

>简单解释：`default`应该在`switch`的最后。

```
void example(int a)
{
    switch (a) {
        case 1:
            break;
        default:  // the default case should be last
            break;
        case 2:
            break;
    }
}
```

6、InvertedLogic

>简单解释：倒置逻辑不易理解。

```objc
int example(int a)
{
    int i;
    if (a != 0)             // if (a == 0)
    {                       // {
        i = 1;              //      i = 0;
    }                       // }
    else                    // else
    {                       // {
        i = 0;              //      i = 1;
    }                       // }

    return !i ? -1 : 1;     // return i ? 1 : -1;
}
```

7、MissingBreakInSwitchStatement

> 简单解释：在`switch`语句中缺失了`break`，很有可能引发`bug`。

``` objc
void example(int a)
{
    switch (a) {
        case 1:
            break;
        case 2:
            // do something
        default:
            break;
    }
}

```

### 五、消除OCLint警告

OCLint提供了多种消除警告的方法，首先对于规则本身存在问题的情况下可以自主修改规则代码，详情可见官方文档。对于不需要这个规则的情况，可以在.oclint文件中禁用这个规则，或者可以将规则从目录下移除。如果一些特殊情况，可以用标注去声明消除某个规则，遇到这类声明OCLint不会分析声明消除的规则。

<b>标注语法</b>

> Unused method parameter是这个规则的名称。表示在区域内忽略这个规则。

```
__attribute__((annotate("oclint:suppress[unused method parameter]")))
```

> 有多个规则需要忽略时使用

```
__attribute__((annotate("oclint:suppress[high cyclomatic complexity]"), annotate("oclint:suppress[high npath complexity]"), annotate("oclint:suppress[high ncss method]")))
```

> 忽略所有规则

```
__attribute__((annotate("oclint:suppress")))
```

<b>作用域</b>	

在作用域内，标注都是有效的。

```objc
bool __attribute__((annotate("oclint:suppress"))) aMethod(int aParameter)
{
    // warnings within this method are suppressed at all
    // like unused aParameter variable and empty if statement
    if (1) {}

    return true;
}

- (IBAction)turnoverValueChanged:
    (id) __attribute__((annotate("oclint:suppress[unused method parameter]"))) sender
    // suppress sender variable
{
    int i; // won't suppress this one
    [self calculateTurnover];
}

- (void)dismissAllViews:(id)currentView parentView:(id)parentView
   __attribute__((annotate("oclint:suppress")))
   // again, suppress the warnings for entire method
{
    [self dismissTurnoverView];
    // plus 30+ lines of code of dismissing other views
}
```

<b>!OCLint注释</b>

另外也可以用注释去消除某个规则。通过添加<font color='red'>`//!OCLint`</font>注释。

```objc
void a() {
    int unusedLocalVariable; //!OCLINT
}
```

当然便于更好的理解，应该添加一些注释去修饰一下原因，以便后续维护。

```objc
int Results::numberOfWarnings()
{
    std::lock_guard<std::mutex> lock(_mutex); //!OCLint(FP - meant to be unused)
    //Everything after the letter 't' is ignored, but just for better readability
    return _compilerWarningSet->numberOfViolations();
}
```

