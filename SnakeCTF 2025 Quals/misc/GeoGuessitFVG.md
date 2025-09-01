# GeoGuessitFVG

> **Found with Redistan**, **[Writeup by: Redistan](https://github.com/RiccardoCherchi)**

<img width="611" height="532" alt="Image" src="https://github.com/user-attachments/assets/4146241b-6bdc-4ae1-9460-2355180d48a8" />

**Attachment:** [geoguessitfvg.zip](https://github.com/user-attachments/files/22083768/geoguessitfvg.zip)

## Writeup

This was a GeoGuessr-style challenge. The authors gave us a video, likely from a dashcam, showing a vehicle driving along what appeared to be a _provinciale_ road (you could tell by the type of asphalt and the roadside posts). We already knew the correct location had to be somewhere near Udine and the surrounding area, since the GeoGuessr website for this challenge restricted both the movement and the zoom to that region.

<img width="1480" height="784" alt="Image" src="https://github.com/user-attachments/assets/e811a1a4-e56f-4bd3-9883-5f54da323797" />

Looking at the video frame, we noticed two power lines with towers crossing the road. With that clue, we searched for power lines in the area using **[OpenInfraMap](https://openinframap.org?utm_source=chatgpt.com)**:

<img width="1568" height="892" alt="Image" src="https://github.com/user-attachments/assets/810e1d61-3c75-4819-a81d-8a7b6d02a9bb" />

<img width="870" height="688" alt="image" src="https://github.com/user-attachments/assets/7cfe8e20-f1c3-4885-b7c5-21034ca58ee4" />

We found a section where two major power lines converge. From there, we started checking the roads that pass underneath these lines and eventually discovered the exact spot: **Via Podgora**.

<img width="2040" height="1086" alt="image" src="https://github.com/user-attachments/assets/e7a794a2-cc05-48ea-8919-18d6cf536766" />

<img width="2824" height="1560" alt="image" src="https://github.com/user-attachments/assets/8a416939-7e37-42bf-9234-5d0206d8ece5" />

Flag: **snakeCTF{Ov3r_9000_v0lts_e9253fd28b4c08e4}**
