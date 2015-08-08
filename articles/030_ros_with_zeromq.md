---
layout: default
title: 使用ZeroMQ跟相關的函式庫來開發ROS
html_title: ZeroMQ and Friends
permalink: articles/ros_with_zeromq.html
abstract:
  這篇文章討論如何使用[ZeroMQ](http://blog.ez2learn.com/2011/12/31/transport-lib-of-new-era-zeromq/)跟其他的函式庫來開發ROS 2.0。此外，我們在OSRF(Open Source Robotic Foundation)使用ZeroMQ開發出的prototype也會在這篇文章中被討論。
published: true
author: '[William Woodall](https://github.com/wjwwood)'
translator: 賴柏任
---

* This will become a table of contents (this text will be scraped).
{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

> **這篇文章是在決定使用DDS當作ROS 2.0的資料傳輸工具之前所撰寫的**

Original Author: {{ page.author }}

Translator: {{ page.translator }}

雖然這篇文章主要聚焦在使用ZeroMQ來建一個新的中介軟體（middleware），但其實也討論了如何利用數個函式庫來構建出一個新的中介軟體。值得注意的是，使用數個函式庫來打造來ROS的策略，跟使用已存在且提供許多功能的中介軟體來打造ROS的策略是大相逕庭的。

## 從頭打造一個中介軟體的Prototype

為了滿足ROS的需求，我們新開發出的中介軟體需要提供幾個重要的服務。

首先，這個中介軟體需要讓建立在其之上的各個子系統可以發現彼此(discovery)，並且可以在執行期間動態地建立連結以利互相溝通。第二，建立連結之後，要有一個以上的傳輸機制讓子系統之間可以傳遞訊息，以ROS而言，至少要提供訂閱-發布的溝通機制。如此一來，額外的溝通機制（例如送出要求跟回應的service機制）都可以使用訂閱-發布的機制來實作。最後，這個中介軟體應該提供方法讓使用者定義訊息格式並提供傳輸機制（也就是將訊息序列化）。不過即使中介軟體本身不實作這個機制，也能透過使用其他函式庫來解決這個問題。

### 讓各個子系統發現彼此

在考慮discovery這個問題時，我們最先想到的解決方案是[Zeroconf](http://blog.csdn.net/ccskyer/article/details/7616673)這個協議，搭配Avahi跟Bonjour這兩套實作zeroconf協議的函式庫。
我們使用[pybonjour](https://code.google.com/p/pybonjour/)做了一些簡單的實驗，來測試以zeroconf協議為基礎的系統中discovery的效果。
其中的核心技術是`mDNSresponder`，它是蘋果公司提供的自由軟體，同時被Bonjour(OS X跟Windows)以及Avahi(Linux)所使用。

但是經過我們的實驗發現，這些實作zeroconf協議的函式庫並不能非常穩定地維持多機器之間的網路狀態圖。如果使用子程序一次加入或移除超過20台機器，在整個網路中至少會有一台以上的電腦無法跟其他電腦保持相同的網路狀態圖。舉例來說，假設A電腦已經發現了幾個nodes(一個node表示網路中的一個連線單位，在此例中是電腦)，B跟C電腦也發現這幾個nodes並已經把這些nodes顯示在自己的連線狀態圖中。當這些nodes所表示的電腦在幾秒之後被關掉電源，理論上，A~C這三台電腦上的網路狀態圖都應該同時把這些nodes移除掉，但問題就是，通常只有部份的nodes會被移除（被留下來的nodes就變成了zombie nodes），更麻煩的是，三台電腦移除的nodes還不一致。這個問題只有在多機器網路中使用Avahi才會出現，所以我們更進一步地深入了解Avahi的核心，來確認這個問題是否可以解決。然而，更進一步的了解讓我們更擔心Avahi的穩定性和品質，尤其是在[Multicast DNS](http://en.wikipedia.org/wiki/Multicast_DNS)和[DNS Service Discovery](http://en.wikipedia.org/wiki/Zero_configuration_networking#Service_discovery)這兩項技術的實作上。

Further more DNS-SD seems to prefer the trade-off of light networking load for eventual consistency.
This works reasonably well for something like service name look up, but it did not work well for quickly and reliably discovering the proto-ROS graph in the experiments.
This lead to the development of a custom discovery system which is implemented in a few languages as part of the prototype here:

[https://bitbucket.org/osrf/disc_zmq/src](https://bitbucket.org/osrf/disc_zmq/src)

The custom discovery system used multicast UDP packets to post notifications like "Node started", "Advertise a publisher", and "Advertise a subscription", along with any meta data required to act, like for publishers, an address to connect to using ZeroMQ.
The details of this simple discovery system can be found at the above URL.

This system, though simple, was quite effective and was sufficient to prove that implementing such a custom discovery system, even in multiple languages is a tractable problem.

### 資料傳輸

For transporting bytes between processes, a popular library is [ZeroMQ](http://zeromq.org/), but there are also libraries like [nanomsg](http://nanomsg.org/) and [RabbitMQ](http://www.rabbitmq.com/).
In all of those cases the goal of the library is to allow you to establish connections, explicitly, to other participants and then send strings or bytes according to some communication pattern.
ZeroMQ is an LGPL licensed library which has recently become very popular, is written in C++ with a C API, and has bindings to many languages.
nanomsg is a new MIT licensed library which was created by one of the original authors of ZeroMQ, is written in C with a C API, but is far less mature than ZeroMQ.
RabbitMQ is a broker that implements several messaging protocols, mainly AMQP, but also provides gateways for ZeroMQ, STOMP and MQTT. By being a broker, it meets some of the discovery needs as well as the transport needs for ROS. Although ZeroMQ is usually used in brokerless deployments, it can also be used in conjunction with RabbitMQ to provide persitence and durability of messages.
RabbitMQ is licensed under the Mozilla Public License.
All of these libraries could probably be used to replace the ROSTCP transport, but for the purposes of this article we will use ZeroMQ in a brokerless deployment.

In this prototype:

[https://bitbucket.org/osrf/disc_zmq/src](https://bitbucket.org/osrf/disc_zmq/src)

ZeroMQ was used as the transport, which conveniently has bindings in C, C++, and Python.
After making discoveries using the above described simple discovery system, connections were made using ZeroMQ's `ZMQ_PUB` and `ZMQ_SUB` socket's.
This worked quite well, allowing for communication between process in an efficient and simple way.
However, in order to get more advanced features, like for instance latching, ZeroMQ takes the convention approach, meaning that it must be implemented by users with a well known pattern.
This is a good approach which keeps ZeroMQ lean and simple, but does mean more code which must be implemented and maintained for the prototype.

Additionally, ZeroMQ, in particular, relies on reliable transports like TCP or [PGM (Pragmatic General Multicast)](http://en.wikipedia.org/wiki/Pragmatic_General_Multicast), so it makes it unsuitable for soft real-time scenarios.

### 訊息序列化

In ROS 1.x, messages are defined in `.msg` files and code is generated at build time for each of the supported languages. ROS 1.x generated code can instantiate and then later serialize the data in a message as a mechanism for exchanging information.
Since ROS was created, several popular libraries which take care of this responsibility have come about.
Google's [Protocol Buffers (Protobuf)](https://code.google.com/p/protobuf/), [MessagePack](http://msgpack.org/), [BSON](http://bsonspec.org/), and [Cap'n Proto](http://kentonv.github.io/capnproto/) are all examples of serialization libraries which have come to popularity since ROS was originally written.
An entire article could be devoted to the pros and cons of different message definition formats, serialization libraries, and their wire formats, but for the purposes of this prototype we worked with either plain strings or Protobuf.

## 結論

After implementing the custom middleware prototype, some points worth noting were made.
First, there isn't any existing discovery systems which address the needs of the middleware which are not attached to other middlewares.
Implementing a custom discovery system is a possible but time consuming.

Second, there is a good deal of software that needs to exist in order to integrate discovery with transport and serialization.
For example, the way in which connections are established, whether using point to point or multicast is a piece of code which lives between the transport and discovery systems.
Another example is the efficient intra-process communications, ZeroMQ provides an INPROC socket, but the interface to that socket is bytes, so you cannot use that without serialization without constructing a system where you pass around pointers through INPROC rather than serialized data.
At the point where you are passing around pointers rather than serialized data you have to start to duplicate behavior between the intraprocess and interprocess communications which are abstracted at the ROS API level.
One more piece of software which is needed is the type-safety system which works between the transport and the messages serialization system.
Needless to say, even with these component libraries solving a lot of the problems with implementing a middleware like ROS's, there still exists quite a few glue pieces which are need to finish the implementation.

Even though it would be a lot of work to implement a middleware using component libraries like ZeroMQ and Protobuf, the result would likely be a finely tuned and well understood piece of software.
This path would most likely give the most control over the middleware to the ROS community.

In exchange for the creative control over the middleware, comes the responsibility to document its behavior and design to the point that it can be verified and reproduced.
This is a non-trivial task which ROS 1.x did not do very well because it had a relatively good pair of reference implementations.
Many users which wish to put ROS into mission critical situations and into commercial products have lamented that ROS lacks this sort of governing design document which allows them to certify and audit the system.
It would be of paramount importance that this new middleware be well defined, which is not a trivial task and almost certainly rivals the engineering cost of the initial implementation.
