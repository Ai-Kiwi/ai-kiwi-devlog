# Voxelith overview.
This has started being written after voxelith is a good chunk into development, so a fair amount is going to be glossed over or missed. For this, I will also focus only on the rendering side; throughout the whole project, the rendering engine code was rewritten; however, the core program data outside of rendering was kept through graphics recode. 

# The goal
I played a game called Teardown once, and I really liked the idea of a world split into voxel-like terrain, so I thought a game like that would be really cool to make. However, I felt the teardown was kind of a waste of the opportunity. I mean no disrespect to teardown; it just wasn't the game I felt I wanted. So my goal was to create a game that was a merge between Teardown and Minecraft. I should be clear here, I am not really trying necessarily to make the game, as game development isn't really my area of expertise, and I don't really wanna be making a game and tuning everything more, making the engine, if you would say so.  

# Raylib
My first approach for creating this game was to render it in Raylib. The reason behind this was that Raylib was what I had learned before for making a game, and it used Rust, which I was familiar with. The previous project I learned this on was https://github.com/Ai-Kiwi/dungeon-crawler, if I recall correctly. Now, it probably doesn't come as a surprise that it didn't run well at all under Raylib, so I was going to need something a bit more low-level. To achieve this, I chose to use Macroquad, as it lets me avoid GPU-based graphics rendering while still achieving much better performance.  

# Macroquad
Under macroquad, I began programming the project; however, I noticed that the performance was not great, to say the least. In the 2nd image, you see I was getting under 2 fps, which was not the ideal gameplay experience I was hoping for, to say the least. I discovered the reason was the number of different "objects" it was rendering, since each chunk was a mesh. To address this issue, I decided to add chunk merging.  
<img alt="image" src="https://github.com/user-attachments/assets/fb709c96-04ac-4c01-b5d6-f09cbce5b8a1" />
<img alt="image" src="https://github.com/user-attachments/assets/f014b849-a1be-473d-9116-7ff2db6821f5" />
For the next image, I'm not completely sure which stage it was on. But from here, I coded a system that automatically merged nearby chunks into a single chunk. I discovered that the index limit for macroquad was 65,535, as it was 16-bit indices. This meant that there were only a couple of chunks that could merge before it lead to everything over getting deleted, I ended up downloading macroquad source code and editing this so that it would use a larger value however in the end althrough it worked it didn't solve the core problem and I decided I needed more control, so i set off to learn wgpu. 
<img alt="image" src="https://github.com/user-attachments/assets/2da5a2b2-71fa-45db-8797-8a51bcf9c6c4" />
<img alt="image" src="https://github.com/user-attachments/assets/2fd9c923-144b-440e-b687-694feab41627" />


# wgpu
I would be lying if I said I wasn't worried about learning WGPU, since I've heard it's extremely difficult. However, I decided to see how far I got into it and try my best with it. Below are some of the images I made as I progressed through the basic learning stages. I should note that I owe a large thanks to https://sotrh.github.io/learn-wgpu/intermediate/tutorial12-camera/ for teaching me most of this stuff. I mainly used it for the beginner stage; however, it was very helpful in getting started.   
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
With all this done, I still noticed the performance wasn't quite what I wanted. I wanted to be able to look out on landscapes and see massive mountains miles and miles away, a real scale to behold. And to achieve this, something would have to change, as this was nowhere near fast enough. I'm not quite sure where or how anymore, but around this time, I discovered the lag was almost certainly due to GPU draw calls, since GPU usage was low while CPU usage was high. An interesting point I discovered while learning this is that AMD tends to handle large numbers of draw calls worse than NVIDIA. I don't know if this is a driver or a hardware difference; however, I observed it when a friend with a much better GPU wasn't getting the numbers I expected.  

To fix this, I needed a way to render multiple meshes at once. I had the idea to merge them into one big mesh; however, I thought this was a bad idea, as any little change, like breaking a block, would require a complete remesh, which would cost far too much CPU time and GPU bandwidth. Not to mention even if it were real-time changes in the world. This could probably be worked around if it were chunk loading, but chunk unloading would almost certainly cause issues, as it would need to move around vertices to keep it as a single continuous list. To fix this issue, I started reading about API calls in WebGPU and discovered one that sounded very interesting. I found a method which was called 'multi draw indirect'. I found this method did exactly what I wanted, but it had a problem: everything had to be in a single buffer. I thought about how I was going to achieve this for a while;   

