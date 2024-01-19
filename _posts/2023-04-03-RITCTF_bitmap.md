---
title: 2023RITCTF bitmap wp
permalink: /posts/2023/04/2023RITCTF/
date: 2023-04-03
tags: 
  - Steganography
---

## Steganography bitmap wp

Thanks to the simplicity of RITCTF, I achieved the achievement of AK Steg for the first time, very happy : )))  

![bitmap](https://cdn.jsdelivr.net/gh/albedowang/cdn@1.4.0/wp/bitmap.bmp)

A QR code picture. After scanning, you will get the URL: https://tinyurl.com/3acwfenx. Click on it to jump to a YouTube video. The BGM is Never Gonna Give You Up. It is trash.  

Next, open it with a hexadecimal editor. After browsing for a while, I saw that there is an XOR BY 'FF'. I don't understand it very well. It is obviously useless to XOR the black and white QR code image with FF, so I will leave it for now. We will use it later.   

![bitmap_hex](https://cdn.jsdelivr.net/gh/albedowang/cdn@1.4.0/wp/bitmap_hex.png)

According to the prompt in the title description "Some of these squares are not like the others." I looked at the QR code carefully and found that the two strings of squares in the red frame seemed to be added later, but because I understood that the two strings of squares were used It means Timing, which is useless, so the focus is here.  

![bitmap_wrong](https://cdn.jsdelivr.net/gh/albedowang/cdn@1.4.0/wp/bitmap_wrong.bmp)

If you look carefully, you can see a series of strange things in the lower right corner of the picture.  

![bitmap_color](https://cdn.jsdelivr.net/gh/albedowang/cdn@1.4.0/wp/bitmap_color.bmp)

After zooming in, we can guess that this string is the data that contains the flag. According to the previously discovered XOR BY 'FF', we can probably get the flag by XORing this string of data with FF.  

![bitmap_colorstring](https://cdn.jsdelivr.net/gh/albedowang/cdn@1.4.0/wp/bitmap_colorstring.png)

After taking the screenshot, use a hexadecimal editor to view it.  

![bitmap_flagstring](https://cdn.jsdelivr.net/gh/albedowang/cdn@1.4.0/wp/bitmap_flagstring.png)

To sort it out it is:  

    ff ff ff ff ff ff ff ff ff ff ff ff ff bd b3 ff b0 a8 b9 ff b6 ac b7 ff a0 b4 ba ff a6 c2 d8 ff a0 ad b6 ff ab ac ba ff bc a0 d8 ff df df df ff df b6 a9 ff c2 d8 bd ff b3 b0 a8 ff b9 b6 ac ff b7 d8 df ff df df df ff b2 b0 bb ff ba c2 bc ff bd bc df ff df df df ff b6 b1 c2 ff b7 ba a7 ff df df df ff df b0 aa ff ab c2 b7 ff ba a7 df ff df df df ff bc b6 af ff b7 ba ad ff d7 aa ac ff ba df b6 ff b1 df b7 ff ba a7 d6 ff c2 d8 9b ff bb a4 f5 ff d4 4e 40 ff 5a 65 27 ff ca 09 d8 ff 9e b2 63 ff 2e 4c 1d ff ac 7b 04 ff f3 2f 52 ff b6 68 d3 ff e5 5c ed ff ff d8 ff ff ff ff ff ff ff ff ff ff

Then use FF XOR and convert to ASCII code to get: BLOWFISH_KEY='\_RITSEC\_' IV='BLOWFISH' MODE=CBC IN=HEX OUT=HEX CIPHER(USE IN HEX)='64 44 5b 0a 2b b1 bf a5 9a d8 35 f6 27 61 4d 9c d1 b3 e2 53 84 fb 0c d0 ad 49 97 2c 1a a3 12 00'  

After searching, I learned that BLOWFISH is an encryption method. Use Cyberchef's blowfish decrypt directly, enter the KEY, IV, MODE, IN, OUT given by him, and then convert the resulting hexadecimal string into ASCII code to get the flag.  

        get flag: RS{CONSIDER_THESE_BITS_MAPPED}
