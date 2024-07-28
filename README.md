# Honeypot-Attack-Mapping-Project
Project using Azure Sentinel to create a vulnerable virtual machine and map out attacks targeting the VM.

<h2>Details</h2>

This project is based on [this video](https://www.youtube.com/watch?v=02RE3B2uIvw) by Anastasia Kuznetsova. It involves setting up a vulnerable honeypot VM in Azure Sentinel, then logging and mapping the IP addresses any attacks originate from.

<h2>Steps</h2>

<h3>Creating the Resources</h3>

The first step is creating our virtual machine to be used as a honeypot. The most relevant settings to change here are related to allowed inbound and outbound ports:

![image](https://github.com/user-attachments/assets/20496a7f-5695-4b45-9718-1c584880e9db)


Inbound HTTP, HTTPS, SSH, and RDP will be allowed on this VM to _increase_ our attack surface, since we want attackers to try to get to it.

![image](https://github.com/user-attachments/assets/64091ae5-2b25-4895-8045-d30fc1cf04f5)

Under the network security group, we create a rule allowing all inbound traffic on any port. This will create as many opportunities as possible for threat actors to access the machine and its resources. From here, we can deploy our virtual machine. 

Moving on to the next step of the project, creating the log analytics workspace. The log analytics workspace ingests logs from the Honeypot VM. There are no settings that necessarily need to be changed here, so we can simply name the workspace resource and move on to the Security Center.

I'll then add the Azure Security resource, which we can use to store raw data from Windows Security Events on our VM

![image](https://github.com/user-attachments/assets/236f7f62-7996-4784-8337-1d76464e2c50)


We've now created the resources needed and can move on to configuring them.

<h3>Adding the SIEM to Our Workspace</h3>

![image](https://github.com/user-attachments/assets/3668cf1b-ff61-4ff1-a2a0-e02639e1666a)

Microsoft Sentinel will be the SIEM we use to analyze and visualize the incoming traffic to the honeypot.

<h3>Accessing our VM</h3>

My operating system of choice is Ubuntu, so to RDP to the desktop, I'll have to go through the extra step of downloading an application. The application I use is [Remmina](https://remmina.org/), downloaded using the APT package manager:

![image](https://github.com/user-attachments/assets/009ddf22-2192-418d-92bf-2c6d1a867565)

Now that Remmina is downloaded, we can use it to connect to our VM over port 3389 using RDP:

![image](https://github.com/user-attachments/assets/a23db705-1632-48fd-9fcb-0928a91c6a33)

With the connection to the VM established, I can go ahead and completely turn off Windows Defender Firewall on it (It is **not** recommended to do this on any desktop you plan to actually use):

![image](https://github.com/user-attachments/assets/596bd6ab-1854-4fdf-885a-97889f94e018)

Now that the firewall is turned off, a new PowerShell script can be created to export security logs. I'll be using Anastasia Coskuner's [Custom_Security_Log_Exporter.ps1](https://github.com/AnastasiaCoskuner/Sentinel-Lab/blob/main/Custom_Security_Log_Exporter.ps1). We'll be replacing her API key for ipgeolocation.io with our own in the script. The PowerShell script filters for failed RDP events from Windows Event Viewer and uses our APi key to access the [IP Geolocation API](https://app.ipgeolocation.io). This API is used to look up the geographic location of each IP address from which the failed RDP events originate.
When the script is ran:

![image](https://github.com/user-attachments/assets/47b5c4a4-ce38-464f-97c6-acca10607c3b)

Success! There are a number of incoming failed RDP events being detected and logged within our log file. Now that the script is set up on the VM, we can move on to setting up the map that will show the locations of attack origins.

<h3>Setting Up the Map</h3>

Firstly, the table is set up by uploading a sample file to train Log Analytics on the format of the log. From there, the path to the log file within the virtual machine is entered:

![image](https://github.com/user-attachments/assets/79967fd0-2a4f-4091-9981-5e05c8818c99)

Now, we can query our logs using [this query](https://github.com/AnastasiaCoskuner/Sentinel-Lab/blob/main/query_log), which will extract and properly sort the relevant values, leaving out the value which pertain to the samples used to train Log Analytics earlier. We enter this query into a Microsoft Sentinel Workbook and set the visualization to 'Map', resulting in:

![image](https://github.com/user-attachments/assets/ca4351ae-c469-4a10-9c29-f3afff7e5c82)

Hm, while this is a heatmap of the attacks' origins, it seems our labels aren't quite right. To fix this, I'll head into the map settings and change the 'Metric Label' setting to the country the number of attacks came from:

![image](https://github.com/user-attachments/assets/8d20e2ba-865c-4f9f-9501-ca66f51499f4)

![image](https://github.com/user-attachments/assets/f49e8b6f-e3b9-4e9c-92ee-129c6f3ab635)

And there it is! Our map shows the majority of the attempts came from the Åland Islands. Honestly, I had expected the majority to come from Russia. From here, we could do a number of things, such as using the IP addresses from which each attack originated to create a blacklist, preventing such attacks. 

<h3>Analyzing the Data</h3>

We can also take a look at the inputs attackers guessed to get into the machine, specifically usernames:

![image](https://github.com/user-attachments/assets/cc909c0a-7e71-4b71-bf7f-79fec1386bd0)

It seems our friends (that are using a VPN that is) in Finland used a combination of "FRANTZ" and "FOY" last names along with a first initial. One attempt per first inital seems to suggest a password spraying attack, where a common/default password is used to attempt to log in with different usernames. After scrolling across some more first inital + last name username attempts, we get to Iran and Russia's attempts:

![image](https://github.com/user-attachments/assets/0f3f69a3-c9fb-476e-bad1-8685a82f24ec)

Both Iran and Russia go with the trusty old "administrator" and its alternate spellings. Iran simply keeps guessing different passwords with username "Administrator" while Russia seems to cycle through alternate spellings.

![image](https://github.com/user-attachments/assets/0808554c-8a76-4c03-923f-af86dfbe2e4d)

The adversary from Germany is nothing if not persistent, simply guessing different passwords for username "ADMIN" over and over. Specifically, all 71 attempts were using ADMIN as the username. Negative points for creativity.

![image](https://github.com/user-attachments/assets/0074cbcf-3fff-4915-bbbe-8db120ec76e4)

The attacks from Bulgaria take a different route, using the name of the machine and variations in capitalization as the username.

![image](https://github.com/user-attachments/assets/27f3155d-0a6d-4120-9bd7-7db9f0f18f69)

Finally, the attacks from the Netherlands use an even mix of "ADMIN" and "ADMINISTRATOR".

<h2>Takeaways</h2>
From this project, I can see how important port filtering is for security hardening. With port filtering, none of these attacks could have even been attempted; all it takes is blocking port 3389 and these wouldn't be possible. The project also speaks to the importance of changing default credentials. The only reason none of these attacks worked was because of the strong password and (somewhat) strong username I had put on the VM.
