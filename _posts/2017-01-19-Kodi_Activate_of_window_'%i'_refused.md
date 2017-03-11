
Kodi Debug Log.
```
11:00:50 T:1733399232   DEBUG: Activating window ID: 11108
11:00:50 T:1733399232    INFO: Activate of window '11108' refused because there are active modal dialogs
11:00:50 T:1733399232   DEBUG: ------ Window Deinit (DialogProgress.xml) ------
```

谷歌搜索后在Kodi的bug跟踪系统中找到：

http://trac.kodi.tv/ticket/15960  

摘录解决问题的关键：  


This problem is becoming a major issue with the Isengard builds. Any dialog that tries to open another window no longer works because of the modal dialog check (ie- using playercontrols.xml to open visualization or full screen video). I find myself having to put <onclick>Dialog.Close(all,force)</onclick> all over the place in my skin in order to compensate, which defeats the purpose of the check to begin with.

Also, I don't think the failure of the modal check should not be relegated to an INFO entry in the debugging log. It should show up as a WARNING at minimum. Having it only show up as an INFO makes it very difficult to diagnose.

Last edited at 2015-07-01T03:52:19+01:00 by ZexisStryfe

