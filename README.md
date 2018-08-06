# Mfatigue.exe
stability fixes for the game Metal Fatigue

# heruistics virus scanner false positives

some antivirus *heuristics scanners* falsely accuse the exe of containing a virus, that is a false positive by a heruistics scanner, let me explain:

the game was written for Windows 98 / NT, 
long before multiple processor cores were common for consumer PCs, 
and the game use multithreading, and when running on a  PC with *multiple CPU cores* (which is all modern gaming PCs) it will eventually run into a race condition bug that will crash the game within an hour or so. 
however, this race condition bug does not occur when running on just a single CPU core (which the game was designed for), and thus i patched the game exe to call 
"SetProcessAffinityMask", with just 1 bit set, which tells windows to only run the game on 1 cpu core. the problem is the way i patched in this code, i found small unused regions of the game.exe, wrote jmp instructions to them at the start of the program entry point, and did my SetProcessAffinityMask() call there, then jumped back to the start of the game. to an antivirus heuristics scanner, this looks very similar to how viruses "infect" .exe files to spread themselves (usually viruses insert themselves to .exe files by doing a jmp call right at the start of the program), so a herustics scanner may assume that the .exe is infected.

both the original game.exe, and my patched game.exe is in the repository, compare the 2, and you'll see:

1: only the modified version looks like a virus to a virus to a scanner
2: the difference between the original .exe and the modified .exe is just a few bytes (less than 20 bytes, i imagine), 
which is not nearly enough bytes to embed actual malware in the .exe

if you're an assembly programmer, you can also check the patches i made with a disassembler (like ollydbg), 
the original game's entrypoint starts with: 

CPU Disasm
Address   Hex dump          Command                                  Comments
004C6E5E  /$  55            push ebp
004C6E5F  |.  8BEC          mov ebp,esp
004C6E61  |.  6A FF         push -1
004C6E63  |.  68 B0B84F00   push offset MFatigue.004FB8B0
004C6E68  |.  68 2CC94C00   push MFatigue.004CC92C
004C6E6D  |.  64:A1 0000000 mov eax,dword ptr fs:[0]
               


and my patched version starts with: 

CPU Disasm
Address   Hex dump          Command                                  Comments
004C6E5E      55            push ebp
004C6E5F      8BEC          mov ebp,esp
004C6E61      6A FF         push -1
004C6E63    ^ E9 1CC9FFFF   jmp MFatigue.004C3784
004C3784      60            pushad
004C3785      9C            pushfd
004C3786      6A 01         push 1 ; second argument to SetProcessAffinityMask, a single bit meaning only the first cpu core.
004C3788      E8 4CE0A174   call GetCurrentProcess
004C378D      EB 15         jmp short MFatigue.004C37A4
004C37A4      50            push eax ; first argument to  SetProcessAffinityMask 
004C37A5      E8 12FFA974   call SetProcessAffinityMask
004C37AA      9D            popfd
004C37AB      61            popad
004C37AC      EB 16         jmp short MFatigue.004C37C4
004C37C4      68 B0B84F00   push offset MFatigue.004FB8B0
004C37C9      E9 9A360000   jmp MFatigue.004C6E68
004C6E68      68 2CC94C00   push MFatigue.004CC92C                  
004C6E6D  |.  64:A1 0000000 mov eax,dword ptr fs:[0]


after that, everything in the exe is the same. why all the jumps, you ask? because the small unused regions i found was too small so i had to hop between them (luckily `jmp short` instructions are just 2 bytes long), i think they were put in place by the compiler for alignment issues (while x86 cpus doesn't require aligned data, they run faster if data is properly aligned)
