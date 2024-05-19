import discord
import requests
import xml.etree.ElementTree as ET
import asyncio
import time

# Token-ul de acces pentru botul de Discord
TOKEN = 'MTIzMzgzMTU3NTkzNDg2NTQ0OA.GaWdsb.mJM6J8do_Vnl6uRYVPCKkLtEJrJuupptjL5MU8'

# ID-urile canalelor pe care botul va trimite mesajele pentru fiecare server
CHANNEL_IDS = {
    "server1": 1241445505347616949,
    "server2": 1241460718461649018
}

# Structura pentru a stoca informațiile despre jucători și timpul de conectare pe server
player_info = {
    "server1": {},
    "server2": {}
}

# Funcție pentru a citi XML-ul și a extrage informațiile despre jucători
def get_player_info(url):
    response = requests.get(url)
    if response.status_code == 200:
        root = ET.fromstring(response.content)
        players = root.findall(".//Player[@isUsed='true']")
        return {player.text: time.time() for player in players}
    else:
        return None

# Clientul Discord
intents = discord.Intents.default()
client = discord.Client(intents=intents)

# Funcție pentru a trimite un mesaj pe canalul specific
async def send_message(server, message):
    channel_id = CHANNEL_IDS.get(server)
    if channel_id:
        channel = client.get_channel(channel_id)
        await channel.send(message)

# Funcție care monitorizează schimbările în lista de jucători și trimite mesaje atunci când se conectează sau se deconectează un jucător
async def monitor_players():
    servers = {
        "server1": "http://194.163.137.166:8170/feed/dedicated-server-stats.xml?code=xhTRkLUh",
        "server2": "http://194.163.131.171:8140/feed/dedicated-server-stats.xml?code=ryK98PjwQ9kuRbvc"
    }

    current_players = {server: set() for server in servers}
    while True:
        try:
            for server, url in servers.items():
                player_info_new = get_player_info(url)
                if player_info_new is not None:
                    connected_players = player_info_new.keys()
                    new_players = connected_players - current_players[server]
                    disconnected_players = current_players[server] - connected_players
                    for player in new_players:
                        player_info[server][player] = time.time()
                        await send_message(server, f"Jucătorul {player} s-a alăturat serverului.")
                    for player in disconnected_players:
                        if player in player_info[server]:
                            time_connected = time.time() - player_info[server][player]
                            time_connected_hours = int(time_connected // 3600)
                            time_connected_minutes = int((time_connected % 3600) // 60)
                            await send_message(server, f"Jucătorul {player} a petrecut {time_connected_hours} ore și {time_connected_minutes} minute pe server.")
                            del player_info[server][player]
                current_players[server] = connected_players
        except Exception as e:
            print(f"A apărut o eroare: {e}")
        await asyncio.sleep(10)  # Așteaptă 10 secunde între verificări

# Funcție care rulează botul Discord
@client.event
async def on_ready():
    print('Botul a pornit!')
    client.loop.create_task(monitor_players())

# Pornirea botului
client.run(TOKEN)
