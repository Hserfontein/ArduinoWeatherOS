ArduinoWeatherOS
================

Arduino Uno, 433MhzRx and OS WMR86 Weather Station

This project allows the 433MHz signals from an Oregon Scientific WRM86 weather station to be intercepted and decoded  into simple decimal values using an Arduino Uno. The values for Wind Direction, Wind Speed, Temperature, Humidity and Rainfall are then sent via the USB/Serial port on the Arduino to the host computer.  In my case I use a Python program to interpret this CSV formatted string of characters and plot the above parameters for display on my website http://www.laketyersbeach.net.au/weather.html

The Arduino listens continually for the three WMR86 sensor's broadcasts and merges them every minute for an output of a CSV string.  There are three transmitters in the WMR86 package, Wind Direction+Wind Speed (average and gusts), Temperature+Humidity, and Rainfall (cumulative total and rate).  They each have different periods between transmission eg Wind every 14 Seconds, Temp/Hum amd Rainfall at longer periods.  This does cause overlap of sensor transmissions and inevitable corruption of the packet, however the protocol does have a simple arithmetic checksum based on nybbles that helps eliminate most bad data.  However the chance of substitution errors causing bad data to go un-detected is much higher than if a higher bit CRC based on a polynomial process was used.  Range validation would be necessary (in the Python or Arduino) to check if the resulting readings were sensible to improve reliablity.  This program reports every minute for all three sensors whether a good packet has been received within that minute or not, so the Python program can sum the good and bad packets each minute and report the relative numbers each week in an email.

The Ardunio algorithm to decode the Macnchester protocol uses timed delays to sample the waveform from the 433MHz Rx and not interrupts and direct measurement of waveform transitions.  This has a number of implications.  The main one is that Arduino is continually sampling and analysing the incoming waveform.  As it is a dedicated processor in this application and has no other function, this is not a problem. The benefit is that it does simplify the reception and analysis of the waveforms.  The simple decoding strategy should also be worth studying by anyone else attempting to leaverage other systems that use Manchester encoding. 

The Manchester protocol is a bias neutral protocol where each bit is composed of a high and low signal of equal length (or close to it).  This means that a transmitter can be totally keyed off and on (ie the 433Mhz signal is mot modulated, but is either fully transmitting or 'silent')and no matter what sequences of 1's and 0's are transmitted, on average the Tx will consequently be on for the same time as it is off.  A data bit in a Manchester waveform always has a high and low signal component for both encoding 1's and 0's.  So a Bit Waveform for a Data 1 is a high signal followed by a low signal (hi/lo), and a Bit Waveform for a Data 0 is a low signal followed by a high signal (lo/hi). Fro the OS Weather station each high and low signal duration are both about 430uS.  So a full Bit Waveform will last about 860uS. (as will be seen later the tolerance in decoding these timings is quite wide).

The cheap 433Mhz Rx's available have a simple Automatic Gain Control and Bias Detection built in. The AGC allows the sensitivity to be maximised wwith low signals, and adjusted back when a stronger 433MHz signal is received.  The Bias Detection allows the average of the output signal voltage to be sampled and applied to one half of a comparator, the other comparator input has the received signal.  With Manchester protocol the on/off ratio is equal, so the Bias Detection will be at voltage half way, consequently the transitions of the signal from on to off and back again, can produce very clean logic signals with simple circuitry. This Manchester decoding program relies on the timing of transitions, rather than the overall shape of the waveform (ie it does not have to be a very clean square wave to work).

In other words, the decoding program needs to be able to determine whether it has seen a 430uS 'no signal' followed by a 430uS 'on signal' and so has received a 0, OR has received a 430uS 'no signal" followed by a 430uS 'on signal' and so received a 1. 'on signal' makes the Rx output go 'hi' and 'no signal' makes the Rx output go 'lo'.

How is this decoded?

