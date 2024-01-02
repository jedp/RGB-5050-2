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

The RGB5050 encodes 0 and 1 values as high/low transitions like so:

```
0 -> 0.3µs high ± 0.15µs, 0.9µs low ± 0.15µs
1 -> 0.6µs high ± 0.15µs, 0.6µs low ± 0.15µs
```

My first attempt was to try a transfer rate of 2.5Mbps, since this would
require the minimum number of encoded output bits (0 -> 0b100, 1 -> 0b110).
This is obviously out of spec for the timings. It was worth a go, bit it didn't
work reliably.

Bumping the transfer rate to 5Mbps doubles the number of bits required, but
has the benefit of actually working:

```
0 -> 0b100000 (0.2µs high, 1.0µs low)
1 -> 0b111000 (0.6µs high, 0.6µs low)
```

To implement this, one could iterate over each source byte of and do some fancy
bit arithmetic for every single byte (see previous commit), or one could follow
the advice of the mighty Ross Glashan and just use a lookup table, since there
are only 16 possible encodings of each nibble.

For example:

```
static uint32_t color_nibbles[16] = {
		0x820820, // 0b0000 -> 0b100000100000100000100000
		0x820838, // 0b0001 -> 0b100000100000100000111000
		0x820e20, // 0b0010 -> 0b100000100000111000100000
		0x820e38, // etc.
		0x838820,
		0x838838,
		0x838e20,
		0x838e38,
		0xe20820,
		0xe20838,
		0xe20e20,
		0xe20e38,
		0xe38820,
		0xe38838,
		0xe38e20,
		0xe38e38, // 0b1111 -> 0b111000111000111000111000
};

/*
 * Convert the pixel represented by (r, g, b), using the scheme described
 * at the head of this file, and pack it into buf in RGB 5050 order:
 * G, R, B; MSB first.
 */
void RGB5050_pixel_write(uint8_t r, uint8_t g, uint8_t b, uint8_t *buf) {
	// For each nibble of r, g, b, get the 24-bit output encoding of that nibble.
	uint32_t r_high = color_nibbles[r >> 4];
	uint32_t r_low  = color_nibbles[r & 0xf];
	uint32_t g_high = color_nibbles[g >> 4];
	uint32_t g_low  = color_nibbles[g & 0xf];
	uint32_t b_high = color_nibbles[b >> 4];
	uint32_t b_low  = color_nibbles[b & 0xf];

	buf[0]  = (g_high >> 16) & 0xff;
	buf[1]  = (g_high >>  8) & 0xff;
	buf[2]  = (g_high)       & 0xff;
	buf[3]  = (g_low  >> 16) & 0xff;
	buf[4]  = (g_low  >>  8) & 0xff;
	buf[5]  = (g_low)        & 0xff;

	buf[6]  = (r_high >> 16) & 0xff;
	buf[7]  = (r_high >>  8) & 0xff;
	buf[8]  = (r_high)       & 0xff;
	buf[9]  = (r_low  >> 16) & 0xff;
	buf[10] = (r_low  >>  8) & 0xff;
	buf[11] = (r_low)        & 0xff;

	buf[12] = (b_high >> 16) & 0xff;
	buf[13] = (b_high >>  8) & 0xff;
	buf[14] = (b_high)       & 0xff;
	buf[15] = (b_low  >> 16) & 0xff;
	buf[16] = (b_low  >>  8) & 0xff;
	buf[17] = (b_low)        & 0xff;
}
```


So you could set a region of a buffer like this:
```
RGB5050_pixel_write(0xAB, 0xCD, 0xEF, (uint8_t*) (some_buffer + 18 * offset));
```

And then transmit that buffer over SPI:
```
HAL_SPI_Transmit(&hspi1, (uint8_t*) some_buffer, sizeof(some_buffer), 1);
```

After transmission, leave a reset cycle of at least 80µs before transmitting again.

