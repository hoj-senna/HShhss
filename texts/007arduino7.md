I want to use a passive buzzer to play a song. (More precisely: Happy birthday, due to an easily guessable occasion.) 

Relevant part of the wiring: pin d11 to buzzer to GND.

Also, let's keep the button from the previous chapter. 

```rust 
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let button=pins.d8.into_pull_up_input();
    let mut buzzer=pins.d11.into_output();


    loop {
        if button.is_low(){
            buzzer.set_high();
        }
    }
}
```
Of course, setting the buzzer to `high` does not yield any sound. For that, we have to make it oscillate between high and low some hundred times a
second.
```rust
    loop {
        if button.is_low(){
            buzzer.set_high();
            arduino_hal::delay_ms(2);
            buzzer.set_low();
            arduino_hal::delay_ms(2);
        }
    }
```
Now, there is a sound (while I press the button). 

(Right now, it's late at night, and I don't want to wake the neighbours, so I add a resistance between buzzer and GND. Then the humming becomes a bit
quieter.)

Now, this is only one tone, and rather inflexible. We have to switch the pin on and off explicitly each time and need the program to run at sufficient
speed to never miss (or delay) one of these switchings. (That should not be a problem with this simple program, but in principle could be
problematic.) 

Can we again use timers? 

Yes, certainly. This time let's have a closer look. 

https://content.arduino.cc/assets/Pinout-Mega2560rev3_latest.pdf shows that d12 as "timer" is connected to "OC1A". (Here, OC stands for "Output
Compare" of the corresponding timer.) 

How does the board decide whether it should use our calls to `.set_high()` and `.set_low()` or this OC1A to decide whether there should be high or low
voltage at pin d11? 

