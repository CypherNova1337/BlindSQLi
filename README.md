# Blind SQL Injection: Unmasking Hidden Parameters and Bypassing WAFs

Web Application Firewalls (WAFs) are ubiquitous, standing guard against common attacks like SQL Injection (SQLi). While they block many straightforward attempts, attackers continuously refine techniques to slip past these defenses. Blind SQL Injection, where the application doesn't directly return database errors or query results, coupled with sophisticated WAF evasion, remains a potent threat.

Often, the key to a successful Blind SQLi doesn't just lie in payload crafting but in finding the *right place* to inject – specifically, parameters that aren't immediately obvious. Furthermore, even with a vulnerable parameter, getting your payload past the WAF requires careful setup and iteration.

This guide outlines a phased approach combining deep parameter discovery with robust WAF evasion techniques using popular open-source tools. This methodology is crucial for penetration testers and bug bounty hunters aiming to uncover vulnerabilities hidden behind modern defenses.

> **Disclaimer:** This information is for educational and ethical security research purposes only. Always obtain explicit permission before testing any system you do not own.

---

## Phase 1: The Hunt Begins - Discovery & Parameter Identification

Before exploitation, we need a comprehensive map of the application's attack surface, including parameters that might not be visible during normal Browse or basic spidering.

### 1. Gather URLs

The first step is quantity. Collect every URL associated with the target domain. Tools like `gau` (Get All URLs) or `waybackurls` excel at retrieving historical and known URLs from various sources. Basic application spidering with tools like `hakrawler` can supplement this.


### Example using gau - retrieves URLs including subdomains
```bash
echo "targetdomain.com" | gau --threads 5 --subs > urls.txt
```

### 2. (Optional) Pre-filter with gf

If you have a massive URL list, you can perform an initial, rough filter using gf (grep filter) with SQLi-specific patterns. This can help prioritize URLs that look like they contain injectable parameters based on syntax (?id=, &user=, etc.). However, be aware this might cause you to miss less obviously named parameters, which arjun (step 3) is designed to find.

