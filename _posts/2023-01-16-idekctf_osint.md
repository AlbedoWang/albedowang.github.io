---
title: 2023idekCTF Osint/Osint Crime Confusion 1:W as in Where
date: 2023-01-16
tags: 
  - Osint
permalink: /posts/2023/01/2023idekCTF/
---

## Osint Crime Confusion 1:W as in Where


First look at the tips given on the website:  

Someone has died unexpectedly. The police is on it, but between you and me, I cannot wait for the police. I am a private investigator and I need your help. Unfortunately, we might be tracked so I cannot give you the information directly. Start in a major social network. Certainly not a problem for the best hacker I know right...? Alright here goes a beautiful poem:

*Some people in weird ways were connected
Some were a triangle, some were less directed
For one night they all met
At doctor's Jonathan Abigdail the third they wept 
Things were said, threats in the air
A few days later someone is dead
Who is that someone? That is for you to find,
Also who is the killer, if you really don't mind.*

So I directly searched doctor's Jonathan Abigdail the third on Google and found that I couldn't find anything. Then I searched for dr.Jonathan Abigdail III and luckily found a relevant Instagram account.*https://www.instagram.com/abigdail3djohn/*

I clicked in and found that this person only posted a picture of eyes and a description of the picture:  

*Now imagin you could INK IT!
That is right, at the famous convention for the eye yours truly is presenting!
Hopefully reunited with a lot of old friends to see it!
The hashtag is #TheEye12tothe3isthekeytoBEyousee?
Get HYPEEEED*

Next, I clicked on the tag *#TheEye12tothe3isthekeytoBEyousee* and found the account of another person *https://www.instagram.com/hjthepainteng/*. The account described him as coming from a university that looked suspicious:  

*Study and Teached blue birds at the University of Dutch ThE of Topics in Science (UThE_TS)*

Combining the clues I found in social media before, I guessed that *blue birds* here refers to twitter, so I went to twitter to search for UThE_TS and found an account *https://twitter.com/UThE_TS*, and then There is no more. According to the description of confusion 2, this account should be useful in the second question, and probably not useful in the first question.  

Go back to Instagram and look at another photo of water droplets posted by the person just now (@hjhepainteng). Photo description:  

*I do not know what happened, only 3 days after that stupid eye convention you appear dead. Only if there was someone that could find what happened. I only hope you know that you died somewhere after the best performance ever at the great_paintball_portugal competition. I write this still there, arranging for moving your body back home. Farewell. I love you. Also they said they would sell something on ebay for you <3*

The key information captured should be great_paintball_portugal and ebay, so I went directly to eBay to search for great_paintball_portugal, but found nothing. Then I went to Google to search for great_paintball_portugal, and found an eBay link in the search results at the bottom: https://www.ebay. com/usr/great_paintball_portugal, click on it and find:  

*About
So after the death we actually decided instead of selling just to make a little rip post at https: franparrefrancisco.wixsite.com/great-paintball-pt. We do not want the blog post to be very obvious though because of the publicity.
Location: Portugal
Member since: May 12, 2022*

So I opened the website *https://franparrefrancisco.wixsite.com/great-paintball-pt* and found that there was nothing there. There was no response when I clicked learn more, and I couldn’t find any useful information in the search box. Then I thought about the sentence *We do not want the blog post to be very obvious though because of the publicity.*, and then tried searching for rip, blog and post, but there was no result. Finally, I tried to change the URL to *https://franparrefrancisco.wixsite.com/great-paintball-pt/post*, and got a hidden web page with the following content:  

*Death at the Great Paintball Portugal of Heather James
Yes, we are sad to confirm that yesterday one athlete by the name Heather James was killed. Authorities are investigating as we speak as are YOU, the reader, I hope.

We confirm that it was indeed here at idek{TGPP_WCIYD}.*  
