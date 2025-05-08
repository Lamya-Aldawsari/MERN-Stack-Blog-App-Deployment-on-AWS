# MERN-Stack-Blog-App-Deployment-on-AWS
# MERN Stack Blog App Deployment on AWS

## Scenario

The application consists of:

* Backend: Express.js running on an EC2 instance. 
* Frontend: React hosted via S3 static website hosting.
* Database: MongoDB Atlas. 
* Media Uploads: Handled through a separate S3 bucket. 

## Architecture

The architecture involves:

* EC2 instance (t3.micro) for the Express.js backend. 
* S3 for static website hosting of the React frontend and storage of media files. 
* MongoDB Atlas for the database. 
* IAM user with programmatic access. 
* Security Groups for controlling access to the EC2 instance. 

## Assignment Tasks

### Part 1: MongoDB Atlas Configuration (Optional)

### Part 2: S3 Bucket for Frontend

1.  Create a bucket named `yourname-blogapp-frontend` in `eu-north-1
2.  Disable "Block all public access". 
3.  Enable static website hosting. 
4.  Add a bucket policy for public access.

    Bucket Policy:

    ```json
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
    `
### Part 3: S3 for Media Uploads

1.  Create a second bucket: `lamya-blogapp-media`. 
2.  Disable "Block all public access". 
3.  Configure CORS for browser upload support. 

    ```json
    [
    {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": ["ETag"]
    }
    ]
    ```
4.  Test uploading and retrieving a file.


### Part 4: IAM User and Policy for S3 Media Bucket Access

1.  Go to IAM Console > Users > Add users.
2.  Username: `blog-app-user`. 
3.  Permissions > Attach existing policies directly. 
4.  Create a custom policy with the following JSON: 

    ```json
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
    "arn:aws:s3:::lamya-blogapp-media",
    "arn:aws:s3:::lamya-blogapp-media/*"
    ]
    }
    ]
    }
    ```
5.  Attach this policy to the user. 
6.  \*Important\*: Create and save the Access Key ID and Secret Access Key for this user. You will not be able to view the secret key again.  And DO NOT share these credentials in your submission and solution GitHub repo!
### Part 5: EC2 Backend Setup

1.  Launch a `t3.micro` instance in `eu-north-1` using Ubuntu 22.04 LTS. 

2.  Allow incoming traffic only on necessary ports (SSH: 22, HTTP: 80, HTTPS: 443, Custom TCP: 5000). 

    Inbound Rule:

    | Type       | Protocol | Port Range | Source     | Description            |
    | :--------- | :------- | :--------- | :--------- | :--------------------- |
    | Custom TCP | TCP      | 5000       | 0.0.0.0/0  | Allow public traffic |

3.  SSH into the instance and run the User Data script below: 

    User Data Script:

    ```bash
    #!/bin/bash
    apt update -y
    apt install -y git curl unzip tar gcc g++ make unzip

    su ubuntu << 'EOF'
    export NVM_DIR="$HOME/.nvm"
    curl -o- [https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh](https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh) | bash
    source "$NVM_DIR/nvm.sh"
    nvm install --lts
    nvm use --lts
    npm install -g pm2
    EOF

    # MongoDB Shell
    curl -L [https://downloads.mongodb.com/compass/mongosh-2.1.1-linux-x64.tgz](https://downloads.mongodb.com/compass/mongosh-2.1.1-linux-x64.tgz) -o mongosh.tgz
    tar -xvzf mongosh.tgz
    mv mongosh-*/bin/mongosh /usr/local/bin/
    chmod +x /usr/local/bin/mongosh
    rm -rf mongosh*

    # AWS CLI
    curl "[https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip](https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip)" -o "awscliv2.zip"
    unzip awscliv2.zip
    ./aws/install
    rm -rf aws awscliv2.zip
    ```
4.  Clone the MERN app from this repo URL:  `https://github.com/cw-barry/blog-app-MERN.git/home/ubuntu/blog-app` 
5.  Configure your MERN app Backend on the instance with the commands below.  Remember to update placeholder data in the AWS S3 Configuration part with your own data.  Don't share your `AWS_ACCESS_KEY_ID` key and `AWS_SECRET_ACCESS_KEY` in your submission and don't send it to the solution Github repo.

    ```bash
    # Navigate to the backend directory
    cd backend

    # Create .env file
    cat .env << EOF
    PORT=5000 
    HOST=0.0.0.0 
    MODE=production

    # Database configuration
    # if you have your own MongoDB connection string you can use it here
    MONGODB=mongodb+srv://test:qazqwe123@mongodb.txkjsso.mongodb.net/blog-app

    # JWT Authentication
    JWT_SECRET=$(openssl rand -hex 32)
    JWT_EXPIRE=30min
    JWT_REFRESH=$(openssl rand -hex 32)
    JWT_REFRESH_EXPIRE=3d

    # AWS S3 Configuration
    AWS_ACCESS_KEY_ID=<access-key-from-step-6>
    AWS_SECRET_ACCESS_KEY=<secret-key-from-step-6>
    AWS_REGION=eu-north-1
    S3_BUCKET=<your-s3-media-bucket-name>
    MEDIA_BASE_URL=https://<your-s3-media-bucket-name>.eu-north-1.amazonaws.com

    # Misc
    DEFAULT_PAGINATION=20
    EOF
    ```
6.  Configure your MERN app Frontend on the instance with the commands below.  Remember to update placeholder data in the `.env` configuration part with your own data. 

    ```bash
    # Navigate to the frontend directory
    cd../frontend

    # Create .env file
    cat > .env << EOF
    VITE_BASE_URL=http://<your-ec2-dns>:5000/api
    VITE_MEDIA_BASE_URL=https://<your-s3-media-bucket-name>.eu-north-1.amazonaws.com
    EOF
    ```
7.  Configure AWS CLI: 

    ```bash
    # Configure AWS CLI with the IAM credentials created earlier
    aws configure

    # Enter your AWS Access Key ID when prompted
    # Enter your AWS Secret Access Key when prompted
    # Set default region to eu-north-1
    # Set default output format to json
    ```
8.  Deploy Backend: 

    ```bash
    # Navigate to the backend directory
    cd /home/ubuntu/blog-app/backend

    # Install dependencies
    npm install

    mkdir -p logs

    # Start the server using PM2
    pm2 start index.js --name "blog-backend"

    # Make PM2 start on system reboot
    pm2 startup
    sudo pm2 startup systemd -u ubuntu
    sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/v22.15.0/bin /home/ubuntu/.nvm/versions/node/v22.15.0/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu
    pm2 save
    ```
9.  Build and Deploy Frontend.  Don't forget to update placeholder data with your own s3 bucket name for frontend.

    ```bash
    # Navigate to the frontend directory
    cd /home/ubuntu/blog-app/frontend

    # Install dependencies
    npm install -g pnpm@latest-10
    pnpm install

    # Build the frontend
    pnpm run build

    # Deploy to S3
    aws s3 sync dist/ s3://<your-s3-frontend-bucket-name>/
    ```

   


## Cleanup Reminder

* Stop EC2 instance. 
* Remove IAM credentials. 
* Delete S3 buckets if no longer needed. 
