# Candle.NET
.NET wrapper for the Candle API for controlling CAN bus gateways/analysers using the [Candlelight firmware](https://github.com/candle-usb/candleLight_fw) (e.g. candleLight, CANable, CANtact, etc)

## Instructions

* You need to copy the native `candle.dll` from `bin\$(Platform)\$(Configuration)\candle.dll` to your project's target folder so that it can be found by `DllImport`.

## Notes for general use

* A background thread manages sending and receiving
* You don't have to worry about regularly receiving. We do that for you
* Sending and receiving are non-blocking functions
* This is only tested on Windows (10 64bit). And since the underlying driver uses WinUSB, it is unlikely to work on other platforms

## Notes if you work directly with `NativeFunctions`

I strongly recommend that you work with `Device` and `Channel` classes rather than with the `NativeFunctions`. So these notes are mostly for me:

* You must periodically call `candle_frame_read` between your `candle_frame_send`s (e.g. if I perform 94 sends in a row then they start to fail, but no error is reported)
* I had issues with the device path being returned as a `wchar_t*` so instead we pass in a pre-allocated `StringBuffer` into an argument.

# Example

Example apps can be found in the `TestApp` folder.

```c#
using System;
using System.Threading;
using Candle;

namespace TestApp
{
	class Program
	{
		static void Main(string[] args)
		{
			var devices = Device.ListDevices();

			foreach (var device in devices)
			{
				device.Open();
				foreach(var keyValue in device.Channels) {
					var channel = keyValue.Value;
					channel.Start(500000);

					// Send frame
					{
						var frame = new Frame();
						frame.Identifier = 1 << 19;
						frame.Extended = true;
						frame.Data = new byte[3] { 0, 0, 0};
						channel.Send(frame);
					}

					Thread.Sleep(100);

					// Receive frames
					var receivedFrames = channel.Receive();
					foreach (var frame in receivedFrames)
					{
						Console.WriteLine(frame);
					}

					channel.Stop();
				}

				var errors = device.ReceiveErrors();
				foreach (var error in errors)
				{
					Console.WriteLine(error);
				}

				device.Close();
			}
		}
	}
}
```

# Credits

Candle.NET wraps a modified version of the CandleAPIDriver found in the [Cangaroo repo](https://github.com/HubertD/cangaroo/tree/master/src/driver/CandleApiDriver/api). The modifications are mostly to make things cleaner to wrap using the DLLImport mechanism of C#/.NET.

