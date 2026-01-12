### Frontend: step-by-step flow
- User enters domain (https://www.schoolspider.co.uk/)  
DNS resolves to the CloudFront distribution (CNAME or alias record).
Browser initiates HTTPS to CloudFront.  
- WAF (attached to CloudFront) inspects request  
WAF evaluates rules (rate limits, geo/IP block, OWASP rules) and either allows or blocks.  
If blocked → WAF returns 403 and the request stops.  
- CloudFront (edge) receives request  
If the requested asset is cached at the edge → the static React assets are served directly from the nearest edge location, which significantly reduces latency. 
If cache-miss → CloudFront forwards request to the origin (S3) and connects to S3 using Origin Access Control (OAC). S3 bucket policy only allows CloudFront to read objects.  
S3 returns the object and then CloudFront caches the object according to TTL/cache-control and serves it to the client.  
- Client receives response  

### How security is implemented  
- “Frontend security is implemented at multiple layers.  
- All traffic is encrypted using HTTPS with ACM certificates. CloudFront serves the site over HTTPS using an ACM certificate.
- AWS WAF on CloudFront: WAF inspects every request at the edge and blocks common threats (OWASP rules), bad IPs, rate-limit attackers and stop bots before they hit CloudFront or S3 origin.  
- Origin Access Control / Block Public Access: The S3 bucket is not public — only CloudFront is allowed to read objects (via Origin Access Control and bucket policy), preventing anyone from bypassing the CDN.  
- S3 encryption at rest: Objects are encrypted (SSE-S3 or SSE-KMS) so data at rest is protected; KMS key policy limits who can decrypt.  
- Signed URLs / Signed Cookies (for private assets): For any restricted content we generate time-limited CloudFront signed URLs or cookies so only authorized users can fetch those files.  
- Logging & audit trails: WAF logs, CloudFront access logs, S3 access logs and CloudTrail are enabled and sent to a secure account/S3 for detection & forensics.  
- Cache + WAF reduce attack surface: By serving cached content at edge and blocking malicious traffic early, origin load and exposure are minimized.  

### How Hight Availability is achieved?  
- High availability for the frontend is achieved by using CloudFront’s globally distributed edge locations in front of S3, which is a multi-AZ regional service, so there is no single point of failure.  
- CloudFront is globally distributed across hundreds of edge locations, User is automatically routed to the nearest healthy edge, If one edge location fails, traffic is transparently routed to another.  
- edge failures are handled automatically by AWS.  
- Amazon S3 is a regional, multi-AZ service, Objects are stored redundantly across multiple Availability Zones, so the origin layer is inherently highly available.
- CloudFront caches frontend assets at edge locations which reduces load on origin and and improves resilience. 

- “Frontend HA is achieved using CloudFront’s globally distributed edge network in front of S3, which is a multi-AZ regional service. Edge caching further reduces dependency on the origin, and failures are automatically handled by AWS without manual intervention.”

