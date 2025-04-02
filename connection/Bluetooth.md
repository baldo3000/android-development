# Bluetooth
## Intro
Most used classes and interface:
- `BluetoothAdapter`: physical Bluetooth adapter of the device.
- `BluetoothDevice`: represents a remote Bluetooth device.
- `BluetoothSocket`: similar to TCP socket, allows an app to exchange data with another Bluetooth device.
- `BluetoothServerSocket`: represents an open server socket that listens for incoming requests

Getting Bluetooth adapter:
``` kotlin
val bluetoothManager: BluetoothManager = getSystemService(BluetoothManager::class.java)
val bluetoothAdapter: BluetoothAdapter? = bluetoothManager.getAdapter()
if (bluetoothAdapter == null) {
  // Device doesn't support Bluetooth
}
```
### Enabling Bluetooth
Once Bluetooth is enabled on the device, you can use both Bluetooth Classic and Bluetooth Low Energy.
``` kotlin
if (bluetoothAdapter?.isEnabled == false) {
  val enableBtIntent = Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE)
  startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT) // REQUEST_ENABLE_BT must be an Int >= 0
  // Actually startActivityForResult() is deprecated
}
```

## Bluetooth Classic
### Find Bluetooth Devices
Because discoverable devices might reveal information about the user's location, the device discovery process requires location access.
On Android 8.0 and above you can use the Companion Device Manager API for discovery which does not require location permissions.
<br/>
<br/>
When you connect to a device for the first time, a pairing request it automatically sent the the other device.
When a device is paired, you can see the basics information about that device, such as the name and the MAC address, even if it's not connected. 
Using the MAC address you can initiate a connection.  
<br/>
<br/>
Getting list of paired devices:
``` kotlin
val pairedDevices: Set<BluetoothDevice>? = bluetoothAdapter?.bondedDevices
```
The discovery proess lasts about 12 seconds and is initiated with the call to `startDiscovery()`, which returns a boolean value indicating wether discovery was successfully started
To receive information about discovered devices, your app must register `BroadcastReceiver` for `ACTION_FOUND` intent, then extract device info from extra field `EXTRA_DEVICE`:
``` kotlin
override fun onCreate(savedInstanceState: Bundle?) {
   ...

   // Register for broadcasts when a device is discovered.
   val filter = IntentFilter(BluetoothDevice.ACTION_FOUND)
   registerReceiver(receiver, filter)
}

// Create a BroadcastReceiver for ACTION_FOUND.
private val receiver = object : BroadcastReceiver() {

   override fun onReceive(context: Context, intent: Intent) {
       val action: String = intent.action
       when(action) {
           BluetoothDevice.ACTION_FOUND -> {
               // Discovery has found a device. Get the BluetoothDevice
               // object and its info from the Intent.
               val device: BluetoothDevice =
                       intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE)
               val deviceName = device.name
               val deviceHardwareAddress = device.address // MAC address
           }
       }
   }
}

override fun onDestroy() {
   super.onDestroy()
   ...

   // Don't forget to unregister the ACTION_FOUND receiver.
   unregisterReceiver(receiver)
}
```
### Enable discoverabilty
To make the devices visible to other devices create an intent with `ACTION_REQUEST_DISCOVERABLE` action. 
A callback is needed for getting result code indicating wether the discoverability was allowed or not by the user.
By default, the device remains discoverable for 2 minutes (configurable).
```kotlin
val requestCode = 1;
val discoverableIntent: Intent = Intent(BluetoothAdapter.ACTION_REQUEST_DISCOVERABLE).apply {
   putExtra(BluetoothAdapter.EXTRA_DISCOVERABLE_DURATION, 300)
}
startActivityForResult(discoverableIntent, requestCode)
```
You can register a broadast receiver for `ACTION_SCAN_MODE_CHANGED` intent to know when the discoverability has ended.

### Connect Bluetooth devices
To connect two bluetooth devices, one must open a server socket and the other must initiate the connection.
If the devices are not paired, a dialog for pairing is automatically shown during the connection period.

#### Server
```kotlin
private inner class AcceptThread : Thread() {

   private val mmServerSocket: BluetoothServerSocket? by lazy(LazyThreadSafetyMode.NONE) {
       bluetoothAdapter?.listenUsingInsecureRfcommWithServiceRecord(NAME, MY_UUID)
   }

   override fun run() {
       // Keep listening until exception occurs or a socket is returned.
       var shouldLoop = true
       while (shouldLoop) {
           val socket: BluetoothSocket? = try {
               mmServerSocket?.accept()
           } catch (e: IOException) {
               Log.e(TAG, "Socket's accept() method failed", e)
               shouldLoop = false
               null
           }
           socket?.also {
               manageMyConnectedSocket(it)
               mmServerSocket?.close()
               shouldLoop = false
           }
       }
   }

   // Closes the connect socket and causes the thread to finish.
   fun cancel() {
       try {
           mmServerSocket?.close()
       } catch (e: IOException) {
           Log.e(TAG, "Could not close the connect socket", e)
       }
   }
}
```
#### Client
We create a new Thread since `connect()` is a blocking call
```kotlin
private inner class ConnectThread(device: BluetoothDevice) : Thread() {

   private val mmSocket: BluetoothSocket? by lazy(LazyThreadSafetyMode.NONE) {
       device.createRfcommSocketToServiceRecord(MY_UUID)
   }

   public override fun run() {
       // Cancel discovery because it otherwise slows down the connection.
       bluetoothAdapter?.cancelDiscovery()

       mmSocket?.let { socket ->
           // Connect to the remote device through the socket. This call blocks
           // until it succeeds or throws an exception.
           socket.connect()

           // The connection attempt succeeded. Perform work associated with
           // the connection in a separate thread.
           manageMyConnectedSocket(socket)
       }
   }

   // Closes the client socket and causes the thread to finish.
   fun cancel() {
       try {
           mmSocket?.close()
       } catch (e: IOException) {
           Log.e(TAG, "Could not close the client socket", e)
       }
   }
}
```
