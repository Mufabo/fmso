---
title: 'Build a tagged file explorer in Python'
date: 2026-10-22
tags: [python, tutorial]
description: 'A single-file, standard-library Python web app for browsing, previewing, tagging and searching a local media folder, with ffmpeg thumbnails and video seeking.'
draft: true
---

## What you'll build

You will build a local file explorer in one Python file (about 450 lines) with no third-party packages. It runs a small web server on your machine and gives you a browser UI for one media folder, with:

- A thumbnail grid with folder navigation and a clickable path bar
- A preview pane: images, playable video with seeking, and text files
- Tags you add to one file or many at once, stored in a JSON file
- Search across the whole folder by name, `#tag`, `-#tag` (exclude) and `#untagged`
- An "Open in Windows" button for formats the browser can't play

The design has three parts: a Python server that reads the disk, a single HTML page embedded in that server, and a hidden `.explorer` folder that holds the tags and the thumbnail cache. The browser never touches the disk directly; every action is an HTTP call to the server.

```
Browser (one HTML page)
    │  HTTP: /api/*, /thumb, /file
    ▼
explorer.py (Python server)
    ├── your media folder      listing, preview, streaming
    ├── .explorer/tags.json    tag storage
    ├── .explorer/thumbs/      thumbnail cache
    └── ffmpeg / ffprobe       thumbnail generation
```

The tutorial builds the server from the bottom of that diagram upward (paths, tags, search, thumbnails, routes), then the page at the top.

## Prerequisites and project layout

You need Python 3.8 or newer and ffmpeg (which includes ffprobe) on your PATH. On WSL or Ubuntu, `sudo apt install ffmpeg` covers it. Check both with:

```bash
python3 --version
ffmpeg -version | head -1
```

Everything lives in a hidden folder inside the folder you want to browse, so the app travels with your files:

```
bbs/
  .explorer/
    explorer.py     # server + embedded web page
    tags.json       # created on first tag
    thumbs/         # thumbnail cache, safe to delete
  ...your files...
```

The leading dot matters: the explorer hides dot-folders, so it never lists its own files. Create the folder and an empty `explorer.py`, then add the pieces below in order.

## Step 1: Configuration, safe paths and listing

Start with imports and settings. The root defaults to the folder that contains `.explorer`, and `--port` is removed from the argument list before the root is read:

```python
import hashlib, json, mimetypes, os, re, shutil, subprocess, sys, threading, time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import parse_qs, urlparse

HERE = os.path.dirname(os.path.abspath(__file__))
args = sys.argv[1:]
PORT = 8765
if "--port" in args:
    i = args.index("--port")
    PORT = int(args[i + 1])
    del args[i:i + 2]
ROOT = os.path.realpath(args[0] if args else os.path.dirname(HERE))
TAGS_FILE = os.path.join(HERE, "tags.json")
THUMBS = os.path.join(HERE, "thumbs")
SEARCH_SKIP = {".explorer", "Calibre Portable/Calibre", "Calibre Portable/Calibre Settings"}

IMAGE = {"jpg", "jpeg", "png", "gif", "webp", "bmp", "avif"}
VIDEO = {"mp4", "mkv", "avi", "mov", "wmv", "webm", "m4v", "flv", "ts", "mpg", "mpeg"}
TEXT = {"ass", "srt", "txt", "json", "opf", "md", "nfo", "log"}

os.makedirs(THUMBS, exist_ok=True)

def kind_of(name):
    ext = name.rsplit(".", 1)[-1].lower() if "." in name else ""
    if ext in IMAGE: return "image"
    if ext in VIDEO: return "video"
    if ext in TEXT: return "text"
    return "other"
```

`SEARCH_SKIP` lists program folders that search should ignore; you can still browse into them. Put your own large non-content folders there.

The most important function is `resolve`. Every path from the browser goes through it, so a request like `../../etc/passwd` can never escape the root:

```python
def resolve(rel):
    p = os.path.realpath(os.path.join(ROOT, rel or ""))
    if p != ROOT and not p.startswith(ROOT + os.sep):
        raise PermissionError(rel)
    return p
```

Listing a folder uses `os.scandir`, which returns file stats in one pass. Paths are always root-relative with forward slashes, and dotfiles are hidden:

