from flask import Flask, request, jsonify, send_file
from flask_cors import CORS
import yt_dlp
import os
import tempfile
import threading
import uuid
import time

app = Flask(__name__)
CORS(app)  # Allows requests from your frontend

# In-memory job store (use Redis in production)
jobs = {}
DOWNLOAD_DIR = tempfile.gettempdir()

# ─────────────────────────────────────────
# 1. Fetch video info (title, duration, thumbnail)
# ─────────────────────────────────────────
@app.route("/info", methods=["POST"])
def get_info():
    data = request.json
    url = data.get("url", "")
    if not url:
        return jsonify({"error": "No URL provided"}), 400

    ydl_opts = {"quiet": True, "no_warnings": True, "skip_download": True}
    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=False)
            return jsonify({
                "title":     info.get("title"),
                "channel":   info.get("uploader"),
                "duration":  info.get("duration_string") or str(info.get("duration", 0)) + "s",
                "views":     f'{info.get("view_count", 0):,}',
                "thumbnail": info.get("thumbnail"),
            })
    except Exception as e:
        return jsonify({"error": str(e)}), 500


# ─────────────────────────────────────────
# 2. Start a download job (async)
# ─────────────────────────────────────────
@app.route("/download", methods=["POST"])
def start_download():
    data = request.json
    url    = data.get("url", "")
    title  = data.get("title", "video")
    fmt    = data.get("format", "mp4")   # mp4 | 720 | mp3

    if not url:
        return jsonify({"error": "No URL provided"}), 400

    job_id = str(uuid.uuid4())
    jobs[job_id] = {"status": "pending", "progress": 0, "file": None, "error": None}

    thread = threading.Thread(target=_do_download, args=(job_id, url, title, fmt))
    thread.daemon = True
    thread.start()

    return jsonify({"job_id": job_id})


def _do_download(job_id, url, title, fmt):
    safe_title = "".join(c for c in title if c.isalnum() or c in " _-").strip()
    out_path = os.path.join(DOWNLOAD_DIR, f"{job_id}_{safe_title}")

    def progress_hook(d):
        if d["status"] == "downloading":
            pct = d.get("_percent_str", "0%").strip().replace("%", "")
            try:
                jobs[job_id]["progress"] = float(pct)
            except:
                pass
        elif d["status"] == "finished":
            jobs[job_id]["progress"] = 99

    if fmt == "mp3":
        ydl_opts = {
            "format": "bestaudio/best",
            "outtmpl": out_path + ".%(ext)s",
            "postprocessors": [{"key": "FFmpegExtractAudio", "preferredcodec": "mp3", "preferredquality": "192"}],
            "progress_hooks": [progress_hook],
            "quiet": True,
        }
    elif fmt == "720":
        ydl_opts = {
            "format": "bestvideo[height<=720]+bestaudio/best[height<=720]",
            "outtmpl": out_path + ".%(ext)s",
            "merge_output_format": "mp4",
            "progress_hooks": [progress_hook],
            "quiet": True,
        }
    else:  # mp4 1080p
        ydl_opts = {
            "format": "bestvideo[height<=1080]+bestaudio/best",
            "outtmpl": out_path + ".%(ext)s",
            "merge_output_format": "mp4",
            "progress_hooks": [progress_hook],
            "quiet": True,
        }

    try:
        jobs[job_id]["status"] = "downloading"
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.download([url])

        # Find the actual output file
        ext = "mp3" if fmt == "mp3" else "mp4"
        file_path = out_path + f".{ext}"
        jobs[job_id].update({"status": "done", "progress": 100, "file": file_path, "ext": ext})
    except Exception as e:
        jobs[job_id].update({"status": "error", "error": str(e)})


# ─────────────────────────────────────────
# 3. Poll job status
# ─────────────────────────────────────────
@app.route("/status/<job_id>")
def job_status(job_id):
    job = jobs.get(job_id)
    if not job:
        return jsonify({"error": "Job not found"}), 404
    return jsonify({
        "status":   job["status"],
        "progress": job["progress"],
        "error":    job["error"],
    })


# ─────────────────────────────────────────
# 4. Download file to browser
# ─────────────────────────────────────────
@app.route("/file/<job_id>")
def get_file(job_id):
    job = jobs.get(job_id)
    if not job or job["status"] != "done":
        return jsonify({"error": "File not ready"}), 404

    file_path = job["file"]
    ext = job.get("ext", "mp4")
    mime = "audio/mpeg" if ext == "mp3" else "video/mp4"

    def cleanup():
        time.sleep(60)  # Delete file 60s after serving
        try: os.remove(file_path)
        except: pass
        jobs.pop(job_id, None)

    threading.Thread(target=cleanup, daemon=True).start()
    return send_file(file_path, mimetype=mime, as_attachment=True,
                     download_name=os.path.basename(file_path).split("_", 1)[1])


if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)
