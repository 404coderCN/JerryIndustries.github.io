---
layout: post
title: Cricket synchronization with raspberry pi pico
image: 
  path: /assets/img/projects/cricket_top.jpg
description: >
  chirp chirp chirp 🦗
sitemap: false
hide_last_modified: true
---
# Cricket synchronization with raspberry pi pico

## Background
Synchronization is a fascinating phenomenon common to a lot of biological systems. One of the most intriguing one can be observed on snowy tree crickets. When you wonder around in the woods or take a night walk in Ithaca, you can always hear these little creatures chirp. Most of the times, our brains filter them out as white noises, but if you pay attention, you will notice they chirp in synchrony. That's already better than some of us who lacks the "music gene"! In a addition to synchronization, snowy tree crickets change the frequency and speed of their chirps as a function of the ambient temperature. On the east coast of the US, you can estimate the temperature outside (in Fahrenheit) by adding 40 to the number of chirps that you count in 15 seconds, so forget about Apple weather, next time, ask a snowy tree cricket for the temperature! This fascinating relationship between cricket chirps and temperature has a name--<a href="https://en.wikipedia.org/wiki/Dolbear%27s_law">Dolbear's law</a>.

Now that we know a bit about crickets, let's see how we can simulate their very fascinating behaviors. To accomplish this goal, we'll separate the project into three steps: 
1. Synthesize cricket chirps
2. Detect other chirps 
3. Synchronization

Before we dive into the technical details, I want to first say a few words about the insights I gained from this project and how it changed the way I view things. For a long time, I thought the goal of Electrical engineering is to make cool intelligent products. Phones, computers, robots and drones; I took my classes thinking I'm getting myself closer to designing one of these on my own. Whenever I have the freedom to choose my own final project, I always think about building some kind of robot, as it feels like a direct derivative of all the cool things I learnt in ECE, and anything other than that feels not so related and lacks the excitement. One a level, I thought of ECE and cool technologies as the same thing. They are the reason we learn about this subject and they are the final products. This project reminds me that ECE is much more than that. It can be much more than just the "objective", because it could also be a tool that we use to learn about all the exiciting things around us. And this applies to not just ECE. Engineering, in the general sense, is exiciting because we have this extra tool and extra perspective we can use to learn about the fascinating phenomenon in nature and all other things that we're interested in. There is a great big world outside of engineering, go explore it with all the tools you have!

## Technical implementation

### Synthesize cricket chirps
Before anything happens, let there be crickets. How? You ask. Well, it's time to introduce our very first algorithm -- **Direct Digital Synthesis**(DDS). Long story short, Direct Digital Synthesis produces an analog waveform by performing digital-to-analog conversion on a time-varying digital signal. The algorithm takes advantage of the homomorphism between an accumulator variable and the phasor angle: that is, rotating the phasor resembles the act of incrementing a 32-bit accumulator variable. To be more specific, the algorithm first initialize a 32-bit variable called Accumulator, which stores information of a scaled version of the phase angle. A phasor angle of 0 degrees, therefore, corresponds to an accumulator value of 0, and a phasor angle of 360 degrees corresponds to an accumulator value of 2^32-1. Because the analog waveform is generated by sending new samples to the Digital Analog Converter (DAC) at the sampling frequency, F𝑠, we obtain the relationship between the sampling frequency and the output frequency – the frequency of the chirp sounds that we wish to produce:  

![400x200](/assets/img/projects/DDS_1.png) 

Be aware of the values for F𝑠 and Fout. The required accumulator increment amount can be calculated with the following formula: 

![400x200](/assets/img/projects/DDS_2.png)

The accumulator variable is then used as an index to a sine Lookup Table, and the corresponding output signal magnitude is sent to the Digital to analog converter (DAC). The sine table itself has 256 entries. Hmm, wait a minute, didn't you say the accumulator has 2^32 different states? How is 256 enough? To answer this,we first have to know what less entries means in our case. Basically, the less entries that you have in your sine table, the more harmonic distortion you will hear in your generated tones. This is due to the fact that the DAC acts like a zeroth-order hold. If you send it a value, it retains that value until you send it a new one. So, smooth sine waves become the jagged approximations to sine waves shown below. 

<img src="{{ "/assets/img/projects/jagged_sine.png" | prepend: site.baseurl | prepend: site.url}}" alt="jagged sine wave" />

The fundamental frequency correspond to fft index 1, and the first error harmonic is approximately at the number of entries in the table (i.e. the sample frequency) minus 1. So as the number of entries in the table increase, the first error harmonic moves away from rthe fundamental. Now, human ears can only hear sounds within the range 20Hz~20kHz, so as long as we can push the first error harmonic beyond 20kHz, we won't be able to tell the difference! Plus, the amplitude of the error harmonic will also decrease as the entries in the table increase. This way, we can save ourselves from creating a gigantic 2^32 element table! For the same reason,we only use the most significant 8-bits of the accumulator to index into the sine table. The remaining bits are used to remain fractional precision in determining when we need to output a different value to the DAC. Now, let's do some math. Our desired syllable frequency is 2300 Hz, and the DAC sampling frequency is 40 kHz, thus syllable is expected to be 17ms long, corresponding to 680 cycles. Similarly, the separation time between each syllable is approximately 2ms, corresponding to 80 audio cycles. The chirp repeat interval is approximately 260ms, corresponding to 10400 audio cycles. As the DAC sampling frequency is 40 kHz, it’s expected to receive 40000 values computed by DDS every second. While it’s possible to compute all the values before playing the chirp sound, such implementation takes up too much memory and requires long set-up time. To solve this problem, timer interrupt, which allows one to momentarily exit the loop() function at precisely timed intervals while executing the separate ISR function, is introduced into the program. Particularly, the ISR would be triggered every 1/40000 second, during which the DDS algorithm computes the analog signal amplitude and sends it to the DAC. 

