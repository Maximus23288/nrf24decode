# gr-nrf24-sniffer

## How to compile the decoder
As simple as `gcc -o nrf-decoder -O3 nrf-decoder.c`. No particular dependencies. As i said, Linux only, but maybe with Cygwin or something like this it can work on Windows. Please don't ask me for support for this however.

## How to use
Compile the decoder. Make sure your SDR is connected and switched on. Create a named pipe called `fifo_grc` in `/tmp` (`cd /tmp && mkfifo fifo_grc`). Open `nrf-receiver.grc` with Gnuradio 3.8 (might also work with 3.9, untested; will not work with 3.7). Then **first** start the decoder using `cd /tmp && cat $fifo_grc | ./nrf-decoder $options` (see below for `$options`) and **then** start the receiver from inside GNU Radio (or directly start the generated Python3 code). If you forget to start the decoder first the GUI of the receiver will not show up!  
  
Once the GUI is up and running you need to select the nRF24 speed (250kbps/1Mbps/2Mbps). Note that if you need 2Mbps you first have to select 1Mbps and then 2Mbps, if you directly go from 250kbps to 2Mbps some internal test will fail and the GUI crashes... You also need to select the channel on which you want to receive. After that you can tweak some parameters of the receiver inside the GUI to get a good decoded signal on the virtual oscilloscope at the bottom of the screen. If everything looks good you can start piping data to the decoder via the named pipe by ticking the "Write to file/pipe" checkbox. You now should see some output from the decoder (depending on the selected options).  
  
Please note that using GNU Radio inside a VM with USB-passtrough is not recommanded. It may work or may not work or only work sometimes and produce weird errors including nasty crashes of Python3.

## Options of the decoder
The order of the options does not matter.
### general options
* `--spb $number` **Mandatory!** The number of samples spit out by the receiver for each bit. The value depends on the selected speed and is displayed inside the GUI. It is 8 (samples/bit) for 250kbps and 1Mbps or 6 for 2Mbps. **Note that this value must be correct or you won't see any valid packets!** Also note that due to the way the decoder is implemented this value must be somewhat high, let's say bigger or equal to 4 (?) to make it work. This is not a problem with a HackRF One that can go up to 20Msps sampling rate but could be a problem with other SDR.
* `--sz-addr $number` **Mandatory!** The size of the address-field inside the packets. This can be between 3 and 5 bytes. ($number!=5 untested)
* `--sz-payload $number` The size of the payload inside data-packets. This can be from 1 to 32 bytes and is mandatory unless you specify `--dyn-lengths`.
* `--sz-ack-payload $number` The size of payload inside *ACK*-packets. This can be from 0 (no payload) to 32 bytes and is mandatory unless you specify `--dyn-lengths`.  
   
*If `--dyn-lengths` is specified or `--sz--payload` and `--sz-ack-payload` have the same value there is no way for the decoder to distinguish between data-packets and ACK-packets with payload so a more generic output format will be used.*
### display/output options
By default the decoder will not show all the packet details but only a summary and will not spit out the packet-payload as raw bytes. You can change this using these options:
* `--disp [verbose|retransmits|none]` Show everything|just retransmits|nothing (printed to stderr). Note that option 2 requires the decoder to be able to distinguish between data-packets and ACK-packets, so `--dyn-lengths` is not allowed and `--sz-payload` must be different from `--sz-ack-payload`.
* `--dump-payload [data|ack|all]` Dump payload of data-packets|of ack-packets|of both packets on stdout. Note that the latter two options cannot be combined with `--mode-compatibility` and option 1 and 2 requires the decoder to be able to distinguish packets (see just above).
### other options
* `--mode-compatibility` Compatibility-mode for nRF2401A, nRF2402, nRF24E1 and nRF24E2 (no packet control field, no auto-ack, no auto-retransmit). See datasheet of the nRF24L01+ section 7.10. For nRF24L01+ you don't need this unless you configured your nRF specifically for compatibility (EN_AA=0x00, ARC=0, speed 250kbps or 1Mbps).
* `--dyn-lengths` Tell the decoder that data-packets and/or ACK-packets have a dynamic payload length specified inside the packet control field. For fixed payload-size use `--sz-payload $number` and `--sz-ack-payload $number` instead to allow the decoder to detect the type of a packet (data or ACK). 
* `--crc16` Use this if your wireless link uses a 2 byte CRC instead of the default 1 byte. I recommand using this with your own projects for better error-detection / less false positives (bit CRCO in register CONFIG set).
* `--filter-addr $addr_in_hex` Only consider packets for the specified address (in hex with or without leading "0x"). By default the decoder is in promiscous-mode. The size of the specified address (number of bytes) must match `--sz-addr`.
