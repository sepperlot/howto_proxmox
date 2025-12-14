# howto_proxmox

Gotchas around Proxmox and installed LXCs

## paperless-ngx

Install via <https://community-scripts.github.io/ProxmoxVE/scripts?id=paperless-ngx>

To make it work via Nginx Proxy Manager (NPM) edit `/opt/paperless/paperless.conf` and set `PAPERLESS_URL` and `PAPERLESS_CSRF_TRUSTED_ORIGINS` to desired domains otherwise you'll get either CSRF errors after login or the dashboard won't fully load.

Stop running services:

```shell
systemctl stop paperless-consumer paperless-webserver paperless-scheduler paperless-task-queue
```

Edit config options

```shell
vi /opt/paperless/paperless.conf
PAPERLESS_URL=<https://paperless.example.com>
PAPERLESS_CSRF_TRUSTED_ORIGINS=<https://paperless.example.com,https://paperless-ngx.example.com> # can be set using PAPERLESS_URL
```

Start services

```shell
systemctl start paperless-consumer paperless-webserver paperless-scheduler paperless-task-queue
```

Enable Websockets Support in NPM  
Don't force SSL in NPM or else upload of files breaks!

### backup and restore of data/documents/settings/etc

My installation was running on a Debian 12 and for the life of me I couldn't get it upgraded to Debian 13 using <https://github.com/community-scripts/ProxmoxVE/discussions/7489> hence I decided to do a clean install of the LXC (see above) and just import my data.

Paperless offers an export/import function see <https://docs.paperless-ngx.com/administration/#exporter> and <https://docs.paperless-ngx.com/administration/#importer>  
In the old LXC I just exported the data as follows:

```shell
cd /opt/paperless/src
python3 manage.py document_exporter /home/$USER/ -z
```

Installed a new LXC and copied the zip file over (using `scp`).  
In the new container I had to source the `venv` first to get the script to run:

```shell
cd /opt/paperless
source .venv/bin/activate
cd src/
python3 manage.py document_importer /home/$USER/export-2025-12-14.zip
```

## bookstack

Install via <https://community-scripts.github.io/ProxmoxVE/scripts?id=bookstack>

To make it work via Nginx Proxy Manager (NPM) edit `/opt/bookstack/.env` and set `APP_URL` to desired domain. Otherwise it will only show the IP address in the browser. Didn't had to stop or start any services it just worked.

Edit config:

```shell
vi /opt/bookstack/.env
APP_URL=APP_URL=https://bookstack.example.com
```

I did enable Websockets Support in NPM but not sure if that is needed.

### rename organization elements

If you don't like "shelves", "books", etc. you can rename them using a custom script. This is done via the WebGUI. Head to **Settings -> Customization** and scroll down to **Custom HTML Head Content** and copy below script in there.

> [!NOTE]  
> This renaming works all over Bookstack except for when you in your settings menu. There it still will show shelves, etc.

```js
<script>
  // ============================
  // Replacement Terms
  // ============================
  // These redefine BookStack's default terms into business-relevant ones
  const replace = {
    "shelves": "categories",
    "shelf": "category",
    "books": "topics",
    "book": "topic",
    "chapters": "sections",
    "chapter": "section"
  };

  // Build a regex to match any of the above keys, word-boundary safe and case-insensitive
  const regex = new RegExp(`\\b(${Object.keys(replace).join("|")})\\b`, "gi");

  // ============================
  //  Formatting
  // ============================

  // Capitalizes just the first letter of a string
  function capitalizeFirstLetter(string) {
    return string.charAt(0).toUpperCase() + string.slice(1);
  }

  // Checks whether a node is inside actual editable page content (to be excluded from changes)
  function isInContentArea(node) {
    const main = node.closest('main');
    return main && main.querySelector('.page-content');  // Only skip areas that have .page-content
  }

  // ============================
  // Text Replacement
  // ============================

  function walkText(node) {
    // Tags that should never be touched (scripts, inputs, code, etc.)
    const skipTags = ["SCRIPT", "STYLE", "NOSCRIPT", "INPUT", "TEXTAREA", "CODE", "PRE", "KBD"];
    if (skipTags.includes(node.nodeName)) return;

    // Skip anything inside editable page content areas
    if (node.closest && isInContentArea(node)) return;

    // For text nodes, apply the replacement logic
    if (node.nodeType === 3) {
      node.data = node.data.replace(regex, function(matched) {
        const replacement = replace[matched.toLowerCase()];
        return matched.charAt(0) === matched.charAt(0).toUpperCase()
          ? capitalizeFirstLetter(replacement)  // Preserve capitalization
          : replacement;
      });
    }

    // For element nodes, recursively walk through child nodes
    if (node.nodeType === 1) {
      for (let i = 0; i < node.childNodes.length; i++) {
        walkText(node.childNodes[i]);
      }
    }
  }

  // ============================
  // On Page Load
  // ============================

  window.addEventListener('DOMContentLoaded', function () {
    // Update the page title if the first chunk matches a replaceable term
    let titleParts = document.title.split("|").map(t => t.trim());
    if (replace[titleParts[0].toLowerCase()]) {
      titleParts[0] = capitalizeFirstLetter(replace[titleParts[0].toLowerCase()]);
      document.title = titleParts.join(" | ");
    }

    // Begin DOM walk from the body, replacing only where allowed
    walkText(document.body);
  });
</script>
```

Sources:  
<https://github.com/BookStackApp/BookStack/issues/406#issuecomment-1156340618>  
<https://github.com/BookStackApp/BookStack/issues/406#issuecomment-2795312067>