I can't remember all the ideas I came up with anymore, but for one reason or another, most had problems, like bitmap being wasteful and not working well when chunks can be different sizes depending on how complex the meshes are.
In the end, I decided I would need to keep track of which spaces I could use. To achieve this, I decided to create a list of free ranges. I decided to use free instead of used ranges, as I found that when I wanted to insert something, it was much simpler to find what I needed: I could just look in each of them and see if it was large enough. This approach seemed great; however, there are 2 problems with it.
 1. What happens when a small mesh gets freed, and then you have too small a free mesh spot left?   
 2. What happens when tons of meshes are in use, and you have a bunch of free mesh spots in between used ranges?

## Mesh buffer defragmentation

To fix both these problems, I developed a defragmentation system. The idea is that, before doing anything, sort the entire list of free spots by start location. After this, it would loop over it, and whenever the end of the free range and the start of the next free range were touching, it would delete the next one and make itself larger so that all the free ranges were as large as they could be. This fixed the first issue; however, the second still remained.
For the second method, I had to introduce a more expensive fix. My original, naive approach to fix this was to have each frame spend a fixed amount of time shifting everything to the left. The idea was to spend only a fixed amount of time to avoid being too expensive. As can likely be guessed, there are many issues with this, such as:
 1. A large amount of tiny meshes needing tons of move calls.
 2. If there is tons of space free, it would use up the same amount of time as if there were a very small amount free.

As can be guessed, this approach is extremely wasteful. And unsurprisingly, it ran really badly. That's why I decided to move them only when needed. The way I found to do this was to think of the problem as only a problem if it actually leads to an issue. This means I would look at all the gaps and decide whether to act on them. The way I did this was that I would look at all the free space available and decide if its large enough to put a mesh inside of by comparing to normal mesh sizes, if it was ignore it and move on, if it wasn't move the data to the right to where this current free mesh space is and merge this mesh free space with the next one. This effectively merged both these meshes. Odds are, after this, it will be large enough for a mesh; if not, it can keep repeating until it is. A final optimisation is to treat all these "issues" as non-issues unless they're preventing something. This means that the mesh only needs these gaps removed if there isn't enough space left to put in a mesh. To fix this, I made it loop over all the free space, and if too much of it is too small to be used compared to what is large enough, it would run the defragmentation code. The result was a massive performance increase: instead of taking tons of draw calls to render everything, it could be done in a single large draw call. The results of the labour are shown below.
<img alt="image" src="https://github.com/user-attachments/assets/378c3717-5050-4978-8585-1a2cf6fa9864" />
<img alt="image" src="https://github.com/user-attachments/assets/de7a808b-5cf9-4945-845f-c3f6b7a3a816" />
I should note that the above lists multiple buffers, not just one. The reason is that, in my testing, the code ran fine on my machine, but on other machines, it couldn't create a buffer large enough to store it all. It also meant that I would have to decide the maximum VRAM my program can use at launch and couldn't control it as it ran. The reason for this was Nvidia vs AMD in my case, but a more general solution was the better approach. The fix was fairly simple: I would create more buffers when they became too full to get anything else out during defragmentation. I should note that I decided to set the amount below the range for when defragmentation runs, so that when they are full, they wouldn't run defragmentation forever or run it for a long time before they ran out of space. Defragmentation is based on the ratio of unusable free space to usable free space, while the new free buffer is based on the total free space percentage remaining. For this, I decided to use 256MB, as it was the largest supported by WebGPU and a safe bet that many GPUs would support it. 
<img alt="image" src="https://github.com/user-attachments/assets/7d68c89e-d9e6-44bc-804b-3f11b5a92e35" />
<img alt="image" src="https://github.com/user-attachments/assets/ad64dd20-c758-41aa-a729-033ef3dde808" />
Some fun bugs I ran into when coding the system.

The full code is a bit involved to simplify cleanly, so for those of you interested.  
The actual code can be seen at: https://github.com/Ai-Kiwi/Voxelith/blob/main/src/render/mesh.rs  
<details>
<summary>Full defragmentation function</summary>

