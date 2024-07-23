# Run Left 4 Dead 2 dedicated server on Ubuntu 22.04 in AWS

### Step 1: Launch an EC2 Instance

1. **Log in to AWS Management Console**:
   - Go to the [AWS Management Console](https://aws.amazon.com/console/).
   - Sign in with your AWS account.

2. **Launch an EC2 Instance**:
   - Navigate to the EC2 dashboard and click `Launch Instance`.
   - Select `Ubuntu Server 22.04 LTS (HVM), SSD Volume Type`.
   - Choose an instance type, e.g., `t2.medium` for a balance of performance and cost.
   - Configure the instance details, e.g., default settings are usually fine for basic setups.
   - Add storage, e.g., 30 GB of General Purpose SSD (gp2).
   - Configure security group to allow necessary ports:
     - Add rules for SSH (port 22), Game Traffic (port 27015 for Source engine games), and RCON (port 27015 if you want to use it).

3. **Review and Launch**:
   - Review your instance settings.
   - Click `Launch`.
   - Choose an existing key pair or create a new one, and download it.

### Step 2: Connect to Your EC2 Instance

1. **Open a Terminal**:
   - On your local machine, open a terminal.

2. **Connect via SSH**:
   - Use the following command to connect to your EC2 instance (replace `your-key-pair.pem` and `your-ec2-public-dns` with your actual key pair and instance DNS):
     ```sh
     chmod 400 your-key-pair.pem
     ssh -i "your-key-pair.pem" ubuntu@your-ec2-public-dns
     ```

### Step 3: Install Dependencies

1. **Update Packages**:
   ```sh
   sudo apt update && sudo apt upgrade -y
   ```

2. **Install Required Packages**:
   ```sh
   sudo apt install -y lib32gcc1 lib32stdc++6 wget tmux unzip
   ```

### Step 4: Install SteamCMD

1. **Create a SteamCMD Directory**:
   ```sh
   mkdir ~/SteamCMD
   cd ~/SteamCMD
   ```

2. **Download and Extract SteamCMD**:
   ```sh
   wget https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz
   tar -xvzf steamcmd_linux.tar.gz
   ```

### Step 5: Install Left 4 Dead 2 Dedicated Server

1. **Run SteamCMD**:
   ```sh
   ./steamcmd.sh
   ```

2. **Install the Server**:
   - In the SteamCMD prompt, run the following commands:
     ```sh
     login anonymous
     force_install_dir ~/l4d2_server
     app_update 222860 validate
     quit
     ```

### Step 6: Configure the Server

1. **Navigate to the server configuration directory**:
   ```sh
   cd ~/l4d2_server/left4dead2/cfg
   ```

2. **Create or edit the `server.cfg` file**:
   - Use a text editor like `nano` to create or edit the `server.cfg` file:
     ```sh
     nano server.cfg
     ```
   - Add the following lines to configure your server:
     ```cfg
     hostname "My L4D2 Server"
     rcon_password "your_rcon_password"
     sv_password "your_server_password"  // Leave empty for no password
     exec banned_user.cfg
     exec banned_ip.cfg
     sv_lan 0
     sv_region 255
     sv_allow_lobby_connect_only 0
     sv_maxplayers 4
     ```

### Step 7: Install MetaMod and SourceMod

1. **Download MetaMod**:
   ```sh
   wget https://mms.alliedmods.net/mmsdrop/1.11/mmsource-1.11.0-git1148-linux.tar.gz
   tar -xzvf mmsource-1.11.0-git1148-linux.tar.gz -C ~/l4d2_server/left4dead2/
   ```

2. **Download SourceMod**:
   ```sh
   wget https://sm.alliedmods.net/smdrop/1.11/sourcemod-1.11.0-git6936-linux.tar.gz
   tar -xzvf sourcemod-1.11.0-git6936-linux.tar.gz -C ~/l4d2_server/left4dead2/
   ```

### Step 8: Verify MetaMod and SourceMod Installation

1. **Verify MetaMod installation**:
   - Ensure the `~/l4d2_server/left4dead2/addons/metamod.vdf` file exists.

2. **Verify SourceMod installation**:
   - The `addons` folder should contain both `metamod` and `sourcemod` folders.

### Step 9: Install SirPlease L4D2-Competitive-Rework

1. **Download the L4D2-Competitive-Rework**:
   ```sh
   wget https://github.com/SirPlease/L4D2-Competitive-Rework/archive/refs/heads/master.zip
   unzip master.zip
   ```

2. **Copy the files**:
   ```sh
   cp -r L4D2-Competitive-Rework-master/* ~/l4d2_server/left4dead2/
   ```

### Step 10: Install draxios's bizzymod

1. **Download bizzymod**:
   ```sh
   wget https://github.com/draxios/bizzymod/archive/refs/heads/master.zip
   unzip master.zip
   ```

2. **Copy the files**:
   ```sh
   cp -r bizzymod-master/* ~/l4d2_server/left4dead2/
   ```

### Step 11: Run the Server

1. **Navigate to the server directory**:
   ```sh
   cd ~/l4d2_server
   ```

2. **Start the server**:
   - Use `tmux` to keep the server running even after disconnecting:
     ```sh
     tmux
     ./srcds_run -console -game left4dead2 +map c1m1_hotel
     ```

3. **Detach from `tmux`**:
   - Press `Ctrl + B`, then `D` to detach from the `tmux` session and leave the server running.

### Step 12: Testing and Configuration

1. **Connect to your server**:
   - Launch Left 4 Dead 2 on your client machine.
   - Open the console (`~` key) and type:
     ```sh
     connect your-ec2-public-dns:27015
     ```

2. **Test SourceMod**:
   - Use the console to test if SourceMod commands are working. For example:
     ```sh
     sm_admin
     ```
   - This should open the SourceMod admin menu if everything is set up correctly.

### Step 13: Further Customization

1. **SourceMod Configs**:
   - Place your SourceMod configuration files in `~/l4d2_server/left4dead2/addons/sourcemod/configs`.

2. **Plugins**:
   - Add SourceMod plugins to `~/l4d2_server/left4dead2/addons/sourcemod/plugins`.

3. **Scripts**:
   - Add custom scripts to `~/l4d2_server/left4dead2/addons/sourcemod/scripting`.
