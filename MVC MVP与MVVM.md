# MVC MVP与MVVM

## MVC 

Model-View-Controller，模型-视图-控制器，是一种典型的三层软件体系架构，在这种分层的设计思想下，软件展示界面与逻辑分离，可以极大地提高代码的可读性与可维护性。

- M：model， 模型，负责数据处理，包括网络数据和持久化数据的获取、加工等，在Android中典型的实现一般为数据结构的定义类；
- V：view，视图，负责界面绘制、展示数据以及用户交互等，在Android中典型的实现一般是activity和fragment等；
- C：controler，控制器，负责处理业务逻辑等。

在标准的MVC架构中，Controller和View都依赖于Model。MVC架构通过Controller来更新数据，通过View展示数据。

根据Model（模型）层的被动与主动，可将MVC分为被动模式和主动模式。被动模式下，Model不会主动将变化通知到View进行更新，而是由Controller通知View，Model变化了需要进行更新，而且只有Controller可以对Model进行更新。主动模式下，Model的修改会通知View更新，主动模式利用了观察者模式，View作为观察者，Model为被观察者。

<img src="C:\Users\27253\AppData\Roaming\Typora\typora-user-images\image-20210508174516067.png" alt="image-20210508174516067" style="zoom:80%;" />

被动模式与主动模式相比，它的缺点主要是View无法感知Model的变化，而需要通过Controller来间接地通知更新。但同时它也是一个优点，Controller可以更好地控制View的更新，而不必担心Model主动通知变化而产生同步问题。

MVC的核心思想是表现层分离，即将领域模型和视图模型分离。在MVC中，领域模型是Model，视图模型是View与Controller协作的整体。Controller不仅负责操作Model进行业务逻辑处理，还负责进行View相关的界面展示处理，其中包括View的处理细节，因此，我们可以说Controller肩负着过重的责任，只适用于小型的app，应用于大型app时，会出现Massive View Controller（过重的视图控制器）的问题。 

## MVP

Model-View-Presenter 模型-视图-主持人，三层架构模式。

- Mode层：模型层，负责数据处理，包括网络数据和持久化数据的获取、加工等，在Android中典型的实现时数据结构的定义类，与MVC中Model类似。
- View层：视图层，负责处理界面绘制，向用户展示Model数据，在Android中典型实现为Activity/Fragment等。
- Presenter层：主持人层，模型和视图的“中介“。视图层将用户交互响应事件传递给Presenter，Presenter进行数据处理，并通知View进行数据展示等操作。

在MVP模式中，Presenter层可以依赖View层和Model层，充当沟通的桥梁。View层的事件传递流经过Presenter层操作Model进行业务逻辑处理。Model层数据处理完成会通知Presenter层，Presenter层再与View层进行沟通。Presenter层与View、Model之间是双向数据流，而View与Model之间没有直接联系。MVP模型如下图所示：

<img src="C:\Users\27253\AppData\Roaming\Typora\typora-user-images\image-20210510182059436.png" alt="image-20210510182059436" style="zoom:50%;" />

MVP的核心思想，除了有与MVC相同的表现层分离外，还有面向接口编程和德莫忒尔定律。

面向接口编程是将类中的方法提炼出来成为接口，由实现类实现该接口，而调用方通过接口与实现类进行交互。面向接口编程的优点：

- 易于解耦，使组件之间减少依赖
- 有利于提升模块扩展性
- 有利于提升代码可维护性
- 使程序结构更加清晰，增强代码可读性

在Android的MVP开发中，经常使用一种协议类-Contract，将View与Presenter之间的接口建立协议关联关系。Contract接口中包含View和Presenter接口。

德莫忒尔定律其实就是“高内聚低耦合”的设计思想。

MVP与MVC的区别主要在于“C”与“P”的区别和各自产生的事件流。

 - 在MVP模式中，Presenter只通知View进行展示，View展示逻辑由View自己控制，符合德莫忒尔定律；而在MVC模式中，由Controller控制View的展示逻辑，不符合德莫忒尔定律。
 - MVP为面向接口编程，Presenter与View的通信通过接口；而在MVC模式中Controller直接操作View。
 - 在MVP模式中，View与Model之间无法进行通信，而在MVC的主动模式中，Model的变化可以直接通知给View。
 - MVP更有利于单元测试，而MVC组件之间依赖性较强，不利于单元测试。

