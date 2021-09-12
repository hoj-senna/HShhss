
Up to now, each colour is either on or off. Can we have it "half-on"? For example, if we want to have orange light (red twice as bright as green), is
that possible? 

Well, we can switch the green LED quickly on and off, such that it is on only half the time. If we're fast enough, this will look like green being
switched on with half the possible brightness. (This idea is the basis of "Pulse Width Modulation", PWM.) 

However, we will not use `delay_ms` for this purpose. 

Instead, we will rely on the Arduino's timers, 

(The tutorial code wants me to use an `AnalogWrite` function, which `avr_hal` does not provide. And the `Pwr` functionality seems to be undergoing a
rewrite. Fortunately, there are some instructions [here](https://github.com/Rahix/avr-hal/issues/194).)

Let's start with the green light. It is connected to Pin 5, and from https://content.arduino.cc/assets/Pinout-Mega2560rev3_latest.pdf we can see that
register OC3A has something to do with this pin. And from the number we guess that we need timer no. 3. 

What do we want to do? We want to increase the timer (which should be done automatically by the hardware), but not with the same frequency the
processor works at. Instead we introduce a prescaler, a factor saying: only increase every n-th CPU cycle. And that decides the frequency with which
the counter runs through all values from 0 to 255. And if the counter reaches a certain value, we want to set the output value to 0 (clear on match).
This means, the output is on for all smaller and off for all larger values of the counter.
```rust
    let tc=dp.TC3;
    tc.tccr3a.write(|w| w.wgm3().bits(0b01).com3a().match_clear());
    tc.tccr3b.write(|w| w.wgm3().bits(0b01).cs3().prescale_64());
```
Here tccr is the "timer counter control register". (And while I have found the precise names "tccr3a" etc. by relying on some autocomplete, one can
also look them up [here](https://docs.rs/avr-device/0.3.1/avr_device/atmega2560/index.html).)

We still have to set with which value we want to compare the counter. This is the number we write to the output compare register 3A: 
```rust
            tc.ocr3a.write(|w| unsafe{w.bits(duty as u16)});
```
where `duty` is the value to compare with, and would be the value to insert in the `AnalogWrite` function. Testing different values after each other
(making the green light brighter and brighter) could look like this: 

```rust
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let tc=dp.TC3;
    tc.tccr3a.write(|w| w.wgm3().bits(0b01).com3a().match_clear());
    tc.tccr3b.write(|w| w.wgm3().bits(0b01).cs3().prescale_64());

    let mut red = pins.d6.into_output();
    let mut green=pins.d5.into_output();
    let mut blue=pins.d3.into_output();

    loop {
        for duty in 0u8..=255u8{
            tc.ocr3a.write(|w| unsafe{w.bits(duty as u16)});
            arduino_hal::delay_ms(10);
        }
    }
}
```

If we want to have, say, the red light in addition, we have to look up the register for the timer of pin 6, and modify the program slightly, e.g.:
```rust
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let tc3=dp.TC3;
    tc3.tccr3a.write(|w| w.wgm3().bits(0b01).com3a().match_clear());
    tc3.tccr3b.write(|w| w.wgm3().bits(0b01).cs3().prescale_64());

    let tc4=dp.TC4;
    tc4.tccr4a.write(|w| w.wgm4().bits(0b01).com4a().match_clear());
    tc4.tccr4b.write(|w| w.wgm4().bits(0b01).cs4().prescale_64());


    let mut red = pins.d6.into_output();
    let mut green=pins.d5.into_output();
    let mut blue=pins.d3.into_output();

    tc4.ocr4a.write(|w| unsafe{w.bits(100 as u16)});

    loop {
        for duty in 0u8..=255u8{
            tc3.ocr3a.write(|w| unsafe{w.bits(duty as u16)});
            arduino_hal::delay_ms(10);
        }
    }
}
```


#
[Continue](005arduino5.md)
