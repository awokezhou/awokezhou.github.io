---
title: Python通过注册表获取串口列表
date: 2022-03-07 16:24:36
tags:
---

工作中需要使用CameraLink协议中的串口和设备进行通信，DALSA采集卡软件中可以将该串口映射到一个COM口上，但是该COM口在Windows的设备管理器中无法识别。使用Python的serial模块或者npm的serialport模块自带枚举函数都无法获取到该COM口。



JavaScript中，使用以下代码定时轮询串口列表

```javascript

import serialport from 'serialport'

setInterval(() => {
    serialport.list().then(
        ports => {
	        if (ports.length > 0) {
			    /* do something */
			}
		}
	)
}, 500)
```

Python中，使用以下代码获取串口列表

```python
import serial
import serial.tools.list_ports

plist = list(serial.tools.list_ports.comports())
ports = [p.name for p in plist]
```

以上方式都无法获取采集卡映射的COM口。Python中可以使用win32api和win32con通过枚举注册表中的信息来获取串口列表

```python
import win32api
import win32con

def get_serial_ports():
    ports = []
    key = win32api.RegOpenKey(win32con.HKEY_LOCAL_MACHINE,
        "HARDWARE\DEVICEMAP\SERIALCOMN", 0, win32con.KEY_READ)
    try:
        i = 0
        while True:
            name, value, type1 = win32api.RegEnumValue(key, i)
            print("name:{} value:{}".format(name, value))
            i += 1
            ports.append(value)
    except:
        print('except')
    win32api.RegCloseKey(key)
    return ports
```

使用`RegOpenKey`接口来读取键值，使用完毕后需要用`RegCloseKey`关闭。通过`RegEnumValue`来枚举串口列表，COM口名称在value字段