Ensure gf patterns are installed ([https://github.com/1ndianl33t/Gf-Patterns](https://github.com/1ndianl33t/Gf-Patterns))

# Filter the gathered URLs for potential SQLi patterns
```bash
cat urls.txt | gf sqli > potential_sqli_urls.txt
```

### 3. Discover Hidden Parameters with Arjun
This is arguably the most critical step in this phase. Standard tools often miss parameters that aren't in common wordlists or aren't linked directly in the application's frontend code. Arjun specializes in finding these hidden gems by sending multiple requests with potential parameter names based on its extensive built-in wordlist and context analysis. Feed it the comprehensive URL list gathered earlier.

### Run Arjun on the full list of URLs to find hidden parameters
Adjust threads (-t) based on target responsiveness and your connection
```bash
arjun -i urls.txt -t 10 -oT arjun_results.txt
```

### Or run it on the pre-filtered list if you chose that path
```bash
 arjun -i potential_sqli_urls.txt -t 10 -oT arjun_results.txt
```

-i: Specifies the input file containing URLs.

-t: Sets the number of concurrent threads.

-oT: Outputs the results (URLs appended with discovered parameters) in text format.

Carefully review the arjun_results.txt file. This list now contains URLs potentially augmented with newly discovered parameters – prime candidates for injection testing.

## Phase 2: Setting the Stage for Evasion - WAF Bypass Prep
With potential injection points identified, the next hurdle is the WAF. We need to configure our attack traffic to mask its origin and obfuscate payloads.

### 1. Configure Proxychains
Routing sqlmap traffic through proxies (like Tor or a pool of residential/rotating proxies) is essential for several reasons: it masks your real IP, can circumvent IP-based blocking/rate-limiting, and makes your traffic harder to attribute. proxychains is a fantastic tool for forcing TCP connections made by other programs through a proxy server.


Edit your configuration file (often /etc/proxychains.conf or ~/.proxychains/proxychains.conf).
Ensure dynamic_chain or strict_chain is uncommented (dynamic is usually preferred as it skips dead proxies).
Add your proxy servers under the [ProxyList] section.
Ini, TOML

```bash
[ProxyList]
# type host port [user pass]
# Example using Tor (assuming Tor service is running on localhost:9050)
socks5  127.0.0.1 9050
# Example using a local HTTP proxy like Burp Suite or OWASP ZAP
# http    127.0.0.1 8080
# Example using a commercial SOCKS proxy
# socks4  proxy.example.com 1080 user password
# Add your list of reliable proxies here...
```
Test your chain: Run proxychains curl ipinfo.io/ip. The output should be the IP address of your exit proxy, not your own.

### 2. Select sqlmap Tamper Scripts
sqlmap comes with a powerful arsenal of tamper scripts designed to modify injection payloads in ways that confuse or bypass WAF pattern matching rules. The choice of scripts depends heavily on the specific WAF being targeted (e.g., Cloudflare, Akamai, F5). Based on common WAF bypass techniques, consider scripts like:

space2comment.py, space2plus.py, multiplespaces.py: Obfuscate spaces, a common WAF trigger.

randomcase.py: Change the case of SQL keywords (e.g., SELECT becomes SeLeCt).

chardoubleencode.py, charencode.py: Apply URL encoding (sometimes multiple layers) to characters.

equaltolike.py: Replace the = operator with LIKE.

unionalltounion.py: Avoid UNION ALL SELECT, using UNION SELECT instead.

apostrophemask.py, apostrophenullencode.py: Obfuscate single quotes.

between.py: Replace greater-than comparisons (>) with BETWEEN.

percentage.py: Insert % characters within keywords.

You can list all available scripts with sqlmap --list-tampers. Selecting the right combination is often an iterative process. Start with a few likely candidates based on the suspected WAF or general evasion principles.

## Phase 3: The Payload Delivery - Exploitation with Finesse
Now, combine the discovered parameters, proxy setup, and tamper scripts using sqlmap.

### 1. Run sqlmap via Proxychains
Construct your sqlmap command, prepending it with proxychains to route the traffic. Target a specific URL identified by Arjun, including the discovered parameters.

Example command targeting a URL from Arjun's output
```bash
proxychains sqlmap \
    -u "TARGET_URL_FROM_ARJUN_RESULTS?param1=value1&hiddenparam=value2" \
    --technique=B,T \
    --level=3 --risk=2 \
    --tamper=space2comment,randomcase,multiplespaces \
    --random-agent \
    --batch \
    --dbs
    --current-db
    --tables -D database_name
    --dump -T table_name -D database_name
```


### 2. Key sqlmap Options for Blind SQLi & WAF Evasion:

-u "URL": The full target URL, including parameters discovered by Arjun.

--technique=B,T: Crucial for Blind SQLi. Focuses tests on Boolean-based (B) and Time-based (T) techniques, which are necessary when direct output isn't available.

--level=(1-5): Increases the number and complexity of tests performed. Start around 3; increase to 5 if initial attempts fail or you suspect deeper vulnerabilities.

--risk=(1-3): Increases the potential "riskiness" of payloads. Higher risk might use more potentially harmful SQL statements but can sometimes be necessary for detection. Start around 2; increase cautiously.

--tamper=script1,script2,...: Apply the selected tamper scripts. Use a comma-separated list.

--random-agent: Use random User-Agent strings for requests, helping to bypass simple UA filtering rules.

--batch: Run non-interactively, accepting default answers. Use carefully, as it might make suboptimal choices. Better to run interactively first.

--dbs, --current-db, --tables, --dump: Standard sqlmap actions to enumerate and extract data once an injection is confirmed.

--flush-session: Clears sqlmap's session cache for the target. Useful if the WAF starts blocking you after initial successes.

--delay=SECONDS: Adds a delay (in seconds) between requests (e.g., --delay=0.5). Can help avoid rate-limiting.

--time-sec=SECONDS: For time-based techniques, specifies how long a delay must be to register as TRUE (default is 5). Increase if network latency is high or WAFs interfere (--time-sec=10).

--threads=N: Number of concurrent requests. Keep this low (e.g., 1-3) when dealing with WAFs and proxies to avoid triggering defenses and overwhelming your proxy setup.

--smart: If sqlmap detects WAF heuristics, attempt only tests declared as WAF-safe.

### Phase 4: Iterate and Analyze - The Feedback Loop


WAF bypass is rarely a one-shot process. You need to observe and adapt.


Monitor Output: Watch sqlmap's output closely. Look for HTTP error codes (especially 403 Forbidden), custom WAF block pages, or timeouts, which indicate detection.

Adjust Tampers: If blocked, change your tamper script combination. Add more, remove some, try different ones entirely.

Tune Aggressiveness: Increase --level and --risk incrementally if needed. Conversely, if you're getting blocked quickly, decrease --threads and add/increase --delay.

Check Proxies: Ensure your proxies listed in proxychains.conf are active and not blocked by the target or the WAF. Rotate them if necessary.

Try Other Parameters: If one parameter proves resilient or clean, move on to other parameters identified by arjun.

## Conclusion

Successfully exploiting Blind SQL Injection vulnerabilities behind modern WAFs requires a methodical approach that goes beyond basic scanning. By leveraging tools like gau and arjun for deep parameter discovery, proxychains for traffic obfuscation, and sqlmap's powerful tamper scripts and technique specification, security researchers can significantly increase their chances of uncovering hidden flaws. Remember that persistence and iterative refinement based on the target's responses are key to piercing the veil of these defenses. Always conduct such testing ethically and legally.
