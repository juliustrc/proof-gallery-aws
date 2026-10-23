# Client Proof Gallery

A serverless photo proofing gallery on AWS. A photographer shares proofs through a **private, expiring link**, the client picks their favorites, and the photographer gets the list by email.

Built as my final project for CS 79C (AWS Architecture Portfolio Project) at Santa Monica College.

![Architecture diagram](diagram/architecture.png)

## The problem

Shared photo links (public folders, cloud drive links) usually never expire and can be forwarded to anyone. For client work, that means proofs can leak and there is no easy way to collect a client's choices.

## What it does

- Each client gets a unique, unguessable gallery link that **expires on a set date**.
- Photos live in a **private S3 bucket**. The page only shows them through **presigned URLs that expire after 15 minutes**.
- The client taps photos to mark favorites and clicks **Send my favorites**.
- Favorites are saved in DynamoDB and **emailed to the photographer** through SNS.
- Expired or unknown links return a clear error instead of any photos.

## AWS services

| Service | Role in this project |
|---|---|
| **S3** (private, encrypted) | Stores proof photos with all public access blocked |
| **Lambda** (Python 3.12) | `GetGallery` validates the link and creates presigned URLs; `SaveFavorites` stores picks and notifies |
| **API Gateway** (HTTP API) | Public HTTPS endpoints with throttling |
| **DynamoDB** | Gallery records (client name, expiry) and saved favorites |
| **SNS** | Emails the favorites list to the photographer |
| **CloudFront** + private S3 origin | Serves the gallery page over HTTPS using Origin Access Control |
| **CloudWatch** | Lambda logs and an error alarm that notifies through SNS |
| **IAM** | One least-privilege role for the Lambda functions, no access keys |
| **CloudFormation** | Defines the whole backend as code in `iac/proof-gallery.yaml` |

## How it works

1. The browser loads the page from CloudFront (`?g=<gallery-id>` in the link).
2. The page calls `GET /gallery/{id}` on API Gateway, which runs `GetGallery`.
3. The function looks the ID up in DynamoDB and rejects it if missing (404) or expired (410).
4. It lists the photos under `galleries/<id>/` in S3 and returns a presigned URL for each.
5. The browser loads the images directly from S3 with those short-lived URLs.
6. When the client sends favorites, `POST /gallery/{id}/favorites` runs `SaveFavorites`, which stores the picks in DynamoDB and publishes to SNS, which emails the photographer.

## Security design

- Photo bucket: Block Public Access on, encryption at rest (AES-256).
- Two layers of expiry: the gallery link (date in DynamoDB) and every image URL (15 minutes).
- Gallery IDs are random UUIDs, so they cannot be guessed.
- Lambda functions use an IAM role scoped to the exact table, bucket and topic they need.
- `SaveFavorites` only accepts photo keys that belong to that gallery's own folder.
- API throttling limits abuse and cost.
- HTTPS through CloudFront and API Gateway.

**Known limitations** (see the report for the full Well-Architected review): CORS is open to any origin for the demo, the gallery link is the only secret (there is no client login), and images are not cached by CloudFront.

## Repository layout

```
.
├── README.md
├── diagram/        Architecture diagram (PNG and draw.io source)
├── iac/
│   └── proof-gallery.yaml   CloudFormation template for the backend
├── src/
│   └── index.html           Gallery web page
├── evidence/       Screenshots of the working system
└── report/         Project report (PDF)
```

## How to deploy

You need an AWS account and the region **us-west-2**.

1. **Deploy the backend.** In CloudFormation, create a stack named `proof-gallery` from `iac/proof-gallery.yaml`. Provide:
   - `NotifyEmail`: the address that receives favorites
   - `PhotoBucketName`: a globally unique, lowercase bucket name

   Tick the IAM acknowledgment, then wait for `CREATE_COMPLETE`.
2. **Confirm the SNS subscription** by clicking the link in the email AWS sends.
3. **Create a gallery.** Add an item to the DynamoDB table with `galleryId` (a random UUID), `clientName`, and `expiresAt` (a Unix timestamp in the future).
4. **Upload photos** to the photo bucket under `galleries/<galleryId>/`.
5. **Publish the page.** Put your stack's `ApiUrl` output into `API` in `src/index.html`, upload the file to a second private S3 bucket, and create a CloudFront distribution with Origin Access Control and `index.html` as the default root object.
6. **Open** `https://<distribution-domain>/?g=<galleryId>`.

## Screenshots

<!-- Replace with your own files from the evidence folder -->

| Gallery page | Favorites email |
|---|---|
| ![Gallery](evidence/01-gallery.png) | ![Email](evidence/02-email.png) |

| Private bucket (AccessDenied) | Expired link |
|---|---|
| ![Private bucket](evidence/05-access-denied.png) | ![Expired](evidence/04-expired.png) |

## Cost

Designed to stay inside the AWS free tier: serverless pay-per-use services, 5 provisioned DynamoDB units, small resized images, and no NAT gateway, load balancer or always-on compute. The estimate is in the report.

## What I learned

<!-- Write 3 to 5 sentences in your own words: what surprised you, what was hardest to debug, what you would change. -->

## Status

The demo was torn down after capturing evidence to avoid ongoing charges. Everything can be rebuilt from `iac/proof-gallery.yaml`.

## Author

Julio, Web Development, Santa Monica College.
