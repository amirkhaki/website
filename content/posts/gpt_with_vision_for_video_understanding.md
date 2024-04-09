---
title: " تولید روایت صوتی برای ویدیو"
description: "این Notebook نشان می‌دهد چگونه می‌توان از توانایی‌های بصری GPT-4 برای درک محتوای یک ویدیو و تولید متن متناسب با آن و نهایتا تبدیل متن تولید شده به صدا استفاده کرد. GPT-4 به طور مستقیم ویدیوها را به عنوان ورودی قبول نمی‌کند، اما می‌توانیم از قابلیت vision و طول کانتکست 128K برای توصیف فریم‌های این راهنما شامل دو مرحله است:"
tags:
- openai
- gpt-4
- gpt-4-vision
- text-to-speech
- video-processing
- gilas.io
- blog
weight: 2001
og_image: "/posts/gpt_with_vision_for_video_understanding/banner.png" 
---

{{< postcover src="/posts/gpt_with_vision_for_video_understanding/banner.png" >}}

## پردازش ویدیو با استفاده از GPT-4-Vision برای تولید متن مناسب و صداگذاری روی آن

این Notebook نشان می‌دهد چگونه می‌توان از توانایی‌های بصری GPT-4 برای درک محتوای یک ویدیو و تولید متن متناسب با آن و نهایتا تبدیل متن تولید شده به صدا استفاده کرد. GPT-4 به طور مستقیم ویدیوها را به عنوان ورودی قبول نمی‌کند، اما می‌توانیم از قابلیت vision و طول کانتکست 128K برای توصیف فریم‌های ثابت یک ویدیو در هر زمان استفاده کنیم. این راهنما شامل دو مرحله است:

۱- استفاده از GPT-4  برای دریافت توصیفی از محتوای ویدیو به صورت متن

۲- تولید صدای روایت از روی متن تولید شده با استفاده از GPT-4 و TTS API

