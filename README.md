# Multi-Cloud Weather Tracker with Terraform (AWS & Azure)
Multi-cloud weather tracking application deployed on AWS and Azure with disaster recovery, managed and automated through Terraform.

## Project Overview
This project involves deploying a weather tracker application across AWS and Azure, incorporating disaster recovery capabilities. The app's front-end (HTML, CSS, JS) is hosted statically on AWS S3 (with CloudFront for CDN) and Azure Blob Storage. The entire infrastructure is managed using Terraform, automating deployments on both cloud platforms.

### Key components include:
- **Host the weather app** on AWS S3 and Azure Blob Storage.
- **Register a domain** through Namecheap for DNS configuration.
- **Implement disaster recovery** with Route 53 DNS failover using AWS and Azure endpoints.
- **Automate the infrastructure setup** with Terraform.

## Architecture

<img width="1032" height="560" alt="diagram-multicloud-weather-tracker" src="https://github.com/user-attachments/assets/a8c7f36a-a589-4bc4-a9f6-3d8ee8933e5d" />

### Services Used
- **AWS S3 & Azure Blob Storage:** Host the weather app statically. [Hosting]
- **AWS CloudFront**: Distribute content globally. [CDN]
- **Route 53:** Automate failover between AWS and Azure functions. [DNS Failover]
- **Terraform:** Automate multi-cloud infrastructure provisioning. [Infrastructure as Code]

### How It Works
1. When a user enters the website domain in their browser, the DNS request is routed to AWS Route 53, which holds the DNS records for the domain.
2. The hosted zone in Route 53 stores the routing rules for the primary and secondary endpoints. It directs traffic to either the AWS or Azure infrastructure depending on the health checks.
3. The primary website is hosted on AWS S3 for static content (HTML, CSS, JS) and served globally via AWS CloudFront CDN.
4. If the primary AWS endpoint fails, Route 53 reroutes the traffic to the secondary endpoint on Azure Blob Storage present under a resource group.

## Project Setup
1. Prerequisites
2. Define AWS Resources on Terraform
3. Define Azure Resources using Terraform
4. Implement Disaster Recovery with Route 53 DNS Failover

## 1. Prerequisites

### 1.1 Install Terraform
We will use Terraform as the Infrastructure as Code (IaC) tool, to simplifie cloud infrastructure provisioning.

### 1.2 Setup Azure and AWS Accounts
You need accounts on Azure and AWS to deploy resources.

### 1.3 Configure Access for Terraform
Install and configure Azure CLI & AWS CLI.

### 1.4 Clone this repository
Create a directory and clone this repository:
```bash
mkdir multi-cloud-weather-tracker
```
Inside this directory, you will have the following Terraform files:
- **`main.tf`:** To define providers and resources.
- **`variables.tf`:** To declare input variables.
- **`aws_credentials.tfvars`**: To securely store AWS credentials.
- **`azure_credentials.tfvars`**: To securely store Azure credentials.

### 1.5 Secure Sensitive Information
Create a file named `.gitignore`. Add sensitive files to `.gitignore` to prevent them from being pushed to version control, you can do this by adding the following line in the file:
```plaintxt
*.tfvars
```

### 1.6 Initialize Terraform
Run the following command to initialize the project and download the required provider plugins:
```bash
terraform init
```

### 1.7 Validate and Plan the Setup
Validate Configuration Files:
```bash
terraform validate
```
This command checks the syntax and validity of the Terraform configuration files.  

<img width="1124" height="459" alt="1 7" src="https://github.com/user-attachments/assets/96f4947d-387b-4aa0-a13c-ea49a01bcd92" />

## 2. Define AWS Resources on Terraform
The first step to create the multi-cloud weather tracker is to host the static content (HTML, CSS, JS, and assets) on an AWS S3 bucket with static website hosting enabled.

