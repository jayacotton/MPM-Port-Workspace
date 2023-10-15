# MPM-Port-Workspace
Porting DRI MP/M os to S100 Z180 SBC board.

This is all WIP at this time.

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

