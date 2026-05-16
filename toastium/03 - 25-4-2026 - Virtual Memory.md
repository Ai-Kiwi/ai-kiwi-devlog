# Virtual Memory System
I'd like to start by correcting myself. While reading the specifics of how RISC-V implements virtual memory tables, I learned that it has a flag that allows a section to be accessed only by the supervisor. I was curious about the reason and discovered that my model was wrong.      
Previously, my understanding was that when syscall ran, it would raise to the supervisor level and then would disable the table mapping. This is incorrect; I found that, typically, people design with it so that it actually keeps the table mapping and access kernel code using it. This is done by mapping kernel data to a location, such as the 2nd half of the process's virtual memory. The reasoning is that switching tables is expensive and requires a TLB flush, which makes it even more expensive. This is also how meltdown really works, it's a process of getting cpu to prefetch from this kernel area using predictive pipeline and then looking at how hot cpu data is by reply latency (if its in cache or not) as the data of kernel is in process's virtual memory, the reaosn why on older cpu its expesnive to fix its because it requires switching table to stop this happening.

> [!NOTE]
The regions the kernel occupies are marked as kernel-only, meaning they cause a page fault if accessed from a user process. 

Arguably, you could do it the way I originally planned, but, as mentioned before, it would be much slower due to table switching and TLB flushing. 

However, I did notice a problem with this. The kernel currently loads everything in at the process start location. If you then tried to use data here, like the pointers, after a userspace process starts, they would point to completely wrong locations, causing incorrect data to be read and written. The obvious fix I first thought of for this was, of course, moving everything over, pointer by pointer. However, there is actually an easier fix to this. What if instead of moving everything, we started with everything moved? The "cheat" here is that we, in the bootloader before C loads, set up a table map for the kernel with the kernel data in the 2nd half, load it into that table map, and then start the kernel using that table map. We can then change the linker to be offset. This does raise an issue, however. Locations such as UART would be in the wrong place. The solution I thought of was to instead just add the offset for each of these; this is messy, but most likely the best fix. The same idea would apply to DTB addresses as well.   
Surprisingly, this same approach is used in Linux itself, confirming to me that there is some amount of sense to this approach: https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S

Another thing to note is that, when I was researching whether they start locations at 0 or not, as I thought they might not because of reasons like the ELF format, possibly or something. They, in fact, leave a gap at the start, whose purpose is to cause null pointers to crash the system. This is a good idea, and the same thing will be done here.  

In terms of actual implementation, I thought it was best to code most of this as arch-specific. The reason for this is that, although RISC-V, SV39 works perfectly for this use case, as each table is exactly the size of a page. Other archs may use different formats that work differently, such as table pointers, depth count, permissions, and size of allocations themselves. This means that, because of the significant differences per arch, I decided to just follow the previously used approach of a generic function type across different arches. I should also mention that the pointer offsets for things like UART will also be passed. 

Another point worth noting is that tables also serve another purpose: they enable efficient data sharing between processes. You can have multiple processes with the same RAM location pointer, meaning different processes can share data. This is very useful for use cases such as a framebuffer. Where a syscall can request the framebuffer and where it would like it, and then that location can be mapped. This is also another thing to note with things such as framebuffer that is passed to userspace, is that you don't want kernel data to go into any "padding area" that each of these has if they don't take up the full 4096 bytes for a page, as that will also be passed to the user. The solution I thought of for this is to align your pages to 4096 bytes. This raises a problem with the original approach I used when coding the pager. Although I made sure the pages were aligned, I didn't make the bitmaps aligned, which means that in rare cases where space is free after, say, a framebuffer, it would try to write there, potentially causing issues in user space. The fix is to ensure that the locations for bitmap data are also aligned to 4096.  

In terms of coding this, I figured I would need a location to store the map for the top half data to kernel data, and this will need to happen in assembly. I thought the cleanest way was likely to just put it right after the program in RAM, then send the leftover start location to the kernel when it starts. 

Similar to before, there is again a cheat, which I found out about while researching how this is actually implemented: the whole thing can more or less be done on one page, since the top 256 locations can all be huge pages instead of normal, smaller ones. However, from what I have heard, this does actually have a downside, unlike previous cheats. The downside is that if the kernel tries to run code in data locations or to save code in a location for running, both will succeed. The normal approach to stop this is to use certain pages as read-only and others as non-executable; however, this can't be done when all are large, gigabyte-sized pages. For now, I have decided to use gigabyte pages with the plan to revisit this later, as the effort to change it shouldn't be huge.

# Coding it
Coding this was definitely an adventure, probably the longest part of this operating system yet. It was definitely good to see it working. I didn't menstion before but I choose sv39 as it worked well with currently page size of 4096B which is a common size between archs and it also supported up to 512GB for ram which is enough that it won't be a problem for a good bit and by the time it is this project is probs gonna be far enough along i can code a handler for another one.   

