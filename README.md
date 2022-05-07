# Migrating WhatsApp messages from android to iOS 

Info below is current as of December 2021. Please read the whole guide before performing any steps.

## Introduction

This is a guide to migrating WhatsApp messages from Android to iOS using open source software for free.

Currently no official method exists for migrating historical messages in WhatsApp when moving from an android to iOS device.

There is an [official method for migrating from iOS to Samsung android phones](https://faq.whatsapp.com/general/chats/how-to-migrate-your-whatsapp-data-from-iphone-to-a-samsung-phone) using Samsung's SmartSwitch utility. Android > iOS is apparently a [work in progress for the WhatsApp team](https://faq.whatsapp.com/general/account-and-profile/about-changing-phones/?lang=en) but there is no ETA provided.

3rd party utilities exist to migrate WhatsApp messages from Android to iOS such as Dr. Fone, MobileTrans, Mobitrix, BackupTrans however these are all paid solutions, closed source, and have ambiguous privacy policies that mean these tools cannot be trusted that they are not exporting whole copies of your messages.

The steps below go through getting the decrypted message database from an android device through backing up an iPhone locally, modifying the whatsapp backup and restoring the backup to the iOS device.

## Limitations

- Activating whatsapp on a new device will prevent viewing any messages on a previous device
- Currently, only messages are transferred. Media files received a replaced by placeholder `<Image>` message
- Historical messages in groups chats will appear immediately on transfer by clicking on the group, historical messages from individuals will not appear as conversations on the homepage until you send/receive a message to/from that person. Messages from individuals can be always found by search.
- Performing the below steps will erase and restore the iPhone, it is best to do this early on in the setup process of your new device. 


## Requirements

- [adb](https://dl.google.com/android/repository/platform-tools-latest-windows.zip) tools or full [Android studio](https://developer.android.com/studio) - adb tools should be added to path
- [allow USB debug access from android device](https://www.xda-developers.com/install-adb-windows-macos-linux/)
- [watoi](https://github.com/residentsummer/watoi)
- python3
- Access to macOS with xcode installed (Genuine mac or [virtual machine](https://techrechard.com/install-macos-monterey-on-virtualbox-on-windows-pc/))

## Steps

### *Step 1 - Obtain a copy of msgstore.db.crypt14 + key from the Android device*

The message database is user accessible on android but encrypted with AES-256.
The encryption key for WhatsApp is stored with in a protected portion on the android device. 
This key is created at install and is calculated based on your phone number (therefore does not change between installations).

**a. If your android device is rooted, connect to pc and run the following**
```bash
mkdir WhatsApp-Migration
cd WhatsApp-Migration

adb root 
adb pull /sdcard/Android/media/com.WhatsApp/WhatsApp/Databases/msgstore.db.crypt14
adb pull /data/data/com.WhatsApp/files/key
adb pull /data/data/com.WhatsApp/databases/wa.db 
```

**b. If your Android device is not rooted (method 1 - using old WhatsApp APK)**

[try this guide here, however i did not have any success with it](https://gist.github.com/proxium/ee76c9a599a55e6d50b31e8159f66b39)  

**c. If your Android device is not rooted (method 2 - Using Android Emulator)**

1. Backup WhatApp on your Android device to Google Drive
2. [Install Android Studio](https://developer.android.com/studio)
3. [Download the WhatsApp APK (any version)](https://www.apkmirror.com/apk/WhatsApp-inc/WhatsApp/)
4. In Android Studio - Tools > AVD Manager
5. Create an Android virtual device (AVD) - Select an image with Google APIs, but without Google Play to be able to access as `root`
6. Start the AVD
7. Install WhatsApp APK (drag and drop the APK file onto the virtual device) 
8. Open and activate WhatsApp on the AVD
9. Restore from the Google Drive backup, this will require signing into your Google Account on the virtual device.
10. Once restored, connect with adb as per root instructions above and retrieve the db and key (The virtual device will have root access provided you followed step 5)

### *Step 2 - Decrypt msgstore.db.crypt14*

Perform from within the same directory as your msgstore.db.crypt14 and key.

```bash
pip install pycrypto pycryptodome
git clone https://github.com/ElDavoo/WhatsApp-Crypt14-Decrypter
python3 ./WhatsApp-Crypt14-Decrypter/decrypt14.py key msgstore.db.crypt14 msgstore.db   
rm -rf WhatsApp-Crypt14-Decrypter

mkdir original-files
mv msgstore.db.crypt14 original-files
mv wa.db original-files
mv key original-files
```

### *Step 3 - Create local iPhone backup*

Recent versions of iTunes on macOS don't have an option for local backup. Either install an older version of iTunes or use the Apple Configurator 2 App to backup. Backups can be created on macOS or Windows with iTunes but all further steps need to be done on macOS. 

1. Install WhatsApp from the Appstore and activate WhatsApp
2. Create a local backup with iTunes/or Apple Configurator 2
3. **If using Windows:** Copy the contents of `%APPDATA%\Apple Computer\MobileSync\Backup` to `~/Library/Application\ Support/MobileSync/Backup` on macOS
4. Make a backup of the backup files and get the Backup ID (folder name within Backup directory)

```bash
git clone https://github.com/residentsummer/watoi
watoi/scripts/bedit.sh list-backups
export BACKUP_ID="put ID of the backup here" # use the latest ID from the list-backups command
cp -R ~/Library/Application\ Support/MobileSync/Backup/${BACKUP_ID} original-backup
```

### *Step 4 - Get a copy of the WhatsApp.ipa file*

You will need a copy of WhatsApp.ipa of same version you have on your device. You can get this from the internet or following [this guide](https://dev.to/tranthanhvu/how-to-download-ipa-file-from-appstore-37im) to obtain via Apple Configurator 2, the steps of which are summarised below. 

1. Update WhatsApp on the iPhone
2. Install and open Apple Configurator 2 (AC2)
3. Connect your phone
4. Right click on the phone in the AC2 window and click add > apps > WhatsApp
5. When prompted that WhatsApp already exists don't click anything and open finder
6. Copy the ipa file from `~/Library/Group Containers/K36BKF7T3D.group.com.apple.configurator/Library/Caches/Assets/TemporaryItems/MobileApps/` to your working directory
7. Cancel the previous prompts and close AC2

### *Step 5 - Extract the contents of WhatsApp.ipa*

```bash
unzip ./Whatsapp.ipa -d app
```

### *Step 6 - Extract the message database from the iPhone backup*

The commands below extract the message files from the backup and important manifest files.

```bash
export ORIGINALS="originals/$(date +%s)"
mkdir -p $ORIGINALS
watoi/scripts/bedit.sh extract-chats $BACKUP_ID $ORIGINALS/ChatStorage.sqlite
watoi/scripts/bedit.sh extract-blob $BACKUP_ID Manifest.db $ORIGINALS/Manifest.db
cp $ORIGINALS/ChatStorage.sqlite ./ChatStorage.sqlite
```

### *Step 7 - Run the migration*

```bash
xcodebuild -project watoi/watoi.xcodeproj -target watoi
watoi/build/Release/watoi msgstore.db ./ChatStorage.sqlite app/Payload/WhatsApp.app/Frameworks/Core.framework/WhatsAppChat.momd
```

### *Step 8 - Replace the database file in the backup*

```bash
scripts/bedit.sh replace-chats $BACKUP_ID ./ChatStorage.sqlite
```

### *Step 9 - Restore from backup*

Restore the backup with iTunes/Apple Configurator 2.  

> **_Note for virtual machine users:_** I had considerable issues getting my iPhone to be detected by Apple Configurator 2 when restoring the backup. Apple Configurator was stating my device was already provisioned and then not detecting the device at all after a factory restore. I created a backup using iTunes in Windows, transferred this to the VM using a shared folder. Once the backup was modified i transferred the backup to Windows again and restored with iTunes.

### *Optional step - Use whatsapp-viewer to view whatsapp messages on pc*

Browse your msgstore.db file on your pc with [whatsapp-viewer](https://github.com/andreas-mausch/whatsapp-viewer) if you need to view these at a later date or just have no success in patching the iPhone backup and just want a way to see the messages.  
Using the wa.db file will show contact names along with the messages.

## Acknowledgements
This guide uses these fantastic tools:
- https://github.com/residentsummer/watoi
- https://github.com/ElDavoo/WhatsApp-Crypt14-Decrypter
- https://github.com/andreas-mausch/whatsapp-viewer

Also was inspired by these guides:
- https://github.com/tim25651/WhatsApp2iOS
- https://gist.github.com/proxium/ee76c9a599a55e6d50b31e8159f66b39
- https://dev.to/tranthanhvu/how-to-download-ipa-file-from-appstore-37im

I had also tried these tools but could not get them to work - your milage may vary:
- https://github.com/YuvrajRaghuvanshiS/WhatsApp-Key-Database-Extractor

## Useful Paths
- private Android WhatsApp data - ` data/data/com.whatsapp/files/key`
- Android WhatsApp data - `/sdcard/Android/media/com.whatsapp/`
- Windows iTunes backup path - ` %APPDATA%\Apple Computer\MobileSync\Backup`
- macOS Apple Configurator 2 backup path - `~/Library/Group Containers/K36BKF7T3D.group.com.apple.configurator/Library/Caches/Assets/TemporaryItems/MobileApps/`

## Disclaimer + License
All of the above steps are at your own risk. This is a fiddly process and took me many attempts. Make sure you make backups along the way, i take no responsibility if you lose data performing any of these steps.

This guide is provided under the GPLv3 License.  

All tools used are provided under their own license terms check the project pages for details. 

```
Copyright (C) 2021 needs-coffee

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.
```
