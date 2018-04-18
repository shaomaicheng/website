---
title: 初探Kotlin和anko进行Android开发
date: 2017-02-23 02:54:02
tags:
---
kotlin是一门基于jvm的编程语言，最近进行了关于kotlin和 anko的研究。并且结合现在的APP设计模式，设想了初步的开发方式。并且准备应用在新的项目中。

**关于Kotlin和anko**
Kotlin是大名鼎鼎的JB公司开发的jvm语言，官网地址为；http://kotlinlang.org/
官网的介绍为
`Statically typed programming language for the JVM, Android and the browser`

Kotlin的设计思想非常的轻量，尽可能的去复用java代码，不到万不得已的时候，一般不会自己去实现一套大而全的库。这使得Kotlin非常的轻量，集成到Android的project并不会很明显的影响最终的打包大小。

关于Kotlin的优点，自己总结了几点：
1. 和Java的无缝调用，这在初期不需要投入非常大的精力，即使遇到搞不定的坑，也不必担心影响业务开发的进度，直接换成java就好了
2. 大量的语法糖，使得代码非常的简洁，熟悉之后的开发效率也要高于Java。例如扩展函数，简单的封装再也不需要写一大堆Utils工具类，直接灵活的给某些类添加扩展方法就可以了。
比如希望Activity中有一个弹出Toast的方法，就可以写成
```
inline fun Activity.toast(message : Int) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }
```
这样在Activity类中就多出了一个toast方法，实际上在anko中，也有大量已经写好的扩展方法，可以直接使用DSL语法去写UI。
再例如when语句
```
when(x) {
  1-> {}
  2-> {}
}
```
很明显比
```
switch(x) {
  case 1:
    break;
  case 2:
    break;
  default:
    break;
}
```
要简洁很多。
3. 更加安全，Kotlin似乎比较想消灭空引用，在Java中，调用一个null对象会抛出NullPointException，在Kotlin中，不能为空的对象，例如String对象，会写成
```
var a: String? = "abc"
```
这个时候只有变量为非空的时候才会调用方法
4. 良好的环境和社区
Kotlin目前还是属于比较新的技术，很多人也都在尝试它的有点。包括Rx系列也出了RxKotlin，既RxJava的Kotlin版

至于更深入的比如其他有点和语言特性，包括和Java的对比，我会继续研究下去。下面用一个登录界面的实例，来讲解一下Kotlin+anko开发的方式入门

首先是Kotlin和anko的获取，在这里不多说
Kotlin可以在Android Studio装插件进行支持。关于Anko，有几个坑，我这里指出来一下。
Anko的github地址为https://github.com/Kotlin/anko
在JB的gitbook文档里面关于Gradle获取anko的代码没有给全，导致坑了我。在这我给出我的project里面的gradle代码。方便大家参考
首先在project的build文件加入如下代码：
```
dependencies {
        classpath 'com.android.tools.build:gradle:2.2.3'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"  //这是需要你加入的
    }

allprojects {
    repositories {
        jcenter()
        maven {url 'https://dl.bintray.com/jetbrains/anko'} //这是你需要加入的
    }
}
```
然后，在你app moudle的build文件里面添加依赖：
```
//anko
    compile "org.jetbrains.anko:anko-sdk15:0.9.1" // So here it's 15 too
    compile "org.jetbrains.anko:anko-appcompat-v7:0.9.1"
    compile "org.jetbrains.anko:anko-design:0.9.1"
    compile "org.jetbrains.anko:anko-recyclerview-v7:0.9.1"
```
相信这个大家都能看明白，就不多说了。

接下来介绍下这个登录页面的结构
设计模式使用平时常用的MVP模式，其中因为P层为逻辑层，负责数据处理、网络请求，比较严谨和复杂，所以，目前阶段设定的是纯java进行编写。
而view层，根据kotlin的优势，选择使用anko进行编写，不使用xml进行编写。
这样的好处在anko的github README文件中是这样描述的：
* 不安全
* 没有空安全
* 迫使你为了每一个布局去写很多相似甚至重复的代码
* XML在设备上浪费CPU时间和电量（应该是需要进行解析的原因）
* 不允许代码重用（没有完全理解，可能说的不是include标签而是自定义的layout）
至于Contract接口以及实体对象，可以直接使用Kotlin编写，第一为了语法简洁，第二不用写一大堆setter/getter方法

