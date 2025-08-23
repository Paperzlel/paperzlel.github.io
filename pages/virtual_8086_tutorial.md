# Virtual 8086 Mode Tutorial
This tutorial will attempt to give a good working example of how to set up a Virtual 8086 Mode for a standalone OS. This tutorial assumes some competence with OS development, but will guide you through most of the process regardless.

## Introduction
Most modern OSes run in protected mode. This has obvious upsides, most notably the vast increase in RAM access and access to userspace via rings. However, most BIOS interrupts that the user was able to have during the bootloader ae no longer available, and require an alternative method to access such interrupts. This is where Virtual 8086 Mode comes in.

Virtual 8086 Mode (or V86, as this article will refer to it) is a hugely helpful tool for the operating system developer, especially ones that may require these BIOS interrupts. Its setup is quite convoluted however, and this tutorial will aim to guide the user through said process.

## Requirements
In order to use V86, there are several items one will need to have set up prior. They are as follows:
1. A Global Descriptor Table, with two 32-bit code and segment registers, and a Task State Segement register. You can also add 16-bit code and segment registers, but they are not required.
2. A filled-out Interrupt Descriptor Table with the ability to add a handler to an Interrupt Service Routine. A good way of doing this is to have one global ISR that calls another ISR for that specific interrupt.
3. A Task State Segment, with `ss0`, `es`, `cs`, `ds`, `fs`, `gs`, `esp0` and `iobp` set. All of the segment registers bar `cs` should be set to your 32-bit data segment, and `cs` should be your 32-bit code segment. `esp0` should be at the top of your stack when it is initialized, but this may not matter. Assuming you also have `SPP` in your TSS, the size of the TSS for `iobp` is sizeof(TSS) - 4.

Some form of debugging tools, such as `bochs` or `QEMU` is also highly recommended.

## How to Load your Program
The next part of this tutorial assumes that you are loading the desired V86 routine from within your kernel. If you have it saved to a `.bin` file or some other item within your filesystem, you are sadly on your own for the moment, but the principles should be the same.

Generally, 16-bit real mode programs cannot operate beyond the 1 MiB barrier - the limitations of the segmented memory model forbids this. So in order to run our programs, we should relocate the code to run at a lower address that is usable instead. This can be done in several ways, so the method I use may not be the best.

*For a quick note, and V86 process one may want to run is reffered to within this tutorial as a "task". This is simply the terminology I have come to refer to them by.*

### Exposing the data
The method I use to expose the task data is as follows:
1. Have a `.asm` file with a program I want to run in it:
```asm
global __task_run
__task_run:
    mov al, 3
    mov bl, 2
    mul bl
    int 0xfe        ; Returning to this later

global __task_end
__task_end:
```
2. Expose those globals as numbers within a `.c` file:
```c
extern uint8_t __task_run;
extern uint8_t __task_end;
```

Any loading code should go within the same `.c` file as above, so we don't have to expose them to any other tasks we may want to run. If you wish to have any variables within your task, make sure to define them below the end of the function (the `int 0xfe` line).

### The V86 Monitor
V86 programs are fickle beings and tend to incur more than a fair share of General Protection Faults (GPFs). This is due to their privileges being much less than our code, and often code you write for a V86 program, especially interrupts, will cause a GPF. For most of us, these flags are a false positive, and we need a way to emulate said instructions so that we don't panic our kernel every line. This is where the V86 Monitor comes in.

The name itself is far more eloquent than what it actually is - all a V86 Monitor does is hand-hold some of the Assembly instructions from protected mode and then return from the interrupt back to the regular program.

For us, the exception handler should look something like this:
```c
void v86_monitor_exception_handler(Registers *p_regs) {
    // Code goes here
}
```
The `Registers *p_regs` is a pointer to all of the register values at the time of the interrupt occuring.

From here, we need to register this function with our ISR handler, which can then dispatch any GPFs running during the program to this handler. It is good practice to remove these handlers once the V86 task has been ran, as the rest of the kernel can cause a GPF and be misdirected.

### Loading to RAM
Next up is the actual loading function. This will direct the program from its initial place in memory, into its new position elsewhere. The function looks something like this:
```c
void v86_load_task(void *p_task_start, void *p_task_end) {
    uint32_t size = p_task_end - p_task_start;
    memcpy((void *)(V86_CODE_SEGMENT << 4), p_task_start, size);

    isr_register_handler(13, v86_monitor_exception_handler);
    isr_register_handler(0xfd, v86_monitor_entry_handler);  // Come back to this later

    v86_enter_v86_handler(V86_STACK_SEGMENT, 0x0000, V86_CODE_SEGMENT, 0);
}
```
Since `p_task_end` and `p_task_start` are linear, subtracting them should give the number of bytes we need to copy. From there, we can register our handlers and eventually boot into next part of the code.

