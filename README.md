# Yet Another RGB5050 Mount

A chainable board for mounting addressable [RGB 5050
LED](https://www.digikey.com/en/products/detail/inolux/IN-PI554FCH/7604874)s to things.

![PCB Image](3d-view.png)

## BOM

- [5050
  LED](https://www.digikey.com/en/products/detail/inolux/IN-PI554FCH/7604874)
- 0.1μF 0805 ceramic capacitor

## Bit-banging

These high-speed LEDs deal in ones and zeros at microsecond speed. One way to get this
kind of speed out of an STM32 without tying up the CPU and/or writing a lot of custom
assembly is to use SPI to transmit the data, possibly using DMA to send the data to SPI.

For example, this function packs 24-bit RGB data into a buffer that can be sent over
SPI at 5Mbps on an STM32F042:

```
/*
 * Rather non-portable. Packs RGB5050 data into strings of six bits
 * intended for sending over SPI at 5Mbps. That should nevertheless
 * be easy to configure on almost any STM32.
 *
 * (I tried a transfer rate of 2.5Mbps and three bits per zero or one,
 * but that falls just outside of the published tolerances and indeed
 * it did not work reliably. Hency my use of 6 bits at 5Mbps.)
 *
 * With a SPI transfer rate of 5Mbps, we can encode 0 and 1 as follows:
 *
 * 0 -> 0b100000 (0.2µs high, 1.0µs low)
 * 1 -> 0b111000 (0.6µs high, 0.6µs low)
 *
 * This is within the tolerance of the specification:
 *
 * 0 -> 0.3µs high ± 0.15µs, 0.9µs low ± 0.15µs
 * 1 -> 0.6µs high ± 0.15µs, 0.6µs low ± 0.15µs
 */
void RGB5050_color_write(int r, int g, int b, uint8_t *buf) {
  // For the sake of having normal function parameters, the bits end
  // up packed backwards here. We will reverse them in the loop below.
  int rgb_data = ((g & 0xff) << 16) | ((r & 0xff) << 8) | (b & 0xff);

  // RGB 5050 order: G, R, B; MSB first.
  int offset = 0;
  for (int i = 0; i < 24; i++) {
    switch (i % 4) {
    case 0:
      // current[5:0] | next[5:4]
      buf[offset] |= ((rgb_data & (1<<(23-i))) ? 0b11100000 : 0b10000000)
                  |  ((rgb_data & (1<<(22-i))) ?       0b11 :       0b10);
      break;
    case 1:
      // current[3:0] | next[5:2]
      buf[offset] |= ((rgb_data & (1<<(23-i))) ? 0b10000000 :          0)
                  |  ((rgb_data & (1<<(22-i))) ?     0b1110 :     0b1000);
      break;
    case 2:
      // current[1:0] | next[5:0]
      buf[offset] |= ((rgb_data & (1<<(22-i))) ? 0b00111000 : 0b00100000);
      // Lookahead read next bit, so advance i.
      i += 1;
      break;
    }

    offset += 1;
  }
}
```

So you could set a region of a buffer like this:
```
RGB5050_color_write(0xAB, 0xCD, 0xEF, (uint8_t*) (some_buffer + 18 * offset));
```

And then transmit that buffer over SPI:
```
HAL_SPI_Transmit(&hspi1, (uint8_t*) some_buffer, sizeof(some_buffer), 1);
```

After transmission, leave a reset cycle of at least 80µs before transmitting again.