```rust
const MIN_FREE_SPACE_SIZE: u32 =  (0.5 * 1024.0 * 1024.0) as u32;


pub fn mesh_buffer_cleanup(render_state : &mut RenderState, buffer_number : usize) {
    let chunk_cleanup_started = Instant::now();
    let mesh_buffer = render_state.mesh_buffers.get_mut(buffer_number).unwrap();
    //delete dead meshs
    mesh_buffer.meshs.retain(|_key, mesh| {
        let alive = mesh.alive_pointer.strong_count() > 0;
        if alive == false {
            //mesh has been dropped
            mesh_buffer.free_mesh_buffer_ranges.push(FreeBufferSpace {
                byte_start: mesh.byte_vertex_position,
                byte_len: mesh.byte_vertex_length,
            });
        }
        alive
    });

    //delete from actual free spaces anything that has no size
    mesh_buffer.free_mesh_buffer_ranges.retain(|fs| fs.byte_len > 0);
    
    //setup free space info
    let mut free_spaces: Vec<_> = mesh_buffer.free_mesh_buffer_ranges.iter_mut().collect();
    free_spaces.sort_by(|a, b|  a.byte_start.cmp(&b.byte_start));
    
    //delete next to each other and merge into one
    let mut i = 0;
    while i + 1 < free_spaces.len() {
        if free_spaces[i].byte_start + free_spaces[i].byte_len == free_spaces[i + 1].byte_start {
            free_spaces[i].byte_len += free_spaces[i + 1].byte_len;
            free_spaces[i + 1].byte_len = 0;
            free_spaces[i + 1].byte_start = 0;
            free_spaces.remove(i + 1);
        } else {
            i += 1;
        }
    }

    let mut free_space = 0;
    let mut real_free_space = 0;
    let mut fragments = 0;
    let mut need_resizing_fragments = 0;

    for space in &free_spaces {
        free_space += space.byte_len;
        fragments += 1;
        if space.byte_len < MIN_FREE_SPACE_SIZE {
            need_resizing_fragments += 1;
        }else{
            real_free_space += space.byte_len;
        }
    };

    //update specs
    mesh_buffer.stat_percent_mesh_buffer_use = free_space as f32 / MAP_VRAM_SIZE as f32;
    mesh_buffer.stat_percent_mesh_buffer_usable = real_free_space as f32 / MAP_VRAM_SIZE as f32;
    mesh_buffer.stat_fragments_mesh_buffer = fragments;
    mesh_buffer.stat_bad_fragments_mesh_buffer = need_resizing_fragments;
    mesh_buffer.stat_buffer_defragmentation = false;

    //not critical so don't bother
    //leaving till later lets huge areas build up as well which can be skipped
    if mesh_buffer.stat_percent_mesh_buffer_usable > 0.10 {
        return;
    }
    mesh_buffer.stat_buffer_defragmentation = true;

    //move items to clean up gaps
    let mut mesh_list: Vec<_> = mesh_buffer.meshs.iter_mut().collect();
    mesh_list.retain(|mesh| mesh.value().byte_vertex_length != 0);
    mesh_list.sort_by_key(|mesh| mesh.value().byte_vertex_position);

    let mut command_encoder = render_state.device.create_command_encoder(&CommandEncoderDescriptor {
        label: Some("Chunk buffer defrag"),
    });



    let mut next_mesh_pos = 0;
    for mut mesh in mesh_list {
        //println!("{} {}", next_mesh_pos, mesh.1.byte_vertex_position);
        if next_mesh_pos == mesh.value().byte_vertex_position || mesh.value().byte_vertex_position - next_mesh_pos > MIN_FREE_SPACE_SIZE {
            next_mesh_pos = mesh.value().byte_vertex_length + mesh.value().byte_vertex_position;
            continue;
        }

        //find the free spot for this
        'space_test : for free_space in &mut free_spaces {
            if free_space.byte_start != next_mesh_pos {
                continue;
            }
            //write data to temp buffer
            command_encoder.copy_buffer_to_buffer(
                &mesh_buffer.mesh_buffer, 
                    mesh.value().byte_vertex_position as u64, 
                &render_state.temporary_move_buffer, 
                0, 
                Some(mesh.value().byte_vertex_length as u64),
            );
            //write data back in new place
            command_encoder.copy_buffer_to_buffer(
                &render_state.temporary_move_buffer, 
                    0, 
                &mesh_buffer.mesh_buffer, 
                free_space.byte_start as u64, 
                Some(mesh.value().byte_vertex_length as u64),
            );


            //save changes to mesh info
            mesh.value_mut().byte_vertex_position = free_space.byte_start;
            mesh.value_mut().vertex_position = free_space.byte_start / mem::size_of::<Vertex>() as u32;

            //save changes to byte info
            free_space.byte_start += mesh.value().byte_vertex_length;

            break 'space_test;
        }

        //merge all free spaces
        let mut i = 0;
        while i + 1 < free_spaces.len() {
            if free_spaces[i].byte_start + free_spaces[i].byte_len == free_spaces[i + 1].byte_start {
                free_spaces[i].byte_len += free_spaces[i + 1].byte_len;
                free_spaces[i + 1].byte_len = 0;
                free_spaces[i + 1].byte_start = 0;
                free_spaces.remove(i + 1);
            } else {
                i += 1;
            }
        }

        next_mesh_pos = mesh.value().byte_vertex_length + mesh.value().byte_vertex_position; //will go 1 larger then te amount which is expected. As it is 0 based

        //base the limit by how much free room is left in vram
        if chunk_cleanup_started.elapsed().as_secs_f32() > 0.03 / ((real_free_space as f32 / MAP_VRAM_SIZE as f32) * 10.0) {
            break;
        }
    }

    let command_buffer = command_encoder.finish();
    render_state.queue.submit(Some(command_buffer));



    //remove buffers that are empty
    //render_state.free_mesh_buffer_ranges.retain(|x| x.byte_len != 0);
}
```

