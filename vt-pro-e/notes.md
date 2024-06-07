# VTPro-E & Touch Panel Notes

## Screen Dimensions

### x70 Series

* TSx-1070 screen saver logo dimensions: 684x130
* TSx-1070: 1920x1200
* TSW-570: 1280x720
* TSW-570P: 720x1280
* TSx-770: 1280x800

### x60 Series

* TSW-560: 960x540
* TSW-760: 1024x600
* TSW-1060: 1280x800

### x50 Series
* TSW-550: 800x480
* TSW-552: 800x480
* TSW-750: 800x480
* TSW-752: 800x480
* TSW-1050: 1280x800
* TSW-1052: 1280x800

## Activity Detection
A media player object in a user project on a touch panel will register with the control processor every couple of minutes, which triggers the processor to believe there is user activity and will cause an Activity Detection extender to never report no activity.

## Themes
Compiled user themes go in `C:\ProgramData\crestron\Core3\UserThemes`. they are put there automatically when importing a VTA, but if you need to install one manually it goes there.

## Punctuation in File Names
Touch panel projects shouldn't have punctuation in them. If a touch panel shows a message like "load failed" briefly on the screen when you send a project to it, chances are there's some character in the file name that it doesn't like. This will not log an error in the event log.
