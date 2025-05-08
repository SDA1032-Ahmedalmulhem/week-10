Step 2: Create S3 Bucket for Frontend Hosting
Go to the S3 Console in AWS.

Create a bucket named yourname-blogapp-frontend in region eu-north-1.

Uncheck Block all public access.

Enable Static website hosting in the bucket properties:

Index document: index.html

Error document: index.html (or leave blank)

Go to Permissions > Bucket Policy and add:

json


{
 "Version": "2012-10-17",
 "Statement": [
   {
     "Effect": "Allow",
     "Principal": "*",
     "Action": "s3:GetObject",
     "Resource": "arn:aws:s3:::yourname-blogapp-frontend/*"
   }
 ]
}
Save the bucket policy.
 Step 3: Create S3 Bucket for Media Uploads
Create another S3 bucket: yourname-blogapp-media in the same region (eu-north-1).

Uncheck Block all public access.

Add CORS configuration under Permissions > CORS:

json


[
 {
   "AllowedHeaders": ["*"],
   "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
   "AllowedOrigins": ["*"],
   "ExposeHeaders": ["ETag"]
 }
]
Test the upload:

Try uploading an image and retrieving it via public URL (should work if CORS and permissions are correct).

 Step 4: Create IAM User for Media Uploads
Go to the IAM Console > Users > Add user.

Username: blog-app-user

Enable Programmatic access (access key).

Attach a custom policy with this JSON (replace yourname-blogapp-media):

json


{
 "Version": "2012-10-17",
 "Statement": [
   {
     "Effect": "Allow",
     "Action": [
       "s3:PutObject",
       "s3:GetObject",
       "s3:DeleteObject",
       "s3:ListBucket"
     ],
     "Resource": [
       "arn:aws:s3:::yourname-blogapp-media",
       "arn:aws:s3:::yourname-blogapp-media/*"
     ]
   }
 ]
}
Attach this policy to the user.

Download or copy the Access Key ID and Secret Access Key — you’ll use them in .env, and they are shown only once.

 Step 5: EC2 Backend Deployment
5.1 Launch EC2 Instance
Go to the EC2 Dashboard.

Launch an instance with:

AMI: Ubuntu 22.04 LTS

Type: t3.micro (Free Tier eligible)

Region: eu-north-1

Configure Inbound Rules:

SSH (22), HTTP (80), HTTPS (443), Custom TCP (5000) from 0.0.0.0/0

5.2 Add User Data Script (on EC2 launch):
Paste this in the Advanced > User Data field when launching the EC2:

bash


#!/bin/bash
apt update -y
apt install -y git curl unzip tar gcc g++ make unzip

su - ubuntu << 'EOF'
export NVM_DIR="$HOME/.nvm"
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source "$NVM_DIR/nvm.sh"
nvm install --lts
nvm use --lts
npm install -g pm2
EOF

curl -L https://downloads.mongodb.com/compass/mongosh-2.1.1-linux-x64.tgz -o mongosh.tgz
tar -xvzf mongosh.tgz
mv mongosh-*/bin/mongosh /usr/local/bin/
chmod +x /usr/local/bin/mongosh
rm -rf mongosh*

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
rm -rf aws awscliv2.zip
5.3 SSH into the instance
bash


ssh -i <your-key.pem> ubuntu@<your-ec2-public-ip>
5.4 Clone the application
bash

git clone https://github.com/cw-barry/blog-app-MERN.git /home/ubuntu/blog-app
cd /home/ubuntu/blog-app
5.5 Configure Backend Environment (.env)
bash


cd backend
cat > .env << EOF
PORT=5000
HOST=0.0.0.0

# Database
MONGODB=<your-mongodb-connection-string>

# JWT
JWT_SECRET=$(openssl rand -hex 32)
JWT_EXPIRE=30min
JWT_REFRESH=$(openssl rand -hex 32)
JWT_REFRESH_EXPIRE=3d

# AWS
AWS_ACCESS_KEY_ID=<access-key>
AWS_SECRET_ACCESS_KEY=<secret-key>
AWS_REGION=eu-north-1
S3_BUCKET=<your-s3-media-bucket-name>
MEDIA_BASE_URL=https://<your-s3-media-bucket-name>.s3.eu-north-1.amazonaws.com

# Other
DEFAULT_PAGINATION=20
EOF
5.6 Configure Frontend Environment
bash


cd ../frontend
cat > .env << EOF
VITE_BASE_URL=http://<your-ec2-public-dns>:5000/api
VITE_MEDIA_BASE_URL=https://<your-s3-media-bucket>.s3.eu-north-1.amazonaws.com
EOF
5.7 Setup AWS CLI
bash


aws configure
# Enter Access Key ID
# Enter Secret Access Key
# Region: eu-north-1
# Output format: json
 Step 6: Run and Deploy
6.1 Run Backend with PM2
bash


cd /home/ubuntu/blog-app/backend
npm install
mkdir -p logs
pm2 start index.js --name "blog-backend"
pm2 save
pm2 startup
sudo pm2 startup systemd -u ubuntu --hp /home/ubuntu
6.2 Build and Deploy Frontend
bash


cd /home/ubuntu/blog-app/frontend
npm install -g pnpm@latest-10
pnpm install
pnpm run build
aws s3 sync dist/ s3://<your-frontend-bucket-name>/
 Deliverables
 Screenshot of backend running with pm2 ls

Screenshot of curl -I <your-frontend-s3-url> showing 200 OK

 Screenshot of index.html loading in browser (S3 site)

 Screenshot of file uploaded to media S3

 MongoDB Atlas shows blog data saved

 Cleanup
After completing the assignment:

Stop the EC2 instance

Remove IAM credentials

Delete S3 buckets if no longer needed
