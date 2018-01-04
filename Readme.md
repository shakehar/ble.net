<img src="http://public.nexussays.com/ble.net/logo_256x256.png" width="128" height="128" />

# ble.net ![Build status](https://img.shields.io/vso/build/nexussays/ebc6aafa-2931-41dc-b030-7f1eff5a28e5/7.svg?style=flat-square) [![NuGet](https://img.shields.io/nuget/vpre/ble.net.svg?style=flat-square)](https://www.nuget.org/packages/ble.net) [![MPLv2 License](https://img.shields.io/badge/license-MPLv2-blue.svg?style=flat-square)](https://www.mozilla.org/MPL/2.0/) [![API docs](https://img.shields.io/badge/apidocs-DotNetApis-blue.svg?style=flat-square)](http://dotnetapis.com/pkg/ble.net)

`ble.net` is a Bluetooth Low Energy (aka BLE, aka Bluetooth LE, aka Bluetooth Smart) cross-platform library to enable simple development of BLE clients on Android, iOS, and UWP/Windows.

It provides a consistent API across all supported platforms and hides many of the poor API decisions of each respective platform.

You can make multiple simultaneous BLE requests on Android without worrying that some calls will silently fail. You can simply `await` all your calls without dealing with the book-keeping of an event-based system. If you know which characteristics and services you wish to interact with, then you can just read/write to them without having to query down into the device's attribute heirarchy and retain references to these characteristics and services. And so on, and so on...

> Note: Currently UWP only supports listening for broadcasts/advertisements, not connecting to devices. The UWP BLE API is... proving difficult.

### [These projects are using BLE.net](https://github.com/nexussays/ble.net/wiki/Showcase)

## Getting Started

### 1. Install NuGet packages

Install the `ble.net (API)` package in your (PCL/NetStandard) shared library
```powershell
Install-Package ble.net -Version 1.0.0-beta0008 
```

If you are making a library or are implementing support for a new platform, you are done.

If you are making an app, then install the relevant package in each platform project:

```powershell
Install-Package ble.net-android -Version 1.0.0-beta0008 
```
```powershell
Install-Package ble.net-ios -Version 1.0.0-beta0008 
```
```powershell
Install-Package ble.net-uwp -Version 1.0.0-beta0008 
```

### 2. Add relevant app permissions

#### Android

```csharp
[assembly: UsesPermission( Manifest.Permission.Bluetooth )]
[assembly: UsesPermission( Manifest.Permission.BluetoothAdmin )]
```

> If you are having issues discovering devices when scanning, try adding coarse location permissions.
> ```csharp
> [assembly: UsesPermission( Manifest.Permission.AccessCoarseLocation )]
> ```

#### iOS

Add a description string to display to the user indicating you want to use Bluetooth.

```xml
<!-- Info.plist -->
<key>NSBluetoothPeripheralUsageDescription</key>
<string>[MyAppNameHere] would like to use bluetooth.</string>
```