The first very important prior knowledge to have is what polarity of Manchester encoding this system uses.  Arguments are put forward for both possibilities as being correct, but either polarity works as good as the other and with a simple audio sampler is simple to work out.  The polarity used by OS is that a Data 1 is hi/lo and Data 0 is lo/hi.

However any hi/lo transition may either indicate the middle of a Data 1 Bit Waveform and could mean a 1 has been sent, or possibly just the transition between two Bit Waveforms and as such not indicate anything. Similarly any lo/hi may indicate the middle of a Data 0 Bit, or just a meaningless transition between two Bit Waveforms.

Curiously a long string of Data 1's will have the same looking waveform as a long string of Data 0's.  Whether they are 1's or 0's depends on where we begin to analyse, we would need some sort of marker, or unambiguous beginning point. More information later...

Critically the only guaranteed meaningful transition of states therefore always occurs in the middle of the Bit Waveform.  Should a Data 1 be followed by a Data 0, then the signal will be hi/lo-lo/hi, and for a Data 0 followed by a Data 1, the signal will be lo/hi-hi/lo (where the - (dash) separates the Bit Waveforms and the / (slash) indicates the middle of the Bit Waveforms).  Note that in the previous two examples the logic levels do not change between the two Bit Waveforms.  If we have a Data 1 followed by a Data 1, it will be hi/lo-hi/lo and a Data 0 followed by a Data 0 it will be lo/hi-lo/hi.

The last two examples above, Data 11 and  Data 00, have transitions at both the - and the /. (Hence curious effect alluded to above) So if there is a change from a Data 1 to a Data 0, or Data 0 to a Data 1, then there will no transition at the - point, however if there is a 1 followed by a 1, or a 0 followed by a 0, then there will always be a transition at the - point (but opposite polarity). Any transition between the Bit Waveforms (ie the - above) can be thought of as 'optional" and really only required to make sure the transition at the middle of the next Bit Waveform (ie the / above) is correct for the data bit being represented.  Consequently is essential to concentrate the algorithm around the center of the Bit Waveform.

Here we need to digress to RF practicalities... 

Part of the practical application of the Manchester protocol combined with simple 433MHz Tx/Rx combo's is use a header sequence of bits, usually 30 or so 1's.  This stablizes the Rx AGC and establishes the Bias Detection point so the simple 433Mhz Rx has a good chance of settling down and producing a clean logic waveform after say 10 on/off transmissions.  The decoding program can then sample the Bit Waveform by synchronising (ie looping and waiting) the algorithm to the on/off transition, which is the midway point of a Bit Waveform for a 1.  This is how it works for the OS protocol, as this is its polarity, hi/lo=1, lo/hi=0 (see above).

![alt text](AGC_Starting.png?raw=true "Rx AGC kicking in") Graphic 1: The AGC is stabilising fairly quickly here.

This Graphic 1 below is showing a stream on 1's as the header.  Now the algorithm is expecting a stream of Data 1's and to begin with, and looking for any hi to lo transition on the Rx and assuming it is the middle of a Data 1 bit Waveform.  After detecting a hi/lo transition it begins to check the waveform. 

![alt text](header.png?raw=true "Header Manchester Encoding") Diag 1

So section A represents random noise on the RX.  Section B represents the AGC kicking in and at the start of C section the first hi to lo transition is found that has a regular Bit Waveform following it, such that during section C the processor has counted the minimum 15 properly formed data Bit Waveforms and arriving in the right frequency, such that by the end of the C section, it is declared a valid header.  The section D could be shorter or longer, but at this stage the program swaps over and is assuming it is synched to a valid stream of ones, and when section E arrives it can detect it as a zero, the synch bit, and then goes off to further decode the packet of data beginning in section F and could go on for 80 or more bits of data bits of zeroes or ones.

