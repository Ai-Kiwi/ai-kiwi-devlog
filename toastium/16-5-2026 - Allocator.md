# Allocator

## Problem

After setting up the pages, the next logical step is to provide a way to break up that data so the kernel can store temporary data without using a large amount of space. Currently, precedent can be done with BSS; however, this has 2 problems.  
 - Data can't be expanded to a larger size  
 - Fixed size means unused space is wasted  
 - Created at compile time, meaning if the driver isn't loaded or the variable isn't used, space is wasted  
The fix for all these is to use some temporary variables. The cleanest way to do this is to break up our already existing pages. However, the question now arises. How do we store which locations are in use and which are free? The program that stores this data and handles its location is called an allocator.  

## Ideas to store allocated data

My first thought to approach this was to do a bitmap allocator like pages; however, the cost here would be much higher, as the information would be much smaller, in this case, single bytes, as the information doesn't necessarily align to borders. 

My next thought on this was to split the data into pages for each different size. This means the bitmap can be much larger, so it can be 1 BIT per item of data for storing used and free info, instead of either splitting it per-byte for size or leaving gaps per item. For performance, I decided to limit these to sizes that followed 2^N and round up into those. The reason for this is that a lot of different sizes could make wasteful page assignments that are rarely used. 
> [!NOTE]
> This idea is very similar to another called 'slab allocator', which is roughly what I ended up landing on.

Another idea is bump allocator, the idea is you store a current up to of where you are and you return this location and increment by the size you assigned, this idea is easy and simple however doesn't support releasing data as there is no way to go back without possibly overwriting data and you don't know what is free and whats released, for this reason it wouldn't work.
> [!NOTE]
> This idea is simple enough and has some use cases, such as drivers that have permanent variables but might not be loaded. However, it is a fairly close use case to BSS. In practice this was easy enough to code I just coded it as well increase its needed for something. In my code base, it is less than 20 lines of code.  

Another idea is to store lists, as with Voxelith, where each list stores the start and end locations. But once again, a similar problem to the pager would occur; it would become messy, however, possibly not as much as you could store index pages and have them chain to each other, but it would still be messy and would get fragmentation problems.  

## Result decided on

In the end, I decided on a bitmap; it's not as large as you might think for an 8-byte value, which is most common for small. You would be spending 1 BIT to store it, which means 1/64th, or 2%, waste. Which isn't a huge amount. However, as with the pager, it is very beneficial for CPU cache, as it can be quickly iterated over.  

An idea I had to speed up bitmap reading was to have a counter that tracked how much free space there is in "chunk" of bitmap entries, it could then be smartly skipped or read depending on the value of this and could split areas, this idea was skipped as it would introduce branching throwing off prediction, even if normally false and also introduces complexity for not a large amount of benefit. "An idiot admires complexity, a genius admires simplicity," - Terry Davis.   

For actually storing the data, the thought crossed my mind to do linked data pages, and for each data page loop over header to header and then jump page to page, however in the end I decided the better approach would be to have an index page as one fetch from ram can fetch 64 indexes instead of one and the CPU will have a much easier time predicting future fetches. A note here is that I again used the idea of groups for sizes rounded up to follow 2^N. This does waste some space: 33-byte allocations are actually stored as 64-bit values, since they round up. I felt the cost here was acceptable for the benefit it brings. 

I saw around this time that C stores data right before each value in RAM, containing info about it, such as its size. My thought was that I could skip this, since most of the info isn't needed. However, we do need to know, for each value, which variable size it stores, as well as its location on the page. The page location can be fetched fairly easily by rounding down to 4096B. This still leaves where page location data would be stored, it could be stored at the start however this means a full page worth of data couldn't be stored and half a page would only allow page as the little extra means a whole another section couldn't be stored.  
> [!NOTE]
> This is a bad thought process. I basically went because it wouldn't be perfect, and I'm going to make it worse. 

For actual coding, anything larger than or equal to a page will cause a kernel panic, as it is too large to fit into a data page due to header data. The solution I had for this was to use large pages for storing data; however, my current pager doesn't support this, so the feature will be added in the future when it is supported.  

## Implantation details

