1.1 Activity的生命周期全面分析
===
1.1.1 典型情况下的生命周期分析
---
onCreate 做一些初始化工作；  
onRestart 当前Activity从不可见重新变为可见状态时调用；   
onStart 表示Activity正在被启动，这是Activity已经可见了，但是还没有出现在前台；  
onResume 表示Activity显示到前台；  
onPause 表示Activity正在停止。此时可以做一些不太耗时的操作，会影响新Activity的显示，onPause必须先执行完，新Activity的onResume才会执行；  
onStop 表示Activity即将停止，可以做一些稍微重量级的回收工作；  
onDestroy 表示Activity即将被销毁，在这里可以做一些回收工作和最终资源的释放；  


Activity首次启动流程：  
onCreate -> onStart -> onResume  
Activity打开另一个Activity或回到Home：  
onPause -> onStop  
注意！若新的Activity采用了透明主题，则不会调用onStop。  
回到原Activity：
onRestart -> onStart ->onResume  
用户按back键：  
onPause -> onStop ->onDestroy  
Activity回收后再次打开，同首次启动流程  

onCreate对应onDestroy，只有一次调用，标识着Activity的启动和销毁；  
onStart对应onStop，不止一次调用；  
onResume对应onPause，不止一次调用；
1.1.2 异常情况下的生命周期分析
---
【情况一 系统配置发生改变导致Activity被杀死并重建】  
典型情况——横竖屏切换  
onSavedInstanceState -> onStop -> onDestroy  
onPause和onSavedInstanceState没有绝对的先后时序  
onStart -> onRestoreInstanceState  
Activity和View都有onSavedInstanceState和onRestoreInstanceState方法。  
保存数据——委托的思想：Activity委托Window去保存数据；Window再委托它上级容器保存数据。  
onCreate和onRestoreInstanceState均可获得保存的数据，但onCreate中需要判断是否为空。Google建议使用onRestoreInstanceState。  

【情况二 内存不足导致低优先级的Activity被杀死】  
Activity优先级顺序：前台Activity > 可见的非前台Activity > 后台Activity  
一些后台工作不是和脱离四大组件而独立运行在后台，这样进程很容易被杀死，因此，考虑使用Service。  

当系统配置发生改变，取消重建Activity的方法：在AndroidManifest.xml中相应Activity节点下添加configChanges属性，多个值和使用“|”连接。  
当minSdkVersion和targetSdkVerison其中一个大于13时，还需多添加screenSize防止重启。  
1.2 Activity的启动模式
===
1.2.1 Activity的LaunchMode
---
默认情况：多个Activity为“栈”结构，符合“后进先出”的原则；当栈中没有任何Activity时，系统将会回收这个栈。  
Activity的四种启动模式：standard、singleTop、singleTask、singleInstance。  

stadard：系统默认的启动模式，该模式需要一个Activity的实例来启动另一个Acitivity，新的Activity会进入启动它的Activity的栈中。  
注意！若使用非Activity类型的Context启动Activity，需要加上FLAG_ACTIVITY_NEW_TASK标记，此时会新建任务栈，模式为singleTask。  

singleTop：栈顶复用模式，如果要启动的Activity处于栈顶，则会调用onNewIntent，不会调用onCreate和onStart，栈结构保持不变。  

singleTask：栈内复用模式，如果要启动的Activity处于栈中，则会清除在其之上的所有Activity，将自己置于栈顶，回调onNewIntent方法；如果要启动的Activity不处于栈中，则要看所需的任务栈。若所需的任务栈不存在，则会新建任务栈，将其置于栈中；反之则会直接创建该Activity的实例，置于栈顶。  
singleTask默认具有clearTop效果。  

singleInstance：单实例模式，该模式的Activity只能单独位于一个任务栈中，后续的启动请求均不会创建新的Activity。  

特殊情况！当存在两个任务栈，其中一个为前台任务栈AB，另一个为后台singleTask模式的CD。当启动D时，整个栈将变为ABCD；当启动C时，整个栈将变为ABC。  

何为所需的任务栈？即TaskAffinity。默认情况下，所有的Activity所需的任务栈名字都是包名。也可为其命名单独的TaskAffinity属性名，该名称不能和包名重复。该属性主要用于配合singleTask或allowTaskReparenting属性使用，其他情况下没有意义。  
TaskAffinity和singleTask结合：具有该模式的Activity会进入指定名称的任务栈中；  
TaskAffinity和allowTaskReparenting结合：程序A启动程序B的Activity C，当allowTaskReparenting为true时，再次启动B，Activity C会从A的任务栈转移到B的任务栈中。  

设置Activity的启动方式有两种，分别是AndroidManifest.xml和Intent标志位。优先级上，后者优于前者。范围上各有限制。
1.2.2 Activity的Flags
---
1.2.3 IntentFilter的匹配规则
---
启动Activity有两种方式，显式和隐式。隐式启动需匹配IntentFilter。  
IntentFilter中的过滤信息有三种：action、category、data。需要同时匹配过滤列表中的三种类别才算完全匹配，只有完全匹配才能成功启动Activity。  
一个Activity可以有多个intent-filter组合，只需匹配其中一组即可成功启动指定的Activity。  

action匹配规则：action的匹配要求Intent中的action存在且必须和过滤规则中的其中一个action相同；  
category匹配规则：category匹配要求如果Intent中含有category，那么所有的category都必须和过滤规则中的其中一个category相同（可以为空）；  
data匹配规则：data匹配要求如果过滤规则中定义了data，那么Intent中必须也要定义可匹配的data。  

对于Service和BroadcastReceiver，也可采用隐式匹配。系统对于Service建议使用显式方式启动。  
为防止出现异常，通常使用queryIntentActivities或resolveActivity的方法来判断是否存在成功匹配的Activity。
