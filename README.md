iio-sensor-proxy version 2
================

I(Anthony Guevara) modified this program so that I could get my 2-in-1 touch screen to rotate based on the screen orientation. This means that if you rotate the laptop 90 degrees then the screen will follow and adjust to the current orientation. I run Gentoo Linux and this was an issue on my distro and this is the solution. Once you compile the program you should write a daemon to make sure the program is always running and that it restarts if it crashes. An RC script should also be created. I will eventually write these wehn I get time. 

The modifications were made to the `monitor-sensor.c` file. Here is the diff:

```javascript
diff --git a/src/monitor-sensor.c b/src/monitor-sensor.c
index 7eb35cc..0f10ae7 100644
--- a/src/monitor-sensor.c
+++ b/src/monitor-sensor.c
@@ -7,6 +7,9 @@
  *
  */
 
+#include <stdio.h>
+#include <stdlib.h>
+#include <string.h>
 #include <gio/gio.h>
 
 static GMainLoop *loop;
@@ -22,6 +25,11 @@ properties_changed (GDBusProxy *proxy,
 	GVariant *v;
 	GVariantDict dict;
 
+	const char *normal    = "normal";
+	const char *left_up   = "left-up";
+	const char *right_up  = "right-up";
+	const char *bottom_up = "bottom-up";
+
 	g_variant_dict_init (&dict, changed_properties);
 
 	if (g_variant_dict_contains (&dict, "HasAccelerometer")) {
@@ -34,7 +42,24 @@ properties_changed (GDBusProxy *proxy,
 	}
 	if (g_variant_dict_contains (&dict, "AccelerometerOrientation")) {
 		v = g_dbus_proxy_get_cached_property (iio_proxy, "AccelerometerOrientation");
-		g_print ("    Accelerometer orientation changed: %s\n", g_variant_get_string (v, NULL));
+		//printf("Debug string: %s", (char *)g_variant_get_string(v, NULL));
+		if(strncmp((char *)g_variant_get_string(v, NULL), left_up, 43) == 0) {
+			system("/usr/bin/xrandr -o left");
+			printf("  -> Rotating screen left with 'xrandr -o left'.\n");
+		}
+		else if(strncmp((char *)g_variant_get_string(v, NULL), right_up, 43) == 0) { 
+			system("/usr/bin/xrandr -o right");	
+			printf("  -> Rotating screen right with 'xrandr -o right'.\n");
+		}
+		else if(strncmp((char *)g_variant_get_string(v, NULL), bottom_up, 43) == 0) {
+			system("/usr/bin/xrandr -o inverted");	
+			printf("  -> Inverting screen with 'xrandr -o inverted' now.\n");
+		}
+		else if(strncmp((char *)g_variant_get_string(v, NULL), normal, 43) == 0) {
+			system("/usr/bin/xrandr -o normal");	
+			printf("  -> Setting screen back to normal with 'xrandr -o normal'.\n");
+		}
+		g_print ("Accelerometer orientation changed: %s\n", g_variant_get_string (v, NULL));
 		g_variant_unref (v);
 	}
 	if (g_variant_dict_contains (&dict, "HasAmbientLight")) {
```

IIO sensors to D-Bus proxy

See https://developer.gnome.org/iio-sensor-proxy/1.0/ for
developer information.

Installation
------------
```
./configure --prefix=/usr --sysconfdir=/etc
make
make install
```
It requires libgudev and systemd (>= 233 for the accelerometer quirks).

Usage
-----

With a GNOME 3.18 (or newer) based system, orientation changes will
automatically be applied when rotating the panel, ambient light will be used
to change the screen brightness, and Geoclue will be able to read the compass
data to show the direction in Maps.

Note that a number of kernel bugs will prevent it from working correctly on
some machines so please make sure to use the latest upstream kernel (kernel
crashes on the Surface Pro, sensor failing to work after suspend on the Yoga
Pro, etc.).

You can verify that sensors are detected by running `udevadm info --export-db`
and checking for an output resembling this one:
```
P: /devices/platform/80860F41:04/i2c-12/i2c-BMA250E:00/iio:device0
N: iio:device0
E: DEVNAME=/dev/iio:device0
E: DEVPATH=/devices/platform/80860F41:04/i2c-12/i2c-BMA250E:00/iio:device0
E: DEVTYPE=iio_device
E: MAJOR=249
E: MINOR=0
E: SUBSYSTEM=iio
E: SYSTEMD_WANTS=iio-sensor-proxy.service
E: TAGS=:systemd:
E: USEC_INITIALIZED=7750292
```

You can now check whether a sensor is detected by running:
```
gdbus introspect --system --dest net.hadess.SensorProxy --object-path /net/hadess/SensorProxy
```

After that, use `monitor-sensor` to see changes in the ambient light sensor
or the accelerometer. Note that compass changes are only available to GeoClue
but if you need to ensure that GeoClue is getting correct data you can run:
`su -s /bin/sh geoclue -c monitor-sensor`

If that doesn't work, please file an issue, make sure any running iio-sensor-proxy
has been stopped:
`systemctl stop iio-sensor-proxy.service`
and attach the output of:
`G_MESSAGES_DEBUG=all /usr/sbin/iio-sensor-proxy`
running as ```root```.

Accelerometer orientation
-------------------------

When the accelerometer is not mounted the same way as the screen, we need
to modify the readings from the accelerometer to make sure that the computed
orientation matches the screen one.

`iio-sensor-proxy` reads this information from the device's
`ACCEL_MOUNT_MATRIX` udev property. See [60-sensor.hwdb](https://github.com/systemd/systemd/blob/master/hwdb/60-sensor.hwdb)
for details.

Compass testing
---------------

Only the Geoclue daemon (as the geoclue user) is allowed to access the `net.hadess.SensorProxy.Compass`
interface, the results of which it will propagate to clients along with positional information.

If your device does not contain a compass, you can run tests with:
- If your device does not contain a real compass:
  - Add FAKE_COMPASS=1 to the environment of `iio-sensor-proxy.service` if your device does not contain a real one
  - Run the daemon manually with `systemctl start iio-sensor-proxy.service`
- Verify that iio-sensor-proxy is running with `systemctl` or `ps`
- As root, get a shell as the `geoclue` user with `su -s /bin/bash geoclue`
- Run, as the `geoclue` user, `monitor-sensor`

References
----------

- [sensorfw](https://git.merproject.org/mer-core/sensorfw/tree/master)
- [android-iio-sensors-hal](https://github.com/01org/android-iio-sensors-hal)
- [Sensor orientation on MSDN](https://msdn.microsoft.com/en-us/windows/uwp/devices-sensors/sensor-orientation)

Tested on
---------

- Apple MacBook Air (4,2)
- Apple MacBook Air (6,2)
- Apple MacBook Pro (8.2)
- Asus Transformer Book TP500LB
- Asus Zenbook UX31A, UX303L, UX305
- Cube i9
- Dell Inspiron 13 7000
- Dell Venue 11 Pro (7140)
- Dell Venue 8 Pro
- HP Pavilion X360
- HP Spectre x360 (Kaby Lake)
- Lenovo IdeaPad Yoga 13
- Lenovo ThinkPad Twist
- Lenovo X1 Carbon 2014 (rev2)
- Lenovo X1 Tablet
- Lenovo Yoga 2 13" and 11"
- Lenovo Yoga 2 Pro
- Lenovo Yoga 460
- Lenovo Yoga 900
- Microsoft Surface Pro 2
- Onda v975w
- Toshiba Portégé Z10t
- Toshiba Radius 11 L10WC10C
