Here’s a cleaner, more polished version of your README with enhanced formatting, clarity, and grammar while keeping your technical flow intact:

---

# 🚀 Simple Guide to Run Gaia Nodes on Gaianet  
**Tested and verified for `hashtag.gaia.domains` domain only**

---

### 🌐 Domain: `hashtag.gaia.domains`  
### 🤖 Model: `Qwen2.5-0.5B-Instruct-Q5_K_M`

> **Minimum Requirements**  
- 🧠 RAM/VRAM: **10 GB**  
- ⚡ Recommended GPU: **NVIDIA 3080 or above**  
- 🔌 GPU Power: **200W+**

---

## 🔧 1. Install Required Packages and CUDA 12.8

```bash
apt update && apt upgrade -y
apt install pciutils lsof curl nvtop btop jq -y

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
apt-get update
apt-get -y install cuda-toolkit-12-8
```

---

## ⚙️ 2. Install and Configure the Node

```bash
home_dir="gaianet"
gaia_port="8000"
gaia_config="Qwen2.5-0.5B-Instruct-Q5_K_M"

# Kill any process on the same port and cleanup old setup
lsof -t -i:$gaia_port | xargs kill -9
rm -rf $HOME/$home_dir
curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/uninstall.sh' | bash

# Create working directory and install node
mkdir $HOME/$home_dir
curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/install.sh' | bash -s -- --ggmlcuda 12 --base $HOME/$home_dir
source /root/.bashrc

# Download and set config
wget -O "$HOME/$home_dir/config.json" https://raw.githubusercontent.com/Jayanth2407/gaiaNode/main/config2.json
CONFIG_FILE="$HOME/$home_dir/config.json"

jq '.chat = "https://huggingface.co/gaianet/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/Qwen2.5-0.5B-Instruct-Q5_K_M.gguf"' "$CONFIG_FILE" > tmp.json && mv tmp.json "$CONFIG_FILE"
jq '.chat_name = "Qwen2.5-0.5B-Instruct-Q5_K_M"' "$CONFIG_FILE" > tmp.json && mv tmp.json "$CONFIG_FILE"

# Verify config
grep '"chat":' $CONFIG_FILE
grep '"chat_name":' $CONFIG_FILE

# Final initialization
gaianet config --base $HOME/$home_dir --port $gaia_port
gaianet init --base $HOME/$home_dir
```

---

## ▶️ 3. Start the Node

```bash
gaianet start --base $HOME/gaianet
```

---

## 🆔 4. Get Node and Device Info

```bash
gaianet info --base $HOME/gaianet
```

---

## 🤖 5. Chatbot Script Setup

> **Key Points:**  
- Each node can handle **max 10 requests per cycle**.  
- Replace `"gaia-api1"` with your **actual API key(s)**.  
- If using 1 API key → `THREAD_COUNT=10`  
- If using 5 API keys → `THREAD_COUNT=2` (5×2=10 total requests per cycle)

### Create/Edit the `chatbot.sh` Script

Paste the following script inside `chatbot.sh`:

