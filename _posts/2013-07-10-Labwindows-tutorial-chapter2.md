---
layout: post
title: "Labwindows.Tutorial.Chapter 2"
modified: 2013-07-18 5:00
comments: true
tags: [Tech, C, Labwindows Tutorial]
---

导语：本篇文章将向大家演示如何在**主Panel上**创建一个**Graph控件**(Object)和两个**按钮**，并为这两个按钮绑定事件回调函数：  

 - 点下**按钮1**，Graph控件显示Sine波形；  
 - 点下**按钮2**，Graph控件显示Cosine波形。

效果如下图所示：  

<!--more-->

![][1]

 [1]: http://ww3.sinaimg.cn/large/6480dca9tw1e66pudgy7aj20ej0l6abp.jpg

------

LabWindows/CVI为开发者提供了三种类型的图表控件，在界面编辑器下，他们的创建方式为：  
在主面板上右击，在弹出的控件列表中，依次选择

*   **Graph** >> **Graph** 即可在主面板上新建一个Graph 
*   **Graph** >> **Strip Chart** 即可在主面板上新建一个Strip Chart 
*   **Graph** >> **Digital Graph** 即可在主面板上新建一个Digital Graph 

这里简单**介绍一下** LabWindows/CVI提供的**三种Graph图表**的功能和特点：

*   Graph：一般用于显示一个或多个**静态**图表。图表可由**一个曲线**、 **一个点**、 **一个几何形状**、 **一张图片**或 **一个字符串**组成 
*   Strip Chart：一般用于**动态显示实时数据**，它可同时显示一路或多路信号。例如我的毕设是信号采集系统，采集到的信号都是实时变化的，这时候用Strip Chart是最好的选择 
*   Digital Graph：顾名思义，它用于显示**数字信号**，显示内容其实就是一系列的**方波** 

* * *

虽然上面介绍了三种图表控件，但鉴于本教程只为帮助初学者入门，所以此处只简单介绍三个图表控件中**Graph控件**的使用方法，至于其他两个图表控件的使用方法，可在掌握Graph用法后自行查找资料(官方的帮助文档及在线论坛)进行学习。

OK,here we go !

* * *

1) 打开LabWindows/CVI，新建一个**.uir**文件；

2) 双击创建出来的面板，在面板的**属性编辑框**里，设置主面板的标题(Panel title:)为：*Using Graph Demo*

3) 在面板上创建三个按钮(Command Button，创建方法见第一篇教程HelloWorld)  
双击**第一个**按钮，在弹出的按钮**属性编辑框**里，把**Callback function**的值设为 *showSineWave* | 把**Label**的值设为 *Sine*  
之后点击OK(确定)回到主面板  
按上述方法，设置**第二个**按钮的**Callback function**的值为 *showCosineWave* | **Label**的值为 *Cosine*  
接着设置**第三个**按钮的**Callback function**的值为 *quitSystem* | **Label**的值为 *Quit*

4) 在面板上创建一个Graph控件。具体方法是：对着面板右击，在弹出的控件列表里，依次选择**Graph** >> **Graph**(读者可用鼠标随意调整按钮和图表的位置及大小)  
目前为止，我们做出来的东西效果如下图所示  
![][2]

 [2]: http://ww2.sinaimg.cn/large/6480dca9tw1e66pscg411j20ec0a7wfb.jpg

5) 双击新建的Graph控件，在弹出的Graph**属性编辑框**里，在**Label Appearance**区域，设置**Label**的值为 *Graph Demo*  
之后点击OK(确定)回到主面板

* * *

好的，我们所有的界面编辑工作已经完成了，接下来，我们要做的就是编写程序，实现文章开头说到的那些功能了

1) 生成源代码文件，即**.c**文件  
操作步骤：依次点击系统菜单栏 **Code** >> **Generate** >> **All code**  
操作图示：  
![][3]  
此时**系统提示**需要首先保存**.uir**文件，将**.uir**文件命名为 *2-Using-Graph-Demo.uir*，接着新建文件夹**2**，把该**.uir**文件保存到新建的文件夹下  
(PS:为什么要新建文件夹并把.uir文件保存到这里？因为之后的**.c**代码文件以及包含此工程信息的**所有文件**后续都会保存到这个文件夹下，便于我们进行工程管理，也不会使电脑硬盘显得太乱)

 [3]: http://ww1.sinaimg.cn/large/6480dca9tw1e66pyn5o90j20cf09f75c.jpg

2) 接下来，系统又会提示需要指定一个**.c**的源文件，依次点**Yes**、 **OK**新建**.c**文件

3) 此时会弹出**Generate All Code**窗口，各项设置及意义如下图所示(可右键查看大图)，之后点击**OK**  
![][4]

 [4]: http://ww3.sinaimg.cn/large/6480dca9tw1e66qc4donfj20d20ep0uz.jpg

4) 在弹出的**New Project Options**窗口  
将**Project Location**设置为 *Create Project in Current Workspace*  
在**Transfer Project Options**区域，把四个复选框都勾选上(默认就是全部勾选)  
之后点击**OK**

