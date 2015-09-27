---
layout: post
title: Gflags文档（my翻译）
category: another
---

##Glags

###介绍

GFlags是Google开源的一个命令行flag（区别于参数）库。和 getopt() 之类的库不同，flag的定义可以散布在各个源码中，而不用放在一起。一个源码文件可以定义一些它自己的flag，链接了该文件的应用都能使用这些flag。这样就能非常方便地复用代码。如果不同的文件定义了相同的flag，链接时会报错。

GFlags是一个C++库，同时也有一个Python移植，使用完全相同的接口。

###在程序中定义flags
在程序中定义flags很简单，只需要用合适的宏来定义就可以了(这些宏定义在`gflags/gflags.h`文件末尾)，比如下面的例子:

```
    #include <gflags/gflags.h>
    DEFINE_bool(big_menu, true, "Include 'advanced' options in the menu listing");
    DEFINE_string(languages, "english,french,german",
            "comma-separated list of languages to offer in the 'lang' menu");
```
下面是支持的宏类型:

* `DEFINE_bool`: boolean
* `DEFINE_int32`: 32-bit integer
* `DEFINE_int64`: 64-bit integer
* `DEFINE_uint64`: unsigned 64-bit integer
* `DEFINE_double`: double
* `DEFINE_string`: C++ string

gflags追求简单的设计,因此没有复杂类型的宏,比如列表类型。

每个宏有三个参数：**flag名称**，**flag默认值** 和 **描述用法的字符串(help string)**。 当用户在执行程序时输入``--help``时,会显示描述用法的字符串.

注意: 

* 只能定义flag一次. 如果想使用该flag多次,可以在一处定义,其他地方声明.
* flag可以用在main()函数之外,这很方便,但如果flag的默认值在某些环境下不可用,就可以使用 flag_validators来检查.
* `DEFINE_flag` `DECLARE_flag`应该用在全局命名空间中

###获取flag的值
所有定义好的flags都可以像一个普通变量一样使用，只需要在flag名称前面加FLAGS_前缀。在上面的例子中，宏定义了两个变量，`FLAGS_BIG_MENU`(Bool类型)和`FLAGS_languages`(String类型)

可以像使用普通的变量一样使用它们:

```
    if (FLAGS_consider_made_up_languages)
        FLAGS_languages += ",klingon";   // implied by --consider_made_up_languages
    if (FLAGS_languages.find("finnish") != string::npos)
        HandleFinnish();
```

在`gflags.h`中有`get`和`set` flags的值的方法,但一般很少使用.

###使用DECALRE在不同文件中使用同一个Flag

前面介绍的使用flag的方式为:你在文件的头部定义一些Flags,然后在文件中使用它们;如果不定义,你会得到未知变量的错误.

`DECLARE_type`宏定义使你可以获取其他文件中定的Flags,比如你在另一个文件中想要使用big_menu flag，你可以在该文件头部加上`DECLARE_bool(big_menu)`, 就像`extern FLAGS_big_menu`一样.

这样做会产生文件间的依赖,对于大型项目会难以管理.因此,提供一个建议:

***如果你在foo.c中定义了一个flag,要么不要DECLARE它,只在对应的测试中DECLARE或者只在foo.h中DECLARE***

###注册验证函数(RegisterFlagValidator)来检查flag的值
定义了flags以后,可以选择为其注册一个验证函数.如果注册了,每当flag从命令行传入,或者通过 `SetCommandLineOption()`函数改变它的值的时候,验证函数就会被调用. 如果传入的值合理,验证函数反悔true,否则返回false. 如果新设置值时返回false, flag会保持原来的值. 如果默认值返回false,`ParseCommandLineFlags`会失败.

下面是一个使用验证函数的例子:

```
    static bool ValidatePort(const char* flagname, int32 value) {
        if (value > 0 && value < 32768)   // value is ok
        return true;
        printf("Invalid value for --%s: %d\n", flagname, (int)value);
        return false;
    }
    DEFINE_int32(port, 0, "What port to listen on");
    static const bool port_dummy = RegisterFlagValidator(&FLAGS_port, &ValidatePort);
```

最好在定义flag之后紧接着注册验证函数,这样可以保证在main()函数传入命令行flags前完成注册.

`RegisterFlagValidator()` 在注册成功时返回true,否则返回false. 注册失败有可能因为:

* 第一个参数不是一个命令行flag
* flag已经注册了验证函数

###综上所述,如何使用flag

最后一步是告诉可执行程序去处理命令行flags,设置合适的FLAGS_变量的值,只需要使用一个简单的函数:

`google::ParseCommandLineFlags(&argc, &argv, true);`

一般而言,该函数放在main()函数最开始的地方.

函数参数:

* 前两个参数argc,argv就是main()函数的参数.因为函数可能会改变argc和argv,所以传指针.
* 第三个参数称为remove_flags.如果设为true,函数会移除argv中的flags及其参数, 并调整argc的值.(如`foo -big_menu arg`, arg为foo的命令行参数, 会变为`foo arg`); 如果设为false,函数会调整flags的位置在其他命令行参数前面(如`/bin/foo" "arg1" "-q" "arg2"`调整为`"/bin/foo", "-q", "arg1", "arg2"`)


###在命令行中设置flags

有如下方式来设置string类型字符串,以上面的例子为例:

