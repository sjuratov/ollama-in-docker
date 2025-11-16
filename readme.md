# Ollama Models Setup with Docker Compose

A step-by-step guide for running Ollama models locally using Docker Compose. This setup allows you to run large language models (LLMs) with persistent storage and optional web UI access.

**Source:** This guide is based on [Setting up Ollama Models with Docker Compose: A Step-by-Step Guide](https://collabnix.com/setting-up-ollama-models-with-docker-compose-a-step-by-step-guide/) by Collabnix.

---

## 1. Install Docker

Before starting, ensure Docker is installed on your system:

### Windows
1. Download [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)
2. Run the installer and follow the setup wizard
3. Docker Compose comes bundled with Docker Desktop
4. Verify installation:
   ```powershell
   docker --version
   docker-compose --version
   ```

### macOS
1. Download [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/)
2. Open the `.dmg` file and drag Docker to Applications
3. Launch Docker Desktop from Applications
4. Docker Compose comes bundled with Docker Desktop
5. Verify installation:
   ```bash
   docker --version
   docker compose version
   ```

### Linux
```bash
# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose plugin
sudo apt-get update
sudo apt-get install docker-compose-plugin

# Verify installation
docker --version
docker compose version
```

### Optional: NVIDIA Container Toolkit (for GPU support)
If you have an NVIDIA GPU and want to leverage GPU acceleration:

**Linux:**
```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

**Windows:** GPU support is available through Docker Desktop with WSL2 backend.

---

## 2. Run Docker Compose

### Understanding the Docker Compose Configuration

The included `docker-compose.yml` file sets up:
- **Ollama service**: The main LLM engine running on port 11434
- **WebUI service**: A web interface accessible on port 3000
- **Persistent storage**: Model data stored in the `./data` directory

### Volume Prerequisites

#### Windows
The `./data` directory will be automatically created in your project folder. Docker Desktop must have access to the drive:
1. Open Docker Desktop
2. Go to Settings → Resources → File Sharing
3. Ensure your drive (e.g., `C:\`) is shared
4. The `./data` folder will be created automatically and mapped to `/root/.ollama` inside the container

#### Linux
```bash
# Create the data directory if it doesn't exist
mkdir -p ./data

# Set appropriate permissions
chmod -R 755 ./data
```

### CPU vs GPU Runtime

**CPU-only runtime** (default in this setup):
- Works on any system without GPU requirements
- Slower inference times, especially for larger models
- Suitable for smaller models (7B parameters and below)
- The current `docker-compose.yml` is configured for CPU

**GPU runtime** (optional):
To enable GPU support, modify `docker-compose.yml` to add GPU configuration:
```yaml
services:
  ollama:
    # ... existing configuration ...
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

### Starting the Services

Run the container in the background (detached mode):

```powershell
# Windows PowerShell
docker-compose up -d
```

```bash
# Linux
docker compose up -d
```

**What happens:**
1. Docker pulls the `ollama/ollama:latest` and `ollama-webui` images (first run only)
2. Creates the `ollama` container with port 11434 exposed
3. Creates the `ollama-webui` container with port 3000 exposed
4. Mounts the `./data` directory for persistent model storage
5. Starts both services in the background

Verify the containers are running:
```powershell
docker ps
```

You should see both `ollama` and `ollama-webui` containers listed.

---

## 3. "Hello World" Test - Verify Ollama is Running

Before pulling models, verify the Ollama service is responding:

```powershell
curl http://localhost:11434/api/tags
```

**Expected output:** An empty list `{"models":[]}` (if no models are pulled yet) or a JSON list of installed models.

If you get a connection error, check the logs:
```powershell
docker-compose logs ollama
```

---

## 4. Pulling Models

There are two methods to pull models into Ollama:

### Method 1: Using Docker Exec (Recommended)

Pull a model using the Ollama CLI inside the container:

```powershell
docker exec -it ollama ollama pull qwen2.5:3b
```

**What this does:**
- `docker exec -it ollama` - Executes a command inside the running `ollama` container
- `ollama pull qwen2.5:3b` - Downloads the Qwen 2.5 3B model from the Ollama registry

Other popular models to try:
```powershell
docker exec -it ollama ollama pull llama3.2      # Meta's Llama 3.2
docker exec -it ollama ollama pull phi3          # Microsoft's Phi-3
docker exec -it ollama ollama pull mistral       # Mistral 7B
docker exec -it ollama ollama pull codellama     # Code-specialized Llama
```

### Method 2: Using curl and the REST API

Pull a model via HTTP API:

```powershell
curl -X POST http://localhost:11434/api/pull -H "Content-Type: application/json" -d '{"name": "qwen2.5:3b"}'
```

**Note:** This method provides real-time progress updates in JSON format.

### Verify Downloaded Models

List all pulled models:
```powershell
docker exec -it ollama ollama list
```

Or via API:
```powershell
curl http://localhost:11434/api/tags
```

---

## 5. Testing Your Model

Once a model is pulled, test it with a simple prompt:

### Using curl
```powershell
curl -X POST http://localhost:11434/api/generate -H "Content-Type: application/json" -d '{
  "model": "qwen2.5:3b",
  "prompt": "Explain Docker Compose in one paragraph",
  "stream": false
}'
```

**Response:** You'll receive a JSON object with the model's response in the `response` field.

### Using the Ollama CLI
```powershell
docker exec -it ollama ollama run qwen2.5:3b "Explain Docker Compose in one paragraph"
```

### Interactive Chat
Start an interactive chat session:
```powershell
docker exec -it ollama ollama run qwen2.5:3b
```

Type your messages and press Enter. Type `/bye` to exit.

---

## 6. Managing Your Ollama Setup

### Essential Commands

**Start services:**
```powershell
docker-compose up -d
```

**Stop services:**
```powershell
docker-compose down
```

**View logs:**
```powershell
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f ollama
```

**Restart services:**
```powershell
docker-compose restart
```

**Rebuild and restart (after config changes):**
```powershell
docker-compose up -d --build
```

**Remove volumes (⚠️ WARNING: deletes all models!):**
```powershell
docker-compose down -v
```

### Model Management

**List all models:**
```powershell
docker exec -it ollama ollama list
```

**Remove a specific model:**
```powershell
docker exec -it ollama ollama rm qwen2.5:3b
```

**Show model information:**
```powershell
docker exec -it ollama ollama show qwen2.5:3b
```

### Performance Tuning (Optional)

For better performance with multiple requests, add environment variables to the `ollama` service in `docker-compose.yml`:

```yaml
services:
  ollama:
    # ... existing configuration ...
    environment:
      - OLLAMA_NUM_PARALLEL=4          # Parallel requests per model
      - OLLAMA_MAX_LOADED_MODELS=3     # Max models in memory
```

---

## 7. Connect to WebUI

The WebUI provides a ChatGPT-like interface for interacting with your models.

### Access the WebUI

Open your browser and navigate to:
```
http://localhost:3000
```

### First-Time Setup
1. The WebUI will automatically connect to the Ollama API at `http://ollama:11434/api`
2. Create an account (stored locally, no external registration)
3. Select a model from the dropdown menu
4. Start chatting!

### WebUI Features
- Multiple chat sessions
- Model switching on the fly
- Conversation history
- Markdown rendering
- Code syntax highlighting
- Model parameters adjustment (temperature, top_p, etc.)

### Troubleshooting WebUI Connection

If the WebUI can't connect to Ollama:

1. **Check container networking:**
   ```powershell
   docker network inspect ollama_default
   ```
   Both containers should be on the same network.

2. **Verify Ollama is accessible from WebUI:**
   ```powershell
   docker exec -it ollama-webui curl http://ollama:11434/api/tags
   ```

3. **Check environment variables:**
   Ensure `OLLAMA_API_BASE_URL=http://ollama:11434/api` is set in the webui service.

---

## Troubleshooting

### GPU Not Detected
1. Verify NVIDIA Container Toolkit is installed
2. Check GPU drivers: `nvidia-smi`
3. Add to `docker-compose.yml`:
   ```yaml
   environment:
     - NVIDIA_VISIBLE_DEVICES=all
     - NVIDIA_DRIVER_CAPABILITIES=compute,utility
   ```

### Out of Memory Errors
1. Use smaller models (e.g., `phi3` instead of `llama3.2:70b`)
2. Limit loaded models: `OLLAMA_MAX_LOADED_MODELS=1`
3. Add memory limits:
   ```yaml
   deploy:
     resources:
       limits:
         memory: 16G
   ```

### Port Already in Use
If ports 11434 or 3000 are occupied:
```yaml
ports:
  - "11435:11434"  # Change external port
  - "3001:8080"    # Change external port
```

### Models Disappearing After Restart
Ensure the volume mapping is correct in `docker-compose.yml`:
```yaml
volumes:
  - ./data:/root/.ollama
```

Check if the `./data` directory exists and has proper permissions.

---

## Additional Resources

- [Ollama Official Documentation](https://ollama.com/)
- [Ollama Model Library](https://ollama.com/library)
- [Ollama GitHub Repository](https://github.com/ollama/ollama)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

## License

This setup guide is based on the [Collabnix tutorial](https://collabnix.com/setting-up-ollama-models-with-docker-compose-a-step-by-step-guide/). The Ollama project and its components have their own respective licenses.
