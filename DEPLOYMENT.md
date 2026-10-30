# Deploying BookBuddy on a Cloud VM

This guide deploys BookBuddy on an Ubuntu 24.04 virtual machine on **AWS EC2** or **Azure**.
Only step 1 (creating the server) differs between the two; every step after it is the same.

Steps marked **(local)** run on your own machine; steps marked **(server)** run on the VM over SSH.
Replace `<SERVER_IP>` with your VM's public IP address and `<USER>` with the login user:

| | AWS EC2 | Azure |
|---|---|---|
| `<USER>` | `ubuntu` | `azureuser` |
| Firewall | Security Group | Network Security Group (NSG) |

---

## 1a. Create the Server on AWS EC2
EC2 → **Launch instance**, then:

| Field | Value |
|---|---|
| Name | `bookbuddy` |
| AMI | **Ubuntu Server 24.04 LTS** |
| Instance type | **t3.medium** (4 GB RAM). `t2.micro` runs out of memory. |
| Key pair | Create new → `bookbuddy-key`, RSA, `.pem` |
| Security group | SSH (22) from **My IP**; Custom TCP **8501** from **My IP** |
| Storage | **25 GB** gp3 (the default 8 GB fills up during install) |

New AWS accounts must finish phone and card verification before instances can be launched.
Activation can take up to 24 hours.

## 1b. Create the Server on Azure
Virtual machines → **Create** → **Azure virtual machine**, then:

| Field | Value |
|---|---|
| Resource group | Create new → `bookbuddy-rg` |
| Name | `bookbuddy` |
| Availability options | No infrastructure redundancy required |
| Image | **Ubuntu Server 24.04 LTS - x64 Gen2** |
| Size | 2 vCPUs, 4 GiB RAM, e.g. **Standard_D2ls_v6** (see quota note below) |
| Authentication | SSH public key, user `azureuser`, generate new key pair `bookbuddy-key` |
| Public inbound ports | SSH (22) |

Click **Download private key and create resource** when prompted; the key cannot be downloaded again.

**Quota note:** many subscriptions have a quota of 0 cores for some VM families, which fails with
`QuotaExceeded`. Check which families you can use in a region from Cloud Shell:
```bash
az vm list-usage -l centralindia -o table | grep vCPUs
```
Pick a size from a family with a limit of 2 or more (e.g. `Standard Dlsv6 Family` → `Standard_D2ls_v6`).

After the VM is created, go to **Networking → Network settings**:
- **Create port rule → Inbound port rule**: Source **My IP address**, destination port **8501**, TCP, Allow.
- Edit the existing **SSH** rule: change Source from **Any** to **My IP address**.

## Keep the App Private
Keep port 8501 restricted to **My IP**. The app has no login, so anyone who can reach it can ask
questions and use up your Groq API quota. A phone on mobile data has a different IP, so it is
blocked; to test from a phone, temporarily set the 8501 rule's source to **Any** and set it back
afterwards.

---

## 2. Connect via SSH (local)
Move the downloaded key into place (check the downloaded file name; Azure may name it
`bookbuddy_key.pem`):
```bash
mkdir -p ~/.ssh
mv ~/Downloads/bookbuddy-key.pem ~/.ssh/bookbuddy-key.pem
chmod 400 ~/.ssh/bookbuddy-key.pem
ssh -i ~/.ssh/bookbuddy-key.pem <USER>@<SERVER_IP>
```