How does it know it is properly formed and timed Bit Waveforms?  To filter out noise, the algorithm resamples the Rx output about a 1/4 of a Bit Waveform later, to see if it is still lo.  If is lo then it was possibly a genuine middle of a 1 Bit Waveform, however if it is not a lo then it was not a genuine hi/lo middle of a 1 Bit Waveform, and the algorithm begins the search for a hi/lo transition all over again.  However if this preliminary test is true, it has possibly sampled a midpoint of a 1, so it waits for another half a Bit Waveform.  This timing is actually 1/4 of the way into the next Bit Waveform, and because we know we are looking for another 1, then the signal should have gone hi by then. If is not hi, then the original sample, that was possibly the mid point of a 1 Bit Waveform is rejected, as overall, it has not followed the 'rules', and the search (looping) for then next hi/lo transition begins at the start, all over again. Here is a diagram to highlight the previous ideas.

![alt text](manchester.png?raw=true "Manchester Encoding of data bits") Diag 2

The pink lines are the signal arriving from the 433MHzRx. The long vertical blue lines (eg at A & C) are indicating the start and end of each Bit Waveform.  These are partially covered by the pink signal trace if they coincide.  This diagram show 7 bit patterns. B is positioned at th emiddle of a Bit Waveform, and these are also indicated by the M's.  The data contained in each bit pattern is determined by the direction of the transition at M, the middle of the Bit Waveform and not by what happens at the finish and start of each Bit Waveform.  

This simple filtering allows the program to detect and count the number of successfully detected Data 1's received, and once a minimum has been counted in sequence, then the program can assume it has a valid header coming in. This sampling, by looking for transitions and waiting periods of time to sample, is also applied equally to all subsequent 1's and 0's received and can eliminate badly formed packets if the waveform pattern of the Manchester encoding is not followed.  Hence it forms a simple but effective digital filter that greatly reduces spurious results from random or unwanted 433MHz signals. 

Okay so we can detect a header... Let's say the Tx sends a header of 30 Data 1's, and it takes 5 Data ones to stabilise the Rx, then we have 25 left to detect.  If the program locks in and begins receiving the Data 1's and counts to the minimum require for a header, such as 15, then where will be 10 Data 1's more to go.  However we cannot be sure how many Data 1's it will take to stabilise the Rx, it maybe 5 some days, maybe 7 or 3 other days, so we need a marker to indicate the header has finished and the data or payload is beginning.

This is where the next bit of inside information is required, but this practice is fairly standard.  As any excess header Data 1 bits are being soaked up, the program is now looking for the header Data 1's to stop and a Data 0 to be detected. This 0 indicates that the data payload is about to begin.  The rules for interpretting the hi and lo's change slightly here.  As we know that we are dealing with a valid header, and we have a series of Data 1's detected, then the program is definitely synched to the Mid-point of the Bit Waveform.

We can subsequently decode both Data 1 and 0's by continuing to sample at 3/4 time (Mid+1/4 or T2 below), and across into 1/4 time (Mid+1/4+1/2 or T3 below) of the next Bit Waveform, to check the integrity of the waveform against the Manchester protocol, but also more importantly we can now predict what the next transition should be, synch to it, and when satisfied with the filtering/sampling results (which continues for every bit received) accumulate the data payload bits into into bytes.
         
The transitions at E/S are do not carry data, but if they occur they are to make sure the subsequent transitions that must occur at the Mid points match the bit stream correctly (you can think of E/S setting up the data transition for the next Mid). The critical aspect of this processing is that it is always synching to the Mid point of the Bit Waveform, where the critical data indicating transitions occur.  This makes the decoding of the bit stream very tolerant any errors in sampling the waveforms. This allows the calculations between receiving bits to take different lengths of time and still not effect the reliablity. The delays can be altered plus or minus 15% (or more) and still get reliable reception. 


The synchronising 0 bit and may or may not be included in the byte boundaries.  Some implementations send a single 0 bit then all the bytes of information, whereas the OS example we are dealing with here just makes sure the first bit in the first byte sent, is always a 0.

