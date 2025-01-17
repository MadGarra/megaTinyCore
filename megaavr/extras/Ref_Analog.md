# ADC/DAC Reference for megaTinyCore
These parts all have a large number of analog inputs (11 pins on 20 and 24-pin 0/1-series, 15 on 2-series, 9 on 14-pin parts and 5 on 8-pin parts.) - plus the one on the UPDI pin which is not totally usable because the UPDI functionality). They can be read with analogRead() like on a normal AVR. While the `An` constants (ex, `A3`) are supported, and will read from the ADC channel of that number, they are in fact defined as then digital pin shared with that channel. Using the `An` constants is deprecated - the recommended practice is to just use the digital pin number, or better yet, use `PIN_Pxn` notation when calling analogRead(). Particularly with the release of 2.3.0 and tinyAVR 2-series support, a number of enhanced ADC features have been added to expose more of the power of the sophisticated ADC in these parts.

**Be sure to disable to the ADC prior to putting the part to sleep if low power consumption is required!**

## Reference Voltages
Analog reference voltage can be selected as usual using analogReference(). Supported reference voltages are listed below:

| tinyAVR 0/1-series                     | VDD<sub>min</sub> | tinyAVR 2-series, Dx, and EA            | VDD<sub>min</sub> |
|----------------------------------------|--------|-----------------------------------------|--------|
| `VDD` (Vcc/supply voltage - default)   | n/a    | `VDD` (Vcc/supply voltage - default)    | n/a    |
| `INTERNAL0V55`                         | 1.8V   | `INTERNAL1V024`                         | 1.8    |
| `INTERNAL1V1`                          | 1.8V   | `INTERNAL2V048`                         | 2.6    |
| `INTERNAL1V5`                          | 1.9V   | `INTERNAL4V096`                         | 4.6    |
| `INTERNAL2V5`                          | 2.9V   | `INTERNAL2V500`                         | 3.0    |
| `INTERNAL4V3` (alias of INTERNAL4V34)  | 4.75V  | `INTERNAL4V1` (alias of INTERNAL4V096)  | -      |
| `INTERNAL4V34`                         | -      | `EXTERNAL`                              | V<sub>ref</sub>+0.5V |
| `EXTERNAL` (16k and 32k 1-series only) | V<sub>ref</sub>+0.4V |                           |        |

### Important notes regarding reference voltages


1. We do not provide a reference named "INTERNAL" like some classic AVR cores do; because the available voltages vary, this would be a detriment to cross-compatibility - by generating code that would compile, but behave differently, that would introduce the potential for new bugs that would be difficult to debug. Especially since the internal reference voltage isn't the same one that classic AVRs where that usage was commonplace is.... and so these would behave wrongly no matter what was done; minor modifications to sketches are required whenever the internal references are used when porting from classic to modern AVRs.


2. As you can see in the table above, more conservative values were spec'ed for the 2-series. this could be because of increased sensitivity of the ADC to adverse operating conditions, or may reflect a generally more careful and deliberate approach taken with the 2-series (compare the length of the errata for the 2-series with the 1-series, for example).

3. The External Reference option on the 16k and 32k 1-series parts - like most of the other features specific to just those parts - was added is a somewhat slapdash manner. They say, and I quote:
```text
When using the external reference voltage VREFA, configure ADCnREFSEL[0:2] in the corresponding VREF.CTRLn
register to the value that is closest, but above the applied reference voltage. For external references higher than
4.3V, use ADCnREFSEL[0:2] = 0x3 [4.3V].
```
However, one must NOT have ADCnREFEN bit set when using an external reference, so it's far from clear why the above is required, this warning was never present until 2.5.12, and nobody complained. This suggests that it one of the following is true:
  a. Configuring refsel per the datasheet isn't that important after all, or makes small enough difference in accuracy that it has gone unnoticed so far.
  b. Failure to configure refsel per the datasheet causes only poor ADC accuracy, and the few individuals using an external reference read the relevant paragraph in the datasheet and configured refsel.
  b. Configuring refsel per the datasheet is a *safeguard* against some undesirable outcome that could occur if ADCnREFEN was also set with an external reference.
  c. Configuring refsel per the datasheet is a safeguard against some undesirable outcome that could occur whether or not ADCnREFEN is set, but the undesirable outcome impacts the internal references only, and the few people who have encountered this have not encountered the consequences because the don't use the internal reference source
  d. Nobody is using the external voltage reference on these things in the first place.

