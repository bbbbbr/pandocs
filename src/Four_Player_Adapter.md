# 4-Player Adapter

The 4-Player Adapter (DMG-07) is an accessory that allows 4 Game Boys
to connect for multiplayer via [serial data transfer](./Serial_Data_Transfer_(Link_Cable).md).
The device is primarily designed for DMG consoles, with later models
requiring Link Cable adapters.

## Power

The DMG-07 will not power on until it's Player 1 cable is plugged into
a Game Boy link port to supply power. The Player 1 cable is the only one
permanently attached to the device and has the power pin connected
unlike typical link port cables.

## Communication Phases

The DMG-07 protocol can be divided into 2 sections, the "ping" phase, and
the "transmission" phase. The initial ping phase involves sending packets
back and forth between connected Game Boys probing for their current
connection status. Afterwards, the DMG-07 is switched to the "transmission"
phase where the Game Boys exchange data across the link cable network.

An important thing to note is that all Game Boys transfer data across
the DMG-07 in external clock mode (bit 0 of the [SC] set to 0) with
the clock source provided by the DMG-07. Trying to send data via internal
clock mode results in garbage data and should not be used.

## Ping Phase

When the DMG-07 is powered up it will begin operation by automatically
sending out "ping" packets periodically. In order to receive
these ping packets a connected Game Boy should use external clock mode
(bit 0 of [SC] set to 0) and request a transfer (bit 7 of [SC] set to 1).

All connected Game Boys will receive 4 bytes as part of the ping packet.
Transfer of the 4 bytes is not spread evenly over the packet interval,
instead transfer is clustered at the start of the time period followed
by a much longer delay.

The power-up timing for ping packets is as follows:
- Serial Clock period: 15.95 microseconds (62.66 KHz)
- Transfer time per byte: 128 microseconds
- Delay between bytes: 1.42 milliseconds
- Transfer time for all bytes: 4.71 milliseconds
- Delay between packets: 12.29 milliseconds
- Total packet and delay time: 17 milliseconds

This means one ping packet with 4 bytes and it's subsequent delay takes a little
more time than a single Game Boy video frame.

### Ping Packet Fields

The ping data received by each Game Boy looks like this:
Byte | Value | Description
-----|-------|-------------
  1  | \$FE  | PING HEADER
  2  |   ??  | STAT1
  3  |   ??  | STAT2
  4  |   ??  | STAT3

The chart below illustrates how Game Boys should respond to bytes in a ping packet.
- Note: When a byte in the DMG-07 column is received the matching byte in the Reply
column should be loaded into the [SB] register as a reply that will be transmitted
during the next serial transfer.

Received From DMG-07 | Game Boy reply sent during next transfer
---------------------|-----------------------
PING HEADER (\$FE)	 | ACK1 = (\$88)
STAT1                | ACK2 = (\$88	)
STAT2                | RATE (Packet Timing)
STAT3                | SIZE (Packet Size)

### ACK Responses

The ACK1 and ACK2 responses use a fixed value (\$88) and are sent in response to
the PING HEADER and STAT1 bytes.

### RATE Response
The RATE setting configures packet timing and works differently in the Ping and
Transmission phases. It is sent in response to the STAT2 byte.

Note: In both phases the value \$00 for RATE has special behavior where it does not
change the speed, so it should not be used.

#### RATE in Ping Phase
In Ping phase RATE only adjusts the delay between packets and changes take effect
immediately upon the next packet. The timing is calculated as follows:
```
Delay between packets = (12.2 milliseconds) + ((RATE & 0x0F) * 1 millisecond)
```
Where:
- Transfer time for all bytes: 4.71 milliseconds (always)
- Delay between packets: 12.2 to 27.21 milliseconds

This yields a range of 12.20 to 27.21 milliseconds for the total packet and delay time.