For this project to work, I needed to get a table which it uses the location of this. I decided to set it as a location in the linker file, since it's a safe place that can be used and easily referenced. Although this isn't a great fix as its arch spesfic 4096B is a common size, though I do stand by the opinion  

> [!NOTE]
> A better approach could actually be as simple as making it depend on the arch selected, which it reads from the C header during linking.
> I decided against figuring out all this at runtime, such as choosing a location at the end of RAM or something similar, since it added a lot of complexity for little real-world benefit.    

Around here was actually when I found out, using GDB, that the locations were all the real locations you would expect; this surprised me. I later found out from asking AI about the possible reason for these assembly instructions, like la, and when you reference them, the linker locations, even in C, are actually pc relative, not real RAM locations. This explains why my code ran even with a previously incorrect offset in the linker; for this reason, I removed all the fixed offsets in the linker and moved everything to use direct linker location instead (meaning without a start offset). This approach is, as expected, how Linux does it and makes the OS a huge amount more portable, making the system I used for differences between boards now bassicly useless as the other feature, which was for UART adjustments, as I'm pretty sure is instead a big issue with how I implemented it and isn't a different size between archs but will have to look more into. Because of this misunderstanding, I also got most of the way through coding a system to pass the start location to the kernel on boot using assembly, pc at boot, remove the offset, then pass it as a parameter to C, but scrapped it as not needed when I found out it does it for you.   



> [!NOTE]
> Personally, I'm not a fan of AI for programming; it takes away the learning and fun of it. I tend to use it only to explain concepts; for actual implementation, I try to figure it out through trial and error, then get it to give me thoughts on it. This approach is, however, fundamentally flawed because it tends to say it's always good, better than nothing, and sometimes gives good suggestions.
> I also for this project asked it for advice on some matters like the cleanest way to organise stuff like folders and files, id think about how to do it, then get thoughts, same idea for things like IPC for compistor and things like that, although I was more asking it how OS like Linux and whatnot does it.   
> I am tending to use AI less and have made a commitment to use it as little as possible a bit further ahead.   
> I'll make an actual write-up one day on my stance for AI for my personal projects.


An annoying thing that had to be done here was that I had to write to the table used as a location, set the lower and upper halves as both kernel, enable it, then jump to the 2nd half, rewrite it again to remove that part, then apply a TLB flush. A bit messy but a required step. This can be seen in assembly.

```asm
blank_first_half_vma_table:
    bgeu t1, t6, real_start
    add t4, zero, zero //final value

    add t3, t1, t0
    sd t4, 0(t3) //output

    addi t1, t1, 8
    j blank_first_half_vma_table

```

Here is the code for actually creating the VMA table (Some parts removed)

```asm
setup_vma:
    //setup virtual memory maps
    la t0, _kernel_vma_offset_table //location for vma table
    li t6, 4096
    div t0, t0, t2
    mul t0, t0, t2 //rounds it down
    li t6, 2048 // 2048 stored
    addi t1, zero, 0 //loop number
    j setup_vmn_table

setup_vmn_table:
    //t0 : vma location
    //t1 : loop number (bytes)
    //t2 : loop number
    //t3 : output location
    //t4 : page entry
    //t5 : PPN DATA
    //t6 : 2048

    bgeu t1, t6, finish_vma_setup
    li t2, 8
    div t2, t1, t2

    //setup kernel address range
    li t4, 0xEF //allow read/write/execute, mark as global for optimization, is valid and also mark as already dirty and accessed for performance .

    slli t5, t2, 28 //move the current iter into PPN[2] (located at bit 28)

    add t4, t4, t5 //add together, final page entry

    //get location
    add t3, t1, t0
    add t3, t3, t6

    sd t4, 0(t3) //write out

    //setup userspace, for now kernel until jump then blank
    li t4, 0xCF //allow read/write/execute, is valid and also mark as already dirty and accessed for performance .

    add t4, t4, t5 ///add together, final page entry

    //get location
    add t3, t1, t0

    sd t4, 0(t3) //write out

    addi t1, t1, 8
    j setup_vmn_table
```

And code to apply the table itself
```asm
finish_vma_setup:
    //t0 is base vma table location 
    //ASID needs to be highest number as this will be reserved for kernel process
    la t1, _kernel_vma_offset_table
    li t2, 4096
    div t1, t1, t2

    li t3, 0xFFFF // ASID id, max possible
    slli t3, t3, 44

    add t1, t1, t3 // Add ASID value

    li t3, 8 // type value (sv39)
    slli t3, t3, 60
    add t1, t1, t3 // Add type value

    
    csrw satp, t1 //apply table location and info
    
    sfence.vma zero, zero //flush TLB so it applies

    la t3, jump_blank_first_half_vma_table
    li t2, KERNEL_VMA_START
    add t3, t3, t2
    jalr zero, 0(t3) //make the jump to real location

jump_blank_first_half_vma_table:
    la t0, _kernel_vma_offset_table
    j blank_first_half_vma_table
```

