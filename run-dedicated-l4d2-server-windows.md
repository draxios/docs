# Run a L4D2 Dedicated Server on Windows

### Step 1: Install SteamCMD

**SteamCMD** is a command-line version of the Steam client, used to install and update dedicated servers for various games.

1. **Download SteamCMD**:
   - Go to the [SteamCMD download page](https://developer.valvesoftware.com/wiki/SteamCMD).
   - Click on the Windows download link to download the `steamcmd.zip` file.

2. **Extract SteamCMD**:
   - Create a folder where you want to install SteamCMD (e.g., `C:\SteamCMD`).
   - Extract the contents of `steamcmd.zip` to this folder.

### Step 2: Install Left 4 Dead 2 Dedicated Server

1. **Open Command Prompt**:
   - Press `Win + R`, type `cmd`, and press Enter.

2. **Navigate to the SteamCMD directory**:
   ```cmd
   cd C:\SteamCMD
   ```

3. **Run SteamCMD**:
   ```cmd
   steamcmd
   ```

4. **Login to Steam**:
   - In the SteamCMD console, login as anonymous:
     ```cmd
     login anonymous
     ```

5. **Install the Left 4 Dead 2 dedicated server**:
   - Set the installation directory:
     ```cmd
     force_install_dir C:\l4d2_server
     ```
   - Install the server:
     ```cmd
     app_update 222860 validate
     ```

### Step 3: Configure the Server

1. **Navigate to the server configuration directory**:
   ```cmd
   cd C:\l4d2_server\left4dead2\cfg
   ```

2. **Create or edit the `server.cfg` file**:
   - Use a text editor (like Notepad) to create or edit the `server.cfg` file.
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

### Step 4: Install MetaMod and SourceMod

**MetaMod** and **SourceMod** are required to run plugins like SirPlease's and draxios's mods.

1. **Download MetaMod**:
   - Go to the [MetaMod: Source website](https://www.metamodsource.net/).
   - Download the latest version of MetaMod for Windows.

2. **Extract MetaMod**:
   - Extract the MetaMod files to `C:\l4d2_server\left4dead2\addons`.

3. **Download SourceMod**:
   - Go to the [SourceMod website](https://www.sourcemod.net/).
   - Download the latest version of SourceMod for Windows.

4. **Extract SourceMod**:
   - Extract the SourceMod files to `C:\l4d2_server\left4dead2\addons`.

### Step 5: Verify MetaMod and SourceMod Installation

1. **Verify MetaMod installation**:
   - Ensure the `C:\l4d2_server\left4dead2\addons\metamod.vdf` file exists.

2. **Verify SourceMod installation**:
   - The `addons` folder should contain both `metamod` and `sourcemod` folders.

### Step 6: Install SirPlease L4D2-Competitive-Rework

1. **Download the L4D2-Competitive-Rework**:
   - Go to the [L4D2-Competitive-Rework GitHub page](https://github.com/SirPlease/L4D2-Competitive-Rework).
   - Click on the `Code` button and select `Download ZIP`.

2. **Extract the L4D2-Competitive-Rework files**:
   - Extract the downloaded ZIP file.
   - Copy the contents of the extracted folder to `C:\l4d2_server\left4dead2`, overlaying the existing files.

### Step 7: Install draxios's bizzymod

1. **Download bizzymod**:
   - Go to the [bizzymod GitHub page](https://github.com/draxios/bizzymod).
   - Click on the `Code` button and select `Download ZIP`.

2. **Extract the bizzymod files**:
   - Extract the downloaded ZIP file.
   - Copy the contents of the extracted folder to `C:\l4d2_server\left4dead2`, overlaying the existing files.

### Step 8: Run the Server

1. **Navigate to the server directory**:
   ```cmd
   cd C:\l4d2_server
   ```

2. **Start the server**:
   ```cmd
   srcds.exe -console -game left4dead2 +map c1m1_hotel
   ```
   - This command will start the server with the specified map.

### Step 9: Testing and Configuration

1. **Connect to your server**:
   - Launch Left 4 Dead 2 on your client machine.
   - Open the console (`~` key) and type:
     ```cmd
     connect localhost
     ```

2. **Test SourceMod**:
   - Use the console to test if SourceMod commands are working. For example:
     ```cmd
     sm_admin
     ```
   - This should open the SourceMod admin menu if everything is set up correctly.

### Step 10: Further Customization

1. **SourceMod Configs**:
   - Place your SourceMod configuration files in `C:\l4d2_server\left4dead2\addons\sourcemod\configs`.

2. **Plugins**:
   - Add SourceMod plugins to `C:\l4d2_server\left4dead2\addons\sourcemod\plugins`.

3. **Scripts**:
   - Add custom scripts to `C:\l4d2_server\left4dead2\addons\sourcemod\scripting`.

