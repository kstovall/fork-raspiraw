Raspiraw: An example app that receives data directly from CSI sensors
on the Raspberry Pi.

The register sets for OV5647 and IMX219 are often under NDA, which means
that support can not be offered over their contents without breaking
those NDAs. Anything added here by RPF/RPT has to be demonstrable as
already being in the public domain, or from 3rd parties having
reverse engineered how the firmware is working (eg by listening to the
I2C communications).


## Common Command line Options 

Execute raspiraw to get command help:

	raspiraw

TODO: add description of uncovered options. 

## raspiraw high framerate options

Lookup ov5647, imx219 or adv7282m datasheets for register details.

#### I2C register setting options

	--regs,		-r	Change (current mode) regs

Allows to change sensor regs in selected sensor mode. Format is a semicolon separated string of assignments.
An assignment consists of a 4 hex digits register address, followed by a comma and one or more 2 hex digit byte values.
Restriction: Only registers present in selected sensor mode can be modified. Example argument: "380A,003C;3802,78;3806,05FB".
In case more than one byte appears after the comma, the byte values get written to next adresses.

	--hinc,		-hi	Set horizontal odd/even inc reg

Sets the horizontal odd and even increment numbers. Argument is a 2 hex digits byte. "-hi xy" is convenience shortcut for "3814,xy" in --regs for ov5647 sensor. Lookup the sensor mode registers for your sensor header file for default values. TODO: Needs to be extended to deal with the other sensors as well.

	--vinc,		-vi	Set vertical odd/even inc reg

Sets the vertical odd and even increment numbers. Argument is a 2 hex digits byte. "-vi xy" is convenience shortcut for "3815,xy" in --regs for ov5647 sensor. TODO: Needs to be extended to deal with the other sensors as well.

	--fps,		-f	Set framerate regs

Sets the requested framerate. Argument is a floating point number. This is convenience shortcut for computing hh/ll values and "380E,hhll" in --regs for ov5647 sensor. TODO: Needs to be extended to deal with the other sensors as well.


#### Sensor mode setting options

The following options allow to overwrite some sensor mode settings for current sensor mode.

	--width,	-w	Set current mode width

Sets the width value of current mode.

	--height,	-h	Set current mode height

Sets the height value of current mode.

	--vts,		-v	Set current mode min_vts

Sets the min_vts value of current mode.

	--line,		-l	Set current mode line_time_ns

Sets the line_time_ns value of current mode.


#### File output settings

	--header0,	-hd0	Write the BRCM header to output file 0

For high framerate modes writing BRCM header to each file is a bottleneck.
So this option is a replacement for "--header"/"-hd" option.
Instead of writing header to each frame file, it is written to frame 0 file only.
Since frame 0 will not be written normally it is a good place.
For decoding ith frame with **dcraw** later, you need to concatenate frame 0 and frame i and store somewhere, then **dcraw** that file.

	--timestps,	-ts	Write timestamps to output file -1

With this option timestamps for the captured frames get written to output file -1.
This happens after video has been captured, so does not negatively affect capture framerate.
If this option is selected, a timestamp delta analysis is done and written to output, inclusive frame miss output.

	--empty,	-emp	Write empty output files

This option allows to determine the maximal framerate **raspiraw** callback will be triggered. Only empty files will be written for the frames, but the filenames allow to count how many. This would be an example use:

	raspiraw -md 7 -t 3000 -emp {some options from this section} -sr 1 -o /dev/shm/out.%04d.raw 2>/dev/null

Using **/dev/shm** ramdisk for storage is essential for high framerates. You precede this command by "rm /dev/shm/out.&ast;.raw" and do "ls -l /dev/shm/out.&ast;.raw | wc --lines" afterwards to determine the number of frames written ("-sr 1" means saverate 1 or writing all frames received from camera). "--empty" option allows to determine upper bounds for the absolute maximal framerate achievable for a given set of high framerate options.



#### Examples:

This is an example making use of most high framerate command line options:

	$ rm /dev/shm/out.*.raw
	$ raspiraw -md 7 -t 1000 -hd0 -h 64 -v 65 -l 10000 --vinc 3D --fps 600 -r "380A,0040;3802,78;3806,05FF" -sr 1 -o /dev/shm/out.%04d.raw 2>/dev/null
	Using i2C device /dev/i2c-0
	$ ls -l /dev/shm/out.*.raw | wc --lines
	604
	$

This command captures video from ov5647 camera on CSI-2 interface:
* based on 640x480 mode (-md 7)
* captures for 1s (-t 1000)
* stores BCRM header needed for dcraw only once in output file 0 (-hd0)
* sets frame capture height to 64 (-h 64)
* sensor mode vts setting a bit higher (-v 65)
* line_time_ns to 10000 (-l 10000)
* doubles line scanning speed from 0x35 to 0x3D (--vinc 3D, sum 8 vs 16)
* asks for 600 fps (--fps 600)
* sets some ov5647 registers (380A,0040;3802,78;3806,05FF)
* sets saverate to 1 (save all frames)
* outputs in "/dev/shm" ramdisk files starting with "out.0001.raw"
* redirects standard error output (lots of mmal messages) to /dev/null (2>/dev/null)

For being able to convert frame 123 captured frame with **dcraw** these steps are necessary (because of -hd0):

	cat /dev/shm/out.0000.raw /dev/shm/out.0123.raw > out.0123.raw
	dcraw out.0123.raw

Since line scanning speed was doubled, the captured 128x64 frames need to be stretched by factor 2.
You can use this small C code and know exactly what happens, or any other stretching program (gimp, netpbm tools, ...):
[double.c](https://stamm-wilbrandt.de/en/forum/double.c)

	double out.0123.ppm > out.0123.ppm.d

#### Creating .ogg video from dcraw processed and stretched .ppm frames

First you need to convert the .ppm frames you are interested in into .png format, eg. with netpbm tools:

	pnmtopng out.0123.ppm.d > out.0123.ppm.d.png 

This gstreamer pipeline creates .ogg video. You can choose framerate the video should play with (eg. 1fps for very slow motion), and start index of the frames to be taken. 

	gst-launch-1.0 multifilesrc location="out.%04d.ppm.d.png" index=300 caps="image/png,framerate=\(fraction\)1/1" ! pngdec ! videorate ! videoconvert ! videorate ! theoraenc ! oggmux ! filesink location="$1.ogg"

#### Creating of animated .gif from .ogg video

You can create high quality animated .gif from .ogg video with ffmpeg based [gifenc.sh](http://blog.pkh.me/p/21-high-quality-gif-with-ffmpeg.html). Yu only need to adjust **fps** and **scale** in **filters** variable of that script to match what you want.

	gifenc.sh $1.ogg $1.anim.gif