Throughout this, as expected, I ran into a lot of different error messages. For some of them, I could actually chuck in UART debug messages; for some that didn't work, I had to set up GDB to give messages; and for others, I had nothing, so I just had to figure it out. Here are some examples I ran into. 
 - In one of my add instructions, I put t3 instead of t0, which caused a trap. Traps were completely broken, ending up finding out functions only ran, which I confirmed with a for loop, then later figured out this was because the stack wasn't set, as it was set after VMA, not before.
 - Also, the issue above, before I made it freeze, the system would spam loop panic, but would panic right after running any function, so nothing useful was said, just the first line in the trap was run. 
 - Location to jump to was the actual VMA, not with it added, again typo.
 - A nice trick I learned while making this is that I could call my UART println before the OS had actually started, which made printing data, like what is going into the table and where, much easier. As i didn't have to get out GBA to see reg and ram.
 - I learned that PPN warl (pointer to table) was not a ram pointer but a ram pointer divided by 4096B, as it was located in pages. This meant I had to code the linker to be aligned to 4096B for the location of the vma table, and also had to set up code to divide the location on boot.
 - Trap locations are virtual addresses, not real addresses.
 - On reading data for the table in GDB, it appeared to be going up in increments of 4 instead of 1. This later turned out to be because it was bit 10, not the first bit of the byte, so it was making it look larger as the GBA printed each byte. Also, an interesting thing I noticed is it was little endian, which followed as expected in the docs, but I was interested to see, I confirmed this wasn't he issue, and it wasn't. (By confirm, I mean Google)
 - I found that the location for vma was incorrect; it wasn't the same as the actual value. This is because it was rounded down, so I did this on load instead and passed that value to the code to set up the VMA table. I also had to add padding below the VMA table in the linker as well, for this reason, so it doesn't go into the stack data below it. I found this by running GBA right up to the stop instruction, then looking at the output, decoding in binary and reading what it looked like. This fix does waste bytes, but the cost isn't huge in the scale of things. Possibly might come back in future, but unlikely to. 
Most of the fixes I ended up making were done by just reading over and over and over and over the RISC-V documentation till something popped out. Notably, one annoying one was if the root PPN for entries was the first one or the last. I ended up thinking first, so had bit 28 picked, but left code in for bit 10 as the other was. When I eventually got everything else working, it still wasn't doing what I expected, and I found that it ended up being this issue, and it started working after this, and I immediately booted my OS, and the kernel panicked as I hadn't set up VMA offsets in it lol. I didn't even expect it to get that far, to be honest. I thought something else would not work, but all the jump code and everything worked fine on the first try (I was debugging it before it was even setting the VMA).   


Start dump of what the table looks like.   
```
0x8021b000:    0xcf    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b008:    0xcf    0x04    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b010:    0xcf    0x08    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b018:    0xcf    0x0c    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b020:    0xcf    0x10    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b028:    0xcf    0x14    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b030:    0xcf    0x18    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b038:    0xcf    0x1c    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b040:    0xcf    0x20    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b048:    0xcf    0x24    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b050:    0xcf    0x28    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b058:    0xcf    0x2c    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b060:    0xcf    0x30    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b068:    0xcf    0x34    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b070:    0xcf    0x38    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b078:    0xcf    0x3c    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b080:    0xcf    0x40    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b088:    0xcf    0x44    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b090:    0xcf    0x48    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b098:    0xcf    0x4c    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0a0:    0xcf    0x50    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0a8:    0xcf    0x54    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0b0:    0xcf    0x58    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0b8:    0xcf    0x5c    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0c0:    0xcf    0x60    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0c8:    0xcf    0x64    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0d0:    0xcf    0x68    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0d8:    0xcf    0x6c    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0e0:    0xcf    0x70    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0e8:    0xcf    0x74    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0f0:    0xcf    0x78    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b0f8:    0xcf    0x7c    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b100:    0xcf    0x80    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b108:    0xcf    0x84    0x00    0x00    0x00    0x00    0x00    0x00
0x8021b110:    0xcf    0x88    0x00    0x00    0x00    0x00    0x00    0x00



```

Some random other notes are that I set accessed and dirty bits on create, as from my understanding of RISC-V docs, it's better for performance, as it doesn't have to make another call on those happen and as I don't have swap or other systems that need it currently, it's best to leave blank. I also set the upper half, but not the lower half, to global for performance. 



At this point, I thought it was best to stop. I don't know many details about user space and processes, how it's going to be planned, or how it's going to store who owns a page and who is using it. I felt that starting this already before everything else is ready would lead to bad decisions. 
