### chall name: Christmas Lottery
hex advent day 12   
   
we are given an apk: christmas-lottery.apk  
  
first i decompiled the apk with jadx  
from com/christmas/lottery/christmaslottery and mainactivity, and some other helper functions, heres what the flow looks like:  
```
signInAnonymously()
    ↓
Receive Firebase UID
    ↓
Get ID Token (JWT containing UID)
    ↓
POST /init with idToken
    ↓
Backend Flow:
    1. Verify JWT signature
    2. Extract UID from token
    3. Generate/retrieve seed for this UID
    4. Return seed: { "seed": 1216949965 }
    ↓
Store seed in MainActivity.secretSeed  
    ↓
generate ticket  
```
  
didnt realise the android app dev stuff in sch wld ever be helpful but yay ig    
  
but also uh i didnt know about frida which is:  
'In cybersecurity, **Frida** is a powerful, open-source **dynamic instrumentation toolkit** that lets security researchers and developers inject JavaScript or their own libraries into running applications (on Windows, macOS, Android, iOS, etc.) to inspect and modify their behavior in real-time'  
  
basically, this is how to setup:  
run app apk with emulator in android studio  
  
on host:  
``` sh 
adb push frida-server /data/local/tmp/
adb root
adb shell
```
  
in the shell:  
``` sh 
cd /data/local/tmp
chmod 755 frida-server
./frida-server &
```
  
on host:   
``` sh
frida -U -f com.christmas.lottery -l script.js
```
  
script to:  
hook into the app at runtime,  
intercept the random seed used,  
then recreate the same ticket generation logic  
``` js
Java.perform(function () {    
    function generateTicket(seed) {
        // beautifully translated code by deepseek 
        var alphabet = "ABCDEFGHJKLMNPQRSTUVWXYZ";
        var digits = "0123456789";
        var ticket = [];
        
        for (var i = 0; i < 3; i++) {
            var index = (seed >> (i * 3)) % 24;
            ticket.push(alphabet.charAt(index));
        }
        ticket.push('-');
        
        for (var i = 0; i < 4; i++) {
            var index = (seed >> ((i * 4) + 9)) % 10;
            ticket.push(digits.charAt(index));
        }
        ticket.push('-');
        
        for (var i = 0; i < 2; i++) {
            var index = (seed >> ((i * 5) + 25)) % 24;
            ticket.push(alphabet.charAt(index));
        }
        
        var code = ticket.join('').replace(/-/g, '');
        var sum = 0;
        for (var i = 0; i < code.length; i++) {
            var c = code.charAt(i);
            if (c >= '0' && c <= '9') {
                sum += parseInt(c);
            } else {
                sum += c.charCodeAt(0) - 64;
            }
        }
        ticket.push(digits.charAt(sum % 10));
        
        return ticket.join('');
    }
    
    var JSONObject = Java.use("org.json.JSONObject");
        
    JSONObject.getLong.overload('java.lang.String').implementation = function(key) {
        var x = this.getLong(key);
        
        if (key === "seed") {
            var seedStr = x.toString();
            
            console.log("seed: " + x);
                var ticket = generateTicket(x);
                console.log("ticket: " + ticket);
        }
        
        return x;
    };
});
```
  
then input the ticket, and on success, u get toast:  
Valid ticket! Flag: `HEX{Fr1d4_i5_S4nT4s_b3sT_fRi3nD}`  
heheh  