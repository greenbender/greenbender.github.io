---
title: "cybearsCTF 2019: Block Dude Writeup"
excerpt_separator: "<!--more-->"
categories:
  - CTF
  - Writeup
tags:
  - CTF
  - Writeup
  - cybearsCTF
  - Block Dude
---

Block Dude is a simple pygame.

We aren't told much about the game but are supplied with a tarball containing
the following:

```console
$ tar -tf handout.tar.gz
./
./tiles/
./tiles/wall.png
./tiles/door.png
./tiles/right.png
./tiles/empty.png
./tiles/left.png
./tiles/block.png
./client.c
./blockdude.h
./BlockDude.py
./libblock_dude.so
```

And instructions for spawning the game:

```console
python3 BlockDude.py --port 31337 block-dude.chal.cybears.io
```

Unfortunately at the time of writing I don't have access to the server, and I
didn't keep any screenshots during the CTF so I can't show images of the game.
I'll address this when Cybears release the CTF publicly.

If you want to attempt this CTF challenge yourself *stop* reading here.


### Analysis ###

After extracting the tarball I had a quick play with the game. Its a simple
platformer. I can move the character around and there is a wall impeding our
progress to the left and a block at the far right of the game. I mess around
for a little bit not discovering anything. At this point I've got no idea what
the objective is. I know I need to get the flag for the CTF, maybe just
finishing the game will give me the flag? Do I simply have to bypass the wall
to do that? Time to look at the game code and see what I can do.

#### BlockDude.py ####

Not much here. Its just taking user input UP, DOWN, LEFT, RIGHT and drawing the
game map tiles based on what is supplied by the server. I notice there is DOOR
tile that I haven't seen yet. Still no idea how to play the game, let alone get
the flag.

Time to see what else we've got. Lets go for the source code first.


#### blockdude.h ####

OK this appears to be a header for the server-side portion of the game.

There is a an extern for the map state global.

```c
extern u8 gMAP[MAP_Y][MAP_X];
```

And the game state struct.

```c
typedef struct State_t State;
typedef void (*update_t)(State *);

typedef struct Block_t {
    u8 x;
    u8 y;
} Block;

struct State_t {
    u8 num_blocks;
    u8 cur_block;
    u8 has_block;
    u8 y;
    u8 x;
    i8 dir;
    char cmd;
    u8 success;
    update_t update;
    // TODO: is 8 blocks enough?
    Block blocks[8];
};
```

This struct is very suspect, it has so much potential. Integer overflow for the
`u8`'s, negative index into `blocks` and maybe I can modify the `update`
function pointer, or overflow `blocks` to overwrite stack or heap memory. The
`TODO` kind of points toward the last option. NOTE: at this point I haven't even
figured out how to create a block, maybe 8 blocks is enough?

I definitely need to have a look at how this struct is used, and especially
how the `blocks` member is accessed.


#### client.c ####

This is a source file for the server-side portion of the game (despite the name
client.c). We can see that is implementing functions defined in the
`blockdude.h` header, and updating the games state.

A quick browse through the source code and I see this

```c
void update_south(State *state)
{
    // ...
    // Pickup Box if not carrying and there is a free block in front
    // plus special case for spawn box
    else if ((!state->has_block && horizontal == TILE_BLOCK && diagonal != TILE_BLOCK) ||
             (state->x == 1 && state->y == 1 && state->dir == DIR_RIGHT))
    // ...
}
```

What! Thats how you pickup a block! Just stand next to the block on the screen
and press down. Feeling pretty stupid right now. Time to play the game again
with this new info.

So... You pickup and put down blocks to build stairs over the walls (theres two
of them) then walk out the exit door to finish the game. No flag but at least I
now know how the game works.


#### libblock_dude.so ####

This is a shared library that is loaded by the server-side of the game. It
implements functions that are defined in `blockdude.h` and is the complied
version of `client.c` plus some more stuff, as can be seen here:

```console
$ nm libblock_dude.so 
         U __assert_fail
00003000 B __bss_start
00000c48 T client_game
         U close
00000c79 T debug_flag
00002f78 d _DYNAMIC
00003000 B _edata
000033a4 B _end
         U exit
000006f0 T game_complete
00000cef T get_block_at
00000dbb T get_block_index
00003020 B gMAP
00001142 T load_map
00000b40 T main_loop
         U memset
         U open
00000e35 T output_window
000006a0 T parse_command
00001258 r __PRETTY_FUNCTION__.2924
         U read
         U recv
         U send
00003000 B SOCKET
00000b2a T update_east
00000a2a T update_horizontal
00000719 T update_north
00000802 T update_south
00000b14 T update_west
```

