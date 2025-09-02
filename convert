import io
import os
import shutil
import tempfile
import uuid
from urllib.parse import urlparse

from flask import Flask, request, send_file, render_template_string, abort, after_this_request
import requests
import ffmpeg

app = Flask(__name__)

# Hard block list for downloads (metadata lookup is still allowed)
BLOCKED_HOSTS = {
    "open.spotify.com", "spotify.link",
    "music.apple.com", "itunes.apple.com",
    "youtube.com", "www.youtube.com", "m.youtube.com", "youtu.be",
    "soundcloud.com", "m.soundcloud.com", "api.soundcloud.com"
}

ALLOWED_OUTPUTS = {"wav", "aiff"}
ALLOWED_CONTENT_PREFIXES = ("audio/",)  # keep strict

HTML = """
<!doctype html>
<title>Music Link Converter (Legal)</title>
<h1>Music Link Converter (Legal)</h1>
<p><strong>Heads up:</strong> This app will not download from Spotify, Apple Music, YouTube, or SoundCloud. Use uploads or direct audio file URLs you have rights to.</p>

<form method="post" action="/convert" enctype="multipart/form-data" style="margin-bottom:2rem;">
  <fieldset>
    <legend>1) Provide source</legend>
    <label>Upload audio file:
      <input type="file" name="file">
    </label>
    <br><br>
    <label>OR direct audio file URL:
      <input type="url" name="file_url" placeholder="https://example.com/song.flac" style="width:32rem;">
    </label>
  </fieldset>
  <br>
  <fieldset>
    <legend>2) Choose output</legend>
    <label><input type="radio" name="format" value="wav" checked> WAV</label>
    <label><input type="radio" name="format" value="aiff"> AIFF</label>
  </fieldset>
  <br>
  <label>
    <input type="checkbox" name="rights" required>
    I confirm I own the content or have permission to convert and download it.
  </label>
  <br><br>
  <button type="submit">Convert</button>
</form>

<hr>

<h2>Metadata (display only)</h2>
<form method="get" action="/meta">
  <input type="url" name="link" placeholder="Spotify/Apple/YouTube/SoundCloud link" style="width:32rem;" required>
  <button type="submit">Fetch</button>
</form>
"""

def _safe_tempdir():
    d = tempfile.mkdtemp(prefix="musicconv_")
    return d

def _download_direct_audio(url: str, dest_dir: str) -> str:
    """Download a direct audio file URL (not from blocked hosts)."""
    parsed = urlparse(url)
    host = (parsed.netloc or "").lower()
    if host in BLOCKED_HOSTS:
        abort(400, "Downloading from that domain is not allowed. Use a direct file URL or upload the file.")

    # HEAD to check content-type
    try:
        head = requests.head(url, timeout=10, allow_redirects=True)
    except requests.RequestException:
        abort(400, "Could not reach the URL.")
    ctype = head.headers.get("Content-Type", "")
    if not any(ctype.startswith(p) for p in ALLOWED_CONTENT_PREFIXES):
        abort(400, f"URL does not look like a direct audio file (Content-Type: {ctype or 'unknown'}).")

    # Stream download with a reasonable cap (e.g., 200 MB)
    max_bytes = 200 * 1024 * 1024
    try:
        r = requests.get(url, stream=True, timeout=30)
        r.raise_for_status()
    except requests.RequestException:
        abort(400, "Failed to download the file.")

    suffix = os.path.splitext(parsed.path)[1] or ".bin"
    in_path = os.path.join(dest_dir, f"in_{uuid.uuid4().hex}{suffix}")
    size = 0
    with open(in_path, "wb") as f:
        for chunk in r.iter_content(chunk_size=1024 * 1024):
            if not chunk:
                continue
            size += len(chunk)
            if size > max_bytes:
                f.close()
                os.remove(in_path)
                abort(400, "File is larger than the 200 MB limit.")
            f.write(chunk)
    return in_path

def _save_upload(file_storage, dest_dir: str) -> str:
    if not file_storage or file_storage.filename == "":
        abort(400, "No file uploaded.")
    suffix = os.path.splitext(file_storage.filename)[1] or ".bin"
    in_path = os.path.join(dest_dir, f"in_{uuid.uuid4().hex}{suffix}")
    file_storage.save(in_path)
    return in_path

def _convert(in_path: str, out_format: str) -> str:
    base = os.path.splitext(os.path.basename(in_path))[0]
    out_path = os.path.join(os.path.dirname(in_path), f"{base}.{out_format}")

    # 44.1 kHz, 16-bit PCM stereo. Use endian that matches container.
    acodec = "pcm_s16le" if out_format == "wav" else "pcm_s16be"
    try:
        (
            ffmpeg
            .input(in_path)
            .output(
                out_path,
                acodec=acodec,
                ar=44100,
                ac=2,
                loglevel="error"
            )
            .overwrite_output()
            .run()
        )
    except ffmpeg.Error as e:
        abort(400, f"Conversion failed: {e.stderr.decode('utf-8', errors='ignore') if e.stderr else 'unknown error'}")
    return out_path

