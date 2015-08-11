---
layout: default
title: 使用DDS來開發ROS
permalink: articles/ros_on_dds.html
abstract:
  這篇文章討論如何使用DDS來開發ROS 2.0，列出這種作法的優缺點，並討論使用者經驗和API帶來的影響。我們也會針對我們實作的prototype "ros_dds"給一些總結意見，並探討其中潛藏的問題。
author: '[William Woodall](https://github.com/wjwwood)'
published: true
translator: 賴柏任
---

* This will become a table of contents (this text will be scraped).
{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Original Author: {{ page.author }}

Translator: {{ page.translator }}

專業術語:

- [Data Distribution Service (DDS)](http://en.wikipedia.org/wiki/Data_Distribution_Service)
- The [Object Management Group (OMG)](http://www.omg.org/)
- OMG [Interface Description Language (IDL)](http://www.omg.org/gettingstarted/omg_idl.htm)
  | [Formal description](http://www.omg.org/cgi-bin/doc?formal/2014-03-01)


## 為什麼考慮DDS?

當我們在探索可以用於開發下一代ROS的通訊系統的選項時，最先考慮的是改進ROS 1.x的傳輸機制或是使用現有的函式庫來建構(例如使用ZeroMQ、Protocol Buffer和zeroconf)。但是，若採取上述這些選項，我們要不是得從頭打造整個中介軟體、不然就是要從各個部分開始組合，所以我們也考慮直接使用功能完整的其他中介軟體來打造ROS 2.0。經過我們的調查之後，其中一個脫穎而出的選項就是DDS。

### 一個完整的中介軟體

使用一個像DDS這種完整的中介軟體來開發的好處是，我們需要維護的程式碼少了很多，而且這種中介軟體的行為和詳細規格都已經被詳細地撰寫成文件。除了系統層級的說明文件之外，DDS也有推薦的使用個案和軟體API。有了這些具體的說明文件和規格，第三方的開發者就可以檢驗、審視，甚至可以根據自己的需求來實作具備不同程度的互動性的版本。這項特色是ROS從未有過的(ROS只有少量的基礎說明文件和一些可供參考的實作版本)，而且，就算使用現有的一些函式庫來建構起ROS，對於規格和軟體行為的詳細說明文件還是不可或缺的。

當然，這種開發方式也有缺點，就是ROS 2.0也會被DDS的設計所限制。如果DDS的設計不夠用於解決我們想要的使用情境或是不夠彈性，我們就必須去更動DDS的設計。某種程度上來說，採用完整的中介軟體來開發也包含了要採用這個中介軟體的開發哲學和社群文化，這件事也不應被輕忽。

## DDS是什麼?

DDS提供跟ROS 1.x很相似的一種訂閱-發佈傳輸機制。它使用[Object Management Group (OMG)](http://www.omg.org/)定義的“Interface Description Language (IDL)”來實作訊息的定義和序列化。雖然DDS還沒有提供request-response的傳輸機制(這可以被當作ROS的service系統)，但這種傳輸方式的實作已經有初步規劃，第一個Beta版也已經在2015年春天發佈(稱作 [DDS-RPC](http://www.omg.org/spec/DDS-RPC/))。

DDS提供的預設discovery系統是一種分散式的discovery系統(這有別於ROS 1.x必須由master來掌管所有node的方式)，其中也使用DDS的發佈-訂閱傳輸機制。這使得任意兩個DDS程式可以不透過ROS master這種機制就能相互溝通。這使整個系統的容錯程度更高、也更彈性。而且，我們不一定要使用動態的discovery機制，有幾種DDS的實作版本都提供了靜態discovery機制的選項。

### DDS是怎麼出現的?

DDS起源自一群擁有提供相似中介軟體的公司，因為這些公司的客戶希望這些公司提供的中介軟體可以彼此溝通，所以這些中介軟體漸漸地被整合起來形成一套標準。DDS的標準是由Object Management Group建立起來的，跟建立UML、CORBA和SysML和其他軟體標準的是同一群人。這可能是好消息也可能是壞消息，看你怎麼想。一方面，你擁有一個標準制定委員會，他們會不斷討論而且在軟體工程社群擁有極大的影響力；但另一方面，他們前進地相對緩慢、對於改變的適應力也較差，所以也很可能無法跟上軟體工程界的最新潮流。

DDS一開始是幾個相似的中介軟體，後來因為彼此變得太相近，制定一個統一的標準來整合這些中介軟體才變得合理。在這樣的脈絡之下，雖然DDS規格的制定是由委員會撰寫出來的，但他的演進原本就相當貼近使用者的需求。由於在標準制定之前就已經是藉由跟使用者的互動來演進，這多少可以消弭大家對於DDS的擔心，擔心它只是一個憑空想像出來的標準、但不符合實際應用的需求。歷史上也曾有一些標準委員會所制訂出的標準，雖然立意良善且制訂得很完整，但沒人想要使用或是不符合使用者需求。不過DDS看起來沒有這種問題。

除了上述的擔憂，也有人擔心DDS將會是一個不更新的標準，這種刻板印象來自於UML和CORBA給人的不良回憶，因為這兩者也都是OMG所制定的標準。不過DDS看起來不會有這種問題，除了最近仍持續在更新之外，現在也還在新增更多的規格，包含websockets、SSL安全協議、可擴充的型別、request-response傳輸機制以及C++11風格的API(用來取代現在的C++介面)。對我們來說，DDS的標準仍持續地演進是一件很正面的事，雖然跟現今軟體工程的潮流比起來仍相對緩慢，但它依然在往滿足其使用者需求的方向在前進。

### DDS技術上的可信度

有許多要求高精密度的系統都已經在使用DDS，它已經被用在:

- 軍艦
- 大型公共建設(例如水壩)
- 金融系統
- 航太系統
- 飛行系統
- 火車控制系統

除了這些使用情境之外，還有許多同等重要的不同系統都已經在使用DDS，這些成功的案例使得DDS設計的穩定度和彈性更加被信任。

DDS不僅滿足了這些使用情境下的需求，我們進一步訪問了政府及NASA的使用者(其中有些人也是ROS的使用者)，他們對於DDS的穩定性和彈性都讚譽有加。不過他們也提醒，DDS具備高彈性的代價就是高複雜度，它在API以及設定上的複雜度是我們應用DDS來開發ROS時需要注意的問題。

The DDS wire specification (DDSI-RTPS) is extremely flexible, allowing it to be used for reliable, high level systems integration as well as real-time on embedded devices.
Several of the DDS vendors have special implementations of DDS for embedded systems which boast specs related to library size and memory footprint on the scale of tens or hundreds of kilobytes.
Since DDS is implemented, by default, on UDP, it does not depend on a reliable transport or hardware for communication.
This means that DDS has to reinvent the reliability wheel (basically TCP plus or minus some features), but in exchange DDS gains portability and control over the behavior.
Control over several parameters of reliability, what DDS calls Quality of Service (QoS), gives maximum flexibility in controlling the behavior of communication.
For example, if you are concerned about latency, like for soft real-time, you can basically tune DDS to be just a UDP blaster.
In another scenario you might need something that behaves like TCP, but needs to be more tolerant to long dropouts, and with DDS all of these things can be controlled by changing the QoS parameters.

Though the default implementation of DDS is over UDP, and only requires that level of functionality from the transport, OMG also added support for DDS over TCP in version 1.2 of their specification.
Only looking briefly, two of the vendors (RTI and PrismTech) both support DDS over TCP.

From RTI's website ([http://community.rti.com/kb/xml-qos-example-using-rti-connext-dds-tcp-transport](http://community.rti.com/kb/xml-qos-example-using-rti-connext-dds-tcp-transport)):

> By default, RTI Connext DDS uses the UDPv4 and Shared Memory transport to communicate with other DDS applications.
> In some circumstances, the TCP protocol might be needed for discovery and data exchange.
> For more information on the RTI TCP Transport, please refer to the section in the RTI Core Libraries and Utilities User Manual titled "RTI TCP Transport".

From PrismTech's spec sheet, they support TCP as of DDSI-RTPS version 1.2:

[http://www.prismtech.com/opensplice/products/opensplice-enterprise/opensplice-dds-core](http://www.prismtech.com/opensplice/products/opensplice-enterprise/opensplice-dds-core) (search for TCP)


### Vendors and Licensing

The OMG defined the DDS specification with several companies which are now the main DDS vendors.
Popular DDS vendors include:

- RTI
- PrismTech
- Twin Oaks Software

Amongst these vendors is an array of reference implementations with different strategies and licenses.
The OMG maintains an active [list](http://dds-directory.omg.org/vendor/list.htm) of DDS vendors.

RTI's Connext DDS is available under a custom "Community Infrastructure" License, which is compatible with the ROS community's needs but requires further discussion with the community in order to determine its viability as the default DDS vendor for ROS.
By, "compatible with the ROS community's needs," we mean that, though it is not an OSI-approved license, research has shown it to be adequately permissive to allow ROS to keep a BSD style license and for anyone in the ROS community to redistribute it in source or binary form.
RTI also appears to be willing to negotiate on the license to meet the ROS community's needs, but it will take some iteration between the ROS community and RTI to make sure this would work.
Like the other vendors this license is available for the core set of functionality, basically the basic DDS API, whereas other parts of their product like development and introspection tools are proprietary.
RTI seems to have the largest on-line presence and installation base.

PrismTech's DDS implementation, OpenSplice, is licensed under the LGPL, which is the same license used by many popular open source libraries, like glibc, ZeroMQ, and Qt.
It is available on [Github](https://github.com):

[https://github.com/PrismTech/opensplice](https://github.com/PrismTech/opensplice)

PrismTech's implementation comes with a basic, functioning build system and was fairly easy to package.
OpenSplice appears to be the number two DDS implementation in use, but that is hard to tell for sure.

TwinOaks's CoreDX DDS implementation is proprietary only, but apparently they specialize in minimal implementations which are able to run on embedded devices and even bare metal.

Given the relatively strong LGPL option and the encouraging but custom license from RTI, it seems that depending on and even distributing DDS as a dependency should be straight forward.
One of the goals of this proposal would be to make ROS 2.0 DDS vendor agnostic.
So, just as an example, if the default implementation is RTI, but someone wants to use OpenSplice, they simply need to recompile the ROS source code with some options flipped and they could use the OpenSplice implementation.

![DDS and ROS API Layout](/img/ros_on_dds/api_levels.png "DDS and ROS API Layout")

This is made possible because of the fact that DDS defines an API in its specification.
Research has shown that making code which is vendor agnostic is possible if not a little painful since the API's of the different vendors is almost identical, but there are minor differences like return types (pointer versus shared_ptr like thing) and header file organization.


### Ethos and Community

DDS comes out of a set of companies which are decades old, was laid out by the OMG which is an old-school software engineering organization, and is used largely by government and military users.
So it comes as no surprise that the community for DDS looks very different from the ROS community and similar modern software projects like ZeroMQ.
Though RTI has a respectable on-line presense, the questions asked by community members are almost always answered by an employee of RTI and though technically open source, neither RTI nor OpenSplice have spent time to provide packages for Ubuntu or Homebrew or any other modern package manager.
They do not have extensive user contributed wiki's or an active Github repository.

This staunch difference in ethos between the communities is one of the most concerning issues with depending on DDS.
Unlike options like keeping rostcp or using ZeroMQ, there isn't the feeling that there is a large community to fall back on with DDS.
However, the DDS vendors have been very responsive to our inquires during our research and it is hard to say if that will continue when it is the ROS community which brings the questions.

Even though this is something which should be taken under consideration when making a decision about using DDS, it should not disproportionately outweigh the technical pros and cons of the DDS proposal.


## ROS built on DDS

The goal is to make DDS an implementation detail of ROS 2.0.
This means that all DDS specific API's and message definitions would need to be hidden.
DDS provides discovery, message definition, message serialization, and publish-subscribe transport.
Therefore, DDS would provide discovery, publish-subscribe transport, and at least the underlying message serialization for ROS.
ROS 2.0 would provide a ROS 1.x like interface on top of DDS which hides much of the complexity of DDS for the majority of ROS users, but then separately provides access to the underlying DDS implementation for users that have extreme use cases or need to integrate with other, existing DDS systems.
Accessing the DDS implementation would require depending on an additional package which is not normally used.
In this way you can tell if a package has tied itself to a particular DDS vendor by just looking at the package dependencies.
The goal of the ROS API, which is on top of DDS, should be to meet all the common needs for the ROS community, because once a user taps into the underlying DDS system, they will lose portability between DDS vendors.
Portability among DDS vendors is not intended to encourage people to frequently choose different vendors, but rather to enable power users to select the DDS implementation that meets their specific requirements, as well as to future-proof ROS against changes in the DDS vendor options.
There will be one recommended and best-supported default DDS implementation for ROS.


### Discovery

DDS would completely replace the ROS master based discovery system.
ROS would need to tap into the DDS API to get information like a list of all nodes, a list of all topics, and how they are connected.
Accessing this information would be hidden behind a ROS defined API, preventing the users from having to call into DDS directly.

The advantage of the DDS discovery system is that, by default, it is completely distributed, so there is no central point of failure which is required for parts of the system to communicate with each other.
DDS also allows for user defined meta data in their discovery system, which will enable ROS to piggyback higher level concepts onto publish-subscribe.


### Publish-Subscribe Transport

The DDSI-RTPS (DDS-Interoperability Real Time Publish Subscribe) protocol would replace ROS's rostcp and rosudp wire protocols for publish/subscribe.
The DDS API provides a few more actors to the typical publish-subscribe pattern of ROS 1.x.
In ROS the concept of a node is most clearly paralleled to a graph participant in DDS.
A graph participant can have zero to many topics, which are very similar to the concept of topics in ROS, but are represented as separate code objects in DDS, and is neither a subscriber nor a publisher.
Then, from a DDS topic, DDS subscribers and publishers can be created, but again these are used to represent the subscriber and publisher concepts in DDS, and not to directly read data from or write data to the topic.
DDS has, in addition to the topics, subscribers, and publishers, the concept of DataReaders and DataWriters which are created with a subscriber or publisher and then specialized to a particular message type before being used to read and write data for a topic.
These additional layers of abstraction allow DDS to have a high level of configuration, because you can set QoS settings at each level of the publish-subscribe stack, providing the highest granularity of configuration possible.
Most of these levels of abstractions are not necessary to meet the current needs of ROS.
Therefore, packaging common work flows under the simpler ROS-like interface (Node, Publisher, and Subscriber) will be one way ROS 2.0 can hide the complexity of DDS, while exposing some of its features.


### Efficient Transport Alternatives

In ROS 1.x there was never a standard shared-memory transport because it is negligibly faster than localhost TCP loop-back connections.
It is possible to get non-trivial performance improvements from carefully doing zero-copy style shared-memory between processes, but anytime a task required faster than localhost TCP in ROS 1.x, nodelets were used.
Nodelets allow publishers and subscribers to share data by passing around `boost::shared_ptr`'s to messages.
This intraprocess communication is almost certainly faster than any interprocess communication options and is orthogonal to the discussion of the network publish-subscribe implementation.

In the context of DDS, most vendors will optimize message traffic (even between processes) using shared-memory in a transparent way, only using the wire protocol and UDP sockets when leaving the localhost.
This provides a considerable performance increase for DDS, whereas it did not for ROS 1.x, because the localhost networking optimization happens at the call to `send`.
For ROS 1.x the process was: serialize the message into one large buffer, call TCP's `send` on the buffer once.
For DDS the process would be more like: serialize the message, break the message into potentially many UDP packets, call UDP's `send` many times.
In this way sending many UDP datagrams does not benefit from the same speed up as one large TCP `send`.
Therefore, many DDS vendors will short circuit this process for localhost messages and use a blackboard style shared-memory mechanism to communicate efficiently between processes.

However, not all DDS vendors are the same in this respect, so ROS would not rely on this "intelligent" behavior for efficient **intra**process communication.
Additionally, if the ROS message format is kept, which is discussed in the next section, it would not be possible to prevent a conversion to the DDS message type for intraprocess topics.
Therefore a custom intraprocess communication system would need to be developed for ROS which would never serialize nor convert messages, but instead would pass pointers (to shared in process memory) between publishers and subscribers using DDS topics.
This same intraprocess communication mechanism would be needed for a custom middleware built on ZeroMQ, for example.

The point to take away here is that efficient **intra**process communication will be addressed regardless of the network/interprocess implementation of the middleware.


### Messages

There is a great deal of value in the current ROS message definitions.
The format is simple, and the messages themselves have evolved over years of use by the robotics community.
Much of the semantic contents of current ROS code is driven by the structure and contents of these messages, so preserving the format and in memory representation of the messages has a great deal of value.
In order to meet this goal, and in order to make DDS an implementation detail, ROS 2.0 should preserve the ROS 1.x like message definitions and in memory representation.

Therefore, the ROS 1.x `.msg` files would continue to be used and the `.msg` files would be converted into `.idl` files so that they could be used with the DDS transport.
Language specific files would be generated for both the `.msg` files and the `.idl` files as well as conversion functions for converting between ROS and DDS in memory instances.
The ROS 2.0 API would work exclusively with the `.msg` style message objects in memory and would convert them to `.idl` objects before publishing.

![Message Generation Diagram](/img/ros_on_dds/message_generation.png "Message Generation Diagram")

At first, the idea of converting a message field by field into another object type for each call to publish seems like a huge performance problem, but experimentation has shown that the cost of this copy is insignificant when compared to the cost of serialization.
This ratio, which was found to be at least one order of magnitude, between the cost of converting types and the cost of serialization holds true with every serialization library that we tried, except [Cap'n Proto](http://kentonv.github.io/capnproto/) which doesn't have serialization step.
Therefore, if a field by field copy will not work for your use case, neither will serializing and transporting over the network, at which point you will have to utilize an intra-process or zero-copy interprocess communication.
The intra-process communication in ROS would not use the DDS in memory representation so this field by field copy would not be used unless the data is going to the wire.
Because this conversion is only invoked in conjunction with a more expensive serialization step, the field by field copy seems to be a reasonable trade-off for the portability and abstraction provided by preserving the ROS `.msg` files and in-memory representation.

This does not preclude the option to improve the `.msg` file format with things like default values and optional fields.
But this is a different trade-off which can be decided later.


### Services and Actions

DDS currently does not have a ratified or implemented standard for request-response style RPC which could be used to implement the concept of services in ROS.
There is currently an RPC specification being considered for ratification in the OMG DDS working group, and several of the DDS vendors have a draft implementation of the RPC API.
It is not clear, however, whether this standard will work for actions, but it could at least support non-preemptable version of ROS services.
ROS 2.0 could either implement services and actions on top of publish-subscribe (this is more feasible in DDS because of their reliable publish-subscribe QoS setting) or it could use the DDS RPC specification once it is finished for services and then build actions on top, again like it is in ROS 1.x.
Either way actions will be a first class citizen in the ROS 2.0 API and it may be the case that services just become a degenerate case of actions.


### Language Support

DDS vendors typically provide at least C, C++, and Java implementations since API's for those languages are explicitly defined by the DDS specification.
There are not any well established versions of DDS for Python that research has uncovered.
Therefore, one goal of the ROS 2.0 system will be to provide a first-class, feature complete C API.
This will allow bindings for other languages to be made more easily and to enable more consistent behavior between client libraries, since they will use the same implementation.
Languages like Python, Ruby, and Lisp can wrap the C API in a thin, language idiomatic implementation.

The actual implementation of ROS can either be in C, using the C DDS API, or in C++ using the DDS C++ API and then wrapping the C++ implementation in a C API for other languages.
Implementing in C++ and wrapping in C is a common pattern, for example [ZeroMQ](http://zeromq.org/) does exactly this.
The author of [ZeroMQ](http://zeromq.org/), however, did not do this in his new library, [nanomsg](http://nanomsg.org/), citing increased complexity and the bloat of the C++ stdlib as a dependency.
Since the C implementation of DDS is typically pure C, it would be possible to have a pure C implementation for the ROS C API all the way down through the DDS implementation.
However, writing the entire system in C might not be the first goal, and in the interest of getting a minimal viable product working, the implementation might be in C++ and wrapped in C to begin with and later the C++ can be replaced with C if it seems necessary.


### DDS as a Dependency

One of the goals of ROS 2.0 is to reuse as much code as possible (do not reinvent the wheel) but also minimize the number of dependencies to improve portability and to keep the build dependency list lean.
These two goals are sometimes at odds, since it is often the choice between implementing something internally or relying on an outside source (dependency) for the implementation.

This is a point where the DDS implementations shine, because two of the three DDS vendors under evaluation build on Linux, OS X, Windows, and other more exotic systems with no external dependencies.
The C implementation relies only on the system libraries, the C++ implementations only rely on a C++03 compiler, and the Java implementation only needs a JVM and the Java standard library.
Bundled as a binary (during prototyping) on both Ubuntu and OS X, the C, C++, Java, and C# implementations of OpenSplice (LGPL) is less than three megabytes in size and has no other dependencies.
As dependencies go, this makes DDS very attractive because it significantly simplifies the build and run dependencies for ROS.
Additionally, since the goal is to make DDS an implementation detail, it can probably be removed as transitive run dependency, meaning that it will not even need to be installed on a deployed system.


## The ROS on DDS Prototype

Following the research into the feasibility of ROS on DDS, several questions were left, including but not limited to:

- Can the ROS 1.x API and behavior be implemented on top of DDS?
- Is it practical to generate IDL messages from MSG messages and use them with DDS?
- How hard is it to package (as a dependency) DDS implementations?
- Does the DDS API specification actually make DDS vendor portability a reality?
- How difficult is it to configure DDS?

And other questions.
In order to answer some of these questions a prototype and several experiments were created in this repository:

[https://github.com/osrf/ros_dds](https://github.com/osrf/ros_dds)

More questions and some of the results were captured as issues:

[https://github.com/osrf/ros_dds/issues?labels=task&page=1&state=closed](https://github.com/osrf/ros_dds/issues?labels=task&page=1&state=closed)

The major piece of work in this repository is in the `prototype` folder and is a ROS 1.x like implementation of the Node, Publisher, and Subscriber API using DDS:

[https://github.com/osrf/ros_dds/tree/master/prototype](https://github.com/osrf/ros_dds/tree/master/prototype)

Specifically this prototype includes these packages:

- Generation of DDS IDL's from `.msg` files: [https://github.com/osrf/ros_dds/tree/master/prototype/src/genidl](https://github.com/osrf/ros_dds/tree/master/prototype/src/genidl)
- Generation of DDS specific C++ code for each generated IDL file: [https://github.com/osrf/ros_dds/tree/master/prototype/src/genidlcpp](https://github.com/osrf/ros_dds/tree/master/prototype/src/genidlcpp)
- Minimal ROS Client Library for C++ (rclcpp): [https://github.com/osrf/ros_dds/tree/master/prototype/src/rclcpp](https://github.com/osrf/ros_dds/tree/master/prototype/src/rclcpp)
- Talker and listener for pub-sub and service calls: [https://github.com/osrf/ros_dds/tree/master/prototype/src/rclcpp_examples](https://github.com/osrf/ros_dds/tree/master/prototype/src/rclcpp_examples)
- A branch of `ros_tutorials` in which `turtlesim` has been modified to build against the `rclcpp` library: [https://github.com/ros/ros_tutorials/tree/ros_dds/turtlesim](https://github.com/ros/ros_tutorials/tree/ros_dds/turtlesim).
  This branch of `turtlesim` is not feature-complete (e.g., services and parameters are not supported), but the basics work, and it demonstrates that the changes required to transition from ROS 1.x `roscpp` to the prototype of ROS 2.0 `rclcpp` are not dramatic.

This is a rapid prototype which was used to answer questions, so it is not representative of the final product or polished at all.
Work on certain features was stopped cold once key questions had been answered.

The examples in the `rclcpp_example` package showed that it was possible to implement the basic ROS like API on top of DDS and get familiar behavior.
This is by no means a complete implementation and doesn't cover all of the features, but in stead it was for educational purposes and addressed most of the doubts which were held with respect to using DDS.

Generation of IDL files proved to have some sticking points, but could ultimately be addressed, and implementing basic things like services proved to be tractable problems.

In addition to the above basic pieces, a pull request was drafted which managed to completely hide the DDS symbols from any publicly installed headers for `rclcpp` and `std_msgs`:

[https://github.com/osrf/ros_dds/pull/17](https://github.com/osrf/ros_dds/pull/17)

This pull request was ultimately not merged because it was a major refactoring of the structure of the code and other progress had been made in the mean time.
However, it served its purpose in that it showed that the DDS implementation could be hidden, though there is room for discussion on how to actually achieve that goal.


## Conclusion

After working with DDS and having a healthy amount of skepticism about the ethos, community, and licensing, it is hard to come up with any real technical criticisms.
While it is true that the community surrounding DDS is very different from the ROS community or the ZeroMQ community, it appears that DDS is just solid technology on which ROS could safely depend.
There are still many questions about exactly how ROS would utilize DDS, but they all seem like engineering exercises at this point and not potential deal breakers for ROS.
