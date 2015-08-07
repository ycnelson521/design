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

考慮到ROS社群的進一步成長跟發展性，我們對一些新的使用情境特別有興趣，這些使用情境是我們在剛開始開發ROS時尚未考慮到的，以下是我們有興趣的使用情境:

- 多機器人協作: 雖然使用ROS 1.0可以開發多機器人系統，但標準作法並不存在。此外，目前的作法都是建構在ROS 1.0單一master架構上的特殊解法。
- 嵌入式平台開發: 我們希望小型電腦，包含各式各樣的微處理機，都能直接而良好地運行ROS，而不是得透過裝置的驅動程式跟ROS溝通。
- real-time系統: 我們希望ROS可以直接支援real-time控制，而且一次控制週期內的控制命令中可以包含跨process或是跨機器之間的溝通。(假設有良好的作業系統和硬體支援)
- 不理想的網路狀況: 我們希望即使網路連線不佳(例如存在封包遺失或延遲的狀況)，ROS還能盡可能地擁有正常的表現。這些情況可能發生在不良的無線網路訊號或是地面至太空的網路連線。
- 適用於產品開發:持續讓學術研究機構使用ROS來開發機器人prototype很重要，但除此之外，我們也希望這些prototype可以進一步演化為搭載ROS的產品，成為在現實生活中具有實際用途的應用。
- 提供既定的系統建立與架構模式:ROS的一個標誌性特色就在於由多個process構成的系統可以依使用者需求彈性地變動，在保持這個特色的同時，我們也希望提供清晰的系統架構模式。此外，我們也希望支援life cycle management和static configurations for deployment等功能。


## 新技術的引進

ROS的核心是具有匿名性發佈與訂閱機制的中介軟體(middleware)，這個middleware是我們從無到有開發出來的。自2007年開始，我們自行撰寫了程序管理、自定義訊息、序列化跟資料傳輸的工具。在這七年之間，有許多新出現的技術可以處理上述的這些功能，包含:

- Zeroconf;
- Protocol Buffers;
- ZeroMQ (and the other MQs);
- Redis;
- WebSockets; and
- DDS (Data Distribution Service).

隨著這些新技術的出現，採用現成的開源函式庫來開發ROS這類型的中介軟體式相當可行的，而且這種做法具備相當多的好處，例如:

- 需要維護的程式碼量會減少，可以專注在非特定機器人(或者說較能被廣泛使用的)的程式碼;
- 我們可以運用這些開源函式庫的特色，而不需要自行開發這些功能;
- 我們可以得利於他人對這些函式庫的改進，讓ROS自然地變得更好;
- 當其他人問我們ROS何時準備就緒時，我們可以參考使用相同開源函式庫的系統並提供洞見.


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
