# Switching Modes
This section just talks about x86_64 for now. 

## Getting to User Mode
This is pretty straight forward! We're going to make use of the `iret` instruction to do this.
Remember in 64-bit mode, all stack operations are 64-bits wide.

`iret` pops 5 arguments off of the stack, and then performs several operations at once (atomically):

- It pops `rip` and `cs` from the stack, this is like a far jump/return. `cs` sets the mode for instruction fetches, and `rip` is the first instruction to run after `iret`.
- It pops `rflags` into the flags register.
- It pops `rsp` and `ss` from the stack. This changes the mode for data accesses, and the stack used after `iret`.

This is a very powerful instruction because it allows us to change the mode of both code and data accesses, as well as jump to new code all at once. It has the added benefit of switching the stack at the same time, which is fantastic.

Changing the flags atomically like this means we can from having interrupts disabled in supervisor mode, to enabling interrupts are soon as we're in user code. Without any risk of getting an interrupt in the middle of changing to user mode.

### What to Push Onto The Stack
Now let's talk about what these values should be. `rflags` is an easy one: set it to 0x202. Bit 2 is a legacy feature and must always be set, the other bit (`0x200`) is the interrupt enable flag. This means all other flags are cleared, and is what C/C++ and other languages expect flags to look like.

For `ss` and `cs` it depends on the layout of your GDT. We'll assume that you have 5 entries in your GDT:

- 0x00, Null
- 0x08, Supervisor Code (ring 0)
- 0x10, Supervisor Data (ring 0)
- 0x18, User Code (ring 3)
- 0x20, User Data (ring 3)

Now `ss` and `cs` are *selectors*, which you'll remember is not just the byte offset into the gdt, but the lower three bits contain a field called RPL. Requested Priviledge Level is a legacy feature, but it's still enforced by the cpu, so we have to use it. RPL is a sort of 'override' for the target ring, it's useful in some edge cases, but otherwise is best set to the ring you want to jump to.

So if we're going to ring 0 (supervisor), RPL can be left at 0. If going to ring 3 (user) we'd set it to 3.

This means our selectors for `ss` and `cs` end up looking like this:

```c
kernel_ss = 0x08 | 0;
kernel_cs = 0x10 | 0;
user_cs   = 0x18 | 3;
user_ss   = 0x20 | 3;
```

As you might have noticed, the kernel/supervisor selectors don't need to have their RPL set explicitly, since it'll be zero by default. This is why you may not have dealt with this field before. 

If RPL is not set correctly, you'll get a #GP.

As for what to set `rip` and `rsp` to, the target code you want to run, and the stack you want to run it on. It's a good idea to run user and supervisor code on separate stacks. This way the supverisor stack can have the U/S bit cleared, and prevent user mode accessing supervisor data that may be stored on the stack.

### Extra Considerations
Since we have paging enabled, that means page-level protections are in effect. If we try to run code from a page that has the NX-bit set (bit 63), we'll page fault. The same is true for trying to run code or access a stack from a page with the U/S bit cleared. On x86 this bit must be set at every level in the paging structure.

*Authors Note: For my VMM, I always set write-enabled + present flags on every page entry that is present, and also the user flag if it's a lower-half address. The exception is the last level of the paging structure (pml1, or pml2 for 2mb pages) where I apply the flags I actually need. For example, for a read-only user data page I would set the R/W + U/S + NX + Present bits in the final entry. This keeps the rest of implementation simple. - DT.*

### Actually Getting to User Mode
First we push the 5 values on to the stack, in this order:
- `ss`, ring 3 data selector.
- `rsp`, the stack we'll use after `iret`.
- `rflags`, what the flags register will look like.
- `cs`, ring 3 code selector.
- `rip`, the next instruction to run after `iret`. 

Then we execute `iret`, and we're off! Welcome to user mode!

This is not how it should be done in practice, but an example function of what this might look like, using the example GDT from above. Here we're pre-calculated the user cs as `0x1B` and user ss as `0x23` for simplicity.

```c
__attribute__((naked, noreturn))
void SwitchToUserMode(uint64_t stack_addr, uint64_t code_addr)
{
    asm volatile(" \
        push $0x23 \n\
        push %0 \n\
        push $0x202 \n\
        push $0x1B \n\
        push %1 \n\
        iretfq \n\
        " :: "r"(stack_addr), "r"(code_addr));
}
```

And voila! We're running user code with a user stack. 
In practice this should be done as part of a task-switch, usually as part of the assembly stub used for returning from an interrupt (hence using `iret`).

## Getting Back to Supervisor Mode
This is trickier! Since you dont want user programs to just execute kernel code, there are only certain ways for supervisor code to run. The first is to already be in supervisor mode, like when the bootloader gives control of the machine to the kernel. The second is to use a system call, which is a user mode program asking the kernel to do something for it. This is often done via interrupt, but there are specialized instructions for it too. We have a dedicated section to system calls.

The third way is inside of an interrupt handler. While you *can* run interrupts in user mode (an advanced topic for sure), most interrupts will result in supervisor code running, in the form of the interrupt handler. Any interrupt will work, for example a page fault or ps/2 keyboard irq, but the most common one is a timer. Since you can program a timer to tick at a fixed interval, you can ensure that supervisor code gets to run at a fixed interval. That code may return immediately, but it gives the kernel a chance to look at the program and machine states and see if anything needs to be done. Commonly the handler code for the timer also runs the scheduler tick, and can trigger a task switch.

Handling interrupts while the cpu is in user mode is a surpringly big topic, and so it has a section of it's own. If you're here just for reference, the short of it is:

- Populate and and load a TSS into the task register (TR).
- Set `rsp0` stack to something valid.
- Enter user mode and wait for a hardware interupt.

### Software Interrupts
TODO: this would make more sense in the interrupts section?
On x86(_64) IDT entries have a 2-bit DPL field. The DPL (Descriptor Privilege Level) represents the highest ring that is allowed to call that interrupt from software. This is usually left to zero as default, meaning that ring 0 can use the `int` instruction to trigger an interrupt from software, but all rings higher than 0 will cause a a general protection fault. This means that user mode software (ring 3) will always trigger a #GP instead of being able to call an interrupt handler.

While this is a good default behaviour, as it stops a user program from being able to call the page fault handler for example, it presents a problem: without the use of dedicated instructions (which may not exist), how do we issue a system call?

Fortunately the solution is less words than the question: Set the DPL field to 3.

Now any attempts to call an IDT entry with a `DPL < 3` will still cause a general protection fault. If the entry has `DPL == 3`, the interrupt handler will be called as expected. Note that the handler runs with the code selector of the IDT entry, which is kernel code, so care should be taken when accessing data from the user program. This is how most legacy system calls work, linux uses the infamous `int 0x80` as it's system call vector.