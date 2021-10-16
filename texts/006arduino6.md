## Data transmission, again
Earlier, we have received some data sent by the Arduino by writing via `ufmt::uwriteln!`. It was shown directly in the terminal running `cargo run`. 

But wouldn't it be nice to receive signals sent by the board in a separate program? Like, say, using the board as a fancy new input device for our
great single-developer-triple-A-MMORPG? (Or, more realistically, use the readout of some thermometer or other sensor connected to the Arduino as part
of a nice, practical GUI weather app or something?) 

Well, I have not yet progressed to thermometers, so we'll stick with a button. And a GUI in Rust ... well, that's probably another post (series) of its own.

But transmitting some data ... that should not be too far off from where we stand with the previous programs. 

We keep a button such that pin d8 is connected to GND while said button is pressed. In the program, we increase a counter while the button is pressed.
And we send the value of the counter if there has been an increase: 

```rust 
#![no_std]
#![no_main]

use panic_halt as _;

#[arduino_hal::entry]
fn main() -> ! {
    let dp = arduino_hal::Peripherals::take().unwrap();
    let pins = arduino_hal::pins!(dp);

    let button=pins.d8.into_pull_up_input();

    let mut serial = arduino_hal::default_serial!(dp, pins, 57_600);
    let mut counter:u8=1;

    loop {
        if button.is_low(){
            counter=counter.wrapping_add(1);
            serial.write_byte(counter);
        }
    }
}
``` 
Two lines of that are connected to the transmission: 
```rust
    let mut serial = arduino_hal::default_serial!(dp, pins, 57_600);
```
— we have used this one before; the number 57 600 is the baud rate: how many symbols are sent per second — and 
```rust
            serial.write_byte(counter);
```
which seems rather self-explanatory. 

On the other side of the USB cable, we need a program receiving these bytes. New project and
```
cargo add serialport
``` 
(By the way, [cargo-edit](https://github.com/killercup/cargo-edit) is great.) 
```rust 
fn main() {
    let ports = serialport::available_ports().expect("No ports found!");
    for p in ports {
        println!("{}", p.port_name);
    }
    let mut port = serialport::new("/dev/ttyACM0", 57_600)
        .timeout(core::time::Duration::from_millis(10))
        .open()
        .expect("Failed to open port");
    loop {
        let mut serial_buf: Vec<u8> = vec![0; 4];
        if let Ok(bytes_read) = port.read(serial_buf.as_mut_slice()) {
            for b in &serial_buf[..bytes_read] {
                println!("{}", b);
            }
        }
    }
}
```
The first few lines (listing all ports) are unnecessaryfor the final program, but helpful during setup. The name of the port very probably is
different between operating systems (I don't care). 

It may be interesting to observe that we, once again, set the same baud rate.


When we let both programs run, we may notice that not all numbers are read by the PC-side program with `serialport`. That's because the terminal in
which `cargo run` was invoked still reads from the same port. If we close this terminal, only the other program remains and the numbers increase (up
to 255, before starting again), as expected.


#
[Continue](007arduino7.md)
