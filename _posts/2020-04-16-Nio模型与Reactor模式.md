---
title: "Nio模型与Reactor模式"
date: 2020-04-16
categories:
  - NIO模型
tags:
  - NIO模型
  - Reactor模式
header:
  teaser: /assets/images/NIO.jpg
  image: /assets/images/NIO.jpg
---
- Tomcat的NIO模型主要指的是接收Socket、读取Socket、写入Socket时所用的模型（IO线程模型）通过NIO模型服务器可以不用在读取和写入时阻塞了，如果Accept Socket就开新线程的话（BIO），会导致每个线程都在读取或者写入时阻塞，这时候不如把读取和写入Socket的操作在Web服务器层面通过NIO线程模型来统一解决；框架线程模型如Servlet或者Spring Webflux则是在处理请求时所用的模型（业务线程模型）
        
- 对Reactive和非阻塞好处的预期关键在于使用小,固定的线程数和更少的内存来扩展的能力.这使应用程序在加载的时候更加有弹性,因为他们以一种更可以预测的方式扩展.然而为了看到这些好处,需要一些延迟(包括比较慢的不可预知的网络I/O)的代价，开多个线程可以达到同非阻塞一样的效果，但是由于线程上下文切换的缘故，开多个线程的策略在垂直扩展上不能达到线性增长的要求，甚至可能反向下降

- Spring Web Flux 为我们实现了subscribe，所以我们只需要和Flux以及Mono打交道就可以了

- flatmap 要求返回一个Mono，因此肯定也是一个异步操作（比如在webclient调用以后再用webclient，从而用到了Compose）， map则是一个简单的函数调用不涉及异步，只是一个同步操作罢了

- Tomcat处理流程：在accept队列中接收连接（当客户端向服务器发送请求时，如果客户端与OS完成三次握手建立了连接，则OS将该连接放入accept队列）；在连接中获取请求的数据，生成request（这里有BIO和NIO的区别）；调用servlet容器处理请求（这里都是提交一个线程给线程池处理）；返回response（这里有BIO和NIO的区别）。

- Servlet业务线程模型通过使用比CPU核心数量多得多的线程数，使CPU忙碌起来，大大提高CPU的利用率，但也极大地增加了上下文切换的开销。

- Reactor模型就是通过高层Reactor分发（EventLoop）把原来单个线程负责的单个完整的处理流程（该流程可能存在阻塞的操作）划分成多个非阻塞的阶段（accpet->read->decode->process->encode->send，依然是单个请求流程由单个线程处理，但是变成了分阶段处理，所以单个线程可以同时处理多个请求流程了，例如可能先处理A请求的accept，然后处理B请求的read然后在回来处理A请求的read），在此基础上每个阶段之间的阻塞操作通过NIO以及事件驱动来唤醒下一个阶段，整体上就是分而治之的思想，每个阶段本身就是非阻塞的方式来执行，而阻塞操作如IO操作采用异步的形式并穿插在阶段和阶段之间并通过事件驱动的方式来唤醒下一个阶段，从而可以让线程可以不断地运行而无需浪费CPU资源在阻塞上（单线程也可以实现Reactor模型，如Python），Reactor模式使得线程数尽量可控，同时基于事件驱动和NonBlock可以尽量减少因为等待IO而浪费的CPU时间，尽可能提高了CPU的利用率，减少线程数，降低上下文切换带来的开销。

- 如果所有的操作都是同步操作的话，Reactor模型和传统BIO模型没有任何的区别甚至性能会差，但是当前很多IO操作，包括磁盘IO、网络IO都提供了基于DMA的异步操作，所以可以基于Reactor模型提供更好的CPU利用率，减少线程上下文切换带来的开销

- 因为TCP是基于流式数据的传输协议，而且Reactor模式将一次处理流程划分成了多个阶段，所以需要通过一个类来包装对应的Socket然后在该类中内置一个InputBuffer来缓存每次读到的信息，并通过自己实现的isComplete函数来判定是否已经读完了一次请求，这里需要注意的是，在Socket注册到Selector的时候，该类需要作为attachment一起被注册，可以根据不同的状态attach不同的类（GoF的State-Object模式）

- 如果业务本身就是阻塞的话，使用Spring Web Flux相较于Spring MVC可能会带来更大的延迟，因为Spring Web Flux一般只会使用固定数量的线程（同CPU数一致）来进行整个请求的处理，而一旦业务本身阻塞而线程数又不够，就会因为线程数不够导致排队的问题，而在相同情况下Spring MVC会通过开具有上百个线程数的线程池方式来处理业务，虽然会有线程上下文切换的代价，但是起码不用排队了呀，所以Spring Web Flux更多的可能用在一些不需要或者极少阻塞业务，如请求分发（Spring Cloud Gateway），Reactive 数据库调用， 中间件等业务场景