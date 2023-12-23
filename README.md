# Makita XGT "40V Max" Communication Protocol

This document has been created by examining the data sent between batteries and
devices, and making educated guesses as to what the data means.  It is not an
official document and will contain mistakes and omissions.  Exercise due caution
when using this information as it is possible to brick some devices.

This information is made available free of charge, and free of responsibility.
By using this information you accept any and all liability for your actions and
agree that nobody involved in the creation of this documentation has any
responsibility for any mishaps you may encounter.  In short - if you follow
these instructions correctly and damage an expensive device, we are not liable.
If you do not agree that you are proceeding at your own risk, then do not take
any action based on this information.

Note that some information here may be covered by Makita patents, so you must
seek your own legal advice to ensure you are not infringing.  We are not lawyers
so cannot advise what is permissable.

If you use this information, please link back to this page as a way of saying
thanks to those who took the time to discover this information and make it
public.

## Physical interface

Aside from the 40V DC positive and negative power connections, the batteries
have four data connections.  These are:

 * DS/WK: Discharge/Wake - is the battery ok to discharge or is it flat?
 * DT: Detection - is a battery or charger connected?
 * TR: Transmit/Receive - digital communication (UART)
 * CS: Charge Sense - is charging allowed or prohibited?

There are two long contacts on a Makita 40V battery marked positive and
negative.  Between these two contacts there are four smaller contacts.
Two of these contacts are located closest to the grey locking mechanism, where
you press to release the battery - these are CS and TR, with CS being closest
to battery positive, and TR closest to battery negative.  The other two
contacts, furthest away from the grey release button, are DT and DS.  DT is
closest to battery positive, and DS is closest to battery negative.

When a battery is inserted and removed, the DS pin in the charger/tool is
briefly connected to the TR line on the battery, and the charger/tool CS pin is
briefly connected to the battery's DT pin.  The implications of this, and
whether it requires special handling, are currently unknown (e.g. does voltage
on the DS pin trigger a start bit in the UART on the DT pin?)

Electrically speaking it could be possible for up to 42 volts DC to appear on
any of these pins, so you should design your circuit to be tolerant of these
voltages.  During testing, the TR pin never went above 5 V on an official
Makita battery, but it did reach 33 V on a Chinese clone brand.  If you measure
the voltage on the TR pin first and verify it is below 5 V then you should be
fine while prototyping, however if you are designing a circuit that could be
used without someone testing the TR voltage first, then your design should be
40 volt tolerant.

### DS/WK: Discharge/Wake

This pin outputs a signal indicating whether the battery is safe to be
discharged.

It is also used as an input to supply Vdd to the internal microcontroller (uC)
in the battery, to power it up if the battery is in the shut-down state, which
presumably can happen if it is left flat for too long (and perhaps when first
shipped from the factory?)

Once the battery SoC increases above a certain level, the uC Vdd is generated
internally under battery power.  This internally generated Vdd appearing on the
DS line is presumably the "ok to discharge" signal.

In other words, this pin is the power line for the battery uC, and if it has
voltage on it, the uC is running on internal power and the battery is safe to
discharge.  If there is no voltage on this line, the battery has shut down the
uC to avoid over-discharge and so no power should be drawn from the battery.

TODO: Work out whether this line acts as a power supply for the uC in the tools
or whether they generate that off the 40V rails.

### DT: Detection

This pin is used to detect whether a battery is present.  A fully charged
battery outputs around 34.3 V on this pin.

A DC40RA rapid charger will not attempt communication unless some voltage is
present on this pin.  Voltages as low as 3.3 V are enough to trigger a charger
to start communication.

Further investigation is necessary.  There is some explanation on page 5 of the
patent document.

### TR: Data transmit/receive