One thing to note is that the values for `V86_STACK_SEGMENT` and `V86_CODE_SEGMENT` should be values that reference legitimate segment:offset addresses. For example, I set `V86_CODE_SEGMENT` to `0x1000` which when moved to a segment:offset addrss will be `0x10000`, a location in memory we can read/write from. Likewise, `V86_STACK_SEGMENT` is set to `0x8000` which becomes `0x80000`, also a usable location in memory.

### One Step Back, Two Steps Forward
Usually, this would be the position that one would expect to boot into V86 mode and be happy. However, there is one (very large) catch.

When we want to *leave* V86 mode, we can't just call to `ret` and expect the program to return back to 32-bit protected mode. This is because we enter V86 mode by manually setting the registers to different addresses, and since there's no way for the V86 task to retrieve the data within its own state, we have to save the registers prior to entering V86 mode.

So when we want to *enter* V86 mode, we first need to get the data of all registers prior to entering it, then actually enter, and when we want to exit V86 mode, we reload the registers into their previous state and can leave. 

One (relatively) small issue is that we can't re-map `ret` to do this. Instead, we use a custom interrupt code, in our case `0xfe` to handle this instead.

In order to get the register data however, we need to call an interrupt in the first place to push all that information to the stack, hence the binding of `v86_monitor_entry_handler` to `0xfd`.

The function `v86_enter_v86_handler` looks like the following:
```asm
global v86_enter_v86_handler
v86_enter_v86_handler:
    push ebp
    mov ebp, esp

    push ebx

    mov eax, [ebp + 8]      ; SS
    mov ebx, [ebp + 12]     ; ESP
    mov ecx, [ebp + 16]     ; CS
    mov edx, [ebp + 20]     ; EIP
    int 0xfd                ; Call to our handled interrupt

    pop ebx

    mov esp, ebp
    pop ebp
    ret                     ; Return like usual
```

And the redirected interrupt to `v86_monitor_entry_handler` will end up looking something like this:
```c
void v86_monitor_entry_handler(Register *p_regs) {
    // Copy pre-V86 registers to predetermined location
    // a_reg_state is a local variable that we can copy to
    memcpy((void *)&a_reg_state, p_regs, sizeof(Registers));
    // Important: The TSS needs to know what SP to return to (SS is already set when we initialize the TSS)
    i686_tss_set_esp(v86_cpu_get_esp());
    // Important: See above, similar reasons
    i686_tss_set_eip(v86_cpu_get_eip());
    // Actually enter V86 mode
    v86_enter_v86(p_regs->eax, p_regs->ebx, p_regs->ecx, p_regs->edx);
}
```
One thing to note here is that we **must** let the TSS know what `ESP` and `EIP` to return to. Without this, the hardware switch to V86 mode will no know where to return to on an interrupt, and will cause a lot of errors.

### Into Virtual 8086
At long last, we are ready to enter V86 mode. There are several ways of doing this, but the core idea is to "trick" the CPU into thinking it returned from an interrupt instead of regular code, and it begins executing from the first instruction that `CS:IP` points to.

```asm
global v86_enter_v86
v86_enter_v86:
    mov ebp, esp                ; Don't create a new stack frame; we won't return from it
    push 0
    push 0
    push 0
    push 0                      ; Push GS, FS, ES, DS for iret to use

    push dword [ebp + 4]        ; SS
    push dword [ebp + 8]        ; ESP
    pushfd
    or dword [esp], (1 << 17)   ; Set VM flag
    push dword [ebp + 12]       ; CS
    push dword [ebp + 16]       ; EIP (now points to the new instruction set)
    iret
```
When the CPU runs an `iret` instruction, it pops the above items off the stack. From our values we input above, the CPU thinks we input `0x1000:0x0000` as our `CS:IP` and `0x8000:0x0000` as our `SS:SP`. With the `VM` bit set in `EFLAGS`, the CPU now runs in V86 mode. Phew.

## Handling Those Interrupts
Now we're halfway to our precious BIOS interrupts in protected mode. The second half is a little more pesky: we need to emulate some of the instructions that can trigger a GPF in V86 mode. For the sake of simplicity I will restrict this to just `int n`, but the overall instructions can be seen [here](https://wiki.osdev.org/Virtual-8086_Monitor#Role_of_General_Protection_Faults) if you want to expand your V86 monitor's capabilities.

### General Functions to Emulate
There's several things we'll want to do when emulating a GPF-triggering instruction, the first of that being to read the instruction itself. We can do that as `CS:IP` was pushed to the stack at the fault-triggering position, allowing us to read the machine code and make some decisions on-the-fly. The code for `int n` is `0xCD`, so if we read the instruction byte and it matches, we should go on to emulate said instruction. To read the stack is as follows:
```c
uint8_t getb(uint16_t p_seg, uint16_t p_ofs) {
    return *((uint8_t *)(p_seg << 4) + p_ofs);
}
```
This reads the byte at the given segment:offset. Incrementing the offset will give us the next byte, and so forth. Getting a word/`uint16_t` is the same process but with a `uint16_t` instead, of course.

