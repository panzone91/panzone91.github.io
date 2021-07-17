---
layout: post
title:  "Reverse engineering a videogame"
date:   2021-07-17 21:45:00 +0200
categories: re videogame
excerpt_separator: <!--more-->
---
A month ago a friend told me they were trying to extract the audio data form a videogame but they encountered some problems because the assets format was a little weird.<!--more--> Since I'm trying to improve my security and reverse engineering skills but I've never workdc on a real application I've decided to take this occasion and try this little project. This article doesn't want to be a traditional writeup but I want to present the process I've used for approaching this project: it wasn't the most efficient way to handle it but I've learned a lot and I wanted to share my journey.

## Starting the analysis

In the main directory we have a 32-bit Windows .exe and two files named *data.dat* and *sound.dat*. From the name we can probably assume that these .dat files contains respectively the graphic assets and the sound assets. My first approach was to check if these files were some kind of standard archive but a *file* command didn't gave me anything interesting. Maybe these files are encrypted or obfuscated? 

{% highlight shell %}
$ file mikkus.exe 
mikkus.exe: PE32 executable (GUI) Intel 80386, for MS Windows
$ file data.dat 
data.dat: data
$ file sound.dat 
sound.dat: data
{% endhighlight %}

The game however is able to open and load assets from these archives so I've started to analyze the executable. As a first step I've tried to decompile the .exe with Ghidra.

This is the first time I've tried to analyze a Windows binary with Ghidra but I knew from past experiences that the entrypoint is not the classic main function but WinMain. After having found the entrypoint I've tried to understand from high level what it was trying to do:

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-1.jpg"/>

The first section tries to open a message box and, from the code and the context, I assumed it was asking me if I wanted to run the game fullscreen. Then there are a couple of functions that seems interesting:

* a function call with the argument *".dat"*
* a function call with the argument *"1472"*

After a closer inspection I've understood that the first function simply saves the string *".dat"* in a global buffer while the second function is a little more interesting: this function copies 12 characters from the argument string into a buffer. If the string is shorter than 12 chars then the string is expanded to occupy exactly 12 bytes (so, in our case, the string saved in the global buffer is *"147214721472"*). This function is interesting because it's setting a memory location of a specific size with a specific value: it could be a seed or a key. 

Searching where the global buffer is read I've found this function:

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-2.jpg"/>

I can't report the full function here (and the decompiled form is even worse) but this is most likely a scrambling procedure. At this point I was interested to see the result of this operation but I didn't wanted to reimplement the procedure so I've taken note of its location and I've decided to execute the game in a debugger and then I've retrieved the final value.

## Debugging for fun and profit