</details>



This approach, however, still has some downsides that I, as of writing this, still have not fixed.    
1. Instead of moving every chunk mesh individually, it should batch move everything in one go. Meaning lowered calls to the GPU.   
2. The system currently doesn't delete buffers when they are used; they just stay forever.   
  
Neither of these issues is particularly complex or time-consuming to fix, but I haven't addressed them in my code yet.

# Shadow system

For the shadows for my project, I wanted them to be real-time. Conventional baked lighting wouldn't work as the world is dynamic and procedural. I considered for a while how Minecraft handles light levels per side; however, I decided I wanted the world to look more alive and the lighting to be more complex. The way Minecraft handles it would also take a fair amount of CPU resources, as I was thinking of storing it in vertex data and passing that over. The end result was that i decided to get he GPU to handle shadows. I wasn't completely sure where to start with this, and I can't remember the exact way I figured it out, whether it was researching shadow methods or figuring out my own way to the solution, but in the end I landed on the idea of using a depth camera to get the distnace to the sun and then using math in the vertex shader to find out how much it is "in the sun". This is done by performing a reverse matrix operation on the sun depth camera's matrix, then comparing the resulting depth to the reported depth at that position on the camera. If they are close its in the sun if not they are not in the sun and it is in the shadow, this means it can be drawn darkier. This approach is called shadow mapping.  

To improve performance and quality, I decided to implement a method called Cascaded Shadow Mapping. The idea is that you render areas farther from the player at lower resolution, while rendering areas closer to the player at higher resolution. The result is that it looks the same to you, but allows you to stop up close for really high-quality, nice shadows, while when you look out, it goes for a much longer distance. The way this was achieved was by passing multiple cameras and shadow maps, each with different resolutions and sizes, and then using them.  

<details>
<summary>Example shader</summary>
 
