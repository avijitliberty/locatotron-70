# Locatotron

A serverless geolocation-based notification system that enables organizers to send targeted alerts to users within a specific geographic radius.

## Project Overview

Locatotron allows event organizers to create location-based notification campaigns and send alerts to subscribed users who are within a defined distance from an event location. Built entirely on AWS serverless infrastructure, it combines:

- **PostgreSQL with PostGIS** for geographic spatial queries
- **AWS Lambda** for serverless compute
- **API Gateway** for RESTful endpoints
- **S3** for static website hosting
- **Amazon SES** for email notifications

### Key Features

- **Geographic Campaigns**: Organizers create campaigns with a central location and description
- **User Subscriptions**: Users subscribe to campaigns by providing their location
- **Radius-Based Notifications**: When an organizer triggers a notification from a specific location with a radius, only users within that distance receive the alert via email
- **Interactive Map Interface**: Google Maps integration for selecting locations
- **Token-Based Security**: Campaign admins and users use secure tokens for authentication

### Architecture

The project is structured in three independently deployable layers:

1. **Infrastructure Layer** (`infrastructure/`): VPC, NAT Gateway, RDS PostgreSQL with PostGIS
2. **Backend Layer** (`backend/`): Node.js Lambda functions with API Gateway
3. **Frontend Layer** (`frontend/`): Static HTML/CSS/JS site deployed to S3

## Prerequisites

### Required Software

- **Node.js** (v14 or later recommended, though project uses legacy v6)
- **npm** (comes with Node.js)
- **AWS CLI** v2.x configured with credentials
- **Serverless Framework** v1.10.0 or later

Install globally:
```bash
npm install -g serverless
```

### Required AWS Permissions

Your AWS account/IAM user needs permissions to create:
- VPC, Subnets, Internet Gateway, NAT Gateway, Elastic IPs
- RDS instances (PostgreSQL)
- Lambda functions and IAM roles
- API Gateway
- S3 buckets
- CloudFormation stacks

### External Services

1. **Amazon SES**: AWS Simple Email Service for notifications (included with AWS)
   - Free tier: 62,000 emails/month when sending from Lambda
   - Starts in sandbox mode (users verify their email addresses)
   - Sender email must be verified before use

2. **Google Maps API Key**: Required for location selection interface
   - Get from Google Cloud Console
   - Enable Maps JavaScript API
   - Free tier: 28,000 map loads/month
   - See detailed setup instructions in Step 2.5 below

## Setup Instructions

### Step 1: Configure AWS CLI Profile

Create a named AWS profile for this project:

```bash
aws configure --profile locatotron
```

Enter your AWS Access Key ID, Secret Access Key, default region (e.g., `us-east-1`), and output format.

Verify the profile:
```bash
aws sts get-caller-identity --profile locatotron
```

### Step 2: Clone and Install Dependencies

```bash
git clone <repository-url>
cd locatotron-70

# Install backend dependencies
cd backend
npm install
cd ..

# Install frontend dependencies
cd frontend
npm install
cd ..
```

### Step 2.5: Create Google Maps API Key

Detailed instructions for obtaining and configuring your Google Maps API key:

1. **Go to Google Cloud Console**:
   - Visit https://console.cloud.google.com
   - Sign in with your Google account

2. **Create a New Project** (or select existing):
   - Click the project dropdown at the top
   - Click "NEW PROJECT"
   - Enter project name: "Locatotron" (or your preferred name)
   - Click "CREATE"
   - Wait for project creation and select it

3. **Enable Maps JavaScript API**:
   - In the left sidebar, go to **APIs & Services** → **Library**
   - Search for "Maps JavaScript API"
   - Click on "Maps JavaScript API"
   - Click **ENABLE** button
   - Wait for API to be enabled

4. **Create API Credentials**:
   - Go to **APIs & Services** → **Credentials**
   - Click **+ CREATE CREDENTIALS** at the top
   - Select **API key**
   - Your API key will be created and displayed
   - Copy the API key (starts with `AIza...`)

