---
layout: post
title:  "Fix Kernel Error after installing jupyter in conda envs"
tag: python anaconda jupyter twequickfixepy
---

When building an env with python pandas numpy and jupyter, after starting the notebook a kernel error is displayed.
>  File "<PATH>", line 359, in win32_restrict_file_to_user
    import win32api
ImportError: DLL load failed: The specified procedure could not be found.

This can be solved by switching into the user/anaconda/envs/YOURENV folder and run
```
python [environment path]/Scripts/pywin32_postinstall.py -install
```
After restarting, everything should work alright.