

# 🚀 Secure, Serverless Static Website on AWS

A production-ready, highly secure, and cost-effective static website infrastructure hosted on AWS. The platform utilizes **Amazon S3**, **Amazon CloudFront**, **AWS Certificate Manager (ACM)**, and **Amazon Route 53**, integrated with an external **.sa** country-code domain registrar.

👉 **Live Demo:** [https://ali-bagabas.sa](https://www.google.com/search?q=https://ali-bagabas.sa)

---

## 📐 Architecture Overview

The system architecture follows strict zero-trust storage principles: **S3 public access is completely blocked**, and content is served globally via CloudFront edge locations using Origin Access Control (OAC) with SigV4 request signing.

```
[ User Browser ]
       │
       │  1. DNS Query (ali-bagabas.sa)
       ▼
┌──────────────┐
│  Route 53    │ ── 2. Returns CloudFront Anycast IP / Domain
└──────────────┘
       │
       │  3. HTTPS Request (TLS 1.2/1.3)
       ▼
┌──────────────┐       ┌──────────────────────┐
│  CloudFront  │ ◄───► │ ACM (us-east-1)      │
│  (Edge CDN)  │       │ SSL/TLS Certificate  │
└──────────────┘       └──────────────────────┘
       │
       │  4. Authenticated REST Request (OAC / SigV4)
       ▼
┌──────────────┐
│  Amazon S3   │ ── [Block All Public Access: ENABLED]
│ (Private)    │
└──────────────┘

```

---

## ✨ Key Technical Features

* **Hybrid DNS Delegation (.sa Domain):** Configured custom Name Server (NS) delegation records from an external local registrar (Sahara Net) to an AWS Route 53 Public Hosted Zone.
* **Strict S3 Security (OAC):** Configured S3 using the **REST API origin** combined with **Origin Access Control (OAC)** instead of public S3 website endpoints. "Block All Public Access" remains **100% enabled**.
* **Global Edge Caching & Acceleration:** CloudFront distribution caches static assets globally to ensure low latency and reduced origin fetch requests.
* **Automated HTTPS/TLS Security:** TLS 1.2/1.3 encryption managed via AWS Certificate Manager (ACM) provisioned in `us-east-1` for edge deployment.
* **Apex Domain Routing:** Configured Route 53 `A` and `AAAA` **Alias records** to route apex domain queries (`ali-bagabas.sa`) directly to CloudFront without CNAME restrictions.

---

## 🛠️ AWS Services Used

| Service | Role / Function |
| --- | --- |
| **Amazon S3** | Private object storage origin hosting website static files (`index.html`, assets). |
| **Amazon CloudFront** | Content Delivery Network (CDN) enforcing HTTPS and authenticating with S3 via OAC. |
| **AWS Certificate Manager (ACM)** | Issues and manages SSL/TLS certificates in `us-east-1`. |
| **Amazon Route 53** | Public DNS hosting and dynamic Alias record management for `.sa` domain routing. |

---

## 💡 Key Architectural Lessons Learned

### 1. S3 REST API Origin vs. S3 Website Endpoint

* **Issue:** S3 Static Website endpoints require public bucket policies and do not support SigV4 authentication, preventing the use of standard Origin Access Control (OAC).
* **Solution:** Used the S3 **REST API domain name** (`bucket-name.s3.region.amazonaws.com`) as the CloudFront origin. Combined with a bucket policy restricting `s3:GetObject` strictly to the CloudFront distribution ARN via `AWS:SourceArn`, the S3 bucket remains entirely private from the open internet.

### 2. External Country-Code TLD (.sa) Management

* Top-Level Domains like `.sa` are not directly purchasable within Route 53 Domain Name Registrar.
* Resolving authority required delegating the 4 unique Route 53 Name Servers (`ns-xxxx.awsdns-xx...`) directly into the registrar's (Sahara Net) domain management console via custom NS records.

---

## 🔒 Bucket Policy Sample

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipalReadOnly",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
                }
            }
        }
    ]
}

```

---

## 📝 Step-by-Step Implementation Guide

### Step 1: External Domain Setup & Route 53 Delegation
1. Purchase your domain through your registrar (you can find them here: https://nic.sa/en/registrars).
2. Open **AWS Route 53 Console** $\rightarrow$ **Hosted zones** $\rightarrow$ **Create hosted zone**.
3. Enter your domain name (`ali-bagabas.sa`), select **Public Hosted Zone**, and click **Create**.
4. Copy the 4 generated Name Server (NS) addresses (`ns-xxxx.awsdns-xx.com...`).
5. In your third-party registrar dashboard, update your domain's custom NS records with those 4 AWS Name Servers to delegate DNS control to AWS.

### Step 2: Request SSL/TLS Certificate via ACM
1. Open **AWS Certificate Manager (ACM)** in region **`us-east-1` (N. Virginia)** *(Note: CloudFront requires certificates to be issued in `us-east-1`)*.
2. Click **Request Certificate** $\rightarrow$ **Request a public certificate**.
3. Add Domain Names: `ali-bagabas.sa` and `*.ali-bagabas.sa`.
4. Choose **DNS validation** and click **Request**.
5. Once created, click **Create records in Route 53** inside the certificate details page to complete validation automatically.

### Step 3: Configure Private S3 Storage
1. Open **AWS S3 Console** $\rightarrow$ **Create bucket**.
2. Enter a unique bucket name and select your preferred AWS region.
3. Ensure **"Block *all* public access"** remains **CHECKED**.
4. Leave static website hosting **DISABLED** (content will be served via CloudFront REST API integration).
5. Upload your static site files (`index.html`, `style.css`, assets).

### Step 4: Deploy CloudFront Distribution with OAC
1. Open **AWS CloudFront Console** $\rightarrow$ **Create distribution**.
2. **Origin Domain:** Select your S3 bucket's **REST API endpoint** (e.g., `bucket-name.s3.amazonaws.com`).
3. **Origin Access:** Select **Origin Access Control settings (recommended)** $\rightarrow$ **Create control setting** (Sign requests with SigV4).
4. **Viewer Protocol Policy:** Select **Redirect HTTP to HTTPS**.
5. **Alternate Domain Names (CNAME):** Add `ali-bagabas.sa` and `www.ali-bagabas.sa`.
6. **Custom SSL Certificate:** Select the ACM certificate created in Step 2.
7. **Default Root Object:** Set to `index.html`.
8. Click **Create distribution**.
9. Copy the generated **Bucket Policy** prompt from CloudFront and attach it to your S3 Bucket policy settings (S3 Console $\rightarrow$ Permissions $\rightarrow$ Bucket Policy).

### Step 5: Route 53 Alias Record Configuration
1. Return to **Route 53 Hosted Zones** $\rightarrow$ Select your domain.
2. Click **Create record**:
   * **Record Name:** Leave blank (apex domain `ali-bagabas.sa`).
   * **Record Type:** `A - Routes traffic to an IPv4 address...`
   * **Toggle Alias:** Enabled.
   * **Route Traffic To:** Alias to CloudFront distribution $\rightarrow$ Select your distribution.
3. Repeat step 2 to create an **`AAAA` record** (IPv6) pointing to the same CloudFront distribution.

---

## 👤 Author

**Ali Bagabas**

* Website: [ali-bagabas.sa](https://www.google.com/search?q=https://ali-bagabas.sa)
* LinkedIn: [Ali Bagabas](https://www.google.com/search?q=https://www.linkedin.com/in/alibagabas/)
