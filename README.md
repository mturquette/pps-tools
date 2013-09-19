# pps-tools

A suite of Python scripts for remote control and acquisition of energy
consumption data from power supplies.

These tools support the Atten PPS-3205t-3s power supply. There are
variants and rebadged versions of this supply based on region. A
non-exhaustive list:

* [Atten PPS-3205t-3s](http://www.atten.eu/power-supply/atten-pps3205t-3s-programmable-power-supply.html)
* [Atten PPS-3203t-3s](http://www.atten.eu/atten-pps3203t-3s-programmable-power-supply.html) (max 1A output)
* [Tenma 72-5678](http://www.newark.com/tenma/72-8795/programmable-dc-power-supply-32v/dp/32T0685)
* [Circuit Specialists CSIPPS55T](http://www.circuitspecialists.com/programmable-bench-power-supply-csipps55t.html)

## Theory of (entirely stupid) operation

The firmware on these supplies is terrible. There is no read method for
gathering power data; instead a 24-bit packet (defined below) must be
sent to the supply to configure it, only then will a 24-bit packet
containing measurement data be sent back. That packet contains a single
sample of instantaneous current consumption for each channel.

Sampling rate is thus I/O bound; the speed with which reads &
writes are performed over UART directly impacts the number of samples
gathered on the host PC. There seems to be no target-side aggregation or
averaging of samples. Each write/read transaction gives us a single
measurement for each channel.

These limitations suck and have informed the design of these tools. In
particular pps-monitor does absolutely nothing besides writing
configuration data, reading back measurement data and printing the
results to STDOUT. It is recommended that the user configure the supply
to run at 19200 baud (the fastest rate) to increase sample rate.

## Communication protocol

A document floating around on the interwebs provides just enough
information to understand the 24-bit packet format for sending data over
the wire. A copy of this doc can be found in my [Google
Drive](https://docs.google.com/file/d/0Bzx9x7R7ZLSiTE4yeWxGR1dGeXc/edit?usp=sharing).

A byte-for-byte breakdown of the packet:
<table>
    <tr><td>Byte #</td><td>Name<td>Description</td></tr>
    <tr><td>00</td><td>Start</td><td>Packet header; always 0xaa</td></tr>
    <tr><td>01</td><td>Address</td><td>Unused, zero-fill</td></tr>
    <tr><td>02</td><td>Channel 1 Voltage High Byte</td><td>Multiplied by 2.56V</tr>
    <tr><td>03</td><td>Channel 1 Voltage Low Byte</td><td>Multiplied by 0.01V</tr>
    <tr><td>04</td><td>Channel 1 Current High Byte</td><td>Multiplied by 0.256A</tr>
    <tr><td>05</td><td>Channel 1 Current Low Byte</td><td>Multiplied by 0.001A</tr>
    <tr><td>06</td><td>Channel 2 Voltage High Byte</td><td>Multiplied by 2.56V</tr>
    <tr><td>07</td><td>Channel 2 Voltage Low Byte</td><td>Multiplied by 0.01V</tr>
    <tr><td>08</td><td>Channel 2 Current High Byte</td><td>Multiplied by 0.256A</tr>
    <tr><td>09</td><td>Channel 2 Current Low Byte</td><td>Multiplied by 0.001A</tr>
    <tr><td>10</td><td>Channel 3 Voltage High Byte</td><td>Multiplied by 2.56V</tr>
    <tr><td>11</td><td>Channel 3 Voltage Low Byte</td><td>Multiplied by 0.01V</tr>
    <tr><td>12</td><td>Channel 3 Current High Byte</td><td>Multiplied by 0.256A</tr>
    <tr><td>13</td><td>Channel 3 Current Low Byte</td><td>Multiplied by 0.001A</tr>
    <tr><td>14</td><td>RESERVED</td><td>Zero-fill</td></tr>
    <tr><td>15</td><td>Output</td><td>Bitmask for toggling per-channel output (HIGH bit == ON)</td></tr>
    <tr><td>16</td><td>Alarm</td><td>Set to enable alarm, clear to disable</td></tr>
    <tr><td>17</td><td>RESERVED</td><td>Zero-fill</td></tr>
    <tr><td>18</td><td>OCP</td><td>Set for constant current output, zero for over-current protection</td></tr>
    <tr><td>19</td><td>Connection</td><td>0 - independent output, 1 - in series, 2 - in parallel</td></tr>
    <tr><td>20</td><td>RESERVED</td><td>Zero-fill</td></tr>
    <tr><td>21</td><td>RESERVED</td><td>Zero-fill</td></tr>
    <tr><td>22</td><td>RESERVED</td><td>Zero-fill</td></tr>
    <tr><td>23</td><td>Calibration</td><td>Unused, zero-fill</td></tr>
</table>

## Utilities

### pps-config

Takes in a TTY argument and configuration options for controlling the
power supply. This tool writes its output to the configuration file
(~/.config/pps-tools/config by default, or ~/.pps-tools.config, or a
file specified by the user). Normally this tool does not communicate
directly with the power supply but instead it can inform the pps-monitor
process that it needs to update its cache of configuration data. There
is an option for initiating monitor mode from this command if not
already started.

### pps-monitor

Takes in a TTY arguement and an optional channels argument. This tool
reads in configuration data from the config file, programs the power
supply to operate via that same configuration and then monitors power
consumption data until interrupted with SIGINT. Note that pps-config can
be invoked while pps-monitor is running but monitoring will be disabled
during the critical section where pps-monitor updates its configuration
cache (bounded by SIGUSR1 and SIGUSR2).
