# Test.git.com
import requests
from bs4 import BeautifulSoup
import subprocess
import os
from PIL import Image, ImageDraw, ImageFont
from telethon import TelegramClient, events
import asyncio
import glob

# --- Identifiants Telegram ---
API_ID = 25232938
API_HASH = '1de8d3fc18dd71b9017daa55bb2878da'
BOT_TOKEN = '8107779079:AAESsSkycMFYVvqIvjMpyb7y9g50GNUNxyg'
CHANNEL_ID = -1002555274702

# --- Dossier des téléchargements ---
DOWNLOAD_DIR = "downloads"

# --- Message de bienvenue ---
welcome_message = (
    "✨ 𝗕𝗶𝗲𝗻𝘃𝗲𝗻𝘂𝗲 𝗱𝗮𝗻𝘀 𝗹𝗲 𝗻𝗼𝘂𝘃𝗲𝗮𝘂 𝙣𝙚𝙨𝙩 𝗱𝗲𝘀 𝗼𝘁𝗮𝗸𝘂𝘀 ! ✨\n\n"
    "🔸 *NYAA PROJECT BOT* est maintenant en ligne !\n"
    "📡 Je surveille 24h/24 les nouveaux animes publiés sur [Nyaa.si](https://nyaa.si) en VOSTFR et MULTI (FR).\n"
    "🎬 Dès qu’un épisode est détecté :\n"
    " ✅ Téléchargement automatique\n"
    " ✅ Extraction des sous-titres (.srt/.ass)\n"
    " ✅ Envoi dans ce canal avec miniature\n\n"
    "🧠 Propulsé par *DJODJO JSK*, le maître des systèmes autonomes 🤖\n\n"
    "🕒 Recherche active toutes les **3 minutes**. Reste connecté ! 🔥"
)

# --- Fonction pour scraper Nyaa ---
def scrape_nyaa():
    url = "https://nyaa.si/?f=0&c=1_2&sort=added&order=desc"
    r = requests.get(url)
    soup = BeautifulSoup(r.text, 'html.parser')
    rows = soup.find_all('tr', class_='default')
    torrents = []

    for row in rows:
        title_td = row.find('td', class_='col-2')
        magnet_a = row.find('a', title='Magnet link')
        if title_td is None or magnet_a is None:
            continue

        title = title_td.text.strip()
        if any(keyword in title.lower() for keyword in ['vostfr', 'multi', 'french']):
            magnet_link = magnet_a.get('href')
            if magnet_link:
                torrents.append({'title': title, 'magnet': magnet_link})
    return torrents

# --- Télécharger via aria2c ---
def download_torrent(magnet_link, dest_dir):
    os.makedirs(dest_dir, exist_ok=True)
    print(f"Téléchargement : {magnet_link}")
    result = subprocess.run(['aria2c', '--dir=' + dest_dir, '--seed-time=0', '--max-download-limit=0', magnet_link], capture_output=True)
    return result.returncode == 0

# --- Chercher fichiers vidéos et sous-titres ---
def find_video_and_subtitles(folder):
    video_files = []
    subtitle_files = []
    for ext in ['*.mkv', '*.mp4', '*.avi']:
        video_files.extend(glob.glob(os.path.join(folder, ext)))
    for ext in ['*.ass', '*.srt']:
        subtitle_files.extend(glob.glob(os.path.join(folder, ext)))
    return video_files, subtitle_files

# --- Générer miniature avec PIL ---
def generate_thumbnail(text, filename):
    img = Image.new('RGB', (320, 180), color=(30, 30, 30))
    draw = ImageDraw.Draw(img)
    try:
        font = ImageFont.truetype("arial.ttf", 20)
    except:
        font = ImageFont.load_default()
    draw.text((10, 70), text, fill=(255, 255, 255), font=font)
    img.save(filename)
    print(f"Miniature générée : {filename}")

# --- Envoi fichiers sur Telegram (async) ---
async def send_files_telegram(client, video_path, subtitles_paths, thumbnail_path, caption):
    print(f"Envoi sur Telegram : {video_path}")
    thumb = await client.upload_file(thumbnail_path)
    await client.send_file(CHANNEL_ID, video_path, caption=caption, thumb=thumb, supports_streaming=True)
    for sub in subtitles_paths:
        await client.send_file(CHANNEL_ID, sub, caption=f"Sous-titres : {os.path.basename(sub)}")

# --- Fonction principale async ---
async def main_loop(client):
    processed_titles = set()
    while True:
        torrents = scrape_nyaa()
        for torrent in torrents:
            safe_title = "".join(c for c in torrent['title'] if c.isalnum() or c in (' ', '.', '_')).rstrip()
            if safe_title in processed_titles:
                print(f"Déjà traité : {safe_title}")
                continue

            target_folder = os.path.join(DOWNLOAD_DIR, safe_title)
            if os.path.exists(target_folder) and os.listdir(target_folder):
                print(f"Déjà téléchargé : {safe_title}")
                processed_titles.add(safe_title)
                continue

            if not download_torrent(torrent['magnet'], target_folder):
                print("Échec téléchargement.")
                continue

            videos, subs = find_video_and_subtitles(target_folder)
            if not videos:
                print("Aucune vidéo trouvée.")
                continue

            video_path = videos[0]
            thumbnail_path = os.path.join(target_folder, "thumbnail.jpg")
            size_mb = os.path.getsize(video_path) / (1024*1024)
            caption = f"{torrent['title']}\nTaille : {size_mb:.1f} MB\nLangue : VOSTFR / FRENCH"

            generate_thumbnail(caption, thumbnail_path)
            await send_files_telegram(client, video_path, subs, thumbnail_path, caption)
            processed_titles.add(safe_title)

        print("Pause 3 minutes avant la prochaine recherche...")
        await asyncio.sleep(180)  # Pause 3 minutes

# --- Gestion commande /start ---
@events.register(events.NewMessage(pattern='/start'))
async def start_handler(event):
    await event.respond(welcome_message, parse_mode='md')

# --- Démarrage du bot ---
async def main():
    client = TelegramClient('bot_session', API_ID, API_HASH)
    await client.start(bot_token=BOT_TOKEN)

    # Envoyer message dans le canal au lancement
    await client.send_message(CHANNEL_ID, "🚀 NYAA PROJECT lancé avec succès ! Le bot surveille les nouveaux animes VOSTFR / MULTI sur Nyaa.si… 🔎")

    # Ajouter gestionnaire d'événements
    client.add_event_handler(start_handler)

    # Lancer boucle scraping
    await main_loop(client)

    # Ne quitte pas le bot
    await client.run_until_disconnected()

if __name__ == "__main__":
    asyncio.run(main())
