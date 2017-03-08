
---
title: Activity详解（一）——Activity的生命周期全分析
date: 2017-01-04 13:58:29
reward: true
tags: Android
---
Android的生命周期相信大家都很熟悉了，这里记录的是自己在学习开发过程中的一些经验之谈，希望以后自己忘记了能有个笔记翻一下。
### 1.Activity正常的生命周期
&ensp;&ensp;&ensp;&ensp; 我们平常所说的生命周期就是正常的生命周期，也是Activity典型的生命周期，从Activity创建到结束会调用如下方法：

&ensp;&ensp;&ensp;&ensp; （1）onCreate：在Activity创建的时候调用。我们新建一个Activity的时候就会看到这个方法，通常在里面初始化资源。

&ensp;&ensp;&ensp;&ensp; （2）onStart：onCreate方法调用完成之后就会调用onStart方法。此时Activity已经属于可见状态，只不过我们还看不到，因为此时Activity还处于后台。要看到Activity得等onResume方法执行完毕。

&ensp;&ensp;&ensp;&ensp; （3）onResume：该方法表示Activity已经被调到前台来了，不仅属于可见状态，我们肉眼还可以看到。要注意该方法和onStart方法的对比，onStart方法虽然也表示Activity处于可见状态，当时此时Activity还看不到，而onResume方法调用之后，Activity才算真正的可见。

&ensp;&ensp;&ensp;&ensp; （4）onPause：在用户点击返回键或者后台键时，Activity会调用该方法，表示Activity被挂起了，看不到了。

&ensp;&ensp;&ensp;&ensp; （5）onStop：在onPause方法执行完之后就会紧接着执行该方法，理论上是有可能在执行完onPause方法之后回到Activity而不执行onStop方法，不过这要求用户拥有极快的手速，所以一般不考虑这种情况，即调用完onPause之后马上调用onStop方法，而不考虑着两个方法之间还会有什么逻辑。onStop距离Activity真正被销毁只有一步之遥了，所以可以在这个方法里面做一些轻量级资源的回收保存的操作，但不能太耗时。

&ensp;&ensp;&ensp;&ensp; （6）onDestroy：调用完该方法之后Activity算是真正地被销毁了，所以需要在这个方法里面做一些资源释放操作，例如释放数据库连接等。

&ensp;&ensp;&ensp;&ensp; （7）onRestart：该方法是在调用onStop方法时，如果用户返回Activity，那么就会回调该方法，然后调用onStart，进入Activity的生命周期，而不会调用onCreate。





