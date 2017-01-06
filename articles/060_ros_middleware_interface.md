---
layout: default
title: ROS 2 middleware interface
permalink: articles/ros_middleware_interface.html
abstract:
  This article describes the rational for using an abstract middleware interface between ROS and a specific middleware implementation.
  It will outline the targeted use cases as well as their requirements and constraints.
  Based on that the developed middleware interface is explained.
  這篇文章描述在ROS與某一特定中介軟體十座中使用抽象中介軟體介面的理由。這將會約略描述目標使用案例們以及它們的需求與限制。
  我們將基於前述的項目說明目前開發的中介軟體。
author: '[Dirk Thomas](https://github.com/dirk-thomas)'
published: true
---

* This will become a table of contents (this text will be scraped).
{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Original Author: {{ page.author }}


## The *middleware interface* 中介軟體介面

### Why does ROS 2 have a *middleware interface*? 為何 ROS 2 有一個中介軟體介面

The ROS client library defines an API which exposed communication concepts like publish / subscribe to users.
ROS客戶端資料庫(client library)定義一個應用程式介面(API)，此API將發佈/訂閱(publish/subscribe)通訊概念公開給使用者們。

In ROS 1 the implementation of these communication concepts were build on custom protocols (e.g., [TCPROS](http://wiki.ros.org/ROS/TCPROS)).
在ROS 1，這些通訊概念建立於客製的協定(如[TCPROS](http://wiki.ros.org/ROS/TCPROS))。

For ROS 2 the decision has been made to build it on top of an existing middleware solution (namely [DDS](http://en.wikipedia.org/wiki/Data_Distribution_Service)). The major advantage of that approach is that ROS 2 can leverage an existing and well developed implementation of that standard.
在ROS 2，這些通訊概念將建立於現存的中介軟體解決方案(分散式資料服務，[DDS](http://en.wikipedia.org/wiki/Data_Distribution_Service))。最大的好處在於 ROS 2 可以利用已開發的DDS實作基礎。

ROS could build on top of one specific implementation of DDS.
But there are numerous different implementations available and each has its own pros and cons in terms of supported platforms, programming languages, performance characteristics, memory footprint, dependencies and licensing.
ROS可建立於某一特定DDS的實作，但目前有很多DDS實作，每個實作在支援的平台、程式語言、效能特性、記憶體使用量、相依姓、與授權各有優劣。

Therefore ROS aims to support multiple DDS implementations despite the fact that each of them differ slightly in their exact API.
In order to abstract from the specifics an abstract interface is being introduces which can be implemented for different DDS implementations.
This *middleware interface* defines the API between the ROS client library and any specific implementation.
因此ROS以支援多種DDS實作為目標，即便每個實作在具體API設計都稍微不同。為了從這些具體API差異抽象化出來，一個可以用於不同DDS實作的抽像介面被提出。此中介軟體介面定義了ROS客戶端資料庫與任一特定DDS實作間的API。

Each implementation of the interface will usually be a thin adapter which maps the generic *middleware interface* to the specific API of the middleware implementation.
In the following the common separation of the adapter and the actual middleware implementation will be omitted.
每個介面的實作通常會是一層薄薄的轉接器(adapter)，此轉接器將一般中介軟體介面對映至某個特定中介軟體實作的API。在下面，轉接器與實際中介軟體實作間的共通分界將被省略。

    +-----------------------------------------------+
    |                   user land                   |
    +-----------------------------------------------+
    |              ROS client library               |
    +-----------------------------------------------+
    |             middleware interface              |
    +-----------------------------------------------+
    | DDS adapter 1 | DDS adapter 2 | DDS adapter 3 |
    +---------------+---------------+---------------+
    |    DDS impl 1 |    DDS impl 2 |    DDS impl 3 |
    +---------------+---------------+---------------+


## Why should the *middleware interface* be agnostic to DDS? 為何中介軟體介面應該與DDS無關

The ROS client library should not expose any DDS implementation specifics to the user.
This is primarily to hide the intrinsic complexity of the DDS specification and API.
ROS客戶端資料庫不應向用戶公開任何DDS實現細節。這主要是為了隱藏DDS規範和API的內在複雜度。

While ROS 2 only aims to support DDS based middleware implementations it can strive to keep the *middleware interface* free of DDS specific concepts to enable implementations of the interface using a different middleware.
It would also be feasible to implement the interface by tying together several unrelated libraries providing the necessary functions of discovery, serialization and publish / subscribe.
雖然ROS 2目標僅在支援基於DDS的中介軟體實作，它可以努力保持中介軟體介面沒有特定DDS細節概念，以使其能夠使用不同的中介軟體來實作介面。透過將幾個提供發現、序列化、發布/訂閱必要功能的不相關的資料庫連結在一起來實作介面也是可行的。

    +-----------------------------------+
    |             user land             |   no middleware implementation specific code
    +-----------------------------------+
    |        ROS client library         |   above the interface
    +-----------------------------------+
    |       middleware interface        |   ---
    +-----------------------------------+
    | mw impl 1 | mw impl 2 | mw impl 3 |
    +-----------+-----------+-----------+


## How does the information flow through the *middleware interface*? 資訊如何在中介軟體介面中流通

One goal of the *middleware interface* is to not expose any DDS specific code to the user land code.
Therefore the ROS client library "above" the *middleware interface* needs to only operate on ROS data structures.
ROS 2 will continue to use ROS message files to define the structure of these data objects and derive the data structures for each supported programming language from them.
中介軟體介面的目標之一是不公開任何DDS細節原始碼給用戶土地(user land)原始碼。因此，ROS客戶端資料庫“以上”的中介軟體介面只需要操作ROS資料結構。 ROS 2將繼續使用ROS訊息檔案來定義這些資料物件的結構，並從中產生每種支援的程式語言資料結構。

The middleware implementation "below" the *middleware interface* must convert the ROS data objects provided from the client library into its own custom data format before passing it to the DDS implementation.
In reverse custom data objects coming from the DDS implementation must be converted into ROS data objects before being returned to the ROS client library.
在中介軟體介面”以下”的中介軟體實作必須將從客戶端資料庫提供的ROS資料物件轉換為自己的自定義資料格式，然後再將其傳遞給DDS實作。在來自DDS實作的反向自定義資料物件必須在回傳到ROS客戶端資料庫之前轉換為ROS資料物件。

The definition for the middleware specific data types can be derived from the information specified in the ROS message files.
A defined mapping between the primitive data types of ROS message and middleware specific data types ensures that a bidirectional conversion is possible.
中介軟體特定資料型態的定義可以從ROS訊息檔案中定義的訊息導出。ROS訊息原始資料類型和中介軟體特定資料型態之間預先定義的映射確保了雙向轉換是可能的。

    +----------------------+
    |      user land       |   1) create a ROS message
    +----------------------+      v
    |  ROS client library  |   2) publish the ROS message
    +----------------------+      v
    | middleware interface |      v
    +----------------------+      v
    |      mw impl N       |   3) convert the ROS message into a DDS sample and publish the DDS sample
    +----------------------+

<!--- separate code blocks -->

    +----------------------+
    |      user land       |   1) use the ROS message
    +----------------------+      ^
    |  ROS client library  |   2) callback passing a ROS message
    +----------------------+      ^
    | middleware interface |      ^
    +----------------------+      ^
    |      mw impl N       |   3) convert the DDS sample into a ROS message and invoke subscriber callback
    +----------------------+

Depending on the middleware implementation the extra conversion can be avoided by implementing serialization functions directly from ROS messages as well as deserialization functions into ROS messages.
根據中介軟體實作，可以通過直接從ROS訊息實作序列化函數以及將反序列化函數實作為ROS訊息來避免額外的轉換。


## Considered use cases

The following use cases have been considered when designing the middleware interface:


### Single middleware implementation

ROS applications are not built in a monolithic way but distributed across several packages.
Even with a *middleware interface* in place the decision which middleware implementation to use will affect significant parts of the code.

E.g. a package defining a ROS message will need to provide the mapping to and from the middleware specific data type.
Naively each package defining ROS messages might contain custom (usually generated) code for the specific middleware implementation.

In the context of providing binary packages of ROS (e.g., Debian packages) that implies that a significant part of them (at least all packages containing message definitions) would be specific to the selected middleware implementation.

                    +-----------+
                    | user land |
                    +-----------+
                         |||
          +--------------+|+-----------------+
          |               |                  |
          v               v                  v
    +-----------+   +-----------+   +-----------------+   All three packages
    | msg pkg 1 |   | msg pkg 2 |   | middleware impl |   on this level contain
    +-----------+   +-----------+   +-----------------+   middleware implementation specific code


### Static vs. dynamic message types with DDS

DDS has two different ways to use and interact with messages.

On the one hand the message can be specified in an IDL file from which usually a DDS implementation specific program will generate source code from.
The generated code for C++, e.g., contains types specifically generated for the message.

On the other hand the message can be specified programmatically using the DynamicData API of the [XTypes](http://www.omg.org/spec/DDS-XTypes/) specification.
Neither an IDL file nor a code generation step is necessary for that.

Some custom code must still map the message definition available in the ROS .msg files to invocations of the DynamicData API.
But it is possible to write generic code which performs the task for any ROS .msg specification passed in.

                    +-----------+
                    | user land |
                    +-----------+
                         |||
          +--------------+|+----------------+
          |               |                 |
          v               v                 |
    +-----------+   +-----------+           |            Each message provides its specification
    | msg pkg 1 |   | msg pkg 2 |           |            in a way which allows a generic mapping
    +-----------+   +-----------+           |            to the DynamicData API
          |               |                 |
          +-------+-------+                 |
                  |                         |
                  v                         v
     +-------------------------+   +-----------------+   Only the packages
     | msg spec to DynamicData |   | middleware impl |   on this level contain
     +-------------------------+   +-----------------+   middleware implementation specific code

However the performance using the DynamicData API will likely always be lower compared to the statically generated code.


### Switch between different implementations

When ROS supports different middleware implementations it should be as easy and low effort as possible for users to switch between them.

One obvious way will be for a user to rebuild all ROS packages from source selecting a different middleware implementation.
While the workflow won't be too difficult (probably half a dozen command-line invocations), it still requires quite some build time.

To avoid the overhead of rebuilding everything a different set of binary packages could be provided.
While this would reduce the effort for the user the buildfarm would need to build a completely separate set of binary packages.
The effort to support N packages with M different middleware implementation would require significant resources for maintaining the service as well as the necessary computing power.


#### Reduce the number of middleware implementation specific packages

One way to at least reduce the effort for building for different middleware implementations is to reduce the number of packages depending on the specific middleware implementation.
This can, e.g., be achieved using the DynamicData API mentioned before.
Since only a few packages need to be built for each middleware implementation it would be feasible to generate binary packages for them on the buildfarm.

The user could then install (one | a few) binary package(s) for a specific middleware implementation together with its specific implementation of the mapping between the message specification and the DynamicData API.
All other binary packages would be agnostic to the selected middleware implementation.

            +-----------+
            | user land |
            +-----------+
                  |
                  v
            +-----------+           Generic binary packages
            | msg pkgs  |           agnostic to selected
            +-----------+           middleware implementation
                  |
                  ?                 Select middleware implementation
                /   \               by installing (one | a few)
              /       \             binary package(s)
    +-----------+   +-----------+
    | mw impl 1 |   | mw impl 2 |
    +-----------+   +-----------+


#### Generate "fat" binary packages

Another way to enable the user to switch between middleware implementations without the need to use the DynamicData API is to embed the middleware specific generated code for all supported implementations into each binary package.
The specific middleware implementation would then be selected, e.g., at link time and only the corresponding generated code of that middleware implementation will be used.


### Using single middleware implementation only

When building ROS with a single middleware implementation the result should follow the design criteria:
> Any features that you do not use you do not pay for.

That implies that there should be no overhead for either the build time nor the runtime due to the ability to support different middleware implementations.
However the additional abstraction due to the middleware interface is still valid in order to hide implementation details from the user.


## Design of the *middleware interface*

The API is designed as a pure function-based interface in order to be implemented in C.
A pure C interface can be used in ROS Client Libraries for most other languages including Python, Java, and C++ preventing the need to reimplement the core logic.


### Publisher interface

Based on the general structure of ROS nodes, publishers and messages for the case of publishing messages the ROS client library need to invoke three functions on the middleware interface:

* `create_node()`
* `create_publisher()`
* `publish()`


#### Essential signature of `create_node`

Subsequent invocations of `create_publisher` need to refer to the specific node they should be created in.
Therefore the `create_node` function needs to return a *node handle* which can be used to identify the node.

    NodeHandle create_node();


#### Essential signature of `create_publisher`

Beside the *node handle* the `create_publisher` function needs to know the *topic name* as well as the *topic type*.
The type of the *topic type* argument is left unspecified for now.

Subsequent invocations of `publish` need to refer to the specific publisher they should send messages on.
Therefore the `create_publisher` function needs to return a *publisher handle* which can be used to identify the publisher.

    PublisherHandle create_publisher(NodeHandle node_handle, String topic_name, .. topic_type);

The information encapsulated by the *topic type* argument is highly dependent on the middleware implementation.


##### Topic type information for the DynamicData API

In the case of using the DynamicData API in the implementation there is no C / C++ type which could represent the type information.
Instead the *topic type* must contains all information to describe the format of the message.
This information includes:

* the name of the package in which the message is defined
* the name of the message
* the list of fields of the message where each includes:
  * the name for the message field
  * the type of the message field (which can be either a built-in type or another message type), optionally the type might be an unbounded, bounded or fixed size array
  * the default value
* the list of constants defined in the message (again consisting of name, type and value)

In the case of using DDS this information enables to:

* programmatically create a *DDS TypeCode* which represents the message structure
* register the *DDS TypeCode* with the *DDS Participant*
* create a *DDS Publisher*, *DDS Topic* and *DDS DataWriter*
* convert data from a *ROS message* into a *DDS DynamicData* instance
* write the *DDS DynamicData* to the *DDS DataWriter*


##### Topic type information for statically generated code

In the case of using statically generated code derived from an IDL there is are C / C++ types which represent the type information.
The generated code contains functions to:

* create a *DDS TypeCode* which represents the message structure

Since the specific types must be defined at compile time the other functionalities can not be implemented in a generic (not specific to the actual message) way.
Therefore for each message the code to perform the following tasks must be generated separately:

* register the *DDS TypeCode* with the *DDS Participant*
* create a *DDS Publisher*, *DDS Topic* and *DDS DataWriter*
* convert data from a *ROS message* into a *DDS DynamicData* instance
* write the *DDS DynamicData* to the *DDS DataWriter*

The information encapsulated by the *topic type* must include the function pointers to invoke these functions.


#### `get_type_support_handle`

Since the information encapsulated by the *topic type* argument is so fundamentally different for each middleware implementation it is actually retrieved through an additional function of the middleware interface:

    MessageTypeSupportHandle get_type_support_handle();

Currently this function is a template function specialized on the specific ROS message type.
For C compatibility an approach using a macro will be used instead which mangles the type name into the function name.


#### Essential signature of `publish`

Beside the *publisher handle* the `publish` function needs to know the *ROS message* to send.

    publish(PublisherHandle publisher_handle, .. ros_message);

Since ROS messages do not have a common base class the signature of the function can not use a known type for the passed ROS message.
Instead it is passed as a void pointer and the actual implementation reinterprets it according to the previously registered type.


#### Type of the returned handles

The returned handles need to encapsulate arbitrary content for each middleware implementation.
Therefore these handles are just opaque objects from a user point of view.
Only the middleware implementation which created it knows how to interpret it.

In order to ensure that these information are passed back to the same middleware implementation each handle encodes a unique identifier which the middleware implementation can check before interpreting the handles payload.


### Subscriber interface

The details of the interface necessary for the subscriber side are not (yet) described in this document.


## Current implementation

The described concept has been implemented in the following packages:

* the package [rmw](https://github.com/ros2/rmw/tree/master/rmw) defines the middleware interface
  * the functions are declared in [rms/rmw.h](https://github.com/ros2/rmw/blob/master/rmw/include/rmw/rmw.h)
  * the handles are defined in [rmw/types.h](https://github.com/ros2/rmw/blob/master/rmw/include/rmw/types.h)
* the package [rmw_connext_cpp](https://github.com/ros2/rmw_connext/tree/master/rmw_connext_cpp) implements the middleware interface using [RTI Connext DDS](http://www.rti.com/products/dds/index.html) based on statically generated code
  * the package [rosidl_typesupport_connext_cpp](https://github.com/ros2/rmw_connext/tree/master/rosidl_typesupport_connext_cpp) generates
    * the DDS specific code based on IDL files for each message
      * the package [rosidl_generator_dds_idl](https://github.com/ros2/rosidl_dds/tree/master/rosidl_generator_dds_idl) generates DDS IDL files based on ROS msg files
    * additional code to enable invoking the register/create/convert/write functions for each message type
* the package [rmw_connext_dynamic_cpp](https://github.com/ros2/rmw_connext/tree/master/rmw_connext_dynamic_cpp) implements the middleware interface using *RTI Connext DDS* based on the DynamicData API
  * the package [rosidl_typesupport_introspection_cpp](https://github.com/ros2/rosidl_dds/tree/master/rosidl_typesupport_introspection_cpp)
    * generates code which encapsulated the information from each ROS msg file in a way which makes the data structures introspectable
* the package [rmw_opensplice_cpp](https://github.com/ros2/rmw_opensplice/tree/master/rmw_opensplice_cpp) implements the middleware interface using [PrismTech OpenSplice DDS](http://www.prismtech.com/opensplice) based on statically generated code
  * the package [rosidl_typesupport_opensplice_cpp](https://github.com/ros2/rmw_opensplice/tree/master/rosidl_typesupport_opensplice_cpp) generates
    * the DDS specific code based on IDL files for each message
    * additional code to enable invoking the register/create/convert/write functions for each message type


### One or multiple type support generators

The packages contributing to the message generation process of each message are called *type support generators*.
Their package name starts with the prefix `rosidl_typesupport_`.

Each message package will contain the generated code from all type support generators which are available when the package is configured.
This can be only one (when building against a single middleware implementation) or multiple type support generators.


### User land code decides at link time

The packages implementing the middleware interface are called *middleware implementations*.
Their package name starts with the prefix `rmw_` followed by the name of the used middleware (e.g., *connext* or *opensplice*).

A user land executable using ROS nodes, publishers and subscribers must link against one specific middleware implementation library.
The used middleware implementation library will only use the corresponding type support from each message package and ignore any additionally available type supports.
Only middleware implementations can be selected for which the corresponding type support generators have been available when building the message packages.
