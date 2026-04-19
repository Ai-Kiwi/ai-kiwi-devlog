# Voxelith overview.
This has started being written after voxelith is a good chunk into development, so a fair amount is going to be glossed over or missed. For this, I will also only focus on the rendering side; throughout the whole project, the rendering engine code was recoded; however, the core program data outside of rendering was kept through graphics recode. 

# The goal
I played a game called Teardown once, and I really liked the idea of a world split into voxel-like terrain, so I thought a game like that would be really cool to make. However, I felt the teardown was kind of a waste of the opportunity. I mean no disrespect to teardown; it just wasn't trying to be the game I felt I wanted. So my goal was to create a game that was a merge between Teardown and Minecraft. I should be clear here, I am not really trying nessasorlly to make the game, as game devlopment isn't really my area of exspertese and i don't really wanna be making a game and tunning everthing more making the engine if you would say so.  

# Raylib
My first approach for creating this game was to render it in Raylib. The reason behind this was that Raylib was what I had learned before for making a game, and it used Rust, which I was familiar with. The previous project I learned this on was https://github.com/Ai-Kiwi/dungeon-crawler, if I recall correctly. Now, it probably doesn't come as a surprise that it didn't run well at all under Raylib, so I was going to need something a bit more low-level. To achieve this, I chose to use Macroquad, as it lets me avoid GPU-based graphics rendering while still achieving much better performance.  

# Macroquad
Under macroquad, I began programming the project; however, I noticed that the performance was not great, to say the least. In the 2nd image, you see I was getting under 2 fps, which was not the ideal gameplay experience I was hoping for, to say the least. I discovered the reason was the number of different "objects" it was rendering, since each chunk was a mesh. To address this issue, I decided to add chunk merging.  
<img alt="image" src="https://github.com/user-attachments/assets/fb709c96-04ac-4c01-b5d6-f09cbce5b8a1" />
<img alt="image" src="https://github.com/user-attachments/assets/f014b849-a1be-473d-9116-7ff2db6821f5" />
For the next image, I'm not completely sure which stage it was on. But from here, I coded a system that automatically merged nearby chunks into one. I discovered that the index limit for macroquad was 65,535, as it was 16-bit indices. This meant that there were only a couple of chunks that could merge before it lead to everything over getting deleted, I ended up downloading macroquad source code and editing this so that it would use a larger value however in the end althrough it worked it didn't solve the core problem and I decided I needed more control, so i set off to learn wgpu. 
<img alt="image" src="https://github.com/user-attachments/assets/2da5a2b2-71fa-45db-8797-8a51bcf9c6c4" />
<img alt="image" src="https://github.com/user-attachments/assets/2fd9c923-144b-440e-b687-694feab41627" />


# wgpu
I would be lying if I said I wasn't worried about learning WGPU, as I heard it was extremely difficult. However, I decided that I would see how far I got into it and try my best with it. Below are some of the images as I made my way through the basic learning stages. I should note a large thanks to https://sotrh.github.io/learn-wgpu/intermediate/tutorial12-camera/ for teaching me most of this stuff. I mainly used it for the beginner stage; however, it was a large help with getting started.   
I should note that there are definitely some downsides to learning with this project. Noteably, because I was following the tutorial closely to avoid any confusion, I followed their approach with the app structure, which did lead to this project having a very messy "master struct", which, as of writing this today, is still in place, and I will probably fix it when I get the chance. 
<img alt="image" src="https://github.com/user-attachments/assets/5300d562-3342-4d7e-a816-77350097a488" />
<img alt="image" src="https://github.com/user-attachments/assets/e833046d-3843-4b9f-a4cd-ae3695e6c71d" />
<img alt="image" src="https://github.com/user-attachments/assets/bfcc8ac9-4693-435e-a903-4d89c6828662" />
<img alt="image" src="https://github.com/user-attachments/assets/7c83675d-5ad7-4e7a-853d-1ec6a212fa53" />
<img alt="image" src="https://github.com/user-attachments/assets/b06205bd-fd21-4ca5-8d63-eb2453598bcd" />
<img alt="image" src="https://github.com/user-attachments/assets/9605274b-430c-4c34-93cf-66ded2a15d51" />
<img alt="image" src="https://github.com/user-attachments/assets/66c31e5b-f544-4828-8c15-6ce431830217" />
<img alt="image" src="https://github.com/user-attachments/assets/40aae405-a07f-495b-9d85-632729ffde33" />
The end result was much, much faster than what I was getting with macroquad, but it lacked the performance improvements I had planned. (side of converting to meshes and avoiding drawing inside areas)  
I also tested adding LOD as well, which is exaggerated in this image, so you can see how it looks.
<img alt="image" src="https://github.com/user-attachments/assets/38211614-4e21-49be-ba5f-dda99840c1ea" />

# Mesh buffer
With all this done I still noticed the performance wasn't quite what I wanted, I wanted to be able to look out on landscapes and see massive mountains miles and miles away, a real sacle to behold. And to achieve this something was going to have to change as this was no where near fast enough for this. I'm not quite sure where or how anymore, but around this time, I discovered the lag was almost certainly due to GPU draw calls, since GPU usage was low while CPU usage was high. An interesting point I discovered while learning this is that AMD tends to handle large numbers of draw calls worse than NVIDIA. I don't know if this is a driver or a hardware difference; however, I observed it when a friend with a much better GPU wasn't getting the numbers I expected.  

