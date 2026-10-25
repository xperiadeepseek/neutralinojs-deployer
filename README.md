# NeutralinoJS Deployer

Turn any website into a small Windows `.exe` straight from GitHub. Type an app name and a URL into a GitHub Actions popup, wait about a minute, and download a standalone app that opens that site in its own window.

No local setup. Nothing to install on your computer. No config files to edit.

Built on [Neutralinojs](https://neutralino.js.org/), which uses the WebView2 component already present on current Windows 10 and 11. The result is a few MB, not the 100+ MB you get from Electron-style wrappers.

---

## Quick start

### 1. Add the workflow to your repository

1. Create a new GitHub repository (or use an existing one).
2. Add the workflow file at exactly this path:

   ```
   .github/workflows/dynamic-packager.yml
   ```

3. Commit it to your default branch.

### 2. Run it

1. Open the **Actions** tab of your repository.
2. Select **Dynamic Website Packager** in the left sidebar.
3. Click **Run workflow** and fill in the popup:

   | Input | Description | Example |
   |---|---|---|
   | **Application Name** | Name of the window and of the final `.exe` | `Instagram` |
   | **Website URL** | Site to load. `https://` is added if you leave it out | `instagram.com` |
   | **Start in fullscreen mode** | Opens the app fullscreen, like pressing F11 (ticked by default) | on / off |
   | **Keep the app on its own site** | Links to other sites (for example a YouTube link inside the app) open in your default browser instead of inside the app (ticked by default) | on / off |
   | **Extra domains that must stay inside the app** | Comma-separated domains that should not be sent to the browser, such as login or SSO domains. Optional | `auth.openai.com, accounts.example.com` |

4. Click the green **Run workflow** button.

### 3. Download your app

1. When the run finishes (green check), open it.
2. Scroll to **Artifacts** at the bottom of the run page.
3. Download `<AppName>-Windows`. It is a zip containing `<AppName>.exe`.
4. Extract the zip and run the exe.

---

## How it works

The workflow runs on a GitHub-hosted Ubuntu runner and does the following, in order:

1. **Installs the Neutralino CLI** (`neu`) with Node.js 20.
2. **Scaffolds a project** with `neu create`. This produces the default template and a `neutralino.config.json`.
3. **Rewrites the config with `jq`.** It points the window at your URL instead of the bundled template and sets:
   - `url` and `modes.window.url` to your site
   - `modes.window.title` to your app name
   - `modes.window.fullScreen` to your fullscreen choice
   - `modes.window.enableInspector = false` so DevTools does not open
   - `modes.window.newWindowPolicy = "browser"` and `modes.window.injectScript` (if "Keep the app on its own site" is ticked), see [Keeping the app on its own site](#keeping-the-app-on-its-own-site)
   - window size to 1200x800 with a 400x300 minimum
   - `enableServer = false`, because nothing is served locally
4. **Updates the core binaries** with `neu update`.
5. **Builds** with `neu build --embed-resources`. This is the important flag. Without it, Neutralino writes your config into a separate `resources.neu` file next to the exe. If you only distribute the exe, it falls back to the default template instead of your site. Embedding puts the config inside the exe so it works on its own.
6. **Renames** the Windows binary to `<AppName>.exe` and uploads it as a workflow artifact.

User inputs are passed to the shell through environment variables, not pasted into the script, so unusual characters in a name or URL cannot break the build.

### Keeping the app on its own site

When this option is ticked, the workflow writes a small script into the app (`resources/js/external-links.js`) and tells Neutralino to run it on every page, before the site's own scripts. The script does not use Neutralino's native API, so the website gets no extra access to your computer.

It listens for clicks on links and decides, for each one:

| Link | What happens |
|---|---|
| Points to the app's own site or a subdomain of it | Stays in the app. A link that asks for a new tab or window (`target="_blank"`, Ctrl-click, middle-click) is opened in the same window instead |
| Points to any other `http(s)` site | Cancelled inside the app and opened in your default browser |
| `mailto:`, `tel:`, downloads, `#anchors`, `javascript:` | Left alone |

Scripted `window.open()` calls follow the same rule: same-site URLs navigate in place, other URLs go to the browser.

**What counts as "the app's own site":** the registrable domain of the URL you entered. For `web.whatsapp.com` that is `whatsapp.com`, so `faq.whatsapp.com` stays in the app but `youtube.com` does not. A built-in list of common two-part endings (`co.uk`, `com.au`, `co.in` and others) is handled, so `news.bbc.co.uk` counts as `bbc.co.uk`.

**Sites that sign in on a different domain** (for example ChatGPT signing in through `auth.openai.com`) need that domain listed in **Extra domains that must stay inside the app**. Otherwise the sign-in page opens in your browser and the login cannot finish inside the app.

The script only runs in the top-level page. Links inside embedded frames (captchas, login frames, video embeds) are not touched.

### Login and data persistence

The app renders the site with WebView2, which stores cookies, localStorage and cache in a **user data folder**. By default this is created next to the exe as `<AppName>.exe.WebView2`, and it is not deleted when the app closes. In practice:

- You normally stay logged in between launches, as long as the site uses persistent (expiring) cookies.
- To log out or reset the app, close it and delete that folder.

See [Microsoft's documentation on the user data folder](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/user-data-folder) for details.

---

## Potential drawbacks

- **Windows only.** The workflow uploads just the Windows build. Neutralino can build for macOS and Linux, but this workflow does not publish them.
- **Needs the WebView2 runtime.** It is preinstalled on current Windows 10 and 11. Older or stripped-down systems may need the [Evergreen runtime](https://developer.microsoft.com/en-us/microsoft-edge/webview2/) installed first.
- **Data folder lives next to the exe.**
  - Moving or renaming the exe starts a fresh profile, so you will be logged out.
  - If the exe sits in a write-protected location such as `Program Files`, WebView2 can fail to start. Use a normal user folder.
  - Whoever can open that folder can read the saved session, so treat it like a browser profile.
- **Session-only cookies are not kept.** Sites that rely on cookies with no expiry will log you out on every launch.
- **Fullscreen is a starting state only.** The app starts fullscreen, but F11 does not toggle it back to a window. Close the app with **Alt+F4**.
- **It is a thin wrapper, not a full browser.** There are no tabs, no address bar, no extensions and no built-in ad blocking.
- **External link handling covers clicks, not everything.** The script catches link clicks and `window.open()`. It cannot catch navigation the site triggers itself without a click, such as `location.href = ...` in a script, a redirect, or a submitted form. A site can still send the app to another domain that way.
- **Sign-in on another domain needs a manual allow-list.** The script cannot know which extra domains a site uses to log in. List them in **Extra domains that must stay inside the app**, or the login page opens in your browser and the sign-in fails inside the app.
- **Popup-based sign-ins are fragile.** A same-site `window.open()` navigates the current window and returns nothing to the page, and a blank popup (`window.open('')`) opens in your browser. "Log in with ..." popups that talk back to the opener window will not work. Untick **Keep the app on its own site** for a site like that.
- **Domain matching is a heuristic.** "Own site" is the last two parts of the host name, plus a built-in list of common two-part endings. Hosting platforms that share one domain between unrelated sites (for example `github.io`) are all treated as one site.
- **Links inside iframes are not intercepted.** Only the top-level page is guarded.
- **Untested in some situations.** The script is bundled in the exe and its link logic is covered by automated tests, but whether Neutralino runs it on every remote page is based on its documentation. Check it once after building (see Troubleshooting).
- **Some sites restrict embedded browsers.** Certain sign-in flows (Google sign-in is a known example) may refuse to work inside an embedded webview.
- **Unsigned executable.** Windows SmartScreen or antivirus software may warn about it, because it is not code-signed.
- **Default icon.** Every app uses the Neutralino template icon. A custom icon is not part of this workflow yet.
- **Nightly core binaries.** The template config uses Neutralino's `nightly` binary version, so a build made months later may behave slightly differently from an earlier one.
- **No auto-update or installer.** Each build is a single portable exe. Rebuild to change the URL, name or options.
- **Artifacts expire.** GitHub deletes workflow artifacts after the repository's retention period (90 days by default). Keep a copy of any exe you want to keep.
- **Respect the site's terms.** You are loading someone else's website. Make sure wrapping it this way is allowed by its terms of service.

---

## Customizing

All options are set in the **Configure App for Remote URL** step of `dynamic-packager.yml`. Common tweaks:

- **Window size:** change `.modes.window.width` and `.modes.window.height`.
- **Keep DevTools available:** set `.modes.window.enableInspector` to `true`.
- **Borderless window:** add `.modes.window.borderless = true`.
- **New-window links:** `newWindowPolicy` accepts `system` (the webview's default behavior), `browser` (default browser) or `custom` (emits an event for your own handling).
- **Link rules:** the matching logic lives in the heredoc inside the same step of `dynamic-packager.yml`. Edit it there to change how links are classified.

The full list of window options is in the [Neutralinojs configuration reference](https://neutralino.js.org/docs/configuration/neutralino.config.json).

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| The app opens the default Neutralino page | The build was made without `--embed-resources`, or you copied an old exe. Re-run the workflow and use the new artifact. |
| DevTools opens with the window | `enableInspector` is still `true`. Make sure you are using the latest workflow file. |
| Links to other sites still open inside the app | Make sure **Keep the app on its own site** was ticked. If it was, the script may not be running. Rebuild with `enableInspector` set to `true`, open DevTools and look for errors in the Console. Also remember the script only sees clicks, not scripted redirects or form submits. |
| A login page opens in the browser and sign-in fails | The site signs in on another domain. Add that domain to **Extra domains that must stay inside the app** and rebuild. |
| You are logged out on every launch | The site uses session-only cookies, or the exe was moved or renamed. See *Login and data persistence*. |
| The window will not start | WebView2 may be missing or the exe is in a write-protected folder. Install the runtime or move the exe. |
| The workflow fails at `neu update` | The generated config is missing its `cli` block. Do not overwrite the whole config. Only modify fields with `jq`, as this workflow does. |

---

## Credits

- [Neutralinojs](https://neutralino.js.org/) for the lightweight runtime and CLI
- Microsoft WebView2 for rendering