```python
def entry(rel, st):
    name = rel.rsplit("/", 1)[-1]
    return {"path": rel, "name": name, "size": st.st_size,
            "mtime": st.st_mtime, "kind": kind_of(name)}

def list_dir(rel):
    base = resolve(rel)
    dirs, files = [], []
    with os.scandir(base) as it:
        for e in it:
            if e.name.startswith("."):
                continue
            r = f"{rel}/{e.name}" if rel else e.name
            if e.is_dir():
                dirs.append({"path": r, "name": e.name})
            else:
                files.append(entry(r, e.stat()))
    key = lambda x: x["name"].lower()
    return {"dir": rel, "dirs": sorted(dirs, key=key), "files": sorted(files, key=key)}
```

## Step 2: Tag storage

Tags are a plain JSON map from relative path to a sorted list of tags. It is human-readable, easy to back up, and fast enough for thousands of files:

```json
{"clips/beach.mp4": ["favorite", "summer"], "1.webp": ["wallpaper"]}
```

Writes go to a temp file first and are swapped in with `os.replace`, which is atomic. A crash mid-write can't leave a half-written `tags.json`. A lock stops two simultaneous requests from overwriting each other's changes:

```python
tags_lock = threading.Lock()

def load_tags():
    try:
        with open(TAGS_FILE, encoding="utf-8") as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return {}

def save_tags(tags):
    tmp = TAGS_FILE + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump(tags, f, ensure_ascii=False, indent=1, sort_keys=True)
    os.replace(tmp, TAGS_FILE)
```

One function handles every tag change: add and remove lists applied to a list of paths. That makes bulk tagging free. Tags are normalized (lowercase, no leading `#`, spaces become dashes) so "Summer" and "#summer" end up the same tag, and files with no tags left are dropped from the map:

```python
def update_tags(paths, add, remove):
    norm = lambda t: re.sub(r"\s+", "-", t.strip().lower().lstrip("#"))
    add = [norm(t) for t in add if norm(t)]
    remove = {norm(t) for t in remove}
    with tags_lock:
        tags = load_tags()
        for p in paths:
            resolve(p)  # reject paths outside the root
            cur = [t for t in tags.get(p, []) if t not in remove]
            cur += [t for t in add if t not in cur]
            if cur:
                tags[p] = sorted(cur)
            else:
                tags.pop(p, None)
        save_tags(tags)
        return tags
```

## Step 3: Search

Search covers the whole root, not just the open folder, and supports a tiny query language:

| Token | Meaning |
| --- | --- |
| `beach` | path contains "beach" |
| `#summer` | has the tag summer |
| `-#summer` | does not have the tag summer |
| `#untagged` | has no tags at all |

All tokens must match, so `beach #summer -#draft` narrows step by step.

Walking a whole drive on every keystroke is too slow. On a Windows drive mounted in WSL (`/mnt/d`), one walk of about 1,500 files took about 5 seconds. So the file list is cached for 5 minutes, and dot-folders and `SEARCH_SKIP` folders are pruned during the walk by editing `dirnames` in place:

```python
index_cache = {"t": 0, "files": []}

def all_files():
    if time.time() - index_cache["t"] < 300:
        return index_cache["files"]
    out = []
    for dirpath, dirnames, filenames in os.walk(ROOT):
        rel_dir = os.path.relpath(dirpath, ROOT).replace(os.sep, "/")
        rel_dir = "" if rel_dir == "." else rel_dir
        dirnames[:] = [d for d in dirnames if not d.startswith(".")
                       and (f"{rel_dir}/{d}" if rel_dir else d) not in SEARCH_SKIP]
        for fn in filenames:
            if fn.startswith("."):
                continue
            r = f"{rel_dir}/{fn}" if rel_dir else fn
            try:
                out.append(entry(r, os.stat(os.path.join(dirpath, fn))))
            except OSError:
                pass
    index_cache.update(t=time.time(), files=out)
    return out
```

The search itself splits the query into three lists and filters the index:

```python
def search(q):
    inc, exc, words = [], [], []
    for tok in q.lower().split():
        if tok.startswith("-#") and len(tok) > 2:
            exc.append(tok[2:])
        elif tok.startswith("#") and len(tok) > 1:
            inc.append(tok[1:])
        else:
            words.append(tok)
    tags = load_tags()
    res = []
    for f in all_files():
        ft = set(tags.get(f["path"], []))
        if "untagged" in inc and ft:
            continue
        if any(t not in ft for t in inc if t != "untagged"):
            continue
        if any(t in ft for t in exc):
            continue
        low = f["path"].lower()
        if any(w not in low for w in words):
            continue
        res.append(f)
    res.sort(key=lambda x: x["path"].lower())
    return {"dir": None, "dirs": [], "files": res}
```

