# SWO Parser
This python script is used to parse ITM trace packets from the SWO pin on the
STM32 using OpenOCD. It is written for python 3, but shouldn't be too hard to
port to python 2 if you're one of those people. It communicates with OpenOCD
through the Tcl server (default at localhost:6666).

```
+-------+    +-----------+
|       |SWO |           |
|  MCU  +----+  ST-LINK  |
|       |    |           |
+-------+    +-----+-----+
                   |
                   |USB
                   |
             +-----+-----+
             |  OpenOCD  |
             +-----+-----+
                   |
                   |TCP:6666(default)
                   |
             +-----+-----+
             |  Python   |
             |  Script   |
             +-----------+
```



## OpenOCD Settings
To use this script, first you must add some flags when you start up OpenOCD
(or add these to your startup cfg file for OpenOCD).

```
openocd -d2 -f interface/stlink.cfg  -f target/stm32f4x.cfg -c init -c targets
```

The "target/stm32f4x.cfg" represents stm32f4x chip, substitute your own cfg here.

The "interface/stlink.cfg" represents stlink debugger, substitute your own debugger here.

## STM32 Settings
You shouldn't need to configure anything on the target. This fact took me
**forever** to figure out. When the debugger starts, it automatically sets the
proper registers to the right values. One nice side effect of this is that, if
the debugger is not attached to the target, your code will skip over sending
the trace messages. This is really nice since ITM_SendChar is blocking (i.e.,
the code stops until every byte is sent out on the SWO pin).

To send some data in your code, use ITM_SendChar. You can easily redirect
printf over this channel using that. Keep in mind though that **Messages will
not be displayed on the console until you send a newline character, '\n'**.
This is a result of the way I decided to implement things in this python
script. ITM_SendChar sends messages out on channel 0, but it's simple enough
to write your own version of it that can choose which channel to send data. You
could use this to filter types of messages (e.g, info, warnings, and errors).

## Running the Python Script
After starting up OpenOCD, open a terminal in the directory where you placed
swo_parser.py and type

```
python swo_parser.py 180000000 50002
```

**note：**
The first parameter represents the target clock frequency, the second parameter 
represents the Tcl server port; If you don't use these parameters, default 
CPU clock=80000000 Tcl server port=6666.

By default, messages will be parsed from channels 0, 1, and 2. Messages from
channels 1 and 2 will be prepended with "WARNING: " and "ERROR: " respectively.
You can configure these however you like. The following lines of the script
control the configuration:

```
# Create a stream manager and add three streams
streams = StreamManager()
streams.add_stream(Stream(0, '', tcl_socket))
streams.add_stream(Stream(1, 'WARNING: ', tcl_socket))
streams.add_stream(Stream(2, 'ERROR: ', tcl_socket))
```

The third parameter in the Stream constuctor (tcl_socket) is optional. Leave
it out if you don't want that message to be echoed to the GDB console.