Of particular interest here is the `debug_flag` function. Heres the flag. I
take a quick look just in case... but it reads the flag from a file called
`flag.txt` on the server.

```nasm
.text:00000C79 ; void debug_flag()
.text:00000C79                 public debug_flag
.text:00000C79 debug_flag      proc near
.text:00000C79
.text:00000C79 c               = byte ptr -5
.text:00000C79 fd              = dword ptr -4
.text:00000C79
.text:00000C79                 sub     esp, 8
.text:00000C7C                 push    0               ; oflag
.text:00000C7E                 push    offset file     ; "flag.txt"
.text:00000C83                 call    open
```

So to get the flag I need to call this function. Looking at the disassembly it
seems there is no path that calls this function. I need to expoit a
vulnerability!


### Finding bugs ###

Time to take a look at how that dodgey State struct is used.

Its allocated on the stack in the `main_loop` function.

```c
void main_loop(void)
{
    State state;
    memset(&state, 0, sizeof(state));
    // ...
}
```

Nice... and hey it's the *only* stack allocation in this function. That means
that the return address for `main_loop` is right below it on the stack. That
means that if I can write off the end of the `blocks` member of the state I
can modify the return address. This is now my goal - make `main_loop` return
to `debug_flag`.

So how do I overflow `blocks`? The only place it is used is in the
`update_south` function.

```c
void update_south(State *state)
{
    // ...
    // Drop Box
    // ...
        state->blocks[state->cur_block].y = height + 1;
        state->blocks[state->cur_block].x = state->x + state->dir;
        state->has_block = 0;
    // ...

    // Pickup Box if not carrying and there is a free block in front
    // plus special case for spawn box
    else if ((!state->has_block && horizontal == TILE_BLOCK && diagonal != TILE_BLOCK) ||
             (state->x == 1 && state->y == 1 && state->dir == DIR_RIGHT))
    {
        u8 index = 0;
        if (state->x == 1 && state->y == 1 && state->dir == DIR_RIGHT)
        {
            // TODO: do newly spawned boxes need to be initialized?
            index = state->num_blocks++;
        }
        else {
            index = get_block_index(state->x+state->dir, state->y, state);
        }

        state->has_block = 1;
        state->cur_block = index;
    }
}
```

Analysing this function I see that when dropping a block the `cur_block` member
is used to index into `blocks` and set the `x` and `y` coordinates of the block
based on where "Block Dude" drops it.

I also see that spawning a new block simply increments `num_blocks` without
bounds checks and that the initial location of the block is not set (see the
`TODO`). This means that when I spawn block 9 (remembering there is only enough
space for 8 blocks in the State struct) it's `x` and `y` coordinates are
intially the lower two bytes of the return address of `main_loop`, and if I
spawn block 10 its coordinates are the upper two bytes of the return address.
Nice I have an info leak! lets give it a go.

I spin up the game and immediately spawn 10 blocks. Hey one block is floating
in the air but where is the second one? Oh... the one in the air is block 9.
Block 10 is rendered above the characters head until block 11 is spawned. So
after block 11 is spawned I have two floating blocks. The location of these two
floating blocks is actually telling me the return address of the `main_loop`
function!

It is also interesting to note that after doing this several times the
locations of the floating blocks changes so we can't just overwrite the return
address with a static value. They also always land in the game windows (well
done to the designers of the CTF challenge, thats very neat!)

Modifying the return address of `main_loop` should a simple matter of picking
up blocks 9 and 10 and dropping them somewhere else. May as well give this a
go.

So I spin up the game again and when box 9 spawns I move the character to a
random location and drop the block. Then finish the game (the main loop returns
when you finish the game) and sure enough the connection to the server drops.
Looks like I have a crash!


### Writing the exploit ###

Heres the plan: Spawn 10 blocks minimum, move blocks 9 and 10 so they point at
`debug_flag` and finish the game.

So how can I reliably set the return address?

