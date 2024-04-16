---
layout: post
title: Reverse engineering the YK-H/531E AC remote control IR protocol
date: "2024-04-16 00:00:00"
tags:
    - homeassistant
---

My air conditioning unit uses the YK-H/531E remote controller. My unit is an Electrolux, but this remote is evidently an OEM part used by many other brands and units. I wanted to control my AC with Home Assistant, which meant getting a so-called IR blaster that can both receive and send IR signals, which let me reverse engineer the remote controller's IR protocol in order to create IR signals myself that could control the AC. In this post I'll describe what I found about the protocol and how it (mostly) works.

[In the next post I'll describe how I use it with Home Assistant](/2024/04/16/remotely-controlling-an-ac-unit-from-home-assistant-with-an-ir-blaster.html).

# Receiving IR messages and decoding them

To help with the reverse engineering process, I wrote [a script that receives a stream of IR messages from the IR blaster](https://github.com/Spanfile/ac-remote-reverse-engineer/blob/main/ir_receive.py) and prints them out, which can be redirected into a file. This script is very simple and designed only for my IR blaster, so you might have to adapt it for your environment if you want to use it.

For the actual decoding, I used [a script that can read multiple received IR messages from a file and decode them all](https://github.com/Spanfile/ac-remote-reverse-engineer/blob/main/ir_decode.py). It displays each decoded message as a binary string with helpful bit location guides, and for multiple messages it displays where any of them have a different bit set than any other message for that bit location. Finally it decodes all the AC settings from the messages and displays them. It should be able to catch most forms of corrupted messages, such as messages being too short, the raw IR signal timings being off or some setting not decoding properly.

# IR blaster

The IR blaster I ended up using is the [Moes UFO-R11](https://moeshouse.com/products/zigbee-smart-ir-remote-control). It is a rebrand of the similar Tuya model, except it is entirely battery powered with two AA-batteries instead of requiring external power input like the Tuya model. It is well supported in [Zigbee2MQTT](https://www.zigbee2mqtt.io/), which is what I use as a Zigbee gateway. It can be put into a "learning" mode, where it listens for an IR signal, and returns it in an encoded form. It can also transmit IR signals given in this same encoded form.

I bought mine from AliExpress, its list price hovers around 25-35€ but I got mine for a whopping 5,22€ with free shipping thanks to some sale and those coupons AliExpress loves handing out.

# Tuya's IR signal encoding

The IR blaster returns the IR signals in an encoded form as such:

```
BwAjnhEyAo4GgAPgBwHAF+ADB0AB4AcP4AkBAWUC4Bkj4BUBQD9AAcAH4IcB4EeXwE9AB0ADCzICMgKOBjICjgYyAg==
```

This is obviously base64-encoded data but how do we get the actual IR signal out of it? Luckily [someone else has already done the work for us](https://gist.github.com/mildsunrise/1d576669b63a260d2cff35fda63ec0b5), explaining how the base64 decodes into a tinyLZ-compressed sequence of 16-bit little-endian integers that represent the "raw" IR signal. They even provided a Python script to decode and encode your own "Tuya"-signals, for which I'm eternally grateful because I don't think I could've ever figured it out myself.

# IR signal

Once base64-decoded and tinyLZ-decompressed, the resulting list of integers looks like:

```
8973, 4505, 561, 1685, 561, 1685, 561, 561, 561, 561, 561, 561, 561, 561, 561, 1685, 561, 1685, ...
```

For this particular protocol, there should be exactly 211 numbers in the list. As the gist earlier also explains, these numbers are the "raw" IR signal. The numbers correspond to durations for how long, in microseconds, the IR signal was first high, and how long low afterwards. The signal starts by being high, then alternating between low and high. Since there's an odd number of numbers in the list, the last duration in the list is a high signal, which acts as a kind of footer to end the signal without encoding any data.

The first two durations, `8973` and `4505` are a kind of preamble to notify the target unit that a signal is about to be sent, they don't contain any data. Afterwards the durations should be split into adjacent pairs (skipping the last footer):

```
561, 1685
561, 1685
561, 561
561, 561
561, 561
561, 561
561, 1685
561, 1685
...
```

These durations are either "short", around 600µs for this protocol, or "long", around 1700µs. Each of these pairs encode a single bit, such that the first duration is always "short", then followed by either another "short" or a "long". A short-short pair encodes the bit `0` and a short-long pair encodes the bit `1`. Decoding these bits least-significant-bit first creates the actual control message we're interested in. This is the decoded message displayed most-significant-bit first, with bit indices displayed above:

```
 100   96   92   88   84   80   76   72   68   64   60   56   52   48   44   40   36   32   28   24   20   16   12    8    4    0
1101_1010_0000_0000_0000_0000_0010_0000_0000_0000_0000_0000_0000_0000_0000_0000_1010_0000_0000_0000_1110_0000_0111_0111_1100_0011
```

# AC control protocol

From here on out reverse engineering the control protocol was fairly straight-forward. I already knew that this one message contains all the parameters for the AC; the remote control has a display for the current settings, and by including all settings in the message it prevents the AC unit desynchronising from the remote. The remote control is an OEM product, so a lot of different AC units also use this same protocol. I was able to find [someone else's guide on having reverse engineered a similar remote](https://www.instructables.com/Reverse-engineering-of-an-Air-Conditioning-control/), which meant I largely knew what I was looking for in the message.

The message encodes the following settings:

| **Start bit** | **Length** | **Setting**        | **Example** |
| ------------- | ---------- | ------------------ | ----------- |
| 0             | 8 bits     | Preamble           | `1100 0011` |
| 8             | 3 bits     | Swing              | `111`       |
| 11            | 5 bits     | Temperature (C)    | `01101`     |
| 21            | 3 bits     | Unknown            | `111`       |
| 32            | 2 bits     | Timer pt. 1        | `01`        |
| 37            | 3 bits     | Fan speed          | `101`       |
| 41            | 4 bits     | Timer pt. 2        | `1111`      |
| 49            | 1 bit      | Celsius/Fahrenheit | `0`         |
| 50            | 1 bit      | Sleep              | `0`         |
| 53            | 3 bits     | AC mode            | `000`       |
| 77            | 1 bit      | On/off             | `1`         |
| 81            | 7 bits     | Temperature (F)    | `100 1101`  |
| 88            | 3 bits     | Button             | `001`       |
| 92            | 1 bit      | Unknown            | `0`         |
| 96            | 8 bits     | Checksum           | `1101 1010` |

Let's go through all the different settings.

## Preamble

The first byte is a preamble, in this unit's case `1100 0011`. It doesn't contain any actual information, it's likely just an identifier for this particular AC unit.

## Swing

Bits 8 to 10 are swing; the AC unit has a flap that can be set to "swing", i.e. oscillate up and down. In this unit, `111` means swing off, `000` means swing on. Other AC units might have multiple swing options, hence there being more than two possible options.

## Target temperature (Celsius)

Bits 11 to 15 is the set target temperature in Celsius. The AC unit has a range of set target temperatures from 16C to 32C, however the encoded value here is 8C to 24C, so offset by -8. I'm not entirely sure why its offset, but my guess is to save one bit in the value - the upper end of the range, 32C, would require six bits. Note the AC's target temperature can also be set in ~~weird units~~ Fahrenheit, which is encoded later in the message.

## Timer

The AC has a timer functionality that lets you set a timeout after of which it'll turn on automatically. I didn't bother reverse engineering this one, since I've never used it and would now achieve the same thing with Home Assistant. It probably works the same as in the article I previously linked.

## Fan speed

Bits 37 to 39 set the AC's fan's speed. There are four possible options for this unit:

| **Bit pattern** | **Fan speed** |
| --------------- | ------------- |
| `001` 1         | High          |
| `010` 2         | Mid           |
| `011` 3         | Low           |
| `101` 5         | Auto          |

Note that not all options are avaible depending on the AC's mode.

## Celsius/Fahrenheit

The AC's set target temperature can be set in either Celsius or Fahrenheit. If the 49th bit is `0`, the target temperature is in Celsius and `1` if it is in Fahrenheit. Note that both units of temperature can be encoded in the message at once; this bit decides which one is used.

## Sleep

The AC has a "sleep"-mode, which is a boolean toggle on the 50th bit; `0` if off, `1` if on.

## AC mode

Bits 53 to 56 is the AC's mode, which is one of four options:

| **Bit pattern** | **Mode** | **Notes**                                                          |
| --------------- | -------- | ------------------------------------------------------------------ |
| `000` 0         | Auto     | Fan speed can only be set to Auto                                  |
| `001` 1         | Cool     |                                                                    |
| `010` 2         | Dry      | Fan speed can only be set to Low                                   |
| `110` 6         | Fan      | a) Fan speed can not be Auto<br>b) The set target temperature is 0 |

## On/off

The 77th bit is the AC's on/off state, `0` if off, `1` if on.

## Target temperature (Fahrenheit)

Bits 81 to 87 is the set target temperature in Fahrenheit. The range for Fahrenheit is 60F to 90F, but oddly the range in the protocol is 68F to 98F, so offset by +8. This is similar to the -8 offset in the Celsius temperature field, however this doesn't save any bits in the message. I'm not sure at all why this offset is there, but I don't use the weird units myself anyway so.

## Button

Bits 88 to 91 tells which button was pressed on the remote to generate this message. To be honest, I have no clue why the protocol would require this information. [This article I found has a somewhat similar situation](https://blog.depau.eu/2021/06/12/ir-remote-reveng/), except it's only for the power button. This unit's protocol already includes all the necessary information in the payload, so maybe the button is just a leftover thing for some other kinds of units? Anyway, since it's there, I have to deal with it.

| **Bit pattern** | **Button**                                                                                    |
| --------------- | --------------------------------------------------------------------------------------------- |
| `0000` 0        | + (temperature up)                                                                            |
| `0001` 1        | - (temperature down)                                                                          |
| `0010` 2        | Swing                                                                                         |
| `0100` 4        | Fan speed                                                                                     |
| `0101` 5        | On/off                                                                                        |
| `0110` 6        | Mode                                                                                          |
| `0111` 7        | Unit (C/F)<br>Holding down + and - for 3 seconds changes the temperature unit between C and F |
| `1011` 11       | Sleep                                                                                         |
| `1101` 13       | Timer                                                                                         |

## Checksum

The last byte in the message is a simple checksum of the message's payload; it is the sum of every byte in the payload, truncated to one byte. It seems this same method is used in many other IR remotes as well.

## Mystery bits

There are still a couple bits in the message I don't know what they mean, but the unit doesn't seem to mind what they are so I'm not bothered about them.

Bits 21 to 23 seem to always be set as all `1`. I've never seem them be anything else, so I've included them in my own encoded messages later.

Bit 92 seems to flip between `0` and `1`, no idea why and when exactly. I've always set it to `0`, the unit doesn't seem to mind. It's right next to the button bits, maybe something to do with it?

# Encoding my own control messages

Now that I understand how the protocol works, it's simple to encode my own messages. [I wrote an encoder script](https://github.com/Spanfile/ac-remote-reverse-engineer/blob/main/ir_encode.py) that takes all the different settings as input, generates the 104-bit control message, converts it into signal timings for IR and feeds it to the Tuya IR encoder program to finally get a base64-string out that can be fed into the IR blaster to send out.

For example, taking these settings:

| **Setting**        | **Value** |
| ------------------ | --------- |
| On/off             | On        |
| Target temperature | 20C       |
| Mode               | Cool      |
| Fan speed          | High      |
| Swing              | No        |
| Sleep              | No        |
| Button             | On/off    |

Would produce the following 104-bit message (again, most-significant-bit first):

```
 100   96   92   88   84   80   76   72   68   64   60   56   52   48   44   40   36   32   28   24   20   16   12    8    4    0
0110_1111_0000_0101_0000_0000_0010_0000_0000_0000_0000_0000_0010_0000_0000_0000_0010_0000_0000_0000_1110_0000_0110_0111_1100_0011
```

Converting this message into the short-short/short-long duration scheme I explained earlier produces the following list of IR signal durations:

```
9100, 4500, 600, 1700, 600, 1700, 600, 600, 600, 600, 600, 600, 600, 600, 600, 1700, 600, 1700, ..., 600
```

You'll note I used approximate values for each duration, including the first two values for the preamble. The AC unit seems to allow quite a lot of leeway on the exact durations for the values, and it could be the IR blaster can't exactly reproduce these durations anyway. Note to include the single short-value as a footer at the very end to finalise the last bit in the message.

Encoding this list as a Tuya-message results in:

```
B4wjlBFYAqQGgAPgBwHAF+ADA+AHG+APAeALM+AjAeAvN+A/P+BDn8E34Q8v4Qtj4QM7
```

This message is a lot shorter than the ones from the remote, since the uniformity of each duration value means the list compresses very well (the encoding uses tinyLZ level 2 by default).

Sending this message to the IR blaster with the AC unit nearby, after a short delay the AC beeps and turns on with these settings! [In the next post I'll describe how I integrated this work into Home Assistant](/2024/04/16/remotely-controlling-an-ac-unit-from-home-assistant-with-an-ir-blaster.html).

[The project's code can be found in this Github repository](https://github.com/Spanfile/ac-remote-reverse-engineer).
