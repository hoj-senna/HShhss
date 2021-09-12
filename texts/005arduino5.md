We have used the Arduino to control output in some form (blinking LEDs), but it's time to receive some input.
For this, we declare one of the pins to be an input pin, which is "pulled up", i.e.: if nothing happens, it should have a value of "high voltage". But
if we short-circuit it to ground, we can read this as "this pin now has a low value". 
```rust 
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let mut blue=pins.d2.into_output();
    let button=pins.d8.into_pull_up_input();
```
As you can see, we also keep the blue light. On the board, I connect a push button with pin d8 such that d8 is connected to GND as long as the button
is pushed.

In the program, we should, of course, include the loop:
```rust 
    loop {
        if button.is_low(){
            blue.set_high();
        }
        if button.is_high(){
            blue.set_low();
        }
    }
}
```
Here we switch the LED on when the button is pressed, and off while it's not pressed. (And yes, obviously the same behaviour can be achieved entirely
without a microprocessor or programming. But that's not the point.) 