```nasm
.text:00000C48 ; void __cdecl client_game(i32 socket)
.text:00000C48                 public client_game
.text:00000C48 client_game     proc near
.text:00000C48
.text:00000C48 pad             = byte ptr -404h
.text:00000C48 socket          = dword ptr  4
.text:00000C48
.text:00000C48                 push    edi
.text:00000C49                 sub     esp, 400h
.text:00000C4F                 mov     edx, esp
.text:00000C51                 mov     eax, 0
.text:00000C56                 mov     ecx, 100h
.text:00000C5B                 mov     edi, edx
.text:00000C5D                 rep stosd
.text:00000C5F                 mov     eax, [esp+404h+socket]
.text:00000C66                 mov     ds:_edata, eax
.text:00000C6B                 call    main_loop
.text:00000C70                 nop
.text:00000C71                 add     esp, 400h
.text:00000C77                 pop     edi
.text:00000C78                 retn
.text:00000C78 client_game     endp
.text:00000C78
.text:00000C79
.text:00000C79 ; =============== S U B R O U T I N E =======================================
.text:00000C79
.text:00000C79
.text:00000C79 ; void debug_flag()
.text:00000C79                 public debug_flag
.text:00000C79 debug_flag      proc near
.text:00000C79
.text:00000C79 c               = byte ptr -5
.text:00000C79 fd              = dword ptr -4
.text:00000C79
.text:00000C79                 sub     esp, 8
```

The function that calls `main_loop` is `client_game`. Hey... it also seems to be
'allocating' 1024 bytes on the stack and zeroing it, then not using it at all!
Lets draw the stack for `main_loop`

```
+---------------------------+
+                           +
+     State                 +  <-- Game state
+                           +
+---------------------------+
+     return address        +  <-- Return to `client_game`
+---------------------------+
+                           +
+     1024 bytes of zeros   +
+---------------------------+
```
 
This means that there is 1K of slack space after the return address that won't
affect the process if it is modified. I can use this space to spawn up to 512
additional blocks! NOTE: I realised while proof reading this that although
there is space for 512 blocks in the 'slack' space the maximum number of blocks
that can be spawned is 256, since `num_blocks` will wrap.

OK back to how can I reliably set the return address? From the above
disassembly I can see the return address for the call to `main_loop` is
0x00000C70 and conviently the `debug_flag` function is as 0x00000C79 thats
*only* 9 bytes away!

So I go back and review the code, looking at how dropping a block sets the `x`
and `y` value of the block and how this can be used to modify the return
address of `main_loop` and *eventually* forehead slap myself when I realise how
simple it is. Ready for it? The return address of `main_loop` can be modified
to point at `debug_flag` by moving block 9 just nine places to the left in the
game and dropping it.

Remember block 9 is the lower 2 bytes of the return address. Therefore, the `x`
member of that block is the low byte of the return address. So all I have to
do is increment `x` by 9 to point to `debug_flag` and moving the block nine
places to the left does exactly this!

Oh and I have up to ~~512 extra~~ 254 blocks available to help me with moving the block.


### Getting the flag ###

I could have scripted up the python client to do a 'speed run' and perhaps I
would have if things went differently, however, I was feeling relatively
confident that this *might* work (notice all the qualifiers here) so I just
decided to have one shot at playing the game out.

So heres how the game went:

1. Move to the spawn block.
2. Spawn 11 blocks (9 and 10) end up floating in the air.
    * I didn't at the time, but I could have used blocks 1 through 8 to start
      building stairs overs the walls. This doesn't really matter though. I
      still had plenty of blocks to spare.
3. Leave the floating blocks alone.
4. Spawn blocks one at a time to build stairs over the walls in such a way that
   I can walk back and forward across the map enough to go to and from block 9, 9
   places to the left of block 9, and the spawn block. Also make sure I can get
   the exit door.
5. Spawn blocks one at a time to build a ramp whos top level is two blocks
   wide, one block lower than block nine, and ends nine places to the left of
   block 9.
6. Spawn blocks one at a time to build a ramp that allows me to pick up block nine.
7. Pickup block 9.
8. Move to the top of the first ramp I created.
9. Drop block nine. It should now be at the same height it was originally and
   exactly nine places to the left.
10. Walk to the exit door.
11. *WIN*!! The flag is displayed in the pygame debug/error window.

Heres the ascii art version (kind of):

```
Legend:
# = block
{ = door
_ and | = floor and wall

After spawning 11 blocks:

                                                          # <-- block 10
                                                                        
                                                                        
                                                                        
                                                                        
                                                    # <-- block 9       
                        |                                               
                        |                      |                        
                        |                      |                        
{_______________________|______________________|_______________________#


After building the ramps:

                                                          # <-- block 10
                                                                        
                                                                        
                                                                        
                                                                        
                                                    # <-- block 9       
                        |                  ##       ##                  
                        |#                 ### |    ###                 
                        |##                ####|#   ####                
{_______________________|###_______________####|##__#####______________#


After moving block nine:

                                                          # <-- block 10
                                                                        
                                                                        
                                                                        
                                                                        
                                           # <-- block 9                
                        |                  ##       ##                  
                        |#                 ### |    ###                 
                        |##                ####|#   ####                
{_______________________|###_______________####|##__#####______________#


Now just walk out the door and win.
```
