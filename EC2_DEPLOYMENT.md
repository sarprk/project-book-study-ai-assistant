# EC2 Deployment Steps for BookBuddy

This guide outlines the steps to deploy the BookBuddy application on an AWS EC2 instance.

---

## 1. Open Required Ports
- In your EC2 Security Group, open port **8501** for inbound traffic (used by Streamlit).
- Set the source to **My IP** rather than `0.0.0.0/0`. The app has no login, so anyone
  who can reach it can ask questions and use up your Groq API quota.

## 2. Connect to EC2 via SSH
```bash
ssh -i "ec2-key-pair-2.pem" ubuntu@<ec2-public-dns>
```
Replace `<ec2-public-dns>` with your instance's public DNS.

## 3. Install Dependencies
```bash
sudo apt update
sudo apt install python3-venv
sudo apt install tesseract-ocr
sudo apt install poppler-utils
```

## 4. Clone the Project Repository
```bash
git clone https://github.com/sarprk/project-book-study-ai-assistant.git
```
If prompted, use your access token as the password.

## 5. Set Up the Project
```bash
cd project-book-study-ai-assistant
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

## 6. Configure Environment Variables
Create `src/.env` from the template and set your Groq API key:
```bash
cp src/env_template.txt src/.env
nano src/.env
```

## 7. Copy the Textbook PDFs
The PDFs are not stored in git, so copy them from your machine to the server.
Run this **on your local machine**, from the project folder:
```bash
scp -i "ec2-key-pair-2.pem" -r data ubuntu@<ec2-public-dns>:~/project-book-study-ai-assistant/
```

## 8. Vectorize Data
```bash
python src/vectorize_script.py
```

## 9. Run the Application
### For Development:
```bash
streamlit run src/main.py
```
### For Production (Background):
```bash
nohup streamlit run src/main.py > streamlit.log 2>&1 &
```

## 10. Monitor Logs
```bash
tail -f streamlit.log
```

## 11. Check Running Process
```bash
lsof -i:8501
```

## 12. Stop the Application
```bash
pkill -f "streamlit run src/main.py"
```

## 13. Access the App
Open in your browser:
```
http://<ec2_public_ip>:8501/
```
Replace `<ec2_public_ip>` with your EC2 instance's public IP address.

---

**Notes:**
- Always run the Streamlit app in `nohup` for background execution on production.
- `nohup` does not survive a server reboot. Start the app again after a restart.
- Ensure your `src/.env` file is correctly configured for the app to work.
- Make sure the PDFs are copied and the vectorization step completes successfully before running the app.
