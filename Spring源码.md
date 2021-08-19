# Spring源码

循环引用的三个MAp

**SingleonObjects** 单例池  也就是我们常说的SpringIoc容器 该容器生产的是spring bean  一级缓存

**SingleOnFactories** 单例工厂 该Map生产的是对象  二级缓存

**earlySingleOnObjects** 三级缓存  早期提前曝光的单例对象

this() register refersh 