My guess (and it is ONLY A **GUESS**) is that b or c is the case, and the undesired outcome takes the form of damage to the selected analog reference. Which will usually be 0.55V. if all that was done was immediately set it to the external reference. That reference voltage is rarely used, so this could plausibly have gone unnoticed by megaTinyCore users.

analogReference() as you know takes only a single argument, and the overhead of measuring an external reference voltage to determine what the internal reference should be set to in order to comply with that restriction would be prohibitive. If you are using the external reference on the 1-series parts, first call analogReference with a internal reference per their quoted instructions, then set it to the external reference.

3. Note the change from the heinous near random reference voltages used on the 0/1-series - which you often end up either checking against thresholds calculated at compile time or using floating point math in order to do anything useful with - to maximally convenient ones on everything released later than the 0/1-series. I say they are maximally convenient because with 12-bit resolution selected, 4.096V, 2.048V, and 1.024V references correspond to 1 mV/LSB, 0.5mv/LSB, and 0.25mv/LSB.

4. On 0/1-series, you must be sure to slow down the ADC clock with analogClockSpeed when using the 0.55V reference, as it has a much more restricted operating clock speed range: 100 - 260 kHZ. The other references can be used at 200 - 2000 kHz.

## Internal Sources
In addition to reading from pins, you can read from a number of internal sources - this is done just like reading a pin, except the constant listed in the table below is used instead of the pin name or pin number:
| tinyAVR 0/1-series                     | tinyAVR 2-series                    | AVRDA            | AVRDB             | AVRDD             | AVR EA           |
|----------------------------------------|-------------------------------------|------------------|-------------------|-------------------|------------------|
| `ADC_INTREF`                           | `ADC_VDDDIV10`                      | `ADC_TEMPERATURE`| `ADC_VDDDIV10`    | `ADC_VDDDIV10`    | `ADC_VDDDIV10`   |
| `ADC_TEMPERATURE`                      | `ADC_TEMPERATURE`                   | `ADC_GROUND` *   | `ADC_TEMPERATURE` | `ADC_TEMPERATURE` | `ADC_TEMPERATURE`|
| `ADC_DAC0` (1-series only)             | `ADC_GROUND` (offset calibration?)  | `ADC_DACREF0` *  | `ADC_GROUND` *    | `ADC_GROUND`      | `ADC_GROUND`     |
| `ADC_GROUND` (offset calibration?)     | `ADC_DACREF0`                       | `ADC_DACREF1` *  | `ADC_DACREF0` *   | `ADC_DACREF0`     | `ADC_DACREF0`    |
| `ADC_DACREF0` (alias of ADC_DAC0)      | `ADC_DAC0` (alias of `ADC_DACREF0`) | `ADC_DACREF2` *  | `ADC_DACREF1` *   | `ADC_DAC0`        | `ADC_DACREF1`    |
| `ADC_PTC` (not particularly useful)    |                                     | `ADC_DAC0`       | `ADC_DACREF2` *   | `ADC_VDDIO2DIV10` | `ADC_DAC0`       |
| 'ADC1_DACREF1' (16/32k 1-series only)† |                                     |                  | `ADC_DAC0` *      | `ADC_BGTEMPSNSE`? |                  |
| 'ADC1_DACREF2' (16/32k 1-series only)† |                                     |                  | `ADC_VDDIO2DIV10` |                   |                  |


The Ground internal sources are presumably meant to help correct for offset error. On Classic AVRs they made a point of talking about offset cal for differential channels. They are thus far silent on this, though a great deal of tech briefs related to the new ADC has been coming out, greatly improving the quality of the available information.

DACREF0 is the the reference voltage for the analog comparator, AC0. On the 1-series, this is the same as the `DAC0` voltage (yup, the analog voltage is shared). Analog comparators AC1 and AC2 on 1-series parts with at least 16k of flash also have a DAC1 and DAC2 to generate a reference voltage for them (though it does not have an output buffer), . On the 2-series, this is only used as the AC0 reference voltage; it cannot be output to a pin. . Unlike the AVR DA-series parts, which have neither `VDDDIV10` nor `INTREF` as an option so reading the reference voltage through `DACREF0` was the only way to determine the operating voltage, there is not a clear use case for the ability to measure this voltage.

