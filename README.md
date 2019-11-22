# ads1x15_A9
Micropython driver for ADS1115 , ADS1015 , ADS1114 and ADS1113 for AI-Thinkers GPRS module A9/A9G
A9 port of the ESP8266 lib: https://github.com/robert-hh/ads1x15 

# Driver for the ADS1015/ADS1115 Analogue-Digital Converter

## Features

Control the operation of the ADS1x15 ADC and read back the data

## Connection

The ADS1X15 use a I2C interface. So SCL and SDA have to be connected as minimum.
If in continuous mode the CPU shall be triggered, the ALERT/RDY pin has to be
connected too, and obviously VDD, GND and the analogue input(2). You might also
set the address pin to low (address = 72), high (address = 73), SDA (74), SCL(75), 
so you can have 4 ADS1x15 modules using only one i2c interface (16 ADC pins).

## Class

The driver contains the ADS1115 class and the derived ADS1114, ADS1113 and
ADS1015 classes. Since the these devices only differ by the minor parameters
link the number of channels or conversion size, the same methods can be applied,
with different interpretation of the parameters.

As A9 has 3 i2c interfaces, you have to choose and set it's value as i2c_id.

```
adc = ADS1115(i2c, i2c_id, address, gain)
```
or
```
adc = ADS1114(i2c, i2c,id, address, gain)
```
or
```
adc = ADS1113(i2c, i2c_id, address)
```
or
```
adc = ADS1015(i2c, i2c_id, address, gain)
```
The default value for the address is 72, for gain is 0. Gain is an index into a
table. It defines the full range of the ADC.  Acceptable values are:
```
0 : 6.144V # 2/3x
1 : 4.096V # 1x
2 : 2.048V # 2x
3 : 1.024V # 4x
4 : 0.512V # 8x
5 : 0.256V # 16x
```

## Methods

### adc.read()
```
value = adc.read([rate, [channel1[, channel2]]])
```
Start a conversion on channel at speed rate and return the value.
Channel1 is the single input channel (0 .. 3). If channel2 is supplied,
the difference between channel1 and channel2 is taken. Rate is the
conversion rate. Suitable values are (ADS1015 / ADS1115/ ADS1114):
```
0 :  128/8      samples per second
1 :  250/16     samples per second
2 :  490/32     samples per second
3 :  920/64     samples per second
4 :  1600/128   samples per second (default)
5 :  2400/250   samples per second
6 :  3300/475   samples per second
7 :  - /860     samples per Second
```
The first value applies to the ADS1015, the second to the ADS1115 and ADS1114. The time
required for a single conversion is 1/samples\_per\_second plus the time
needed for communication with the ADC, which is about 1 ms on an esp8266
at 80 MHz. Slower conversion yields in a less noisy result.
The data sheet figures of the ads1x15 are given for the slowest sample rate.
The value returned is a signed integer of the raw ADC value. That value can be converted to a voltage with the method raw_to_v().

###  adc.set_conv and adc.read_rev()

Pair of methods for a time optimized sequential reading triggered by a time.
For using, you would first set the conversion parameters with set_conv() and then get
the values in a timer callback function with read_rev().
```
adc.set_conv([rate, [channel1[, channel2]]])
value = adc.read_rev()
```
The definition of channel1, channel2 and rate are the same as with adc.read(). The methods
read_rev() reads first the last conversion value back, and the starts a new
conversion. Care has to be taken, that the time needed for conversion and
communication is shorter than the timer period plus the time needed to process the data.
A sample code is shown below. The timing jitter observed on an esp8266 was
about 1 ms, but the time period is defined by the micro's timer, which has
it's own issues.
The value returned by read_rev is a signed integer of the raw ADC value.
That value can be converted to a voltage with the method raw_to_v().

###  adc.alert_start() and adc.alert_read()

Pair of methods to start a continuous sampling on a single channel and trigger
an alert once a certain threshold is reached.
```
adc.alert_start([rate, [channel1[, channel2]]][, threshold_high, threshold_low, latched])
value = adc.alert_read()
```
The values of channel1, channel2 and rate are the same as for adc.read().
Threshold_high tells upper value of the threshold register, and threshold_low 
the lower value. Boths must be be within
the range of the ADC, 0..32767 for ADC1115 and
0..2047 for ADS1015. Rate should be chosen according to the input signal
change rate and the precision needed. The mode set is the traditional
comparator mode, with the lower threshold set to 0.
latched tells whether thge aler is latched or not. If False, only one 
alter is generated when the threshold is reached until the threshold condition
is not valid, if True, alert signals are generated as long as the treshold conditions
match.
The value returned by alert_read is a signed integer of the raw ADC value.
That value can be converted to a voltage with the method raw_to_v().

###  adc.conversion_start() and adc.alert_read()

Pair of methods to start a continuous sampling on a single channel and trigger
an alert at every sample. This function pair is provided for an IRQ-based set-up.
```
adc.conversion_start([rate, [channel1 [, channel2]]])
value = adc.alert_read()
```
The values of channel1, channel2 and rate are the same as for adc.read().
The timing jitter seen is about 200 ns. However the ADC's timer is not very
precise. In applications where this is of importance some control and
calibration of the returned timing pattern has to be done.
The value returned by alert_read is a signed integer of the raw ADC value.
That value can be converted to a voltage with the method raw_to_v().

### adc.raw_to_v()
```
voltage = adc.raw_to_v(raw)
```
Convert the raw ADC result to a voltage that matches the gain setting of the
constructor. It returns a float value of the voltage.

###  adc.\_read_register()

Read a register of the ADC.
```
value = adc._read_register(register)
```
Register is the number of the register according to the data sheet. Reading the conversion register returns the value of
the most recent sampling. Bit 15 of the configuration register is set when a conversion
is finished.

###  adc.\_write_register()

Write a register of the ADC.
```
value = adc._write_register(register, value)
```
Register is the number of the register according to the data sheet, value a 16 bit
quantity coded accordingly.
# Sample Code

## Continuous sampling
```
# Sample code for A9, Micropython.org firmware
import i2c #as right now it's not yet inside Machine
import ads1x15
from time import sleep_ms, ticks_ms, ticks_us
from array import array

addr = 72
gain = 1
_BUFFERSIZE = const(512)

data = array("h", (0 for _ in range(_BUFFERSIZE)))
timestamp = array("L", (0 for _ in range(_BUFFERSIZE)))
i2c.init(2,100) # change here chosen i2c interface
ads = ads1x15.ADS1115(i2c, 2, addr, gain) # change also here chosen i2c interface

#
# Interrupt service routine for data acquisition
# called by a timer interrupt
#
def sample(x, adc = ads.alert_read, data=data, timestamp = timestamp):
    global index_put, irq_busy
    if irq_busy:
        return
    irq_busy = True
    if index_put < _BUFFERSIZE:
        timestamp[index_put] = ticks_us()
        data[index_put] = adc()
        index_put += 1
    irq_busy = False

irq_busy = False

index_put = 0
ADC_RATE = 5

# set the conversion rate to 860 SPS = 1.16 ms; that leaves about
# 3 ms time for processing the data with a 5 ms timer
ads.set_conv(7, 0) # start the first conversion
ads.read_rev()
sleep_ms(ADC_RATE)
while index_put < _BUFFERSIZE:
    sample(adc=ads.read_rev)
    sleep_ms(ADC_RATE)

# at that point data contains the sampled values, and
# timestamp the timer ticks which correlate to the conversion time
#
```
