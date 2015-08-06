---
layout: default
title: 為什麼要開發ROS 2.0?
permalink: articles/why_ros2.html
abstract:
  這篇文章描述為什麼要對ROS的API做突破性修改，即ROS 2.0。
published: true
author: Brian Gerkey
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


我們從2007年的11月開始開發ROS。
自那時起，有很多的更新跟ROS版本的升級，我們認為現在已經是時候著手開發下一個世代的ROS。接下來，我們會解釋原因。

## 我們怎麼到達今天的狀態

ROS的誕生，是為了成為PR2機器人（由Willow Garage製造）的軟體開發環境。
我們的主要目標是希望ROS成為用PR2做出創新研究和開發的使用者們，所需要的軟體開發工具。但我們也知道，PR2不會是唯一且最重要的機器人，所以我們希望ROS也能在其他機器人上良好的運作。因此我們花了許多精神在定義多階的抽象層，將機器人的硬體，韌體，ROS和應用程式層做出區隔（層與層之間透過我們定義好的通訊介面來溝通），這種架構使得程式碼很容易被重複使用，進而節省開發的時程。

不過，在開發的過程中，我們仍然依循PR2的使用需求來引導開發方向，這使得ROS具有以下幾個顯著的特色：

- 主要運行於單一機器人上;
- 擁有工作站等級的運算資源;
- 沒有real-time的運算需求（或者說，若有特殊的real-time的運算需求，可以用特殊解法來處理）;
- 優良的網路連線能力（不論是有線網路或是近距離的高頻寬無線網路）;
- 有許多學術界的研究成果和應用;
- 最大的開發彈性(例如我們並不要求你的程式必須從main()開始執行).

我們可以說，今日的ROS滿足了PR2的使用需求，但令我們意想不到的是，ROS也被運用在[相當多種類的機器人](http://wiki.ros.org/Robots)上。
除了PR2跟與PR2相仿的機器人之外，各種尺寸的輪型機器人、雙足人形機器人、工業機器人、戶外自動車（包含無人駕駛自動車）、飛行器、水面航行艇等等廣義的機器人都在使用ROS。

此外，我們觀察到ROS已經不僅被我們一開始鎖定的學術界所採用。截至今日，搭載ROS的產品已經面世，包含自動化製造機器人、農業生產機器人、清潔機器人等等。政府相關單位也積極了解ROS可以怎麼被運用於先進技術的開發，例如NASA預計將ROS運行於將在國際太空站上工作的Robonaut 2機器人。

隨著這些新用途的誕生，ROS的可能性被拉伸的程度是我們原先完全無法預期的。
雖然ROS 1.0運行良好並被廣泛延伸，我們相信，我們可以開始面對新的使用需求來更好地滿足日益寬廣的ROS社群。

## 新的使用情境

Of specific interest to us for the ongoing and future growth of the ROS community are the following use cases, which we did not have in mind at the beginning of the project:

- Teams of multiple robots: while it is possible to build multi-robot systems using ROS today, there is no standard approach, and they are all somewhat of a hack on top of the single-master structure of ROS.
- Small embedded platforms:  we want small computers, including "bare-metal" micro controllers, to be first-class participants in the ROS environment, instead of being segregated from ROS by a device driver.
- Real-time systems: we want to support real-time control directly in ROS, including inter-process and inter-machine communication (assuming appropriate operating system and/or hardware support).
- Non-ideal networks: we want ROS to behave as well as is possible when network connectivity degrades due to loss and/or delay, from poor-quality WiFi to ground-to-space communication links.
- Production environments: while it is vital that ROS continue to be the platform of choice in the research lab, we want to ensure that ROS-based lab prototypes can evolve into ROS-based products suitable for use in real-world applications.
- Prescribed patterns for building and structuring systems: while we will maintain the underlying flexibility that is the hallmark of ROS, we want to provide clear patterns and supporting tools for features such as life cycle management and static configurations for deployment.


## 新技術的引進

At the core of ROS is an anonymous publish-subscribe middleware system that is built almost entirely from scratch.
Starting in 2007, we built our own systems for discovery, message definition, serialization, and transport.
The intervening seven years have seen the development, improvement, and/or widespread adoption of several new technologies that are relevant to ROS in all of those areas, such as:

- Zeroconf;
- Protocol Buffers;
- ZeroMQ (and the other MQs);
- Redis;
- WebSockets; and
- DDS (Data Distribution Service).

It is now possible to build a ROS-like middleware system using off-the-shelf open source libraries.
We can benefit tremendously from this approach in many ways, including:

- we maintain less code, especially non-robotics-specific code;
- we can take advantage of features in those libraries that are beyond the scope of what we would build ourselves;
- we can benefit from ongoing improvements that are made by others to those libraries; and
- we can point to existing production systems that already rely on those libraries when people ask us where ROS is "ready for prime time".


## API的改進

A further reason to build ROS 2.0 is to take advantage of the opportunity to improve our user-facing APIs.
A great deal of the ROS code that exists today is compatible with the client libraries as far back as the 0.4 "Mango Tango" release from February 2009.
That's great from the point of view of stability, but it also implies that we're still living with API decisions that were made several years ago, some of which we know now to be not the best.

So, with ROS 2.0, we will design new APIs, incorporating to the best of our ability the collective experience of the community with the first-generation APIs.
As a result, while the key concepts (distributed processing, anonymous publish/subscribe messaging, RPC with feedback [i.e., actions], language neutrality, system introspectability, etc.) will remain the same, you should not expect ROS 2.0 to be API-compatible with existing ROS code.

But fear not: there will be mechanisms in place to allow ROS 2.0 code to coexist with existing ROS code.
At the very least, there will be translation relays that will support run-time interactions between the two systems.
And it is possible that there will be library shims that will allow existing ROS code to compile/run against ROS 2.0 libraries, with behavior that is qualitatively similar to what is seen today.

## 為什麼不改進 ROS 1 就好?

In principle, the changes described above could be integrated into the existing core ROS code.
E.g., new transport technologies could be added to `roscpp` and `rospy`.
We considered this option and concluded that, given the intrusive nature of the changes that would be required to achieve the benefits that we are seeking, there is too much risk associated with changing the current ROS system that is relied upon by so many people.
We want ROS 1 as it exists today to keep working and be unaffected by the development of ROS 2.
So ROS 2 will be built as a parallel set of packages that can be installed alongside and interoperate with ROS 1 (e.g., through message bridges).