```wgsl
// Vertex shader
struct CameraUniform {
    view_proj: mat4x4<f32>,
    inverted_view_proj: mat4x4<f32>,
    position: vec3<f32>,
};
@group(0) @binding(0) // 1.
var<uniform> camera: CameraUniform;

@group(1) @binding(0) var depth_texture_lod0_view: texture_depth_2d;
@group(1) @binding(1) var depth_texture_lod0_samplier: sampler_comparison;
@group(1) @binding(2) var depth_texture_lod0_distance_samplier: sampler;
@group(1) @binding(3) var<uniform> depth_texture_lod0_camera: CameraUniform;

@group(1) @binding(4) var depth_texture_lod1_view: texture_depth_2d;
@group(1) @binding(5) var depth_texture_lod1_samplier: sampler_comparison;
@group(1) @binding(6) var depth_texture_lod1_distance_samplier: sampler;
@group(1) @binding(7) var<uniform> depth_texture_lod1_camera: CameraUniform;

@group(1) @binding(8) var depth_texture_lod2_view: texture_depth_2d;
@group(1) @binding(9) var depth_texture_lod2_samplier: sampler_comparison;
@group(1) @binding(10) var depth_texture_lod2_distance_samplier: sampler;
@group(1) @binding(11) var<uniform> depth_texture_lod2_camera: CameraUniform;

@group(1) @binding(12) var depth_texture_lod3_view: texture_depth_2d;
@group(1) @binding(13) var depth_texture_lod3_samplier: sampler_comparison;
@group(1) @binding(14) var depth_texture_lod3_distance_samplier: sampler;
@group(1) @binding(15) var<uniform> depth_texture_lod3_camera: CameraUniform;

struct VertexInput {
    @location(0) position: vec3<i32>,
    @location(1) color: vec4<f32>,
    @location(2) extra: vec4<f32>, //reflectiveness, roughness, metalic. Normal
};

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) color: vec4<f32>,
    @location(1) world_pos: vec4<f32>,
    @location(2) normal: vec3<f32>,
    @location(3) extra: vec4<f32>,
};

@vertex
fn vs_main( model: VertexInput, instance: InstanceInput, ) -> VertexOutput {
    // ...
}

@fragment
fn fs_main(in: VertexOutput) -> GbufferOutput {

    // ...
    
    //get camera distance to point to pick shadow
    let shadow_camera_diff = vec3<f32>(0 - in.world_pos.x, 0 - in.world_pos.y, 0- in.world_pos.z);
    let shadow_camera_distance = length(shadow_camera_diff);
    let light_clip_pos = depth_texture_lod0_camera.view_proj * in.world_pos;
    let light_coords = light_clip_pos.xyz / light_clip_pos.w;    
    var closeness_response = 0.0;
    if abs(light_coords.x) < 1 && abs(light_coords.y) < 1 {//sun_shadow_size_relative
        let shadow_texture_uv = vec2(light_coords.x, light_coords.y * -1) * 0.5 + 0.5;//sun_shadow_size_relative
        let depth = light_coords.z - 0.001;
        closeness_response = textureSampleCompare(
            depth_texture_lod0_view,
            depth_texture_lod0_samplier,
            shadow_texture_uv,
            depth
        );
    }else if abs(light_coords.x) < 3 && abs(light_coords.y) < 3 {//sun_shadow_size_relative
        let shadow_texture_uv = vec2(light_coords.x / 3, light_coords.y * -1 / 3) * 0.5 + 0.5;//sun_shadow_size_relative
        let depth = light_coords.z - 0.001;
        closeness_response = textureSampleCompare(
            depth_texture_lod1_view,
            depth_texture_lod1_samplier,
            shadow_texture_uv,
            depth
        );
    }else if abs(light_coords.x) < 8 && abs(light_coords.y) < 8 {//sun_shadow_size_relative
        let shadow_texture_uv = vec2(light_coords.x / 8, light_coords.y * -1 / 8) * 0.5 + 0.5;//sun_shadow_size_relative
        let depth = light_coords.z - 0.001;
        closeness_response = textureSampleCompare(
            depth_texture_lod2_view,
            depth_texture_lod2_samplier,
            shadow_texture_uv,
            depth
        );
    }else if abs(light_coords.x) < 24 && abs(light_coords.y) < 24 {//sun_shadow_size_relative
        let shadow_texture_uv = vec2(light_coords.x / 24, light_coords.y * -1 / 24) * 0.5 + 0.5;//sun_shadow_size_relative
        let depth = light_coords.z - 0.001;
        closeness_response = textureSampleCompare(
            depth_texture_lod3_view,
            depth_texture_lod3_samplier,
            shadow_texture_uv,
            depth
        );
    }else{
        closeness_response = 0;
    }

    // ...

    return gbuffers;
}
```

</details>

Personally, I feel this current approach is messy; it results in a massive list of textures and cameras to import and consumes a lot of code. I attempted to use 3D Textures; however, they have size limits much lower than 2D, and area is also an extra feature that has to be enabled, if I am not mistaken. This is an area I would like to improve; however, I am currently unsure how to go about it.  
  
<img alt="image" src="https://github.com/user-attachments/assets/3b60c9d0-2eae-4430-8d2a-7d8cf9801379" />