```bash
#!/bin/bash

# API keys (defined like general questions, one per line)
API_KEYS=(
    "gaia-api1"
    "gaia-api2"
    "gaia-api3"
)

# Single API URL
API_URL="https://hashtag.gaia.domains/v1/chat/completions"

# Thread count (can be changed as needed)
THREAD_COUNT=2

# Function to get a random general question
generate_random_general_question() {
    general_questions=(
    "What is the national currency of the United Kingdom?"
    "Which element is necessary for breathing and survival?"
    "What is the tallest mountain in the world?"
    "Which is the largest desert in the world?"
    "Who painted the famous artwork Mona Lisa?"
    "What is the capital of Australia?"
    "Which gas is most abundant in Earth's atmosphere?"
    "Who discovered penicillin?"
    "Which continent has the most countries?"
	  "What do you wear on your head when riding a bike?"
    "Which is the smallest country in the world by land area?"
    "What is the chemical symbol for gold?"
    "Who was the first President of the United States?"
    "Which planet has the most moons in our solar system?"
    "What is the hardest natural substance on Earth?"
    "Which ocean is the largest by surface area?"
    "Who wrote the play Romeo and Juliet?"
    "What is the national flower of India?"
    "How many bones are there in the adult human body?"
    "Which bird is known for its ability to mimic human speech?"
    "What is the currency of Japan?"
    "Which is the longest wall in the world?"
    "What is the main ingredient in traditional Japanese miso soup?"
    "Which is the only planet that rotates on its side?"
    "What is the name of the fairy tale character who leaves a glass slipper behind at a royal ball?"
    "Who invented the light bulb?"
    "Which country is famous for the Great Pyramids of Giza?"
    "What is the chemical formula of water?"
    "What is the fastest land animal in the world?"
    "Who is known as the 'Father of Computers'?"
    "Which two colors are on the flag of Canada?"
    "Which planet is the hottest in the solar system?"
    "Who wrote the famous book The Origin of Species?"
    "What is the main language spoken in Brazil?"
    "Which country is known as the Land of the Rising Sun?"
    "What is the longest railway in the world?"
    "Which element is represented by the symbol 'O' on the periodic table?"
    "Which organ in the human body produces insulin?"
    "What is the deepest ocean in the world?"
    "Who was the first woman to win a Nobel Prize?"
    "Which sport is played at Wimbledon?"
    "Why do leaves change color in autumn?"
    "What is the greenhouse effect and why is it important?"
    "How do airplanes stay in the air despite their weight?"
    "Why do we have different time zones around the world?"
    "What causes tides in the ocean?"
    "How does a rainbow form in the sky?"
    "What is the purpose of the United Nations?"
    "How does a compass work to show direction?"
    )

    echo "${general_questions[$RANDOM % ${#general_questions[@]}]}"
}

# Function to handle the API request with retries and exponential backoff
send_request() {
    local message="$1"
    local api_key="$2"
    local retry_count=0
    local max_retries=3
    local initial_delay=1

    while [ $retry_count -lt $max_retries ]; do
        echo "📬 Sending Question using key: ${api_key:0:4}****"  # Mask API key for security
        echo "📝 Question: $message"

        json_data=$(cat <<EOF
{
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "$message"}
    ]
}
EOF
        )

        response=$(curl -s -w "\n%{http_code}" -X POST "$API_URL" \
            -H "Authorization: Bearer $api_key" \
            -H "Accept: application/json" \
            -H "Content-Type: application/json" \
            -d "$json_data" --max-time 10)  # Add a timeout to avoid hanging

        http_status=$(echo "$response" | tail -n 1)
        body=$(echo "$response" | head -n -1)

        # Extract the 'content' from the JSON response using jq (Suppress errors)
        response_message=$(echo "$body" | jq -r '.choices[0].message.content' 2>/dev/null)

        if [[ "$http_status" -eq 200 ]]; then
            if [[ -z "$response_message" ]]; then
                echo "⚠️ Response content is empty!"
            else
                echo "✅ [SUCCESS] Response Received!"
                echo "💬 Response: $response_message"
            fi
            return 0
        else
            echo "⚠️ [ERROR] API request failed | Status: $http_status | Retrying in $initial_delay seconds..."
            sleep $initial_delay
            retry_count=$((retry_count + 1))
            initial_delay=$((initial_delay * 2))  # Exponential backoff
        fi
    done

    echo "❌ [FATAL] Max retries reached. Giving up."
    return 1
}

# Main Loop
while true; do
    # Start threads
    for ((i = 1; i <= THREAD_COUNT; i++)); do
        random_message=$(generate_random_general_question)
        for api_key in "${API_KEYS[@]}"; do
            send_request "$random_message" "$api_key" &
        done
    done

    # Wait for all background processes to finish
    wait

    # Sleep before the next batch of requests
    sleep 1
done
```

### Make it Executable

```bash
chmod +x chatbot.sh
```

---

## 📟 6. Run Chatbot Script in Background (using `screen`)

### Install `screen` (if not installed)

```bash
apt install screen
```

### Create a New Screen Session

```bash
screen -S chatbot
```

### Run the Script

```bash
./chatbot.sh
```

### Detach from Session

Press `Ctrl + A`, then `D`

### Reconnect Anytime

```bash
screen -r chatbot
```

---

### ✅ You’re all set to run Gaia nodes and interact with the model!