## 3. Install System Packages (server)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-venv python3-pip git tesseract-ocr poppler-utils libgl1
```

## 4. Clone the Repository (server)
```bash
git clone https://github.com/sarprk/project-book-study-ai-assistant.git
cd project-book-study-ai-assistant
```
The repository is private. When prompted, enter your GitHub username and, as the password, a
fine-grained Personal Access Token with **Contents: Read-only** access to this repository
(GitHub → Settings → Developer settings → Personal access tokens).

## 5. Set Up the Python Environment (server)
Install the CPU-only PyTorch build first, at the versions pinned in `requirements.txt`. Without
the CPU index, pip downloads several GB of CUDA libraries; without the pins, pip can pick a
`torchvision` that does not match `torch`, which breaks the embeddings with a misleading
"Could not import sentence_transformers" error.
```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install torch==2.9.1 torchvision==0.24.1 --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt
python -c "from langchain_huggingface import HuggingFaceEmbeddings; HuggingFaceEmbeddings(); print('OK')"
```

## 6. Copy the Textbooks, Vector Databases and .env (local)
These are not stored in git. From the project folder on your machine:
```bash
scp -i ~/.ssh/bookbuddy-key.pem -r data vector_db chapters_vector_db src/.env <USER>@<SERVER_IP>:~/project-book-study-ai-assistant/
ssh -i ~/.ssh/bookbuddy-key.pem <USER>@<SERVER_IP> "mv ~/project-book-study-ai-assistant/.env ~/project-book-study-ai-assistant/src/.env && chmod 600 ~/project-book-study-ai-assistant/src/.env"
```
If you have no local `src/.env`, create one on the server from `src/env_template.txt` and set
`GROQ_API_KEY`.

Copying the prebuilt vector databases skips the slow vectorizing step. To rebuild them on the
server instead, copy only `data` and run `python src/vectorize_script.py` on the server.

## 7. Test Run (server)
```bash
cd ~/project-book-study-ai-assistant
source venv/bin/activate
streamlit run src/main.py --server.address 0.0.0.0 --server.port 8501 --server.headless true --server.baseUrlPath book-study
```
Open `http://<SERVER_IP>:8501/book-study` in your browser. The first question is slow while the
embedding model downloads. Press Ctrl+C to stop.

## 8. Run as a Service (server)
A systemd service keeps the app running after you log out, restarts it if it crashes, and starts
it again when the VM boots. Replace `<USER>` in all three places.
```bash
sudo tee /etc/systemd/system/bookbuddy.service > /dev/null <<'EOF'
[Unit]
Description=BookBuddy Streamlit app
After=network.target

[Service]
User=<USER>
WorkingDirectory=/home/<USER>/project-book-study-ai-assistant
ExecStart=/home/<USER>/project-book-study-ai-assistant/venv/bin/streamlit run src/main.py --server.address 0.0.0.0 --server.port 8501 --server.headless true --server.baseUrlPath book-study
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now bookbuddy
```

## 9. Manage the App (server)
| Task | Command |
|---|---|
| Status | `sudo systemctl status bookbuddy` |
| Live logs | `journalctl -u bookbuddy -f` |
| Deploy code changes | `git pull && sudo systemctl restart bookbuddy` |
| Stop | `sudo systemctl stop bookbuddy` |

---

## Stopping, Starting and Deleting the VM
A running VM is billed by the hour. Stop it from the cloud console when not in use; the service
starts the app automatically on the next boot.

| | AWS EC2 | Azure |
|---|---|---|
| Stop (no compute charge) | Instance state → **Stop** | Overview → **Stop** (must show *Stopped (deallocated)*) |
| Still billed while stopped | EBS disk | OS disk, static public IP |
| Delete everything | Instance state → **Terminate**, then release any Elastic IP | Resource groups → `bookbuddy-rg` → **Delete resource group** |

On Azure, `sudo shutdown` from inside the VM leaves it *Stopped* but **not deallocated**, and
compute is still billed. Always stop from the portal.

---

## Troubleshooting
- **Page does not load:** port 8501 is missing from the firewall, or your IP changed since you
  selected **My IP**. Update the inbound rule.
- **`Permission denied (publickey)`:** check the login user (`ubuntu` on AWS, `azureuser` on
  Azure) and the `.pem` file.
- **Azure `QuotaExceeded`:** the VM family has no core quota in that region. See the quota note in 1b.
- **"Could not import sentence_transformers":** `torch` and `torchvision` versions do not match.
  Re-run the `pip install torch==... torchvision==...` line from step 5 and restart the service.
- **Install or app is "Killed":** out of memory. Use a VM with at least 4 GB RAM.
- **IP changed after stop/start:** use the new IP from the console, or reserve a static IP
  (Elastic IP on AWS; set the public IP to Static on Azure).