5. **Restrict the API Key** (Important for security):
   - Click **Edit API key** (or the pencil icon next to your key)
   - Under **Application restrictions**:
     - For development: Select "HTTP referrers (web sites)"
     - Add: `http://localhost:8000/*` (for local testing)
     - Add: `http://*.s3-website-*.amazonaws.com/*` (for S3 hosting)
     - For production: Add your specific S3 bucket URL after deployment
   - Under **API restrictions**:
     - Select "Restrict key"
     - Check only **Maps JavaScript API**
   - Click **SAVE**

6. **Enable Billing** (Required even for free tier):
   - Go to **Billing** in the left sidebar
   - Click **LINK A BILLING ACCOUNT** or create new
   - Add credit card (required but won't be charged within free tier)
   - Free tier: 28,000 map loads/month
   - After free tier: $7 per 1,000 additional loads

7. **Save Your API Key**:
   - Copy the API key for use in `config.dev.json` (next step)
   - Keep it secure - don't commit to public repositories

**Troubleshooting**:
- If maps don't load: Check browser console for errors
- "This page can't load Google Maps correctly": Billing not enabled or API restrictions too strict
- "RefererNotAllowedMapError": Add your website URL to API key restrictions

### Step 3: Create Configuration File

Copy the template configuration:

```bash
cp config.tmpl.json config.dev.json
```

Edit `config.dev.json` with your values:

```json
{
  "db": {
    "name": "locatotron",
    "username": "locatotron",
    "password": "YourSecurePassword123!",
    "port": "5432"
  },
  "ses": {
    "senderEmail": "notifications@yourdomain.com",
    "senderName": "Locatotron"
  },
  "frontend": {
    "bucket": "locatotron-frontend-yourname-unique",
    "baseURL": "",
    "googleMapsAPIKey": "AIzaSyXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  }
}
```

**Important**: 
- Choose a strong database password
- `senderEmail`: Use an email address you own (will be verified in next step)
- The S3 bucket name must be globally unique across all AWS accounts
- `googleMapsAPIKey`: Use the API key from Step 2.5
- Leave `baseURL` empty for now (we'll update it after backend deployment)

### Step 4: Deploy Infrastructure Layer

This creates the VPC, networking, and RDS PostgreSQL database:

```bash
cd infrastructure
serverless deploy --stage dev
```

**Expected output**: CloudFormation stack creation showing VPC, subnets, NAT gateway, and RDS instance.

**Time**: 10-15 minutes (RDS instance takes longest)

**Troubleshooting**:
- If deployment fails, check AWS service limits for VPC and RDS
- Verify your AWS profile has sufficient permissions
- Check CloudFormation console for detailed error messages

### Step 5: Configure AWS SES for Email Sending

Before deploying the backend, set up Amazon SES to send notification emails.

#### 5.1: Verify Your Sender Email Address

AWS SES requires you to verify ownership of the email address you'll use to send notifications:

```bash
# Replace with the email from your config.dev.json
aws ses verify-email-identity \
  --email-address notifications@yourdomain.com \
  --region us-east-1 \
  --profile locatotron
```

**Expected output**: No output means success.

**What happens next**:
1. AWS sends a verification email to `notifications@yourdomain.com`
2. Check your inbox (and spam folder)
3. Click the verification link in the email
4. You'll see a confirmation page from AWS

#### 5.2: Confirm Email is Verified

```bash
aws ses get-identity-verification-attributes \
  --identities notifications@yourdomain.com \
  --region us-east-1 \
  --profile locatotron
```

**Expected output**:
```json
{
  "VerificationAttributes": {
    "notifications@yourdomain.com": {
      "VerificationStatus": "Success"
    }
  }
}
```

If status shows "Pending", wait and check your email for the verification link.

#### 5.3: Understanding SES Sandbox Mode

**New AWS accounts start in SES "sandbox mode" with these restrictions:**

- ✅ **Sender emails**: Must be verified (just completed above)
- ⚠️ **Recipient emails**: Must also be verified before they can receive emails
- ⚠️ **Daily limit**: 200 emails per 24 hours
- ⚠️ **Rate limit**: 1 email per second

**How this affects Locatotron:**

When users subscribe to a campaign, AWS SES automatically sends them a **verification email**. Users must click the verification link before they can receive campaign notifications. This provides:
- ✅ **Double opt-in security**: Confirms users actually want notifications
- ✅ **Spam prevention**: Can't send to unverified/fake email addresses
- ✅ **Free tier**: 62,000 emails/month (vs Mailgun's 5,000)

**Subscription workflow in sandbox mode:**
1. User subscribes on your website with their email
2. SES sends verification email to user (auto-generated by AWS)
3. User clicks verification link in email
4. User's email is now verified and can receive notifications
5. When campaign sends notification, verified users within radius receive it

**For development/testing:**
- Manually verify test email addresses:
  ```bash
  aws ses verify-email-identity --email-address testuser@example.com --profile locatotron
  ```
- Each test user clicks their verification link
- Now they can subscribe and receive notifications

**Moving to production** (optional, only if needed):
- If you need >200 emails/day OR want to skip user email verification
- Request production access: AWS SES Console → Account Dashboard → "Request production access"
- Approval takes 24-48 hours
- See "AWS SES Production Access" section below for details

#### 5.4: Check Your Sending Limits

```bash
aws ses get-send-quota \
  --region us-east-1 \
  --profile locatotron
```

**Expected output**:
```json
{
  "Max24HourSend": 200.0,
  "MaxSendRate": 1.0,
  "SentLast24Hours": 0.0
}
```

This shows you can send 200 emails/day at 1 email/second in sandbox mode.

### Step 6: Deploy Backend Layer

This creates Lambda functions and API Gateway:

```bash
cd backend
serverless deploy --stage dev
```

**Expected output**:
```
endpoints:
  PUT - https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/user
  POST - https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/user/{id}
  GET - https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/user/{id}
  DELETE - https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/user/{id}
  PUT - https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/campaign
  GET - https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/campaign/{id}
  POST - https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/campaign/{id}
  PUT - https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/notify
```

**Save the base URL**: Copy `https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev` for the next step.

### Step 7: Initialize Database

Run these Lambda functions once to set up PostGIS and create tables:

```bash
# Enable PostGIS extension
serverless invoke -f setupPostGIS --stage dev

# Create users and campaigns tables
serverless invoke -f createTables --stage dev
```

**Expected output**: `{"result":"success"}` for each command.

### Step 8: Update Configuration with Backend URL

Edit `config.dev.json` and add the backend API URL:

```json
{
  "frontend": {
    "bucket": "locatotron-frontend-yourname-unique",
    "baseURL": "https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev",
    "googleMapsAPIKey": "your-google-maps-api-key"
  }
}
```

### Step 9: Deploy Frontend Layer

Build and deploy the static website to S3:

```bash
cd ../frontend
npm run build
serverless client deploy --stage dev
```

**Expected output**:
```
Serverless: Success! Your site should be available at http://locatotron-frontend-yourname-unique.s3-website-us-east-1.amazonaws.com/
```

**Note**: The bucket will be publicly readable for website hosting.

### Step 10: Verify Deployment

1. Open the S3 website URL in your browser
2. You should see the Locatotron homepage with a Google Map
3. Try creating a campaign (you'll need to provide an email address)
4. Subscribe as a user to test the full workflow

## Using Locatotron

### For Campaign Organizers

1. **Create Campaign**: On the homepage, enter campaign details:
   - Name and description
   - Admin email
   - Select location on map
   - Save and note the Campaign ID and Admin Token

2. **Share Campaign**: Provide users with the Campaign ID

3. **Send Notifications**: 
   - Navigate to the notification page
   - Enter Campaign ID and Admin Token
   - Select notification origin on map
   - Set radius (in meters)
   - Send - only users within the radius receive emails

### For Users

1. **Subscribe**: Visit the subscribe page
2. Enter Campaign ID
3. Provide email address
4. Select your location on map
5. Save and note your User Token
6. Receive notifications when within campaign radius

## Development Workflow

### Backend Changes

After modifying Lambda functions:
```bash
cd backend
serverless deploy --stage dev
```

Or deploy a single function:
```bash
serverless deploy function -f createUser --stage dev
```

### Frontend Changes

After modifying HTML, CSS, or JavaScript:
```bash
cd frontend
npm run build
serverless client deploy --stage dev
```

### Local Frontend Development

Preview changes without deploying:
```bash
cd frontend
npm run local
```
Opens local web server at `http://localhost:8000`

### Database Schema Changes

Update `backend/lambda/initDB.js`, then run:
```bash
cd backend
serverless invoke -f createTables --stage dev
```

**Warning**: This drops existing tables and recreates them.

## Project Structure

```
.
├── config.dev.json          # Stage-specific configuration
├── infrastructure/
│   └── serverless.yml       # VPC, RDS, networking CloudFormation
├── backend/
│   ├── serverless.yml       # Lambda functions, API Gateway
│   ├── package.json         # Dependencies: pg, rand-token
│   └── lambda/
│       ├── user.js          # User CRUD operations
│       ├── campaign.js      # Campaign CRUD + notifications
│       └── initDB.js        # Database initialization
└── frontend/
    ├── serverless.yml       # S3 bucket configuration
    ├── build.js             # Metalsmith static site builder
    ├── src/                 # HTML source files
    ├── layouts/             # Handlebars templates
    └── assets/              # CSS and JavaScript
        └── js/
            ├── map.js       # Google Maps integration
            ├── campaign.js  # Campaign management
            └── subscribe.js # User subscription
```

## Troubleshooting

### Lambda Timeout in VPC

**Symptom**: Lambda functions timeout after 3 seconds

**Cause**: Lambda in VPC without NAT Gateway can't reach external services

**Solution**: Verify NAT Gateway created in infrastructure layer

### Database Connection Errors

**Symptom**: "Connection refused" or "timeout" errors

**Cause**: Lambda not in correct VPC/subnets or security group misconfigured

**Solution**: 
- Check CloudFormation exports match: `locatotronVPC-dev`, `locatotronDBAddress-dev`
- Verify Lambda functions are in private subnets
- Check security group allows port 5432 from 10.0.0.0/16

### PostGIS Extension Errors

**Symptom**: "type geography does not exist"

**Cause**: PostGIS extension not installed

**Solution**: Run `serverless invoke -f setupPostGIS --stage dev`

### SES Email Delivery Issues

**Symptom**: Users not receiving notification emails

**Possible Causes**:
1. Sender email not verified in SES
2. Recipient email not verified (in sandbox mode)
3. User hasn't clicked SES verification link
4. Daily sending quota exceeded (200 emails/day in sandbox)
5. Email in spam folder

**Solutions**: 
- **Verify sender email**:
  ```bash
  aws ses verify-email-identity --email-address your@email.com --profile locatotron
  ```
- **Check sender verification status**:
  ```bash
  aws ses get-identity-verification-attributes --identities your@email.com --profile locatotron
  ```
- **Manually verify test users** (for development):
  ```bash
  aws ses verify-email-identity --email-address testuser@example.com --profile locatotron
  ```
- **Check sending statistics**:
  ```bash
  aws ses get-send-quota --profile locatotron
  aws ses get-send-statistics --profile locatotron
  ```
- **In production**: Request SES production access (see section below) to send to unverified emails

### CORS Errors in Browser

**Symptom**: "No 'Access-Control-Allow-Origin' header"

**Cause**: API Gateway CORS misconfigured

**Solution**: Verify `cors: true` in serverless.yml and Lambda responses include CORS headers

## Cost Estimation

Running Locatotron 24/7 with minimal usage:

- **RDS db.t2.micro**: ~$15/month
- **NAT Gateway**: ~$32/month (+ data transfer)
- **Lambda**: Free tier covers most development usage
- **S3**: < $1/month for static hosting
- **API Gateway**: Free tier covers most development usage
- **Amazon SES**: Free tier (62,000 emails/month from Lambda)
- **Google Maps**: Free tier (28,000 map loads/month)

**Total**: ~$50/month for always-on infrastructure

**Email costs after free tier**: $0.10 per 1,000 emails

**Cost Savings**: Stop RDS instance when not in use (still pay for storage ~$2/month)

## AWS SES Production Access

### When Do You Need Production Access?

Stay in **sandbox mode** if:
- ✅ Development and testing only
- ✅ Fewer than 200 emails/day
- ✅ Users are willing to verify their email (double opt-in is good security)
- ✅ You have time for users to click verification links

Request **production access** if:
- ❌ Need to send >200 emails per day
- ❌ Want to send to users without email verification step
- ❌ Need faster sending rate (>1 email/second)
- ❌ Building a production service at scale

### How to Request Production Access

1. **Go to AWS SES Console**:
   - https://console.aws.amazon.com/ses/
   - Select your region (us-east-1 or where you deployed)

2. **Navigate to Account Dashboard**:
   - In left sidebar: **Account dashboard**
   - Look for "Sending statistics" section

3. **Click "Request production access"**:
   - Usually a button at top right or in account status section

4. **Fill Out Request Form**:

   **Use case description** (be specific):
   ```
   We are operating Locatotron, a location-based notification system that allows 
   event organizers to send targeted alerts to users within a specific geographic 
   radius. Users voluntarily subscribe to campaigns by providing their email address 
   and location. When an organizer sends a notification, only users within the 
   specified radius (e.g., 500 meters) receive the email alert.
   
   Email volume: Approximately [X] emails per day
   Use case: Transactional notifications for subscribed users
   Users opt-in: Yes, explicit subscription required
   ```

   **Website URL**:
   ```
   http://[your-bucket-name].s3-website-[region].amazonaws.com
   ```

   **How you will handle bounces and complaints**:
   ```
   We monitor bounce and complaint rates via the SES dashboard. We will:
   - Remove hard-bounced emails from our database
   - Investigate complaint rates and remove problematic addresses
   - Maintain bounce rate <5% and complaint rate <0.1%
   - Implement SNS notifications for bounces if volume increases
   ```

   **How do recipients sign up**:
   ```
   Users explicitly subscribe via our web interface by:
   1. Selecting a campaign to join
   2. Providing their email address
   3. Selecting their location on a map
   4. Clicking subscribe
   
   In sandbox mode, users also verify their email via AWS SES verification link.
   ```

   **Compliance with AWS policies**:
   ```
   ✓ We only send to users who explicitly subscribe
   ✓ We will comply with CAN-SPAM Act and email best practices  
   ✓ Emails include sender information
   ✓ We respect user requests to unsubscribe
   ✓ We monitor sending reputation metrics
   ```

5. **Submit and Wait**:
   - Review typically takes **24-48 hours**
   - AWS may ask follow-up questions via support case
   - You'll receive email notification of approval or questions

6. **After Approval**:
   - ✅ Send to any email address (no verification required)
   - ✅ Higher limits: 50,000 emails/day initially
   - ✅ Can request further increases via support
   - ✅ Better email deliverability

### Monitoring Your SES Account Health

Even in production, maintain good sending reputation:

```bash
# Check current sending limits
aws ses get-send-quota --profile locatotron

# View sending statistics (last 15 days)
aws ses get-send-statistics --profile locatotron

# Check account sending enabled status  
aws ses get-account-sending-enabled --profile locatotron
```

**Important metrics to monitor**:
- **Bounce rate**: Keep below 5%
- **Complaint rate**: Keep below 0.1%
- **Sending enabled**: Should be "true"

If rates exceed thresholds, AWS may pause your sending ability.

### Alternative: Stay in Sandbox with Better UX

Instead of requesting production access, improve the sandbox experience:

1. **Send welcome email immediately** when user subscribes (before verification)
2. **Explain the verification step** in welcome email:
   ```
   "You'll receive a verification email from Amazon Web Services. 
   Please click the link to complete your subscription and start 
   receiving notifications."
   ```
3. **Check spam folder reminder** in UI after subscription
4. **Send reminder email** if verification not completed in 24 hours

This maintains double opt-in security while managing user expectations.

## Cleanup

Remove all AWS resources to avoid charges:

```bash
# Remove frontend S3 bucket
cd frontend
serverless client remove --stage dev

# Remove backend Lambda functions
cd ../backend
serverless remove --stage dev

# Remove infrastructure (VPC, RDS)
cd ../infrastructure
serverless remove --stage dev
```

**Note**: RDS deletion creates a final snapshot (retention: 7 days). Delete manually in AWS Console if needed.

## Security Considerations

- RDS database is in private subnets (not publicly accessible)
- Token-based authentication for users and campaigns
- CORS enabled for frontend domain only
- Use strong passwords for database
- Rotate tokens regularly for production use
- Consider AWS Secrets Manager for sensitive config in production

## Upgrading Node.js Runtime

Project uses legacy `nodejs6.10`. To upgrade:

1. Update `runtime: nodejs18.x` in all serverless.yml files
2. Test Lambda functions locally
3. Update dependencies in package.json
4. Redeploy all layers

## License

ISC

## Credits

Original Author: Robin Norwood