Tags are read fresh on each search, so a tag you just added shows up right away even though the file list is cached.

## Step 4: Thumbnails with ffmpeg

Thumbnails are 360 px wide JPEGs made on first request and cached forever. ffmpeg handles both images and videos, so one function covers every format, including WebP and AVIF.

Three details make this work well:

1. **Cache key.** The file name is a SHA-1 of path, size and modified time. Replace a file and it gets a new thumbnail automatically; nothing ever needs invalidating.
2. **A frame worth seeing.** Frame 0 of a video is often black. ffprobe reads the duration and ffmpeg seeks to 20% in. Putting `-ss` before `-i` makes the seek fast even on multi-gigabyte files.
3. **A concurrency limit.** A grid of 400 lazy-loaded images can fire hundreds of requests at once. A semaphore allows 4 ffmpeg processes at a time, and the result is written to a temp file and renamed so a half-written thumbnail is never served.

```python
thumb_sem = threading.Semaphore(4)

def thumb(rel):
    src = resolve(rel)
    st = os.stat(src)
    key = hashlib.sha1(f"{rel}|{st.st_size}|{st.st_mtime}".encode()).hexdigest()
    dst = os.path.join(THUMBS, key + ".jpg")
    if os.path.exists(dst):
        return dst
    with thumb_sem:
        if os.path.exists(dst):  # another request made it while we waited
            return dst
        pre = []
        if kind_of(src) == "video":
            try:
                dur = float(subprocess.run(
                    ["ffprobe", "-v", "error", "-show_entries", "format=duration",
                     "-of", "csv=p=0", src],
                    capture_output=True, text=True, timeout=30).stdout.strip())
                pre = ["-ss", f"{dur * 0.2:.2f}"]
            except (ValueError, subprocess.TimeoutExpired):
                pass
        tmp = dst + ".tmp.jpg"
        subprocess.run(["ffmpeg", "-v", "error", *pre, "-i", src, "-frames:v", "1",
                        "-vf", "scale=360:-2", "-q:v", "5", "-y", tmp],
                       capture_output=True, timeout=120)
        if os.path.exists(tmp) and os.path.getsize(tmp):
            os.replace(tmp, dst)
            return dst
    return None
```

`scale=360:-2` keeps the aspect ratio and rounds the height to an even number, which many encoders require. In testing, a first video thumbnail took about 6 seconds; repeats are instant.

## Step 5: The HTTP server

The server is the standard library's `ThreadingHTTPServer`, one thread per request, so a slow thumbnail never blocks the page. These are its routes:

| Method | Route | Returns |
| --- | --- | --- |
| GET | `/` | the web page |
| GET | `/api/list?dir=` | folders and files in one folder |
| GET | `/api/search?q=` | matching files across the root |
| GET | `/api/tags` | the whole tag map |
| POST | `/api/tags` | applies `{paths, add, remove}`, returns the new map |
| GET | `/thumb?path=` | cached JPEG thumbnail |
| GET | `/file?path=` | the raw file, with Range support |
| POST | `/api/open` | opens the file in its Windows default app |
| POST | `/api/rescan` | clears the cached search index |

All request handling lives in `class Handler(BaseHTTPRequestHandler):`. The `send_path`, `do_GET` and `do_POST` functions below are its methods, so indent them one level inside that class.

### Streaming with Range requests

A `<video>` tag only lets you seek if the server answers HTTP Range requests.[^1] The browser asks for `bytes=START-END` and expects a `206 Partial Content` reply with a `Content-Range` header. A range that can't be served, like one starting past the end of the file, gets `416 Range Not Satisfiable` instead of a nonsensical negative length. Files are sent in 1 MB chunks so a 5 GB video never sits in memory:

```python
def send_path(self, path, ctype=None, cache=False):
    size = os.path.getsize(path)
    ctype = ctype or mimetypes.guess_type(path)[0] or "application/octet-stream"
    if path.lower().endswith(".mkv"):
        ctype = "video/webm"  # lets Chromium-based browsers try to play it
    start, end, status = 0, size - 1, 200
    m = re.match(r"bytes=(\d*)-(\d*)", self.headers.get("Range", ""))
    if m and size:
        s, e = m.groups()
        if s:
            start, end = int(s), min(int(e), size - 1) if e else size - 1
        elif e:  # "bytes=-500" means the last 500 bytes
            start = max(0, size - int(e))
        if start > end:  # e.g. a start past the end of the file
            self.send_response(416)  # Range Not Satisfiable
            self.send_header("Content-Range", f"bytes */{size}")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return
        status = 206
    self.send_response(status)
    self.send_header("Content-Type", ctype)
    self.send_header("Accept-Ranges", "bytes")
    self.send_header("Content-Length", str(end - start + 1))
    if status == 206:
        self.send_header("Content-Range", f"bytes {start}-{end}/{size}")
    if cache:
        self.send_header("Cache-Control", "max-age=86400")
    self.end_headers()
    with open(path, "rb") as f:
        f.seek(start)
        left = end - start + 1
        while left > 0:
            chunk = f.read(min(1 << 20, left))
            if not chunk:
                break
            self.wfile.write(chunk)
            left -= len(chunk)
```

The GET handler routes by path and turns exceptions into status codes. `BrokenPipeError` is normal: the browser drops the connection every time you seek or close a video, so it is silently ignored.

```python
def do_GET(self):
    if not self.from_this_app():
        return self.send_error(403)
    u = urlparse(self.path)
    q = {k: v[0] for k, v in parse_qs(u.query).items()}
    try:
        if u.path == "/":
            body = PAGE.encode()
            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=utf-8")
            self.send_header("Content-Length", str(len(body)))
            self.end_headers()
            self.wfile.write(body)
        elif u.path == "/api/list":
            self.send_json(list_dir(q.get("dir", "")))
        elif u.path == "/api/search":
            self.send_json(search(q.get("q", "")))
        elif u.path == "/api/tags":
            self.send_json(load_tags())
        elif u.path == "/file":
            self.send_path(resolve(q["path"]))
        elif u.path == "/thumb":
            t = thumb(q["path"])
            if t:
                self.send_path(t, "image/jpeg", cache=True)
            else:
                self.send_error(404)
        else:
            self.send_error(404)
    except (BrokenPipeError, ConnectionResetError):
        pass
    except PermissionError:
        self.send_error(403)
    except (FileNotFoundError, KeyError):
        self.send_error(404)
```

`do_POST` follows the same pattern. It reads the JSON body and dispatches `/api/tags` to `update_tags`, `/api/open` to the function below, and `/api/rescan` to a cache reset:

```python
def do_POST(self):
    if not self.from_this_app():
        return self.send_error(403)
    u = urlparse(self.path)
    try:
        n = int(self.headers.get("Content-Length", 0))
        data = json.loads(self.rfile.read(n) or b"{}")
        if u.path == "/api/tags":
            self.send_json(update_tags(data.get("paths", []),
                                       data.get("add", []), data.get("remove", [])))
        elif u.path == "/api/open":
            open_in_windows(data["path"])
            self.send_json({"ok": True})
        elif u.path == "/api/rescan":
            index_cache["t"] = 0  # the next search walks the disk again
            self.send_json({"ok": True})
        else:
            self.send_error(404)
    except (BrokenPipeError, ConnectionResetError):
        pass
    except PermissionError:
        self.send_error(403)
    except (KeyError, ValueError):  # missing field or bad JSON
        self.send_error(400)

def send_json(self, obj):
    body = json.dumps(obj, ensure_ascii=False).encode()
    self.send_response(200)
    self.send_header("Content-Type", "application/json; charset=utf-8")
    self.send_header("Content-Length", str(len(body)))
    self.end_headers()
    self.wfile.write(body)

def log_message(self, *args):
    pass  # keep the console quiet
```

### Only answer the explorer's own page

Binding to `127.0.0.1` keeps other machines out, but it doesn't keep out other *websites*. Any page open in your browser can send requests to `http://localhost:8765`. Without a check, a malicious site could rewrite your tags, or call `/api/open` on a `.bat` or `.exe` in your media folder and run it. A related attack, DNS rebinding, points an attacker's domain at `127.0.0.1` so their page can read your files.

The fix is to check two headers on every request. `Host` must name this server, which defeats DNS rebinding. `Origin`, which browsers add to cross-site POSTs and `fetch` calls, must be absent or be the explorer itself:

```python
def from_this_app(self):
    ok = {f"localhost:{PORT}", f"127.0.0.1:{PORT}"}
    origin = self.headers.get("Origin")
    return (self.headers.get("Host") in ok
            and (origin is None or origin.split("://", 1)[-1] in ok))
```

