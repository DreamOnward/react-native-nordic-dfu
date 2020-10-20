# react-native-nordic-dfu 

## This fork contains add two timeouts to fix firmware uploads for iOS 13.
## It is the same fix as [this other fork](https://github.com/Jurpp/react-native-nordic-dfu) but on a later version of the trunk.


This library allows you to do a Device Firmware Update (DFU) of your nrf51 or
nrf52 chip from Nordic Semiconductor. It works for both iOS and Android.

For more info about the DFU process, see: [Resources](#resources)

## Installation

Remove react-native-nordic-dfu from your package.json and yarn.lock files and run:

```
yard add https://github.com/DreamOnward/react-native-nordic-dfu
```

or

```bash
yarn add react-native-nordic-dfu
```

For React Native below 60.0 version

```bash
react-native link react-native-nordic-dfu
```

### Minimum requirements

This project has been verified to work with the following dependencies, though other versions may work as well.

| Dependency   | Version |
| ------------ | ------- |
| React Native | 0.59.4  |
| XCode        | 10.2    |
| Swift        | 5.0     |
| CocoaPods    | 1.6.1   |
| Gradle       | 5.3.1   |

### iOS

The iOS version of this library has native dependencies that need to be installed via `CocoaPods`, which is currently the only supported method for installing this library. (PR's for alternative installation methods are welcome!)

Previous versions supported manual linking, but this was prone to errors every time a new version of XCode and/or Swift was released, which is why this support was dropped. If you've previously installed this library manually, you'll want to remove the old installation and replace it with CocoaPods.

#### CocoaPods

On your project directory;

```bash
cd ios && pod install
```

If your React Native version below 0.60 or any problem occures on pod command, you can try these steps;

Add the following to your `Podfile`

```ruby
target "YourApp" do

  ...
  pod "react-native-nordic-dfu", path: "../node_modules/react-native-nordic-dfu"
  ...

end
```

and in the same folder as the Podfile run

```bash
pod install
```

Since there's native Swift dependencies you need to set which Swift version your project complies with. If you haven't already done this, open up your project with XCode and add a User-Defined setting under Build Settings: `SWIFT_VERSION = <your-swift-version>`.

If your React Native version is higher than 0.60, probably it's already there.

#### Bluetooth integration

This library needs access to an instance of `CBCentralManager`, which you most likely will have instantiated already if you're using Bluetooth for other purposes than DFU in your project.

To integrate with your existing Bluetooth setup, call `[RNNordicDfu setCentralManagerGetter:<...>]` with a block argument that returns your `CBCentralManager` instance.

If you want control over the `CBCentralManager` instance after the DFU process is done you might need to provide the `onDFUComplete` and `onDFUError` callbacks to transfer back delegate control.

Example code;

```swift
...
...
#import "RNNordicDfu.h"
#import "BleManager.h"

@implementation AppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions{
  ...
  ...

    [RNNordicDfu setCentralManagerGetter:^() {
    return [BleManager getCentralManager];
  }];

  // Reset manager delegate since the Nordic DFU lib "steals" control over it
  [RNNordicDfu setOnDFUComplete:^() {
    NSLog(@"onDFUComplete");
    CBCentralManager * manager = [BleManager getCentralManager];
    manager.delegate = [BleManager getInstance];
  }];

  [RNNordicDfu setOnDFUError:^() {
    NSLog(@"onDFUError");
    CBCentralManager * manager = [BleManager getCentralManager];
    manager.delegate = [BleManager getInstance];
  }];

  return YES;
}

```

You can find them aslo in example project.

On iOS side this library requires to BleManager module which that [react-native-ble-manager](https://github.com/innoveit/react-native-ble-manager) provides.

It required because;

- You need `BleManager.h` module on AppDelegate file for integration.
- You should call `BleManager.start()` (for once) before the trigger a DFU process on iOS or you will get error like [this issue](https://github.com/Pilloxa/react-native-nordic-dfu/issues/82).

### Android

Android requires that you have `FOREGROUND_SERVICE` permissions.
You will need the following in your AndroidManifest.xml

```
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

## API

<!-- Generated by documentation.js. Update this documentation by updating the source code. -->

### startDFU

Starts the DFU process

Observe: The peripheral must have been discovered by the native BLE side so that the
bluetooth stack knows about it. This library will not do a scan but only
the actual connect and then the transfer. See the example project to see how it can be
done in React Native.

**Parameters**

- `obj` **[Object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)**
  - `obj.deviceAddress` **[string](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String)** The `identifier`\* of the device that should be updated
  - `obj.deviceName` **[string](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String)** The name of the device in the update notification (optional, default `null`)
  - `obj.filePath` **[string](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String)** The file system path to the zip-file used for updating
  - `obj.alternativeAdvertisingNameEnabled` **[boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Boolean)** Send unique name to device before it is switched into bootloader mode (iOS only) - defaults to `true`

\* `identifier` — MAC address (Android) / UUID (iOS)

**Examples**

```javascript
import { NordicDFU, DFUEmitter } from "react-native-nordic-dfu";

NordicDFU.startDFU({
  deviceAddress: "C3:53:C0:39:2F:99",
  deviceName: "Pilloxa Pillbox",
  filePath: "/data/user/0/com.nordicdfuexample/files/RNFetchBlobTmp4of.zip",
})
  .then((res) => console.log("Transfer done:", res))
  .catch(console.log);
```

Returns **[Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)** A promise that resolves or rejects with the `deviceAddress` in the return value

### DFUEmitter

Event emitter for DFU state and progress events

**Examples**

```javascript
import { NordicDFU, DFUEmitter } from "react-native-nordic-dfu";

DFUEmitter.addListener(
  "DFUProgress",
  ({ percent, currentPart, partsTotal, avgSpeed, speed }) => {
    console.log("DFU progress: " + percent + "%");
  }
);

DFUEmitter.addListener("DFUStateChanged", ({ state }) => {
  console.log("DFU State:", state);
});
```

## Selecting firmware file from local storage

If your user will select the firmware file from local storage you should keep on mind some issues;

You can use [react-native-document-picker](https://github.com/Elyx0/react-native-document-picker) library for file selecting process.

### On iOS

You should select file type as `public.archive` or you will get null type error as like [this issue](https://github.com/Pilloxa/react-native-nordic-dfu/issues/100)

```js
DocumentPicker.pick({ type: "public.archive" });
```

If your device getting disconnect after enable DFU, you should set `false` value to `alternativeAdvertisingNameEnabled` prop while starting DFU.

```js
NordicDFU.startDFU({
  deviceAddress: "XXXXXXXX-XXXX-XXXX-XXXX-XX",
  filePath: firmwareFile.uri,
  alternativeAdvertisingNameEnabled: false,
});
```

### On Android

Some Android versions directly selecting file may can cause errors. If you get any file error you should copy it to your local storage. Like cache directory.

You can use [react-native-fs](https://github.com/itinance/react-native-fs) for copying file.

```js
const firmwareFile = await DocumentPicker.pick({ type: DocumentPicker.types.zip })
const destination = RNFS.CachesDirectoryPath + "/firmwareFile.zip");

await RNFS.copyFile(formatFile.uri, destination);

NordicDFU.startDFU({ deviceAddress: "XX:XX:XX:XX:XX:XX", filePath: destination })
```

If you getting disconnect error sometimes while starting DFU process, you should connect the device before start it.

## Example project

Navigate to `example/` and run

```bash
npm install
```

Run the iOS project with

```bash
react-native run-ios
```

and the Android project with

```bash
react-native run-android
```

## Development

PR's are always welcome!

## Resources

- [DFU Introduction](http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.sdk5.v11.0.0/examples_ble_dfu.html?cp=6_0_0_4_3_1 "BLE Bootloader/DFU")
- [Secure DFU Introduction](http://infocenter.nordicsemi.com/topic/com.nordic.infocenter.sdk5.v12.0.0/ble_sdk_app_dfu_bootloader.html?cp=4_0_0_4_3_1 "BLE Secure DFU Bootloader")
- [How to create init packet](https://github.com/NordicSemiconductor/Android-nRF-Connect/tree/master/init%20packet%20handling "Init packet handling")
- [nRF51 Development Kit (DK)](http://www.nordicsemi.com/eng/Products/nRF51-DK "nRF51 DK") (compatible with Arduino Uno Revision 3)
- [nRF52 Development Kit (DK)](http://www.nordicsemi.com/eng/Products/Bluetooth-Smart-Bluetooth-low-energy/nRF52-DK "nRF52 DK") (compatible with Arduino Uno Revision 3)

## Sponsored by

[![pilloxa](https://pilloxa.com/images/pilloxa-round-logo.svg)](https://pilloxa.com)
