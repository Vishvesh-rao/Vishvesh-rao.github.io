---
title: "PROJECTS"
order: 3
icon: fas fa-stream
date: 2021-06-15T17:55:28+08:00
lastmod: 2021-06-15T17:55:28+08:00
draft: false
author: "Vishvesh"
authorLink: "Vishvesh-rao.github.io"
description: "Video converter bot"

---

# YtoMp3 

[![Click to view code](https://codimd.s3.shivering-isles.com/demo/uploads/02a6da1fb9d19367162f4239e.jpeg)](https://github.com/Vishvesh-rao/ytomp3)

Simple video search bot for telegram that lets the user search for videos on youtube via inline command. Now you can take full advantage of telegrams unlimited free storage and built in mp3 player without having to worry about backing up your songs or loosing them!! Now with added support for mp4 format too !!

## Requirements [_self hosting_]

1. Inline telegram bot 
2. Bot Tokem from botfather
3. Youtube V3 API creds

## Steps for self hosting
- Clone the repo
- Run ```pip install -r requirements.txt```
- Create an inline bot using botfather ( look [here](https://core.telegram.org/bots/inline) if unsure on the steps )
- You will get a bot-token paste that in ***bot.py*** and ***mp3dldr.py*** and ***mp4dldr.py***
- you will need to get your API keys for youtube from [here](https://developers.google.com/docs/api/quickstart/python) 
- Put the API creds in ***yt_search.py***
- Run bot.py `python3 bot.py`

## Usage

- type `@botusername` in message field and type the video name (_might take a few sec for video to appear_ )
- Wait till the video appears and click on it
- press the `convert to mp3`/`convert to mp4` button which appears after that
- Wait for it to convert
- video will be downloaded


> PS: This bot is made purely for educational purposes and getting an understanding of the bot api. Certain songs are not meant to be downloaded do check TOS of youtube. 

&nbsp;[LINK TO REPO](https://github.com/Vishvesh-rao/ytomp3)

##########################################################################

# JAVA APP

[![Click to view code](https://codimd.s3.shivering-isles.com/demo/uploads/02a6da1fb9d19367162f4239d.png)](https://github.com/Vishvesh-rao/E-Hub-Software-company-database)

An implemetation of a software company database and with added functionalities. Backend implemented in sql and frontend created with java swing using JDBC to connect backend to frontend


link to [Project Documentation](https://drive.google.com/file/d/1CJerirHTqw2HPQ1wP8521Y9Gay244b9f/view?usp=sharing)

## Usage Instructions [for linux machines]
 Clone the repository copy the src/ folder to your local machine and add the sql dump to your databse

## Steps to recover db from sql dump

To recover db from a MySQL dump, enter:

`mysql -u [user] -p [database_name] < [filename].sql`

Make sure to include `[database_name]` and `[filename]` in the path.

It’s likely that on the host machine, `[database_name]` can be in a root directory, so you may not need to add the path. Make sure that you specify the exact path for the dump file you’re restoring, including server name (if needed).

&nbsp;[LINK TO REPO](https://github.com/Vishvesh-rao/E-Hub-Software-company-database)