If you want to connect to peripherals in the background, add the [`bluetooth-central`](https://developer.apple.com/library/content/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html#//apple_ref/doc/uid/TP40013257-CH7-SW6) background mode.

```xml
<!-- Info.plist -->
<key>UIBackgroundModes</key>
<array>
   <string>bluetooth-central</string>
</array>
```

#### UWP

```xml
<!-- Package.appxmanifest -->
<Capabilities>
   <DeviceCapability Name="bluetooth" />
</Capabilities>
```

### 3. Obtain a reference to `BluetoothLowEnergyAdapter`

Each platform project has a static method `BluetoothLowEnergyAdapter.ObtainDefaultAdapter()` with various overloads. Obtain this reference and then provide it to your application code using your dependency injector, or manual reference passing, or a singleton, or whatever strategy you are using in your project.

Examples:
* [Android Xamarin.Forms](src/ble.net.sampleapp-android/MyApplication.cs#L108)
* [iOS Xamarin.Forms](src/ble.net.sampleapp-ios/MyApplication.cs#L59)
* [UWP Xamarin.Forms](src/ble.net.sampleapp-uwp/MainPage.xaml.cs#L12)

#### Android-specific setup

If you want `IBluetoothLowEnergyAdapter.State.DisableAdapter()` and `EnableAdapter()` to work, in your main `Activity`:
```csharp
protected override void OnCreate( Bundle bundle )
{
   // ...
   BluetoothLowEnergyAdapter.InitActivity( this );
   // ...
}
```

If you want `IBluetoothLowEnergyAdapter.State.Subscribe()` to work, in your calling `Activity`:
```csharp
protected sealed override void OnActivityResult( Int32 requestCode, Result resultCode, Intent data )
{
   BluetoothLowEnergyAdapter.OnActivityResult( requestCode, resultCode, data );
}
```

## API

> See [sample Xamarin Forms app](/src/ble.net.sampleapp/) included in the repo for a complete app example.

All the examples presume you have some `adapter` passed in as per the setup notes above:
```csharp
IBluetoothLowEnergyAdapter adapter = /* platform-provided adapter from BluetoothLowEnergyAdapter.ObtainDefaultAdapter()*/;
```

### Control the Bluetooth Adapter on the device

#### Enable Bluetooth

> There are corresponding methods to disable the adapter.

```csharp
if(adapter.AdapterCanBeEnabled && adapter.CurrentState.Value == EnabledDisabledState.Disabled) {
   await adapter.EnableAdapter();
}
```

#### See/Watch Adapter Status

```csharp
Debug.WriteLine(adapter.CurrentState.Value); // EnabledDisabledState.Enabled
adapter.CurrentState.Subscribe( state => Debug.WriteLine("New State: {0}", state) );
```

### Scan for broadcast advertisements

```csharp
await adapter.ScanForBroadcasts(
   // Optional scan filter to ensure that the
   // observer will only receive peripherals
   // that pass the filter. If you want to scan
   // for everything around, omit this argument.
   new ScanFilter()
      .SetAdvertisedDeviceName( "foobar" )
      .SetAdvertisedManufacturerCompanyId( 76 )
      // Discovered peripherals must advertise at-least-one
      // of any GUIDs added by AddAdvertisedService()
      .AddAdvertisedService( guid )
      .SetIgnoreRepeatBroadcasts( false ),
   // IObserver<IBlePeripheral> or Action<IBlePeripheral>
   // will be triggered for each discovered peripheral
   // that passes the above can filter (if provided).
   ( IBlePeripheral peripheral ) =>
   {
      // read the advertising data...
      var adv = peripheral.Advertisement;
      Debug.WriteLine( adv.DeviceName );
      Debug.WriteLine( adv.Services.Select( x => x.ToString() ).Join( "," ) );
      Debug.WriteLine( adv.ManufacturerSpecificData.FirstOrDefault().CompanyName() );
      Debug.WriteLine( adv.ServiceData );

      // ...or connect to the device (see next example)...
   },
   // TimeSpan or CancellationToken to stop the scan
   TimeSpan.FromSeconds( 30 )
   // If you omit this argument, it will use
   // BluetoothLowEnergyUtils.DefaultScanTimeout
);

// scanning has been stopped when code reached this point
```

For the common case of ignoring duplicate advertisements (i.e., repeated advertisements from the same device), there is a static `ScanFilter.UniqueBroadcastsOnly` you can use as the scan filter.

You can also create a `ScanFilter` using an object initializer if you prefer that syntax:
```csharp
new ScanFilter()
{
   AdvertisedDeviceName = "foobar",
   AdvertisedManufacturerCompanyId = 76,
   AdvertisedServiceIsInList = new List<Guid>(){ guid },
   IgnoreRepeatBroadcasts = true
}
```

### Connect to a BLE device

If you just want to connect to a specific device, you can do so without manually scanning:
```csharp
var connection = await adapter.FindAndConnectToDevice(
   new ScanFilter()
      .SetAdvertisedDeviceName( "foo" )
      .SetAdvertisedManufacturerCompanyId( 0xffff )
      .AddAdvertisedService( guid ),
   TimeSpan.FromSeconds( 30 ) );
```

If you have already scanned and discovered a peripheral you want to connect to:
```csharp
var connection = await adapter.ConnectToDevice(
   // The IBlePeripheral to connect to
   peripheral,
   // TimeSpan or CancellationToken to stop the
   // connection attempt.
   // If you omit this argument, it will use
   // BluetoothLowEnergyUtils.DefaultConnectionTimeout
   TimeSpan.FromSeconds( 15 ),
   // Optional IProgress<ConnectionProgress>
   progress => Debug.WriteLine(progress)
);

if(connection.IsSuccessful())
{
   var gattServer = connection.GattServer;
   // do things with gattServer here... (see later examples...)
}
else
{
   // Do something to inform user or otherwise handle unsuccessful connection.
   Debug.WriteLine( "Error connecting to device. result={0:g}", connection.ConnectionResult );
   // e.g., "Error connecting to device. result=ConnectionAttemptCancelled"
}
```

### Get descriptions for GATT GUIDs

You can provide information for the GUIDs representing services, characteristics, and descriptors with `KnownAttributes`.

```csharp
var known = new KnownAttributes();
// You can include attributes that have
// been adopted by the Bluetooth SIG.
known.AddAdoptedServices();
known.AddAdoptedCharacteristics();
known.AddAdoptedDescriptors();
// You can add descriptions for attributes
// that aren't officially adopted.
known.AddService( guid1, "foo" );
known.AddCharacteristic( guid2, "bar" );
known.AddDescriptor( guid3, "baz" );
```

### Enumerate all services on the GATT Server

```csharp
foreach(var guid in await gattServer.ListAllServices())
{
   Debug.WriteLine( $"service: {guid} \"{known.Get(guid)?.Description}\"" );
}
```

### Enumerate all characteristics of a service

```csharp
Debug.WriteLine( $"service: {serviceGuid}" );
foreach(var guid in await gattServer.ListServiceCharacteristics( serviceGuid ))
{
   Debug.WriteLine( $"characteristic: {guid} \"{known.Get(guid)?.Description}\"" );
}
```

### Read a characteristic

```csharp
try
{
   var value = await gattServer.ReadCharacteristicValue( someServiceGuid, someCharacteristicGuid );
}
catch(GattException ex)
{
   Debug.WriteLine( ex.ToString() );
}
```

### Listen for notifications on a characteristic

```csharp
IDisposable notifier;

try
{
   // Will also stop listening when gattServer
   // is disconnected, so if that is acceptable,
   // you don't need to store this disposable.
   notifier = gattServer.NotifyCharacteristicValue(
      someServiceGuid,
      someCharacteristicGuid,
      // IObserver<Tuple<Guid, Byte[]>> or IObserver<Byte[]> or
      // Action<Tuple<Guid, Byte[]>> or Action<Byte[]>
      bytes => {/* do something with notification bytes */} );
}
catch(GattException ex)
{
   Debug.WriteLine( ex.ToString() );
}

// ... later, once done listening for notifications ...
notifier.Dispose();
```

### Write to a characteristic

```csharp
try
{
   // The resulting value of the characteristic is returned. In nearly all cases this
   // will be the same value that was provided to the write call (e.g. `byte[]{ 1, 2, 3 }`)
   var value = await gattServer.WriteCharacteristicValue(
      someServiceGuid,
      someCharacteristicGuid,
      new byte[]{ 1, 2, 3 } );
}
catch(GattException ex)
{
   Debug.WriteLine( ex.ToString() );
}
```

### Do a bunch of things

> If you've used the native BLE APIs on Android or iOS, hopefully you appreciate the simplicity here :)

```csharp
try
{
   var read = gattServer.ReadCharacteristicValue( service, char1 );
   await
      Task.WhenAll(
         new Task[]
         {
            gattServer.WriteCharacteristicValue(
               service, char1, new Byte[]{/* bytes */}
            ),
            gattServer.WriteCharacteristicValue(
               service, char2, new Byte[]{/* bytes */}
            ),
            gattServer.WriteCharacteristicValue(
               service2, char3, new Byte[]{/* bytes */}
            ),
            gattServer.WriteCharacteristicValue(
               service2, char4, new Byte[]{/* bytes */}
            ),
         } );
   // Our read will have executed first, so we will
   // have the value prior to the write.
   return await read;
}
catch(GattException ex)
{
   Debug.WriteLine( ex.ToString() );
}
```
