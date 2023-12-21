---
title: 'Learning from my mistakes in C programming'
date: 2023-12-20
permalink: /posts/2023/12/cmistakes/
excerpt: An exercise in learning from my mistakes while learning C programming
---


I'm brushing up on and learning more C in preparation for my upcoming Georgia Tech course **CS6200: Graduate Introduction to Operating Systems**. In this particular instance, I was learning binary file input/output from Beej's guide. I wanted to give myself an extra challenge and use functions for reading and writing to the file. I had already completed a similar task where I wrote and read strings from a file, so I figured this couldn't be *that* much different.

Things seemed to go fine until I read the bytes back from that file and got way more back than I had put in. I could already see the issue manifesting as I stepped through with a debugger. Below is the code in question. If you know C then you can probably see the problem, and maybe even other less obvious ones. If you do, please bear with me. I am acutely aware of the fact that I'm only just grasping some of these concepts.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BUFFER 5

void readBytes(char *fileName, unsigned char *bytes);
void writeBytes(char *fileName, unsigned char *bytes, int bufferSize);

int main() {
    unsigned char c[BUFFER] = {10, 56, 69, 8, 7};
    char fileName[] = "myfile.bin";
  
    writeBytes(fileName, c, BUFFER);
    readBytes(fileName, c);

    getchar(); // Clunky execution pause so I can see output
    return 0;
}

void readBytes(char *fileName, unsigned char *bytes) {
    FILE *fp = fopen(fileName, "rb");

    while(fread(bytes, sizeof(unsigned char), 1, fp) > 0)
        printf("%d\n", *bytes);
    fclose(fp);
}

void writeBytes(char *fileName, unsigned char *bytes, int bufferSize) {
    FILE *fp = fopen(fileName, "wb");

    fwrite(bytes, sizeof(bytes), bufferSize, fp); // Hint: issue on this line
    fclose(fp);
}
```

There's a pretty obvious error here that I'll get to in a moment, but I'm just figuring things out and didn't even notice it. All I knew was my file was getting more bytes than expected. I tried running code directly from Beej's (without the functions) and it worked fine, so it was my code that was the problem. I decided to run it through `strace` to see if there was a use case for it here.
### Tracing system calls
I started with the two short programs from Beej's guide on reading and writing binary files. I used `pstree -p` to find the PID I wanted and then `strace -fp` to follow the execution of the script. I used `getchar()` to pause execution at the beginning of the main function since I don't know any other way to do so at the moment. I could have used `strace -fp 9` to follow the system calls directly from bash, but bash tends to make extra noise that I didn't want. The programs functioned correctly and returned the following starting with binary writing and then binary reading.

![pstree of binwritetest.exe and strace of PID]({{"/images/posts/mysterybytes/strace3.JPG" | absolute_url }})

![pstree of binreadtest.exe and strace of PID]({{"/images/posts/mysterybytes/strace4.JPG" | absolute_url }})

In both examples, you can see `openat` open the file with attributes like 'read-only' for reading and 'write-only', 'create', and 'truncate' for writing. In the case of writing, the image at the top, you get a single write syscall that writes the bytes to the file. The read program has more write syscalls because it's writing the data from the file to STDOUT. Hence the slight difference between the two writes: `write(3, data, buffer size)` and `write(1, data, buffer size)`. The 1 and 3 are file-descriptors that tell the system call where it's writing data. The file-descriptor (fd) 1 is for STDOUT and fd 3 is the opened file. You can clearly see where it's reading from and writing to by looking at the read/write syscalls.

Now that we've got that out of the way, lets see what my program looks like.

![Compiling and running my broken program, and getting back lots of bytes]({{"/images/posts/mysterybytes/strace7.JPG" | absolute_url }})

Oh boy, yea, that's more than the 5 bytes in the array I created. Similarly, if you open each file up using a hex editor you see the following.

![HxD of Beej's output file]({{"/images/posts/mysterybytes/strace5.JPG" | absolute_url }})

![HxD of my broken program]({{"/images/posts/mysterybytes/strace6.JPG" | absolute_url }})

The first image is the working program from Beej's, and the second is the file from my program. It's more than the initial 5 bytes that I was expecting. The output of `strace` for my program is similarly running off the edge of my screenshot, but there's something else to see there too.

![pstree and strace of broken program showing many byte writes to STDOUT]({{"/images/posts/mysterybytes/strace8.JPG" | absolute_url }})

Take a look at the first `write(3, ...` above. It's unusual because it's way long and contains the name of the output file. That's.... not right....

At this point `strace` wasn't going to help me further, and I already had an idea of what was happening.

### Break out the debugger
I used GDB to look into the program because I not only wanted to see the problem in the program, but I wanted to look into the program stack as well. I removed the `getchar()` in main since I no longer needed it, I set a breakpoint at the `writeBytes()` function, and I ran the debugger. I asked for help from ChatGPT, which, while not entirely useful, did guide me towards the correct answer. The image below covers the debugger output just after setting the breakpoint and running the program.

![Debugger showing program execution and the stack]({{"/images/posts/mysterybytes/strace11.JPG" | absolute_url }})

A lot is going on here, so let me explain. From the top where it says "(gdb) step":
- I took a step into the program to the line where the file is opened
- I ran `x/10xb bytes` which says "examine the first 10 bytes in hexadecimal format at the memory address for variable bytes"
- I ran it again, this time for 20 bytes because I wanted to see a little more

Notice the red boxes and numbering. The output to STDOUT is in decimal (base 10) so I needed to convert the hex (base 16) to decimal so I could compare outputs. The hex 0x6d is 109 in decimal and confirmed what I believed to be the issue. I messed up in my program and my write function was writing binary data from the stack and into my file. The numbers 1 through 5 in red show the numbers that were put into the file from the program array `unsigned char c[]`, and everything after that is coming from memory outside of the array. 

If you didn't discover the issue initially allow me to explain it now.

```c
void writeBytes(char *fileName, unsigned char *bytes, int bufferSize) {
    FILE *fp = fopen(fileName, "wb");

    fwrite(bytes, sizeof(bytes), bufferSize, fp); <--- The problem is here
    fclose(fp);
}
```

The write function `fwrite()` takes the following parameters (left to right): 
- `bytes` - The pointer to the array of elements to be written
- `sizeof(bytes)` - The size in bytes of each element
- `bufferSize` - The number of elements 
- `fp` - The pointer to the FILE object 

The issue is at the `sizeof(bytes)` because bytes is a *pointer*! The pointer is 4 bytes in size whereas the correct size I needed, `sizeof(unsigned char)`, is 1 byte! I was telling my program to write 4 times more information than I meant to! This explains why, rather than writing 5 bytes of data I was writing 20 bytes of data. As such, the program just kept going right off into the stack and writing data from there. When it read the bytes back I got more than I expected.

### Conclusion
I fixed the bug and my program worked flawlessly. I didn't catch the mistake initially because I was handling a lot of concepts that were new to me. I also discovered that yes, `strace` is useful, but sometimes you just gotta break out the debugger and look at the code directly.  Hopefully if you've made it this far you've learned something. I know I did!

Sources used:

[Stack Overflow - File Descriptors](https://stackoverflow.com/questions/5256599/what-are-file-descriptors-explained-in-simple-terms)

[Tutorials Point - fwrite()](https://www.tutorialspoint.com/c_standard_library/c_function_fwrite.htm)

[Man page for write(2)](https://www.man7.org/linux/man-pages/man2/write.2.html)

ChatGPT