{{< hint info >}}
**برای اجرای کدهای زیر ابتدا باید یک کلید API را از طریق پنل کاربری گیلاس تولید کنید.  برای این کار
ابتدا یک  [حساب کاربری جدید](https://dashboard.gilas.io) بسازید یا اگر صاحب حساب کاربری هستید [وارد پنل کاربری](https://dashboard.gilas.io) خود شوید. سپس، به صفحه [کلید API](https://dashboard.gilas.io/apiKey)  بروید و با کلیک روی دکمه “ساخت کلید API” یک کلید جدید برای دسترسی به Gilas API بسازید.**
{{< /hint >}} 

 ```python
from IPython.display import display, Image, Audio

import cv2  # We're using OpenCV to read video, to install !pip install opencv-python
import base64
import time
from openai import OpenAI
import os
import requests

client = OpenAI(
    api_key=os.environ.get(("GILAS_API_KEY", "<کلید API خود را اینجا بسازید https://dashboard.gilas.io/apiKey>")), 
    base_url="https://api.gilas.io/v1/" # Gilas APIs
)
 ```

## استفاده از توانایی‌های بصری GPT برای دریافت توصیفی از محتوای ویدیو

 ما از OpenCV برای استخراج فریم‌های یک ویدیوی آزمایشی با محتوای طبیعت که نمایشی از شکار یک بیزانس توسط گرگ‌ها را به تصویر می‌کشد استفاده می‌کنیم:

```python
video = cv2.VideoCapture("data/bison.mp4")

base64Frames = []
while video.isOpened():
    success, frame = video.read()
    if not success:
        break
    _, buffer = cv2.imencode(".jpg", frame)
    base64Frames.append(base64.b64encode(buffer).decode("utf-8"))

video.release()
print(len(base64Frames), "frames read.")


```
خروجی:

```text
618 frames read.
```



فریم‌ها را نمایش می‌دهیم تا مطمئن شویم که آن‌ها را به درستی خوانده‌ایم:

```python
display_handle = display(None, display_id=True)
for img in base64Frames:
    display_handle.update(Image(data=base64.b64decode(img.encode("utf-8"))))
    time.sleep(0.025)
```

خروجی:

<img src="/posts/gpt_with_vision_for_video_understanding/frame.png" alt="screenshot" style="width: 734px;"/>

حال که فریم‌های ویدیو را داریم، می‌توانیم آنها را از طریق Gilas API به GPT-4-Vision ارسال کنیم تا متن متناسب با آنها تولید شود. (توجه داشته باشید که نیازی به ارسال همه‌ی فریم‌ها برای درک GPT از اتفاقات در حال رخداد نیست):

```python
PROMPT_MESSAGES = [
    {
        "role": "user",
        "content": [
            "These are frames from a video that I want to upload. Generate a compelling description that I can upload along with the video.",
            *map(lambda x: {"image": x, "resize": 768}, base64Frames[0::50]),
        ],
    },
]
params = {
    "model": "gpt-4-vision-preview",
    "messages": PROMPT_MESSAGES,
    "max_tokens": 200,
}

result = client.chat.completions.create(**params)
print(result.choices[0].message.content)
```

خروجی:

```text
"🐺 Survival of the Fittest: An Epic Tale in the Snow ❄️ - Witness the intense drama of nature as a pack of wolves face off against mighty bison in a harsh winter landscape. This raw footage captures the essence of the wild where every creature fights for survival. With each frame, experience the tension, the strategy, and the sheer force exerted in this life-or-death struggle. See nature's true colors in this gripping encounter on the snowy plains. 🦬"

Remember to respect wildlife and nature. This video may contain scenes that some viewers might find intense or distressing, but they depict natural animal behaviors important for ecological studies and understanding the reality of life in the wilderness.
```

{{< hint warning >}} 
توجه: ورودی‌های داده شده و خروجی‌های تولید شده توسط مدل در این مثال به زبان انگلیسی هستند. برای تولید خروجی به زبان فارسی٬ کافی‌ست از مدل بخواهید که خروجی را به زبان فارسی تولید کند.
{{< /hint >}} 

## تولید صدای روایت ویدیو با استفاده از GPT-4 و TTS API 


حال که متن روایت آماده شده است می‌خواهیم آن را به سبک David Attenborough تبدیل کنیم. با استفاده از همان فریم‌های ویدیو، از GPT می‌خواهیم تا یک فیلمنامه کوتاه به ما بدهد:

```python
PROMPT_MESSAGES = [
    {
        "role": "user",
        "content": [
            "These are frames of a video. Create a short voiceover script in the style of David Attenborough. Only include the narration.",
            *map(lambda x: {"image": x, "resize": 768}, base64Frames[0::60]),
        ],
    },
]
params = {
    "model": "gpt-4-vision-preview",
    "messages": PROMPT_MESSAGES,
    "max_tokens": 500,
}

result = client.chat.completions.create(**params)
print(result.choices[0].message.content)
```

خروجی:

```text
In the vast, white expanse of the northern wilderness, a drama as old as time unfolds. Here, amidst the silence of the snow, the wolf pack circles, their breaths visible as they cautiously approach their formidable quarry, the bison. These wolves are practiced hunters, moving with strategic precision, yet the bison, a titan of strength, stands resolute, a force to be reckoned with.
As tension crackles in the frozen air, the wolves close in, their eyes locked on their target. The bison, wary of every movement, prepares to defend its life. It's a perilous dance between predator and prey, where each step could be the difference between life and death.
In an instant, the quiet of the icy landscape is shattered. The bison charges, a desperate bid for survival as the pack swarms. The wolves are relentless, each one aware that their success depends on the strength of the collective. The bison, though powerful, is outnumbered, its massive form stirring up clouds of snow as it struggles.
It's an epic battle, a testament to the harsh realities of nature. In these moments, there is no room for error, for either side. The wolves, agile and tenacious, work in unison, their bites a chorus aiming to bring down the great beast. The bison, its every heaving breath a testament to its will to survive, fights fiercely, but the odds are not in its favor. 
With the setting sun casting long shadows over the snow, the outcome is inevitable. Nature, in all its raw beauty and brutality, does not show favor. The wolves, now victors, gather around their prize, their survival in this harsh climate secured for a moment longer. It's a poignant reminder of the circle of life that rules this pristine wilderness, a reminder that every creature plays its part in the enduring saga of the natural world.
```

حال می‌توانیم فیلمنامه را با استفاده از TTS API به صدا تبدیل کنیم. برای این کار کافی است که یک درخواست به اندپوینت `https://api.gilas.io/v1/audio/speech` بفرستیم:

```python
response = requests.post(
    "https://api.gilas.io/v1/audio/speech",
    headers={
        "Authorization": f"Bearer {os.environ['GILAS_API_KEY']}",
    },
    json={
        "model": "tts-1-1106",
        "input": result.choices[0].message.content,
        "voice": "onyx",
    },
)

audio = b""
for chunk in response.iter_content(chunk_size=1024 * 1024):
    audio += chunk
Audio(audio)
```

خروجی:

<audio controls="controls">
                    Your browser does not support the audio element.
                </audio>