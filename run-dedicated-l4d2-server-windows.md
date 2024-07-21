####Detailed step-by-step guide to setting up a local Left 4 Dead 2 dedicated server on Windows

### Step 1: Install SteamCMD
1. **Download SteamCMD**: Go to the [SteamCMD page](https://developer.valvesoftware.com/wiki/SteamCMD) and download the SteamCMD installer for Windows.
2. **Extract SteamCMD**: Extract the contents of the downloaded file to a directory of your choice, for example `C:\SteamCMD`.

### Step 2: Install Left 4 Dead 2 Dedicated Server
1. **Open Command Prompt**: Press `Win + R`, type `cmd`, and press Enter.
2. **Navigate to SteamCMD directory**: 
   ```cmd
   cd C:\SteamCMD
   ```
3. **Run SteamCMD**: 
   ```cmd
   steamcmd
   ```
4. **Login to Steam**: In the SteamCMD console, login as anonymous:
   ```cmd
   login anonymous
   ```
5. **Install the Left 4 Dead 2 dedicated server**: 
   ```cmd
   force_install_dir C:\l4d2_server
   app_update 222860 validate
   ```
   This will download and install the Left 4 Dead 2 dedicated server to `C:\l4d2_server`.

### Step 3: Configure the Server
1. **Navigate to the server directory**: 
   ```cmd
   cd C:\l4d2_server\left4dead2\cfg
   ```
2. **Create or edit the `server.cfg` file**: Use a text editor to create or edit the `server.cfg` file. Add the following lines to configure your server:
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
1. **Download MetaMod**: Go to the [MetaMod: Source website](https://www.metamodsource.net/) and download the latest version.
2. **Extract MetaMod**: Extract the MetaMod files to `C:\l4d2_server\left4dead2\addons`.
3. **Download SourceMod**: Go to the [SourceMod website](https://www.sourcemod.net/) and download the latest version.
4. **Extract SourceMod**: Extract the SourceMod files to `C:\l4d2_server\left4dead2\addons`.

### Step 5: Verify Installation
1. **Verify MetaMod installation**: Open the file `C:\l4d2_server\left4dead2\addons\metamod.vdf` and ensure it exists.
2. **Verify SourceMod installation**: The `addons` folder should contain both `metamod` and `sourcemod` folders.

### Step 6: Run the Server
1. **Navigate to the server directory**:
   ```cmd
   cd C:\l4d2_server
   ```
2. **Start the server**:
   ```cmd
   srcds.exe -console -game left4dead2 +map c1m1_hotel
   ```
   This command will start the server with the specified map.

### Step 7: Testing and Configuration
1. **Connect to your server**: Launch Left 4 Dead 2 on your client machine, open the console (`~` key), and type:
   ```cmd
   connect localhost
   ```
2. **Test SourceMod**: Use the console to test if SourceMod commands are working, for example:
   ```cmd
   sm_admin
   ```
   This should open the SourceMod admin menu if everything is set up correctly.

### Step 8: Further Customization
- **SourceMod Configs**: Place your SourceMod configuration files in `C:\l4d2_server\left4dead2\addons\sourcemod\configs`.
- **Plugins**: Add SourceMod plugins to `C:\l4d2_server\left4dead2\addons\sourcemod\plugins`.
