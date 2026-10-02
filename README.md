# Server_Operating_System_lab4_assignment
# LAB 4 - APACHE EXERCISE

**Group Name:** Trojan  
**Date:** 1 October 2026  
**Environment:** Killercoda (Ubuntu)

| No | Student Name      | Student ID | Responsibility                         |
|----|-------------------|------------|----------------------------------------|
| 1  | Thin Thiri Zaw    | 6705142020 | Task 1 - Traffic and status code count |
| 2  | Paing Oo Thant    | 6705142006 | Task 1c - Browser vs curl User-Agent   |
| 3  | L Peter San Awng  | 6705142021 | Task 2 - HTTP methods                  |
| 4  | Kaung Myat Tun    | 6705142016 | Task 3 - CGI, GET vs POST              |
| 5  | Aung Kyaw Phyo    | 6705142012 | Task 4 and final report writing        |

---

## TASK 1



### Task 1 answers:


---

## TASK 1c

**Log lines (`tail -3 /var/log/apache2/access.log`)**

Line 1: browser asking for the site icon
```
10.244.5.148 - - [02/Oct/2026:15:20:01 +0000] "GET /favicon.ico HTTP/1.1" 404 436 "https://acb1f908ced291a6-1-80.papa.r.killercoda.com/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36 Edg/154.0.0.0"
```

Line 2: browser asking for the page
```
10.244.5.148 - - [02/Oct/2026:15:20:03 +0000] "GET / HTTP/1.1" 200 701 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36 Edg/154.0.0.0"
```

Line 3: curl asking for the page
```
::1 - - [02/Oct/2026:15:20:22 +0000] "GET / HTTP/1.1" 200 818 "-" "curl/8.5.0"
```

**Summary**

| Client | Request | Status | Referer | User-Agent |
|---|---|---|---|---|
| Browser (Edge) | GET /favicon.ico | 404 | Killercoda page URL | Mozilla/5.0 ... Edg/154.0.0.0 |
| Browser (Edge) | GET / | 200 | - | Mozilla/5.0 ... Edg/154.0.0.0 |
| curl | GET / | 200 | - | curl/8.5.0 |


### Task 1c answers:

- Browser vs curl: the browser's User-Agent is a long Mozilla/5.0 string naming the OS and rendering engine, while curl's is only curl/8.5.0.
- Referer: the favicon request shows the Killercoda page URL as its Referer, because the browser asked for the icon while viewing that page. The page request and the curl request show "-", meaning they were typed or opened directly and not clicked from another page.
- Why admins care: the User-Agent shows what kind of client is making requests, so an admin can spot bots, scripts or unusual traffic.

---

## TASK 2

Predictions (recorded before running): GET = 200  POST = 200  PUT = 403  DELETE = 403

Observed:
```
HTTP/1.1 200 OK                     (GET)
HTTP/1.1 200 OK                     (POST)
HTTP/1.1 405 Method Not Allowed     (PUT)
HTTP/1.1 405 Method Not Allowed     (DELETE)
```

Full DELETE response headers (`curl -s -i -X DELETE ... | head -8`):
```
HTTP/1.1 405 Method Not Allowed
Date: Thu, 01 Oct 2026 13:06:48 GMT
Server: Apache
Allow: GET,POST,OPTIONS,HEAD
Content-Length: 223
Content-Type: text/html; charset=iso-8859-1
```

### Task 2 answers:
- GET and POST predictions were correct. PUT and DELETE were predicted as
  403 but returned 405.
- Why 405 and not 403: 403 means "you are not permitted to access this
  resource". 405 means the resource exists, but this method is not
  supported on it. The server is not refusing us personally; it simply has
  no way to perform PUT or DELETE on a static file.
- 404 vs 405: 404 means the server cannot find the resource. index.html
  exists, so 404 would be wrong. 405 means "found it, but not like that".
- The Allow header (GET,POST,OPTIONS,HEAD) explains the results: GET and
  POST are in the list, so both returned 200; PUT and DELETE are not, so
  both returned 405.
- POST returned 200 because Apache's static file handler treats POST like
  GET: it sent back index.html and ignored the body. No program on the
  server read name=student&id=123.
- The "Server: Apache" header shows no version number, confirming the
  ServerTokens Prod hardening from Step 4.

---


## TASK 3


## TASK 3

GET with query string:
```
Method received: GET
Query string: course=192-442&week=4
```

POST with body:
```
Method received: POST
Query string: 
POST body: name=student&id=123
```

Access log (grep cgi-bin):
```
::1 - - [01/Oct/2026:13:09:50 +0000] "GET /cgi-bin/echo.sh?course=192-442&week=4 HTTP/1.1" 200 175 "-" "curl/8.5.0"
::1 - - [01/Oct/2026:13:10:01 +0000] "POST /cgi-bin/echo.sh HTTP/1.1" 200 186 "-" "curl/8.5.0"
```
(2 log lines, not 4, because only 2 CGI requests were sent.)

