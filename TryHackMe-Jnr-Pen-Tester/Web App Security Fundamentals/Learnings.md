## Content Discovery
Methods for Content Discovery.  
| Method    | Techniques                                                              |
|-----------|--------------------------------------------------------------------------|
| Manual    | robots.txt, sitemap.xml, favicon fingerprinting, HTTP headers, framework stack |
| OSINT     | Google dorking, Wappalyzer, Wayback Machine, GitHub, S3 buckets          |
| Automated | Gobuster dir, dns, and vhost modes                                       |

## Modern Web Stacks
Practices fingerprinting for different stacks - Mern/Express, Next.js Middleware, Django ORM, and Apache Lamp.  
Exploiting each follows the same three-step workflow:
1. Read the signals curl -I http://10.145.174.66:[PORT NUMBER]/
- Mern [3000 or 5000]: X-Powered-By: Express
- Next.js [3001]: next.js, nextjs
- Django [8000]: WSGIServer/0.2 CPython/X.X.X, csrfmiddlewaretoken
- Apache [8080]: Apache/2.4.49 (Unix)
2. Confirm the version
3. Execute the chain (Exploit related CVE)
