# Text Console Tips & Tricks

## Debug messages in the console
If you see a lot of stuff flying on the screen in Text Console/ssh that looks like this

    0000017C:1=[61][76][6D][75][74][65][0A]
    0000017A:1=[61][76][6D][75][74][65][0A]
    0000017E:1=[61][76][6D][75][74][65][0A]
    0000017B:1=[61][76][6D][75][74][65][0A]
    0000017C:1=[75][73][61][67][65][20][6C][69][67][68][74][20][68][6F][75][72][73][0A]
    0000017A:1=[75][73][61][67][65][20][6C][69][67][68][74][20][68][6F][75][72][73][0A]

It's because you have SIMPL Debugger running or it was running and didn't close properly. Enter `DBGSIGNAL RESET` to stop debugging on the processor (you can type even as the debug messages fly by)

## Software Licenses
* Check the status of the SW-MOBILITY license trial period (60 days) on a control processor: `crestrononetpr`
* Show software licenses installed: `showlicense`