#### RATE in Transmission Phase
In Transmission phase the Clock Rate setting is determined by the **last** RATE value
transmitted before exiting Ping mode. The timing is more complex and is calculated
as described below.

First determine the delay between packet bytes:
```
Delay between bytes = ((RATE >> 4) x .106 milliseconds) + 0.887 milliseconds
```

Then the total packet and delay time will be whichever of the following is larger:
```
((RATE & 0x0F) x 1 milliseconds) + 17 milliseconds
or
(Transfer time per byte + Delay between bytes) x Byte Count) + (between .36 to 2.15 milliseconds)
```
Where:
- Transfer time per byte: ~0.128 microseconds
- Byte count: 4, 8, 12 or 16 depending on the setting for SIZE
- It is not yet understood how to determine the additional amount added at the end of the formula.
  
This yields a range of 17.0 to 41.6 milliseconds for the total packet and delay time.

### SIZE Packet Size setting
SIZE sets the number of usable data bytes sent by each Game Boy during a packet
in Transmission phase. It is sent in response to the STAT3 byte.

The total number of bytes broadcasted by the DMG-07 in a packet will be SIZE x 4.
For example if SIZE is 3 then the total packet size is 12 bytes (3 x 4). 

The range of values which work without issue is 1 to 4.

### Ping Connection Status

The 3 STAT bytes sent by the DMG-07 indicate the current connection status of all
the Game Boys. Each byte is usually the same, however, sometimes the status can
change midway through a ping, typically on STAT2 or STAT3. 

Each STAT byte has the following fields:
Bit | Name
----|------------------------
 7  | Player 4 Connected
 6  | Player 3 Connected
 5  | Player 2 Connected
 4  | Player 1 Connected
0-2 | Player ID (1-4)

The Player ID values are determined by whichever port a Game Boy is connected
to. As more Game Boys connect and properly reply to pings, the upper bits of
the STAT bytes are turned on.

In this way, each Game Boy broadcasts across it's presence across the link
cable network. It also acts as a sort of acknowledgement signal, where software
can drop a Game Boy if the DMG-07 detects an improper response during a ping, or
a Game Boy simply quits the network.

The upper-half of STAT1, STAT2, and STAT3 are updated based on the ping responses
to show when Game Boys are "connected". If for whatever reason, the ping responses
are not sent, the status bits are unset.

Some examples of ping packets sent byte the DMG-07 are shown below:
Packet        | Description
--------------|-------------------------------------------------------
`FE 01 01 01` | Ping packet received by Player 1 with no other Game Boys connected.
`FE 11 11 11` | Ping packet received by Player 1 when Player 1 has connected.
`FE 31 31 31` | Ping packet received by Player 1 when Players 1 & 2 have connected.
`FE 71 71 71` | Ping packet received by Player 1 when Players 1, 2, & 3 have connected.
`FE 62 62 62` | Ping packet received by Player 2 when Players 2 & 3 are connected (but not Player 1).

It's possible to have situations where some players are connected but others
are not; the gaps don't matter. For example, Player 1 and Player 4 can be
connected, while Player 2 and Player 3 can be disconnected (or non-existent,
same thing); most games do not care so long as Player 1 is active, as that
Game Boy acts as master and orchestrates the multiplayer session from a
software point of view. Because of the way the DMG-07 hardcodes player IDs
based on which port a Game Boy is physically connected to, in the above
situation Player 4 wouldn't suddenly become Player 2.

## Transmission Phase


### Entering Transmission phase

In ping phase when the connected Game Boys are ready then one of them (typically Player 1)
should send the Begin Transmission sequence, which is 4 bytes of \$AA in a row
(`AA AA AA AA`). This causes the DMG-07 to switch to the transmission phase.

As soon as the third consecutive \$AA byte is received by the DMG-07 it will begin
transmitting four bytes of \$CC in a row (`CC CC CC CC`). This is the indicator that
connected Game Boys should use for switching to transmission mode.