### 2.1 Prepare Your Website Code
You can get the website code from this repository. The directory structure should look like this:
```bash
multi-cloud-weather-tracker/
├── website/
│   ├── index.html
│   ├── styles.css
│   ├── script.js
│   └── assets/
│       ├── image1.jpg
│       ├── image2.png
├── main.tf                  # Terraform configuration for AWS and Azure
├── variables.tf             # Input variables for AWS and Azure
├── aws_credentials.tfvars   # AWS credentials (securely stored)
├── azure_credentials.tfvars # Azure credentials (securely stored)
└── terraform.tfstate        # Terraform state file (auto-created)
```
### 2.2 Define the S3 Bucket and Upload Files
Below is the Terraform code to configure the S3 bucket and upload files (`main.tf`)
```HCL
# Define an S3 bucket for static website hosting
resource "aws_s3_bucket" "weather_app" {
  bucket = "weather-tracker-app-bucket-25102025"  # Use a globally unique name

  website {
    index_document = "index.html"
    error_document = "error.html"
  }

  # Set bucket ownership controls
  lifecycle {
    prevent_destroy = true  # Prevent accidental deletion
  }
}

resource "aws_s3_bucket_public_access_block" "public_access" {
  bucket                  = aws_s3_bucket.weather_app.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# Upload website files to the S3 bucket
resource "aws_s3_object" "website_index" {
  bucket = aws_s3_bucket.weather_app.id
  key    = "index.html"
  source = "website/index.html"
  content_type = "text/html"
}

resource "aws_s3_object" "website_style" {
  bucket = aws_s3_bucket.weather_app.id
  key    = "styles.css"
  source = "website/styles.css"
  content_type = "text/css"
}

resource "aws_s3_object" "website_script" {
  bucket = aws_s3_bucket.weather_app.id
  key    = "script.js"
  source = "website/script.js"
  content_type = "application/javascript"
}

# Upload assets (images) to the S3 bucket
resource "aws_s3_object" "website_assets" {
  for_each = fileset("website/assets", "*")
  bucket   = aws_s3_bucket.weather_app.id
  key      = "assets/${each.value}"
  source   = "website/assets/${each.value}"
}

# Add a bucket policy to allow public read access
resource "aws_s3_bucket_policy" "bucket_policy" {
  bucket = aws_s3_bucket.weather_app.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Sid       = "PublicReadGetObject",
        Effect    = "Allow",
        Principal = "*",
        Action    = "s3:GetObject",
        Resource  = "arn:aws:s3:::${aws_s3_bucket.weather_app.id}/*"
      },
      {
        Sid       = "CloudFrontLogsWrite",
        Effect    = "Allow",
        Principal = {
          Service = "cloudfront.amazonaws.com"
        },
        Action    = "s3:PutObject",
        Resource  = "arn:aws:s3:::${aws_s3_bucket.weather_app.id}/cloudfront-logs/*"
      }
    ]
  })
}
```
*Make sure you are using a globally unique name for your S3 bucket.

### 2.3 Explanation of Key Components

- **S3 Bucket**  
  Static Website Hosting: Enabled with the website block, specifying index.html as the main entry point and error.html for errors.  
  Lifecycle Rule: Prevent accidental deletion with prevent_destroy.  

- **Public Access**  
  The aws_s3_bucket_policy provides public GET access to files while keeping upload access restricted.  

- **File Upload**  
  aws_s3_object uploads the specified files to the S3 bucket.  
  The for_each block iterates over files in the `assets/` directory, uploading them individually.  

### 2.4 Apply Terraform Configuration
Initialize Terraform:
```bash
terraform init
```
Review and Apply Changes:
```bash
terraform plan -var-file="aws_credentials.tfvars" -var-file="azure_credentials.tfvars"
terraform apply -var-file="aws_credentials.tfvars" -var-file="azure_credentials.tfvars"
```
*If you get an error, you can try running the `terraform apply` command again.

**Verify the Resources**  
After the configuration is applied, Terraform will:
- Create the S3 bucket.
- Upload all website files and images.
- Configure static website hosting with a public read policy.