Because cricket chirp’s volume ramps up and down in real life rather than maintaining a consistent loudness, amplitude modulation is applied. We defined an attack time and decay time, during which the signal amplitude smoothly ramps from 0 to its max amplitude, and vice versa. In achieving the amplitude modulation, we multiply the chirp with the amplitude envelope function shown in Figure 1. 




Cum sociis natoque penatibus et magnis <a href="#">dis parturient montes</a>, nascetur ridiculus mus. *Aenean eu leo quam.* Pellentesque ornare sem lacinia quam venenatis vestibulum. Sed posuere consectetur est at lobortis. Cras mattis consectetur purus sit amet fermentum.

> Curabitur blandit tempus porttitor. Nullam quis risus eget urna mollis ornare vel eu leo. Nullam id dolor id nibh ultricies vehicula ut id elit.

Etiam porta **sem malesuada magna** mollis euismod. Cras mattis consectetur purus sit amet fermentum. Aenean lacinia bibendum nulla sed consectetur.

## Inline HTML elements

HTML defines a long list of available inline tags, a complete list of which can be found on the [Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/HTML/Element).

- **To bold text**, use `**To bold text**`.
- *To italicize text*, use `*To italicize text*`.
- Abbreviations, like HTML should be defined like this `*[HTML]: HyperText Markup Language`.
- Citations, like <cite>&mdash; Mark otto</cite>, should use `<cite>`.
- ~~Deleted~~ text should use `~~deleted~~` and <ins>inserted</ins> text should use `<ins>`.
- Superscript <sup>text</sup> uses `<sup>` and subscript <sub>text</sub> uses `<sub>`.

Most of these elements are styled by browsers with few modifications on our part.

## Heading 2
Vivamus sagittis lacus vel augue rutrum faucibus dolor auctor. Duis mollis, est non commodo luctus, nisi erat porttitor ligula, eget lacinia odio sem nec elit. Morbi leo risus, porta ac consectetur ac, vestibulum at eros.

### Heading 3
Vivamus sagittis lacus vel augue rutrum faucibus dolor auctor.

#### Heading 4
Vivamus sagittis lacus vel augue rutrum faucibus dolor auctor.

##### Heading 5
Vivamus sagittis lacus vel augue rutrum faucibus dolor auctor.

###### Heading 6
Vivamus sagittis lacus vel augue rutrum faucibus dolor auctor.

## Code

Cum sociis natoque penatibus et magnis dis `code element` montes, nascetur ridiculus mus.

~~~js
// Example can be run directly in your JavaScript console

// Create a function that takes two arguments and returns the sum of those
// arguments
var adder = new Function("a", "b", "return a + b");

// Call the function
adder(2, 6);
// > 8
~~~

## Lists

Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Aenean lacinia bibendum nulla sed consectetur. Etiam porta sem malesuada magna mollis euismod. Fusce dapibus, tellus ac cursus commodo, tortor mauris condimentum nibh, ut fermentum massa justo sit amet risus.

* Praesent commodo cursus magna, vel scelerisque nisl consectetur et.
* Donec id elit non mi porta gravida at eget metus.
* Nulla vitae elit libero, a pharetra augue.

Donec ullamcorper nulla non metus auctor fringilla. Nulla vitae elit libero, a pharetra augue.

1. Vestibulum id ligula porta felis euismod semper.
2. Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus.
3. Maecenas sed diam eget risus varius blandit sit amet non magna.

Cras mattis consectetur purus sit amet fermentum. Sed posuere consectetur est at lobortis.

HyperText Markup Language (HTML)
: The language used to describe and define the content of a Web page

Cascading Style Sheets (CSS)
: Used to describe the appearance of Web content

JavaScript (JS)
: The programming language used to build advanced Web sites and applications

Integer posuere erat a ante venenatis dapibus posuere velit aliquet. Morbi leo risus, porta ac consectetur ac, vestibulum at eros. Nullam quis risus eget urna mollis ornare vel eu leo.

## Images

Quisque consequat sapien eget quam rhoncus, sit amet laoreet diam tempus. Aliquam aliquam metus erat, a pulvinar turpis suscipit at.

![800x400](https://via.placeholder.com/800x400 "Large example image")

![400x200](https://via.placeholder.com/400x200 "Medium example image")

![200x200](https://via.placeholder.com/200x200 "Small example image")

## Tables

Aenean lacinia bibendum nulla sed consectetur. Lorem ipsum dolor sit amet, consectetur adipiscing elit.

| Name     | Upvotes   | Downvotes |
|:---------|:----------|:----------|
| Alice    |        10 |        11 |
| Bob      |         4 |         3 |
| Charlie  |         7 |         9 |
|==========|===========|===========|
|Totals    |        21 |        23 |

Nullam id dolor id nibh ultricies vehicula ut id elit. Sed posuere consectetur est at lobortis. Nullam quis risus eget urna mollis ornare vel eu leo.

*[HTML]: HyperText Markup Language
*[CSS]: Cascading Style Sheets
*[JS]: JavaScript