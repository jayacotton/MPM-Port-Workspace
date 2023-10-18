# MPM-Port-Workspace
Porting DRI MP/M os to S100 Z180 SBC board.

This is all WIP at this time.

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

>  declare mon1 literally 'ldmon1';
>  declare mon2 literally 'ldmon2';
>
>  mon1:
>    procedure (func,info) external;
>      declare func byte;
>      declare info address;
>    end mon1;
>
>  
This in it self is not a problem.  However....

>
>	public	ldmon1,ldmon2
>
>ldmon1	equ	0d06h
>ldmon2	equ	0d06h
>
>offset	equ	0000hcrtst:                  
>
>fcb	equ	005ch+offset
>fcb16	equ	006ch+offset
>tbuff	equ	0080h+offset
>	public	fcb,fcb16,tbuff

That is a bug looking for a place to happen.

I have not figured out what is at 0x60d yet, bunch of assembly code.
No idea what its doing, but it will eventually get to my console byte write 
code.  It does not write on the console ... yet.

Another gotcha.  The plm80.lib or mpmldr.plm has code that calls the 
crtsts code.  

>crtst:                  ; crt: status
>        in Z180STAT0 
>        ani 080h  
>        rz
>        ori 0ffh 
>        ret

The problem is that the caller says 

>  0E06  CALL 1706                                                               
>  0E09  ANI  01                                                                 
>  0E0B  RZ

So, I guess the caller does not trust the crtst code to
do the correct test.  For fun I will punch this code out
and see what happens.


