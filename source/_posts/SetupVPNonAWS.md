---
title: Deploy VPN on AWS
date: 2020-01-15 11:48:28
tags: [VPN, AWS]
categories: Tech
description: The simplist way to setup a VPN in 15 minutes.
toc: true
---

## Deploy VPN on AWS

[Algo](https://github.com/trailofbits/algo) provides an opportunity to setup personal VPN easily, which allows people to deploy personal VPN sever on AWS for 1 year freely.

<!--more-->

### Setp 0: setup AWS account
1. Head to the Amazon Web Services site and create a free account. You can link your current Amazon account to your web services account if you want.
2. Once you’re logged in, Click Services > IAM. It’s located under the Security, Identity, & Compliance tab.
3. Click the Users tab on the left.
4. Click Add User.
5. Create a user name, then click the box next to Programmatic Access. Then click Next.
6. Click Attach existing policies directly.
7. Type in “admin” to search through the policies. Find “AdministratorAccess” and click the checkbox next to that. Click Next when you’re done.
8. On the final screen, click the Download CSV button. This file includes a couple numbers and access keys you’ll need during the Algo set up process. Click Close and you’re all set.

### Step 1: deploy VPN on AWS with algo
1. Download Algo `git clone https://github.com/trailofbits/algo.git`.
2. Download requirements `cd Algo && python3 -m pip install --user --upgrade virtualenv && python3 -m virtualenv env && source env/bin/activate && python3 -m pip install -r requirements.txt`.
3. Setup personal configures `vim config.cfg` and change **users** setting in line 7 or just ignore it.
4. Deploy Algo on AWS EC2  `./algo` and then choose options `3. Amazon EC2` 
5. Input AWS Access Key and AWS Secret Key in the CSV file that was downloaded when setup AWS account.
6. Choose several `y`s to setup Aglo Server on AWS.
7. Just choose America region instead of Asian, since Tokyo didnot succeed in my machine and Hongkong didnot active by default on AWS.

### Step 2: setup VPN on clients
There are two ways to setup clients, the newer is **WireGuard**, which are now recommended on github page and **strongSwan**, which are often used in linux.

#### WireGuard approach
Certificates and configuration files that users will need are placed in the configs directory. Make sure to secure these files since many contain private keys. All files are saved under a subdirectory named with the IP address of your new Algo VPN server.

##### Apple Devices

WireGuard is used to provide VPN services on Apple devices. Algo generates a WireGuard configuration file, wireguard/<username>.conf, and a QR code, wireguard/<username>.png, for each user defined in config.cfg.

On iOS, install the WireGuard app from the iOS App Store. Then, use the WireGuard app to scan the QR code or AirDrop the configuration file to the device.

On macOS Mojave or later, install the WireGuard app from the Mac App Store. WireGuard will appear in the menu bar once you run the app. Click on the WireGuard icon, choose Import tunnel(s) from file..., then select the appropriate WireGuard configuration file.

On either iOS or macOS, you can enable "Connect on Demand" and/or exclude certain trusted Wi-Fi networks (such as your home or work) by editing the tunnel configuration in the WireGuard app. (Algo can't do this automatically for you.)

Installing WireGuard is a little more complicated on older version of macOS. See Using macOS as a Client with WireGuard.

If you prefer to use the built-in IPSEC VPN on Apple devices, or need "Connect on Demand" or excluded Wi-Fi networks automatically configured, then see Using Apple Devices as a Client with IPSEC.

##### Android Devices

WireGuard is used to provide VPN services on Android. Install the WireGuard VPN Client. Import the corresponding wireguard/<name>.conf file to your device, then setup a new connection with it. See the Android setup instructions for more detailed walkthrough.

##### Windows

WireGuard is used to provide VPN services on Windows. Algo generates a WireGuard configuration file, wireguard/<username>.conf, for each user defined in config.cfg.

Install the WireGuard VPN Client. Import the generated wireguard/<username>.conf file to your device, then setup a new connection with it.
##### Linux WireGuard Clients

WireGuard works great with Linux clients. See this page for an example of how to configure WireGuard on Ubuntu.


#### Swan approach

##### Apple Devices

Inside the **configs** folder, you’ll find a .mobileconfig file. On Mac, double-click that file to install the profile on your Mac. To install the profile on an iPhone or iPad, you can either Airdrop that same file from your Mac to your iOS device, email it to yourself, or upload it to cloud service like iCloud or Dropbox and open it from there. You’ll be asked to confirm the profile installation, and from then on, you’ll be connected to that VPN. You can disconnect by simply deleting the profile.

##### Android Devices

On Android, you need to first install the strongSwan VPN Client app. Then, copy the P12 file inside the Configs folder over to your Android device and open it in strongSwan. Follow the directions from there to set it up. If you need help, this guide will walk you through each part.

##### Windows
1. Head to the **configs** folder, then copy the PEM, P12, and PS1 files to your Windows machine.
2. Double-click the PEM file to import it to the Trusted Root certificate store.
3. Open the Powershell application, then navigate to the folder with the files you copied in step one a second ago.
4. Type in, Set-ExecutionPolicy Unrestricted -Scope CurrentUser and press Enter.
5. Type in the name of your Powershell script and press Enter. This will be something like windows_$usernameyoumadeup.ps1. Follow the directions on screen.
6. Finally, when that’s complete type in Set-ExecutionPolicy Restricted -Scope CurrentUser and press Enter.