Both `do_GET` and `do_POST` call it first and answer `403` if it fails. Your own page and curl still work, because they send `Host: localhost:8765` and either no `Origin` or the explorer's own origin.

### Open in Windows

WSL can launch Windows programs. `wslpath -w` converts `/mnt/d/bbs/x.avi` to `D:\bbs\x.avi`, and `explorer.exe` given a file opens it in its default app. This is the fallback for formats the browser can't play:

```python
def open_in_windows(rel):
    p = resolve(rel)
    win = subprocess.run(["wslpath", "-w", p], capture_output=True, text=True).stdout.strip()
    exe = shutil.which("explorer.exe") or "/mnt/c/Windows/explorer.exe"
    subprocess.Popen([exe, win], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
```

On native Linux use `xdg-open`, and on macOS use `open`.

Finally, start the server bound to `127.0.0.1` only, so nobody else on your network can reach it. A background thread warms the search index at startup:

```python
if __name__ == "__main__":
    threading.Thread(target=all_files, daemon=True).start()
    print(f"Serving {ROOT} at http://localhost:{PORT}")
    ThreadingHTTPServer(("127.0.0.1", PORT), Handler).serve_forever()
```

## Step 6: The frontend

The whole UI is one HTML string, `PAGE = r"""..."""`, with plain JavaScript and no framework or build step. Put it **above** the `if __name__ == "__main__":` block: `serve_forever()` never returns, so anything defined after it never exists, and requests for `/` would fail with a `NameError`. The `r` prefix keeps backslashes in the JavaScript intact. The parts that matter are below; the complete page is in `.explorer/explorer.py`.

### Layout

One CSS grid lays out four areas: a header (path bar and search) across the top, then tag sidebar, file grid and preview pane. The preview column has width 0 until a file is selected:

```css
body{height:100vh;display:grid;grid-template-columns:190px 1fr auto;grid-template-rows:auto 1fr}
header{grid-column:1/-1}
#grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(150px,1fr));gap:10px}
#pv{width:0;overflow:auto}
#pv.open{width:min(46vw,720px);padding:12px}
```

### State and loading

One state object holds everything. The current folder and query are kept in the URL hash, so refresh and bookmarks work. A non-empty query calls search; otherwise the folder is listed:

```js
let S={dir:'',q:'',data:{dirs:[],files:[]},tags:{},sel:new Set(),cur:null,anchor:-1};
const api=(u,b)=>fetch(u,b?{method:'POST',headers:{'Content-Type':'application/json'},
  body:JSON.stringify(b)}:{}).then(r=>r.json());

async function load(){
  writeHash();
  S.data=S.q.trim()?await api('/api/search?q='+encodeURIComponent(S.q))
                   :await api('/api/list?dir='+encodeURIComponent(S.dir));
  S.sel.clear();S.anchor=-1;
  renderCrumbs();renderGrid();
}
```

### The grid

Each file becomes a card. Two rules keep it fast and safe:

- `loading="lazy"` on thumbnails, so only cards on screen request a `/thumb`.
- Every file name goes through an `esc()` helper before it reaches `innerHTML`. Real file names contain `&`, quotes and brackets; unescaped, they break the markup.

```js
const esc=s=>String(s).replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
// inside renderGrid, per file f:
const th=(f.kind==='image'||f.kind==='video')
  ?'<img loading="lazy" src="/thumb?path='+encodeURIComponent(f.path)+'">'
  :esc(ext);
```

### Selection

One click handler on the grid (event delegation) covers every card. Plain click selects one file, Ctrl toggles, and Shift selects a range from the last anchor. This is what makes bulk tagging possible:

```js
if(e.shiftKey&&S.anchor>=0){
  const [a,b]=[Math.min(S.anchor,i),Math.max(S.anchor,i)];
  for(let k=a;k<=b;k++)S.sel.add(S.data.files[k].path);
}else if(e.ctrlKey||e.metaKey){
  S.sel.has(f.path)?S.sel.delete(f.path):S.sel.add(f.path);S.anchor=i;
}else{S.sel=new Set([f.path]);S.anchor=i}
```

### Preview and tagging

The preview pane switches on `kind`. Images use `<img>`, videos use `<video controls autoplay muted>` pointed at `/file`, and text files fetch only the first 64 KB with a Range header. With several files selected, the pane shows only the tags they all share. Adding or removing a tag sends every selected path in one request:

