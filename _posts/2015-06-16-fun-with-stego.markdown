---
layout: post
title: "Picture is Worth 1000 Words"
date: 2015-06-16 20:00:00
categories: steganography golang image
---

About a year ago, I was sick as a dog.  I caught some sort of child generated 
disease which is common to anyone with small kids that go to pre-school.  Anyhow
while on cough medicine I thought it would be fun to explore steganography.  
When I went to college I had a cryptography course in which the professor threw 
two nearly identical images on the board.  He asked the class, what is different
between the two images?  People were confused by the question and started 
rattling off various things, like well, the hue is slightly different, or one is
more blocky than the other, or maybe the compression was different.  Then he let
loose the real answer, and said, the one on the right has the entire contents of
Moby Dick within it.

At that point in time, my mind was blown.  Wow!  You can actually hide data, and
mask data within other data.  The professor mentioned that a common approach for
hiding data within an image is to hide the data within low order bits of the the
color values of each pixel, as those low order bits really don't add too awful 
much to the color of the particular pixel.

Not sure why I got motivated last year to try and implement a simplistic program
but here is what I was thinking:

1. Iterate over every pixel.
2. Break each pixel into 4 colors (Red, Blue, Green, Alpha)
3. Take 2 bits of every RBGA color byte and stuff my data in the LSBs
4. Denote an end of hidden document pixel
5. Break up my input data into chars
6. Put 2 bits of every input byte into each color byte of the pixel

Here is the basic implementation:

{% highlight go %}
        // decode the image
        img, _, image_decode_err := image.Decode(input_reader)
        // panic if image isn't decoded
        panicOnError(image_decode_err)
        // get the bounds of the image
        bounds := img.Bounds()
        // create output image
        output_image := image.NewNRGBA64(img.Bounds())
        // get the rows and columns of the image
        var message_index int = 0
        // loop over rows
        for y := bounds.Min.Y; y < bounds.Max.Y; y++ {
                // loop over columns
                for x := bounds.Min.X; x < bounds.Max.X; x++ {
                        // get the rgba values from the input image
                        r, g, b, a := img.At(x, y).RGBA()
                        // if we have bytes in message
                        if message_index < len(message) {
                            // first two bits
                            newr := uint32(message[message_index]>>6) + (r & lsb_mask)
                            // second two bits
                            newg := uint32(message[message_index]>>4) & ^lsb_mask + (g & lsb_mask)
                            // third two bits
                            newb := uint32(message[message_index]>>2) & ^lsb_mask + (b & lsb_mask)
                            // last two bits
                            newa := uint32(message[message_index]) & ^lsb_mask + (a & lsb_mask)
                            message_index++
                            // set the color in the new output image
                            output_image.SetNRGBA64(x, y, color.NRGBA64{uint16(newr), uint16(newg), uint16(newb), uint16(newa)})
                        } else if message_index == len(message) {
                            // if we are done with our message bytes
                            message_index++
                            // set a null ascii char to know if we are done
                            output_image.SetNRGBA64(x, y, color.NRGBA64{uint16(0), uint16(0), uint16(0), uint16(0)})
                        } else {
                            // otherwise, just put the exact values in the new image
                            output_image.SetNRGBA64(x, y, color.NRGBA64{uint16(r), uint16(g), uint16(b), uint16(a)})
                        }
                }
        }

{% endhighlight %}

Reading in the input image, and then iterating over the bounds, we can break the
image into RGBA color codes.  The real interesting part is the binary operations
we can do to make the bits in the byte:

{% highlight go %}
// first two bits
newr := uint32(message[message_index]>>6) + (r & lsb_mask)
// second two bits
newg := uint32(message[message_index]>>4) & ^lsb_mask + (g & lsb_mask)
// third two bits
newb := uint32(message[message_index]>>2) & ^lsb_mask + (b & lsb_mask)
// last two bits
newa := uint32(message[message_index]) & ^lsb_mask + (a & lsb_mask)
{% endhighlight %}

We are shifting our message char 6 bits to the right, converting it to an unsigned
32 bit int, and then adding our red color value after masking off the lower 2 bits.
Same with the Blue, except we are shifting 4, and masking out everything that isn't
the last two bits.  Think about it like this:

{% highlight text %}

    10101010 <- our message char byte
    00001010 <- our message char byte shifted 4 positions 
&   00000011 <- And our mask inverted
-------------
    00000010 <- result of the second two bits from our origin char message byte
+   11001000 <- green color value after being masked for the least sig bits
------------
    11001010 <- final result with our data incorporated into the color!
{% endhighlight %}

Then to finish up we set our new image color value to these color values.  At 
the end of the secret message we set a completely empty color pixel on the image
so we know where the data stops.

Pretty fun little project.  [Full Gist Here!][gist]

Hope this was helpful to anyone.

[gist]: https://gist.github.com/husobee/64b49e92a57e0d6d90d2