现编写MainActivity类，进行UI展示和事件等逻辑
```
class MainActivity : AppCompatActivity(), MainContract.View {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
    }
}
```
接下来在onCreate中编写ui布局，登录布局比较简单，就是2个输入框和一个按钮，当然为了试用多点的常见控件，我在最上方添加了Toolbar
```
relativeLayout {

            var mToolbar =
            toolbar(R.style.Base_ThemeOverlay_AppCompat_Dark_ActionBar) {
                id = ID_TOOLBAR
                title = "登录"
                backgroundColor = ContextCompat.getColor(this@MainActivity, R.color.colorPrimary)

                popupTheme = R.style.Base_ThemeOverlay_AppCompat_Light
                inflateMenu(R.menu.main)

                setNavigationIcon(R.mipmap.img_back_white)

                onMenuItemClick {
                    menuItem ->
                    val itemId = menuItem!!.itemId
                    when (itemId) {
                        R.id.menu_main -> {
                            toast(R.string.main_toast)
                        }
                    }
                    false
                    }

                lparams {
                    width = matchParent
                    height = wrapContent
                }

                setNavigationOnClickListener {
                    finish()
                }
            }

            var mUserEdit = editText {
                id = ID_USER_EDIT
                hint = "请输入同户名"
                maxLines=1

                lparams {
                    width = matchParent
                    height = wrapContent
                    margin=dip(8)
                    centerInParent()
                }
            }

            var mPsdEdit = editText {
                id= ID_PSD_EDIT
                hint="请输入密码"
                maxLines=1
                maxWidth = 16

                lparams {
                    width = matchParent
                    height = wrapContent
                    margin = dip(8)
                    below(ID_USER_EDIT)
                }
            }


            var mButton = button("登录") {
                id= ID_BTN_LOGIN

                onClick {
                    var username = mUserEdit.text.toString()
                    var password = mPsdEdit.text.toString()

                    mPresenter!!.login(username,password)
                }

                lparams {
                    width= matchParent
                    height = wrapContent
                    margin = dip(8)
                    below(ID_PSD_EDIT)
                }
            }
        }
```
代码中的id使用了常量，在Kotlin中没有static的概念，但是有一个companion object(伴随对象)可以模拟实现类似static的功能。
```
companion object static {
        val ID_TOOLBAR: Int = 1
        val ID_USER_EDIT: Int = 2
        val ID_PSD_EDIT: Int = 3
        val ID_BTN_LOGIN: Int = 4
    }
```
可以看到，这样编写UI的代码非常的简洁。而且可读性非常的高。相信对XML写布局比较熟悉的同学都能看懂这里面代码的含义。
比如一个输入框
在xml中可能是
```
<EditText
    android:id="@+id/user_edit"
    android:width="match_parent"
    android:height="wrap_content"
    android:hint="请输入用户名"
    android:xxx="..."
    .../>
```
然后在java代码中通过findViewById获取，非常的麻烦
但是通过anko库，就可以实现如下写法：
```
var mUserEdit = editText {
                id = ID_USER_EDIT
                hint = "请输入同户名"
                maxLines=1

                lparams {
                    width = matchParent
                    height = wrapContent
                    margin=dip(8)
                    centerInParent()
                }
            }
```
非常的简洁和高效，同时官方还出了一个Android Stduio插件，叫做anko SDL preview，不过这个插件我用不了，暂时还不支持AS2.2之后版本。关于更多的anko知识可以参考anko的github

接下来编写BaseView和BasePresenter接口
```
interface BaseView<T> {
    fun setPresenter(presenter: T)
}
```
```
interface BasePresenter
```
使用Kotlin编写MainContract接口，这个接口建立起了V层和P层的通信
```
interface MainContract {
    interface View : BaseView<Presenter> {
        fun login()
        fun loginNUll()
    }

    interface Presenter : BasePresenter {
        fun login(username: String, password: String)
    }
}
```
使用Java编写P层代码
```
public class MainPresenter implements MainContract.Presenter {

    private MainContract.View mView;

    public MainPresenter(MainContract.View view) {
        mView = view;
        mView.setPresenter(this);
    }

    @Override
    public void login(String username, String password) {
        if (TextUtils.isEmpty(username) || TextUtils.isEmpty(password)) {
            mView.loginNUll();
            return;
        }
        mView.login();
    }
}
```
这里只是模拟了下登录的逻辑，并没有去真的实现一个登录
回到MainActivity，在这里加入我们的代码
```
var mPresenter : MainContract.Presenter? = null
```
```
override fun setPresenter(presenter: MainContract.Presenter) {
    mPresenter = presenter!!
}
```
实现V层应该实现的回调方法：
```
override fun loginNUll() {
    toast("用户名密码不得为空")
}

override fun login() {
    toast("执行登录逻辑...")
}
```

这里，我们就写完了一个入门的使用Kotlin和anko开发Android的实例。在这我仅仅是完成了一次入门，后面还需要进行一部分才坑（虽然成本不高），但是真正的理解Kotlin和anko的很多特性和设计思想还需要很长一段时间的认真学习。