```js
async function changeTags(add,remove){
  const paths=[...S.sel];if(!paths.length)return;
  S.tags=await api('/api/tags',{paths,add,remove});
  renderTags();renderGrid();preview();
}
$('#addtag').onkeydown=e=>{
  if(e.key==='Enter'&&e.target.value.trim()){changeTags(e.target.value.split(','),[]);e.target.value=''}
};
```

A `<datalist>` filled with existing tags gives autocomplete for free, which keeps you from creating near-duplicate tags.

### Tag sidebar and search box

The sidebar counts tags from the tag map. Clicking a tag toggles its `#tag` token in the search box, so sidebar and typed queries are the same mechanism. Typing is debounced by 250 ms so search doesn't run on every keystroke:

```js
let qt;$('#q').oninput=e=>{clearTimeout(qt);qt=setTimeout(()=>{S.q=e.target.value;renderTags();load()},250)};
```

Finally, ← and → step through files and Esc closes the preview, but only when focus isn't in a text box.

## Step 7: Run, test and avoid the pitfalls

Start the server and open http://localhost:8765 in your browser (from WSL, a Windows browser reaches it through localhost forwarding):

```bash
python3 /mnt/d/bbs/.explorer/explorer.py            # default port 8765
python3 /mnt/d/bbs/.explorer/explorer.py --port 9000
python3 /mnt/d/bbs/.explorer/explorer.py /some/other/folder
```

Before trusting the UI, test each route with curl. These checks match what was run on the finished explorer:

```bash
B=http://localhost:8765
curl -s "$B/api/list" | head -c 300                       # JSON listing
curl -s "$B/api/search?q=%23untagged" | head -c 300       # %23 is '#'
curl -s -XPOST $B/api/tags -d '{"paths":["1.webp"],"add":["Test Tag"]}'
curl -s -o /dev/null -w "%{http_code}\n" -H "Range: bytes=100-199" "$B/file?path=1.webp"   # 206
curl -s -o /dev/null -w "%{http_code}\n" -H "Range: bytes=999999999-" "$B/file?path=1.webp"  # 416
curl -s -o /dev/null -w "%{http_code}\n" -XPOST -H "Origin: https://evil.example" $B/api/rescan  # 403
curl -s -o /dev/null -w "%{http_code}\n" "$B/file?path=../../etc/passwd"                  # 403
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" "$B/thumb?path=1.webp"           # 200 image/jpeg
```

The tag call should come back with `"test-tag"`, normalized. Remove it afterwards with `"remove":["test-tag"]`.

### Pitfalls

- **Argument parsing.** The first version took every argument that didn't start with `--` as the root, so `--port 8799` made `8799` the root folder and every request returned 404. Remove the flag and its value before reading positional arguments, as in Step 1.
- **`pkill -f` kills itself.** `pkill -f "explorer.py --port 8799"` run inside a shell command also matches that shell. Use the bracket trick `pkill -f "[e]xplorer.py"`.
- **Search feels stale.** The search index is cached for 5 minutes, so a newly added file appears in folder view at once but in search only later. Restart the server, or wire the `/api/rescan` POST route (it clears the cache) to a button.
- **Some videos won't play.** Browsers play H.264 MP4 reliably. AVI never plays, MKV and HEVC/x265 often don't. Use Open in Windows for those, or transcode with ffmpeg.
- **Moving files drops their tags.** Tags are keyed by path. Rename files from inside the explorer if you add that feature, or edit `tags.json` by hand.
- **The `/mnt/d` mount is slow.** WSL reads Windows drives over a network-style protocol. Keep the walk pruned with `SEARCH_SKIP`, and expect the first thumbnail of each video to take a few seconds.

## Ideas for extending it

- **Rename and move with tags.** Add a `/api/rename` route that calls `os.rename` and moves the file's entry in `tags.json` under the same lock.
- **Sorting.** Sort by size or date as well as name; the listing already returns both.
- **Video duration on cards.** Store ffprobe's duration next to the thumbnail and show it in the badge.
- **OR queries.** Allow `#a|#b` for files with either tag.
- **Rescan button.** One button that POSTs to `/api/rescan`.
- **Start with Windows.** A `.bat` file in the Startup folder running `wsl python3 /mnt/d/bbs/.explorer/explorer.py`.

[^1]: [HTTP range requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Range_requests), MDN Web Docs.
