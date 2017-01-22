---
layout: post
title: Unlocking My Door With a RaspberryPi
---

A couple of years ago, after losing my keys a couple of times and being locked out of my apartment, I started thinking "maybe there's a way I can unlock my door using the power of the internet". I searched the internet and found a couple companies, [lockitron](https://lockitron.com/) and [august](http://august.com/products/august-smart-lock/) who were making these devices, but they weren't shipping yet. After months of waiting, I got tired and thought (foolishly) I can make that.

I found a couple YouTube videos where people were unlocking their doors using a RaspberryPi. I wanted to unlock it using my phone, so I decided to pair the Pi with a service [Twilio](https://www.twilio.com/) I'd used recently to program my condo door-phone to unlock using a secret code (also after losing my keys) by following [this tutorial](http://www.daniellemorrill.com/2010/06/how-i-built-a-multi-user-door-buzzer-for-our-apartment/).

I bought a RaspberryPi kit, some wire, a sheet of metal from HomeDepot and a servo motor off the internet, and went to work. I wrote a small script that listened for the message "unlock" being sent to my Twilio number from users who were on my safe list, and moved the servo motor if it did. Then I duct taped the thing to my door. In order to get Twilio talking to the Pi from behind my router I used [ngrok](https://ngrok.com/).

Click [here](https://www.instagram.com/p/y3RwrMOCp-/?taken-by=seriousswann) to see it in action!

Here's the code:

```
from twilio.rest import TwilioRestClient

# Your Account Sid and Auth Token from twilio.com/user/account
"""account_sid = "#############"
auth_token  = "############"
client = TwilioRestClient(account_sid, auth_token)

message = client.messages.create(body="Hello Kitson - This is Pi",
    to="+18888888888",    # Replace with your phone number
    from_="+19999999999") # Replace with your Twilio number
print message.sid"""

from flask import Flask, request, redirect
import twilio.twiml
import RPi.GPIO as GPIO
import time

app = Flask(__name__)

# Try adding your own number to this list!
callers = {
    "+16045068519": "John",
    "+16043767388": "Billy",
    "+16042501029": "Bob",
}

@app.route("/", methods=['GET', 'POST'])
def hello_monkey():

    sms = request.values.get('Body', None)
    from_number = request.values.get('From', None)

    if (from_number in callers and sms == 'unlock'):

        """Move The Servo Motor To Open Position"""
        GPIO.setmode(GPIO.BCM)
        GPIO.setup(18, GPIO.OUT)
        pwm = GPIO.PWM(18, 50)
        pwm.start(0)
        pwm.ChangeDutyCycle(12.5)
        time.sleep(1)
        pwm.stop()
        GPIO.cleanup()
        message = callers[from_number] + ", the door is now open!"
    elif (from_number in callers and sms == 'lock'):

        """Move The Servo Motor"""
        GPIO.setmode(GPIO.BCM)
        GPIO.setup(18, GPIO.OUT)
        pwm = GPIO.PWM(18, 50)
        pwm.start(0)
        pwm.ChangeDutyCycle(2.5)
        time.sleep(1)
        pwm.stop()
        GPIO.cleanup()
        message = callers[from_number] + ", the door is now locked!"
    else:
        message = "Access Denied!"

    resp = twilio.twiml.Response()
    resp.message(message)

    return str(resp)

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```
