# Your PC drive: Windows setup

Three pieces, all free except your domain:

```
Browser -> website (GitHub -> Cloudflare Worker, the "public" folder)
              |
              |  calls https://files.yourdomain.com
              v
        Cloudflare Tunnel  ->  drive server on your PC (pc-server)  ->  your folder
```

The website is only the interface. Your files and logins stay on your PC.

---

## 1. Install Node.js (once)

Download the **LTS** version from https://nodejs.org and install it with the defaults.
Check it worked: open **Command Prompt** and run `node --version`. It must say v18.15 or higher.

## 2. Set up the drive server on your PC

1. Copy the `pc-server` folder somewhere permanent, for example `C:\drive-server`.
2. Right-click `start-drive.bat` > Edit, and change the two lines marked EDIT:
   - `ROOT` is the folder where uploaded files are stored, for example `D:\DriveFiles`.
   - `ORIGIN` is your website's address, for example `https://drive.yourdomain.com`.
3. Open Command Prompt in that folder and create your account:
   ```
   cd C:\drive-server
   node server.js adduser yourname
   ```
   Pick a password of 12 or more characters. It is shown as you type, so make sure nobody is watching.
4. Double-click `start-drive.bat`. You should see `Drive server on http://127.0.0.1:3000`.
   Leave that window open (minimize it).

Add more people later with the same `adduser` command. Each person gets their own private folder inside `ROOT`.
Running `adduser` with an existing name resets that password. No restart needed.

## 3. Connect your domain with Cloudflare Tunnel

Menu names in the Cloudflare dashboard change now and then. If you can't find one, use its search box for "Tunnels".

1. Cloudflare dashboard > **Zero Trust** > **Networks** > **Tunnels** > **Create a tunnel** > **Cloudflared**.
2. Name it (for example `my-pc`) and choose **Windows**.
3. Cloudflare shows an install command that contains your token. Copy it and run it in **PowerShell as Administrator**.
   This installs `cloudflared` as a Windows service, so it starts automatically with your PC.
4. On the next screen add a **public hostname**:
   - Subdomain: `files`, Domain: your domain
   - Service type: `HTTP`, URL: `localhost:3000`
5. Test it: open `https://files.yourdomain.com/api/me` in a browser. You should see `{"error":"Sign in again."}`. That means the tunnel and server are talking.

## 4. Put the website online

1. Create a GitHub repo and upload everything in this project **except** `pc-server`
   (that folder has your server and must not go on GitHub, the `.gitignore` already keeps `data` and `storage` out).
   You need `public/`, `wrangler.jsonc` and `.gitignore`.
2. Edit `public/config.js`:
   ```js
   window.DRIVE_API = "https://files.yourdomain.com";
   window.DRIVE_NAME = "My Drive";
   ```
3. Cloudflare dashboard > **Workers & Pages** > **Create** > import your GitHub repo.
   Leave the deploy command as `npx wrangler deploy`. It publishes the `public` folder.
4. Give it a nice address: your Worker > **Settings** > **Domains & Routes** > add `drive.yourdomain.com`.
5. Make sure `ORIGIN` in `start-drive.bat` is exactly that address (https, no trailing slash). Close and reopen the server window after changing it.

Every time you push to GitHub, Cloudflare redeploys the site.

## 5. Make it start with your PC

The tunnel is already a service. For the drive server:

1. Press Start, type **Task Scheduler**, open it > **Create Task**.
2. General: name it `Drive server`, tick **Run whether user is logged on or not**.
3. Triggers > New > **At startup**.
4. Actions > New > Start a program > browse to `start-drive.bat`, and set **Start in** to its folder (for example `C:\drive-server`).
5. Settings > untick "Stop the task if it runs longer than".

Also set Windows not to sleep: **Settings > System > Power > Screen and sleep > Sleep: Never** (when plugged in).
When the PC is off or asleep, the site still loads but says it can't reach your PC.

---

## Good to know

- **Full quality.** Files are stored and sent byte for byte. Nothing is compressed or resized.
- **Big files.** Cloudflare's free plan caps each request at 100 MB, so the website uploads in 50 MB pieces and the server joins them. If the connection drops, it picks up where it left off. Downloads are sent in one stream and support resuming.
- **Cloudflare's rules.** Their terms discourage pushing lots of non-web content (large video libraries, for example) through the free proxy. Light personal use is generally fine. If you plan to move many gigabytes a day, test first or use Tailscale Funnel instead.
- **Security.** The server only touches the user's own folder, stores passwords hashed, locks an IP out after 10 wrong tries for 15 minutes, and signs every link. It listens only on your own PC (127.0.0.1), so the only way in is the tunnel. Don't put Cloudflare Access on the `files` hostname: it would block the website's requests. Use long passwords.
- **Not a backup.** Deleting in the website deletes from your PC for good. Keep a copy of anything important elsewhere.
- **Windows file names.** Names with `< > : " | ? *` are refused, because Windows can't store them.
- **Folders** can be uploaded and browsed, but downloads are file by file (no zip yet).

## If something goes wrong

| What you see | What to check |
|---|---|
| "Can't reach your PC" | The server window is open, and the `Cloudflared` service is running (Services app). |
| Page loads but sign-in does nothing, console shows a CORS error | `ORIGIN` in `start-drive.bat` doesn't exactly match the website address. Fix it and restart the server window. |
| "Set DRIVE_API in config.js" | You haven't replaced `files.yourdomain.com` in `public/config.js` yet. |
| Upload shows "Retrying…" a lot | Your upstream internet speed is the limit. It will keep trying and resume. |
