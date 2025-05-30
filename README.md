# Telegram_research2 – Telegram Channel BFS Scraper

Telegram_research2 is a Telegram channel scraping tool that uses a breadth-first search (BFS) "snowball" approach to discover and collect messages from Telegram channels, especially those potentially spreading misinformation. Starting from an initial seed list of channels, the scraper iteratively finds new channels referenced in messages (via @ mentions, forwarded message headers, or t.me links) and adds them to the crawl. This allows researchers to map out networks of related channels and gather their messages for analysis.

---

## Features and Purpose

- **Recursive Channel Discovery:** Automatically finds and adds new channels referenced by the current channels’ messages, performing a multi-layer BFS traversal of the network. This "snowball sampling" helps uncover clusters of related channels (e.g. misinformation networks) beyond the initial seed list.

- **Message Collection:** Fetches messages from each channel (optionally the most recent messages or the entire history, as configured) and saves them to a CSV file for analysis. By default, output is saved in `messages.csv`, with each row containing the message content and metadata (such as channel name and timestamp).

- **Filtering of Irrelevant Mentions:** Ignores junk or non-channel references to avoid false positives. For example, generic mentions like `@here`, `@all`, or nonsensical handles (e.g. `@this`) are filtered out and not treated as channel links.

- **Channel Validation:** Every discovered channel handle is validated using Telegram’s API before adding it to the BFS queue. This ensures the handle corresponds to an actual Telegram channel (not a private user or invalid name) and prevents duplicates or revisiting channels that were already processed.

- **Robust to API Limits:** Implements strategies to respect Telegram API rate limits and avoid flood-wait errors. Requests are batched and throttled – the scraper may fetch data from several channels in batches and then pause as needed. It gracefully handles Telegram’s `FloodWaitError` by pausing for the required duration, ensuring the scraping can continue reliably even for large networks of channels.

- **Easy to Use:** The project is provided as a Python script/Jupyter Notebook using the Telethon library (an async Telegram API client). With a few configuration steps (like inserting your API credentials), you can run the scraper and start collecting data. The repository includes an example seed list (`final_tele.csv`) of channels related to crypto scams and anti-vax misinformation, so you can quickly test the tool or modify it for your own research.

---

## How the BFS Channel Discovery Works

### Overview
The scraper treats Telegram channels as nodes in a graph, where an edge from Channel A to Channel B exists if A’s messages reference B. It performs a breadth-first traversal starting from the seed channels, discovering new channels layer by layer. Below is a walkthrough of the process:

1. **Seed Channels Initialization:** The scraper reads the list of starting channels from `final_tele.csv` (a CSV containing seed channel usernames). These seeds form the first BFS layer. You can customize this list with your own targets (e.g. channels suspected of misinformation) – for convenience, the provided file already lists dozens of such channels with their descriptions.

2. **Fetching Messages:** For each channel in the current layer, the scraper connects to the Telegram API (via Telethon) and retrieves that channel’s messages. By default, it may fetch a certain number of recent messages (to limit volume and focus on current activity), but it can be configured to get more history if needed. Messages are collected asynchronously in batches to improve speed while respecting rate limits.

3. **Extracting Channel Links:** Each message is scanned for any references to other channels:
   - **@mentions:** If the message text contains an `@username` mention, the username is extracted. The scraper filters out obvious non-channel usernames (for example, short or generic words that are not valid Telegram handles). Only plausible channel handles (e.g. `@SomeChannelName`) pass to validation.
   - **t.me Links:** The scraper also looks for Telegram invite/share links in the text (e.g. `https://t.me/SomeChannel` or `t.me/SomeChannel`). It parses out the username part of such URLs. This catches cases where a channel link is shared but not using the `@` syntax.
   - **Forwarded From Headers:** Telegram messages often include a header if they were forwarded from another channel. Using Telethon, the scraper checks if a message was forwarded (`message.fwd_from` data). If so, it obtains the original channel’s identifier (and attempts to resolve its username or channel ID). That origin channel is treated as another discovered node.

4. **Validation of Discovered Handles:** Every unique channel handle found in step 3 goes through a validation check (`is_valid`). This typically involves:
   - Ensuring the handle meets Telegram’s username criteria (e.g. length and allowed characters, skipping words like "here" or "admin" that are not actual channels).
   - Using the Telegram API to resolve the username to an entity. The scraper uses Telethon’s client methods to get the channel info. If the username is invalid, or corresponds to a user or group chat rather than a public channel, it’s discarded. Only valid channels (usually Telegram Channel or MegaGroup entities) that are publicly accessible are kept for the next layer.
   - Avoiding duplicates: the scraper keeps a record of all channels seen so far (including seeds and newly added ones). This prevents processing the same channel multiple times.

5. **BFS Layer Expansion:** All newly validated channels discovered from the current layer’s messages are aggregated into a list (the next layer of the BFS). Once the current layer’s channels are processed, the scraper moves on to the next layer:
   - The new channels are queued and then their messages will be fetched and scanned in the same manner.
   - This process repeats, layer by layer, until no new channels are found or until an optional maximum depth/rounds is reached (to avoid infinite traversal if the network loops, though the duplicate-check prevents actual cycles).

6. **Termination:** The BFS ends when no more new channel links are found in the latest layer, or when a preset limit (such as a maximum number of channels or layers) is reached. At that point, the scraper will have collected messages from the entire discovered network of channels.

This BFS approach ensures a comprehensive discovery of related channels. For example, starting from a few known misinformation channels, the tool can uncover dozens or hundreds of others that are interconnected via cross-posts, mentions, or forwards. This is highly useful for research mapping how misinformation spreads across Telegram.

