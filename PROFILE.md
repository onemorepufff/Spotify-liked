import os
import spotipy
from spotipy.oauth2 import SpotifyOAuth
from yt_dlp import YoutubeDL

# Spotify API Credentials
SPOTIFY_CLIENT_ID = "b4b3960cb5e04275b4c9159fa4a64507"
SPOTIFY_CLIENT_SECRET = "1df09db5735e47aea7abf747fb27af50"
SPOTIFY_REDIRECT_URI = "http://localhost:8888/callback"

# Paths
DOWNLOAD_DIR = "downloads"
DOWNLOADED_LOG = "downloaded_songs.txt"

# Ensure download directory exists
os.makedirs(DOWNLOAD_DIR, exist_ok=True)

# Authenticate Spotify API
sp = spotipy.Spotify(auth_manager=SpotifyOAuth(
    client_id=SPOTIFY_CLIENT_ID,
    client_secret=SPOTIFY_CLIENT_SECRET,
    redirect_uri=SPOTIFY_REDIRECT_URI,
    scope="user-library-read"
))

def fetch_liked_songs():
    """Fetch all Spotify liked songs."""
    liked_songs = []
    results = sp.current_user_saved_tracks(limit=50)
    
    while results:
        for item in results['items']:
            track = item['track']
            song_name = f"{track['artists'][0]['name']} - {track['name']}"
            liked_songs.append(song_name)

        # Fetch next page
        if results['next']:
            results = sp.next(results)
        else:
            break

    return liked_songs

def load_downloaded_songs():
    """Load the list of already downloaded songs."""
    if not os.path.exists(DOWNLOADED_LOG):
        return set()
    
    with open(DOWNLOADED_LOG, "r") as file:
        return set(line.strip() for line in file)

def save_downloaded_song(song):
    """Append a downloaded song to the log file."""
    with open(DOWNLOADED_LOG, "a") as file:
        file.write(song + "\n")

def download_songs(songs):
    """Download songs using yt-dlp in OPUS format."""
    ydl_opts = {
        "format": "bestaudio/best",
        "outtmpl": f"{DOWNLOAD_DIR}/%(title)s.%(ext)s",
        "postprocessors": [{
            "key": "FFmpegExtractAudio",
            "preferredcodec": "opus",
            "preferredquality": "192",
        }],
    }

    downloaded_songs = load_downloaded_songs()
    new_songs = [song for song in songs if song not in downloaded_songs]

    if not new_songs:
        print("✅ All songs are already downloaded!")
        return

    with YoutubeDL(ydl_opts) as ydl:
        for song in new_songs:
            print(f"🎵 Downloading: {song}")
            try:
                ydl.download([f"ytsearch5:{song}"])  # Updated to search top 5 results
                save_downloaded_song(song)  # Mark as downloaded
            except Exception as e:
                print(f"⚠️ Failed to download {song}: {e}")

if __name__ == "__main__":
    print("📡 Fetching liked songs from Spotify...")
    liked_songs = fetch_liked_songs()
    
    print(f"🎶 Found {len(liked_songs)} liked songs!")
    download_songs(liked_songs)
