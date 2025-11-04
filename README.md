# ✅ Flask App Deployment with Systemd and Ansible Rolling Update

## 🧱 1. Application Setup (on QA server)

* Created a new directory for the Flask app:

  ```bash
  mkdir my_flask_app
  cd my_flask_app
  ```

* Created a basic `requirements.txt` containing:

  ```text
  flask
  ```

* Developed a minimal Flask application in `app.py`:

  ```python
  from flask import Flask
  app = Flask(__name__)

  @app.route("/")
  def home():
      return "Hello from Flask!"

  @app.route("/health")
  def health():
      return "OK", 200

  if __name__ == "__main__":
      app.run(host="0.0.0.0", port=5000)
  ```

* Installed dependencies using:

  ```bash
  pip3 install -r requirements.txt
  ```

---

## 🛠️ 2. Service Creation

* Created a **systemd service** file to manage the app:

  ```ini
  # /etc/systemd/system/my_flask_app.service
  [Unit]
  Description=My Flask App
  After=network.target

  [Service]
  User=ubuntu
  WorkingDirectory=/home/ubuntu/my_flask_app
  ExecStart=/usr/bin/python3 app.py
  Restart=always

  [Install]
  WantedBy=multi-user.target
  ```

* Reloaded systemd and started the service:

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable my_flask_app
  sudo systemctl start my_flask_app
  ```

* Verified the app was accessible at `http://<QA_SERVER_IP>:5000/health`.

---

## 🚀 3. Rolling Deployment with Ansible

* Packaged the application into a tarball:

  ```bash
  tar -czf /tmp/my_flask_app.tar.gz my_flask_app/
  ```

* Created an Ansible playbook to perform a **rolling deployment** with health checks:

  ```yaml
  ---
  - name: Rolling deployment of the application
    hosts: all
    serial: 1
    become: yes
    tasks:
      - name: Deploy the new version of the application
        copy:
          src: /tmp/my_flask_app.tar.gz
          dest: /opt/my-app
          owner: root
          group: root
          mode: '0755'

      - name: Extract the new version
        unarchive:
          src: /opt/my-app/my_flask_app.tar.gz
          dest: /opt/my-app
          remote_src: yes

      - name: Restart the application service
        systemd:
          name: my_flask_app
          state: restarted

      - name: Wait for application to become healthy
        uri:
          url: "http://{{ inventory_hostname }}:5000/health"
          status_code: 200
        register: healthcheck
        retries: 10
        delay: 5
        until: healthcheck.status == 200
  ```

* Ran the playbook to deploy the application **one server at a time**:

  ```bash
  ansible-playbook deploy_flask.yml
  ```

---

## ✅ Outcome

* Successfully deployed and ran the Flask application on the **QA server**.
* Set it up as a **managed systemd service**.
* Used **Ansible rolling update** to safely deploy across multiple nodes with **automated health checks**.