After the DMG-07 finishes sending the indicator packet of \$CC bytes it will immediately
begin sending data packets and the transmission phase RATE and SIZE settings take effect.

The following chart is an example of switching from Transmission back to Ping phase.
- The SIZE setting is 1, meaning a total packet size of 4 bytes (1 x 4).
- The other 3 connected Game Boys here all send \$A5 for their contribution to the shared packet.
- Note: When a byte in the DMG-07 column is received the matching byte in the Reply column
should be loaded into the [SB] register as a reply that will be transmitted during the next
serial transfer.

Packet Byte | Received From<br>DMG-07 | Game Boy reply sent <br>during next transfer | Meaning
------------|-------------------------|----------------------------------------------|--------
Byte 1 |$FE | $88 | PING HEADER and ACK1 reply by Game Boy
Byte 2 |$11 | $88 | STAT1 and ACK2 reply by Game Boy
Byte 3 |$11 | $10 | STAT2 and RATE (\$10) reply by Game Boy
Byte 4 |$11 | $01 | STAT2 and SIZE (\$01) reply by Game Boy
 |  |  | 
Byte 1 |$FE | $AA | Game Boy initiates switch to transmission (4 x \$AA)
Byte 2 |$11 | $AA | 
Byte 3 |$11 | $AA | 
Byte 4 |$11 | $AA | 
 |  |  | 
Byte 1 |$CC | $00 | Start of transmission mode indicator from DMG-07 (4 x \$CC)
Byte 2 |$CC | $00 | 
Byte 3 |$CC | $00 | 
Byte 4 |$CC | $00 | Final transmission mode indicator from DMG-07
 |  |  | 
Byte 1 |$AA | $12 | First data packet from DMG-07 (with random data)
Byte 2 |$00 | $00 | 
Byte 3 |$00 | $00 | 
Byte 4 |$D6 | $00 | End of first data packet

### Transmission Protocol

The protocol is simple: At the same time the DMG-07 is sending data it received
during the previous packet the Game Boys are replying with their new data in
response. In effect, while receiving old data, Game Boys are supposed to pump
new data into the network.

Data received by the DMG-07 is buffered until it is broadcast during
the next packet. This means there is a 1 packet delay between when the Game Boys
send data and when they all receive the packet which combines all their data together.


For example, say the packet size is 2 bytes; the flow of data for two consecutive
packets would look like this.
- The format shown for a player byte is P\[player num\].\[packet num\], so P3.1 is player 3, packet 1.
- Note: When a byte in the DMG-07 column is received the matching byte in the Reply column
should be loaded into the [SB] register as a reply that will be transmitted during the next
serial transfer.

Packet Byte | Received From<br>DMG-07 | P1 reply      | P2 reply      | P3 reply      | P4 reply
------------|-------------------------|---------------|---------------|---------------|-----------
1           | P1.0 (byte 1)           | P1.1 (byte 1) | P2.1 (byte 1) | P3.1 (byte 1) | P4.1 (byte 1)
2           | P1.0 (byte 2)           | P1.1 (byte 2) | P2.1 (byte 2) | P3.1 (byte 2) | P4.1 (byte 2)
3           | P2.0 (byte 1)           | 0             | 0             | 0             | 0 
4           | P2.0 (byte 2)           | 0             | 0             | 0             | 0
5           | P3.0 (byte 1)           | 0             | 0             | 0             | 0 
6           | P3.0 (byte 2)           | 0             | 0             | 0             | 0
7           | P4.0 (byte 1)           | 0             | 0             | 0             | 0 
8           | P4.0 (byte 2)           | 0             | 0             | 0             | 0
Next Packet | | | | 
1           | P1.1 (byte 1)             | P1.2 (byte 1) | P2.2 (byte 1) | P3.2 (byte 1) | P4.2 (byte 1)
2           | P1.1 (byte 2)             | P1.2 (byte 2) | P2.2 (byte 2) | P3.2 (byte 2) | P4.2 (byte 2)
3           | P2.1 (byte 1)             | 0             | 0             | 0             | 0 
4           | P2.1 (byte 2)             | 0             | 0             | 0             | 0
5           | P3.1 (byte 1)             | 0             | 0             | 0             | 0 
6           | P3.1 (byte 2)             | 0             | 0             | 0             | 0
7           | P4.1 (byte 1)             | 0             | 0             | 0             | 0 
8           | P4.1 (byte 2)             | 0             | 0             | 0             | 0