To fix this, I needed a way to render multiple meshes at once. I had the idea to merge them into one big mesh; however, I thought this was a bad idea, as any little change, like breaking a block, would require a complete remesh, which would cost far too much CPU time and GPU bandwidth. Not to mention, even if it were real-time things changing in the world. This could probably be worked around if it were chunk loading, but chunk unloading would almost certainly cause issues, as it would need to move around vertices to keep it as a single continuous list. To fix this issue, I started reading about API calls in WebGPU and discovered one that sounded very interesting. I found a method which was called 'multi draw indirect'. I found this method did exactly what I wanted, but it had a problem: everything had to be in a single buffer. I thought about how I was going to achieve this for a while;   

I can't remember all the ideas I came up with anymore, but for one reason or another, most had problems, like bitmap being wasteful and not working well when chunks can be different sizes depending on how complex the meshes are.
In the end, I decided I would need to keep track of which spaces I could use. To achieve this, I decided to create a list of free ranges. I decided to use free instead of used ranges, as I found that when I wanted to insert something, it was much simpler to find what I needed: I could just look in each of them and see if it was large enough. This approach seemed great; however, there are 2 problems with it.
 1. What happens when a small mesh gets freed, and then you have too small a free mesh spot left?   
 2. What happens when tons of meshes are in use, and you have a bunch of free mesh spots in between used ranges?

## Mesh buffer defragmentation

To fix both these problems, I developed a defragmentation system. The idea is that, before doing anything, it would first sort the entire list of free spots by start location. After this, it would loop over it, and whenever the end of the free range and the start of the next free range were touching, it would delete the next one and make itself larger so that all the free ranges were as large as they could be. This fixed the first issue; however, the second still remained.
For the second method, I had to introduce a more expensive fix. My original, naive approach to fix this was to have each frame spend a fixed amount of time shifting everything to the left. The idea was to spend only a fixed amount of time to avoid being too expensive. As can likely be guessed, there are many issues with this, such as:
 1. A large amount of tiny meshes needing tons of move calls.
 2. If there is tons of space free, it would use up the same amount of time as if there were a very small amount free.

As can be guessed, this approach is extremely wasteful. And unsurprisingly, it ran really badly. That's why I decided to move them only when needed. The way I found to do this was to think of the problem as only a problem if it actually leads to an issue. This means I would look at all the gaps and decide whether to act on them. The way I did this was that I would look at all the free space available and decide if its large enough to put a mesh inside of by comparing to normal mesh sizes, if it was ignore it and move on, if it wasn't move the data to the right to where this current free mesh space is and merge this mesh free space with the next one. This effectively merged both these meshes. Odds are, after this, it will be large enough for a mesh; however, if it isn't, this can keep repeating until it is. A final optimisation that could be made is to treat all these "issues" as non-issues unless they're preventing something. This means that the mesh only needs these gaps removed if there isn't enough space left to put in a mesh. To fix this, I made it loop over all the free space, and if too much of it is too small to be used compared to what is large enough, it would run the defragmentation code. The result was a massive performance increase: instead of taking tons of draw calls to render everything, it could be done in a single large draw call. The results of the labour are shown below.
<img alt="image" src="https://github.com/user-attachments/assets/378c3717-5050-4978-8585-1a2cf6fa9864" />
<img alt="image" src="https://github.com/user-attachments/assets/de7a808b-5cf9-4945-845f-c3f6b7a3a816" />
I should note that the above lists multiple buffers, not just one. The reason is that, in my testing, the code ran fine on my machine, but on other machines, it couldn't create a buffer large enough to store it all. It also meant that i would have to decide max vram my program can use at launch and couldn't control as it ran. The reason for this was Nvidia vs AMD in my case, but a more general solution was the better approach. The fix was fairly simple: I would create more buffers when they became too full to get anything else out during defragmentation. I should note for this that i decided to put the ammount below the range for when defragmendation runs so that when they are full they wouldn't be forever running defragemtion or running it a huge ammount before they ran out of space. Defragmentation is based on the ratio of unusable free space to usable free space, while the new free buffer is based on the total free space percentage remaining. For this I decided to use 256MB as it was the largest supported by webgpu and it was a safe bet that alot of gpus would support it. 
<img alt="image" src="https://github.com/user-attachments/assets/7d68c89e-d9e6-44bc-804b-3f11b5a92e35" />
<img alt="image" src="https://github.com/user-attachments/assets/ad64dd20-c758-41aa-a729-033ef3dde808" />
Some fun bugs I ran into when coding the system.

This approach, however, still has some downsides that I, as of writing this, still have not fixed.    
1. Instead of moving every chunk mesh individually, it should batch move everything in one go. Meaning lowered calls to the GPU.   
2. The system currently doesn't delete buffers when they are used; they just stay forever.   
  
Neither of these issues is particularly complex or takes a lot of time to fix, but I haven't addressed them in my code yet.