For actually storing the header data before the data in the data pages, I thought I needed the page number, the ID in the page and the page group size. I came up with a solution to store 2 of these in one value. That said, I would store the page number and page ID in a single value by using a modulo operator based on the total number of items that can fit on the page. I could then use divide again to find which page it is. The result is a global ID number that gives me all the info I need. This still leaves the problem of how to store the size. For the size, we can just store it as a single byte using the compressed size, meaning our range would be 2^0 (1 byte) to 2^255, which at 2^255 would be 57896044618658097711785492504343953926634992332820282019728792003956564819968 max bytes per item, which is safe to say way larger than needed. A single 64-bit value would slot in nicely for headers, as it could be reused and save space, since writing 64-bit values must be aligned to 8-byte boundaries. So, having the header this size naturally achieves this at a lower cost and with greater simplicity. This means we get 7 bytes for the global ID and 1 for the size. This would leave 72057594037927936 possible entries, which at 671100 gigabytes, if each entry is only a byte (which it isn't, as it has 8-byte headers), it would be more than enough; it won't be a problem.

In terms of storing this my first thought on how to store this was to use 1 byte as a header and 7 bytes as the global ID, as such:   
[header : 1 byte][global id : 7 bytes]   
However, this did have a problem: if you wanted to store this info, it would be done with a 64-bit register, which would be written to RAM with a 64-bit write instruction. This sounds fine; however, if the system is little endian instead of big endian, the lowest byte of info will get written first, where the header is, and depending on whether the size or the global id is first, one gets overwritten, leading to corruption. You could go byte by byte, reading and writing, and then stitch it all together. However, I felt that an easier way to do this was to just multiply the global ID by 256, then run it modulo 256 to retrieve the size, as this would, at low cost, store the info. This does create a place for issues to arise; however, I felt it wasn't as big a risk as the alternative, which was better. Another potential way to fix this could have been to store only a 32-bit global ID, reserve 3 bytes, and use 1 byte for the size, but I felt this approach was better because it gave more room for the global ID.  

It's worth noting here how wasteful this is. At 1 byte of info, it would waste 8 bytes just storing it, which is pretty huge. This gets a lot better at larger sizes. At 1 byte, its 800% overhead; at 8 bytes of data, its 100% overhead; while at 64 bytes of info, its 12.5% overhead. However, as this increases, the number of items per data page decreases, so another area for waste arises: the bitmap index page can't store as efficiently, and more page location pointers are required, taking up space that could have been used for a bitmap. [touched on more at the end](#missed-optimisations)

As we have no math library, I came up with a clever trick to find the compressed size of a given size. Basically, I would find the most left bit, count how far it is. I would then take only this bit and compare it to the value before, if the whole value was equal I would use this, the bit distance left would be compressed size, however if it was different I could increase it by 1 and that would be the compressed size. This would give us the size we are going to allocate at a lower cost.  

For the index table for each of these I stored the root index table for each, these values are initially set to zero and the idea is when needed if they are still zero it would create a new index table and use that, the zero basicity defining it as uncreated. For the index table them self the layout it follows is that first it will store a 64bit value saying next index table following location (same idea of zero being nothing) then it would store the repeating pattern of 64bit page location then all 64bit entries for bitmap for used vs free for allocator data page. After this ends, it would repeat the pattern, returning to the data page location. When it reaches the end of the index page, it would smoothly roll over to the next index page, skipping the first 64 bits of the page, as that is where the next page starts. This was implemented using a for loop in my code; however, there are definitely cleaner ways to do it, and I feel my approach for iterating over it is messy.

I did think of a few different ways of doing this like doing index table for index table however I felt it was starting to get messy and the overlap between that and a cache for last free was quite large, through I might revisit again when I revisit the allocator again in future. 

## Programming it

Surprisingly when actually programming this it went better then expected, there were a few bugs however not as many as expected. One problem I did run into is when cleaning up my pager, I apparently broke it and set the location for index to the page start location, which was fine when testing the pager, but when writing data, it would overwrite the page bitmap use list, and as the same data was being written, it would cause the same pages to be returned repeatedly. Another issue I found was the bitmap allocator using OR instead of AND with the bitwise NOT operator, which caused pages to be incorrectly marked as used or not used. Either didn't test it well enough or broke it in code cleanup. 

Funny thing about stuff like this is that the smoother it goes, the more concerning it becomes. I have tested the allocator; however, it will almost have to be seen if issues arise later because it didn't work, and it will be very hard to track down. 

## Finished results thoughts

To be honest, I'm not completely happy with the end result here. This is likely an area I will revisit in the future, but in the interest of getting things done, I am calling this "Good enough" for now and revisiting later, as you could spend many years researching different methods and working on the best approach for this. Partially, the waste around the headers per entry.

## Missed optimisations

Now that all this has been coded and I am writing this it has crossed my mind of a much better approach, instead of a header per entry the table could have a entry storing the table id as well as the size group right at the start of the table. This would mean wasting only 8 bytes per data page instead of 8 per entry. I am not entirely sure why this idea didn't occur to me earlier, as it seems fairly self-explanatory.  

The benefit to this is that it allows for less room per item, on first look it seems that it would cost another ram fetch which is a trade off, however in practice the headers are not gone so it remains as one fetch operation. A real cost, however, is that larger data pages have no easy way to know which page they are on. As you can't align to 4096B and confirm some how if that data makes sense as that is user handed off data so a security risk to use. An idea is to split the data so that smaller data uses 1 page with size info at the front, and larger data uses multiple pages. Could also possibly determine which page size is needed based on the size of the data, but once again, the data size would have to be determined first, so the approach doesn't really work. The clean fix here is to add different types of allocators, which is likely the approach I will take in the future to fix this; the current approach works. So is an area to revisit.