All connected Game Boys should send their data into the buffer during the first few
transfers. Here, the packet size is 2 bytes, so each Game Boy should submit their data
during the first 2 transfers. The other 6 transfers don't care what the Game Boys send;
it won't enter into the buffer. The next 8 transfers return the data each Game Boy
previously sent (if no Game Boy exists for a player, that slot is filled with zeroes).

When the DMG-07 enters the transmission phase, the buffer is initially filled
with garbage data that is based on output the master Game Boy had sent during
the ping phase. At this time, it is recommended to ignore the first packet
received, however, it is safe to start putting new, relevant data into the
buffer.

## Restarting Ping Phase

It's possible to restart the ping phase while operating in the transmission
phase. To do so, any connected Game Boy can send the Ping Restart sequence, which
is 3 or more \$FF bytes in a row. This causes the DMG-07 to switch back to the ping
phase.

After the third consecutive \$FF byte is received and transmission of the current packet
is completed the DMG-07 it will begin transmitting a packet where all the bytes are set
to \$FF. This is the indicator that connected Game Boys should use for when to switch back
to ping mode.

As soon as the third consecutive \$FF byte is received by the DMG-07 it will begin
transmitting a packet where all the bytes are set to \$FF. This is the indicator that
connected Game Boys should use for when to switch back to ping mode.

To avoid false positives the Game Boys should only perform a switch to ping after
receving an entire packet of consecutive \$FF bytes. For example, if the SIZE
setting is 3 then the total packet size is 12 bytes (3 x 4), which will be the
number of consecutive \$FF bytes to require from the DMG-07 for a switch.

After the DMG-07 finishes sending the indicator packet of \$FF bytes it will immediately
begin transmitting ping packets.

The following chart is an example of switching from Transmission back to Ping phase.
- The SIZE setting is 1, meaning a total packet size of 4 bytes (1 x 4).
- The other 3 connected Game Boys here all send \$A5 for their contribution to the shared packet.
- Note: When a byte in the DMG-07 column is received the matching byte in the Reply column
should be loaded into the [SB] register as a reply that will be transmitted during the next
serial transfer.

Packet Byte | Received From<br>DMG-07 | Game Boy reply sent <br>during next transfer | Meaning
------------|-------------------------|----------------------------------------------|--------
Byte 1 |$81 | $81 | Game Boy sends it's last transmission data (\$81)
Byte 2 |$A5 | $00 | Data from Player 2 (\$A5)
Byte 3 |$A5 | $00 | Data from Player 3 (\$A5)
Byte 4 |$A5 | $00 | Data from Player 4 (\$A5)
|  |  | 
Byte 1 |$81 | $FF | Game Boy initiates ping restart (4 x \$FF)
Byte 2 |$A5 | $FF | 
Byte 3 |$A5 | $FF | 
Byte 4 |$A5 | $FF | 
|  |  | 
Byte 1 |$FF | $00 | Start of switch to ping indicator from DMG-07 (4 x \$FF)
Byte 2 |$FF | $00 | 
Byte 3 |$FF | $00 | 
|  |  | 
Byte 4 |$FF | $00 | Final switch to ping indicator from DMG-07
Byte 1 |$FE | $00 | Now returned to ping phase, start of first Ping packet
Byte 2 |$01 | $88 | 
Byte 3 |$01 | $88 | 
Byte 4 |$F1 | $00 | 