Questions like this are answered by the
[datasheet](http://www.atmel.com/Images/Atmel-2549-8-bit-AVR-Microcontroller-ATmega640-1280-1281-2560-2561_datasheet.pdf). 

In section 17.8 we can read "The general I/O port function is overridden by the Output Compare (OCnx) from the Waveform Generator if either
of the COMnx1:0 bits are set." 

That means, we'll have to set the 0- and/or 1-bit of COM1A. But to what value? And, another open question: If "OC" is output compare, then what output
is compared to what value? 

Answers to the question what values we might need we can find in the tables on p.155 in the data sheet. It depends on the Waveform Generation Mode...

Okay, maybe we should start from the other direction. What do we want? We want to toggle the voltage at pin d11 with a certain frequency. That means:
toggling OC1A with a certain frequency. In table 17-4 we find "toggle OC1A on Compare Match" (if WGM is 13, 14 or 15) if the COM bits are "01". 

We probably need 
```rust 
    let timer=dp.TC1;
    timer.tccr1a.write(|w| w.com1a().bits(0b01));
```

Also, this was in the table for "Fast PWM Mode" (about which we can read in Section 17.9.3): "The counter counts from BOTTOM to TOP
then restarts from BOTTOM." Okay. That means the difference between the values of TOP and BOTTOM(=0) decides how long it takes between toggles. 

This is something we want to set (so as to obtain different frequencies), so modes 14 and 15, where TOP is the value in ICR1 or OCR1A, respectively,
look good. The data sheet tells us that ICR1 is good for fixed values, OCR1A also for changes (because it's double buffered; and without double
buffering the current value of the counter could be higher than TOP when we change TOP, resulting in a situation where the values will never again
match). 

Accordingly, we want WGM 15. Or, in bits: 1111 (in order: WGM13, WGM12, WGM11, WGM10). These bits are hidden in two different registers (WGM11 and
WGM10 in TCCR1A, bits 3 and 2 in TCCR1B), hence: 
```rust 
    timer.tccr1a.write(|w| w.wgm1().bits(0b11));
    timer.tccr1b.write(|w| w.wgm1().bits(0b11));
```


And we want to set the right value for TOP in OCR1A.

Next question: What is "the right value"? How fast is the counter? 

The default system clock runs at 16 MHz (cf. datasheet, very first page). The counter is increased with every tick of this clock, or with every eighth,
64th, 256th or 1024th of its ticks, depending on a so-called prescaler. 

Let us set this prescaler. It goes in the TCCR1B register (see 17.11.5) in the CS bits: 
```rust 
    timer.tccr1b.write(|w| w.cs1().bits(0b001));
```

We should combine all of this writing to registers:
```rust
    timer.tccr1a.write(|w| w.com1a().bits(0b01).wgm1().bits(0b11));
    timer.tccr1b.write(|w| w.wgm1().bits(0b11).cs1().bits(0b001));
```

Now, what's the value for TOP? That depends on the note we want to play. For A (440 Hz), we need 880 toggles (that is, 440 on-off cycles) per second,
meaning 16000000/880 steps of the counter (which takes the TOP+1 values from 0 to TOP), i.e. TOP=18180.8; let's say 18181, because we want a `u16`. 

Frequencies for the other notes? Well, should be a factor of 12th root of 2 per half-tone. We can also look them up somewhere. 

```rust 
enum Note{
    G,
    A,
    B,
    C,
    D,
    E,
    F,
    Ghigh,
    Silence
}

fn timer_topvalue(note: Note)->u16{
    match note{
        Note::G => 20407,
        Note::A => 18181, 
        Note::B => 16197,
        Note::C => 15288,
        Note::D => 13620,
        Note::E => 12134,
        Note::F => 11453,
        Note::Ghigh => 10203,
        Note::Silence => 0
    }
}
``` 
(Of course, there are more notes. But those will be sufficient.) 

If we want to play them, we should specify their length. This we do by `delay_ms`. 

```rust 
fn play_note(timer:&arduino_hal::pac::TC1,note: Note,duration: Length){
    timer.ocr1a.write(|w| unsafe{w.bits(timer_topvalue(note))});
  match duration{
    Length::Eighth => {arduino_hal::delay_ms(250);},
    Length::Quarter => {arduino_hal::delay_ms(500);},
    Length::Half => {arduino_hal::delay_ms(1000);}
    Length::ThreeQuarters => {arduino_hal::delay_ms(1500);},
  };
    timer.ocr1a.write(|w| unsafe{w.bits(0)});
    arduino_hal::delay_ms(10);
}

enum Length{
    Eighth,
    Quarter,
    Half,
    ThreeQuarters,
}
``` 
(Also other lengths are not important.)

Note the small break at the end of `play_note`, so that we can have twice the same note sequentially without merely producing one longer note. 

And with that, let's have a look at the final program: 

```rust 
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

   let button=pins.d8.into_pull_up_input();
    let mut buzzer=pins.d11.into_output();

    let timer=dp.TC1;

    timer.tccr1a.write(|w| w.com1a().bits(0b01).wgm1().bits(0b11));
    timer.tccr1b.write(|w| w.wgm1().bits(0b11).cs1().bits(0b001));

    loop {
        if button.is_low(){
            play_note(&timer,Note::G,Length::Eighth);
            play_note(&timer,Note::G,Length::Eighth);
            play_note(&timer,Note::A,Length::Quarter);
            play_note(&timer,Note::G,Length::Quarter);
            play_note(&timer,Note::C,Length::Quarter);
            play_note(&timer,Note::B,Length::Half);

            play_note(&timer,Note::G,Length::Eighth);
            play_note(&timer,Note::G,Length::Eighth);
            play_note(&timer,Note::A,Length::Quarter);
            play_note(&timer,Note::G,Length::Quarter);
            play_note(&timer,Note::D,Length::Quarter);
            play_note(&timer,Note::C,Length::Half);

            play_note(&timer,Note::G,Length::Eighth);
            play_note(&timer,Note::G,Length::Eighth);
            play_note(&timer,Note::Ghigh,Length::Quarter);
            play_note(&timer,Note::E,Length::Quarter);
            play_note(&timer,Note::C,Length::Quarter);
            play_note(&timer,Note::B,Length::Quarter);
            play_note(&timer,Note::A,Length::Quarter);

            play_note(&timer,Note::F,Length::Eighth);
            play_note(&timer,Note::F,Length::Eighth);
            play_note(&timer,Note::E,Length::Quarter);
            play_note(&timer,Note::C,Length::Quarter);
            play_note(&timer,Note::D,Length::Quarter);
            play_note(&timer,Note::C,Length::ThreeQuarters);
        }
    }
}

fn play_note(timer:&arduino_hal::pac::TC1,note: Note,duration: Length){
    timer.ocr1a.write(|w| unsafe{w.bits(timer_topvalue(note))});
  match duration{
    Length::Eighth => {arduino_hal::delay_ms(250);},
    Length::Quarter => {arduino_hal::delay_ms(500);},
    Length::Half => {arduino_hal::delay_ms(1000);}
    Length::ThreeQuarters => {arduino_hal::delay_ms(1500);},
  };
    timer.ocr1a.write(|w| unsafe{w.bits(0)});
    arduino_hal::delay_ms(10);
}

enum Length{
    Eighth,
    Quarter,
    Half,
    ThreeQuarters,
}

enum Note{
    G,
    A,
    B,
    C,
    D,
    E,
    F,
    Ghigh,
    Silence
}

fn timer_topvalue(note: Note)->u16{
    match note{
        Note::G => 20407,
        Note::A => 18181, 
        Note::B => 16197,
        Note::C => 15288,
        Note::D => 13620,
        Note::E => 12134,
        Note::F => 11453,
        Note::Ghigh => 10203,
        Note::Silence => 0
    }
}
```
