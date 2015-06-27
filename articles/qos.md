---
layout: default
title: ROS 2.0 Quality of Service policies
abstract:
  This article describes the approach to provide QoS (Quality of Service) policies for ROS 2.0.
published: false
author: '[Esteve Fernandez](https://github.com/esteve)'
---

* This will become a table of contents (this text will be scraped).
{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Original Author: {{ page.author }}

With the advent of inexpensive robots using unreliable wireless networks, developers and users need mechanisms to control how traffic is prioritized across network links.

## Background

ROS 1.x uses TCP as the underlying transport, which is unsuitable for lossy networks such as wireless links.
With ROS 2.0 relying on DDS which uses UDP as its transport, we can give control over the level of reliability a node can expect and act accordingly.

## DDS Quality of Service policies

DDS provides fine-grained control over the Quality of Service (QoS) setting for each of the entities involved in the system.
Common entities whose behavior can be modified via QoS settings include: Topic, DataReader, DataWriter, Publisher and Subscriber.
QoS is enforced based on a Request vs Offerent Model, however Publications and Subscriptions will only match if the QoS settings are compatible.

## ROS 2.0 proposal

Given the complexity of choosing the correct QoS settings for a given scenario, it may make sense for ROS 2.0 to provide a set of predefined QoS profiles for common usecases (e.g. sensor data, real time, etc.), while at the same time give the flexibility to control specific features of the QoS policies for the most common entities.

## Integration with existing DDS deployments

Both PrismTech OpenSplice and RTI Connext support loading of QoS policies via an external XML file.
In environments where DDS is already deployed and also to enable more extensibility other than the offered by the ROS 2.0 and the predefined profiles, ROS 2.0 may provide loading of the QoS settings via the same mechanisms the underlying DDS implementations use.

## References

* Gordon A. Hunt, [DDS - Advanced Tutorial, Using QoS to Solve Real-World Problems](http://www.omg.org/news/meetings/workshops/RT-2007/00-T5_Hunt-revised.pdf)

* [OpenDDS - QoS Policy Usages](http://www.opendds.org/qosusages.html)

* [OpenDDS - QoS Policy Compliance](http://www.opendds.org/qospolicies.html)

* Angelo Corsaro, [DDS QoS Unleashed](http://www.slideshare.net/Angelo.Corsaro/dds-qos-unleashed)

* Angelo Corsaro, [The DDS Tutorial Part II](http://www.slideshare.net/Angelo.Corsaro/the-dds-tutorial-part-ii)

* [DDS Spec, section 2.2.3](http://www.omg.org/spec/DDS/1.4/PDF/)
