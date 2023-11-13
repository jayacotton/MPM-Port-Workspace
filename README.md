# MPM-Port-Workspace
Porting DRI MP/M os to S100 Z180 SBC board.

This is all WIP at this time.
Below is my journy to getting MPM to load and run.

I have added 2 Z180 manuals to the repo.  These are references for the Z180 
port.

NOTE:  All the information in the repo was found on the internet in other 
repo's.  Names TBD.

On the main page is all the documentation required to port and run MP/M on a Z80.

Things learned so far.  MP/M must be built and launched from CP/M 2.2.  CP/M 3 will
crash when you try to use the build tools.  However,  all the binaries required are
already built in the tree.  

It is possible to do all  the build functions using simh altairz80 simulator.  But do
use CP/M 2.

If you want to get your feet wet on MP/M  try z80pack first.  Udo has a running MP/M
port that works with his z80 simulator.  I also suspect you can do all the builds on
simcpm with CP/M 2 boot disk.

The main porting task is isolated to one file (2 really)  RESXIOS.ASM  (or BNKXIOS.ASM)
The porting guide has 2 examples and there is a file RESXIOS.ASM that can be hacked on.
There is a file called LDRBIOS.ASM  that will have to be modified (maybe) I will update
when I get there.

MP/M uses interrupt driven input and places input bytes into each consol buffer as they
arrive.  The scheduler code can then start up the user task and feed it data or it can
wait for an EOL event or maybe just schedule to user via round robin schedule.  Disk i/o
will also benifit from interrupt code if available.  The printer and consoles are polled
output.  There is a real time clock module that is a must or the schedule will not work.

The bin directory has a complete binary distro of MP/M 2 with the exception of XIOS

The isx directory has source code for ISX.  A tool for changing CP/M 2.2 to an ISIS 
look alike.

Progress:

After reading through the sample RESXIOS.ASM code from the bin directory, which is the
same as the RESXIOS.ASM in the CONTROL directory, and looks the same as whats in the porting
guide.  

I have decided to rewrite the SIO code (for 1 serial port), the timer code for the
z180 timer, and the disk code so it works with SD card on the S100 Z180 SBC board.

Its not totally obvious how the interrupts are handled by the code, it looks like the output
part is polled and the input is buffered by an interrupt routine.  That will require some tweeking
also.  

The timer code in the sample driver looks like its running the 88-rtc board.  This well need a
lot of tinkering.

The disk driver is a total blowout.  I will need to recode all of it.

The good part is, I have RomWBW to guide the way.

As a point of interest, MP/M must be started from a CP/M 2.2 environment.  No big problem there
but, its not going to interface with RomWBW at all.  This is due to an unfortunate issue with the
way RomWBW is designed to handle interrupts from the serial ports.  Perhaps after we get a simplified
version of MP/M running, it will become clear how to move forward with some kind of integration.

The bottom line issue is MP/M  uses the receive interrupts on the serial ports to figure out which user
is demanding CPU time.  So, no interrupts, no service.  I suppose you could poll the input from the
timer loop and figure it out that way, but the responce time would surely suck since CPU intensive
code gets more time by default.

I am currently working on the mpmldr program.  I have managed to build it from scratch on the Z180 S100 SBC
board, but can't use the
mpmldr.sub file since the build droppings fill up the a: drive.  So doing the build one line at a time.
I found a potentally nasty bit of code in the loader. 

```
  declare mon1 literally 'ldmon1';
  declare mon2 literally 'ldmon2';

  mon1:
    procedure (func,info) external;
      declare func byte;
      declare info address;
    end mon1;
```

This in it self is not a problem.  However....

```

	public	ldmon1,ldmon2

ldmon1	equ	0d06h
ldmon2	equ	0d06h

offset	equ	0000hcrtst:                  

fcb	equ	005ch+offset
fcb16	equ	006ch+offset
tbuff	equ	0080h+offset
	public	fcb,fcb16,tbuff
```

That is a bug looking for a place to happen.

I have not figured out what is at 0xd06 yet, bunch of assembly code.
No idea what its doing, but it will eventually get to my console byte write 
code.  It does not write on the console ... yet.

This one does not lead to a bug.  The address is a decoder that figures out what
the function code is and branches to the support code for it.  So this seems to
be a bit of code that will not move, just must remember that its there.

Another gotcha.  The plm80.lib or mpmldr.plm has code that calls the 
crtsts code.  

```
crtst:                  ; crt: status
        in Z180STAT0 
        ani 080h  
        rz
        ori 0ffh 
        ret

```

The problem is that the caller says 

```
  0E06  CALL 1706                                                               
  0E09  ANI  01                                                                 
  0E0B  RZ
```

So, I guess the caller does not trust the crtst code to
do the correct test.  For fun I will punch this code out
and see what happens.

After a long 2 weeks, I have finally got console output working.  First, thanks to Wayne for providing the
hint that I needed.  The bug is that z180 internal registers must be addressed with 16bit i/o operations.
In addition the B register must be set to zero.  Also, many funcions up stream do not know about this 
convention, and they use the BC pair at a string pointer for the console messages.  So, take care to 
preserve the DE and BC registers.

I have gotten a bit hacked off at the PLM compiler.  Its really sensitive to cr/lf etc.  The compiler
also generates a lot of error messages that don't seem to be really there.  I can never figure out
what its up to.  SO, mpmldr is being rewritten in C using the z88dk compiler.  

So far I am almost ready to make the jump into the startpoint.  There is a potential compiler bug
that I am rather supprised to run into, at the moment I am updating z88dk to latest to see if the
bug clears, else I will report it.  I just can't beleave that no one else has had a issue with this.
The thing I see is ld hl,(s+78).  On paper this looks o.k.  BUT it should be 78hex, not 78 dec.
Like I say, this bug should not be here after 25 years of compiler debug.  TBD.

Back to PLM compiler.  I split the mpmldr.sub file into 2 parts breaking at the CPM command.
The Z180 SBC is hanging at the CPM command (no idea why yet) and so this seems like a good 
place to break the build.  After getting a good compile, I just need to mess with the 
ldrbios and ldrbdos files, both 8080 assembler code.  

Jump forward a few days and now ldrbios is integrated with HBIOS and the console code is
working.  The disk code is totally busted at this point.  I tried to include code from 
biosldr.z80 and hit the wall.  I will rewrite the code and do more testing.  It is also
integrated with the HBIOS code.  I am attempting to use the code that converts head/track/sector
into LBA, since this just seems to be the best way to do it.  Also biodldr has a disk deblocking
struct that is already setup for micro SD cards.  

One of the things that is causeing an issue is the bank switching.  mpmldr does not know about
bank switching, MPM does however.  I may try to restrict MPM to the lower 32k of ram in an attempt
to simplify the mpmldr code.  I'd like to get mpm up to a limping state soon.



