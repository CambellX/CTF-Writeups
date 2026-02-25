# web/WayWayback Machine
> I got tired of having to dig around the internet for old files so I started archiving them myself.

## Background
We are given a simple Wayback website with functionality to create snapshots of a website.
<body align="left">
  <img src = "WaybackHome.jpg" width=400>
</body>

Upon entering a website name, the site immediately sets out to generate a snapshot of that site.

<body align="left">
  <img src = "usage.jpg" width=400>
</body>

## Analyzing source code
Lets take a look at the snapshot functionality to see how a site takes a snapshot.

```
app.post('/api/snapshot', async (req, res) => {
  try {
    const { url } = req.body;
    
    if (!url || typeof url !== 'string') {
      return res.status(400).json({ error: 'URL required' });
    }

    // Basic URL validation
    let targetUrl;
    try {
      targetUrl = new URL(url);
      if (!['http:', 'https:'].includes(targetUrl.protocol)) {
        return res.status(400).json({ error: 'Only HTTP/HTTPS URLs allowed' });
      }
    } catch (e) {
      return res.status(400).json({ error: 'Invalid URL format' });
    }

    console.log(`Creating snapshot for: ${url}`);
    
    // Generate snapshot ID
    const snapshotId = generateId();
    snapshotStatuses.set(snapshotId, { status: 'pending' });

    res.json({
      success: true,
      snapshotId,
      url: `/snapshot/${snapshotId}`,
      targetUrl: url,
      message: 'Snapshot queued! Our bot will visit and archive your page shortly.'
    });

    // Trigger Snapshot bot visit asynchronously - bot will fetch and save the page
    setTimeout(() => {
      visitAndSaveSnapshot(snapshotId, url).catch(err => {
        console.error(`Error creating snapshot ${snapshotId}:`, err.message);
        snapshotStatuses.set(snapshotId, { status: 'failed', error: err.message });
      });
    }, 2000);    
  } catch (error) {
    console.error('Error creating snapshot:', error);
    res.status(500).json({ error: 'Failed to create snapshot' });
  }
});
```
First, our snapshots must be either http: or https:, so no file:// or data:// URL schemes will work.
```
if (!['http:', 'https:'].includes(targetUrl.protocol)) {
        return res.status(400).json({ error: 'Only HTTP/HTTPS URLs allowed' });
      }
```
Then we call the function generateId() and set the snapshot status to pending.
```
    const snapshotId = generateId();
    snapshotStatuses.set(snapshotId, { status: 'pending' });
```
And lastly, we call the visitAndSaveSnapshot function.
```
setTimeout(() => {
      visitAndSaveSnapshot(snapshotId, url).catch(err => {
        console.error(`Error creating snapshot ${snapshotId}:`, err.message);
        snapshotStatuses.set(snapshotId, { status: 'failed', error: err.message });
      });
    }, 2000);    
```
On it's own, this code doesn't seem particularly vulnerable. We would need to take a deeper look into the underlying helper functions that it calls.

First, we check generateId. All it does is generate a few random characters. Not helpful.
```
function generateId() {
  return crypto.randomBytes(8).toString('hex');
}
```

The other function is called visitAndSaveSnapshot().
```
async function visitAndSaveSnapshot(snapshotId, targetUrl) {
  console.log(`Snapshot bot fetching: ${targetUrl}`);

  const browser = await puppeteer.launch({
    headless: 'new',
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      '--disable-dev-shm-usage',
      '--disable-features=HttpsFirstBalancedModeAutoEnable'
    ]
  });

  try {
    const page = await browser.newPage();

    console.log(`Navigating to: ${targetUrl}`);

    // Use faster wait condition and longer timeout
    await page.goto(targetUrl, {
      waitUntil: 'domcontentloaded',
      timeout: 30_000
    });

    // Much shorter wait - just let DOM settle
    await new Promise(resolve => setTimeout(resolve, 500));

    console.log(`Page loaded, capturing snapshot...`);

    const htmlContent = await page.content();

    const snapshotPath = path.join(SNAPSHOTS_DIR, `${snapshotId}.html`);
    await fs.promises.writeFile(snapshotPath, sanitizeHtmlContent(htmlContent));

    console.log(`Snapshot saved: ${snapshotId}`);

    // Archive resources asynchronously in background - don't block
    setImmediate(() => {
      archiveResources(htmlContent, targetUrl)
        .then(() => {
          snapshotStatuses.set(snapshotId, { status: 'complete' });
        })
        .catch(err => {
          console.error('Background resource archiving error:', err.message);
      });
    });

    console.log(`Snapshot bot finished processing: ${snapshotId}`);
  } catch (err) {
    console.error('Error in Snapshot bot:', err);
  } finally {
    await browser.close();
  }
}
```
This one seems a bit more interesting.
First, we launch a puppeteer bot and visit the user supplied URL.
```
const browser = await puppeteer.launch({
    headless: 'new',
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      '--disable-dev-shm-usage',
      '--disable-features=HttpsFirstBalancedModeAutoEnable'
    ]
  });
  const page = await browser.newPage();
  
      console.log(`Navigating to: ${targetUrl}`);
  
      // Use faster wait condition and longer timeout
      await page.goto(targetUrl, {
        waitUntil: 'domcontentloaded',
        timeout: 30_000
      });
```

