---
layout: post
category: tool
title: Kindle Tips
---

# Introduction
Normally, you can use your authorised mail account to send an email with attachment to your Kindle mail address, like mykindle@kindle.com(registered at Amazon). 

And if you titled this mail with 'convert', Amazon will convert your attachment to Kindle format (.azw) automatically. After that, you can sync converted file to your Kindle device when connecting wifi.

However, it is boring to send an email every time. Here is a simple way to do that routine thing. Thanks for the interesting website [ifttt](https://ifttt.com/), a great Internet Robot. You only need to create a recipe:

![image](/assets/img/post/ifttt_recipe.png)

# Preparation
* Kindle mail address
* Authorised mail
* [Dropbox](https://dropbox.com)
* [ifttt](https://ifttt.com/) account

# Howto
* Create a public directory under your Dropbox directory, such as **Dropbox/Public/sendToKindle/**
* Login your *ifttt* account, **activate** dropbox and gmail channels. Then create a new recipe:
  * click ***create*** hyperlink
  * click ***this*** hyperlink for trigger, and choose channel dropbox. Fill your directory
  ![image](/assets/img/post/ifttt_trigger.png)
  * click ***that*** hyperlink for action, and choose channel gmail (my authorised mail is gmail)
     1. fill your kindle mail address
     2. fill title with 'convert'
  ![image](/assets/img/post/ifttt_action.png)

# Enjoy
* Drag your file to **Dropbox/Public/sendToKindle/**
* Wait ifttt recipe to be triggered, it might spend a few minutes
* Open your Kindle device, enjoy the converted file that you dragged to dropbox just now
