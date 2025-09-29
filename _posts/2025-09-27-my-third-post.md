---
layout: single
title: "Design a multi-threading file sharing server"
date: 2025-09-27
author: "Cheng Qu"
categories: [blog]
tags: [multi-threading]
---

Design

Converting the code that retrieves the file from disc with code that retrieves it from web, the webproxy pass in a server url which will be passed into the callback registered via the GFS_WORKER_FUNC. After understanding this, the design is pretty straight forward. We would need a buffer to hold the url, and construct a full url from the argument passed in. 

Then we will construct a new curl easy setopt object. We define a function to accept LibCurl’s output into a struct (BufferStruct) and pass a function pointer to LC. This callback function writes the output to BufferStruct. 

Based on the response code we can send header and the data chunk by chunk as before. One downside of current implementation is that the callback function is useful for smaller files but might not be ideal for very large files as it requires enough memory to hold the entire file at once.

The problem(s) I saw when completing this part:
-	Got error: tests that server properly handles requests for non-existent files

How I tested and resolved the issue:
-	I solved issue by fixing the logic for handling the error codes.


## Part 2

Design:

Multithreaded proxy will interreact with multithreaded cache processes located on the same physical machine. Each of the request received by A worker of the proxy need to be relay over to the cache, the cache need to respond to request with the file content with the specific requirement that file transfer must be performed using shared memory and only one shared memory segment need to be used for a file transfer.

The design diagram for major components:

 


The first important design choice is the separation of data channel and communication channel.  Use the shared memory only for storing file content and pass the request via message queue. The separation has many benefits, first and foremost is the simplicity compared with only using shared memory for both storing file and pass request.  The shared memory would require complex coordination. Both cache and proxy worker need to take turns to write and read to it, because it’s sending data in chunks. It will reduce complexity if we can separate request passing functionality. Secondly, because the request needs to make its way from proxy to cache in one direction only, it makes sense to use a message queue for that. 

Therefore, the proxy (webproxy) creates the shared memory segments, and the proxy needs to tell the cache (simplecached) about these segments through the message queue. One key understanding is that the shared segment will need to be accessed by two threads in different processes. The proxy reads from it and the cache writes to it. Synchronization is the key for making sure memory won’t get corrupted. I used semaphore for synchronization, because it's a recommended choice for synchronizing memory access.  I understood there could be other design choices for memory synchronization as well, such as pthreads, but from what I read using semaphore will be more graceful and it’s proven to be effective on every platform, so I did not explore other options. 

Another important design choice is how many segments the processes will use to communicate.  I explored both options. At first, I used one large segment. I used pointer arithmetic to keep track of the segments. This creates issued later on when I tested multithreads using 6200 tools. This problem created a lot of stress and headache for me, but it didn’t reveal itself in the initial design phase. In the end, I ended up with n segments. I gave a unique id for each segment and enqueue them for the proxy workers to consume. When the worker gets the segment from queue and use the segment to transfer data with cache worker, it has a unique id to work with. This unique id is passed to the cache worker in a request through the communication channel i.e. message queue.  This way ensures the proxy and cache worker will access the same memory segment. This is a key element to making sure it will work. 

A messy problem I encountered was making sure the synchronization works when the proxy worker and cache workers reads and writes chunks of data into this memory segment. I used semaphore for synchronization, but a key understanding for how it works comes from reading the Kerrisk Linux IPC slides, reading piazza posts and experimenting. I printed out semaphore values to pinpoint the issue. The access pattern is as follows: proxy worker waits for cache worker to write first, cache worker writes to shared memory, signal proxy status and file size. Proxy worker then reads from the memory segment, gets the information about file size and status. Depending on the status, it sends header and begin sending data only if status is ok. After each chunk is sent, signal cache worker. The cache worker only writes the next chunk after getting the signal. After writing the chunk, the cache worker signal proxy worker data is ready. The proxy worker waits at the beginning of the loop to transmit the next chunk of data. The cache worker also waits at the beginning of the loop for its turn to write the next chunk of data.  Also importantly, when cache worker reads data, it should use pread to read from offset. 

Other design decisions, such as choosing between the POSIX and System V APIs for creating message queues and semaphores, are relatively easy to make. I chose POSIX because it’s newer and more modern, and also because I found the documentation for POSIX i.e. Linux manual page has more code examples. 

The problem(s) I saw when completing this part:

-	Photo were not copied
-	Photo were copied but blurred and corrupted by visual examination
-	Cannot start simplecached first. 
-	Memory leak found by sanitizer. 
-	Seems like chunks to send keeps getting smaller and smaller. 
-	Failed at mq_open.
-	Proxy hung
-	Failed the stress test.
-	Everything passed but still failed Gradescope, got 404 path not found error. 

How I tested and resolved the issue:

-	I tested using one thread, one segment and visually inspecting the photos were copied over and can view them. 

-	I solved the mq_open issue by doing mq_unlink and fix my clean up logic. So basically, if a kill or control-c signal is given, the proxy responsible for creating shared memory segment will make sure the memory segment(s) get clean up. Shared queue for work distribution will need to be destroyed. Cache response for creating the message queue will need to unlink that message queue and call simplecache_destory that’s provided to us.

-	The memory leak occurred because I was trying to print a request that hasn’t been properly allocated. I fixed it by initializing it properly.

-	The proxy hung issue was caused by not properly broadcasting when the proxy worker is done with shared memory. When there is one shared memory to work with the issue will happen.

-	After the simple tests works, the stress test will always fail. I realize this is because when there are multiple threads working on multiple segments transferring file of various sizes and status, my synchronization just wasn’t robust enough to deal with the situation. It took me a lot of time to find what the problem is. In the end, I found a solution by introducing a chunk_size variable in the shared memory struct, in addition to seg_size variable. The seg_size variable is usd to track the segment size, and there is also a seg_size variable in the request struct. These two values are the same. The chunk_size is used to track size of each chunk sent. I am surprised that earlier not using this also worked, for the manual testing of transferring the four photos stored under the file folder. 

-	Even after the stress test were passed, when I submit to Gradescope it fails again. This time I noticed it returns 404, indicating the file path is not parsed correctly. Turns out that for part 2 we don’t need to concatenate the file path like we did for part 1. 

Code reference:
1.	The libcurl API documentation
2.	https://www.hackthissite.org/articles/read/1078
3.	My previous pr1 code i.e. code in Beej’s Guide to Network Programming
4.	My previous pr 1 multi thread code
5.	Kerrisk – An introduction to Linux IPC slides
6.	Linux manual page code examples. 