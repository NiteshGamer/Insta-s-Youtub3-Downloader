from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes
import yt_dlp
import instaloader
import os
import shutil

# Replace with your real token
TOKEN = "8000746724:AAHlQNZIjUXOI18WaYCTVH5NJ87IoLQ53H4"

# Initialize Instagram downloader
L = instaloader.Instaloader()

# === YouTube Downloader (Shorts supported) ===
async def download_youtube(url, update: Update):
    try:
        ydl_opts = {
            'format': 'bestvideo[ext=mp4]+bestaudio[ext=m4a]/best[ext=mp4]',
            'outtmpl': 'downloads/youtube_video.%(ext)s',
            'noplaylist': True,
            'quiet': True,
            'no_warnings': True,
            'merge_output_format': 'mp4',
        }

        os.makedirs("downloads", exist_ok=True)

        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            video_path = f"downloads/youtube_video.{info['ext']}"

            if os.path.getsize(video_path) > 49 * 1024 * 1024:
                await update.message.reply_text("⚠️ Video too large to send (50MB max). Try a shorter or lower-resolution video.")
                os.remove(video_path)
                return

            await update.message.reply_video(video=open(video_path, 'rb'), caption=info.get('title', 'Downloaded'))
            os.remove(video_path)

    except Exception as e:
        await update.message.reply_text(f"❌ Failed to download YouTube video.\nError: {str(e)}")

# === Instagram Reels / Posts Downloader ===
async def download_instagram(url, update: Update):
    try:
        # Extract shortcode from reel/post URL
        parts = [p for p in url.strip('/').split('/') if p]
        shortcode = parts[-1] if parts[-2] in ['reel', 'p', 'tv'] else parts[-2]

        post = instaloader.Post.from_shortcode(L.context, shortcode)
        folder = "insta_download"

        if os.path.exists(folder):
            shutil.rmtree(folder)
        os.makedirs(folder, exist_ok=True)

        L.download_post(post, target=folder)

        for file in os.listdir(folder):
            if file.endswith(".mp4"):
                video_path = os.path.join(folder, file)
                await update.message.reply_video(video=open(video_path, 'rb'))
                os.remove(video_path)

        shutil.rmtree(folder)

    except Exception as e:
        await update.message.reply_text(f"❌ Failed to download Instagram video.\nError: {str(e)}")

# === Handle all user messages ===
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    url = update.message.text.strip()
    if "youtube.com" in url or "youtu.be" in url:
        await download_youtube(url, update)
    elif "instagram.com" in url:
        await download_instagram(url, update)
    else:
        await update.message.reply_text("❗ Please send a valid YouTube or Instagram link.")

# === Start command ===
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 Welcome! Send a YouTube link (video/shorts) or Instagram (reel/post) to download.")

# === Bot Runner ===
if __name__ == "__main__":
    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), handle_message))

    print("🤖 Bot is running...")
    app.run_polling()