@app.route("/", methods=["GET"])
def index():
    return render_template_string(HTML)

@app.route("/convert", methods=["POST"])
def convert_route():
    target_fmt = request.form.get("format", "wav").lower()
    if target_fmt not in ALLOWED_OUTPUTS:
        abort(400, "Unsupported output format.")
    if request.form.get("rights") is None:
        abort(400, "You must confirm you have rights to this content.")

    tmpdir = _safe_tempdir()

    @after_this_request
    def cleanup(response):
        try:
            shutil.rmtree(tmpdir, ignore_errors=True)
        except Exception:
            pass
        return response

    file_url = request.form.get("file_url", "").strip()
    in_path = None

    # Prefer upload if present, otherwise try URL
    if "file" in request.files and request.files["file"].filename:
        in_path = _save_upload(request.files["file"], tmpdir)
    elif file_url:
        in_path = _download_direct_audio(file_url, tmpdir)
    else:
        abort(400, "Provide a file upload or a direct audio file URL.")

    out_path = _convert(in_path, target_fmt)

    # Send file and set a friendly filename
    download_name = os.path.basename(out_path)
    mime = "audio/x-wav" if target_fmt == "wav" else "audio/aiff"
    return send_file(out_path, as_attachment=True, download_name=download_name, mimetype=mime)

@app.route("/meta", methods=["GET"])
def meta_route():
    """
    Very lightweight metadata via oEmbed/lookup where available (display only).
    No audio is downloaded.
    """
    link = request.args.get("link", "").strip()
    if not link:
        abort(400, "Provide a link.")
    parsed = urlparse(link)
    host = (parsed.netloc or "").lower()

    # Minimal oEmbed-ish metadata fetchers
    data = {"source": host, "url": link, "title": None, "author": None, "thumbnail": None}

    try:
        if "open.spotify.com" in host:
            # Spotify oEmbed
            r = requests.get("https://open.spotify.com/oembed", params={"url": link}, timeout=10)
            if r.ok:
                j = r.json()
                data["title"] = j.get("title")
                data["author"] = j.get("author_name")
                data["thumbnail"] = j.get("thumbnail_url")

        elif "music.apple.com" in host or "itunes.apple.com" in host:
            # Use iTunes Search API heuristically with last path part as hint
            # This is best-effort metadata only.
            term = os.path.splitext(os.path.basename(parsed.path))[0]
            q = {"term": term, "limit": 1}
            r = requests.get("https://itunes.apple.com/search", params=q, timeout=10)
            if r.ok and r.json().get("results"):
                it = r.json()["results"][0]
                data["title"] = it.get("trackName") or it.get("collectionName")
                data["author"] = it.get("artistName")
                data["thumbnail"] = it.get("artworkUrl100")

        elif "youtube.com" in host or "youtu.be" in host:
            # Simple oEmbed
            r = requests.get("https://www.youtube.com/oembed", params={"url": link, "format": "json"}, timeout=10)
            if r.ok:
                j = r.json()
                data["title"] = j.get("title")
                data["author"] = j.get("author_name")
                # No thumb in YouTube oEmbed JSON, could derive from video ID if needed

        elif "soundcloud.com" in host:
            # SoundCloud oEmbed
            r = requests.get("https://soundcloud.com/oembed", params={"url": link, "format": "json"}, timeout=10)
            if r.ok:
                j = r.json()
                data["title"] = j.get("title")
                data["author"] = j.get("author_name")
                data["thumbnail"] = j.get("thumbnail_url")
    except requests.RequestException:
        pass

    # Render a tiny result
    html = f"""
    <h3>Metadata</h3>
    <p><strong>Source:</strong> {data['source']}</p>
    <p><strong>URL:</strong> <a href="{data['url']}" target="_blank" rel="noopener">{data['url']}</a></p>
    <p><strong>Title:</strong> {data['title'] or '—'}</p>
    <p><strong>Author:</strong> {data['author'] or '—'}</p>
    <p><strong>Thumbnail:</strong> {(f'<img src="{data["thumbnail"]}" alt="thumbnail">' if data['thumbnail'] else '—')}</p>
    <p>Reminder: audio ripping from these services is blocked by design.</p>
    <p><a href="/">Back</a></p>
    """
    return html

if __name__ == "__main__":
    app.run(debug=True)