Since I'm using a Linux laptop and the game is for Windows I had to create a Windows virtual machine and then I've decided to use [x64dbg](https://x64dbg.com/) as my debugger.

With a breakpoint in the procedure presented before I was able to extract a 12 byte number. While I was in debugging I decided to check if there was any call to classic file reading functions: this way I found a procedure that was doing calls to [CreateFileA](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea), [ReadFile](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile) and [SetFilePointer](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-setfilepointer). I put a breakpoint to this procedure and then I started the game.

Breaking at CreateFileA I've noticed that the game was trying to open a file called *"C:/data.dat"*, failing since the file was in another directory. Exploring the call stack I've noticed the game was using an interesting algorithm for finding the file archive:

* Call [GetCurrentDirectory](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getcurrentdirectory) to obtain the current directory of the exe
* Copy chars from the directory string into a new buffer until it encounter a '/' (copying it)
* Add the string *"data.dat"*
* Try to open the file using CreateFileA using this string
* If the CreateFileA fails, repeat the procedure from the second point
* If the CreateFileA complete succesfully it does some check to the file, like checking if the file is a zero-length file, then it close the file and starts a new thread, passing the sucessful file path as an argument.

I still don't understand why it tries to search the archive file this way: if anyone has any guesses please let me know. 

After this I searched for how the game knows it has to search for the file *"data.dat"*: the *.dat* part is from the string set in the WinMain, while the *data* part is from an argument: the main function of this chain has, as an argument, the string *"data\\MIKKUSUSP.DAT"*. This gave me an insight on how the game asks to load an asset in memory: it uses a format where the first part is the archive to use (for example *data*) and the second part is the name of a file in the archive. After a quick search in the .exe I've found this list of strings:

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-3.jpg"/>

Searching for the sound archive I've found the sound file names too:

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-4.jpg"/>

This confirmed me that these .dat files are archives. Also, the files in these archives are probably classics bitmaps and .WAVs which means that they aren't encoded in some strange way. Now I just needed to know how to find the files in this type of archive.

In the thread created by the procedure with the CreateFileA we have our first call to SetFilePointer and ReadFile. With the debugger I've noticed that it was trying to read the first 0x18 byte from *data.dat*. With a memory watcher to the buffer containing these bytes I've found an interesting snipset:

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-5.jpg"/>

As expected, the file is obfuscated and now we know how: it's a simple XOR between the bytes in the file and the key we generated before. These 0x18 bytes are also probably an header for the entire archive.

## Header time

Thanks to the debugger, Ghidra and some intuition I understood the format of the header. First, let's see our 24 byte header in the *data.dat* file in a slightly better format:

**44 58 03 00 20 38 00 00 18 00 00 00 AC 2C 43 03 2C 19 00 00 F0 37 00 00**

This header is formed by 6 4-byte words (in little endian), used in a different way. I will not follow the words in order because the meaning of some value is better understood after seeing other values.

### First word (44 58 03 00)

The first word is checked with a constant in the application: it's a magic number so the game can be sure that the de-obfuscation worked as intended. I've searched online to see if this was a known magic number but I was unable to find any kind of information so it's probably a proprietary format. If anyone knows anything please let me know!

### Fourth word (AC 2C 43 03) and second word (20 38 00 00)

The fourth word is an offset from the start of the file. It points to an area of the file of size equals to the header' second word (so, in our case, this area is at the index *0x03432CAC* from the start of the file and its size is *0x3820* bytes). Checking the file in this location (after the xor operation) we found that this areas has some strings which are the filenames and some other bytes. This block probably contains the informations I need to find each individual file in the archive.

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-6.jpg"/>

Each file slot is composed by a 4-byte word followed by an ASCII string with the filename in uppercase followed by an ASCII string with the filename in lowercase. The first word doesn't have an apparent meaning to me but after some trail and error I've understood that it's actually the composition of two 2-byte shorts: the first one is the number of 4-byte words that are used by the strings in the name, while the second is still a mystery to me: the game checks against it when searching the file so it's probably some kind of hash of the filename.

### Fifth word (2C 19 00 00) and sixth word (F0 37 00 00)

These two word are offsets in the buffer we described in the previous section. The fifth word is where the file list ends and points to a metadata area we will analyze later. The sixth word is where this metadata area ends. 

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-7.jpg"/>

After some more analysis and some math I understand this area is composed by blocks of size 0x2C of metadata (the size is *0x37F0 - 0x192C = 1EC4* and, from the section described before, I know that this archive contains 179 files). I don't understand all the values but after some trail and error I got that, for each block:

* The first 4-byte word is an offset for the section of the file strings, so it can be used to find the file name.
* The tenth and eleventh word are, respectely, the offset in the global file and the file size

All the other words, at the moment, doesn't have any meaning but it doesn't matter! I probably have all I need to do an extractor. But first let's see the last header word.

### Third word (18 00 00 00)

This word represents the offset in the archive file where the buffer with all the files begin. In this case it starts immediately after the header (0x18 bytes) but I don't know why this information is present in header since the structure seems quite rigid (both *data.dat* and *sound.dat* have an header of 0x18 bytes). This value is added to the index present in the file metadata section described earlier to get the actual offset of the file in the archive.

## Extracting the files

Now that I know how to find the bytes for each individual file in the archive, complete with filename and other informations, I was able to create a simple extractor:

* Xor the entire file with the key
* Get the file metadata location from the header
* For each block of 0x2C byte in this region, I access the file name, global file offest (adding the 0x18 described earlier) and the size. Using these information I create a file and copy size bytes from the global offset.

After a quick scripting section I had my files but I was unable to open them. Checking the hexdump of one of the .BMP (since they have a simple magic number) I've noticed that it's shifted by 9 bytes. This probably means that the files are not saved in the archive as cleartext and maybe they are compressed. Probably the game uses this 9-byte header to recover the original file content.

## Decompressing files

I quickly found where the decompression operation is executed. Ghidra wasn't really helpful, as you can see from this image (and this is after I've modified manually the decompiled code):

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-8.jpg"/>

I was able however to understand the header:

* The first 4-byte word is the decompressed file size
* The second 4-byte word is the size of the compressed file (with the 9-byte header)
* The last byte is the prefix code used for compression

After having obtained this algorithm I was able to apply it to one of the files extracted previously and I got my .BMP!

## Extracting the files (for real)

Now I finally had all the informations I needed so I've started to write a script but I still encountered a couple of problems while I was trying to create an automatic procedure:

* Some files in the list are not present in the archive file itself. I presume, from the names, that these are some sort of markers to indicate where a specific section of files started. Since they are easy to recognize by their name and, at least in this example, they were ony three I was able to do a check to ignore them during the extraction.

* The extractor didn't worked on the *sound.dat* file. After a quick check on the files I've noticed that these are non compressed using the algorithm I've presented before. Probably in the file metadata section there is a flag to indicate if the file is compressed but in this case was quicker to disable the decompression function while extracting this archive.

Finally I had all the assets extracted from these archives. Success!

<img src="/assets/blog/images/rev-eng-a-game/rev-eng-a-game-9.png"/>

## Conclusions

While I didn't explored everything that happens in the application I was able to reach my main goal (extracting the assets) and in doing so I've learned a lot about debugging and reverse engineering. It was the first time I've tried to debug a real application and it was the first time I actually looked to a Windows application! I really enjoyed the exploration, the procedure to create an hypothesis based on what I knew about the application and checking if it was correct or I made a mistake. It was a fun little project and I hope this post can be helpful to someone.
