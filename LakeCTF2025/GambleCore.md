# GambleCore
> Let's go gambling!

## Background
The challenge presents as a generic casino which allows gambling of currency called "Microcoins"
<p align="left"> 
  <img src = "images/homepage1.jpg">
</p>
There is a Currency Exchange functionality and a Black Market functionality to obtain the flag. Thus, I immediately thought that the most likely solve is to 
somehow obtain enough USD to purchase the flag. But how?
  <img src = "images/homepage2.jpg">
</p>

Looking at the starting Microcoins balance of 10, it's pretty clear that its (almost) impossible to obtain the flag purely from gambling off our default money. 

Given that 1 coin = 1,000,000 microcoins, we'd have to successfully gamble 6 times in a row to obtain the flag. Looking at the code however, theres a 9% chance to win 
each gambling attempt. 
```
   // 9% chance to win
    const win = secureRandom() < 0.09;
    let winnings = 0;
```
This led me straight to the source code.

## References
https://dmitripavlutin.com/parseint-mystery-javascript/
