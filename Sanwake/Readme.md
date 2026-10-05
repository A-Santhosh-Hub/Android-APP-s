🚀 Introducing SanWake — Smart Travel Wake-Up App

Hey guys! 👋

I’ve been working on a new Android app called SanWake.

The idea is simple:  
😴 Sleep during your bus/train journey without worrying about missing your destination.

Instead of setting a fixed-time alarm, SanWake uses your location and distance from your destination. You select where you want to get down and how far before the destination you want to be woken up.

For example:

📍 Destination: Erode Bus Stand  
🔔 Wake me at: 2 km before destination

SanWake continuously tracks the journey and automatically triggers the alarm as you get closer.

### What I've built so far

✅ Smart GPS-based destination tracking  
✅ Multiple wake-up stages  
✅ GPS accuracy & false-location protection  
✅ Movement and speed detection  
✅ GPS outage & recovery handling  
✅ Adaptive location sampling to reduce battery usage  
✅ Background/foreground journey tracking  
✅ Alarm + vibration + notification  
✅ Journey recovery after service/process interruption  
✅ Persistent journey & wake-stage state  
✅ Diagnostics to understand why the app did or didn't trigger  
✅ Lots of automated and real-device testing

This is still a testing/release-candidate version, so I need some real people to try it in actual travel conditions.

# 🧪 TODAY'S TEST

If you're travelling by bus/train, I’d really appreciate it if you could test SanWake during your journey.

Try using it normally—especially while you're resting or sleeping—and see whether it wakes you at the configured distance.

Please report anything you notice:

📍 GPS issues  
🔋 Battery usage  
🔔 Alarm behaviour  
📱 Screen/lock-screen behaviour  
🚨 False or missed alarms  
🚌 Behaviour during actual travel  
💥 Any crash or unusual behaviour

Important: Keep a normal alarm as a backup for now. This is still a test version.

Your feedback will help me find real-world problems that automated testing can't reproduce.

🙏 Thanks to everyone who helps test SanWake!

— SanStudio




How to run the test now
On Home, keep your destination (Erode, 11.34706, 77.71997).
Open the Settings tab and scroll to Developer & Diagnostics.
Tap Field Journey (validation).
Set the stage distances, for example 5000 / 3000 / 1000 (metres).
Tap Start field journey and allow the location and notification permissions.
You should see Journey: Active, the distance to Erode, all stages ARMED, and a "SanWake – Erode" notification.
This is the screen I tested on real phones, and starting from it works.
