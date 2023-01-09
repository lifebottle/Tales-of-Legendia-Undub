# Tales-of-Legendia-Undub
There are currently four patches available with this repository:

ToL-ReUndub_1.2.7z
This is a complete Undub of Tales of Legendia with the original Japanese voices restored for the second half of the game

ToL-Pundub_1.1.7z
This is the Partial Undub of Tales of Legendia with the original Japanese voices restored for the second half of the game while retaining the English Dub for the first half of the game, in cutscenes and during combat. This version fixes the crash in the Oresoren Village

ToL-ReUndub-Fix.7z
This can be used to update your existing copy of Tales of Legendia ReUndub v1.1 to v1.2 without requiring the download and reapplication of the much larger patch

ToL-PunFix.7z
This can be used to update your existing copy of Tales of Legendia PunDub v1.0 to v1.1 without requiring the download and reapplication of the much larger patch

Credits:
- The Original Undub Team
- Julian
- Trixarian
- Ethanol

## Technical Stuff
Swapping out voice files from English to Japanese is easy. Find the corresponding AFS archive, move it over, copy the corresponding MBS (contains decompressed file sizes), and update the corresponding record in PTN_INFO.BIN. (contains number of files inside AFS archive)

So, why does the game still not play Character Quest voices when you do that and restore everything?

The answer, this time, is pretty simple. First, we start by looking at the game script files, loaded into memory:

![image](https://user-images.githubusercontent.com/6155506/196266159-02124af6-98fa-457f-8c82-2575ba78bf1e.png)

That 0x05 at the start here indicates the voice clip index.

The first thing to check - does the script for character quests contain the voice clip id?

![image](https://user-images.githubusercontent.com/6155506/196266672-f72e4b7b-b56d-4ed6-badd-633ad28d4d05.png)

The answer, with a huge sigh of relief, is-- yes. That 0x2716 is the voice clip id for that line.

The voice clip id is in the script. Why doesn't it play?

Knowing where the id is being read from, it's super easy to trace and see when the game loads the id, and what it does with it.

We immediately get a hit, and we see it being read, and then written into memory.

![image](https://user-images.githubusercontent.com/6155506/196269477-2ca8bd41-43b8-4a8f-8f4a-fd51d80f29f5.png)

Setting a breakpoint at the address being written to, we find something interesting.

![image](https://user-images.githubusercontent.com/6155506/196269653-780c7540-af46-4952-87b0-f7e3e5bf6694.png)

The id is loaded once, it calls a function, if that function returns 0 it skips the second function and goes to the end, not playing anything. This is the behavior seen when playing character quest lines. So that first function call is likely doing something that tells the game there's nothing to play. Let's investigate.

![image](https://user-images.githubusercontent.com/6155506/196270177-5d4f53e7-860d-40fb-9260-d0ae29ea4319.png)

Two functions to look at. The first one:

![image](https://user-images.githubusercontent.com/6155506/196270310-f8ff341b-13c4-44b1-9635-f42d7928920d.png)

And the second:

![image](https://user-images.githubusercontent.com/6155506/196270383-0eafc229-577a-4b97-9900-a5807f3ce5ff.png)

The first one we can tell pretty easily it's getting some value from a table. The voice clip id is taken, shifted left twice (multiply by 4, as each entry in the table is 4 bytes), and added to some value in memory, aka the start of the table, and then add 0x10 for I suppose a header. Lastly the value from the table is shifted right twice. (Divide by 4)

The second function is straight forward, the decompiled function says it all:
```
return a1 != 0x2023e
```

So.. a value of 0x2023e seems to indicate an invalid value that is skipped.

When we look at the table accessed from the first function, we find it full of 0x0808F9:

![image](https://user-images.githubusercontent.com/6155506/196271389-605ec729-e5fb-46fa-b0d2-56ac5c8b5f06.png)

0x0808F9 shifted right twice = 0x2023e. A-ha!

So... what happens when we edit the table to use the one loaded by the Japanese version? It works! But now, the last remaining challenge is finding this table in the game files so we can replace it.

Setting a write breakpoint at the start of the table, we find a function that is writing it. A closer look, we see it takes two parameters, a0 and a1. a1 is the destination where the file was written to, and a0 looked like the original compressed file. A quick search for it, and we find it's the first file in the SYS_REG.AFS archive. 

The question now is decompressing and re-compressing it. With a ton of help from Ethanol, we determined it was a zeroed buffer LZSS. We decompress the file, and the content matched what we saw in the game. Decompress both NA and JP, find the table, replace NA table with JP table. Re-compress, add CPS header back, rebuild the AFS, rebuild the game, and mission accomplished. 😎

(Note: There is a... probably a somewhat decent chance, that the NA script might have moved voice id's around? Maybe? If anyone notices something wrong please report so we can look into it.)
