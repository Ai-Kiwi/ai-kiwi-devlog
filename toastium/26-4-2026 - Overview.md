# Toastium
For a while now, my dream project has been an operating system. I am not sure why, but there is just something so cool about writing everything yourself and knowing how everything works. I've wanted to learn GPU rendering for a while, as it gave me a lot of power and understanding, and the ability to do a lot, and this is almost the next logical step in that. Creating my own OS would give me a deep understanding of everything and the ability to do much more on computers and understand them better. 

## The goal
Originally, the goal for this project was to get it running for a custom handheld portable game console that my friend and I are making. Although this is a planned target, I decided it would be best to make it not the main target, so a more regular desktop would allow for more future possibilities, as I wouldn't shoot myself in the foot by missing something. It was also not guaranteed that I would be able to do this project, so I felt, for the time being, that I should plan to run Linux on the console and use this OS if it gets far enough along.    

With that said, what is the goal???   
I honestly don't really have one for this project; it's just about doing as much as I wanna do and learning everything I can. It would be quite cool to get a desktop running where you get GUIs for apps, so I suppose that is an end goal, but again, it will just change with time, like as I reach that next milestone, it might be networking, then a better file system, etc. 

## technical specs
For picking an architecture, I decided on RISC-V. The reason for this is the more simplestic nature of it, it being a lot simpler meant I was more likely to actually get this project happening, and if I later wanted to port to another arch, such as x86 or arm, I could always change my code to support it, this mean that a goal with this os was coding as much as i can not OS spesfic so i can change to add support at a later date.    
Because of the above reason, the os will be designed with portability in mind so its architecture-specific stuff compresses down to generic function calls so it can be swapped out fairly easily. 
For if the OS is POSTIX, I decided to follow the majority of POSTIX while making some changes.   
 - Everything is a file: I decided to scrap this. Personally, I am not a fan, and I feel it makes stuff messy.
 - Introduced a new IPC type. The types are 
   - Pipe: one-way send, one-way receive.
   - Socket: predefined code/id which is global and you can access. It's one that receives many processes send.
   - Caster: predefined code/id which is global and you can access. It's one send, many receive. Each client will have its own buffer of items; they decide the size, and it starts deleting once it starts filling up. If no one is subscribed, the data is dropped.

If data is accessed on request, it can be done via a Caster with length one fairly cheaply; if it needs to be even faster, it can be done with memory mapped to RAM.    
For permissions, I plan to add a tag to each process indicating whether it can access resources such as the framebuffer or the UART. This tag idea will apply to types as well for Caster and Socket. The permissions will be decided by the parent process which is launching it, a parent can't launch with more perms then it has. The idea of file access will be scrapped, and will possibly revisit it later.   


