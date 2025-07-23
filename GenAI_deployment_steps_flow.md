# 🩺 Diabetes Prediction App Deployment

This documentation covers manual and automated deployment of a Diabetes Prediction App using Docker, Minikube, and Jenkins Pipeline. It also includes troubleshooting steps and port-forwarding configuration.

---

## 📦 Application Overview

This app is a **Machine Learning model** that predicts whether a patient is likely to have diabetes based on inputs. It exposes an API using **TensorFlow Serving** or **Flask**, depending on implementation.

This is a Streamlit-based web application with the following core features:

🔹 Tabs / Sections:
  - Home — Welcome page with project context
  - Talk2Doc — AI chatbot ("Capsule") for user queries
  - Diagnosis — Predicts diabetes risk based on input data
  - Results — Displays diagnosis results
  - Knowledge Center (future)
  - Ask Queries (search Info about Dianetes)

Each of these is implemented in a separate file in the Tabs/ folder and rendered as tabs via Streamlit’s UI.
---

## 🧩 Application Structure

```bash
.
├── main.py                # Streamlit entry point, loads tabs
├── Tabs/
│   ├── home.py            # Home page
│   ├── talk2doc.py        # Gemini-powered chatbot
│   ├── diagnosis.py       # Input form + ML prediction
│   ├── result.py          # Show results
│   └── kc.py              # (Knowledge Center, WIP)
├── web_functions.py       # Helper functions for prediction
├── requirements.txt
└── .streamlit/
|    └── secrets.toml       # Contains GEMINI_API_KEY
├── k8s-manifests/
│   ├── deployment.yaml
│   └── service.yaml
├── __pycache__
└── Dockerfile
```

## ✅ Prerequisites (on EC2)

- Minikube installed and running (`minikube start`)
- Docker installed
- Jenkins installed and running
- Docker Hub account
- Kubernetes CLI (`kubectl`) installed
- Git repository with:
  - GenAI python project code
  - `Dockerfile`
  - `deployment.yaml` and `service.yaml` Kubernetes manifests

## 📦 Installation Steps

### 🔧 Install Docker

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
newgrp docker
```
### 🔧 Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -s \
https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```
### 🔧 Install Minikube

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl apt-transport-https ca-certificates conntrack
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start --driver=docker
```

### 🤖 Where Gemini API Is Used

1. Inside Tabs/talk2doc.py
  - This is the AI Chatbot (Capsule) tab, acting like a virtual doctor.

So every user query gets **sent to Gemini-Pro**, and the **AI-generated response** is displayed as doctor advice

## ✅ 1. Replace Gemini API with ChatGPT (OpenAI)

🔁 This is the best alternative because:

  - It provides an API for text-based models like gpt-3.5-turbo and gpt-4.
  - we can directly plug it into the talk2doc.py file with minimal code changes.

### 🔧 How to Replace with ChatGPT
🧪 **Step-by-Step:**

**Install OpenAI SDK:**
```bash
    pip install openai
```

**Update** .streamlit/secrets.toml:
```bash
    OPENAI_API_KEY = "your_openai_api_key"
```
**Update** `talk2doc.py` (replace Gemini logic):
```bash
    import openai
import streamlit as st

openai.api_key = st.secrets["OPENAI_API_KEY"]

st.title("Talk to AI Doctor (powered by ChatGPT)")

prompt = st.text_input("Enter your symptoms or query")

if prompt:
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",  # or gpt-4 if you have access
        messages=[
            {"role": "system", "content": "You are a medical assistant helping users with diabetes-related queries."},
            {"role": "user", "content": prompt}
        ]
    )
    st.write(response["choices"][0]["message"]["content"])
```

✅ Done — you’re now using ChatGPT via OpenAI instead of Gemini.

### 📌 What is gemini-pro?
  - gemini-pro is Google's **large language model** that powers conversation, text generation, question-answering, etc., similar to ChatGPT.
  - In this app, it acts as a **medical assistant** — though keep in mind it’s only as reliable as the model + your input prompt (no guarantees of clinical accuracy).


## Dockerfile

```bash
    # Use official Python 3.12 base image
FROM python:3.12-slim

# Set working directory in container
WORKDIR /app

# Copy all project files into the container
COPY . .

# Install dependencies from requirements.txt
RUN pip install --upgrade pip && pip install -r requirements.txt

# Expose the default Streamlit port
EXPOSE 8501

# Run the Streamlit app
CMD ["streamlit", "run", "main.py", "--server.port=8501", "--server.enableCORS=false"]
```

## Generate Gemini API Key

```bash
1. Visit the Gemini API Console
    Open this link:  https://makersuite.google.com/app/apikey

2. Sign In with Your Google Account

3. Generate a New API Key
  - Click "Create API Key"
  - Choose the project or create a new one
  - Copy the API Key shown

4. Store the API Key Securely
  export GEMINI_API_KEY="your_copied_api_key"
  Or store in .env:
  GEMINI_API_KEY=your_copied_api_key

📌 Notes:

  - This key is free for limited usage (read the usage limits in Gemini API pricing).
  - The key allows you to access models like gemini-pro and gemini-pro-vision.
```
## ✅ Test the Key (Optional)
```bash
  curl -H "Content-Type: application/json" \
     -H "Authorization: Bearer $GEMINI_API_KEY" \
     -d '{"contents":[{"parts":[{"text":"Hello, Gemini!"}]}]}' \
     https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent
```


## 🔧 1. Manual Deployment using Docker

### ✅ Steps:

1. Clone the repository:
   ```bash
   git clone https://github.com/your-repo/diabetes-app.git
   cd diabetes-app
   ```
2. Create the Docker image:
    ```bash
    docker build -t diabetes-app:latest .
    ```
3. Run the container:
   ```bash
   docker run -d -p 8501:8501 -e GEMINI_API_KEY=$GEMINI_API_KEY diabetes-app:latest
   ```
4. Access the application:
   ```bash
   Browser: http://<EC2_PUBLIC_IP>:8501
   ```



 ## Jenkins Build Logs:

### Jenkins Build and Deployment Log – Successful WAR Deployment to minikube cluster:

[View Log File for Docker Container Deployment](Build_log_Cisco_MockProject_Minikube.txt)
  
### Build Success Screenshots – WAR Deployment to Tomcat


![Build success status](Cisco_MockProject_Minikube_1.JPG)

![Build success status](Cisco_MockProject_Minikube_2.JPG)

![ApplicationTest](Cisco_MockProject_Minikube_3.JPG)

![ApplicationTest](Cisco_MockProject_Minikube_4.JPG)

## Accessing the Application

- **Automatically (inside pipeline)**:
  - Port-forwarding via:
    ```bash
    kubectl port-forward --address 0.0.0.0 service/jpetstore-service 30036:8080

    ```
Now visit: http://<Public-IP>:30036 in your browser

- **Manually** (if auto fails):
    ```bash
    kubectl port-forward --address 0.0.0.0 service/jpetstore-service 30036:8080
    ```
- **Open in Browser**:
  ```
  http://<host-ip>:30036
  or
  http://localhost:30036
  ```
