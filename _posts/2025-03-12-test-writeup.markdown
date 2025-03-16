---
layout: post
title: "Practice Writeup: SQL Injection in Action"
author: "0x5t"
catalog: true
header-img: "img/post-bg-digital-native.jpg"
header-mask: 0.4
tags:
  - Web Exploitation
  - SQL Injection
  - Vulnerability Assessment
  - HackTheBox
---

## Introduction

In this practice writeup, I'll walk you through a simulated SQL Injection (SQLi) attack on a deliberately vulnerable web application. The objective is to demonstrate how minor oversights in input validation can lead to data exposure and unauthorized access. This exercise was performed in a controlled environment for educational purposes only.

## Environment Setup

- **Target:** A vulnerable web application hosted on a local instance.
- **Tools:**  
  - Burp Suite for intercepting and modifying HTTP requests  
  - SQLMap for automated exploitation  
  - Manual payload testing via browser and curl

## Step 1: Vulnerability Identification

I began by examining the application's login form. By entering a single quote (`'`) in the username field, I noticed an error message indicating a database error. This suggested that the application was not sanitizing user inputs properly, making it susceptible to SQL injection.

**Observation:**  
The error message:
```
SQL syntax error near '...'
```
confirmed that our payload was interfering with the backend SQL query.

## Step 2: Exploiting the Vulnerability

To confirm the SQLi vulnerability, I used Burp Suite to intercept the login request and modified the username parameter to include a typical SQL injection payload:

```
' OR '1'='1
```

This payload bypassed the authentication mechanism by turning the WHERE clause in the SQL query always true, allowing unauthorized access.

**Request Snippet:**
```
POST /login HTTP/1.1
Host: vulnerable-app.local
Content-Type: application/x-www-form-urlencoded

username=' OR '1'='1&password=irrelevant
```

After sending the modified request, I received a successful login response, confirming that the authentication bypass was possible.

## Step 3: Automated Testing with SQLMap

Next, I ran SQLMap against the login endpoint to enumerate database information. The command used was:

```bash
sqlmap -u "http://vulnerable-app.local/login" --data="username=test&password=test" --batch --risk=3 --level=5
```

SQLMap identified the injection point and provided details such as the database banner, current user, and available databases.

**Key Findings:**
- **Database Banner:** MySQL 5.7  
- **User:** `root` with minimal restrictions

## Mitigation and Best Practices

To mitigate SQL Injection vulnerabilities, it's critical to:

- **Use Prepared Statements:** Parameterize all database queries to ensure user inputs are treated as data.
- **Input Validation:** Implement both client-side and server-side validation to filter out harmful inputs.
- **Error Handling:** Avoid exposing detailed error messages to end users; instead, log them securely on the server.
- **Regular Audits:** Periodically test your applications using both manual methods and automated tools like SQLMap.

## Conclusion

This exercise highlights how simple SQL injection payloads can exploit vulnerable applications if proper security measures are not in place. Always follow best practices for input handling and query parameterization to secure your applications against such attacks.

*Disclaimer: This writeup is for educational purposes only. Unauthorized testing of systems without explicit permission is illegal and unethical.*

