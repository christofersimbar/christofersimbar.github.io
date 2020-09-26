---
layout: post
title: Intigriti CTF: Secret Flag in Tweet
---

Last week I saw a tweet from Inti De Ceukelaire about Intigriti CTF. One lucky winner with valid submission will get Burp Suite Pro license and a swag. Intigriti is a vulnerability disclosure and bug bounty platform like Hackerone and Bugcrowd. I thought that this would be good for testing my skill, so I give it a try.

![_config.yml]({{ site.baseurl }}/images/1_tE2z1MwOulDzVx4_sv6kWQ.png)

First step, download image as DweADlgXgAAehHh.jpg from the tweet [https://twitter.com/intigriti/status/1083065044127150080](https://twitter.com/intigriti/status/1083065044127150080)

![_config.yml]({{ site.baseurl }}/images/0_VR3Dd2JHKNyvJtnW.jpg)

To extract content from the image, we can use binwalk:

```
binwalk -e DweADlgXgAAehHh.jpg
```

binwalk will extract all possible file types.

![_config.yml]({{ site.baseurl }}/images/1_xbnljgb9ozL-PJhxDcf9LA.png)

Find nottheflag.pdf from the extracted files, and open with a PDF reader.

![_config.yml]({{ site.baseurl }}/images/1_I1Tmb-LyY4CL58aZcsunHw.png)

We get a base64 string:

```
aHR0cHM6Ly9nby5pbnRpZ3JpdGkuY29tLzA3YjBmTDI0bGttdmE=
```

Decode the base64 string using any base64 decoder will revealed a link [https://go.intigriti.com/07b0fL24lkmva](https://go.intigriti.com/07b0fL24lkmva), from which we can download an encrypted zip file named data.zip
Examine the zip file with `binwalk`, we can see that it contains 441 jpg files.
Now to find the password, reply to above tweet, and find a hidden link [https://t.co/0PFZNd692W](https://t.co/0PFZNd692W)

![_config.yml]({{ site.baseurl }}/images/1_u8Sdogneam0vJO5B0SYUow.png)

which lead to a another twitter profile [](https://twitter.com/WhereIsTheFlag)
Download the cover image of the twitter profile by inspecting page element (available in chrome) or similar tool in other browser.

![_config.yml]({{ site.baseurl }}/images/1_P1QHfP21dexxQC4hT_okpQ.png)

Password is: F1nDBuGz_
Open the zip file. Rename the file so that the name format contains a 3 digits number (i.e 001, 002, and soon)
Combine all file to form a QR code image by using ImageMagick:

```
montage * -tile 21x21 -geometry 11x11 qrcode.jpg
```

![_config.yml]({{ site.baseurl }}/images/1_CNGC-k-cOlI-zf7wPTzKKg.png)

Use qrcode reader to reveal the hidden flag. I use [https://www.onlinebarcodereader.com/](https://www.onlinebarcodereader.com/) to complete this step.

```
FLAG:YOUWINTIGRITI
```

> Is it possible to solve this challenge without hint?

# Advanced solution

Some people managed to solve this challenge without any hint. They determine which files are black or white based on CRC code.
This command will return detailed information about the files in zip file.

```
unzip -v -qq data.zip -X *1_*
```

Result:

![_config.yml]({{ site.baseurl }}/images/1_AEnHEr6029NLz1LkXvGg8g.png)

Use `-qq` to exclude column names and extra header. `-X *1_*` is used to exclude folder name in the list.
To sort by filename we can add a sort command

```
unzip -v -qq data.zip -X *1_* | sort -t _ -g -k 2,2
```

Result:

![_config.yml]({{ site.baseurl }}/images/1_FEoR4u36lOOIlSvKsV402w.png)

We only need the CRC part. In that case we can use `awk` to print only the column we need.

```
unzip -v -qq data.zip -X *1_* | sort -t _ -g -k 2,2 | awk ‘{print $7}’
```

Result:

![_config.yml]({{ site.baseurl }}/images/1_ie4WBFMshe2UojQkg8qQmg.png)

There are more than two variation of CRC code. The top left of QR code is usually black. So, we can assume that `22eb0bb8` is black. Now we just need to see all the variations, so we can determine which one is white. We can do that by adding `| sort | uniq -c` to our previous command, so that the CRC code will be sorted, grouped and counted by occurrence.

```
unzip -v -qq data.zip -X *1_* | sort -t _ -g -k 2,2 | awk ‘{print $7}’ | sort | uniq -c
```

Result:

![_config.yml]({{ site.baseurl }}/images/1_ZC9qaSOV7l43wY4WL4EHKA.png)

`96ee0cb5` has the largest number. I assume this is white. To process this information to QR code, we need a more than just a bash command. First, we store the list of CRC codes to a file.

```
unzip -v -qq data.zip -X *1_* | sort -t _ -g -k 2,2 | awk ‘{print $7}’ > qrcodes.txt
```

Next, we can write a simple python script to convert the information to QR code.

![_config.yml]({{ site.baseurl }}/images/1_AVuBLFnwj0vjyYCXVBRH_A.png)

Result:

![_config.yml]({{ site.baseurl }}/images/1_aa4ZFh6pWfMTJ-_XneOEHA.png)

Almost like a correct QR code image, but need more adjustment. QR code usually has 3 squares with thick line on top left corner, top right corner and bottom left corner. I think there are 15 blocks that should be white. If we look the previous information we have, we can see that a combination of `5b808910` and `81f9bf5a` is 15.

![_config.yml]({{ site.baseurl }}/images/1_15T_ynpu8mV6OaJZAXxBpA.png)

So lets fix our python script:

![_config.yml]({{ site.baseurl }}/images/1_iVkZdSJPn2FwZvsvdNFaQ.png)

Run the script again…

![_config.yml]({{ site.baseurl }}/images/1_sdazhLYv46TOKvpZkj8Lxw.png)

Perfect ! :)

The output above cannot be decoded by QR Code reader tough. To produce high quality QR Code, we need to use image library:

```python
from PIL import Image
img = Image.new( 'RGB', (210,210), "white")     #210 * 210 pixels
pixels = img.load() 

f = open("qrcodes.txt", "r")
for i in range(21):
    for j in range(21):
        x = f.readline()
        for k in range(10):
            for l in range(10):
                if (x in ["96ee0cb5\n","5b808910\n","81f9bf5a\n"]):
                    pixels[j*10+k,i*10+l] = (255, 255, 255)
                else:
                    pixels[j*10+k,i*10+l] = (0, 0, 0)
img.show()
```
