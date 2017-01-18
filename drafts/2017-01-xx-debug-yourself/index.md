CUSTOMER, TO BUG, TO DEVELOPMENT TEAM, TO FIX, TO TEST Every application that exists has bugs. Why ? Because humans are creating them. Humans are not perfect. Thus, applications are not perfect (that makes sense, right?). Stay calm and relax. Take your coffee and let's go. 

  * don't be scared to go to the third party libraries source code you are using. lots of good stuff.
  * put new person to develop/enhance a feature. if he doesn't understand anything, it means this part needs improvement.

  * if you don't know how to do it now, you won't tomorrow. (sometimes you found the answer in the bathroom, what, no ?)

debug client [ debugger, breakpoint, console.log, console.table debug server Remove dead code please. Remove it. identifier client side or server side (+ identifier database ou code) vous pouvez parler à votre chat si vous préférez. Talk aloud protocol "participants merely report how they go about completing a task" To not be confused with the duck typing based on the duck test "If it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck". Visual Studio : make sure the project is running in debug mode and starts (right click on solution : projects startup). If the bullet is still blank, restart VS. It happens! Split your function the more you can. Modularize your thing with proper names. Don't forget to test object (null?). Make a first shot, simple, with a simple design that will work for sure, then do some iterations to add more stuff on it. Don't forget to remove hardcoded things! When you checked in, always check every line you are commiting. You can find things you forgot to remove, see that you missed something, or even improve what you did. The less input and output a function have, the better is. (and the less bugs you can find) We are in 2014, use OOP. Don't do hundred of static/Shared functions with a lot of arguments everywhere. Webservices can make us force to create a static private somewhere to handle or split specific process. It's fine to have small method like this in a Tools or Manager class with some static. But don't start to create a static that needs another that needs two others etc. You are going to regret it. Create classes in their own file like you know, it will save you time. Try to not use strings anywhere, use enum. We make typos, a string with a specific value should be written only once. Let the compiler does its job and check if you did a mistake. Don't try to create the perfect function when beginning. Think stable and reliable, not fast but unstable and unmaintenable. Always think if you are hit by a truck tomorrow, someone else will eventually need to understand what you did and fix it ! I hope that won't happen of course. If you have minified code, maybe it changes how the function is working (like in Angular and the Dependency Injection system based on the parameter name). For instance, we had an issue with the 'this' in a jQuery $.each was different between the non-minified and the minified version. I don't remember exactly why but it was causing an issue! In Javascript (and Chrome mainly), don't forget about the breakpoints, and the console.log/warn/error (and why not some alert() for fun), and console.table when dealing with collections. Or you can totally JSON stringify the object you want to inspect: JSON.stringify(obj,null,2) (null,2 is to have the output indented). When dealing with network, architecture, don't forget about security and permissions. This often leads to solution.