---
title: Building a CHIP-8 interpreter for the IBM Logo ROM
layout: post
redirect_from: chip-8/
---

This post will guide you through the beginnings of writing a CHIP-8 interpreter.
We're going to focus on the 5 instructions needed to run the IBM logo ROM. This
will give you a solid base to continue on your own to finish the rest of the
instructions and add input support for interactivity.

We're going to be using Rust, but this could be implemented in whatever you
feel most comfortable using. I do think it's a great project to learn Rust
though, so consider giving it a go!

1. [What is CHIP-8?](#what-is-a-chip-8)
2. [Loading the ROM](#loading-the-rom)
3. [The Fetch-Decode-Execute loop](#the-fetch-decode-execute-loop)
4. [Fetch](#fetch)
5. [Decode](#decode)
6. [Execute](#execute)
7. [The rest of the owl](#the-rest-of-the-owl)
    - [The `LD I, addr` instruction](#ldi-instruction)
    - [The `LD Vx, byte` instruction](#ld-instruction)
    - [The `DRW Vx, Vy, nibble` instruction](#drw-instruction)
    - [The `ADD Vx, byte` instruction](#add-instruction)
8. [Rendering the frame buffer](#rendering-the-frame-buffer)
9. [What next?](#what-next)

## What is CHIP-8?
<a id="what-is-a-chip-8"></a>

CHIP-8 is an interpreted programming language from the 1970s. It's technically
not an _emulator_ as we're not emulating real hardware, but it is a very
similar experience to writing an emulator so it can be easy to think of it as
one.

According to [Cowgod's CHIP-8 Technical Reference][technical-reference] system
has the following features:

- 4KB (4,096 bytes) of RAM (memory)
- 16 general purpose 8-bit registers (`V0–V15`)
- 1 16-bit register (`I`)
- 16-key hexadecimal keypad
- 64x32-pixel monochrome display
- Delay timer
- Sound timer

To implement an interpreter capable of running the IBM logo ROM we'll need to
implement the memory, registers, and the display. We're going to cheat a bit
with the display and simply produce a single image file.

The original implementation includes 36 different instructions, we'll only need
to implement 4 of them for our ROM.

We're going to be referring to the [Technical Reference][technical-reference] a
fair bit, so keep that open too.

## Loading the ROM
<a id="loading-the-rom"></a>

We're going to start by loading the ROM so we've got a program to work with.

You can find the IBM Logo ROM by searching, but it's tiny so I'll paste it
here, please don't sue me IBM!

```
$ hexdump ~/Downloads/IBM\ Logo.ch8
0000000 00 e0 a2 2a 60 0c 61 08 d0 1f 70 09 a2 39 d0 1f
0000010 a2 48 70 08 d0 1f 70 04 a2 57 d0 1f 70 08 a2 66
0000020 d0 1f 70 08 a2 75 d0 1f 12 28 ff 00 ff 00 3c 00
0000030 3c 00 3c 00 3c 00 ff 00 ff ff 00 ff 00 38 00 3f
0000040 00 3f 00 38 00 ff 00 ff 80 00 e0 00 e0 00 80 00
0000050 80 00 e0 00 e0 00 80 f8 00 fc 00 3e 00 3f 00 3b
0000060 00 39 00 f8 00 f8 03 00 07 00 0f 00 bf 00 fb 00
0000070 f3 00 e3 00 43 e0 00 e0 00 80 00 80 00 80 00 80
0000080 00 e0 00 e0
0000084
```

Some quick formatting gives us a Rust array.

```rust
const IBM_LOGO_ROM: [u8; 132] = [
    0x00, 0xe0, 0xa2, 0x2a, 0x60, 0x0c, 0x61, 0x08, 0xd0, 0x1f, 0x70, 0x09, 0xa2, 0x39, 0xd0, 0x1f,
    0xa2, 0x48, 0x70, 0x08, 0xd0, 0x1f, 0x70, 0x04, 0xa2, 0x57, 0xd0, 0x1f, 0x70, 0x08, 0xa2, 0x66,
    0xd0, 0x1f, 0x70, 0x08, 0xa2, 0x75, 0xd0, 0x1f, 0x12, 0x28, 0xff, 0x00, 0xff, 0x00, 0x3c, 0x00,
    0x3c, 0x00, 0x3c, 0x00, 0x3c, 0x00, 0xff, 0x00, 0xff, 0xff, 0x00, 0xff, 0x00, 0x38, 0x00, 0x3f,
    0x00, 0x3f, 0x00, 0x38, 0x00, 0xff, 0x00, 0xff, 0x80, 0x00, 0xe0, 0x00, 0xe0, 0x00, 0x80, 0x00,
    0x80, 0x00, 0xe0, 0x00, 0xe0, 0x00, 0x80, 0xf8, 0x00, 0xfc, 0x00, 0x3e, 0x00, 0x3f, 0x00, 0x3b,
    0x00, 0x39, 0x00, 0xf8, 0x00, 0xf8, 0x03, 0x00, 0x07, 0x00, 0x0f, 0x00, 0xbf, 0x00, 0xfb, 0x00,
    0xf3, 0x00, 0xe3, 0x00, 0x43, 0xe0, 0x00, 0xe0, 0x00, 0x80, 0x00, 0x80, 0x00, 0x80, 0x00, 0x80,
    0x00, 0xe0, 0x00, 0xe0,
];
```

We want to load the program into memory. CHIP-8 interpreters can access up to
4KB of memory, from `0x000` to `0xfff`, we're going to represent this as an
array (`[u8; 0xfff]`).

We're going to wrap this array in a struct that we're going to use to read and
write.

```rust
const MEMORY_LENGTH: usize = 0xfff;

struct Memory {
    memory: [u8; MEMORY_LENGTH],
}

impl Memory {
    fn new() -> Self {
        Self {
            memory: [0; MEMORY_LENGTH],
        }
    }

    fn read(&self, _address: u16) -> u8 {
        todo!();
    }

    fn write(&mut self, address: u16, value: u8) {
        self.memory[address as usize] = value;
    }
}
```

The Technical Reference says that most programs start at memory address `0x200`
because `0x000` to `0x1ff` was reserved for the actual interpreter
implementation.

We're going to use a simple loop to load the ROM into memory:

```rust
const IBM_LOGO_ROM: [u8; 132] = [
    0x00, 0xe0, 0xa2, 0x2a, 0x60, 0x0c, 0x61, 0x08,
    // SNIP
];
const PROGRAM_OFFSET: u16 = 0x200;

fn main() {
    let mut memory = Memory::new();

    for (address, value) in IBM_LOGO_ROM.iter().enumerate() {
        memory.write(address as u16 + PROGRAM_OFFSET, *value);
    }

    dbg!(&memory.memory[0x200..0x204]);
}
```

Which should give us the following output:

```
[src/main.rs:44] &memory.memory[0x200..0x204] = [
    0,
    224,
    162,
    42,
]
```

## The Fetch-Decode-Execute loop
<a id="the-fetch-decode-execute-loop"></a>

The fetch, decode, execute is a pattern found in lots of emulators. First we
fetch the next instruction from memory, we then decode it into an opcode and
operands, then finally execute it.

Let's define a few function to get a feel for how the interpreter is going to
operate:

```rust
fn main() {
    // Loading the ROM

    loop {
        let instruction = fetch();
        let opcode = decode(instruction);
        execute(opcode);
    }
}

fn fetch() -> u16 {
    todo!();
}

fn decode(_instruction: u16) {
    todo!();
}

fn execute(_opcode: ()) {
    todo!();
}
```

Running this will panic at the first `todo!()` inside the `fetch` function. The
fetch function is where we will read the next instruction from memory.

## Fetch
<a id="fetch"></a>

We don't have a way to read from memory yet, so let's add one to the `impl` for
our `Memory` struct:

```rust
impl Memory {
    // fn new() -> Self

    fn read(&self, address: u16) -> u8 {
      self.memory[address as usize]
    }

    // fn write(&mut self, address: u16, value: u8)
}
```

This allows us to read a single byte from memory. Each instruction is made up
of two bytes that get combined into a single value representing our
instruction. We're going to store this raw instruction in a `u16`.

The Technical Reference says:

> All instructions are 2 bytes long and are stored most-significant-byte first.

If we look at the Wikipedia page for [Endianness][endianness] we can see that
this means CHIP-8 is a "big-endian" system.

This means that the first byte we read will be used as the most significant
byte.

First we read a byte from our current program counter location (`pc`),
`memory.read(pc)` we cast this to a `u16` and shift it left by 8 bytes. This
gives us room to add the least-significant byte. We then read the next byte
(`memory.read(pc + 1)`), cast that to a `u16` and union (`|`) them together.

Let's look at an example of this using some Rust pseudo-code:

```rust
let first = memory.read(pc) as u16 // => 1
// 00000000 00000001

first << 8
// 00000001 00000000

let second = memory.read(pc) as u16 // => 2
// 00000000 00000010

first | second
// 00000001 00000010
```

Using these techniques, here is the full `fetch` method:

```rust
fn fetch(memory: &Memory, pc: u16) -> u16 {
    (memory.read(pc) as u16) << 8 | memory.read(pc + 1) as u16
}
```

One thing we introduced here is the program counter. This is used by the
interpreter to represent the instruction we're going to fetch and will be
updated by our `execute` function to move through the program. We initialize it
to the start of our program (`PROGRAM_OFFSET`).

Here's our updated `main` function:

```rust
fn main() {
    let pc = PROGRAM_OFFSET;
    let mut memory = Memory::new();

    // Loading the ROM

    loop {
        let instruction = fetch(&memory, pc);
        let opcode = decode(instruction);
        execute(opcode);
    }
}
```

## Decode
<a id="decode"></a>

Our decode method is now getting passed the raw instruction. From here we want
to decode it.

```rust
fn decode(instruction: u16) {
    unimplemented!("{:#06x}", instruction);
}
```

This should give us the following error:

```
thread 'main' panicked at 'not implemented: 0x00e0', src/main.rs:63:5
```

If we search the Technical Reference for `00e0` we come to the following
documentation:

```
00E0 - CLS
Clear the display.
```

So this is the `CLS` instruction! Let's think about a data structure to
represent our decoded instructions, I'm going to call them opcodes and
represent them as an enum.

```rust
#[derive(Debug)]
enum Opcode {
    CLS,
}
```

Let's modify our `decode` function to return `Opcode::CLS` when we encouter
`00e0`. For all other values we're going to panic and print the raw
instruction.

```rust
fn decode(instruction: u16) -> Opcode {
    match instruction {
        0x00e0 => Opcode::CLS,
        _ => unimplemented!("{:#06x}", instruction),
    }
}
```

Our `decode` function needs to be updated to return an `Opcode`, which also
means our `execute` function needs to be updated to take an `Opcode`. Let's
also make it print the opcode for good measure.

```rust
fn execute(opcode: Opcode) {
    todo!("{:?}", opcode);
}
```

## Execute
<a id="execute"></a>

Our execute method now takes an `Opcode`, even though there is only a single
value so far, let's match on the opcode:

```rust
fn execute(opcode: Opcode) {
    match opcode {
        Opcode::CLS => {
            // NOTE: We don't need to do anything here as we initialze an empty frame buffer and
            // we only draw a single frame, so we never need to clear it.
        }
    }
}
```

As the comment states, we don't actually need to do anything with this opcode
because we're only going to render a single frame, the IBM logo.

If you run the program now you'll end up in an infinite loop. Let's fix that by
moving on our program counter. We've read two bytes, so let's move it on by 2.
We can do this by making `pc` mutable, and passing a mutable reference into our
`execute` function.

```rust
fn execute(opcode: Opcode, pc: &mut u16) {
    match opcode {
        /* SNIP */
    }

    pc += 2;
}
```

We should now be hitting a new error, back in `decode`. This means we're ready
to draw the rest of the owl.

## The rest of the owl
<a id="the-rest-of-the-owl"></a>

The way we're going to progress our interpreter is to just follow our errors,
first we'll update our `decode` method to decode a new instruction, then we'll
update `execute` to update the state of our interpreter.

### **`LD I, addr`**
<a id="ldi-instruction"></a>

Let's take a look at our new error:

```
thread 'main' panicked at 'not implemented: 0xa22a', src/main.rs:84:14
```

Searching the Technical Reference for `a22a` doesn't return anything this time.
This is because, as mentioned earlier, most instructions are made up of an
opcode and some arguments.

All the opcodes we're going to be looking at can be identified by the first
nibble of the two byte raw instruction. In this case the `a` at the beginning.

To get at the first nibble (the first 4 bytes) can be accessed with some
bit-shifting:

If we look at the value 0xa22a as binary we see the following 16 `1`s and `0`s:

```rust
1010001000101010
```

A bit of formatting shows the nibbles more clearly:

```rust
// 0x a    2    2    a
      1010 0010 0010 1010
```

Shifting that value to the right by 12 places gets us the following

```rust
instruction >> 12
1010 // (hex: 0xa, decimal: 10)
```

Let's modify our `match` to only look at the first nibble. We've had to nest
our `0x00e0` arm into a under a new `0x0` arm.

```rust
match instruction >> 12 {
    0x0 => match instruction {
        0x00e0 => Opcode::CLS,
        _ => unimplemented!("{:#06x}", instruction),
    }
    _ => unimplemented!("{:#x}", instruction >> 12),
}
```

We now get the following error:

```
thread 'main' panicked at 'not implemented: 0xa', src/main.rs:84:14
```

Looking again at the Technical Reference we need to look with the instruction
that starts with `a`, we can see `Annn` listed under the instructions:

```
Annn - LD I, addr
Set I = nnn.

The value of register I is set to nnn.
```

Let's update our `Opcode` enum:

```rust
#[derive(Debug)]
enum Opcode {
    CLS,
    LDI(u16),
}
```

This instruction includes an operand, `nnn`, which represents 3 nibbles, so it
won't fit in a `u8`, hence we represent it using a `u16`.

We now need to decode the `nnn` operand from the raw instruction.lBack to bit
fiddling. This time we want the last 3 nibbles, we can mask them off with the
following intersection:

```rust
instruction & 0x0fff
```

A quick example:

```rust
// 0x 0    f    f    f
      0000 1111 1111 1111

      1111 1111 1111 1111 & 0x0fff
// => 0000 1111 1111 1111
```

Let's update our `match` statement again:

```rust
match instruction >> 12 {
    0x0 => match instruction {
        0x00e0 => Opcode::CLS,
        _ => unimplemented!("{:#06x}", instruction),

    }
    0xa => Opcode::LDI(instruction & 0x0fff),
    _ => unimplemented!("{:#x}", instruction >> 12),
}
```

Running our program again, the next error we encounter is because we're not
handling the `LDI` opcode in our `execute` function:

```rust
error[E0004]: non-exhaustive patterns: `LDI(_)` not covered
  --> src/main.rs:67:11
   |
67 |       match opcode {
   |             ^^^^^^ pattern `LDI(_)` not covered
...
78 | / enum Opcode {
79 | |     CLS,
80 | |     LDI(u16),
   | |     --- not covered
81 | | }
   | |_- `Opcode` defined here
```

Looking back at the Technical Reference:

```
Annn - LD I, addr
Set I = nnn.

The value of register I is set to nnn.
```

This is our first time using a register, in this case the `I` register. The
docs say we need to set the register `I` to the value `nnn`.

As we saw in our intro, we have 16 general purpose 8-bit registers and a single
16-bit register referred to as `I`.

Let's define a struct to represent our registers:

```rust
struct Registers {
    registers: [u8; 16],
    i: u16,
}

impl Registers {
    fn new() -> Self {
        Self {
            registers: [0; 16],
            i: 0,
        }
    }
}
```

We also need to initialize it in `main` and pass it through to our execute
function:

```rust
fn main() {
    // SNIP

    let mut registers = Registers::new();

    loop {
        let instruction = fetch();
        let opcode = decode(instruction);
        execute(opcode, &mut registers, &mut pc);
    }
}

fn execute(opcode: Opcode, registers: &mut Registers, pc: &mut u16) {
    // SNIP
}
```

Again, looking back at our docs:

```
Annn - LD I, addr
Set I = nnn.

The value of register I is set to nnn.
```

This literally means we need to take the value represented by `nnn` and set the
`I` register to that value. Using this information we can add a new arm to our
execute `match` statement:

```rust
Opcode::LDI(nnn) => registers.i = nnn,
```

Running our program should give us our next error in the `decode` function:

```
thread 'main' panicked at 'not implemented: 0x6', src/main.rs:78:14
```

### **`LD Vx, byte`**
<a id="ld-instruction"></a>

This one is slightly harder to decode, we need to extract both the `x` and `kk`
values:

```rust
0x6 => Opcode::LD(
    ((instruction >> 8) & 0xf) as u8, // x
    (instruction & 0x00ff) as u8,     // kk
),
```

The docs say the following:

```
6xkk - LD Vx, byte
Set Vx = kk.

The interpreter puts the value kk into register Vx.
```

I'll leave the updating `execute` as an exercise for the reader. I recommend
adding a `write` function to `Registers` to keep things clean.

### **`DRW Vx, Vy, nibble`**
<a id="drw-instruction"></a>

The draw instruction is a bit more complicated, so let's look at this one
together.

Let's take a look at what the Technical Reference has to say:

> Display n-byte sprite starting at memory location I at (Vx, Vy), set VF =
> collision.
>
> The interpreter reads n bytes from memory, starting at the address stored in
> I. These bytes are then displayed as sprites on screen at coordinates (Vx,
> Vy). Sprites are XORed onto the existing screen. If this causes any pixels to
> be erased, VF is set to 1, otherwise it is set to 0. If the sprite is
> positioned so part of it is outside the coordinates of the display, it wraps
> around to the opposite side of the screen. See instruction 8xy3 for more
> information on XOR, and section 2.4, Display, for more information on the
> Chip-8 screen and sprites.

While implementing the last instruction, I'm going to assume you've added a
`read` function to your `Registers` struct.

Decoding:

```rust
0xd => Opcode::DRW(
    ((instruction >> 8) & 0xf) as u8, // x
    ((instruction >> 4) & 0xf) as u8, // y
    (instruction & 0x000f) as u8,     // n
),
```

Executing:

<aside>
  <p>
    We're not handling sprite wrapping or collision detection here as it's not
    needed to render the IBM logo. It's not much extra code to add it, and
    should probably be done while testing more complicated ROMs.
  </p>
</aside>

```rust
Opcode::DRW(x_register, y_register, n) => {
    let x_offset = registers.read(&x_register);
    let y_offset = registers.read(&y_register);

    for y in 0..n {
        let line = memory.read(registers.i + y as u16);
        for x in 0..8 {
            if (line & (0x80 >> x)) != 0 {
                // NOTE: Not handling wrapping
                let x = x_offset + x;
                let y = y_offset + y;

                // NOTE: Not handling collision detection
                let l = y as usize * WIDTH + x as usize;
                assert!(l < WIDTH * HEIGHT);
                frame_buffer[l] = !frame_buffer[l];
            }
        }
    }
}
```

Firstly, we've added a few new methods, `Registers::read()`, and
`Memory::read()`. We've also introduced `frame_buffer` which acts as the video
memory.

The `frame_buffer` should be initialized before the main loop:

```rust
const WIDTH: usize = 64;
const HEIGHT: usize = 32;

let mut frame_buffer = vec![false; WIDTH * HEIGHT];
```

We also need to pass `frame_buffer` and `memory` into our `execute` function as
mutable references.

Now let's walk through what's happening in our new execute arm. The
`x_register` and `y_register` represent the x and y offset of our sprite. The
`n` nibble represents the height of the sprite.

We have an outer loop from 0 to the height of the sprite, inside the outer loop
we draw one line at a time to the frame buffer.

We do that by reading a byte from memory at register `I`, the byte represents a
line of the sprite where each bit represents a pixel being on or off.

We loop through the 8 bits and check each bit individually by shifting and
masking. If we find something non-zero we toggle the pixel in the frame buffer.

### **`ADD Vx, byte`**
<a id="add-instruction"></a>

This instruction is quite similar to the ones we've already implemented, again
I'm going to leave this as an exercise for the reader.

After implementing the `ADD` instruction you should hit the `JP` instruction
(`0x1`) in your `decode` function, rather than decoding it, we're actually just
going to exit our loop just before this point and render our image:

```rust
loop {
    // SNIP

    if pc == PROGRAM_OFFSET + 0x28 {
        break;
    }
}
```

## Rendering the frame buffer
<a id="rendering-the-frame-buffer"></a>

PBM is an extremely simple plain text 1-bit image format. The
[Wikipedia page for PBM][pbm] has a detailed description of how it works.

We're going to implement a very simple render method that gets called once our
loop is finished.

```rust
fn render(frame_buffer: &[bool]) {
    print!("P1\n{} {}", WIDTH, HEIGHT);
    let mut i = 0;
    for pixel in frame_buffer {
        if i % WIDTH == 0 {
            println!();
        }
        if *pixel {
            print!("1 ")
        } else {
            print!("0 ")
        }
        i += 1;
    }
    println!();
}
```

First we print out the PBM header:

```
P1
64 32
```

Then we loop over the entire frame buffer and print either a `1` or a `0`
depending on if the pixel is `true` or `false`. We also print a line break
whenever we reach the end of a line (`WIDTH`).

And call it after our loop:

```rust
loop {
    // SNIP
}

render(&frame_buffer);
```

Running your program should print a bunch of `1`s and `0`s to your terminal.
Because we've added the line breaks you can probably make out the IBM logo.

If you redirect the output into a file you should be able to open the image in
most image viewing programs.

```
$ cargo run > ibm-logo.pbm
```

Here's my output file in Preview.app:

<p align="center">
  <img src="/assets/images/posts/chip-8-ibm/screenshot.png" width="588" class="banner" />
</p>

Here's [the finished project][chip-8-ibm] on my GitHub.

## What next?
<a id="what-next"></a>

The real rest of the owl.

- Keyboard input
- Stack for the `CALL` instruction
- Delay and sound timers

Here's [my full CHIP-8 interpreter][chip-8] on my GitHub.

[endianness]: https://en.wikipedia.org/wiki/Endianness
[technical-reference]: http://devernay.free.fr/hacks/chip8/C8TECH10.HTM
[pbm]: https://en.wikipedia.org/wiki/Netpbm_format#PBM_example
[chip-8-ibm]: https://github.com/twe4ked/chip-8-ibm
[chip-8]: https://github.com/twe4ked/chip-8