* `app_containing_foo --languages="chinese,japanese,korean"`
* `app_containing_foo -languages="chinese,japanese,korean"`
* `app_containing_foo --languages "chinese,japanese,korean"`
* `app_containing_foo -languages "chinese,japanese,korean"`

对于bool类型的flag,有如下设置方式:

* `app_containing_foo --big_menu`
* `app_containing_foo --nobig_menu`
* `app_containing_foo --big_menu=true`
* `app_containing_foo --big_menu=false`

建议对于非bool型的变量,使用`--variable=value`的方式来设置;对于bool型的变量,使用 `--variable/--novariable`

如果指定一个没有定义的flag,会报fatal error. 你可以使用 --undefok来防止报错.

注意:

* 和getopt()一样, 使用--表示flags终止. 也就是说,在命令`foo -f1 1 -- -f2 2`中,f1是一个flag,f2不是.
* 如果flag被设置多次,则取最后一次的值
* 不支持单字母形式的flag, 也不支持flag合并起来写,如ls -la是非法的
        
###改变flag的默认值

有时候flag在库中定义且含有默认值,但是在使用中你可能需要换一个默认值.这很容易,只要在`ParseCommandLineFlags`函数调用前,为其重新赋值即可.如:

```
    DECLARE_bool(lib_verbose);   // mylib has a lib_verbose flag, default is false
    int main(int argc, char** argv) {
        FLAGS_lib_verbose = true;  // in my app, I want a verbose lib by default
        ParseCommandLineFlags(...);
    }

```
此时仍可以从命令行中读取flag值, 如果没有设置,则为默认值.

###一些特殊的flags
gflag定义了一些内置的flags,分为三类,第一类是"reporting" flag,使用它们来输出一些信息:

* `--help`      shows all flags from all files, sorted by file and then by name; shows the flagname, its default value, and its help string
* `--helpfull`  same as -help, but unambiguously asks for all flags (in case -help changes in the future)
* `--helpshort` shows only flags for the file with the same name as the executable (usually the one containing main())
* `--helpxml`   like --help, but output is in xml for easier parsing
* `--helpon=FILE`       shows only flags defined in FILE.*
* `--helpmatch=S`       shows only flags defined in *S*.*
* `--helppackage`       shows flags defined in files in same directory as main()
* `--version`   prints version info for the executable

第二类影响其他flags如何被设置:

* --undefok=flagname,flagname,...   作为undefok参数的那些flags,如果flags在命令刚中出现但是没有在文件中定义过,可以防止其报错.

第三类是"recursive" flags:

* --fromenv

--fromenv=foo,bar says to read the values for the foo and bar flags from the environment. In concert with this flag, you must actually set the values in the environment, via a line like one of the two below:

   export FLAGS_foo=xxx; export FLAGS_bar=yyy   # sh
   setenv FLAGS_foo xxx; setenv FLAGS_bar yyy   # tcsh
This is equivalent to specifying --foo=xxx, --bar=yyy on the commandline.

Note it is a fatal error to say --fromenv=foo if foo is not DEFINED somewhere in the application. (Though you can suppress this error via --undefok=foo, just like for any other flag.)

It is also a fatal error to say --fromenv=foo if FLAGS_foo is not actually defined in the environment.

* --tryfromenv

--tryfromenv is exactly like --fromenv, except it is not a fatal error to say --tryfromenv=foo if FLAGS_foo is not actually defined in the environment. Instead, in such cases, FLAGS_foo just keeps its default value as specified in the application.

Note it is still an error to say --tryfromenv=foo if foo is not DEFINED somewhere in the application.

* --flagfile

--flagfile=f tells the commandlineflags module to read the file f, and to run all the flag-assignments found in that file as if these flags had been specified on the commandline.

In its simplest form, f should just be a list of flag assignments, one per line. Unlike on the commandline, the equals sign separating a flagname from its argument is required for flagfiles. An example flagfile, /tmp/myflags:

```
    --nobig_menus
    --languages=english,french
```

With this flagfile, the following two lines are equivalent:

```
   ./myapp --foo --nobig_menus --languages=english,french --bar
   ./myapp --foo --flagfile=/tmp/myflags --bar
```

注意在flagfile中很多类型的错误会被忽略掉，比如不能识别的flag，没有指定值的flag。

一般形式的flagfile要复杂一些。写成一组文件名，每行一个，后面加上一组flag，每行一个的形式，可以有多组。文件名可以使用通配符（ * 和 ? ），只有当前可执行模块名和其中一个文件名匹配时才会处理文件名后的flag。flagfile可以直接以一组flag开始，这时这些flag对应到当前可执行模块。

以 # 开头的行作为注释被忽略，前导空白和空行也都会被忽略。

flagfile中还可以使用 --flagfile flag来包含另一个flagfile。

flag会按顺序执行。从命令行开始，遇到flagfile时，执行文件，执行完再继续命令行中后面的flag。

### API
通过API可以直接访问`FLAGS_foo`,以及它的信息,如默认值和help string.`FlagSaver`可以修改flag的值之后又恢复.也有一些API可以在`main()`函数外获取argv的值.

其它还有一些实用的函数,如`google::SetUsageMessage()`和`google::SetVersionString`.

这些函数的用法详见`gflags.h`

### 编译细节
如果你使用下面的代码:

```
    #define STRIP_FLAG_HELP 1    // this must go before the #include!
    #include <gflags/gflags.h>
```

编译时会去除帮助信息,这可以使二进制文件变小,也有利于安全.

{% include references.md %}