`*` Note that the I/O headers omit several options listed in the datasheet; For Dx-series, DACREF options aren't listed in the I/O headers for MUXPOS or Ground, but we still provide the constants listed above, consistent with the datasheet.
`?` The DD-series lists a "BGTEMPSENSE" option in the preliminary datasheet. It's supposed to be a better temperature sensor. It is not mentioned anywhere other than the muxpos table, and may end up being dropped from the final datasheet. This will be classified as an AVR Research Mystery if full information isn't available when silicon starts shipping, with the reward for someone who figures out and finds that it works being an DD-series breakout board, with chip mounted, of your choice (or a quantity of bare boards TBD). If you find, sadly, that it doesn't work, you'll still be eligible for a bounty of a smaller number of bare DD-breakout boards.
`†` This options is available only on ACD1 of the tinyAVR 1-series parts.

### Analog Resolution
The hardware supports increasing the resolution of analogRead() to the limit of the hardware's native resolution (10 or 12 bits); Additionally, we provide automatic oversampling and decimation up to the limit of what can be gathered using the accumulation feature allowing up to 13 bits of resolution (17 on 2-series); this is exposed through the `analogReadEnh()` function detailed below.

### A second ADC? *1-series only*
On some parts, yes! The "good" 1-series parts (1614, 1616, 1617, 3216, and 3217) have a second ADC, ADC1. On the 20 and 24-pin parts, these could be used to provide analogRead() on additional pins (it can also read from DAC2). Currently, there are no plans to implement this in the core due to the large number of currently available pins. Instead, it is recommended that users who wish to "take over" an ADC to use it's more advanced functionality choose ADC1 for this purpose. See [ADC free-run and more](ADCFreerunAndMore.md) for some examples of what one might do to take advantage of it (though it has not been updated for the 2-series yet. As of 2.0.5, megaTinyCore provides a function `init_ADC1()` which initializes ADC1 in the same way that ADC0 is (with correct prescaler for clock speed and VDD reference). On these parts, ADC1 can be used to read DAC1 (DACREF1) and DAC2 (DACREF2), these are channels `0x1B` and `0x1E` respectively. ~The core does not provide an equivalent to analogRead() for ADC1 at this time.~

#### Using ADC1 just like a second ADC0
This is a heinous waste of the second ADC. However it can be done - the following functions are are provided, which are identical to their normal counterparts, on devices with the second ADC:
* init_ADC1
* analogReference1
* analogClockSpeed1
* analogRead1
* analogSampleDuration1
* analogReadEnh1

If you are using this, you must call init_ADC1() before using any of them.

### ADC Sampling Speed
megaTinyCore takes advantage of the improvements in the ADC on the newer AVR parts to improve the speed of analogRead() by more than a factor of three over classic AVRs! The ADC clock which was - on the classic AVRs - constrained to the range 50-200kHz - can be cranked up as high as 1.5MHz at full resolution! On the 0/1-series, we use 1MHz at 16/8/4 MHz system clock, and 1.25 MHz at 20/10/5 MHz system clock. To compensate for the faster ADC clock, we set the sample duration to 14 to extend the sampling period from the default of 2 ADC clock cycles to 16 - providing approximately the same sampling period as on classic AVRs (which is probably a conservative decision). On the 2-series, the corresponding setting has only 0.5 or 1 ADC clock added to it, so we set it to 15 - however, it can be set all the way up to 255. The ADC clock is run at approx. 2.5 MHz on the 2-series parts.  If you are measuring a lower impedance signal and need even faster `analogRead()` performance - or if you are measuring a high-impedance signal and need a longer sampling time, you can adjust that setting with `analogSampleDuration()` detailed below.

## ADC Function Reference
This core includes the following ADC-related functions. Out of the box, analogRead() is intended to be directly compatible with the standard Arduino implementation. Additional functions are provided to use the advanced functionality of these parts and further tune the ADC to your application.

### getAnalogReference() and getDACReference()
These return the numbers listes in the reference table at the top as a `uint8_t` - you can for example test `if (getAnalogReference() == INTERNAL1V1)`

### analogRead(pin)
The standard analogRead(). Single-ended, and resolution set by analogReadResolution(), default 10 for compatibility. Negative return values indicate an error that we were not able to detect at compile time. Return type is a 16-bit signed integer (`int` or `int16_t`).

### analogReadResolution(resolution)
Sets resolution for the analogRead() function. Unlike stock version, this returns true/false. *If it returns false, the value passed was invalid, and resolution was set to the default, 10 bits*. Note that this can only happen when the value passed to it is determined at runtime - if you are passing a compile-time known constant which is invalid, we will issue a compile error. The only valid values are those that are supported natively by the hardware, plus 10 bit for compatibility where that is not natively supported (ie, the 2-series) for compatibility.

