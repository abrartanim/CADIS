# PentestAgent Windows & Docker Hybrid Setup Guide (V1)

This report documents the steps and fixes required to run **PentestAgent locally on Windows** (Host) while utilizing a **Docker container solely as an isolated tools execution environment** (using Kali Linux tools).

---

## 📋 Architectural Overview
* **Orchestration / LLM / UI**: Runs directly on the Windows host machine in a Python virtual environment. Reads your local configurations (`.env`) and manages the interactive TUI.
* **Tools Sandbox**: Runs in an isolated Linux container using `Dockerfile.kali` (built from `kalilinux/kali-last-release`). The Windows host runs commands inside this container using Docker's Exec API (`exec_run`).

---

## 🛠️ Issues Encountered & Solutions

### 1. Docker Build Crashing / Read-Only Filesystem (WSL2 Disk Space Exhaustion)
* **Symptom**: During `docker compose build`, the build process timed out/hung for over 20 minutes and ended with `open: Read-only file system` and `rpc error: ... EOF`.
* **Root Cause**: The standard `Dockerfile` attempts to install heavy web browser dependencies (`playwright install-deps` and `playwright install`). On WSL2/Docker Desktop for Windows, this massive download/compile process frequently runs out of allocated virtual disk space or memory, crashing the WSL2 backend and forcing the virtual disk into read-only mode to prevent corruption.
* **Solution**: 
  1. Restarted WSL2 (`wsl --shutdown`) and Docker Desktop to clear the read-only disk lock.
  2. Cleaned stale Docker caches (`docker system prune -f`).
  3. Bypassed the standard image entirely and built **only** the Kali tools image (`Dockerfile.kali`), tagging it directly as `pentestagent:latest` (the name the local python code expects):
     ```powershell
     docker build -t pentestagent:latest -f Dockerfile.kali .
     ```

### 2. Linux Entrypoint Script Crash (`no such file or directory`)
* **Symptom**: The container would instantly stop after creation. Checking `docker logs pentestagent-sandbox` revealed:
  ```text
  exec /entrypoint.sh: no such file or directory
  ```
* **Root Cause**: Git on Windows automatically clones text files with Windows Carriage Return line endings (**CRLF**). When copied into a Linux container, the shell interpreter (`/bin/bash`) fails to parse the script because of the carriage return (`\r`) character.
* **Solution**: Converted the line endings of `docker-entrypoint.sh` from **CRLF** to Linux **LF** using VS Code (or a .NET PowerShell regex replace) and rebuilt the image.

### 3. Container Exiting Immediately (Default Command Issue)
* **Symptom**: After fixing the line endings, the container still exited immediately upon start (Status: `Exited (255)`).
* **Root Cause**: The default command (`CMD`) in `Dockerfile.kali` was configured to run the agent itself (`python3 -m pentestagent`). Because the container starts detached and doesn't have local API keys or an interactive terminal attached, the Python process fails validation and exits, shutting down the container.
* **Solution**: Overrode the default command in `Dockerfile.kali` (line 92) to keep the container running silently in the background:
  ```dockerfile
  CMD ["sleep", "infinity"]
  ```
  This keeps the sandbox alive indefinitely so the Windows host can run `nmap`, `msfconsole`, etc. inside it at any time.

---

## 🔍 Verification Tests Run
To verify the setup, we ran the following TUI command on Windows:
```text
/assist Run the command 'whoami' in your terminal and check if the 'nmap' tool is installed.
```

* **Result**: The agent successfully executed the command inside the container and returned `root`.
* **Validation**: This confirmed that commands are running inside the Linux container (as root) rather than the local Windows command prompt.

---

## 💻 Useful Commands Reference

### Local Setup & Development Commands
* **Activate Virtual Environment (PowerShell)**:
  ```powershell
  .\venv\Scripts\Activate.ps1
  ```
* **Install Local Project Dependencies (Editable Mode)**:
  ```powershell
  pip install -e ".[all]"
  ```
* **Run PentestAgent TUI locally with Docker Tools Sandbox**:
  ```powershell
  pentestagent tui --docker
  ```
* **Run Headless Task with Docker Tools Sandbox**:
  ```powershell
  pentestagent run -t <target> --docker --task "<instructions>"
  ```

### Docker Management Commands
* **Convert line endings of entrypoint (PowerShell)**:
  ```powershell
  [System.IO.File]::WriteAllText((Get-Item .\docker-entrypoint.sh).FullName, [System.IO.File]::ReadAllText((Get-Item .\docker-entrypoint.sh).FullName).Replace("`r`n", "`n"))
  ```
* **Build ONLY the Kali Sandbox Image**:
  ```powershell
  docker build -t pentestagent:latest -f Dockerfile.kali .
  ```
* **List All Container Statuses (running & stopped)**:
  ```powershell
  docker ps -a
  ```
* **Check Sandbox Container Logs**:
  ```powershell
  docker logs pentestagent-sandbox
  ```
* **Force-Remove Sandbox Container (to clear stale state/conflict)**:
  ```powershell
  docker rm -f pentestagent-sandbox
  ```
* **Clean up unused Docker storage / build caches**:
  ```powershell
  docker system prune -f
  ```
