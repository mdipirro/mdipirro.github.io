---
layout: post
title:  "Developing a DisplayOnly Bluetooth agent in Qt/C++ with BlueZ and D-Bus - Part 2"
desc: "Blog"
keywords: "bluetooth, qt, c++, cpp, embedded, bluetooth agent, bluez, dbus"
date: 2019-09-30
categories: C++
tags: [bluetooth, qt, c++, embedded, bluetooth agent, bluez, dbus]
icon: fa-bookmark-o
---
**Welcome back to this two-part series about Bluetooth, Qt and C++.** In the previous [post](https://mdipirro.github.io/c++/2019/09/18/bluetooth-agent-qt-part1.html) we went through some concepts about Qt, BlueZ and D-Bus. In today's part we're going to dive into the actual implementation of our agent, and develop a basic application that allows pairing only with whitelisted devices.

## Implementing the interface
In this section I'm going to show you a simple, yet powerful, templated implementation of an agent equipped with a validation function (a predicate over a number of typed arguments). Although normally the validation would depend on the device asking to pair (or, more precisely, on its `QDBusObjectPath`), I won't include this restriction at this level. Instead, we're going to implement this logic into a concrete Agent.

The snippets of code below use some features of C++17, in particular [inline variables](https://en.cppreference.com/w/cpp/language/inline). They also use [variadic templates](https://en.cppreference.com/w/cpp/language/parameter_pack). The latter is quite useful to make the validation function as generic as possible, whereas the former just comes handy to initialize static variables in a templated class. 

We shall define two classes. First, `PairingAgentWithValidation` will contain the validation function and define the methods to be implemented. Secondly, `BTMACAddressAgent` will define a class where the validation occurs in terms of Bluetooth MAC addresses. Paring will be authorized if and only if the current device's MAC address was previously whitelisted.

Abstract class `PairingAgentWithValidation` is defined as a templated class over a parameter pack:
```c++
template <class... Args>
class PairingAgentWithValidation : public QObject
```
Note the three dots after `Args`: they indicate that `PairingAgentWithValidation` accepts a variable number of templated arguments, where this number is not known yet when the class is defined. This allows for an unbelievable flexibility. It also extends `QObject`, even though we can't use the `Q_OBJECT` macro in a templated class. This is a limitation in Qt's meta-objects system: the meta-object compiler, `moc`, requires the API of a class to be completely defined by its definition. As a templated class is not fully defined without an actual instantiation, there can be an arbitrary number of instantiations of the same class differing only for the type parameter. Furthermore, all of them would require a different meta-object, and this would be extremely complex to implement, although theoretically possible. Hence, our `PairingAgentWithValidation` cannot contain slots, and this is quite a problem as, as we saw, `Agent1Adaptor` requires parent's method to be known by Qt's meta-objects system. We'll see how to get rid of this limitation soon. 

`PairingAgentWithValidation` contains a single private member, `validationFunction`, whose type is `std::function<bool(Args...)>`, that is, a function taking an arbitrary number of templated parameters and returning a bool. It also contains a couple of protected members:
* a validation method invoking the validation function:
```c++
bool validatePairing(const Args&... args) const {
    return validationFunction(args...);
}
```
Note that `validatePairing` inputs the actual arguments as constant references, so that it is guaranteed that it is a pure predicate (i.e. there will be no side-effects on `args`). Also note that `args` actually needs to be _expanded_ with the three dots in order to be used. This tells the runtime to input `validationFunction` with all the parameters contained in `args`, while checking that the all the types match. This method contains all the magic and encapsulates the validation logic.
* two static constants, `REJECTED` and `CANCELED`, set to the corresponding BlueZ's error objects, `org.bluez.Error.Rejected` and `org.bluez.Error.Canceled`, respectively. They are initialized as follows:
```cpp
static const inline QString REJECTED{"org.bluez.Error.Rejected"};
static const inline QString CANCELED{"org.bluez.Error.Canceled"};
```
C++17's inline variables comes to the rescue here to let us initialize such constants directly in the header file, without having to do it outside the class definition. 

Lastly, `PairingAgentWithValidation` defines a virtual method without implementation for each method in the Agent1 interface. For example, `RequestConfirmation` is defined as follows:
```cpp
virtual void RequestConfirmation(const QDBusObjectPath& device, uint passkey) const = 0;
```

`BTMACAddressAgent` extends `PairingAgentWithValidation` providing a validation logic based on whitelisting. It contains a list of authorized MAC addresses and checks the MAC address of the device trying to pair against this list. The most natural signature for the validation function is `validationFunction(QDBusObjectPath, QList<QString>)`. This leads us to the initialization of `PairingAgentWithValidation`'s template (we're going to add a little refinement to `BTMACAddressAgent`'s inheritance list soon):
```cpp
class BTMACAddressAgent : public PairingAgentWithValidation<QDBusObjectPath, QList<QString>>
```
Here we used the parameters to our still-to-be-implemented validation function as type parameters for the template. Hence, `PairingAgentWithValidation`'s `validationFunction`'s type will be set to `std::function<bool(QDBusObjectPath, QList<QString>)`. This is exactly what we wanted.

Another detail worthy to be noted is the definition of the interface methods. Remember that they couldn't be declared as slots in `PairingAgentWithValidation`? Well, Qt actually provides us with another way to make methods known to its meta-object system: the macro `Q_INVOKABLE`. Declaring `BTMACAddressAgent`'s methods as `Q_INVOKABLE` enables them to be invoked using `QMetaObject::invokeMethod`, solving the issue with slots we fell into.

I won't bother you with the full definition of `BTMACAddressAgent` here, but instead I'll explain just the main points. You can find a link to the full implementation at the end of this post. 

Let's focus on where the magic happens, the constructor of `BTMACAddress`:
```cpp
BTMACAddressAgent::BTMACAddressAgent(QObject *parent): PairingAgentWithValidation<QDBusObjectPath, QList<QString>>(
    [](const QDBusObjectPath& device, const QList<QString>& authDevices) {

        auto getDeviceMACAddress = [](const QString& devicePath) {
            QDBusInterface bluezInterface{"org.bluez", devicePath, "org.freedesktop.DBus.Properties", QDBusConnection::systemBus()};
            QDBusReply<QVariant> reply{bluezInterface.call("Get", "org.bluez.Device1", "Address")};
            return reply.value().toString();
        };

        return authDevices.contains(getDeviceMACAddress(device.path()));
    },
    parent
) {}
```
As usual, the initialization list invokes the constructor of the parent class. What's important here is the definition of the validation function, that can be expressed as a lambda expression. The first line, `[](const QDBusObjectPath& device, const QList<QString>& authDevices)`, defines the signature, capturing no variables from the outside. As we saw, types are statically checked here, and the compiler ensures that the parameters in this lambda match those we wrote in the inheritance list (`public PairingAgentWithValidation` `<QDBusObjectPath, QList<QString>>`). If we used `QString` as a first parameter, for example, the compiler would fail saying "error: no matching constructor for initialization of `PairingAgentWithValidation<QDBusObjectPath, QList<QString> >`". Finally, validating a paring is as simple as checking that `authDevices` (our whitelist) contains the MAC address of the pairing device.

There's one last thing to clarify in this implementation, the helper function `getDeviceMACAddress`, which is defined as a lambda inside the validation function. This is a common practice in functional programming, and, although C++'s not a functional language in the general meaning, we can actually leverage some of its functional features. `getDeviceMACAddress` simply inputs a `QString` containing the device path (in the form "[variable prefix]/{hci0,hci1,...}/dev_XX_XX_XX_XX_XX_XX") and returns the MAC address of that device. To make it more readable I omitted the error handling code, which is instead shown at the end of this post. To further simplify the declaration, I let C++ infer the type of the function with the keyword `auto` (introduced in C++11). Just for you to know, such type is `std::function<QString(const QString&)>`.

For the first time we see Qt D-Bus functions in action. The first step to interact with D-Bus is creating a `QDBusInterface`, a generic class used to dynamically access remote objects for which we do not have any code. This is exactly the case here, as we are to access the object corresponding to the pairing device, with which, as a matter of fact, we cannot interact with "usual" function calls. The constructor of `QDBusInterface` reflects the main components of D-Bus objects:
* the service, "org.bluez" in this case
* the path of the object we want to interact with
* its interface
* the type of the connection, that is, the bus where the object relies (session or system).

This `QDBusInterface` object precisely describes the actual D-Bus object we want to interact with: whom it belongs to (i.e. the service), where to find it (i.e. the path), the subset of operations we're allowed to perform on it (i.e. the interface), and how to connect to it (i.e. on which D-Bus bus).
We then have to perform the actual "function call", using the method `call()`, which basically constructs the message, sends it over the bus and decodes the reply once it arrives. It inputs the name of the function to call and a list of parameters. In this case we're invoking the function `Get`, defined in the interface [`org.freedesktop.DBus.Properties`](https://dbus.freedesktop.org/doc/dbus-specification.html#standard-interfaces-properties), which requires two parameters: the interface defining the property we want to access (`org.bluez.Device1`, representing a BlueZ device) and the name of such property (`Address`, containing the Bluetooth address).

Last but not least, we have to implement `RequestConfirmation`. Its implementation is now surprisingly simple: 
```cpp
void BTMACAddressAgent::RequestConfirmation(const QDBusObjectPath& device, uint) const {
    if(!validatePairing(device, authDevices)) Release();
}
```
`RequestConfirmation` simply relies on the validation function to check whether the device is authorized. Remember that `validatePairing` was defined in `PairingAgentWithValidation`? Its body simply invokes the validation function that `BTMACAddressAgent` specified in its constructor. There's nothing to do if the device is whitelisted: the pairing flow will continue as if nothing happened. If, on the other hand, that device is not authorized, `RequestConfirmation` calls `Release()` to stop the ongoing pairing:
```cpp
void BTMACAddressAgent::Release() const {
    sendErrorReply(REJECTED, "Pairing rejected");
}
```
Remember the refinement in `BTMACAddressAgent` I spoke about before? Here it comes into play. You might be wondering where `sendErrorReply` comes from and what its purpose is. It is defined by `QDBusContext` and simply sends an error reply along with an error message. To have it available in our agent, however, `BTMACAddressAgent` must also inherit from `QDBusContext`. Hence, the complete declaration of the agent is the following:
```cpp
class BTMACAddressAgent : public PairingAgentWithValidation<QDBusObjectPath, QList<QString>>, protected QDBusContext
```

The last thing we need to implement is the `main()`:
```c++
int main(int argc, char *argv[]) {
    BTMACAddressAgent agent{"00:00:00:00:00:00", "01:01:01:01:01:01"};
    QString objectPath{"/com/mdipirro/agent"};

    new Agent1Adaptor(&agent);

    if (QDBusConnection::systemBus().registerObject(objectPath, &agent)) {
        QDBusInterface agentManager{"org.bluez", "/org/bluez", "org.bluez.AgentManager1", QDBusConnection::systemBus()};
        QVariant agentPath{QVariant::fromValue(QDBusObjectPath(objectPath))};
        agentManager.call("RegisterAgent", agentPath, "DisplayOnly");
        agentManager.call("RequestDefaultAgent", agentPath);
    }

    return QCoreApplication(argc, argv).exec();
}
```
The first thing that `main` does is initializing a `BTMACAddressAgent` with a whitelist of devices. The initialization takes advantage of C++11's initializer lists, that are very similar in meaning to Java's _varargs_: we can pass a comma-separated list of values that is passed as an argument to `BTMACAddressAgent`'s **sequence constructor**: 
```c++
BTMACAddressAgent::BTMACAddressAgent(const std::initializer_list<QString>& authDevices): BTMACAddressAgent() {
    this->authDevices = authDevices;
}
```
The constructor first calls the empty constructor we saw before and then initializes its internal white`QList`, leveraging the compatibility between Qt containers and std initializers.

`new Agent1Adaptor(&agent);` instantiates a new Adaptor and links it with our custom agent. Adaptors must be initialized with _new_ and **not** be deleted, as they will be deleted automatically as soon as the object they are connected to is deleted. We then register our custom agent (and **not** the adaptor) on the system bus with a custom path. As before, I'll omit the error handling code to focus on the main operations, but you can find the complete example at the end of this post.

To register our agent we first have to create an instance of the `org.bluez.AgentManager1` D-Bus interface and a `QVariant` representing the path we registered our agent at. It is necessary to use a `QVariant` instead of a `QString`, as QDBus' type system requires a `QDBusObjectPath` for object paths, and we have to wrap it into a more generic type. 

Finally, we perform two calls on the `AgentManager1` interface. First we invoke `RegisterAgent`, to let BlueZ know that we want to use our own agent. Note how we specify **DisplayOnly** as a capability. Secondly, we invoke `RequestDefaultAgent`, to have BlueZ use our `BTMACAddressAgent` by default.

And that's it! In a couple of classes we developed a full-working D-Bus agent authorizing parings based on a whitelist. You can of course implement the validation logic that you want, but I hope this example helped you put your mind around this interesting topic.

## Conclusion
I hope you enjoyed this series about implementing a BlueZ agent with Qt, D-Bus and C++! [Here](https://github.com/mdipirro/bluez-agent-qt) you can find a working example implementing the logic we saw in these two posts.

See you soon!