This pin is used for digital communication between the battery and a connected
device.  It is explained in greater detail in the
[Electrical Protocol](#electrical-protocol) section below.

### CS: Charge Sense

The details of this pin are vague (covered on patent page 5) however it appears
to output two signals from the battery, indicating whether charging is permitted
or prohibited.

Presumably it is a high/low voltage to indicate either of these states, however
further investigation is needed.

u/Tool_Scientist recorded a whole charge cycle and found a relationship between
the charging current and the voltage on this pin:

    Ichg = 28.9 - 7.7 * Vcs

## Electrical Protocol

The communication protocol is based on the UART standard, operating at
9600/8-E-1 (9600 bps, 8 data bits, even parity bit, 1 stop bit).  This is the
same as most TTL UART schemes just with even parity instead of no parity.
Unlike UART, there are no separate TX and RX lines, there is only one single TR
line that handles communication in both directions, and which is referenced to
battery negative.  The TR line is normally at 0V and receives +5V pulses as bits
are transmitted.

Normal UART idles at a high voltage level (3.3V or 5V) and when a zero-bit is
sent, the voltage drops to 0V for (in the case of 9600 bps) 104 microseconds
for each 0 bit.  There is an extra zero-bit (low) at the start of each byte
transmitted called a "start bit", and after the eighth data bit is sent, the
parity bit is sent followed by a final one-bit (high) as a "stop bit".  As the
bus idles high and the start bit is a low, this high-to-low transition is used
to trigger the asynchronous reception of an incoming bit on the receiver side.

This design means only one device can transmit at a time, as the bus is held
high when idle.  To address this, Makita have inverted the voltage signals so
that the bus idles at 0V, and zero-bits such as the start bit cause the bus to
go high at 5V for the necessary transmission time.

When the bus is idle it is weakly pulled down to 0V, so that any device can
begin transmission by sending 5V pulses for binary 0, or not sending anything
for a binary 1, allowing the bus to fall back to 0V on its own.

This means to interface with this bus, the transmitter will need to be able to
output 5V to transmit a binary 0, and to release the bus (not driving it high
or low) when transmitting a binary 1 or at the end of transmission.  It probably
does not matter if the bus is driven low during transmission, however it must
not be driven once the transmission is complete, otherwise no other device will
be able to transmit a response to your message.

To test reception of messages, one can eaves drop on a battery connected to
a charger.  The battery can be fully charged, as messages are sent once a second
regardless, up to around 10-15 minutes when they stop.  Power cycling the
charger will start sending messages again.  To do this, use alligator clips or
similar to manually connect each battery contact to the charger as it would
connect normally.  Be sure to use a fully charged battery to avoid high
currents traversing the alligator clips connected to the positive and negative
terminals.  Connect your receiving device to the TR and negative terminals of
the battery, and once the charger is switched on you should immediately see
messages being sent from the charger and replies from the battery.

If you start to receive UART messages beginning with AD AD then you have not
inverted the high/low signals.  If you start to receive messages once a second
beginning with A5 A5 then everything is fine electrically and you can proceed.

## Protocol

Each byte arriving at the UART must have its bits reversed.  If the byte
received is 0x84 (1000 0100) it should become 0x21 (0010 0001).  If you have
done this correctly, and you are eavesdropping on messages between charger and
battery, you will see the message IDs in word 2 increment by one each time.  If
they increment by larger values like 0x80 then you have not reversed the bits
correctly.

Once reversed, each pair of bytes should be merged into a 16-bit value, in
big-endian order.  If byte 0x12 comes in first followed by 0x34, then it should
be treated as the 16-bit value 0x1234.  This is not mandatory but makes handling
the data much more logical.

Messages always start with 0xA5A5, and are always a multiple of 8 words
(16 bytes).  Messages are at least 8 words/16 bytes, but can be lengthened by
flags in the header (word 1).  If a message is not a multiple of 8 words, it
is padded with 0xFFFF words until it is.  When reading a message these 0xFFFF
padding words should be ignored.

The rest of this section explains the structure of each message, when treated
as a collection of 16-bit words.

### Word 0: Message Header

Always 0xA5A5.

This indicates the arrival of a new message.  When beginning communication,
incoming data should be ignored until an 0xA5A5 word arrives, to handle the
case where your device is powered on in the middle of a transmission.

It is recommended to set a timeout, so that if a message takes too long to
receive (indicating a potential loss of sync due to dropped or corrupted bytes),
the current message be discarded and further bytes ignored until another 0xA5A5
arrives.  This will ensure your code is robust in case of any data corruption
on the wire.

Devices respond relatively quickly, so a timeout of 300 to 500 milliseconds
since the last character was received should be ideal (remember to reset the
timeout after each character received, so that you do not mistakenly discard
long messages).  Likewise when transmitting messages, be sure to buffer them
first so that they can be sent at full line speed without any delays between
bytes.

### Word 1: Message Length

 * bits 0..3 - number of 0xFF padding bytes (0-15) at end of message
 * bit 4 - if set, add 16 bytes (8 words) to the expected message length
 * bit 5 - if set, add 32 bytes (16 words) to the expected message length
 * bit 6 - if set, add 64 bytes (32 words) to the expected message length
 * bit 7 - not yet observed, could be +128 bytes
 * bits 8..15 - unused? Always seems to be zero.

The default message length is 16 bytes.  If bits 5 and 6 are set, message will be
16 + 32 + 64 = 112 bytes.

### Word 2: Message ID

A numeric value used to identify this request.  Responses will contain the same
message ID, theoretically allowing requests and responses to arrive
out-of-order, although this has not been observed.

 * bits 0..11 - a 12-bit message ID
 * bit 12 - unknown, always set to 1
 * bit 13 - unknown, always set to 0
 * bit 14 - set if message is a request (e.g. charger querying battery)
 * bit 15 - set if message is a response (e.g. battery replying to charger)

The message ID appears to be a random number, but chargers start at 1 and
increment it by 1 for each request.  Each response should contain the same
message ID as the request that triggered it, with bits 14/15 switched as
appropriate.

### Word 3: Unknown

Always 0x4D4C.

### Word 4: Unknown

Always 0x00CC from a charger, and 0x000C from currently tested tools.

The battery always seems to include the same value in its response.

### Word 5: Command

Command ID, see below for details.

### Word 6: Command Parameter Length

Size of the parameters for the command, in bytes.  0x0000 if no further data.

### Words 7..n: Command Parameters

Command-specific data, of the length given by the previous word.

If the length was zero, then this field is omitted entirely.

### Word n+1: Checksum

Immediately following the previous field is a checksum.

The checksum is calculated by summing every byte from word 1 (i.e. excluding
0xA5 0xA5) until this point.  For example:

	A5A5 0018 1234 005E FFFF FFFF ...

	Checksum: 00 + 18 + 12 + 34 = 0x005E

### Word n+2: Optional Padding

If the message is not an even multiple of 8 words (16 bytes), 0xFFFF words are
added following the checksum until it is a multiple of this length.

The exact number of padding bytes, and the overall message length, are stored
in word 1.

### Commands

These are the commands available that can be set in word 5.

Although it is clear what most commands are doing, the actual command values
are currently unknown.  Multiple commands appear to do the same thing, so it
seems each command may actually be a bit field instead.  Further investigation
is needed to decipher this.

Some annotated data dumps of various commands follow:

#### 1200 (length 0x2A) - response B200

Announce parameters unsolicited?

Sent by a charger at the start of a charge cycle.

Example:

    A5A5 0036 5002 4D4C 00CC
      1200 002A       // Command and data length
      2101 0002       // Parameter 0x2101 is 2
      2102 2020 4152 3034 4344  // Parameter 0x2102 is "DC40RA  " (model name, ASCII, reversed)
      2103 0000       // Parameter 0x2103 is 0
      2104 0258       // Parameter 0x2104 is 600
      2105 0A21 063C  // Parameter 0x2105 is 2593 and 1596, or 169936444
      2107 0521 0814  // Parameter 0x2107 is 1313 and 2068, or 86050836
      2109 0000       // Parameter 0x2109 is 0
      210C E2D0       // Parameter 0x210C is 58064, seems like random ID for each charging session (76F1, 1FE7, BD3E, etc.)
      07D7            // Checksum
      FFFF FFFF FFFF  // Padding

Example response:

    A5A5 0000 9002 4D4C 00CC
      B200 0000       // Response command and empty length
      02A9            // Checksum

#### 1300 - response B300

Announce parameters unsolicited, from a tool when battery is first connected.

    A5A5 002C 5001 4D4C 000C
      1300 0014       // Command and data length
      3101 1234
      3102 5678
      3103 9ABC
      3105 CC6D
      3106 0000
      05F2            // Checksum
      FFFF FFFF FFFF FFFF FFFF FFFF

Example response:

    A5A5 0000 9001 4D4C 000C
      B300 0000       // Response command and empty length
      01E9            // Checksum

Does a B response indicate the battery has recorded that a tool has been
connected?

#### 1201 (length (0x18) - response 3201, length 0x38

Read multiple parameters?  Sent from a charger to a battery.

Example:

    A5A5 0028 5003 4D4C 00CC
      1201 0018       // Command and data length
      0003 000A       // Array of length 10
      1201 1202 1203  // Parameters to read
      1205 1206 1207 1208 1209 120A 120B
      030A            // Checksum
      FFFF FFFF FFFF  // Padding
      FFFF

Example response:

    A5A5 0048 9003 4D4C 00CC
      3201 0038       // Response command and data length
      0001 0000       // List of indeterminate length?
      1201 0400       // Parameter 0x1201 is 1024
      1202 2046 3035 3034 4C42  // Parameter 0x1202 is "BL4050F " (model name, ASCII, reversed)
      1203 00DB BA00  // Parameter 0x1203 is 14,400,000
      1205 0BD6       // Parameter 0x1205 is 3030
      1206 0B72       // Parameter 0x1206 is 2930
      1207 0E2E       // Parameter 0x1207 is 3630
      1208 0DCA       // Parameter 0x1208 is 3530
      1209 0D02       // Parameter 0x1209 is 3330
      120A 00C9 9108  // Parameter 0x120A is 13,209,864
      120B 0000 0000  // Parameter 0x120B is 0
      0AD6            // Checksum
      FFFF FFFF FFFF  // Padding
      FFFF

#### 1302, response 3302

Read multiple parameters?  Send from a blower tool to a battery.

    A5A5 002C 5002 4D4C 000C
      1302 0014       // Commmand and data length
      0003 0008       // Array of length 8
      1301 1306 1307 130C 130D 130E 130F 1310
      0243            // Checksum
      FFFF FFFF FFFF FFFF FFFF FFFF

Example response:

    A5A5 006F 9002 4D4C 000C
      3302 0051       // Response command and data length, odd number due to one param only being a single byte
      0001 0000       // List of indeterminate length?
      1301 FFFF
      1306 0E2E
      1307 09C4
      130C 58         // Only one byte!
      130D 05B3
      130E 12A0 2C04
      130F 0029 3200 6C78 6299 591B 53B3 4957 4181 3D98 31E0 31E0 0029 3200 6C78 664F 621E 53B3 4957 4181 3D98 31E0 31E0 0D02
      1310 0000
      1801            // Checksum
      FF              // Single padding byte!
      FFFF FFFF FFFF FFFF FFFF FFFF FFFF

#### 1203 (length 0xE), response 3203 (length 0x17)

Read multiple parameters?

Example:

    A5A5 0012 5004 4D4C 00CC
      1203 000E       // Command and data length
      0003 0005       // Array of length 5
      1204 120C 120D 120E 1210  // Parameters to read
      028B            // Checksum
      FFFF            // Padding

Example response:

    A5A5 0029 9004 4D4C 00CC
      3203 0017       // Response command and length
      0001 0000       // List of indeterminate length?
      1204 0000       // Parameter 0x1204 is 0
      120C 5000       // Parameter 0x120C is 20480
      120D 0D34       // Parameter 0x120D is 3380
      120E 6405       // Parameter 0x120E is 25605
      1210 8504       // Parameter 0x1210 is 34052
      83FF            // Checksum
      FFFF FFFF FFFF  // Padding
      FFFF

#### 1204 (length 8) - response B204, length 0

Write multiple parameters?

Example:

    A5A5 0018 5007 4D4C 00CC
      1204 0008       // Command and data length
      2101 0008       // Set parameter 2101 to 8?
      2109 0000       // Set parameter 2109 to 0?
      0246            // Checksum
      FFFF FFFF FFFF  // Padding
      FFFF

Example response:

    A5A5 0000 9007 4D4C 00CC
      B204 0000       // Response command and empty length
      02B2            // Checksum

Command response beginning with B might confirm a successful write to the
battery's memory.

#### 1304, response 3304

    A5A5 002E 5003 4D4C 000C
      1304 0012
      0003 0007
      1302 1303 1304 1308 1309 130A 1311
      0213
      FFFF FFFF FFFF FFFF FFFF FFFF FFFF

Example response:

    A5A5 0020 9003 4D4C 000C
      3304 0020
      0001 0000
      1302 0000
      1303 0001
      1304 6380
      1308 0BC7
      1309 6400
      130A 6400
      1311 0BC7
      05BA            // Checksum

#### 1205 (length 0xA) - response 3205, length 0x10

Read multiple parameters?

Example:

    A5A5 0016 5008 4D4C 00CC
      1205 000A       // Command and data length
      0003 0003       // Array of length 3
      1201 120D 120F  // Parameters to read
      024D            // Checksum
      FFFF FFFF FFFF  // Padding

Example response:

    A5A5 0010 9008 4D4C 00CC
      3205 0010       // Response command and length
      0001 0000       // List of indeterminate length?
      1201 0404       // Parameter 0x1201 is 1028
      120D 0D84       // Parameter 0x120D is 3460
      120F 0000       // Parameter 0x120F is 0
      0341            // Checksum

#### 1206 (length 4) - response B206, length 0

Write one parameter?

Example:

    A5A5 001C 5005 4D4C 00CC
      1206 0004       // Command and length
      2109 0002       // Set parameter 0x2109 to 2?
      021E            // Checksum
      FFFF FFFF FFFF  // Padding
      FFFF FFFF FFFF

Example response:

    A5A5 0000 9005 4D4C 00CC
      B206 0000       // Response command and empty length
      02B2            // Checksum

#### 1306 - response B306

Sent from a tool to a battery.

Not sure what the purpose of this is, unless the battery can refuse to operate
certain tool models, or the battery is recording the tools it has recently been
used with, perhaps to assist diagnosis of returned batteries under warranty.

    A5A5 0016 5004 4D4C 000C
      1306 000A       // Command and length
      3104 2020 4731 3030 5341  // "  G100SA" -> AS001G tool model, reversed
      0313            // Checksum
      FFFF FFFF FFFF

Example response:

    A5A5 0000 9004 4D4C 000C
      B306 0000       // Command and length
      01F2            // Checksum

#### 1307 - response 3307

Read parameters, sent from a tool to a battery.

    A5A5 001A 5005 4D4C 000C
      1307 0006       // Command and length
      0003 0001       // Array of one item
      130B            // 130B = battery model
      0156            // Checksum
      FFFF FFFF FFFF
      FFFF FFFF

Example response:

    A5A5 0012 9005 4D4C 000C
      3307 000E
      0001 0000
      130B 2046 3035 3034 4C42  // " F0504LB" -> BL4050F battery model, reversed
      0370
      FFFF

#### 120C (length 4) - response B20C, length 0

Write register?

Example:

    A5A5 001C 5009 4D4C 00CC
      120C 0004       // Command and length
      2101 0100       // Set parameter 2101 to 256?
      021F            // Checksum
      FFFF FFFF FFFF  // Padding
      FFFF FFFF FFFF

Example response:

    A5A5 0000 9009 4D4C 00CC
      B20C 0000       // Response command and empty length
      02BC            // Checksum

#### 120D (length 8) - response 320D, length 12

Read registers? (here reads 2 params, one fewer than 1205)

Example:

    A5A5 0018 500A 4D4C 00CC
      120D 0008       // Command and length
      0003 0002       // Array of length 2
      1201 120D       // Parameters to read
      0235            // Checksum
      FFFF FFFF FFFF  // Padding
      FFFF

Example response:

    A5A5 0014 900A 4D4C 00CC
      320D 000C       // Response command and length
      0001 0000       // List of indeterminate length?
      1201 0404       // Parameter 0x1201 is 1028
      120D 0D84       // Parameter 0x120D is 3460
      032A            // Checksum
      FFFF FFFF       // Padding

#### Command notes

* Commands starting with 0xBxxx seem to be acknowledgements that something has
  been written to the battery's memory.

# Credits

* The UART electrical signals and data protocol were reverse engineered by
  Malvineous.

* The checksum algorithm was reverse engineered by ESkri:
  https://reverseengineering.stackexchange.com/a/32347/20888

* The patent document was located by gcds:
  https://www.laptopu.ro/community/power-tools-battery-pinout/makita-bl4040-pinout/#post-12648
