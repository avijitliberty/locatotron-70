# Locatotron Development Guide

## Project Overview
Locatotron is a geolocation-based notification system that alerts users about events within their geographic area. Built as a serverless AWS application with three deployment layers: infrastructure, backend (Lambda), and frontend (S3 static site).

## Architecture

### Three-Tier Serverless Deployment
1. **Infrastructure** (`infrastructure/`): VPC, subnets, NAT Gateway, RDS PostgreSQL with PostGIS
2. **Backend** (`backend/`): Node.js Lambda functions behind API Gateway, uses PostGIS for spatial queries
3. **Frontend** (`frontend/`): Static HTML/CSS/JS built with Metalsmith, deployed to S3 via serverless-finch

### Key Components
- **Database**: PostgreSQL with PostGIS extension for geographic queries (ST_DISTANCE, ST_MAKEPOINT)
- **Lambdas**: CRUD operations for users and campaigns, all in VPC to access RDS
- **Notifications**: Mailgun integration for email alerts to users within campaign radius
- **Client**: Google Maps integration for location selection

## Critical Workflows

### Initial Setup
1. Copy `config.tmpl.json` to `config.dev.json` and fill in credentials:
   - PostgreSQL password
   - Mailgun API key and domain
   - S3 bucket name (must be globally unique)
   - Google Maps API key
2. Deploy in order:
   ```bash
   cd infrastructure && serverless deploy
   cd ../backend && npm install && serverless deploy
   cd ../frontend && npm install && npm run build && serverless client deploy
   ```
3. Initialize database (one-time):
   ```bash
   serverless invoke -f setupPostGIS
   serverless invoke -f createTables
   ```

### Development Iteration
- **Backend**: Deploy functions: `cd backend && serverless deploy`
- **Frontend**: Build and deploy: `cd frontend && npm run build && serverless client deploy`
- **Frontend local preview**: `npm run local` (uses local-web-server on `dist/`)

## Project-Specific Patterns

### Lambda Functions
- All handlers use `context.callbackWaitsForEmptyEventLoop = false` to prevent Lambda from waiting for empty event loop
- Connection pooling with `max: 1, min: 0` - critical for Lambda cold starts
- CORS headers included in every response via `getResponse()` helper
- Token-based authentication: users have `token`, campaigns have `admin_token`

### Spatial Data Handling
- Locations stored as PostGIS GEOGRAPHY type: `ST_MAKEPOINT(lng, lat)::GEOGRAPHY`
- Distance queries use `ST_DISTANCE()` with meters as unit
- Always convert to geometry for coordinate extraction: `ST_X(center::geometry)`
- Example from [backend/lambda/campaign.js](backend/lambda/campaign.js#L84):
  ```javascript
  ST_DISTANCE(users.location, ST_MAKEPOINT($4, $5)::GEOGRAPHY) < $6
  ```

### Configuration Management
- Stage-specific config files: `config.${stage}.json` (e.g., `config.dev.json`)
- Serverless imports via `${file(../config.${self:provider.stage}.json):db.password}`
- Environment variables injected into Lambda from config file
- CloudFormation exports link infrastructure to backend via `Fn::ImportValue`

### Frontend Build Process
- Metalsmith static site generator with Handlebars templates
- Config values injected as metadata: `config.frontend.baseURL`, `config.frontend.googleMapsAPIKey`
- Build output to `dist/`, then deployed to S3 bucket
- Assets (CSS/JS) copied from `assets/` to `dist/` root

## Integration Points

### Cross-Component Dependencies
- Backend depends on infrastructure exports: `locatotronDBAddress-${stage}`, `locatotronVPC-${stage}`, etc.
- Frontend needs backend API URL (baseURL) from backend deployment output
- All Lambda functions must be in VPC private subnets to access RDS

### External Services
- **Mailgun**: Campaign notifications sent via `mailgun-js` library to users within radius
- **Google Maps API**: Client-side location picker in `frontend/assets/js/map.js`
- **RDS PostgreSQL**: Database in private subnet, accessed via Lambda in same VPC

## Common Issues

### AWS Profile
All serverless.yml files use `profile: locatotron` - ensure AWS CLI configured with this profile

### Node.js Runtime
Project uses `nodejs6.10` (legacy) - update to `nodejs18.x` or later for new development

### CloudFormation Export Names
Must match pattern: `locatotronVPC-${stage}`, `locatotronDBAddress-${stage}`, etc. Change requires redeployment of infrastructure layer first.
