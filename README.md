Sure! Based on your requirements, I'll provide a step-by-step tutorial for the **`telegram-pdf-to-markdown-bot`** project, assuming:

- You already have the Telegram bot set up at [t.me/ai_pdf_to_markdown_bot](https://t.me/ai_pdf_to_markdown_bot).
- You have the Telegram bot token available.
- The GitHub repository is at [github.com/racoci/telegram-pdf-to-markdown-bot](https://github.com/racoci/telegram-pdf-to-markdown-bot).
- You prefer to minimize manual setup steps by automating as much as possible with scripts.
- Environment variables are stored in separate files (e.g., `telegram.env`, `.env`).
- The `ngrok` URL is automatically saved into a `.env` file to avoid manual copying.

---

# **Telegram PDF to Markdown Bot Setup Guide**

This guide will help you set up the **Telegram PDF to Markdown Bot**, which accepts PDF files via Telegram, processes them using Google Cloud services to convert them into Markdown format, and returns the Markdown file to the user.

We'll minimize manual setup steps by automating as much as possible and using environment variable files to store configurations.

---

## **Table of Contents**

1. [Prerequisites](#prerequisites)
2. [Project Overview](#project-overview)
3. [Setup Instructions](#setup-instructions)
   - [1. Clone the Repository](#1-clone-the-repository)
   - [2. Set Up Environment Variables](#2-set-up-environment-variables)
   - [3. Install Dependencies](#3-install-dependencies)
   - [4. Set Up Google Cloud CLI and Authenticate](#4-set-up-google-cloud-cli-and-authenticate)
   - [5. Set Up Google Cloud Project](#5-set-up-google-cloud-project)
   - [6. Run the Bot Locally](#6-run-the-bot-locally)
   - [7. Test the Bot Using ngrok](#7-test-the-bot-using-ngrok)
4. [Cleanup Instructions](#cleanup-instructions)
5. [Additional Information](#additional-information)
   - [Project Architecture](#project-architecture)
   - [Security Considerations](#security-considerations)
6. [Conclusion](#conclusion)

---

## **Prerequisites**

- **Command-Line Interface**: Access to a terminal with `zsh` shell.
- **Telegram Bot Token**: Already obtained from [BotFather](https://t.me/BotFather).
- **Telegram Bot**: Bot is set up at [t.me/ai_pdf_to_markdown_bot](https://t.me/ai_pdf_to_markdown_bot).
- **Git**: For cloning the repository.
- **Docker**: For containerization.
- **Google Cloud SDK (`gcloud`)**: Installed and configured.
- **Python 3.9+**: For running Python scripts.
- **ngrok**: For testing webhooks locally.

---

## **Project Overview**

- **Repository**: [github.com/racoci/telegram-pdf-to-markdown-bot](https://github.com/racoci/telegram-pdf-to-markdown-bot)
- **Bot Functionality**: Accepts PDF files via Telegram, processes them to convert into Markdown format, and returns the Markdown file to the user.
- **Technologies Used**:
  - **Telegram Bot API**: For interacting with users.
  - **Google Cloud Vertex AI**: For processing and transcribing PDFs.
  - **Docker**: For containerization.
  - **Flask**: For handling webhook requests.
  - **ngrok**: For testing webhooks locally.

---

## **Setup Instructions**

### **1. Clone the Repository**

```zsh
git clone https://github.com/racoci/telegram-pdf-to-markdown-bot.git
cd telegram-pdf-to-markdown-bot
```

### **2. Set Up Environment Variables**

#### **2.1. Save the Telegram Bot Token**

Create a file named `telegram.env` to store your Telegram bot token:

```zsh
echo "TELEGRAM_BOT_TOKEN=your-telegram-bot-token" > telegram.env
```

Replace `your-telegram-bot-token` with your actual Telegram bot token.

#### **2.2. Create Other Environment Variable Files**

Create a `.env` file for other environment variables:

```zsh
touch .env
```

We'll populate this file automatically in later steps.

#### **2.3. Update `.gitignore`**

Ensure that your `.gitignore` file includes the environment variable files to prevent them from being committed to version control:

```zsh
echo "
# Environment variables
telegram.env
.env
" >> .gitignore
```

---

### **3. Install Dependencies**

#### **3.1. Install Python Dependencies**

Create a virtual environment and install dependencies:

```zsh
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

#### **3.2. Install ngrok**

Download ngrok via command line:

```zsh
wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
unzip ngrok-stable-linux-amd64.zip
chmod +x ngrok
```

---

### **4. Set Up Google Cloud CLI and Authenticate**

#### **4.1. Authenticate with Google Cloud**

If you haven't authenticated yet, use the following command:

```zsh
gcloud auth login --no-launch-browser
```

This command will provide a URL that you can open on another device to complete authentication.

Set application default credentials:

```zsh
gcloud auth application-default login --no-launch-browser
```

---

### **5. Set Up Google Cloud Project**

We will use a setup script `setup_project.sh` to automate the project setup.

#### **5.1. Obtain Your Billing Account ID**

List your billing accounts:

```zsh
gcloud beta billing accounts list
```

**Example Output:**

```
ACCOUNT_ID            NAME                OPEN  MASTER_ACCOUNT_ID
012345-678901-234567  My Billing Account  True
```

Copy the `ACCOUNT_ID` value (e.g., `012345-678901-234567`) for use in the setup script.

#### **5.2. Create `setup_project.sh`**

Create the script:

```zsh
touch setup_project.sh
chmod +x setup_project.sh
```

Add the following content to `setup_project.sh` (ensure you replace `YOUR_BILLING_ACCOUNT_ID`):

```zsh
#!/bin/zsh

# Exit immediately if a command exits with a non-zero status
set -e

# Variables
PROJECT_ID="telegram-pdf-to-markdown-bot"
PROJECT_NAME="Telegram PDF to Markdown Bot"
BILLING_ACCOUNT_ID="YOUR_BILLING_ACCOUNT_ID" # Replace with your billing account ID
REGION="us-central1"
SERVICE_NAME="telegram-pdf-to-markdown-bot"
REPO_NAME="telegram-pdf-to-markdown-repo"
SERVICE_ACCOUNT_NAME="telegram-pdf-to-markdown-bot-sa"

# Load Telegram bot token from telegram.env
source telegram.env

# Helper function to check for required variables
function check_variables() {
  if [[ "$BILLING_ACCOUNT_ID" == "YOUR_BILLING_ACCOUNT_ID" ]]; then
    echo "Please replace 'YOUR_BILLING_ACCOUNT_ID' with your actual billing account ID."
    exit 1
  fi
  if [[ -z "$TELEGRAM_BOT_TOKEN" ]]; then
    echo "TELEGRAM_BOT_TOKEN is not set in telegram.env"
    exit 1
  fi
}

# Step 0: Check for required variables
check_variables

# Step 1: Check if the project already exists
echo "Checking if project $PROJECT_ID exists..."
if gcloud projects describe $PROJECT_ID > /dev/null 2>&1; then
  echo "Project $PROJECT_ID already exists."
else
  echo "Creating Google Cloud project..."
  gcloud projects create $PROJECT_ID --name="$PROJECT_NAME"
fi

# Step 2: Set the project
echo "Setting gcloud to use the project $PROJECT_ID..."
gcloud config set project $PROJECT_ID

# Step 3: Link billing account
echo "Linking billing account..."
if gcloud beta billing projects describe $PROJECT_ID | grep 'billingEnabled: true' > /dev/null 2>&1; then
  echo "Billing is already enabled for project $PROJECT_ID."
else
  gcloud beta billing projects link $PROJECT_ID --billing-account=$BILLING_ACCOUNT_ID
fi

# Step 4: Enable required services
echo "Enabling required services..."
gcloud services enable \
    aiplatform.googleapis.com \
    run.googleapis.com \
    artifactregistry.googleapis.com \
    cloudbuild.googleapis.com

# Step 5: Set compute region
echo "Setting compute region..."
gcloud config set compute/region $REGION

# Step 6: Create a service account if it doesn't exist
SERVICE_ACCOUNT_EMAIL="$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com"
echo "Checking if service account $SERVICE_ACCOUNT_EMAIL exists..."
if gcloud iam service-accounts describe $SERVICE_ACCOUNT_EMAIL > /dev/null 2>&1; then
  echo "Service account $SERVICE_ACCOUNT_EMAIL already exists."
else
  echo "Creating service account..."
  gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME \
      --display-name="Telegram PDF to Markdown Bot Service Account"
fi

# Step 7: Assign roles to the service account
echo "Assigning roles to the service account..."
declare -a roles=("roles/aiplatform.user" "roles/logging.logWriter")

for role in "${roles[@]}"; do
  if gcloud projects get-iam-policy $PROJECT_ID \
      --flatten="bindings[].members" \
      --format='table(bindings.members)' \
      --filter="bindings.role:$role AND bindings.members:serviceAccount:$SERVICE_ACCOUNT_EMAIL" | grep "serviceAccount:$SERVICE_ACCOUNT_EMAIL" > /dev/null 2>&1; then
    echo "Service account already has role $role."
  else
    echo "Adding role $role to service account."
    gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
        --role="$role" --quiet
  fi
done

# Step 8: Create an Artifact Registry repository if it doesn't exist
echo "Checking if Artifact Registry repository $REPO_NAME exists..."
if gcloud artifacts repositories describe $REPO_NAME --location=$REGION > /dev/null 2>&1; then
  echo "Artifact Registry repository $REPO_NAME already exists."
else
  echo "Creating Artifact Registry repository..."
  gcloud artifacts repositories create $REPO_NAME \
      --repository-format=docker \
      --location=$REGION \
      --description="Docker repository for Telegram PDF to Markdown Bot"
fi

# Step 9: Build and push the Docker image
echo "Building and pushing Docker image..."
gcloud auth configure-docker $REGION-docker.pkg.dev --quiet

docker build -t $REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/$SERVICE_NAME:latest .

docker push $REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/$SERVICE_NAME:latest

# Step 10: Deploy to Cloud Run
echo "Deploying to Cloud Run..."
gcloud run deploy $SERVICE_NAME \
    --image $REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/$SERVICE_NAME:latest \
    --region $REGION \
    --platform managed \
    --allow-unauthenticated \
    --service-account $SERVICE_ACCOUNT_EMAIL \
    --update-env-vars "TELEGRAM_BOT_TOKEN=$TELEGRAM_BOT_TOKEN"

echo "Setup complete. Your Cloud Run service is deployed and ready to use."
```

**Note**: Replace `YOUR_BILLING_ACCOUNT_ID` with your actual billing account ID.

#### **5.3. Run the Setup Script**

Execute the setup script:

```zsh
./setup_project.sh
```

This script will:

- Check if the project exists; if not, create it.
- Set the project as the active project.
- Link the billing account if not already linked.
- Enable required APIs.
- Create a service account and assign roles if not already present.
- Create an Artifact Registry repository if not existing.
- Build and push the Docker image.
- Deploy the Docker image to Cloud Run.

---

### **6. Run the Bot Locally**

#### **6.1. Build the Docker Image**

```zsh
docker build -t telegram-pdf-to-markdown-bot:local .
```

#### **6.2. Run the Docker Container**

We will create a script to run the Docker container and automatically set the ngrok URL into the `.env` file.

#### **6.3. Create `run_local.sh` Script**

Create the script:

```zsh
touch run_local.sh
chmod +x run_local.sh
```

Add the following content to `run_local.sh`:

```zsh
#!/bin/zsh

# Exit immediately if a command exits with a non-zero status
set -e

# Load Telegram bot token
source telegram.env

# Start ngrok in the background and save the URL to .env
./ngrok http 8080 > /dev/null &

# Wait for ngrok to start
sleep 2

# Get the public ngrok URL
NGROK_URL=$(curl --silent http://localhost:4040/api/tunnels | grep -o '"public_url":"https://[0-9a-z]*\.ngrok\.io"' | sed 's/"public_url":"//;s/"//')

if [[ -z "$NGROK_URL" ]]; then
  echo "Failed to get ngrok URL."
  exit 1
fi

echo "NGROK_URL=$NGROK_URL" > .env

# Set the webhook URL
curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/setWebhook" -d "url=$NGROK_URL/webhook" | grep -q '"ok":true'

if [[ $? -ne 0 ]]; then
  echo "Failed to set webhook."
  exit 1
fi

echo "Webhook set to $NGROK_URL/webhook"

# Run the Docker container
docker run -d -p 8080:8080 --env-file telegram.env --env-file .env --name telegram-pdf-to-markdown-bot telegram-pdf-to-markdown-bot:local

echo "Docker container is running."
```

This script does the following:

- Starts ngrok and captures the public URL.
- Saves the ngrok URL into `.env` file.
- Sets the Telegram webhook URL to point to the ngrok URL.
- Runs the Docker container with the necessary environment variables.

#### **6.4. Run the `run_local.sh` Script**

```zsh
./run_local.sh
```

---

### **7. Test the Bot Using Telegram**

- Open Telegram and start a chat with your bot at [t.me/ai_pdf_to_markdown_bot](https://t.me/ai_pdf_to_markdown_bot).
- Send a PDF file to the bot.
- The bot should process the PDF and return the Markdown file.

---

## **Cleanup Instructions**

When you're done, run the cleanup script to delete all Google Cloud resources.

### **1. Create `cleanup_project.sh`**

Create the script:

```zsh
touch cleanup_project.sh
chmod +x cleanup_project.sh
```

Add the following content to `cleanup_project.sh`:

```zsh
#!/bin/zsh

# Exit immediately if a command exits with a non-zero status
set -e

# Variables
PROJECT_ID="telegram-pdf-to-markdown-bot"
REGION="us-central1"
SERVICE_NAME="telegram-pdf-to-markdown-bot"
REPO_NAME="telegram-pdf-to-markdown-repo"
SERVICE_ACCOUNT_NAME="telegram-pdf-to-markdown-bot-sa"

SERVICE_ACCOUNT_EMAIL="$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com"

# Step 1: Delete the Cloud Run service if it exists
echo "Checking if Cloud Run service $SERVICE_NAME exists..."
if gcloud run services describe $SERVICE_NAME --region $REGION > /dev/null 2>&1; then
  echo "Deleting Cloud Run service..."
  gcloud run services delete $SERVICE_NAME --region $REGION --quiet
else
  echo "Cloud Run service $SERVICE_NAME does not exist."
fi

# Step 2: Delete the Artifact Registry repository if it exists
echo "Checking if Artifact Registry repository $REPO_NAME exists..."
if gcloud artifacts repositories describe $REPO_NAME --location=$REGION > /dev/null 2>&1; then
  echo "Deleting Artifact Registry repository..."
  gcloud artifacts repositories delete $REPO_NAME --location=$REGION --quiet
else
  echo "Artifact Registry repository $REPO_NAME does not exist."
fi

# Step 3: Delete the service account if it exists
echo "Checking if service account $SERVICE_ACCOUNT_EMAIL exists..."
if gcloud iam service-accounts describe $SERVICE_ACCOUNT_EMAIL > /dev/null 2>&1; then
  echo "Deleting service account..."
  gcloud iam service-accounts delete $SERVICE_ACCOUNT_EMAIL --quiet
else
  echo "Service account $SERVICE_ACCOUNT_EMAIL does not exist."
fi

# Step 4: Delete the Google Cloud project
echo "Deleting the Google Cloud project..."
gcloud projects delete $PROJECT_ID --quiet

echo "Cleanup complete. The project and all associated resources have been scheduled for deletion."
```

### **2. Run the Cleanup Script**

```zsh
./cleanup_project.sh
```

---

## **Additional Information**

### **Project Architecture**

- **Dockerized Application**: The bot is containerized using Docker for consistent deployment.
- **Python Application**: Written in Python using `python-telegram-bot` library and `Flask` for webhook handling.
- **Local Storage**: PDF files are temporarily stored locally in the Docker container and deleted after processing.
- **Environment Variables**: Sensitive information like API tokens are stored in environment variable files (`telegram.env`, `.env`).

### **Security Considerations**

- **Secrets Management**: Do not commit the environment variable files (`telegram.env`, `.env`) to version control.
- **Cleanup**: The bot deletes PDF files after processing to minimize storage use and potential data exposure.

---

## **Conclusion**

You've set up the Telegram PDF to Markdown Bot using the provided scripts, minimizing manual setup steps. The bot is now ready to process PDFs sent via Telegram and return the converted Markdown files.

---

**If you have any questions or need further assistance, feel free to reach out!**

---

# **File Structure**

```
telegram-pdf-to-markdown-bot/
├── bot.py
├── Dockerfile
├── requirements.txt
├── setup_project.sh
├── run_local.sh
├── cleanup_project.sh
├── telegram.env
├── .env
├── .gitignore
```

---

# **Files Content**

### **`bot.py`**

*Note: Implement your bot logic in this file, including handling PDF processing and integration with Google Cloud services.*

### **`Dockerfile`**

```dockerfile
# Use the official lightweight Python image.
FROM python:3.9-slim

# Set environment variables
ENV PYTHONUNBUFFERED True
ENV APP_HOME /app

WORKDIR $APP_HOME

# Copy requirements and install
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY bot.py .

# Expose port
EXPOSE 8080

# Run the bot
CMD ["python", "bot.py"]
```

### **`requirements.txt`**

```text
Flask==2.2.5
python-telegram-bot==13.15
google-cloud-aiplatform==1.32.0
PyPDF2==3.0.1
```

### **`.gitignore`**

```gitignore
# Byte-compiled / optimized / DLL files
__pycache__/
*.py[cod]

# Virtual environment
venv/
env/

# Environment variables
telegram.env
.env

# Docker
*.log
```

---

# **Important Notes**

- **Minimizing Setup Steps**: The scripts `setup_project.sh` and `run_local.sh` automate the setup and running processes.
- **Environment Variables**: Using separate files (`telegram.env`, `.env`) for different environment variables helps manage configurations and keeps sensitive data secure.
- **Automating ngrok URL Retrieval**: The `run_local.sh` script automatically captures the ngrok URL and updates the `.env` file, eliminating the need for manual copy-pasting.
- **Using the Bot**: Interact with your bot via Telegram at [t.me/ai_pdf_to_markdown_bot](https://t.me/ai_pdf_to_markdown_bot).

---

**Happy Coding!**