### Task 3 answers:
- GET carried its data in the URL query string (course=192-442&week=4), after the "?".
- POST carried its data in the request body (name=student&id=123), which
  the CGI script read from standard input.
- The access log shows the full GET URL including the query string in
  plain text, but the POST line shows only /cgi-bin/echo.sh, without the
  body.
- Why it matters: anything sent by GET is stored wherever the URL is
  recorded. A password sent by GET would end up in several places, for
  example the web server access log and the browser history (also
  bookmarks, the address bar and proxy logs). That is why login forms use
  POST. POST still needs HTTPS, because the body is not encrypted over
  plain HTTP.
- Unlike Task 2, a program (echo.sh) actually read the POST data this time.

---

---

## TASK 4

### SCENARIO A - File permissions removed (chmod 000 about.html)

**Prediction:** 403  
**Observed code:** 403 http://localhost/about.html

**Evidence (error.log):**
```
[Thu Oct 01 13:16:02.751051 2026] [core:error] [pid 5406:tid 127695233238720] (13)Permission denied: [client ::1:46010] AH00132: file permissions deny server access: /var/www/html/about.html
```

**Diagnosis:** The file still exists, but with permissions 000 the www-data
user that Apache runs as cannot read it, so Apache returns 403 Forbidden
rather than 404 Not Found.

**Restored:** chmod 644 /var/www/html/about.html -> 200

### SCENARIO B - Missing index file (site1 index.html moved to /tmp)

**Prediction:** 404  
**Observed code:** 403

**Evidence (site1-error.log):**
```
[Thu Oct 01 13:18:53.289777 2026] [autoindex:error] [pid 5406:tid 127695149070016] [client ::1:42060] AH01276: Cannot serve directory /var/www/site1/public_html/: No matching DirectoryIndex (index.html,index.cgi,index.pl,index.php,index.xhtml,index.htm) found, and server-generated directory index forbidden by Options directive
```

**Diagnosis:** The directory exists but contains no index file, and
Options -Indexes from Step 4 forbids an automatic file listing, so Apache
returns 403. If the directive were reversed (Options +Indexes), Apache
would return 200 with a list of the files in the directory.

**Restored:** moved index.html back -> site1 returns 200

**Note:** A and B give the same 403 for different reasons. A is a file
permission problem; B is the -Indexes directive combined with a missing
index file. The error was written to site1's own error log, which shows
why per-site logs are useful.

### SCENARIO C - Virtual host disabled (a2dissite site1.conf)

**Prediction:** 200  
**Observed:** 200 http://localhost/ but the title was
`<title>192-442 Lab Server</title>` instead of `<title>Site One</title>`

**Evidence:** curl -H "Host: site1.lab.local" returned the default site's
title, while the status code still said 200.

**Diagnosis:** With site1.conf disabled, no virtual host has ServerName
site1.lab.local, so Apache falls back to the first enabled virtual host
(000-default) and serves the wrong site. This fault is dangerous because
the 200 makes it look like everything works, so the content must be
checked, not only the status code.

**Restored:** a2ensite site1.conf + reload -> `<title>Site One</title>`

### SCENARIO D - Apache stopped (systemctl stop apache2)

**Prediction:** No HTTP status  
**Observed:** 000 http://localhost/ and curl exit status: 7

**Evidence:** there is no log entry, because Apache was not running to write
one. While stopped, "systemctl reload apache2" also reported
"apache2.service is not active, cannot reload". curl exit status 7 means
it failed to connect.

**Diagnosis:** Nothing was listening on port 80, so there was no HTTP response
at all. A 500 is produced by a running server that failed internally;
"no status code" means the server itself is down. A monitoring system
would rather see a 500, because it proves the server is alive and points
to an application problem, while connection refused means a complete
outage.

**Restored:** systemctl start apache2 -> 200

### SCENARIO E - Config typo (ServerNaem)

**Prediction:** 200  
**Observed code:** 200 (the site kept working because it stays active on the old config)

**Evidence (apache2ctl configtest):**
```
AH00526: Syntax error on line 233 of /etc/apache2/apache2.conf:
Invalid command 'ServerNaem', perhaps misspelled or defined by a module not included in the server configuration
```
systemctl reload apache2 -> "Job for apache2.service failed."

**Diagnosis:** The misspelled directive is a fatal syntax error, so the reload
was refused and Apache kept running with its last good configuration in
memory. If systemctl restart had been used instead, Apache would have
stopped and then failed to start, taking the site down.

**Restored:** copied the backup back, configtest -> Syntax OK, reload -> 200
(The AH00558 "fully qualified domain name" message is only a warning.)

---

## FINAL CHECK

```
200 http://localhost/
200 http://localhost/        (Host: site1.lab.local)
200 http://localhost/        (Host: site2.lab.local)
```

All three sites return 200 after every scenario was restored.