5) 好吧，我们的源代码文件终于出现了。现贴出所有代码并注释:  

    #include <cvirte.h>;
    #include <userint.h>;
    #include "2-Using-Graph-Demo.h";

    /* panelHandle是主面板的Handle
    就相当于一个"对象索引";
    我们知道，主面板、按钮、图表控件等等界面元素，在系统内存中都是以
    一个内存块的形式存在的（在C++、Java里就是"对象"的概念）
    当我们需要使用这些对象时，就是通过这个&quot;索引&quot;来指向这些对象的 */
    static int panelHandle;

    /* main函数入口，也是系统总入口 */
    int main (int argc, char *argv[]) {
        if (InitCVIRTE (0, argv, 0) == 0)
            return -1;    /* out of memory */
        // LoadPanel()函数是LabWindows/CVI提供的系统函数
        // 如果想查看某个函数的介绍及参数等详情，可以把光标放到该函数上，然后按"F1"，
        // 这将会打开帮助文档中该函数的详细介绍
        if ((panelHandle = LoadPanel(0, "2-Using-Graph-Demo.uir", PANEL)) <= 0)
            return -1;
        // LoadPanel()之后，需要DisplayPanel()函数来真正把载入的面板数据显示出来
        DisplayPanel (panelHandle);
        RunUserInterface ();
        DiscardPanel (panelHandle);
        return 0;
    }

    /* Sine按钮的回调函数
    event事件包含了所有用户产生的事件，
    如键盘事件，鼠标事件(左键单双击、右键单双击等等)，获得焦点、失去焦点事件等等 */
    int CVICALLBACK showSineWave (int panel, int control, int event,
            void *callbackData, int eventData1, int eventData2) {
        switch (event) {
            case EVENT_COMMIT:

                break;
        }
        return 0;
    }  
    /* Cosine按钮的回调函数 */
    int CVICALLBACK showCosineWave (int panel, int control, int event,
            void *callbackData, int eventData1, int eventData2) {
        switch (event) {
            case EVENT_COMMIT:

                break;
        }
        return 0;
    }  
    /* Quit按钮的回调函数 */
    int CVICALLBACK quitSystem (int panel, int control, int event,
            void *callbackData, int eventData1, int eventData2) {
        switch (event) {
            case EVENT_COMMIT:
                //退出用户界面(退出按钮)
                QuitUserInterface (0);
                break;
        }
        return 0;
    }

6) 编写**Sine**按钮的回调函数  
函数实现为：点击**Sine**按钮后，清空Graph控件上的所有图表内容，然后使用系统提供的*SinePattern()*函数生成一组Sine曲线的Y坐标值，最后通过描出Y坐标值来在Graph图表上汇出一个**Sine**曲线  
下面贴上该回调函数具体代码并注释：  

    int CVICALLBACK showSineWave (int panel, int control, int event,
            void *callbackData, int eventData1, int eventData2) {
        switch(event){
            //case是各种用户事件，到此查看所有事件列表
            // http://t.cn/zQ7jw42
            case EVENT_COMMIT:
                //删除Graph上的某个或所有图像(Plot),可把光标放置到函数名上，按F1查看帮助
                DeleteGraphPlot(panelHandle, PANEL_GRAPH, -1, VAL_IMMEDIATE_DRAW);
                //waveData[]数组用来保存SinePattern函数生成的100个Y坐标值
                double waveData[100], amp, phase, cycles;
                int n;
                n = 100;
                amp = 90.0;
                phase = 0.0;
                cycles = 1.5;
                //该步将生成100个Sine函数的Y坐标点，并将这些值保存到waveData数组里
                SinePattern(n, amp, phase, cycles, waveData);
                //该步把Y坐标值绘制到Graph图表控件中去
                PlotY(panelHandle, PANEL_GRAPH, waveData, n, VAL_DOUBLE, VAL_THIN_LINE, 
                      VAL_EMPTY_SQUARE, VAL_SOLID, VAL_CONNECTED_POINTS , VAL_YELLOW);
                break;
        }
        return 0;
    }

7) 编写**Cosine**按钮的回调函数并注释  

    int CVICALLBACK showCosineWave(int panel, int control, int event,
            void *callbackData, int eventData1, int eventData2) {
        switch (event){
            case EVENT_COMMIT:
                DeleteGraphPlot(panelHandle, PANEL_GRAPH, -1, VAL_IMMEDIATE_DRAW);
                
                double waveData[100], amp, phase, cycles;
                int n;
                n = 100;
                amp = 90.0;
                //和Sine函数不同，此处相位应为90(度)
                phase = 90.0;
                cycles = 1.5;
                SinePattern(n, amp, phase, cycles, waveData);
                
                PlotY(panelHandle, PANEL_GRAPH, waveData, n, VAL_DOUBLE, VAL_THIN_LINE, 
                      VAL_EMPTY_SQUARE, VAL_SOLID, VAL_CONNECTED_POINTS , VAL_YELLOW);
                break;
        }
        return 0;
    }

------

好的，至此两个按钮的回调函数都已经完成。下面需要开始**试运行(Debug程序)**  
有两种方法**试运行**程序

*   通过工具栏的**红色播放按钮**试运行
*   通过菜单栏的**Run** >> **Debug 2-Using-Graph-Demo.exe**选项来试运行程序  
    具体操作如下图所示：  
    ![][5] 

 [5]: http://ww2.sinaimg.cn/large/6480dca9tw1e66qebhm8wj20b009ojsh.jpg

运行结果:  
![][1]

-------

程序源码[托管在Github][6]上，请自行前往查看。

 [6]: https://github.com/HelloLyfing/LabWindows-CVI-Tutorial-For-Newbie-By.Lyfing
