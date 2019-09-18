---
layout: post
title:  "Developing a DisplayOnly Bluetooth agent in Qt/C++ with BlueZ and D-Bus - Part 1"
desc: "Blog"
keywords: "bluetooth, qt, c++, cpp, embedded, bluetooth agent, bluez, dbus"
date: 2019-09-10
categories: C++
tags: [bluetooth, qt, c++, embedded, bluetooth agent, bluez, dbus]
icon: fa-bookmark-o
---
**Welcome to this two-part series about Bluetooth, Qt and C++.** This post outlines the content of the series, gives you a quick introduction on the key concepts, and provides a first taste of implementation.

## Bluetooth agents and Qt: what to expect
Before we dive into today’s topic, let me explain why we are doing this series, who the intended audience is, and what we’ll cover. 

Here, I'm going to explain the main concepts of Bluetooth and D-Bus, linking them with the corresponding Qt API. First, we'll explore the [**basic concepts**](#intro), and then we'll take a look at the API we'll be using ([Qt Bluetooth](#qtbt), [Qt D-Bus](#qtdbus), and [BlueZ Agent](#bluezagent)). At the end, we'll get a first glance at the implementation, diving deeper in how Qt and BlueZ can be connected using D-Bus.

In the next part I'll guide you through the actual implementation of our Bluetooth agent, providing you with a working example that can immediately be integrated into your Bluetooth applications.


<a name="intro"></a>
## Introduction and key concepts
Bluetooth is a wireless technology standard used to exchange data between devices over short distances. It has evolved a lot over the last years, adding more and more functionality to each new version. At the beginning, its purpose was just to develop wireless headsets, but nowadays it can handle much more. For example, you can exchange files, share contact information, stream audio and locate objects, just to cite some of the supported profiles. 

But there's a trouble in paradise. When it comes to Bluetooth, things could sometimes get really obscure, even though, in theory, the flow is pretty simple: a device searches for all the available devices nearby, selects one of them and connects to it, requiring a number of supported profiles (i.e. functionality). When the connection succeeds, the two devices can interact with one another. That's pretty much it. However, before establishing the actual connection, those two very devices have to pair. Pairing is defined as the process to establish an encrypted connection between two devices, in order for them to allow subsequent flawlessly connections. When the information from the pairing process is stored on the devices, the process is also called bonding [\[1\]](#1).

Ok, pairing means letting two devices know each other, but this post was about Bluetooth agents. So, first things first: what's an agent?

Well, the two devices need someone to tell them they can trust each other. This "trust me I'm trustworthy"-step comes in many flavors, as different devices have different input and output capabilities. For example, smartphones can easily display a keyboard to let their users input a secret code, or ask them for a PIN confirmation, whereas headphones usually cannot. Hence, a piece of software handling all those cases is required. This is where **agents** come into play. An agent can usually handle five different possibilities: `DisplayOnly`, `DisplayYesNo`, `KeyboardOnly`, `KeyboardDisplay`, and `NoInputNoOutput`. The two devices share their capability, and the final pairing method is determined based on the combination of them. For example, a `NoInputNoOutput` agent always works without asking for confirmation or prompting for a PIN or passkey, whereas `KeyboardOnly` and `KeyboardDisplay` require the manual input of such PIN/passkey. `DisplayOnly` and `DisplayYesNo` agents just show other device's confirmation code, possibly with "Yes/No" buttons to accept the incoming pairing. See [\[2\]](#2) for a more detailed view of Bluetooth agents and how they are integrated into the BlueZ stack.

In general, a custom agent can be useful for devices interacting with human users as well. One above all, it can provide you with great flexibility to decide which Bluetooth profiles to enable (file transfer, audio playback, phone book access, and so forth). The most common use, anyway, is to restrict the access to those devices that are not able to interact with users. That could be a security requirement: whitelisting a set of authorized devices reduces the risk of unauthorized connections. Or, vice versa, we could blacklist a set of devices (even though whitelisting is generally to be preferred). Or again, for some design or implementation choice, we could handle only a given number of paired devices at the same time. 

It's worth to note that BlueZ will totally ignore the custom agent if it's capability is "NoInputNoOutput". This is a little counterintuitive, as one might think at it as the perfect fit for devices that cannot interact with human users. In these cases the correct value to set is "DisplayOnly". We'll discuss in this post how to tailor this category of agents for devices with no I/O capabilities.

In BlueZ, every application can register its own agent (one and only one), that will be used for every subsequent Bluetooth pairing request as long as the application is running. This is not required, though. If no agent is registered the default one will be used, and in most cases this is a good fit. You should register your own only if you need to perform custom actions during the pairing process. This is what this post is all about.

<a name="qtbt"></a>
### Qt Bluetooth
Qt provides a Bluetooth API to connect Bluetooth-enabled devices [\[3\]](#3). One of the main classes to handle the connection and get information about the current device is `QBluetoothLocalDevice`. Two are the main functionality: first, `hostMode()` and `setHostMode()` can control the Bluetooth status (e.g. if the device is discoverable or not); secondly, `pairingStatus()` and `requestPairing()` manage the entire pairing process. This class also exposes some signals and slots related with both the pairing and the connection, such as `pairingDisplayConfirmation()`, and `pairingDisplayPinCode()` for the former category, and `deviceConnected()`/`deviceDisconnected()` for the latter. However, those that are related with pairing are emitted if and only if the pairing was initiated by the device itself. This also means that they **are not** emitted if the device receives the connection. This immediately leads to the following question: how can we actually *control* the pairing process if we did not start the connection? The answer is simple: implement an agent and register it as the default one. This way the agent will be triggered whenever pairing starts.

This where D-Bus comes into play.

### D-Bus
D-Bus is basically a message system allowing applications to talk to one another in an easy way. It provides a software-bus abstraction to hide the complexity that would arise from a one-to-one communications between processes. Such abstraction is comprised of:
* one **system** bus, generally for system events, and
* a **session** bus (one for each user session) for communications between user applications.

A process can connect to any bus it's been granted access to, normally to the system bus and to the current session bus. Different applications, or even different components of the same application, can share and propagate data using D-Bus. In this series, we're going to see how Bluetooth events can be handled by user applications to provide a fine-grained control over the incoming pairing requests.

Normally, applications interact with a D-Bus daemon, which poses as a server for the application itself. In this context, an **address** specifies where a server will be listening to and where a client is going to connect. When using the D-Bus daemon, the D-Bus library (`libdbus`) automatically detects the addresses of both the per-session daemon and the system daemon. Applications are registered on a bus with a unique **service name**, that is used to route messages from one application to another. Service names are very close in meaning and format to network hostnames: a dot-separated sequence of alphanumeric characters, such as `org.bluez`. 

Processes expose their services by means of _objects_, which in turn have methods and signals. Their meaning is as in usual event-driven object-oriented systems. Methods can be called to make an object perform a certain action, whereas signals indicate that the state of an object changed. For instance, in the Bluetooth context, we could invoke a method on an object representing a connected device to obtain its Bluetooth address, whereas a new signal can be emitted whenever a new device is connected. Each object is uniquely identified by an **object path**, a sequence of alphanumeric characters separated and prefixed (but not terminated) by slashes. The underscore is also a valid character. For example, `org/example/dev_00_00_00_00_00_00` is a valid object path. Objects also have **interfaces** specifying their contract: each object supports one or more interfaces.

To conclude, in order to invoke a method on an object instance, a number of elements have to be specified:
* `Address`, where to find the daemon;
* `Bus name`, the system one or the per-session one;
* `Object path`, where to find the object;
* `Interface`, the type of the object;
* `Method`, the actual action to be performed.

For a more detailed view of D-Bus refer to [\[4\]](#4).

<a name="qtdbus"></a>
### Qt D-Bus
Qt D-Bus is a Qt library to interact with D-Bus by means of the so-called [_Adaptors_](https://doc.qt.io/qt-5/usingadaptors.html). Adaptors are basically dispatchers of messages and bridges between D-Bus messages and Qt classes handling those messages. There are two ways to use an adaptor:
1. Creating a class which inherits `QDBusAbstractAdaptor`, specifying the macro `Q_OBJECT`. The class must contain one, and only one, `Q_CLASSINFO` entry with the `D-Bus Interface` name, declaring which interface it is exporting. The adaptor then declares one slot for each D-Bus message to be supported. Hence, in the end this class will contain at least one slot for each method contained in the D-Bus object's interface.
2. If you already have an xml file describing the interface of the D-Bus object, an useful tool named `qdbusxml2cpp` can be used. Given an xml file it generates a suitable adaptor for that interface. The actual implementation will simply delegate the method invocation using `QMetaObject`'s `invokeMethod`. This is the approach we're going to take in this post.

As a final note, the [`QDBusArgument`](https://doc.qt.io/qt-5/qdbusargument.html) class is used to marshall and demarshall D-bus arguments, allowing user programs to send and receive practically every C++ type. 

<a name="bluezagent"></a>
## BlueZ Agent API
The API for Bluetooth agents in the BlueZ stack is available at [\[5\]](#5). The first interface, `org.bluez.AgentManager1`, provides methods to register and unregister agents (only one agent can be registered at a time). An object implementing this interface is already available at the path `/org/bluez`, so we won't have to implement these methods. On the other hand, we'll need to implement those of the interface `org.bluez.Agent1`. The latter contains many methods, each of which is invoked according to the actual agent's capability. For example, for "NoInputNoOutput" agents, you can limit you worries to `RequestConfirmation` and `AuthorizeService`. The former is called when a device attempts to pair, whereas the latter is called for each service (i.e. Bluetooth profile) requested by the device. 

As we're to let `qdbusxml2cpp` generate the adaptor, we're going to need an xml file containing the actual Agent interface:
```xml
<!DOCTYPE node PUBLIC "-//freedesktop//DTD D-BUS Object Introspection 1.0//EN"
"http://www.freedesktop.org/standards/dbus/1.0/introspect.dtd">
<node>
    <interface name="org.bluez.Agent1">
        <method name="Release" />
        <method name="RequestPinCode">
            <arg direction="in" type="o" />
            <arg direction="out" type="s" />
        </method>
        <method name="DisplayPinCode">
            <arg direction="in" type="o" />
            <arg direction="in" type="s" />
        </method>
        <method name="RequestPasskey">
            <arg direction="in" type="o" />
            <arg direction="out" type="u" />
        </method>
        <method name="DisplayPasskey">
            <arg direction="in" type="o" />
            <arg direction="in" type="u" />
            <arg direction="in" type="q" />
        </method>
        <method name="RequestConfirmation">
            <arg direction="in" type="o" />
            <arg direction="in" type="u" />
        </method>
        <method name="RequestAuthorization">
            <arg direction="in" type="o" />
        </method>
        <method name="AuthorizeService">
            <arg direction="in" type="o" />
            <arg direction="in" type="s" />
        </method>
        <method name="Cancel" />
    </interface>
</node>
```
The interface above describes, for each method, its pts parameters. Along with the direction (`in` = input, `out` = output) is the type: 
* `o` represents an object, and it's represented as a `QDBusObjectPath` in Qt,
* `u` represents a 32-bit unsigned integer, and it's represented with a `uint` in Qt,
* `s` is a string, and translates into `QString`,
* `q` represents a 16-bit unsigned integer, and translates into `ushort`.

If you want to implement a fine-grained control over the allowed Bluetooth profiles, `AuthorizeService` is the method you need to look at. The first parameter contains the object path of the device asking for authorization, whereas the second one contains the ID of the service to authorize. A comprehensive list of IDs can be found [here](https://github.com/pauloborges/bluez/blob/master/lib/uuid.h).

`RequestConfirmation` is the method of main interest for "DisplayOnly" agents. Again, the first parameter represents the connecting device, whereas the second one now contains a randomly-generated passkey. This happens because we told BlueZ that our device can display stuff somewhere, even though it actually can't. As a matter of fact, it'll be enough to just ignore this parameter and implement in `RequestConfirmation` our authorization logic.

`RequestConfirmation` is also invoked for "DisplayYesNo" and "KeyboardDisplay" agents, whereas either `RequestPasskey` or `RequestPinCode`, depending on the actual device, is called for  "KeyboardOnly" capabilities. [\[5\]](#5) provides a detailed explanation of all those methods.

## Generating the adaptor
Given the interface above, generating the corresponding adaptor for Qt is as simple as executing the following command:
> qdbusxml2cpp -qt=qt5 org.bluez.Agent1.xml -a adaptor

The first parameter, `-qt`, indicates the major release of Qt that you want to use. `org.bluez.Agent1.xml` specifies the name of the xml file containing the interface, whereas `-a adaptor` tells `qdbusxml2cpp` the name of the cpp/h files to be generated. 

A closer look to this latter files gives us a lot of information. Let's start with the header file.
```c++
class Agent1Adaptor: public QDBusAbstractAdaptor
{
    Q_OBJECT
    Q_CLASSINFO("D-Bus Interface", "org.bluez.Agent1")
    Q_CLASSINFO("D-Bus Introspection", ""
"  <interface name=\"org.bluez.Agent1\">\n"
"    <method name=\"Release\"/>\n"
"    <method name=\"RequestPinCode\">\n"
"      <arg direction=\"in\" type=\"o\"/>\n"
"      <arg direction=\"out\" type=\"s\"/>\n"
"    </method>\n"
"    <method name=\"DisplayPinCode\">\n"
"      <arg direction=\"in\" type=\"o\"/>\n"
"      <arg direction=\"in\" type=\"s\"/>\n"
"    </method>\n"
"    <method name=\"RequestPasskey\">\n"
"      <arg direction=\"in\" type=\"o\"/>\n"
"      <arg direction=\"out\" type=\"u\"/>\n"
"    </method>\n"
"    <method name=\"DisplayPasskey\">\n"
"      <arg direction=\"in\" type=\"o\"/>\n"
"      <arg direction=\"in\" type=\"u\"/>\n"
"      <arg direction=\"in\" type=\"q\"/>\n"
"    </method>\n"
"    <method name=\"RequestConfirmation\">\n"
"      <arg direction=\"in\" type=\"o\"/>\n"
"      <arg direction=\"in\" type=\"u\"/>\n"
"    </method>\n"
"    <method name=\"RequestAuthorization\">\n"
"      <arg direction=\"in\" type=\"o\"/>\n"
"    </method>\n"
"    <method name=\"AuthorizeService\">\n"
"      <arg direction=\"in\" type=\"o\"/>\n"
"      <arg direction=\"in\" type=\"s\"/>\n"
"    </method>\n"
"    <method name=\"Cancel\"/>\n"
"  </interface>\n"
        "")
public:
    Agent1Adaptor(QObject *parent);
    virtual ~Agent1Adaptor();

public: // PROPERTIES
public Q_SLOTS: // METHODS
    void AuthorizeService(const QDBusObjectPath &in0, const QString &in1);
    void Cancel();
    void DisplayPasskey(const QDBusObjectPath &in0, uint in1, ushort in2);
    void DisplayPinCode(const QDBusObjectPath &in0, const QString &in1);
    void Release();
    void RequestAuthorization(const QDBusObjectPath &in0);
    void RequestConfirmation(const QDBusObjectPath &in0, uint in1);
    uint RequestPasskey(const QDBusObjectPath &in0);
    QString RequestPinCode(const QDBusObjectPath &in0);
Q_SIGNALS: // SIGNALS
};
```
`qdbusxml2cpp` created a class named `Agent1Adaptor`, where `Agent1` is the name of the D-Bus interface we used. As explained before, `Q_CLASSINFO` specifies the qualified name of such interface and shows its xml specification. Furthermore, `Agent1Adapter` contains one Qt slot for each method in the interface. Finally, the constructor inputs, as every `QObject`, a `QObject*` specifying its parent. This parameter is quite important, as it will point to the object implementing the interface.

As a matter of fact, the cpp file reveals how `Agent1Adaptor`'s slots behave: Qt simply forwards the invocation to its parent, using `QMetaObject::invokeMethod`. This means that the parent must be a `QObject` itself, and that the corresponding methods must either be known by Qt's meta-objects system. For instance, the implementation of `RequestConfirmation` is as follows:
```c++
void Agent1Adaptor::RequestConfirmation(const QDBusObjectPath &in0, uint in1)
{
    // handle method call org.bluez.Agent1.RequestConfirmation
    QMetaObject::invokeMethod(parent(), "RequestConfirmation", Q_ARG(QDBusObjectPath, in0), Q_ARG(uint, in1));
}
```

## To be continued...
In this first part we learned a bunch of theoretical stuff about Qt and D-Bus and took a look at the interface exposed by Bluetooth agents. In the next part we're going to see how an actual DisplayOnly agent can be implemented and registered on the system bus, diving into the Qt D-bus library.

This will provide us with a working example of a Bluetooth agent developed in Qu.


## Further reading
<a name="1"></a>[1] [BLE Pairing and Bonding](https://www.kynetics.com/docs/2018/BLE_Pairing_and_bonding/)

<a name="2"></a>[2] [Pairing Agents in BlueZ stack](https://www.kynetics.com/docs/2018/pairing_agents_bluez/)

<a name="3"></a>[3] [Qt Bluetooth](https://doc.qt.io/qt-5/qtbluetooth-index.html)

<a name="4"></a>[4] [D-Bus](https://dbus.freedesktop.org/doc/dbus-tutorial.html)

<a name="5"></a>[5] [BlueZ Agent documentation](https://github.com/pauloborges/bluez/blob/master/doc/agent-api.txt)