Hence, the only valid values are 8, 10 and 12.

This is different from the Zero/Due/etc implementations, which allow you to pass anything from 1-32, and rightshift the minimum resolution or leftshift the maximum resolution as needed to get the requested range of values. Within the limited resources of an 8-bit AVR with this is a silly waste of resources, and padding a reading with zeros so it matches the format of a more precise measuring method is in questionable territory in terms of best practices. Note also that we offer oversampling & decimation with `analogReadEnh()` and `analogReadDiff()`, which can extend the resolution while keep the extra bits meaningful at the cost of slowing down the reading. `analogReadResolution()` only controls the resolution of analogRead() itself. The other functions take desired resolution as an argument, and restore the previous setting.

### analogSampleDuration(duration)
Sets sampling duration (SAMPLEN), the sampling time will be 2 + SAMPLEN ADC clock cycles. Sampling time is the time the sample and hold capacitor is connected to the source before beginning conversion. For high impedance voltage sources, use a longer sample length. For faster reads, use a shorter sample length.
speed. This returns a `bool` - it is `true` if value is valid.

This value is used for all analog measurement functions.

| Part Series | Sample time<br/>(default)  | Conversion time | Total analogRead() time | Default SAMPLEN | ADC clock sample time |
|-------------|--------------|-----------------|-------------------------|-----------------|-----------------------|
| Classic AVR |        12 us |           92 us |                  104 us | No such feature | 1.5                   |
| 0/1-series  |        12 us |       8.8-10 us |              20.8-22 us |     7, 9, or 13 | 2 + SAMPLEN           |
| Dx 10-bit   |   apx. 12 us |  approx.  10 us | Approximately  21-22 us |              14 | 2 + SAMPLEN           |
| Dx 12-bit   |   apx. 12 us |  approx.  12 us | Approximately  23-24 us |              14 | 2 + SAMPLEN           |
| 2-series    |  apx. 6.4 us |  approx. 5.6 us | Approximately     12 us |              15 | 1 + SAMPDUR           |
| EA-series   |  apx. 6.4 us |  approx. 5.6 us | Approximately     12 us |              15 | 1 + SAMPDUR           |


On the 2-series, we are at least given some numbers: 8pF for the sample and hold cap, and 10k input resistance so a 10k source impedance would give 0.16us time constant, implying that even a 4 ADC clock sampling time is excessive, but at such clock speeds, impedance much above that would need a longer sampling period. You probab

