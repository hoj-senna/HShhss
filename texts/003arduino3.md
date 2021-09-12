The previous section has shown how nice it would be to have some feedback like debug!/println! and not only blinking LEDs. Let us add



```rust 
    let mut serial = arduino_hal::default_serial!(dp, pins, 57600);
    ufmt::uwriteln!(&mut serial, "Hello from Arduino!\r").void_unwrap();
``` 
together with the 
```rust 
use arduino_hal::prelude::*;
```
that requires.


Let us insert another message in the loop of our previous program: 
```rust 
#![no_std]
#![no_main]

use arduino_hal::prelude::*;
use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let mut red = pins.d6.into_output();
    let mut green=pins.d5.into_output();
    let mut blue=pins.d3.into_output();

    let mut time : i8=1;

    let mut serial = arduino_hal::default_serial!(dp, pins, 57600);
    ufmt::uwriteln!(&mut serial, "Hello from Arduino!\r").void_unwrap();

    loop {
        if time%10==0{
            red.toggle();
        }
        if time%20==0{
            blue.toggle();
        }
        if time%30==0{
            green.toggle();
        }
        ufmt::uwriteln!(&mut serial, "time:{}\r", time).void_unwrap();

        arduino_hal::delay_ms(100);
        time+=1;
    }
}
```
Now we can observe that the numbers increase until 127 and then no more messages are sent. Why? Probably the program panics (and thus: aborts), as
soon as we add another 1 to time=127. 

Better use something that does not panic, for example: 
```rust 
        time=time.wrapping_add(1);
```
(or just the manual reset at 120 suggested in the previous section).


I still don't know what exactly is wrong with `i32`. Probably `%` does not work as intended. 
While the remainder modulo 2 seems to be okay, this: 
```rust
    for i in 0..100_i32{
        ufmt::uwriteln!(&mut serial, "{}%3 = {}\r", i, i%3).void_unwrap();
    }
```
— i.e. the remainder mod 3 — produces 
```
0%3 = 0
1%3 = 0
2%3 = 0
3%3 = 1
4%3 = 1
5%3 = 1
6%3 = 1
7%3 = 1
8%3 = 1
9%3 = 1
10%3 = 1
11%3 = 1
etc.
```
(and for `%10`, the same happens (with ones starting at 10)). 

Very well: Note to self: Don't use i32.


#
[Continue](004arduino4.md)
