# 🚀 Simple Guide to Run Gaia Nodes on Gaianet  
**Tested and verified for HASHATAG domain only**

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
rm -rf ~/packageskit.sh; curl -O https://raw.githubusercontent.com/Jayanth2407/HASHTAG_Gaia_Domain/main/packageskit.sh; chmod +x packageskit.sh; ./packageskit.sh
```

---

## ⚙️ 2. Install and Configure the Node

```bash
home_dir="gaianet"
gaia_port="8000"
gaia_config="Qwen2.5-0.5B-Instruct-Q5_K_M"
lsof -t -i:$gaia_port | xargs kill -9
rm -rf $HOME/$home_dir
curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/uninstall.sh' | bash
mkdir $HOME/$home_dir
curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/install.sh' | bash -s -- --ggmlcuda 12 --base $HOME/$home_dir
source /root/.bashrc
wget -O "$HOME/$home_dir/config.json" https://raw.githubusercontent.com/Jayanth2407/HASHTAG_Gaia_Domain/main/config.json
CONFIG_FILE="$HOME/$home_dir/config.json"
jq '.chat = "https://huggingface.co/gaianet/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/Qwen2.5-0.5B-Instruct-Q5_K_M.gguf"' "$CONFIG_FILE" > tmp.json && mv tmp.json "$CONFIG_FILE"
jq '.chat_name = "Qwen2.5-0.5B-Instruct-Q5_K_M"' "$CONFIG_FILE" > tmp.json && mv tmp.json "$CONFIG_FILE"
grep '"chat":' $CONFIG_FILE
grep '"chat_name":' $CONFIG_FILE
gaianet config --base $HOME/$home_dir --port $gaia_port
gaianet init --base $HOME/$home_dir
```

---

## ▶️ 3. To Start the Node

```bash
gaianet start --base $HOME/gaianet
```

---

## 🆔 4. Get Node and Device Info

```bash
gaianet info --base $HOME/gaianet
```

---

## 🔴 5. To Stop the Node

```bash
gaianet stop --base $HOME/gaianet
```

---

## 🤖 6. Chatbot Script Setup

> **Key Points:**   
- Replace `"gaia-api1"` with your **actual API key(s)**. 
- Add as many api's you want, the chatbot will utilize all api's equally. DOnt add invalid/broken api's which is restricted/banned. 
- Use proxy if need or else leave it as it is. 

### Create/Edit the `chatbot.sh` Script

Paste the following script inside `chatbot.sh`:

```bash
#!/bin/bash

# API keys
API_KEYS=(
    "gaia-api1"
    "gaia-api2"
    "gaia-api3"
)

# Proxies in http://username:password@ip:port format
PROXIES=(
    #"http://user1:pass1@ip1:port"
    #"http://user2:pass2@ip2:port"
    # Add more if needed
    # Leave as it is if you don't want to use proxies, if need remove '#'
)

# Gaia endpoint
API_URL="https://hashtag.gaia.domains/v1/chat/completions"

# Validated proxies - Dont modify anything here
VALID_PROXIES=()

# Validate proxies once at startup
validate_proxies() {
    echo "🔍 Validating proxies..."
    for proxy in "${PROXIES[@]}"; do
        if curl --silent --proxy "$proxy" --max-time 3 https://api.ipify.org > /dev/null; then
            VALID_PROXIES+=("$proxy")
            echo "✅ Proxy working: $proxy"
        else
            echo "❌ Proxy failed: $proxy"
        fi
    done
    echo "🔎 Valid proxies: ${#VALID_PROXIES[@]}"
}

# Question generator
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
        "What is the greenhouse effect and why is it important?"
        "How do airplanes stay in the air despite their weight?"
        "Why do we have different time zones around the world?"
        "What causes tides in the ocean?"
        "How does a rainbow form in the sky?"
        "What is the purpose of the United Nations?"
        "How does a compass work to show direction?"
        "What is the longest railway in the world?"
        "Which element is represented by the symbol 'O' on the periodic table?"
        "Which organ in the human body produces insulin?"
        "What is the deepest ocean in the world?"
        "Who was the first woman to win a Nobel Prize?"
        "Which sport is played at Wimbledon?"
        "Why do leaves change color in autumn?"
    )
    echo "${general_questions[$RANDOM % ${#general_questions[@]}]}"
}

# Send a request with retry and optional proxy use
send_request() {
    local message="$1"
    local api_key="$2"
    local attempt=1
    local max_retries=3

    while [ $attempt -le $max_retries ]; do
        local proxy_param=""
        local proxy_used=""

        if [ ${#VALID_PROXIES[@]} -gt 0 ]; then
            proxy="${VALID_PROXIES[$RANDOM % ${#VALID_PROXIES[@]}]}"
            proxy_param="--proxy $proxy"
            proxy_used=" [via proxy]"
        fi

        json_data=$(cat <<EOF
{
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "$message"}
    ]
}
EOF
        )

        echo "📬 Sending Question using key: ${api_key:0:4}****$proxy_used (Attempt $attempt)"
        echo "📝 Question: $message"

        response=$(timeout 60 curl -s -w "\n%{http_code}" -X POST "$API_URL" \
            -H "Authorization: Bearer $api_key" \
            -H "Accept: application/json" \
            -H "Content-Type: application/json" \
            $proxy_param \
            -d "$json_data" --max-time 10)

        http_status=$(echo "$response" | tail -n1)
        body=$(echo "$response" | head -n -1)
        reply=$(echo "$body" | jq -r '.choices[0].message.content' 2>/dev/null)

        if [[ "$http_status" -eq 200 && -n "$reply" ]]; then
            echo "✅ [SUCCESS]$proxy_used"
            echo "💬 Response received"
            return
        else
            echo "❌ [FAILED] Status: $http_status$proxy_used (Attempt $attempt)"
            ((attempt++))
            sleep 1
        fi
    done

    echo "⛔ Maximum retry limit reached for API key ${api_key:0:4}****"
}

# Calculate thread distribution across APIs
calculate_thread_distribution() {
    local count=${#API_KEYS[@]}
    local base=$((10 / count))
    local remainder=$((10 % count))
    local threads=()

    for ((i = 0; i < count; i++)); do
        threads[i]=$base
        if [ $i -lt $remainder ]; then
            threads[i]=$((threads[i] + 1))
        fi
    done

    echo "${threads[@]}"
}

# Main loop
validate_proxies
while true; do
    THREADS=($(calculate_thread_distribution))
    for i in "${!API_KEYS[@]}"; do
        api_key="${API_KEYS[i]}"
        thread_count="${THREADS[i]}"
        for ((t = 1; t <= thread_count; t++)); do
            msg=$(generate_random_general_question)
            send_request "$msg" "$api_key" &
        done
    done

    wait
    sleep 1
done
```

### Make it Executable

```bash
chmod +x chatbot.sh
```

---

## 📟 7. Run Chatbot Script in Background (using `screen`)

### Install `screen` (if not installed)

```bash
apt install screen
```

### Create a New Screen Session

```bash
screen -S chatbot
```

### Start the Script

```bash
./chatbot.sh
```

### Stop the Script

Press `Ctrl + C`


### Detach from Session

Press `Ctrl + A`, then `D`

### Reconnect Anytime

```bash
screen -r chatbot
```

---

### ✅ You’re all set to run Gaia nodes and interact with the model!
