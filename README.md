# Locatotron

A serverless geolocation-based notification system that enables organizers to send targeted alerts to users within a specific geographic radius.

## Project Overview

Locatotron allows event organizers to create location-based notification campaigns and send alerts to subscribed users who are within a defined distance from an event location. Built entirely on AWS serverless infrastructure, it combines:

- **PostgreSQL with PostGIS** for geographic spatial queries
- **AWS Lambda** for serverless compute
- **API Gateway** for RESTful endpoints
- **S3** for static website hosting
- **Mailgun** for email notifications

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

1. **Mailgun Account**: Sign up at https://www.mailgun.com for email notifications
   - Free tier: 5,000 emails/month
   - Note your API key and sandbox domain

2. **Google Maps API Key**: Get from https://console.cloud.google.com
   - Enable Maps JavaScript API
   - Restrict key to your domain for production

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
  "mailgun": {
    "apikey": "your-mailgun-api-key",
    "domain": "sandboxXXXXXXXX.mailgun.org",
    "senderName": "Locatotron",
    "sender": "postmaster"
  },
  "frontend": {
    "bucket": "locatotron-frontend-yourname-unique",
    "baseURL": "",
    "googleMapsAPIKey": "your-google-maps-api-key"
  }
}
```

**Important**: 
- Choose a strong database password
- The S3 bucket name must be globally unique across all AWS accounts
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

### Step 5: Deploy Backend Layer

This creates Lambda functions and API Gateway:

```bash
cd ../backend
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

### Step 6: Initialize Database

Run these Lambda functions once to set up PostGIS and create tables:

```bash
# Enable PostGIS extension
serverless invoke -f setupPostGIS --stage dev

# Create users and campaigns tables
serverless invoke -f createTables --stage dev
```

**Expected output**: `{"result":"success"}` for each command.

### Step 7: Update Configuration with Backend URL

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

### Step 8: Deploy Frontend Layer

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

### Step 9: Verify Deployment

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
│   ├── package.json         # Dependencies: pg, mailgun-js
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

### Mailgun Sandbox Limitations

**Symptom**: Emails not delivered

**Cause**: Mailgun sandbox only sends to authorized recipients

**Solution**: 
- Add recipient emails in Mailgun dashboard → Sending → Authorized Recipients
- Or upgrade to paid plan for unrestricted sending

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
- **Mailgun**: Free tier (5,000 emails/month)

**Total**: ~$50/month for always-on infrastructure

**Cost Savings**: Stop RDS instance when not in use (still pay for storage ~$2/month)

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
