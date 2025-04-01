# Simple Guide to Run Multiple Nodes on Gaianet (Works and tested only for DHARMA domain)

Note: 
-Need minimum 10Gb ram/vram
-Recommended atleast 3080 Gpu+ series
-POWER of GPU's should be above 350W+

## **1. Install Required Packages and CUDA 12.8**

```bash
apt update && apt upgrade -y
apt-get update && apt-get upgrade -y

apt install pciutils lsof curl nvtop btop -y
apt-get install jq -y

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
apt-get update
apt-get -y install cuda-toolkit-12-8
```

## **2. Install and Configure the First Node**

```bash
i=101
home_dir="gaia-node-$i"
downloads='gaia-node-101'
gaia_port="8$i"
gaia_config="qwen-2.5-coder-3b-instruct"

lsof -t -i:$gaia_port | xargs kill -9
rm -rf $HOME/$home_dir
curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/uninstall.sh' | bash

mkdir $HOME/$home_dir

curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/install.sh' | bash -s -- --ggmlcuda 12 --base $HOME/$home_dir

source /root/.bashrc

wget -O "$HOME/$home_dir/config.json" https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/qwen-2.5-coder-7b-instruct_rustlang/config.json
CONFIG_FILE="$HOME/$home_dir/config.json"
jq '.chat = "https://huggingface.co/gaianet/Qwen2.5-Coder-3B-Instruct-GGUF/resolve/main/Qwen2.5-Coder-3B-Instruct-Q5_K_M.gguf"' "$CONFIG_FILE" > tmp.$$.json && mv tmp.$$.json "$CONFIG_FILE"
jq '.chat_name = "Qwen2.5-Coder-3B-Instruct"' "$CONFIG_FILE" > tmp.$$.json && mv tmp.$$.json "$CONFIG_FILE"
grep '"chat":' $HOME/$home_dir/config.json
grep '"chat_name":' $HOME/$home_dir/config.json
gaianet config --base $HOME/$home_dir --port $gaia_port
gaianet init --base $HOME/$home_dir
```

## **3. Install and Configure the Second Node**

```bash
i=102
home_dir="gaia-node-$i"
downloads='gaia-node-102'
gaia_port="8$i"
gaia_config="qwen-2.5-coder-3b-instruct"

lsof -t -i:$gaia_port | xargs kill -9
rm -rf $HOME/$home_dir
curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/uninstall.sh' | bash

mkdir $HOME/$home_dir

curl -sSfL 'https://github.com/GaiaNet-AI/gaianet-node/releases/latest/download/install.sh' | bash -s -- --ggmlcuda 12 --base $HOME/$home_dir

source /root/.bashrc

wget -O "$HOME/$home_dir/config.json" https://raw.githubusercontent.com/GaiaNet-AI/node-configs/main/qwen-2.5-coder-7b-instruct_rustlang/config.json
CONFIG_FILE="$HOME/$home_dir/config.json"
jq '.chat = "https://huggingface.co/gaianet/Qwen2.5-Coder-3B-Instruct-GGUF/resolve/main/Qwen2.5-Coder-3B-Instruct-Q5_K_M.gguf"' "$CONFIG_FILE" > tmp.$$.json && mv tmp.$$.json "$CONFIG_FILE"
jq '.chat_name = "Qwen2.5-Coder-3B-Instruct"' "$CONFIG_FILE" > tmp.$$.json && mv tmp.$$.json "$CONFIG_FILE"
grep '"chat":' $HOME/$home_dir/config.json
grep '"chat_name":' $HOME/$home_dir/config.json
gaianet config --base $HOME/$home_dir --port $gaia_port
gaianet init --base $HOME/$home_dir
```

## **4. Start the Nodes**

### **Option 1: Running Two Nodes on the Same GPU**
```bash
gaianet start --base $HOME/gaia-node-101
gaianet start --base $HOME/gaia-node-102
```

### **Option 2: Running Two Nodes on Separate GPUs**
```bash
CUDA_VISIBLE_DEVICES=0 gaianet start --base $HOME/gaia-node-101
CUDA_VISIBLE_DEVICES=1 gaianet start --base $HOME/gaia-node-102
```

## **5. Get Node's id & Device id**

### **Option 1: Running Two Nodes on the Same GPU**
```bash
gaianet info --base $HOME/gaia-node-101
gaianet info --base $HOME/gaia-node-102
```

## **6. GPU Activity Monitoring Script**

Modify the Start Nodes command according to your requirements in the script below:

```bash
nano monitor_gpu_activity.sh
```

Paste the following script inside `monitor_gpu_activity.sh`:

```bash
#!/bin/bash

# Function to check GPU utilization
check_gpu_utilization() {
    utilization=$(nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits)
    echo $utilization
}

# Variables
check_interval=3  # Interval between checks in seconds
inactivity_threshold=240  # Inactivity threshold in seconds
inactivity_duration=0

while true; do
    gpu_util=$(check_gpu_utilization)
    if [ "$gpu_util" -eq 0 ]; then
        inactivity_duration=$((inactivity_duration + check_interval))
        echo "$inactivity_duration"
        if [ "$inactivity_duration" -ge "$inactivity_threshold" ]; then
            # GPU has been inactive for the threshold duration
            # Execute desired bash commands
            echo "GPU has been inactive for over a minute."
            echo "Restarting Gaia node."
            gaianet stop --base $HOME/gaia-node-101
            gaianet stop --base $HOME/gaia-node-102
            gaianet start --base $HOME/gaia-node-101
            gaianet start --base $HOME/gaia-node-102
            echo "Gaia node restarted."
            inactivity_duration=0  # Reset inactivity duration after executing commands
        fi
    else
        inactivity_duration=0  # Reset if GPU is active
    fi
    sleep $check_interval
done
```

Save the file and make it executable:

```bash
chmod +x monitor_gpu_activity.sh
```

## **7. Using Screen to Run the Monitoring Script in the Background**

### **Install Screen**
```bash
apt install screen
```

### **Create a New Screen Session**
```bash
screen -S monitor_gpu_activity
```

### **Start the Monitoring Script**
```bash
./monitor_gpu_activity.sh
```

### **Detach from Screen Session**
Press `Ctrl + A`, then `D`.

### **Reconnect to the Screen Session**
```bash
screen -r monitor_gpu_activity
```