MVP存在的问题：

 - Presenter责任过重：相比MVC，MVP模式中Presenter交出View的实现细节控制权，而只负责业务逻辑，但责任依然过重；
 - 业务逻辑无法服用：传统MVP模式中，一个View对应一个Presenter，Presenter中业务逻辑只能为一个View服务，不能复用于多个模块之间，不同模块为了实现同样的业务逻辑，会多次重复操作导致大量冗余代码；
	- 急剧扩增的接口数量：MVP采用面向接口编程，在进行业务迭代时，因发生变化而修改既有代码，往往需要连同修改接口层，而新增开放方法需要声明在接口中。

## MVVM

Model-View-ViewModel 模型-视图-视图模型。

 - Model层：模型层，负责数据处理，包括网络数据和持久化数据的获取、加工等，在Android中的典型实现一般为数据结构的定义类，与MVCHE MVP中的Model层类似；
 - View层：视图层，负责处理界面绘制，向用户展示Model数据，在Android中的典型实现一般为Activity/Fragment等；
 - ViewModel层：视图模型层，负责处理业务逻辑相关的数据操作，不会与View产生依赖关系，在Android中，可以通过官方提供的DataBinding库实现视图层和模型层数据的双向绑定而不需要ViewModel层通知View层相关的数据变化。

MVVM架构模型图如下：

<img src="C:\Users\27253\AppData\Roaming\Typora\typora-user-images\image-20210511153844923.png" alt="image-20210511153844923" style="zoom:80%;" />

MVVM的核心思想：

 - 进一步解耦：无论是MVC的Controller，还是MVP的Presenter，它们都会与View发生直接或者间接的依赖关系，通过Controller/Presenter来协调View和Model之间的交互，虽然MVP使用面向接口编程间接地将View与Model解耦，但并没有解决本质问题，而且还导致了接口数量和复杂度的增长。在MVVM中，ViewModel不会再持有View的引用，而是通过双向绑定机制来实现数据变化后的视图更新，它不再关心数据应该何时或者如何展示在视图上，一切都由双向绑定机制来完成，所以，MVVM在解耦程度上更进了一步。
 - 基于观察者模式的数据驱动：数据驱动编程是编程范式中的一种，它关注数据的变化，从数据中引发其他组件的变化，是一种基于事件的编程，本质是观察者模式。数据驱动可以阻止数据与功能之间产生耦合，也可以在一定程度上提高开发效率。
 - 双向绑定：双向绑定是数据驱动的一种很好的表现形式，即通过观察者模式，实现View的变化能实时反馈到数据上，数据的变化也可以实时反馈到View上。双向绑定是在单向绑定的基础上，对View增加了事件状态改变的监听，通过监听来动态修改数据。只有View有双向绑定，非View的部分一般只存在单向绑定。单向绑定在代码追踪和事件追踪上比双向绑定更加有优势，而双向绑定具有解耦良好、开发效率高等优势。

MVVM存在的问题： 

 - ViewModel难以复用：ViewModel与View虽然没有直接的依赖关系，但ViewModel是为特定的View进行数据服务的，复用ViewModel会给其他View带来更多组件兼容问题。当不同的ViewModel之间存在相同的业务逻辑时，会出现重复代码，这和Presenter复用一样棘手。
 - 学习成本高：需要了解双向绑定机制和观察者设计模式
 - 调试困难：由于双向绑定机制，界面显示异常时不能很快定位是View的问题还是Data的问题。

## 架构模式对比

| MVC优点                              | MVC缺点                          | MVVM优点                   | MVVM缺点                     |
| ------------------------------------ | -------------------------------- | -------------------------- | ---------------------------- |
| Controller可以直接操作View，更加灵活 | 架构组件耦合性大于MVVM架构       | 一旦掌握MVVM，开发效率更高 | 在小型项目上会增加系统复杂度 |
| 在调试跟踪上，MVC更加只管            | Controller承担责任过重，更难维护 | 解耦更加良好               | 不易于理解                   |
| 在架构理解上，MVC更易于理解          |                                  |                            | 更难以调试                   |



| MVP优点                                    | MVP缺点                                          | MVVM优点                                       | MVVM缺点              |
| ------------------------------------------ | ------------------------------------------------ | ---------------------------------------------- | --------------------- |
| 面向接口设计，修改更加直观清晰             | 比MVVM架构具有更多的接口，需要更高的接口维护成本 | 不需要ViewModel与View频繁交互                  | 学习成本更高          |
| 更利于测试，UI与业务逻辑分离，通过接口交互 | Presenter需要处理与View的交互，复杂度高          | 没有更多需要维护的接口                         | XML定义的代码更难维护 |
| 学习成本更低                               |                                                  | 处理业务逻辑的ViewModel的复杂度比Presenter更低 | 更难以调试            |



总结：没有最好的框架，只有最适合的框架。