---

## Handling API Rate Limits and Flood Waits

Scraping many channels and messages can hit Telegram’s API rate limits, which triggers flood wait errors (Telegram instructs the client to slow down requests for a certain number of seconds). The Telegram_research2 scraper is designed with mechanisms to stay within safe limits and handle these situations gracefully:

### Batching Requests
Instead of making API calls for each message or channel one by one in a rapid-fire fashion, the scraper groups operations. For example, it may fetch messages from a few channels concurrently (using Telethon’s asynchronous capabilities) and then process them together. This reduces the overhead of separate calls and can be more efficient.

### Throttling & Pauses
Between batches or after a set number of API calls, the code inserts delays (pauses) to avoid triggering the flood limit. These delays might be tuned to Telegram’s known thresholds (which are not officially documented, but generally, actions like resolving usernames or fetching history too quickly will cause a wait). The result is a steadier request rate.

### FloodWaitError Handling
If a flood wait does occur, Telethon will raise a `FloodWaitError` exception indicating how long to wait. The scraper catches these exceptions and sleeps for the required duration automatically. For example, if Telegram responds that you must wait 60 seconds, the script will pause itself for that period before retrying. This ensures the scraping can resume from where it left off without crashing.

### Session Caching
Telethon caches certain info (like resolved usernames to channel IDs) in the session file. The scraper leverages this so that once a channel is resolved or accessed, subsequent operations won’t repeatedly hit the network for the same data. This cuts down on redundant calls (which also helps avoid flood limits). Additionally, the list of discovered channels is built up so we never attempt to resolve or fetch the same channel twice.

### Adjustable Parameters
You can configure aspects like how many messages to fetch per channel, or insert manual `time.sleep()` intervals if needed, to further control the pace. By tweaking these, you can crawl slower or faster depending on your risk tolerance for rate limiting. The default settings aim for a robust, safe crawl that might sacrifice some speed in exchange for not getting blocked by Telegram.

With these strategies, Telegram_research2 can scrape large networks of channels over extended runs. It mitigates the risk of the script being stopped mid-way due to hitting Telegram’s limits. However, it’s still good practice for users to monitor the output and adjust settings if they plan to crawl extremely large numbers of channels or very deep networks.

---

## Filtering and Validation Mechanisms

A crucial part of the scraper’s reliability is how it filters out noise and ensures only real, relevant channels are pursued. The code includes the following filtering/validation steps (inside the `is_valid` logic and related checks):

1. **Format and Length Check:** Discovered strings that start with `@` or come from `t.me` links are first checked against Telegram’s username rules. Telegram channel usernames must be 5 to 32 characters long and can only contain letters, numbers, and underscores (and must start with a letter). The scraper quickly skips any mention that doesn’t fit this pattern. For instance, `@hi` or `@$$$` or `@john` (too short) would be dropped immediately.

2. **Known Junk Usernames:** Some words are commonly present in text with an @ but are not actually usernames. The script maintains a small blacklist or heuristic for these. Examples include `@here` or `@everyone` (which are Slack/Discord-style mentions, not Telegram) or things like `@channel`/`@admin` which might appear in copy-pasted text but aren’t actual channels. These are filtered out to avoid needless API calls.

3. **Duplicate Prevention:** Before validating a new channel handle, the scraper checks if it’s already been seen. If we have already added or processed that channel in a previous layer, it will not be added again. This protects against loops and reduces redundant processing.

4. **Telegram API Entity Resolution:** For each candidate channel username that passes the above filters, the scraper uses Telethon to resolve the entity. This typically means calling something like `client.get_entity(username)` which will return the Telegram entity (Channel, Supergroup, User, etc.) if it exists. The scraper examines the result:
   - If the entity is a Channel or Supergroup (megagroup), it’s likely a valid target to crawl. These are accepted. (Both broadcast channels and large public groups are considered, since both can spread information. Depending on your research focus, you might treat them differently, but the tool can handle either.)
   - If the username is valid but points to a User (a person’s account) or a small private chat that cannot be accessed, then it’s not what we want – such cases are skipped. The BFS is focused on public channels that one can join or at least view.

5. **Inaccessible or deleted channels** are also skipped. For example, if a channel was taken down or is private/invite-only, the API might throw an error or return a limited object. The script catches these outcomes and ignores those entries.

6. **Logging/Debugging:** (Optional) The code can be extended to log which references were skipped and why (e.g., “Skipped @foobar – not a valid channel”). In the current implementation, the focus is on just collecting valid channels and moving on, but as a user you can always add print statements or logs if you need insight into the filtering decisions.

By applying these filters, the scraper ensures that it spends time only on meaningful channels and avoids chasing false leads or crashing on invalid data. This is especially important in misinformation content, where posts might contain strange text or bait that could look like a mention but isn’t a real channel. The validation step acts as a safeguard, improving the quality of the channel network that gets explored.

---

## Installation and Setup

To use this scraper, you’ll need a Python environment set up with the required libraries and Telegram API credentials:

1. **Clone the Repository:**

   Download or clone Telegram_research2 from GitHub. It contains the main scraper code (as a Jupyter Notebook `Telegram (5) (24).ipynb`), the seed list CSV, and a placeholder README.

2. **Install Dependencies:**

   Make sure you have Python 3.7+ installed. The main dependency is Telethon (for Telegram API access). You may also need some standard libraries (which are likely already installed, such as `asyncio`, `csv` or `pandas` if used for output). To install Telethon and any other requirements, you can use pip:

   ```bash
   Copy
   Edit
   pip install telethon