### 2.5 Access Your Website
You can now access your hosted website using the S3 website endpoint.  

<img width="1920" height="1080" alt="2 5" src="https://github.com/user-attachments/assets/4494c3b1-5780-4f20-8b75-cbc24a8f1a75" />

## 3. Define Azure Resources using Terraform

### 3.1 Setup a Resource Group and a Storage Account
Define the Resource Group and the Storage Account that will host the static website.

**Define Resource Group**
```HCL
# Define Resource Group
resource "azurerm_resource_group" "rg" {
  name     = "rg-static-website"
  location = "East US"
}
```
- **`name`**: The name of the resource group, which in this case is "rg-static-website"
- **`location`**: The Azure region where the resources will be created. You can use "East US" or any other preferred Azure region.

**Define Storage Account**
```HCL
# Define Storage Account with Static Website
resource "azurerm_storage_account" "storage" {
  name                     = "mystorageaccount25102025"
  resource_group_name       = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier              = "Standard"
  account_replication_type = "LRS"
  account_kind              = "StorageV2"

  static_website {
    index_document = "index.html"
  }
}
```
- **`name`**: A globally unique name for the storage account.
- **`account_tier`**: "Standard" is the most common choice for static websites.
- **`account_replication_type`**: We use "LRS" for locally redundant storage to ensure high availability.
- **`account_kind`**: "StorageV2" enables general-purpose storage.
- **`static_website`**: Specifies that static website hosting will be enabled, with the "index.html" file being the entry point.

### 3.2 Upload Static Files to Azure Blob Storage
After the resource group and storage account are set up, the next step is to upload the static files (HTML, CSS, JS, and other assets) to Azure Blob Storage.

**Upload the index.htlm**
```HCL
# Upload index.html
resource "azurerm_storage_blob" "index_html" {
  name                   = "index.html"
  storage_account_name   = azurerm_storage_account.storage.name
  storage_container_name = "$web"  # Static website container
  type                   = "Block"
  content_type           = "text/html"
  source                 = "website/index.html"  # Path to local file
}
```
- **`name`**: The file name in Azure Blob Storage ("index.html").
- **`content_type`**: MIME type for HTML files ("text/html").
- **`source`**: The local path to the `index.html` file.

**Upload the styles.css**
```HCL
# Upload styles.css
resource "azurerm_storage_blob" "styles_css" {
  name                   = "styles.css"
  storage_account_name   = azurerm_storage_account.storage.name
  storage_container_name = "$web"
  type                   = "Block"
  content_type           = "text/css"
  source                 = "website/styles.css"  # Path to local file
}
```

**Upload the script.js**
```HCL
# Upload script.js
resource "azurerm_storage_blob" "scripts_js" {
  name                   = "script.js"
  storage_account_name   = azurerm_storage_account.storage.name
  storage_container_name = "$web"
  type                   = "Block"
  content_type           = "application/javascript"
  source                 = "website/script.js"  # Path to local file
}
```

**Upload Assets (Images, Files, etc.)**
```HCL
# Upload all assets in the assets folder
resource "azurerm_storage_blob" "assets" {
  for_each = fileset("website/assets", "**/*")

  name                   = "assets/${each.value}"
  storage_account_name   = azurerm_storage_account.storage.name
  storage_container_name = "$web"
  type                   = "Block"
  content_type           = lookup(
    {
      "png"  = "image/png"
      "jpg"  = "image/jpeg"
      "jpeg" = "image/jpeg"
      "gif"  = "image/gif"
      "svg"  = "image/svg+xml"
    },
    split(".", each.value)[length(split(".", each.value)) - 1],
    "application/octet-stream"
  )
  source = "website/assets/${each.value}"  # Path to local assets
}
```
- **`for_each`**: Loops through all files in the `assets` folder and uploads them.
- **`content_type`**: The MIME type for each asset based on its file extension.