Another useful instruction is to push to the stack. Pushing and popping may be allowed in regular V86, but for our emulation we would prefer to do this in C as well so we can push our own values, like so:
```c
void pushw(Registers *p_regs, uint16_t p_word) {
    p_regs->esp -= 2;       // Decrement ESP
    *(uint16_t *)((p_regs->ss << 4) + p_regs->esp) = p_word; // Add value to stack position
}
```
With these functions define, it's time to look at how we emulate that `int n` instruction.

### Emulating INT n
Once we've decided that it is, in fact, an `int n` instruction, we can move on to the second step: checking what the `n` is.

In machine code, the `n` comes directly after the `int`, so if we increment the stack pointer and read the next byte, it should be there. Once read, we need to do a quick check to see if it's the same value as our dear friend `0xfe`. 

Remember, `0xfe` is our escape interrupt, and when it is called we want to leave V86 mode immediately. We do this as described, by copying the data to the passed-in register pointer and returning. Quite simple.

If it's not `0xfe` however, we want to do something quite different. In our V86 mode, we're expecting all interrupts to want to interface directly with the IVT described at the start of memory. Because of that, we need to set up our stack and registers in such a way so that (1) the CPU begins to execute the BIOS interrupt immediately after `ret` is called at the end of the handler, and (2) so that the CPU will return to our V86 program when it calls to `iret`.

To do that, we want to do the following:
1. push `EFLAGS` to the stack
2. Modify `EFLAGS` so that the IF, TF, and AC flags are disabled. The last one isn't in real mode so it doesn't matter for the actual interrupt, but stops the CPU from doing memory alignment checks that could mess up our code.
3. Push `CS`, then `IP` to the stack, incrementing IP so it points to the next instruction prior to pushing.
4. Change `CS` and `IP` so they point to the correct offset for the interrupt in the IVT. This is done by putting the forumla `4 * n + 2` for CS and `4 * n` for IP as the offset in `getw`, and setting the segment to 0.

From there, the CPU should jump to the BIOS interrupt and execute like it's in real mode. Hooray!

The part above in code could look something like this:
```c
uint8_t operation = getb(p_regs->cs, p_regs->eip);
switch (operation) {
    case 0xCD: {    // INT nn
        uint8_t int_id = getb(p_regs->cs, ++p_regs->eip); // Increment SP to get nn
        // Copy old state over and return
        if (int_id == 0xfe) {
            memcpy(p_regs, (void *)&a_reg_state, sizeof(Registers));
            return;
        }

        pushw(p_regs, p_regs->eflags);      // Push EFLAGS
        p_regs->eflags &= ~((1 << 8) | (1 << 9) | (1 << 18));   // Disable IF, TF, AC
        pushw(p_regs, p_regs->cs);          // Push CS
        pushw(p_regs, ++p_regs->eip);       // Push IP
        p_regs->cs = getw(0x0000, 4 * int_id + 2);  // Set to IVT offset
        p_regs->eip = getw(0x0000, 4 * int_id);     // Set to IVT offset
    } break;

    default:
        break;  // Unknown, do nothing
}
```

## Leaving V86 Mode
When the program you have written is done, it should call to `int 0xfe` as its last instruction, **not** `ret`. As stated above, calling to `ret` will start the program over again and cause an infinite loop, which is unlikely to be wanted for a V86 program.

## Conclusion and Final Notes
Well, that's it! Hopefully by now you have some kind of V86 Monitor and way to enter/exit your V86 programs. If you want an example for an OS that implements this, I give you [Aurora](https://github.com/Paperzlel/Aurora), my own OS that I suffered plenty with to get V86 working, and wrote this tutorial because of my pains with it. Happy OS creation!

### Final Notes
Finally, some things to bear in mind when creating V86 tasks themselves, rather than this entire system.
- Only use V86 tasks when there is no way to access a BIOS function or use certain behaviour exclusive to Real Mode. For instance, using V86 for floppy disk systems is impractical, as alternative methods that do not require BIOS interrupts exist for reading/writing to them. Personally, I use V86 for enabling VESA VBE graphics modes within my kernel, as despite having a bootloader I still want to have access to text modes before I enable the graphics driver.
- V86 is not a suitable method for running plain old Assembly. You can do all of that anyway, use the ASM/C interop that already exists.

## Resources
1. [OSDev Wiki on Virtual 8086 Mode](https://wiki.osdev.org/Virtual_8086_Mode)
2. [OSDev Wiki on Virtual 8086 Monitor](https://wiki.osdev.org/Virtual-8086_Monitor)
3. [Aurora (Operating System) for reference](https://github.com/Paperzlel/Aurora) (see src/kernel/i686/v86, and check commit #20 if it's moved)
