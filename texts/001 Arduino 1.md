I have bought "The Most Complete Starter Kit MEGA 2560 Project" by elegoo (elegoo.com), which contains a MEGA 2560 rev3 Controller Board and quite a bit of
other electronic components. (Actually, this was some time ago. But seeing that I have a day of free time, today may be a good day for finally
unpacking it.)

Let's see if I can get it running with Rust (my currently favourite programming language).


I unpack the board and connect it with the USB cable (which was part of the starter kit) and connect that to some USB port on my PC. 

A green LED (labeled "on") lights up and an orange LED starts blinking with 1 Hz. Getting an LED blinking is supposed to be the "Hello World" of
Arduino programming; and this program seems to be pre-installed. 

Concerning "installing": Let's run "sudo apt-get install arduino" so that compiler and utilities (like a program to upload the programs to the board)
are installed. 
Running "arduino", the program greets me with "You need to be added to the 'dialout' group to upload code to an Arduino microcontroller over the USB
or serial ports." Well, sounds like I should click "Add" - and log off and on. 

"java.lang.UnsatisfiedLinkError: /usr/lib/x86_64-linux-gnu/liblistSerialsj.so.1.4.0: /usr/lib/x86_64-linux-gnu/liblistSerialsj.so.1.4.0: undefined
symbol: sp_get_port_usb_vid_pid" 

Perhaps it would have been a better idea to install the IDE from Arduino's webpage; but apparently, this is a known bug and 
https://bugs.launchpad.net/ubuntu/+source/arduino/+bug/1916278 suggests a workaround: 
```
sudo apt install libserialport0 patchelf
sudo patchelf --add-needed /usr/lib/x86_64-linux-gnu/libserialport.so.0 /usr/lib/x86_64-linux-gnu/liblistSerialsj.so.1.4.0
```

Now we can start this IDE and set Board Type and Processing Unit under Tools. (By the way, I'm sorry if I'm calling menus and options by slightly
inaccurate names; I'm using a non-English version and translating them on-the-fly.) We also set Port to "/dev/ttyACM0 (Arduino MEGA or MEGA2560)".

Under File->Examples->Basic->Blink we find code for the blinking example. Let's change some numbers (1000 to 300) so that we can see a change if
everything has worked. 

And then we hit "Upload". 
And ... it works, the LED blinks faster. 

Okay. But I wanted to do this in Rust, which means I need some replacement for this Arduino IDE. (Still, the installation in the previous paragraph is
not unnecessary: We have to have a compiler for this architecture installed somewhere, for example ... - and also the addition tho the dialout group
should not be skipped.)

```
cargo install cargo-generate
cargo install ravedude

cargo generate --git https://github.com/Rahix/avr-hal-template.git
```
This asks for a project name (let's say "first-arduino-project") and which board I use (giving a few options).

Okay, let's head to the new folder and call `cargo run`.

... and the LED is back to blinking once per second. This was surprisingly uncomplicated. (I had feared a much more complicated setup.) Thanks, Rahix.

Some stuff has happened in Cargo.toml, which I'm not going to look at now. Somewhere there are also instructions hidden that say "upload everything to
the board when I call `cargo run`" (in the .cargo/config.toml file). But let us have a look at the actual code: 

```rust
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);
    let mut led = pins.d13.into_output();

    loop {
        arduino_hal::delay_ms(1000);
        led.toggle();
    }
}
```

There is not much memory, there is no operating system around: The environment in which our code is supposed to be running is quite different from
that on my PC. Accordingly, we tell the compiler that it should not rely on the usual `std` crate, but only on a smaller subset of it (without file
system, threads, processes and, most importantly, heap allocations): 
```rust
#![no_std]
```
Also: No panic handling. That means we have to specify what happens when a panic occurs — or have some other crate do that for us. Enter `panic_halt`. 

And, on a related note, the `main()` function is different than usual. (Assumptions on command line arguments and possibly other assumptions on the
environment the program runs in.) 

So, we do not use the usual `main()` function, which we announce by `#![no_main]`. Instead, we have to designate one of our functions as entry point
of the program (by the annotation `#[arduino_hal::entry]`). That's the function `main()` — but we could give it a totally different name if we wanted
to. By the way, this function is a function that never returns (` -> !`).

In the function we first take care of some setup and `take` the peripherals, with our main interest lying on the pins. The struct containing all pins
is instanciated by the convenient macro `arduino_hal::pins!()`. 

One of the pins, pin d13, is connected to the LED on the board. This pin is what we want to use, and actually, we want to use it for signal output.
This is the final step of the setup: 
```rust
    let mut led = pins.d13.into_output();
```

In the `loop` which then runs repeatedly without end, we wait and switch the state of pin d13 from low to high voltage and back. 






### Links: 

[Embeddonomicon](https://docs.rust-embedded.org/embedonomicon/preface.html)

[Very helpful introduction](https://creativcoder.dev/rust-on-arduino-uno/)

[avr-hal repository](https://github.com/Rahix/avr-hal), also with links to the template used
