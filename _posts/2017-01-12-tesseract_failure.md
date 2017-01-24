---
layout: post
title:  "A Failed Case for Cracking Captcha with Tesseract"
date:   2017-01-12 19:59:03 -0800
categories: tesseract ocr
---
One day I came accross some source codes to crawl login pages protected by captcha. The code simply downloaded the captcha locally and asked human to manually crack it. I wondered whether we could use Tesseract to crack the captcha, but failed due to some Tessact limitations. This article shared some of the findings.

## Enviornment Setup
My OS is Ubuntu 16.04. [Tesseract](https://github.com/tesseract-ocr/tesseract) provides a command line tool for OCR purposes. To install it:

```shell
$ sudo apt-get install tesseract-ocr
```

The captcha I was playing with is in gif format.

![]({{ site.url }}/assets/captcha.gif)

To decaptcha, I first ran the tesseract command to take the gif as input and output to terminal:

```shell
$ tesseract captcha.gif stdout
Warning in pixReadMemGif: writing to a temp file, not directly to memory
Error in pixReadStreamGif: Cant use giflib-5.1.2; suggest 5.1.1 or earlier
Error in pixReadStream: gif: no pix returned
Error in pixRead: pix not read
Error in pixReadMemGif: pix not read
Error in pixReadMem: gif: no pix returned
Error during processing.
```

What! To understand why, let's check the tesseract version:

```shell
$ tesseract -v
tesseract 3.04.01
 leptonica-1.73
  libgif 5.1.2 : libjpeg 8d (libjpeg-turbo 1.4.2) : libpng 1.2.54 : libtiff 4.0.6 : zlib 1.2.8 : libwebp 0.4.4 : libopenjp2 2.1.0
```

Turned out that leptonica-1.73, the image processing lib, was incompatible with libgif 5.1.2. You may want to try a different version of the libaries, but a quick and dirty way was to convert the image to some other types, such as png or jpg, so that the code would no longer use libgif library.
Ubuntu provides a *convert* tool for the format converion.

```shell
$ convert captcha.gif captcha.jpg 
```

It generated the captcha in [jpg format]({{ site.url }}/assets/captcha.jpg). We can finally do the OCR now!

## The Real Meat
Run tesseract with single line PSM mode, we get

```
$ tesseract captcha.jpg stdout -psm 7
\W9
```

However, the original captcha said the characters are 'YDT9'! It is not working as expected. To understand why, I segmented each letter using GIMP and call tesseract with the character mode. For the first [character Y]({{ site.url }}/assets/y_rotated.jpg), running
```shell
$ tesseract y_rotated.jpg stdout -psm 10
W
```
If we [rotated the character]({{ site.url }}/assets/y_normal.jpg) with gimp, and run the above command again, it would produce 'Y'.

```shell
$ tesseract y_normal.jpg stdout -psm 10
Y
```

Turned out the rotation actually affected the detection accuracy a lot.
This is also confirmed by the following doc, which says [under rotation](https://github.com/tesseract-ocr/tesseract/wiki/ImproveQuality#rotation--deskewing)
tesseract may perform badly.
As the final take-away, tesseract has some limitations such as the ignorance of rotation, which make it unsuitable for cracking captcha directly.