We obtain the HTML content of that page and save a snapshot of that page using the writeFile() function. (presumably sanitizing it given the input). 
```
const htmlContent = await page.content();

    const snapshotPath = path.join(SNAPSHOTS_DIR, `${snapshotId}.html`);
    await fs.promises.writeFile(snapshotPath, sanitizeHtmlContent(htmlContent));
```

Then we call the archiveResources function, which seems to be the backbone of the archival logic.

```
setImmediate(() => {
      archiveResources(htmlContent, targetUrl)
        .then(() => {
          snapshotStatuses.set(snapshotId, { status: 'complete' });
        })
        .catch(err => {
          console.error('Background resource archiving error:', err.message);
      });
    });
```
At a glance, archiveResources doesn't seem to do anything too dangerous on it's own. The majority of it's functionality seems to come out of its helper functions. 
```
async function archiveResources(htmlContent, targetUrl) {
  console.log(`Extracting archive resources from page...`);
  const resourceUrls = extractResourceUrls(htmlContent, targetUrl);
  
  console.log(`Found ${resourceUrls.length} resources to archive`);
  
  for (const resourceUrl of resourceUrls) {
    try {
      console.log(`Archiving resource: ${resourceUrl}`);
      
      // Parse the URL
      const urlObj = new URL(resourceUrl);
      let filename = path.basename(urlObj.pathname);
      
      // Sanitize filename
      filename = filename.replace(/[^a-zA-Z0-9._-]/g, '_');
      
      if (!filename || filename === '_') {
        filename = 'resource_' + crypto.randomBytes(4).toString('hex');
      }
      
      const savePath = path.join(SNAPSHOTS_DIR, filename);
      
      console.log(`Saving to: ${savePath}`);
      
      // Download the resource
      await downloadFile(resourceUrl, savePath);
      console.log(`Resource archived: ${filename}`);
      
    } catch (err) {
      console.error(`Failed to archive resource ${resourceUrl}:`, err.message);
    }
  }
  
  console.log(`Finished archiving resources`);
}
```
downloadFile() seems like an interesting function with the most room for error. 
```
async function downloadFile(url, savePath) {
  return new Promise((resolve, reject) => {
    const protocol = url.startsWith('https') ? https : http;
    
    protocol.get(url, (response) => {
      if (response.statusCode !== 200) {
        reject(new Error(`Failed to download: ${response.statusCode}`));
        return;
      }
      
      const fileStream = fs.createWriteStream(savePath);
      response.pipe(fileStream);
      
      fileStream.on('finish', () => {
        fileStream.close();
        resolve();
      });
      
      fileStream.on('error', (err) => {
        fs.unlink(savePath, () => {});
        reject(err);
      });
    }).on('error', reject);
  });
}
```
Alright, no luck. All it does is write a saved file to a local save path. 

At this point, I realized that I've tunnelvisioned too hard into the logic flow for the creation of snapshots. But what about everything else?
Looking at how the app decides to view the snapshot, theres hardly any functionaltiy other than calling the preloadSnapshotResources() function.
```
app.get('/snapshot/:id', async (req, res) => {
  const snapshotId = req.params.id;
  const snapshotPath = path.join(SNAPSHOTS_DIR, `${snapshotId}.html`);
  
  if (!fs.existsSync(snapshotPath)) {
    return res.status(404).send('Snapshot not found');
  }
  
  // Pre-load snapshot resources for better performance
  await preloadSnapshotResources(snapshotId);
  
  res.sendFile(snapshotPath);
});
```

Aaaand looking at this, we can immediately tell whats wrong.
```
async function preloadSnapshotResources() {
  try {
    const entries = fs.readdirSync(SNAPSHOTS_DIR, { withFileTypes: true });

    for (const entry of entries) {
      if (!entry.isFile()) continue;

      const filePath = path.join(SNAPSHOTS_DIR, entry.name);      
      // Load optimization helpers
      if (path.extname(entry.name) === '.js') {
        try {
          require(filePath);
        } catch (err) {
          // Skip invalid helpers
        }
      }
    }
  } catch (error) {
    // Ignore resource loading errors
  }
}
```
We use fs.readdirSync to read the previously saved snapshot on our local machine. Then we look through the contents of that snapshot... and require all the .js files...

require() dynamically loads any modules that are hosted on the client-side script which will then run those javascripts locally. This basically just gave us RCE for free. 

## Obtaining the flag
Now that we have all the pieces of the puzzle, we can formulate a solve. 

The process is going to look somewhat like this.

1. Host an attacker controlled server accessible over the internet.
2. On that server, create dynamically loaded modules containing malicious code.
3. Supply the URL of that server to our snapshot machine.
4. Obtain RCE after the machine automatically downloads those files locally. 
## References
https://medium.com/@mtorre4580/understanding-require-function-node-js-bbda09952ded
