---
title: "Notice of using headless chrome on Debian11"
date: 2023-09-28T19:56:24+08:00
draft: false
---

# Notice of using headless chrome on Debian11

When we run the application directly, the error may occured:

```
panic: [launcher] Failed to launch the browser, the doc might help https://go-rod.github.io/#/compatibility?id=os: /path/to/chrome: error while loading shared libraries: libnss3.so: cannot open shared object file: No such file or directory
...
```

This is because we are missing some dependencies. Please run this command to obtain all the dependencies:

```
sudo apt-get install ca-certificates fonts-liberation libasound2 libatk-bridge2.0-0 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgbm1 libgcc1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 lsb-release wget xdg-utils
```

Now you should be able to run your application.