---
layout: post
title: Filtering Down PDM to PCM
date:  2020-10-12 00:00:00 -1000
categories:
---

* TOC
{:toc}

# Introduction

If you've read my previous two posts in this series, you know that I'm improving my general DSP
knowledge by applying it to a concrete example of taking in single bit data stream of
a PDM microphone and converting it into a standard PCM coded samples.

The application itself is only a means to an end, most important is understanding the why and 
the how of every design decision along the way.

After diving into [CIC filters](/2020/09/30/Moving-Average-and-CIC-Filters.html), 
the characteristics of the 
[sigma-delta generated PDM signal of a MEMS microphone](http://localhost:4000/2020/10/04/PDM-Microphones-and-Sigma-Delta-Conversion.html), 
and [learning how to come up with FIR filter coefficients](/2020/10/11/Designing-Generic-FIR-Filters-with-pyFDA-and-Numpy.html),
the time has come to start building up the filter architecture.


# PDM to PCM Problem Statement

The problem that I want to solve is to convert a PDM signal to a PCM signal while
maintaining the the quality level of the microphone.

Going forward, I'll use the characteristics of the ST MP34DT01-M microphone of my
[Adafruit test board](https://www.adafruit.com/product/3492) and let them guide
the specifications of the filter design.

From [the datasheet](https://cdn-learn.adafruit.com/assets/assets/000/049/977/original/MP34DT01-M.pdf): 

![Microphone Characteristics Table](/assets/pdm/pdm2pcm/mic_characteristics.png)

![Microphone Frequency Response](/assets/pdm/pdm2pcm/mic_freq_response.png)


The requirements for our design are:

* An output sample rate of 48kHz

    48kHz is universally supported and probably the most common sample rate for high quality audio, 
    though it's obviously overkill for a microphone that only goes up to 10kHz.

* 2.4 MHz PDM sample rate 

    The microphone supports a PDM clock rate between 1 and 3.25MHz. 

    We shall soon see that some very efficient filters are perfect for 2x decimation, 
    so it's best to select a ratio that's divisible by 2 or by 4.

    3.072MHz is 64 times higher and 2.4MHz is 50 times higher than our desired output rate 
    of 48kHz. The acoustic and electrical characteristics are given for 2.4MHz, so let's
    choose that.

    It also avoids giving the impression that decimation filters should have a power of 2 ratio, 
    and a lower clock reduces power consumption too.

* A passband from 0 to 6kHz

    State of the art MEMS microphones have a flat sensitivity curve that goes all the way up 
    to 20kHz, but this microphone definitely does not fall in that category. The frequency
    response is only flat between 100Hz and 5kHz, after which it starts going up. And
    there's not data above 10kHz.

    6kHz seems like a good point to start the transition band.

* A stop band that starts at 10kHz

    Since the specification of the microphone doesn't say anything about behavior above
    10kHz, I'm simply assuming that this is no-go territory, so that's where the
    stop band will start.

* The SNR of our output signal in the passband is 58dB.

    The microphone datasheet specifies a 61dB SNR, but decimation will alias higher
    frequencies into the remaining frequency band. In the case of PDM signals, these higher
    frequencies contain a lot of noise which will lower the SNR.

    I'm allowing a 3dB SNR deterioration, but this is something with quite a bit of
    filter design consequences and thus something that needs to be discussed more in
    depth soon.

* The maximum ripple we're willing to accept in the passband *at the output* is 0.1dB

    This seems to be a pretty typical requirement for high quality audio.

    In a filter architecture with more than 1 filter, each successive filter will impact the
    passband ripple. As a result, individual filters will need to have tighter specification
    than this overall ripple.

# Stop Band Attenuation for a Decimation Filter

Most of this blog post had already been written when it suddenly hit me: I had been completely
wrong about how to determine the stop band attenuation for the decimation filter.

And since that stop band attenuation influences a whole bunch of other filter parameters,
I had to go back to the drawing board to get things right.

In an earlier [Basics of Decimation](2020/09/30/Moving-Average-and-CIC-Filters.html#intermission-the-basics-of-decimation)
section about CIC filters, I came really close, and still I missed the most important part:

**Higher decimation ratios require higher stop band attenuation to get the same noise floor.**

In the example below, you can see what happens when you keep the stop band attenuation of
a decimation filter the same for different decimation ratios:

![Stop Band Attenuation and Decimation](/assets/pdm/pdm2pcm/pdm2pcm-stop_band_attenuation.svg)

Of interest here is the amount of unwanted aliasing noise that was added underneath the
great signal of interest. In the 2x decimation case, only 1 'unit' of stop band
noise was added, while it's 3x that for the 4x decimation case.

If we wanted the amount of aliasing noise to be the same for the 2x and the 4x decimation, 
we'd have to increase the stop band attenuation for the latter.

Let's now put this in numbers for our microphone case.

* The microphone has a 61dB(A) SNR.

    When talking about SNR, it's important to define the frequency range over which the
    signal and noise power are measured.

    The 'A' of dB(A) gives us a clue: it standards for [A-weighting](https://en.wikipedia.org/wiki/A-weighting).

    When sound is measured over the audible frequency range, the results are weighted by
    the sensitivy of the ear.


* The decimation ratio is 50x.

    We're going from 2.4MHz down to 48kHz.

* The noise level above 10kHz is below -20dB for all frequencies above 10kHz.

    This is almost certainly wrong, but I expect it to be pessimistic with some additional
    margin.

    Recall the noise slopes of various sigma-delta converters:

    ![Sigma Delta Noise Slopes](/assets/pdm/sigma_delta/noise_slope_different_orders.svg)

    The noise here does not go above -30dB.



# PDM to PCM First Try: Just Filter the Damn Thing!

Do we really need a complex filter architecture to convert PDM to PCM? Maybe a single equiripple
filter is sufficient for our needs?

Let's just fill in the numbers and see what happens:

```
...
Trying N=1382
Rpb: 0.094294dB
Rsb: 59.976587dB
Trying N=1383
Rpb: 0.093942dB
Rsb: 60.020647dB
Filter order: 1383
```

After more than a little while, it comes up with a filter that has 1384 taps, and the following
frequency response graph:

![Just Filter the Damn Thing](/assets/pdm/pdm2pcm/filter_the_damn_thing.svg)

Since this is a decimation filter, we only need to evaluate it for each output sample.
At 48kHz, this means that we need 48000 * 1384 = 66M multiplications per second.

That's a lot, but that's good if you're looking for a baseline and want to impress people 
about how you were able to optimize your design!

# Decimation is a Divide and Conquer Problem

All other things equal (sampling rate, pass band ripple, stop band attenuation), the number of
filter taps depends on the size of the transition band, compared to sample rate.

In the example above, the sample rate is 2400 kHz and the transition band is just 4kHz, a ratio
of 600!

If we reduce the sample rate by 5 to 384kHz and keep everything else the same, the number of taps
goes rougly down by 5 as well:

![Fpdm divided by 5 Frequency Response](/assets/pdm/pdm2pcm/fpdm_div5.svg)

Reducing the number of taps from 1384 down to 278 taps, gives

48000 * 278  = 13M multiplications

But that's not a fair comparison, because to be able use that filter, we first need to
decimate the signal at the initial sample rate from 1920kHz to 384kHz.

If we do this in the most naive way possible like any other decimation filter, we create a 
filter that doesn't touch the pass band and the transition band of the final result, and that 
filters away everything above the original sample rate divided by 5. 
In other words: a pass band from 0 to 10kHz, and a stop band that starts at 77kHz. This 
guarantees that none of the frequencies above 77kHz will alias into the range of 0 to 10kHz 
after decimation.

Number of taps required? 13!

![Fpdm to Fpdm/5 - Frequency Response](/assets/pdm/pdm2pcm/fpdm_to_fpdm_div5.svg)

Total number of multiplications:

```
384,000 *  13 =  5M             (5x decimation - from 1920 to 384 kHz)
 48,000 * 278 = 13M             (8x decimation - from 384 to 48 kHz)
-----------------------------------
                18M multiplications/s
```

By splitting up the dumb initial filter into 2 filters, we've reduced the number of multiplications from
66M downto 18M, a factor of more than 3!

If we can get that kind of savings splitting up a 40x decimation filter into a 2 filters, there must
be similar optimization possible by splitting up that 8x decimation filter. 

There must be some kind of optimal arrangment that results in a minimal number of multiplications
to get from our initial to our desired sample rate.

But before we look at that, some clarifications and corrections must be made about specifying the right 
decimation filter parameters.

# Optimal Pass Band / Stop Band Limits for Decimators

As I mentioned earlier, when you design a single filter for a decimator, it's a simple matter
of entering the number of the pass band and stop band parameters and let your filter tool
do its magic.

For a filter with decimation ratio *n*, the pass band and the stop band frequency must be smaller
than the *Fsample/2/n*, where *Fsample* is the original sample rate.

In this section, we'll assume the following parameters:

* Incoming sample rate: 288kHz
* Valid signal bandwidth: 0 to 10kHz
* Outgoing sample rate: 48kHz
* Desired pass band: 0 to 6kHz
* Desired transition band: 6kHz to 10kHz
* Desired stop band: 10kHz to 24kHz

The ratio of incoming and outoing sample rate is 6, so we need a 6x decimating filter.

The fat black line below shows the frequence response if we use a single filter:

![Direct 6x Decimation Filter](/assets/pdm/pdm2pcm/pdm2pcm-div6_decimation_filter.svg)

We already saw earlier that 2 less aggressive filters result in a lower number of taps (and thus
multiplications) and 1 more aggressive filter. In this particular example, that means we can
either decimate by 2x first and 3x after that, or the other way around.

The figure below shows the two paths:

![Cascaded Decimation Steps](/assets/pdm/pdm2pcm/pdm2pcm-cascaded_decimation.svg)

There are 2 major things of note:

1. Naive vs smart filtering in the first stage 

    If we treated the first decimation stage as any other decimation filter, we'd put the
    stop band at end of *Fsample/2/n* to avoid any kind of aliasing of the upper
    frequencies into the remaining frequency band.

    But that is too aggressive!

    What we can do instead is start of the stop band of the first filter at *Fsample/n* - *Fsb*.

    This expands the transition band by the area marked with the green rectangle and makes
    the first stage decimation filter considerably less steep. This is especially true for
    a 2x decimation ratio: in our specific example, between the naive and the smart filter,
    the transition band doubles from 62 (=72-10) to 124kHz (=144-10-10).

    A part of the upper frequencies will now alias into the lower frequency range, but we
    have a second filter stage to clean that up for us.

1. pass band frequency of the output signal is of no importance until the last filter

    The goal of the inital filter stages is to reduce the sample rate but prevent upper range
    frequencies from aliasing into the real signal. In our example, the final signal will
    have a transition band bewteen 6 and 10 kHz, but that doesn't mean that it's fine for 
    other signals to corrupt the original signal in the range.

    That's why the initial decimation filter uses 10kHz as the end of the pass band, not 6kHz.

1. pass band/stop band symmetry for the first stage 2x decimation case

    An interesting aspect is that the frequency response of the 2x decimation filter shows a 
    point symmetry around the *Fsample/4* axis:

    ![2x Decimation Symmetry](/assets/pdm/pdm2pcm/pdm2pcm-half_band_symmetry.svg)

    With some additional ripple constraints, this becomes a so-called half band filter.
    Half band filters have the interesting property that every other filter tap, except for the
    center tap, has a value of zero.

    This reduces the number of multiplications for a half band filter almost by half, which
    makes 2x decimation filters computationally an amazing deal..

# Passband Ripple and Stop Band Attenuation for Cascaded Filters

We're shooting for an overall pass band ripple of 0.1dB and a stop band attenuation of 60dB. 

When there's only 1 filter, meeting that goal is a matter of specifying that as a filter design parameter.

But what when multiple filters are cascaded?

In their 1975 paper 
["Optimum FIR Digital Filter Implementations for Decimation, Interpolation, and Narrow-Band Filtering](https://web.ece.ucsb.edu/Faculty/Rabiner/ece259/Reprints/087_optimum%20fir%20digital%20filters.pdf),
Crochiere and Rabiner write the following:

> As it is desired that the overall pass band ripple for the composite of K stages be maintained 
> within 1+-delta it is necesary to require more severe frequency constraints on the individual filters
> in the cascade. Aconvenient choice which will satisfy this requirement is to specify the pass band
> ripple constraints for each stage *i* to be within 1+delta/K.

In other words: if we split the filter into 3 stages, they suggest to split the joint passband ripple of 0.1dB into
3 pass band ripples of ~0.03dB.

(Note that close to 1, *20 * log10(x)* ~ *x-1*. Since pass band ripple is a deviation around 1, we can simply
divide the dB number without having to convert from dB to linear and back.)

In practise, this will create a joint pass band ripple that's a bit lower than specified because it's
unlikely that all 3 filters will have peaks and throughs at the same location.

Now for the stop band. From the same paper:

> In the stop band the ripple constraint for the composite filter must be *delta* and this
> constraint must be imposed on each of the individual low-pass filter as well, in order to suppress
> the effects of aliasing.

This confused me since as many frequency range will be aliased onto final signal as the decimation
ratio. And you need to account for that.

But then I realized that I whether you use a single filter or multiple ones, you always need to
take the decimation ratio into account when specifying the stop band attenuation.

Let's use this on our example: we want 60dB stop band attenuation. When reducing the sample rate from
1920kHz to 48kHz, there's a 40x decimation ratio. Our filter needs to correct for that: 

```
-60dB = 0.001 / 40 = 0.0000025 -> -72dB
```

Going forward, all 


# Major Sample Rate Reduction with a CIC Filter

Earlier, I wrote about [CIC filters](/2020/09/30/Moving-Average-and-CIC-Filters.html) in preparation 
of this blog post.

CIC filters make it possible to reduce the amount of hardware to decimate a signal by a large
factor with next to no resources: no multipliers, and only a handful of register and adders.

The only price to pay is less than ideal behavior in the pass band, and a terrible stop band behavior
when used as a stand-alone filter, but that's something that can be fixed by having additional stages.

And that's exactly what we're doing here in our divide-and-conquer approach.

Here's the plan: we use a factor 16 CIC decimation filter to bring the sample rate
down from 3072 kHz to 192 kHz. 16x is a good compromise: it's a significant reduction,
yet the pass band attenuation at 14kHz is only XXXX.

If we then use the monolitic approach to decimate by a factor of 4 to end up with a 48kHz
output rate, we'd need 57 taps.

```
    48,000 * 57 = 2.7M multiplications
```

We can do better by splitting up the 4x decimator into 2 2x decimators:

* 192 -> 96: 18 taps
*  96 -> 48: 28 taps

```
     96,000 * 18  = 1.7M multiplications
     48,000 * 28  = 1.3M multiplications
   --------------------------------------
                    3.0M multiplications total

```

Apparently not! There seems to be some kind of cross-over point below which there
isn't a benefit in splitting up.

Still, we went from 104M to just 2.7M multiplications, a 38x improvement!

Can we do better?

# Stop Band Optimization

So far, the stop band of the intermediate decimation filter has been suboptimal: we
simply put it at 1/4th of the input rate, because that's what you do for a monolithic
decimation filter.

But it's not optimal in a divide an conquer configuration!

When we decimate from 192 to 96, it's overkill to put the stop band at 48kHz: we can
put the stop band at (96-20) = 76 kHz instead!

Yes, the frequency range from 48kHz to 76kHz will now alias onto 20kHz to 48kHz range,
but who cares? Nobody can hear anything above 20kHz anyway, and we have another filter
comping up to clean things up in the next decimation step!

With this new stop band constraint, there are only 11 filter taps instead of the earlier
18, for a total of:


```
     96,000 * 10  = 1.1M multiplications
     48,000 * 28  = 1.3M multiplications
   --------------------------------------
                    2.3M multiplications total

```

# Exploiting Filter Symmetry

For audio applications, linear phase behavior is important: the human ear is able to detect
phase differences in otherwise similar signals.

An FIR filter with linear phase behavior must have symmetry in its filter coefficients. And
all the filters we've designed so far have exactly that.

Because of this symmetry, each filter coefficient (except for the center one) will appear
twice, mirrored around the center tap. By adding the input samples that use the same filter coefficient
before doing the multiplication, we can reduce the number of multiplications by half!

Instead of 57 multiplications, we're now at 1+(56/2) = 29 multiplications per output sample.

# Best Case Low Pass Filtering

It should be abundantly clear now that we need that low pass filter before we can decimate to
a lower sample rate.

Let's use the following parameters:

* Original sample rate: 3.072 MHz
* Oversample rate factor: 64
* Desired sample rate: 48 kHz
* Original signal bandwidth: 20 kHz
* Desired pass band: 0 dB
* Desired stop band: 96 dB

I chose 96 dB because that's the theoretical maximum SNR for 16-bit audio. Most PDM microphones only
have an SNR in the low sixties, so this is overly aggressive, but let's just see what we can do.

The transition from pass band to stop band will start at 20 kHz. And if we look at the graph for
a 64x oversampling, 4th order sigma-delta convertor, we see that the noise goes above 96 dB 

Since the noise starts going up immediately above 24 kHz (=48/2), we have 4 kHz to construct a filter
that goes from a pass band to the stop band. 

# References

**Filter Design**

* [Halfband Filter Design with Python/SciPy](https://www.dsprelated.com/showcode/270.php)

    Simple example that shows how to calculate half-band filter coefficients with NumPy using the Remez
    algorithm and with a windowed sinc filter. However, it doesn't discuss how to calculate the weights
    of the Remez algorithm.

* [Multiplier-Free Half-Band Filters](https://www.cs.tut.fi/~ts/sldsp_part2_identical_subfilters_halfband.pdf)

    Excellent discussion about half band filters, ways to design them, and how to design them without
    multipliers. Also has an extensive example on how to convert from PDM to PCM with a CIC filter followed
    by 4 half band filters.

    The [website of this professor](https://www.cs.tut.fi/~ts/) has a lot of course notes online. They are 
    all worth reading.

**Decimation**

* [Optimum FIR Digital Filter Implementations for Decimation, Interpolation, and Narrow-Band Filtering](https://web.ece.ucsb.edu/Faculty/Rabiner/ece259/Reprints/087_optimum%20fir%20digital%20filters.pdf)

    Paper that discusses how to size cascaded filter to optimized for FIR filter complexity.

* [Seamlessly Interfacing MEMS Microphones with Blackfin Processors](https://www.analog.com/media/en/technical-documentation/application-notes/EE-350rev1.pdf)

    Analog Devices application note. C code can be found[here](https://www.analog.com/media/en/technical-documentation/application-notes/EE350v01.zip)

* [The size of an FIR filter for PDM-PCM conversion](https://www.dsprelated.com/thread/11806/the-size-of-an-fir-filter-for-pdm-pcm-conversion)

    Discussion about PDM to PCM conversion on DSPrelated.com.

* XMOS Microphone array library

    https://www.xmos.ai/download/lib_mic_array-%5buserguide%5d(3.0.1rc1).pdf

    Lots of technical info about PDM to PCM decimation.

**Microphones**

* [Understanding Microphone Sensitivity](https://www.analog.com/en/analog-dialogue/articles/understanding-microphone-sensitivity.html)


