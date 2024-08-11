# Introduction
This is a writeup on how to get Open WebUI and its additional features running on linux from scratch.  
The aim is to cover provide an A-Z installation guide that takes into consideration any pain points that a user might have.
# Open WebUI
## Setup WSL (Windows)
1) Install WSL and Ubuntu  
`wsl --install`
2) Connect to a WSL Instance in a new window  
`wsl -d Ubuntu`
3) Install Ollama  
Run: `curl -fsSL https://ollama.com/install.sh | sh`  
***Note: If the `curl` fails and you have tailscale running with MagicDNS enabled, stop tailscale first with `sudo tailscale down`***  
***and run the install command before bringing it back up again with `sudo tailscale up --accept-routes`***
4) Add a model to Ollama  
Go to `https://ollama.com/library` and pick the appropriate model and pull it  
Example: `ollama pull llama2`

## "Watch" GPU Performance in Linux
`watch -n 0.5 nvidia-smi`

## Install Docker
### Add Docker's official GPG key
```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
### Add the repository to Apt sources
```
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
### Install Docker
`sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

## Run Open WebUi Docker Container
```
docker run -d \
  --network=host \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --name open-webui \
  --restart always \
  --add-host=host.docker.internal:host-gateway \
  ghcr.io/open-webui/open-webui:main
```
- Open WebUI will now be accessible from http://localhost:8080
### Troubleshooting
**Server Connection Error:**  
If you're not running the Open WebUI docker container with `--network=host` and with the `-p` flag instead and you're experiencing connection issues,  
it’s often due to the WebUI docker container not being able to reach the Ollama server at 127.0.0.1:11434 (host.docker.internal:11434) inside the container.  
Use the `--network=host` flag in your docker command to resolve this. Note that the port changes from 3000 to 8080, resulting in the link: http://localhost:8080.
## Enable TTS from the openedai-speech repository
- Follow the instructions on https://docs.openwebui.com/tutorial/openedai-speech-integration and use Option 1
- At step 2, make sure the `docker-compose.yml` file is created with the following additional line:
```
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

- The final product should look like this:  
```
services:
  server:
    image: ghcr.io/matatonic/openedai-speech
    container_name: openedai-speech
    env_file: .env
    ports:
      - "8000:8000"
    volumes:
      - tts-voices:/app/voices
      - tts-config:/app/config
    # labels:
    #   - "com.centurylinklabs.watchtower.enable=true"
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities:
                - gpu
    extra_hosts:
      - "host.docker.internal:host-gateway"
volumes:
  tts-voices:
  tts-config:
```

## Exposing Open WebUI to the Internet
### Full Exposure - ngrok
- Follow the instructions on ngrok's quickstart page:  
https://ngrok.com/docs/getting-started/?os=linux
- It happens to be the same as the one that's listed on ngrok's quickstart, but make sure that port `8080` is the one that's being exposed.   
Full command: `ngrok http http://localhost:8080`
- Note that there are usage limits for the free tier. Cloudflaretunnel provides unlimited use and is a good alternative, but requires more set up.
### Private Network - tailscale
1) Install tailscale with `curl -fsSL https://tailscale.com/install.sh | sh`
2) Follow the instructions to log into your tailscale account
- Start tailscale with:  
`sudo tailscale up --accept-routes` (accept-routes needs to be included on linux)
- Stop tailscale with:
`sudo tailscale down`
- Check tailscale's status:  
`sudo tailscale status`
3) Run `tailscale serve 8080` OR follow the instructions in the next session to have it start up automatically whenever linux boots up
#### Run tailscale on Startup in the Background (creation of the named service tailscale-serve)
1) Create a service file for tailscale if you want it to start on boot and run as a background service. For a systemd service, create a file in `/etc/systemd/system/tailscale-serve.service` with the following content:  
```
[Unit]
Description=Tailscale Serve Service

[Service]
ExecStart=/usr/bin/tailscale serve 8080
Restart=always

[Install]
WantedBy=multi-user.target
```
2) After creating the file, enable and start the service:  
```
sudo systemctl enable tailscale-serve
sudo systemctl start tailscale-serve
```  
3) To confirm that your tailscale-serve service is running correctly, run the following command:  
`sudo systemctl status tailscale-serve`
- To view detailed logs for the tailscale-serve service, use:  
`journalctl -u tailscale-serve`
- To stop the tailscale-serve service, run the following command:  
`sudo systemctl stop tailscale-serve`  
- To restart the service, run the following command:
`sudo systemctl restart tailscale-serve`
#### Removing tailscale-serve
1) Remove the symlink:  
`sudo rm /etc/systemd/system/multi-user.target.wants/tailscale-serve.service`
2) Remove the service file:  
`sudo rm /etc/systemd/system/tailscale-serve.service`
3) Reload systemd to apply changes:  
`sudo systemctl daemon-reload`
4) Verify the service is removed:  
`sudo systemctl list-unit-files | grep tailscale-serve`