# How to Solve the Challenge?

In the first step we install the application on an android emulator. we can see that the application have no functionalities and there is no function available for adding notes:
<img width="1080" height="2400" alt="Screenshot_1770687734" src="https://github.com/user-attachments/assets/5a7aaaa3-43b9-4ac4-94c3-7c46f29b2946" />

so what is the next step? the next step is reverse engineering and understanding the flow of the application. we can reverse it using jadx-gui.
https://github.com/skylot/jadx

the first step of android reverse engineering is checking the AndroidManifest.xml file for gathering information about some core configurations and application components. so let's see what is inside this file:
<img width="1557" height="1371" alt="image" src="https://github.com/user-attachments/assets/537badb8-b22e-4cf3-a648-c8e1c3ae1e4a" />

