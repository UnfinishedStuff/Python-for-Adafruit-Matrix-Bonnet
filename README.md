# Python-for-Adafruit-Matrix-Bonnet

This repo contains some documentation for using [Adafruit's Matrix Bonnet](https://learn.adafruit.com/adafruit-rgb-matrix-bonnet-for-raspberry-pi/overview) with Python on the Raspberry Pi.  Strictly speaking it's about using Hub75-matrices with [Hzeller's code](https://github.com/hzeller/rpi-rgb-led-matrix), because unusually for an Adafruit product they recommend using someone else's software.  Anyway, This repo is to remind me of how to get Python scripts running, because I didn't think the documentation was especially great, and I'm sure in a short while my brain will have erased all memory of doing this.

# Hardware

This documentation is written after having run the scripts on a Pi 4B (4GB), powered using the official 5V 3A USB-C power supply.  I used a [64x64 Hub75 matrix from Pimoroni](https://shop.pimoroni.com/products/rgb-led-matrix-panel?variant=3029531983882), powered by a [5V 5A power supply from ThePiHut](https://thepihut.com/products/neopixel-power-brick-5v-5a-25w).  Strictly speaking Adafruit's guide is that a 64x64 matrix may consume up to 7.68A, but that would be if you had all LEDs on full-white, so you can get away with less if you're careful. 

These were connected using [Adafruit's RGB Matrix Bonnet for Raspberry Pi](https://www.adafruit.com/product/3211).  

If you skim through [Adafruit's guide for using Hub75 matrices with their Bonnet](https://learn.adafruit.com/adafruit-rgb-matrix-bonnet-for-raspberry-pi/driving-matrices) you'll see that there are two potential hardware setups:  the Convenience setting requires no hardware alterations, whereas Quality requires a jumper to be soldered.  I ran some initial tests with Convenience and saw some noise in the matrix, and then soldered the jumper and reinstalled with the Quality settings.  The noise disappeared at this point, but that may also have been to do with a better understanding of which parameters can be tweaked in scripts.  Your mileage may vary.

# Software

Install the software [as per Adafruit's guide](https://learn.adafruit.com/adafruit-rgb-matrix-bonnet-for-raspberry-pi/driving-matrices).  The code is primarily written in a flavour of C which I'm frankly not very good with, so I took a look at the Python code in `/rpi-rgb-led-matrix/bindings/python/`.  Unfortunately, I didn't think the documentation was great, so I've made some notes here.

**Setting the matrix options**

The guide suggest using a lot of flags in the command line alter the behaviour of the matrix, e.g. `--led-rows=` to set the number of LED rows.  This doesn't need to be done from the command line when starting scripts, you can do it within Python using the `RGBMatrixOptions` module.

Start by importing the module, and creating an instance of the class:

```
from rgbmatrix import RGBMatrixOptions
options = RGBMatrixOptions()
```

Then set parameters for the `options` object (see below) to fine-tune the performance of the matrix.  When you're finished, create an instance of the `RGBMatrix` class with your desired options:

```
matrix = RGBMatrix(options=options)
```

And that's it.  At this point you can pass it PIL/Pillow images to display and they'll show up on the matrix:

```
matrix.SetImage(Image)
```

Hzeller's module seems to come with it's own Image/Canvas modules for drawing things which need passed to the matrix with the `SwapOnVSync()` function, but I've not delved into that.

# Parameters for RGBMatrixOptions

I skimmed this list of options from one of the C files, but not all of them seem to work:

* hardware_mapping
  * I'm not entirely clear on the meaning of this, must be given a string.
* rows
  * An integer describing the number of rows in the matrix, e.g. `options.rows=64`
* cols:
  * An integer describing the number of columns in the matrix, e.g. `options.cols = 64`
* chain_length
  * This apparently specifies the number of matrices in a chain, I've not explored using more than 1 because I only have one matrix.
* parallel
  * I'm not entirely sure about this one, the main documentation implies it is about running multiple chains of panels off the same raspberry Pi.
* pwm_bits
  * From the C documentation, this is the resolution of the Pulse-Width Modulation used to dim the LEDs.  Higher resolution means more "steps" and so smoother transitions, but also requires more data to be transmitted.  The C documentation suggests reducing this if you're driving a lot of pixels as a means to possibly increase framerate.  Must be given an integer from 1-11 (defaults to 11).
* pwm_lsb_nanoseconds
  * From the C documentation: "This allows to change the base time-unit for the on-time in the lowest significant bit in nanoseconds. Lower values will allow higher frame-rate, but will also negatively impact qualty in some panels (less accurate color or more ghosting)."  Defaults to 130, bust be given a value between 50 and 3,000.  After some quick experimentation, values above ~250 start to make a simple Python text scroller look stuttery, and values >1000 visibly blink.
* brightness
  * Global brightness control, takes a value from 0-100 (so percent).
* scan_mode
  * From the main documentation: "This switches from progressive scan and interlaced scan. The latter might look be a little nicer when you have a very low refresh rate, but typically it is more annoying because of the comb-effect (remember 80ies TV ?)."  0 is progressive (default), 1 is interlaced.  A quick experiment didn't suggest much of a difference.
* multiplexing:
  * [See the main documentation about multiplexing](https://github.com/hzeller/rpi-rgb-led-matrix), apparently only required for certain types of 64x64 matrices (I didn't require it with the one I purchased from Pimoroni).
* row_address_type:
  * See multiplexing, again I didn't require this for my 64x64 matrix.
* disable_hardware_pulsing:
  * I don't know about this one.
* show_refresh_rate
  * If set to 0, does nothing. If set to 1 will print the refresh rate (Hz and uSec) in the terminal when running.
* inverse_colors:
  * If set to 0, does nothing. If set to 1, seems to make the whole display white?
* led_rgb_sequence
  * Apparently for setting whether the matrix is RGB or GRB when interpreting colours.  Takes a string, and while it can be used to switch between more than RGB/GRB this is a little finnicky: BGR Starts colours with blue, but BRG starts them with green?  Probably not really required unless you have an odd type of matrix.
* pixel_mapper_config
  * This can be used to rotate/flip the matrix.  Pass it "Rotate:" followed by an angle in increments of 90 degrees (e.g. `"Rotate:90"`) and the matrix will be rotated clockwise by that many degrees.  According to the main documentation you should be able to pass it an argument of "Mirror:H" for a horizontal (or V for vertical) mirror, but when I tried this it returned a "no such mapper" error.
* panel_type
  * The main documentation states that some matrices have alternative driving hardware.  However, despite being in the code it returned an error if I tried using it.
* pwm_dither_bits
  * Apparently an alternative to PWM-ing LEDs to control brightness is just not showing them on certain matrix refreshes.  This may allow faster refresh rates, but may be dimmer and consume more CPU power.  Must be given a value of 0-2 (defaults to 0).
* limit_refresh_rate_hz
  * Can apparently be used to limit the refresh rate, but if tried it always threw a "object has no attribute" error for me.
* gpio_slowdown
  * Apparently more recent models of Pi can output data too quickly for the LEDs to properly respond.  Use this to slow the GPIO pins if you get poor results.  Adafruit recommends 4 for the Pi 4 and 1 for the Pi3.  Max value is 5.
  
# A Hello World Example

A quick example which will write "Hello World!" on the matrix.  Save this as a python file, and run it with sudo rights.

```
#!/usr/bin/env python
from rgbmatrix import RGBMatrix, RGBMatrixOptions
from PIL import Image, ImageDraw
import time

# Create a 64 x 64 pixel image to draw and show
image = Image.new("RGB", (64, 64), (0,0,0))

# Create a blank image with all pixels set to black to clear the matrix when we're done.
blank_image = Image.new("RGB", (64, 64), (0,0,0))

# Set the desired options
options = RGBMatrixOptions()
# Set 64 rows
options.rows = 64
# Set 64 columns
options.cols = 64
# Use the pin mapping for an Adafruit Bonnet
options.hardware_mapping = "adafruit-hat"
# Cap brightness at 50%
options.brightness=50
# This was run on a Pi 4, so slow the GPIO pins.
options.gpio_slowdown=4

# Set up the matrix with the options selected above
matrix = RGBMatrix(options = options)

# Edit the PIL image object called image (created above)
drawnImage = ImageDraw.Draw(image)
# Add text to the image in red font
drawnImage.text((0,0), "Hello World!", fill=(255,0,0))
# Pass the image to the matrix and wait for 5 seconds
matrix.SetImage(image)
time.sleep(5)

# Blank the matrix by showing the blank_image created above
matrix.SetImage(blank_image)
```
