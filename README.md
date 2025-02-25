[![Pub Package](https://img.shields.io/pub/v/dbus.svg)](https://pub.dev/packages/dbus)
[![codecov](https://codecov.io/gh/canonical/dbus.dart/branch/main/graph/badge.svg?token=rk7NBXldfn)](https://codecov.io/gh/canonical/dbus.dart)

A native Dart client implementation of [D-Bus](https://www.freedesktop.org/wiki/Software/dbus/).

## Accessing remote objects

The easiest way to get started is to generate Dart classes from a D-Bus interface definition, for example:

```xml
<node name="/com/example/Test/Object">
  <interface name="com.example.Test">
    <property name="Version" type="s" access="read"/>
    <method name="ReverseText">
      <arg name="input" type="s" direction="in"/>
      <arg name="output" type="s" direction="out"/>
    </method>
  </interface>
</node>
```

The *dart-dbus* tool processes this interface to generate a Dart source file:

```shell
$ dart-dbus generate-remote-object test-object.xml -o test-remote-object.dart
```

You can then use the generated `test-remote-object.dart` to access that remote object from your program:

```dart
import 'package:dbus/dbus.dart';
import 'test-remote-object.dart';

Future<void> main() async {
  var client = DBusClient.session();
  var object = ComExampleTestObject(client, 'com.example.Test');
  var version = await object.getVersion();
  print('version: $version');
  var reversedText = await object.callReverseText('Hello World');
  print(reversedText);
  await client.close();
}
```

The code generated by *dart-dbus* may not be the cleanest API you want to expose for your service. It is recommended that you use the generated code as a starting point and then modify it as necessary.

## Exporting local objects

The above interface definition can also be processed to generate a class for exporting this object.

```shell
$ dart-dbus generate-object test-object.xml -o test-object.dart
```

You can then use the generated `test-object.dart` in your program:

```dart
import 'package:dbus/dbus.dart';
import 'test-object.dart';

class TestObject extends ComExampleTestObject {
  @override
  Future<DBusMethodResponse> getVersion() async {
    return DBusGetPropertyResponse(DBusString('1.2'));
  }

  @override
  Future<DBusMethodResponse> doReverseText(String input) async {
    var reversedText = String.fromCharCodes(input.codeUnits.reversed);
    return DBusMethodSuccessResponse([DBusString(reversedText)]);
  }
}

Future<void> main() async {
  var client = DBusClient.session();
  await client.requestName('com.example.Test');
  await client.registerObject(TestObject());
}
```

## Using objects without *dbus-dart*

You can also choose to access D-Bus objects directly without interface definitions and *dbus-dart*.
This requires writing more code and handing more error cases.

The following code shows how to access the remote object in the above examples:

```dart
import 'package:dbus/dbus.dart';

Future<void> main() async {
  var client = DBusClient.session();
  var object = DBusRemoteObject(client, name: 'com.example.Test', path: DBusObjectPath('/com/example/Test/Object'));
  var value = await object.getProperty('com.example.Test', 'Version', signature: DBusSignature('s'));
  var version = (value as DBusString).value;
  print('version: $version');
  var response = await object.callMethod('com.example.Test', 'ReverseText', [DBusString('Hello World')], replySignature: DBusSignature('s'));
  var reversedText = (response.values[0] as DBusString).value;
  print('$reversedText');
  await client.close();
}
```

And the following shows how to export that object:

```dart
import 'package:dbus/dbus.dart';

class TestObject extends DBusObject {
  TestObject() : super(DBusObjectPath('/com/example/Test/Object'));

  @override
  Future<DBusMethodResponse> getProperty(String interface, String name) async {
    if (interface == 'com.example.Test' && name == 'Version') {
      return DBusGetPropertyResponse(DBusString('1.2'));
    } else {
      return DBusMethodErrorResponse.unknownProperty();
    }
  }

  @override
  Future<DBusMethodResponse> handleMethodCall(DBusMethodCall methodCall) async {
    if (methodCall.interface == 'com.example.Test') {
      if (methodCall.name == 'ReverseText') {
        if (methodCall.signature != DBusSignature('s')) {
          return DBusMethodErrorResponse.invalidArgs();
        }
	var input = (methodCall.values[0] as DBusString).value;
        var reversedText = String.fromCharCodes(input.codeUnits.reversed);
        return DBusMethodSuccessResponse([DBusString(reversedText)]);
      } else {
        return DBusMethodErrorResponse.unknownMethod();
      }
    } else {
      return DBusMethodErrorResponse.unknownInterface();
    }
  }
}

Future<void> main() async {
  var client = DBusClient.session();
  await client.requestName('com.example.Test');
  await client.registerObject(TestObject());
}
```

## Contributing to dbus.dart

We welcome contributions! See the [contribution guide](CONTRIBUTING.md) for more details.
