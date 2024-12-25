A simple reaper JS rig to analyise the various Midiverb algos.

This has only been prossible from these previous works:

Hardware awesome analysis by:

https://www.youtube.com/watch?v=z4cIt1VPAjU
https://www.youtube.com/watch?v=JNPpU08YZjk
https://www.youtube.com/watch?v=5DYbirWuBaU

and the SRC code from:

https://github.com/thement/midiverb_emulator

and the web demo:

http://ibawizard.net/midiverb/

Has the following algos, hardcoded:

- Large Bright 2.5sec
- Large Warm 5Secs
- Bright 18secs
- Dark 20Secs
- Reverse 300ms
- Reverse 600ms
- Reverb Medium Bloom (from the midiflex)


![](./Images/MidiVerbJS.png)

Also added Gate/Duck control - which will either gate or duck the output from the midiverb algo.
 The Green lines on the UI are the Tap positions for the Reverse600ms algo.

 Here's my current code analysis on the algo:
 
```
function Reverse600ms()
(
// Addrs 1-8 are scratchpad locations
//
// Stereo All Pass Blocks, three per channel

// Gives a pseudo stereo field effect since the AP's are 
// tuned differently for each channel

// AP Addresses (0x08fa to 0x0709) are after the multitap addresses
// and so the AP blocks and calculations never get accidentally
// mixed up into the multitap delay line - which starts/ends at 0x3FFF to 0x0B22

// Post Delay from 0x0B22 to 0x0950 after the multitap

// Multitap delay line is 13,533 samples 580ms

LoadH(0x0009); // Load in sample from the multitap output (stored at 0xB - so a 2 sample delay)

SumH(0x08e5); // AP Right 1 23.1khz/(8FA-8E5) = 23.1khz/(21) = 1.1khz
StoreN(0x08fa);
SumH(0x08e5);
SumH(0x08e5);

SumH(0x08ae); // AP Right 2 23.1khz/53 = 435hz
StoreN(0x08e3);
SumH(0x08ae); 
SumH(0x08ae);

SumH(0x07ff); // AP Right 3 23.1khz/173 = 133hz
StoreN(0x08ac);
SumH(0x07ff);
SumH(0x07ff);

// Add the whole mono sample from 0x950 - which is very late in the delay line
// this gives the original sound - just delayed and bypasses the AP blocks.
// Comment these 2 lines out if you want just pure reverse blur in the wet signal

//SumH(0x0950);
//SumH(0x0950);

StoreP(0x0003); // Store to Left output addr

LoadH(0x0009); // Load in delay line sample from the multitap

SumH(0x07dd); // AP Left 1 23.1khz/32 = 721hz 
StoreN(0x07fd);
SumH(0x07dd);
SumH(0x07dd);

SumH(0x0790); // AP Left 2 23.1khz/75 = 308hz
StoreN(0x07db);
SumH(0x0790);
SumH(0x0790);

SumH(0x0709); // AP Left 3 231.1khz/133 = 173hz
StoreN(0x078e);
SumH(0x0709);
SumH(0x0709);

// Add the whole mono sample from 0x950 - which is late in the delay line
// this gives the original sound - just delayed and bypasses the AP blocks.
// Comment these 2 lines out if you want just pure reverse blur in the wet signal

//SumH(0x0950);
//SumH(0x0950);

StoreP(0x0005); // Store to Right output addr


// Main reverse algo

// 5 sets of summed samples - each set with increasing gain
// Each set spans a specific set of samples from early to late
// and they overlap.

// Spans:

// 1st 0x3BD7 to 0x320F
// 2nd      0x379B to 0x27B1
// 3rd             0x2981 to 0x1C01
// 4th                  0x22F1 to 0x10D6
// 5th                         0x140A to 0x0B22
//
// @GFX section draw the sets - so you can see what's going on
// easily.
// If you look at the entire set of spans - they almost have equal
// time difference and are from 1 linear set that covers the entire total span.
// Each set looks hand chosen:
//    1) To give more control the volume/gain for that set (tap density)
//    since the DSP engine can't have varible constants for each tap. 
//    2) To control the overlap with the next set - guessing to control
//    phase issues?

// Each set sum has a different opposing sign apart from the last stage
// which is a +0.5 in the LoadH(0x0009); at the start of the program

// 1st set = gain -0.5*-0.5*-0.5*-0.5*0.5
// 2st set = gain -0.5*-0.5*-0.5*0.5
// 3rd set = gain -0.5*-0.5*0.5
// 4th set = gain -0.5*0.5
// 5th set = gain 0.5 

// so it's a cascading overlapped multi tap comb filter thing :)
// Tap order in each set is irrelevant

// There's also a Predelay of 0x3ffff-0x3BD7 samples - 21ms

//Set 1 - 13 Taps
SumH(0x3bd7);
SumH(0x3ba1);
SumH(0x3b2f);
SumH(0x3a66);
SumH(0x39ed);
SumH(0x3932);
SumH(0x38ba);
SumH(0x3802);
SumH(0x3730);
SumH(0x3675);
SumH(0x355f);
SumH(0x33cf);
SumH(0x320f);
// Negate the ACC and multiply by 0.5
// Addr 1 = scratchpad area - wiped over on next process
StoreN(0x0001);

//Set 2 - 20 Taps
SumH(0x379b);

SumH(0x35ed);
SumH(0x34c7);
SumH(0x3424);
SumH(0x333d);
SumH(0x3285);
SumH(0x3173);
SumH(0x30cf);
SumH(0x303e);
SumH(0x2fc8);
SumH(0x2f5f);
SumH(0x2e74);
SumH(0x2e34);
SumH(0x2d89);
SumH(0x2d0f);
SumH(0x2bf7);
SumH(0x2b8d);
SumH(0x2a62);
SumH(0x286d);
SumH(0x27b1);
// Negate the ACC and multiply by 0.5
// Addr 1 = scratchpad area - wiped over on next process
StoreN(0x0001);

// Set 3 - 19 Taps
SumH(0x2c56);
SumH(0x2ad3);
SumH(0x2981);
SumH(0x2933);

SumH(0x27f1);
SumH(0x26d2);
SumH(0x267a);
SumH(0x2582);
SumH(0x2535);
SumH(0x2481);
SumH(0x242a);
SumH(0x23cb);
SumH(0x2292);
SumH(0x21c8);
SumH(0x212d);
SumH(0x202e);
SumH(0x1faa);
SumH(0x1e8b);
SumH(0x1c01);
// Negate the ACC and multiply by 0.5
// Addr 1 = scratchpad area - wiped over on next process
StoreN(0x0001);

// Set 4 - // 20 Taps

SumH(0x22f1);
SumH(0x209e);
SumH(0x1f01);
SumH(0x1e04);
SumH(0x1d4b);
SumH(0x1ccc);
SumH(0x1c2d);
SumH(0x1b17);
SumH(0x1ad7);
SumH(0x19db);
SumH(0x1975);
SumH(0x18c2);
SumH(0x1882);
SumH(0x17d3);
SumH(0x16ae);
SumH(0x1612);
SumH(0x14f1);
SumH(0x144f);
SumH(0x123d);
SumH(0x10d6);
// Negate the ACC and multiply by 0.5
// Addr 1 = scratchpad area - wiped over on next process
StoreN(0x0001);

// Set 5 Late - // 16 Taps
SumH(0x1744);
SumH(0x15c1);
SumH(0x140a);
SumH(0x1329);
SumH(0x12e6);
SumH(0x11b9);
SumH(0x10b0);
SumH(0x1009);
SumH(0x0f81);
SumH(0x0f1c);
SumH(0x0e36);
SumH(0x0df1);
SumH(0x0d0a);
SumH(0x0c5f);
SumH(0x0bbe);
SumH(0x0b22);
//Store it to be passed into the final AP Stereo Blocks in 2 samples time 0x9
StoreP(0x000b); 
);
```