### 3.3 Apply Terraform Configuration
Review and Apply the Deployment:
```bash
terraform plan -var-file="aws_credentials.tfvars" -var-file="azure_credentials.tfvars"
terraform apply -var-file="aws_credentials.tfvars" -var-file="azure_credentials.tfvars"
```
Once the files are uploaded, your static website will be accessible via the Azure Storage Account's public URL.  

<img width="1920" height="1080" alt="3 2 5" src="https://github.com/user-attachments/assets/af28a79e-e25b-4b62-9dc4-62e3f33c2816" />

## 4. Implement Disaster Recovery with Route 53 DNS Failover

### 4.1 Buy and Register a Domain Name
You can purchase a domain name from any domain registrar you prefer. In this case, I used Namecheap, so the configuration will be based on that.

### 4.2 Create a Hosted Zone in Route 53
The following code creates the hosted zone:
```HCL
resource "aws_route53_zone" "main" {
  name = "yourdomain.com"
}
```

### 4.3 Request an SSL Certificate for CloudFront
Since CloudFront does not support HTTP to HTTPS redirection by default, you need an SSL certificate in ACM.  

**Request a SSL Certificate:**
1. Go to **AWS Certificate Manager (ACM).**
2. Request a public certificate for your domain name (`yourdomain.com` and `www.yourdomain.com`).
3. Validate the certificate using **DNS validation** (recommended).

**DNS Validation on Namecheap**  
In Namecheap, go to Advanced DNS and add a new CNAME record under Host Records, using the CNAME name (only the part before your domain) and CNAME value provided by ACM, as shown in the example below.  

<img width="1595" height="612" alt="4 3" src="https://github.com/user-attachments/assets/52e8f477-3f0b-4f65-bff4-93cf476b3833" />  

<img width="1595" height="612" alt="4 3ii" src="https://github.com/user-attachments/assets/357935e2-e9a1-4f88-b9a4-936bee9e579d" />  

To verify that your DNS record is propagating correctly, you can run the following command in your terminal:
```bash
nslookup -type=CNAME <CNAME_name_from_ACM>
```
If the output shows the **canonical name** matching the **CNAME value** from ACM, your record is configured correctly, just wait a few minutes for ACM to complete the validation.

### 4.4 Configure CloudFront with Alternate Domain Names
**After obtaining an SSL certificate:**
1. Go to **CloudFront** → Create and select your distribution.
2. Add your Alternate Domain Names (**CNAMEs**).
3. Attach the ACM Certificate to the distribution.

### 4.5 Define Health Checks for Failover
The following code defines the Health Checks in the `main.tf` file:

**AWS (Primary) Health Check**
```HCL
resource "aws_route53_health_check" "aws_health_check" {
  type              = "HTTPS"
  fqdn              = "your-aws-cloudfront.cloudfront.net"
  port              = 443
  request_interval  = 30
  failure_threshold = 3
}
```

**Azure (Secondary) Health Check**
```HCL
resource "aws_route53_health_check" "azure_health_check" {
  type              = "HTTPS"
  fqdn              = "your-azure-site.web.core.windows.net"
  port              = 443
  request_interval  = 30
  failure_threshold = 3
}
```

### 4.6 Setup Route 53 Records for Failover

**Primary Record (AWS - CloudFront)**
```HCL
resource "aws_route53_record" "primary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "yourdomain.com"
  type    = "A"

  alias {
    name                   = "your-aws-cloudfront.cloudfront.net"
    zone_id                = "Z2FDTNDATAQYW2"  # CloudFront's hosted zone ID
    evaluate_target_health = true
  }

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier = "primary"
  health_check_id = aws_route53_health_check.aws_health_check.id
}
```

**Secondary Record (Azure - Blob Storage)**
```HCL
resource "aws_route53_record" "secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.yourdomain.com"
  type    = "CNAME"

  records = ["your-azure-site.web.core.windows.net"]

  ttl = 300

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary"
  health_check_id = aws_route53_health_check.azure_health_check.id
}
```

