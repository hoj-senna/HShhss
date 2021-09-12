
For the second program, I'd like to light an RGB LED. Each of the pins d3, d5, d6 is connected to (an 220 Ω resistance, which then is connected to)
one of the anodes, whereas one of the GND pins is connected to the cathode. 

What do we have to change in our program? 

Well, we need different pins. Okay: 


```rust 
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let mut red = pins.d6.into_output();
    let mut green=pins.d5.into_output();
    let mut blue=pins.d3.into_output();
```

and in the loop we should toggle all LEDs (or, LED components): 

```rust 
    loop {
        arduino_hal::delay_ms(500);
        red.toggle();
        arduino_hal::delay_ms(500);
        blue.toggle();
        arduino_hal::delay_ms(500);
        green.toggle();
    }
}
```

Very well. Next, let's try to have the colours blink with different frequencies. Let's say: 


``` 
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let mut red = pins.d6.into_output();
    let mut green=pins.d5.into_output();
    let mut blue=pins.d3.into_output();

    let mut time=1;

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

        arduino_hal::delay_ms(100);
        time+=1;
    }
}
``` 

Oh. 

```
error: linking with `avr-gcc` failed: exit code: 1
```
linking fails due to some `undefined reference to core::panicking::panic in compiler_builtins.../specialized_div_rem/delegate.rs:85`. 

Following 
https://github.com/rust-lang/compiler-builtins/issues/347, we include 
```
[profile.dev.package.compiler_builtins]
overflow-checks = false
``` 
in `Cargo.toml`. 

We receive the message 
```
warning: profile package spec `compiler_builtins` in profile `dev` did not match any packages
```
but the linking error did disappear.

Still: What happens is a bit unexpected. From the code, I would have expected that after (almost) one second the LED turns red; another second later
it becomes blue, then white, green, yellow, blue, purple, off, yellow, cyan, white. 
However, what I actually observe is very fast blinking in white, four or five times, then a longer white-red-white-red-white-red,
white-white-white-purple, switching between white and purple and finally white light (non-blinking). 

The following small change causes the expected behaviour: 
```
    let mut time : i16=1;
```

If we use `i8` instead, the LED remains switched off after 12 seconds. (That fits with `time` reaching its maximal value 127.) Of course, we can
combine that with something like 
```
if time==120{
            time=0;
        }
```
which leads to a nice, repeating pattern. 


#
[Continue](003arduino3.md)