Once this "0" is detected then the data stream can be considered synchronised with the detector.  The program to decode the Oregon Scientific Weather Station's sensors then decodes a sequence of bytes that have data encoded in both straight binary and other times, BCD.  The number of bytes per packet for each transmitter can also vary.  If the program must be able to recognise each transmitter and change the number of bytes expected for each type.  To do this the program begins downloading all bytes (more on that later) and identifies the ID bytes within the first 2 bytes, and when a particular sensor is detected it immediately sets the exact number of bytes expected.  So temp/Humidity decoding requires 10 bytes (but only uses 9), Anemometer decoding requires 10 bytes and uses them all, and the Rainfall sensor decoding requires 11 bytes but only needs the four most significant bits of the last byte.   If 11 bytes were accepted all the time, the Rainfall would work, but the other two would fail as their signals return to random noise after 10 bytes and this would cause them to be rejected.

Once a data packet is received it is given a checksum check before being declared valid.  We now need to diverge, and return back to the way the bits arive in the data stream and how they are best stored.  The easiest way to explain this is to give an example.  Let's number the raw Rf incoming bits in order, zero arrives first -

`0 1 2 3 4 5 6 7` , and this how it is best to rearrange those bits
`3 2 1 0 7 6 5 4` , so the first bit 0 is actually moved to the fourth position, the next bit is moved to the fifth position and so on and this wraps around, so all the incoming bits are stored in new postions in the stored byte array.

Why Oregon Scientific chose this rearrangement is best left up to them to explain, but applying this swapping of positions makes all the data in the stored bytes array so much more logical as well. Binary numbers are found in the correct ascending order etc.  Plus when it comes to the Checksum, it is also simple to execute as well.  Take each 4 bit nybble in the data packet (excluding the checksum byte) and add them up as an 8 bit result.  This will result in a byte that can be compared to the last byte in the packet, the checksum byte.  The number of nybbles for  Temp/Humidity is 16, Anemometer 18, and rainfall 19 (NB rainfall Check Sum byte, is made up of nybble 20 and 21, ie it bridges the byte boundary).

Once the bits and bytes are stored in this fashion then the checksum becomes trivial, just add up the nybbles and compare to the checksum byte. Reject any packet that does not workout.  A simple checksum like this is prone to substitution errors getting through undetected (ie one byte has an error bit, but is balanced out by an inverted error bit in another byte with the same place value.  Cyclic reduncancy methods for checking data validity are much more robust than the simple aithmetic checksums, but they are not used on the OS Sensors. However by the time the two bytes for the sensor type ID, and the rolling ID code for a particular sensor are put aside, the critical data that changes is down to  5-7 bytes, and probably the checksum for such a small sample is quite acceptable (though if you intend to use any of these sensors for mission critical stuff you may like to disagree on that opinion).  Fortunately for me it was easy to program.

Once the bytes for a particular valid packet are stored in the array, they are processed.  Some have conversion factors applied eg binary weighted data is turned into decimals, and the Anemometer wind speed is converted from metres per second to kilometers per hour.  Others such as relative Humidity are provided directly in two BCD characters.  Eventually a selection of the data is formatted into ASCII values in a CSV string and sent out via the serial port (over the USB connection).  My Python program on the www server then further processes this string into averages and graphs etc

Before any CSV is sent though, the program checks that it has received a valid sample from each of the three sensors, and  only begins sending CSV's when Therm/Hum, Anemometer and Rainfall have all been logged in for valid values. This avoids some values being valid and other being at zero.

This description should give you a good idea of how the OS V3.0 protocol works and how my program tackles decoding that protocol.  It is not very sophisticated, when it is all shown now, but was quite a headache to work through originally.  The OS Sensors are a good balance of quality engineering, accessible protocols and reasonable price.  Really getting a grip on the Manchester protocol was been a major hurdle, but now is well under control.  I am hoping my program and this presentation above will help others tackle this protocol in their own projects.

I hope you enjoy using them as I do, cheers, Rob


May be continued...........