**Apply the Changes:**
```bash
terraform apply -var-file="aws_credentials.tfvars" -var-file="azure_credentials.tfvars"
```

### 4.7 Update Namecheap Nameservers
To use Route 53 as your DNS provider, update the domain’s nameservers in Namecheap.
1. Log in to Namecheap.
2. Go to Domain List → Manage your domain.
3. Under Nameservers, select Custom DNS.
4. Enter the AWS Route 53 nameservers (from your hosted zone).  

<img width="1593" height="284" alt="4 7" src="https://github.com/user-attachments/assets/50168766-1f28-4215-aa59-435f109775d6" />


### 4.8 Verify DNS Propagation
You can check if the domain is resolving correctly in https://www.whatsmydns.net/
- Search for A record of `yourdomain.com`
- Check CNAME record for `www.yourdomain.com`
- Ensure CloudFront URL and domain are resolving correctly
- Now you can try accessing your website by typing in the DNS.  

<img width="1223" height="951" alt="4 8" src="https://github.com/user-attachments/assets/62edc5d5-6eb8-45c9-9570-9aa27e3d7d97" />

### 4.9 Expected Behavior and Limitations
- **Primary Site (AWS CloudFront):**  
  HTTPS enabled and secure  
  Custom domain working with SSL certificate  

- **Failover Site (Azure Blob Storage):**  
  Failover functionality works correctly  
  Shows "Not secure" warning - this is expected behavior  
  Azure Blob Storage static websites don't support HTTPS with custom domains by default  
  In production environments, you would configure Azure CDN to enable HTTPS

**Testing failover functionality:**
1. Disable your CloudFront distribution
2. Visit your domain - it should redirect to Azure
3. You'll see "Not secure" warning - in this case the HTTP fallback on Azure is normal and doesn't indicate any configuration issues
4. The site should remain functional despite the warning

## Fixing Common Issues

**403 Forbidden Error:**  
Add Alternate Domain Names (CNAMEs) to CloudFront and attach the correct ACM certificate.  

**CloudFront Distribution Not Appearing in Route 53:**  
Use an A Record (Alias) instead of a CNAME in Route 53.  

**DNS Not Resolving:**  
Verify that Namecheap’s nameservers are updated to use Route 53.  

**Certificate Mismatch:**  
Ensure your ACM certificate includes both domains (`example.com` and `www.example.com`).  

**Azure Failover “Not Secure” Warning:**  
Expected behavior — Azure Blob Storage doesn’t support HTTPS for custom domains; failover is still functional.  

**InvalidUri (HTTP 400):**  
Map your custom domain inside the Azure Storage Account under Custom Domain Settings so it recognizes your hostname.  

**AccountRequiresHttps (HTTP 400):**  
Disable the “Secure transfer required” option temporarily in your Azure Storage Account to allow HTTP access during testing.  

## Clean-up
To clean-up, run this command in your terminal: 
```bash
terraform destroy
```
It will prompt you to confirm that you want to proceed with the destruction process of all the resources defined in your Terraform configuration.

## Conclusion

This project demonstrated the design and deployment of a Multi-Cloud Weather Tracker website with built-in disaster recovery using Terraform, AWS, and Azure. By leveraging Infrastructure as Code (IaC), it ensured a scalable, repeatable, and automated deployment process across both cloud environments.  

**Key results include:**
- Deploying a high-availability weather tracking application across AWS and Azure.
- Automating infrastructure provisioning with Terraform modules.
- Configuring Route 53 DNS failover to automatically switch traffic between cloud environments during outages.

## Learnings

By completing this project, I gained hands-on experience in automating cloud infrastructure with Terraform, deploying web applications across AWS and Azure, and implementing DNS-based failover using Route 53 to ensure high availability and effective disaster recovery in a multi-cloud environment.