### analogReadEnh(pin, res=ADC_NATIVE_RESOLUTION, gain=0)
Enhanced `analogRead()` - Perform a single-ended read on the specified pin. `res` is resolution in bits, which may range from 8 to `ADC_MAX_OVERSAMPLED_RESOLUTION`. This maximum is 13 bits for 0/1-series parts, and 17 for 2-series. If this is less than the native ADC resolution, that resolution is used, and then it is right-shifted 1, 2, or 3 times; if it is more than the native resolution, the accumulation option which will take 4<sup>n</sup> samples (where `n` is `res` native resolution) is selected. Note that maximum sample burst reads are not instantaneous, and in the most extreme cases can take milliseconds. Depending on the nature of the signal - or the realtime demands of your application - the time required for all those samples may limit the resolution that is acceptable. The accumulated result is then decimated (rightshifted n places) to yield a result with the requested resolution, which is returned. See [Atmel app note AVR121](https://ww1.microchip.com/downloads/en/appnotes/doc8003.pdf) - the specific case of the new ADC on the Ex and tinyAVR 2-series is discussed in the newer DS40002200 from Microchip, but it is a rather vapid document). Alternately, to get the raw accumulated ADC readings, pass one of the `ADC_ACC_n` constants for the second argument where `n` is a power of 2 up to 64 (0/1-series), or up to 1024 (2-series). Be aware that the lowest bits of a raw accumulated reading should not be trusted.; they're noise, not data (which is why the decimation step is needed, and why 4x the samples are required for every extra bit of resolution instead of 2x). On 2-series parts *the PGA can be used for single ended measurements*. Valid options for gain on the 2-series are 0 (PGA disabled, default), 1 (unity gain - may be appropriate under some circumstances, though I don't know what those are), or powers of 2 up to 16 (for 2x to 16x gain). On AVR Dx and tinyAVR 0/1-series parts, the gain argument should be omitted or 0; these do not have a PGA.

| Part Series | Max Acc.     | result trunc?   | Maximum acc+decimate | Max PGA gain  |
|-------------|--------------|-----------------|----------------------|---------------|
| 0/1-series  |   64 samples |  N/A            |             13 bits  | N/A - no PGA  |
| Dx-series   |  128 samples |  16 bits max    |             15 bits  | N/A - no PGA  |
| 2-series    | 1024 samples |  No             |             17 bits  | 0/1/2/4/8/16x |
| EA-series   | 1024 samples |  No             |             17 bits  | 0/1/2/4/8/16x |

**WARNING** trying to take 17-bit readings on a 2-series/EA-series will take a long time for each sample. If the sample is changing, the results will be garbage.
**WARNING** Extreme attention is required to make the least significant bits meaningful, especially when oversampling and decimating to 17 bits. This means very careful hardware design. Otherwise, you will be very accurately measuring the noise and error sources. I am praying that the EA-series will have AVDD on a separate power domain and invite us to filter it further to improve analog stability. In any event, quality external ADC reference is recommended when high accuracy is required.

```c++
  int32_t adc_reading = analogReadEnh(PIN_PD2, 13); //be sure to choose a pin with ADC support. The Dx-series and tinyAVR have ADC on very different sets of pins.
  // Measure voltage on PD2, to 13 bits of resolution.
  // AVR Dx, Ex, and tinyAVR 2-series this will be sampled 4 times using the accumulation function, and then rightshifted once
  // tinyAVR 0/1-series, this will be sampled 64 times, as we need 3 more bits, hence we need to take 2^(3*2) = 64 samples then rightshift them 3 times.,
  int32_t adc_reading2 = analogReadEnh(PIN_PD2, ADC_ACC128);
  // Take 128 samples and accumulate them. This value, k, is 19 bits wide; on Dx-series parts, this is truncated to 16 bits - the hardware does not expose the three LSBs. 0/1-series can only measure to 15
```

Negative values from ADC_ENH always indicate a runtime error; these values are easily recognized, as they are huge negative numbers

### analogReadDiff(positive, negative, res=ADC_NATIVE_RESOLUTION, gain=0) **2-series only**
Differential `analogRead()` - returns a `long` (`int32_t`), not an `int` (`int16_t`). Performs a differential read using the specified pins as the positive and negative inputs. Any analog input pin can be used for the positive side, but only inputs on PORTA, or the constants `ADC_GROUND` or `ADC_DAC0` can be used as the negative input. Information on available negative pins for the Ex-series is not yet available, but is expected to be a subset of available analog pins. The result returned is the voltage on the positive side, minus the voltage on the negative side, measured against the selected analog reference. The `res` parameter works the same way as for `analogReadEnh()`, as does the `gain` function. Gain becomes FAR more useful here than in single-ended mode as you can now take a very small difference and "magnify" it to make it easier to measure. Be careful when measuring very small values here, this is a "real" ADC not an "ideal" one, so there is a non-zero error, and through oversampling and/or gain, you can magnify that such that it looks like a signal.

```c++
  int32_t diff_first_second = analogReadDiff(PIN_PA1, PIN_PA2, 12, 0);
  // Simple differential read to with PA1 as positive and PA2 as negative. Presumable these are close in voltage.
  // (I used two pots and manually adjusted them to be very close; also could have done like 10k-100ohm-10k resistor network)
```

The 32-bit value returned should be between -65536 and 65535 at the extremes with the maximum 17-bit accumulation option, or, 32-times that if using raw accumulated values (-2.1 million to 2.1 million, approximately)

Error values are negative numbers close to -2.1 billion, so it should not be challenging to distinguish valid values from errors.

### analogClockSpeed(int16_t frequency = 0, uint8_t options = 0)
The accepted options for frequency are -1 (reset ADC clock to core default (depends on part)), 0 (make no changes - just report current frequency) or a frequency, in kHz, to set the ADC clock to. Values between 125 and 1500 are considered valid for 0/1-series parts, up to 2000 kHZ for Dx-series, and for EA-series and 2-series parts 300-3000 with internal reference, and 300-6000 with Vdd or external reference. The prescaler options are discrete, not continuous, so there are a limited number of possible settings (the fastest and slowest of which are often outside the rated operating range). The core will choose the highest frequency which is within spec, and which does not exceed the value you requested. If a 1 is passed as the second argument, the validity check will be bypassed; this allows you to operate the ADC out of spec if you really want to, which may have unpredictable results. Microchip documentation has provided little in the way of guidance on selecting this (or other ADC parameters) other than giving us the upper and lower bounds.

**Regardless of what you pass it, it will return the frequency in kHz** as a `uint16_t`.

The 0/1-series has prescalers in every power of two from 2 to 256, and at the extreme ends, typical operating frequencies will result in an ADC clock that is not in spec.

The 2-series has a much richer selection of prescalers: Every even number up to 16, then 20, 24, 28, 32, 40, 48, 56, and 64, giving considerably more precision in this adjustment.

In either case you don't need to know the range of prescalers, just ask for a frequency and we'll get as close as possible without exceeding it, and tell you what was picked

```c++
// ******* for both *******
Serial.println(analogClockSpeed());     // prints something between 1000 and 1500 depending on F_CPU
analogClockSpeed(300);                  // set to 300 kHz
Serial.println(analogClockSpeed());     // prints a number near 300 - the closest to 300 that was possible without going over.

// ****** For 0/1-series *******
Serial.println(analogClockSpeed(3000)); // sets prescaler such that frequency of ADC clock is as
// close to but not more than 2000 (kHz) which is the maximum supported according to the datasheet.
// Any number of 2000 or higher will get same results.
Serial.println(analogClockSpeed(20));   // A ridiculously slow ADC clock request.
// Datasheet says minimum is 125, maximum prescale(256) would give 93.75 kHz - but that's not
// within spec, and no second argument was passed; we will instead set to 128 prescaler
// for clock speed of 187.5 kHz and will return 187,
Serial.println(analogClockSpeed(20, 1)); // As above, but with the override of spec-check
// enabled, so it will set it as close to 20 as it can (93.75) and return 93.

// ****** For 2-series *******
Serial.println(analogClockSpeed(20000));   // Above manufacurer max spec, so seeks out a value that
// is no larger than that spec; 3000 if internal reference selected or 6000 otherwise.
Serial.println(analogClockSpeed(20000,1)); // Above manufacturer's spec. but we instruct
// it to ignore the spec and live dangerously. The requested speed, far in excess of the
// maximum obtainable (which is several times the maximum supported) speed, with limits bypassed,
// will lead it to choose the fastest possible ADC clock, which is only prescaled by a factor of 2,
// and return that value, likely 8000 or 10000 unless for 16 or 20 MHz. (higher if you're overclocking
// the chip in general too). One version that was released shortly after the 2-series contained a bug that would always set the
// ADC prescaler to the minimum. And I'd written about how disappointed I was in the new ADC. After fixing the bug, things improved significantly.
// I am favorably impressed by how well the poor overclocked ADC performed, despite being overclocked

// ******* For both *******
int returned_default = analogClockSpeed(-1); // reset to default value, around 1000 - 1500 kHz for Dx or 0/1-series tiny, and around 2500 for Ex or 2-series tiny.

Serial.println(returned_default);  // will print the same as the first line, assuming you hadn't changed it prior to these lines.
```
If anyone undertakes a study to determine the impact of different ADC clock frequency on accuracy, take care to adjust the sampling time to hold that constant. I would love to hear of any results; I imagine that lower clock speeds should be more accurate, but within the supported frequency range, I don't know whether these differences are worth caring about.
I've been told that application notes with some guidance on how to best configure the ADC for different jobs is coming. Microchip is aware that the new ADC has a bewildering number of knobs compared to classic AVRs, where there was typically only 1 degree of freedom, the reference, which is simple to pick and understand, since only one prescaler setting was in spec.

### getAnalogReadResolution()
Returns either 8, 10 or 12 (2-series only), the current resolution set for analogRead.

### getAnalogSampleDuration()
Returns the number of ADC clocks by which the minimum sample length has been extended.

### ADC Runtime errors
When taking an analog reading, you may receive a value near -2.1 billion - these are runtime error codes.
The busy and disabled errors are the only ones that we never know at compile time.
| Error name                     |     Value   | Notes
|--------------------------------|-------------|---------------------------------------------------------------------
|ADC_ERROR_INVALID_CLOCK         |      -32764 | Returned by analogSetClock() if, somehow, it fails to find an appropriate value. May be a cant-happen.
|ADC_ERROR_BAD_PIN_OR_CHANNEL    |      -32765 | The specified pin or ADC channel does not exist or does support analog reads.
|ADC_ERROR_BUSY                  |      -32766 | The ADC is busy with another conversion.
|ADC_ERROR_DISABLED              |      -32767 | The ADC is disabled at this time. Did you disable it before going to sleep and not re-enable it?
|ADC_ENH_ERROR_BAD_PIN_OR_CHANNEL| -2100000000 | The specified pin or ADC channel does not exist or does support analog reads.
|ADC_ENH_ERROR_BUSY              | -2100000001 | The ADC is busy with another conversion.
|ADC_ENH_ERROR_RES_TOO_LOW       | -2100000003 | Minimum ADC resolution is 8 bits. If you really want less, you can always rightshift it.
|ADC_ENH_ERROR_RES_TOO_HIGH      | -2100000004 | Maximum resolution, using automatic oversampling and decimation is less than the requested resolution.
|ADC_DIFF_ERROR_BAD_NEG_PIN      | -2100000005 | analogReadDiff() was called with a negative input that is not valid.
|ADC_ENH_ERROR_DISABLED          | -2100000007 | The ADC is currently disabled. You must enable it to take measurements. Did you disable it before going to sleep and not re-enable it?

### printADCRuntimeError(uint32_t error, &UartClass DebugSerial)
Pass one of the above runtime errors and the name of a serial port to get a human-readable error message. This is wasteful of space, do don't include it in your code unless you need to.
```c++
int32_t adc_reading=AnalogReadDiff(PIN_PA1, PIN_PA2);
if (adc_reading < -32768 ) {
  Serial.println("An error was returned while taking a differential reading!")
  printADCRuntimeError(adc_reading, Serial);
}
```

### ADCPowerOptions(options) *2-series only prior to 2.5.12* For compatibility, a much more limited version is provided for 0/1-series. See below
The PGA requires powere when turned on. It is enabled by any call to `analogReadEnh()` or `analogReadDiff()` that specifies valid gain > 0; if it is not already on, this will slow down the reading. By default we turn it off afterwards. There is also a "low latency" mode that, when enabled, keeps the ADC reference and related hardware running to prevent the delay (on order of tens of microseconds) before the next analog reading is taken. We use that by default, but it can be turned off with this function.
Generate the argument for this by using one of the following constants, or bitwise-or'ing together a low latency option and a PGA option. If only one option is supplied, the other configuration will not be changed. Note that due to current errata, you **must** have LOW_LAT enabled
* `LOW_LAT_OFF`     Turn off low latency mode. *2-series only*
* `LOW_LAT_ON`      Turn on low latency mode. *2-series only*
* `PGA_OFF_ONCE`    Turn off the PGA now. Don't change settings; if not set to turn off automatically, that doesn't change. *2-series only*
* `PGA_KEEP_ON`     Enable PGA. Disable the automatic shutoff of the PGA. *2-series only*
* `PGA_AUTO_OFF`    Disable PGA now, and in future turn if off after use. *2-series only*
* `ADC_ENABLE`      Enable the ADC if it is currently disabled.     *new 2.5.12*
* `ADC_DISABLE`     Disable the ADC to save power in sleep modes.   *new 2.5.12*
* `ADC_STANDBY_ON`  Turn on ADC run standby mode                    *new 2.5.12*
* `ADC_STANDBY_OFF` Turn off ADC run standby mode                   *new 2.5.12*

Example:
```c++
ADCPowerOptions(LOW_LAT_ON  | PGA_KEEP_ON );            //  low latency on. Turn the PGA on, and do not automatically shut it off. Maximum power consumption, minimum ADC delays.
ADCPowerOptions(LOW_LAT_OFF | PGA_AUTO_OFF);            //  low latency off. Turn off the PGA and enable automatic shut off. Minimum power consumption, maximum ADC delays. **ERRATA WARNING** turning off LOWLAT can cause problems on 2=series parts! See the errata for the specific part you are using.)
ADCPowerOptions(ADC_DISABLE);                           //  turn off the ADC.
ADCPowerOptions(ADC_ENABLE                              //  Turn the ADC back on. If LOWLAT mode was on, when you turned off the ADC it will still be on,. Same with the other options.
```

As of 2.5.12 we will always disable and re-enable the ADC if touching LOWLAT, in the hopes that this will work around the lowlat errata consistently.
**it is still recommended to call ADCPowerOptions(), if needed, before any other ADC-related functions** unless you fully understand the errata and the ramifications of your actions.
**On most 2-series parts LOWLAT mode is REQUIRED in order to use the PGA when not using an internal reference, or measuring the DACREF!**


Lowlat mode is enabled by default for this reason, as well as to generally improve performance. Disabling the ADC will end the power consumption associated with it.

On 0/1-series parts, this function supports functions that are far more limited, since there are few power options.
Only the following are supported
* `ADC_ENABLE`      Enable the ADC if it is currently disabled.     *new 2.5.12*
* `ADC_DISABLE`     Disable the ADC to save power in sleep modes.   *new 2.5.12*
* `ADC_STANDBY_ON`  Turn on ADC run standby mode                    *new 2.5.12*
* `ADC_STANDBY_OFF` Turn off ADC run standby mode                   *new 2.5.12*

In all cases, if no command to turn on or off an option is passed the current setting will remain unchanged

### DAC Support
The 1-series parts have an 8-bit DAC which can generate a real analog voltage. This generates voltages between 0 and the selected VREF (which cannot be VDD, unfortunately). Set the DAC reference voltage via the `DACReference()` function - pass it one of the `INTERNAL` reference options listed under the ADC section above. This voltage must be half a volt lower than Vcc for the voltage reference to be accurate. The DAC is exposed via the analogWrite() function: Call `analogWrite(PIN_PA6,value)` to set the voltage to be output by the DAC. To turn off the DAC output, call digitalWrite() on that pin; note that unlike most* PWM pins `analogWrite(PIN_PA6,0)` and `analogWrite(PIN_PA6,255)` do not act as if you called digitalWrite() on the pin; 0 or 255 written to the `DAC0.DATA` register; thus you do not have to worry about it applying the full supply voltage to the pin (which you may have connected to sensitive devices that would be harmed by such a voltage) if let the calculation return 255; that will just output 255/256ths of the reference voltage.

The 2-series and 0-series don't have DACs, though the 2-series analog comparator has a "DACREF" and 8-bit reference DAC that can only be used as one side of the AC. This should be used through the Comparator library.


## Analog *channel* identifiers
The ADC is configured in terms of channel numbers, not pin numbers. analogRead() hence converts the number of a pin with an analog channel associated with it to the number of that analog channel, so there is no need to deal with the analog channel numbers. The one exception to that is in the case of the non-pin inputs, the constants like ADC_DAC and ADC_VDDDIV10. I have a simple system to internally signal when a number isn';'t an digital pin number, but is instead an analog channel number: Simply set the high bit. I refer to these as analog channel identifiers. When the high bit is masked off, these are the value that you must set the MUX to in order to use this input source. No AVR has ever had more than 127 pins, much less that many analog channels, so this shouldn't be an issue. With 254 valid values, the current design provides room for 127 digital pins and 127 analog inputs, where the largest modern AVRs have only 56 I/O pins (it will be a technical challenge to surpass that, because they don't have any more registers for the VPORTs, and analog multiplexers that only go up to 73 (they use the second highest bit to denote non-pin inputs. )

These numbers (they do have defines in the variants, and `ADC_CH()` will take an analog channel number (ex, "0") and turn it into the analog channel identifier. But you never need to do that unless you're getting very deep into a custom ADC library) The most common example when channels are used is when reading from things that are not pins - like the internal tap on the DAC output, or the VDDDIV10 value to find the supply voltage. These may overlap with pin numbers. Also internally, channel numbers are sometimes passed between functions. They are defined for pins that exist as analog channels, with names of the form `AINn` but **you should never use the AIN values** except in truly extraordinary conditions, and even then it's probably inappropriate. However I felt like mention of them here wax needed. Some macros abd helper functions involved are:

```text
digitalPinToAnalogInput(pin)    - converts an digital pin to an anbalog channel *without* the bit that says it's a channel (designed for the very last step of analogRead preparation where we turn the pin number into the channel to set the MUX)
digitalPinToAnalogInput1(pin)   - As above.... except for ADC1
analogChannelToDigitalPin(p)    - converts an analog channel *number* to a digital pin.
analogInputToDigitalPin(p)      - converts an analog channel identifier to a digital pin number.
digitalOrAnalogPinToDigital(p)  - converts an analog channel identifier or digital pin to a digital pin number
ADC_CH(channel number)          - converts channel numbers to analog channel identifier

```

Try not to use these unless you're getting really deep into library development and direct interaction with the ADC; we provide all the constants you will need. listed above.

## A word of warning to capacitive touch sensing applications
Libraries exist that use trickery and the ADC to measure capacitance, hence detect touch/proximity. Most of these libraries (since Atmel always locked up QTouch) relied on the leftover charge in the ADC S&H cap. The magnitude of this source of error is much larger on classic AVRs. It is considered undesirable (this is why when changing ADC channels in the past, you were advised to throw out the first couple of readings!) and with each generation, this has been reduced. There are still ways to do this effectively on the 2series, but require very different approaches.
